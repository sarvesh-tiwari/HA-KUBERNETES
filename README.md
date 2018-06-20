# HA-KUBERNETES
### Create VM for 3 master, 3 Worker, 1 loadbalancer
-- Install Virtual box , Vagrant in local machine.

-- create directories for 3 master, 3 Worker, 1 loadbalancer VM 
-- Place below VagrantFile in all the directories  with different static private ip and VM name 

cat <<__EOF__>Vagrantfile 
Vagrant.configure("2") do |config|
  
  config.vm.box = "ubuntu/xenial64"
  config.vm.define "master01"
  config.vm.hostname = "master01"
  config.vm.network "private_network", ip: "10.0.0.51"

    config.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.name = "master01"
    end
    config.vm.provision "shell", inline: <<-SHELL
      apt-get update && apt-get install -y curl apt-transport-https
    SHELL
end
__EOF__

-- Go to each directory and start the VM using below command 
	vagrant up 

- Login as root user to all your machines and not switch user unless it is required in the document

- Install pre-requisites softwares:

-- Install docker on all the machine (master, worker, load balancer any ..)

apt-get update
apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')

-- Installing kubeadm, kubelet and kubectl on all the machine (master, worker, load balancer any ..)

apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

-- Export environment variable on all master nodes 
export PEER_NAME=$(hostname)
export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+')


-- Install etcd only on master nodes 

touch /etc/etcd.env
echo "PEER_NAME=${PEER_NAME}" >> /etc/etcd.env
echo "PRIVATE_IP=${PRIVATE_IP}" >> /etc/etcd.env
mkdir -p /var/lib/etcd

ETCD_VERSION="v3.1.12"
curl -sSL https://github.com/coreos/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz | tar -xzv --strip-components=1 -C /usr/local/bin/

- Generate Certificates:

-- Install cfssl and cfssljson on all etcd nodes:

curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x /usr/local/bin/cfssl*

-- Fist generate etcd certs on Master1 only

mkdir -p /etc/kubernetes/pki/etcd
cd /etc/kubernetes/pki/etcd
 
--create below json files

cat <<__EOF__>ca-config.json
 {
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
 }
__EOF__

cat <<__EOF__>ca-csr.json
 {
    "CN": "etcd",
    "key": {
        "algo": "rsa",
        "size": 2048
    }
 }
__EOF__

cat <<__EOF__>client.json
{
  "CN": "client",
  "key": {
      "algo": "ecdsa",
      "size": 256
  }
}
__EOF__

-- Now generate the CA certs and etcd client certs
   cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client


-- Copy certificates to other master nodes from master1 

   scp root@10.0.0.51:/etc/kubernetes/pki/etcd/ca.pem .
   scp root@10.0.0.51:/etc/kubernetes/pki/etcd/ca-key.pem .
   scp root@10.0.0.51:/etc/kubernetes/pki/etcd/client.pem .
   scp root@10.0.0.51:/etc/kubernetes/pki/etcd/client-key.pem .
   scp root@10.0.0.51:/etc/kubernetes/pki/etcd/ca-config.json .

-- Once this is done, run the following on all master nodes to generate peer.pem, peer-key.pem, server.pem, server-key.pem
   
   cfssl print-defaults csr > config.json
   sed -i '0,/CN/{s/example\.net/'"$PEER_NAME"'/}' config.json
   sed -i 's/www\.example\.net/'"$PRIVATE_IP"'/' config.json
   sed -i 's/example\.net/'"$PEER_NAME"'/' config.json
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server config.json | cfssljson -bare server
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer config.json | cfssljson -bare peer
 

-- Create etcd systemd service on all the master nodes

cat <<__EOF__>/etc/systemd/system/etcd.service
 [Unit]
 Description=etcd
 Documentation=https://github.com/coreos/etcd
 Conflicts=etcd.service
 Conflicts=etcd2.service

 [Service]
 EnvironmentFile=/etc/etcd.env
 Type=notify
 Restart=always
 RestartSec=5s
 LimitNOFILE=40000
 TimeoutStartSec=0

 ExecStart=/usr/local/bin/etcd --name ${PEER_NAME} --data-dir /var/lib/etcd --listen-client-urls https://${PRIVATE_IP}:2379,http://127.0.0.1:2379 --advertise-client-urls https://${PRIVATE_IP}:2379 --listen-peer-urls https://${PRIVATE_IP}:2380 --initial-advertise-peer-urls https://${PRIVATE_IP}:2380 --cert-file=/etc/kubernetes/pki/etcd/server.pem --key-file=/etc/kubernetes/pki/etcd/server-key.pem --client-cert-auth --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem --peer-cert-file=/etc/kubernetes/pki/etcd/peer.pem --peer-key-file=/etc/kubernetes/pki/etcd/peer-key.pem --peer-client-cert-auth --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem --initial-cluster master01=https://10.0.0.51:2380,master02=https://10.0.0.52:2380,master03=https://10.0.0.53:2380 --initial-cluster-token my-etcd-token --initial-cluster-state new

 [Install]
 WantedBy=multi-user.target
__EOF__

-- Start etcd service once it is setup on all the master nodes.
    systemctl daemon-reload
    systemctl start etcd

- if there any change in the etcd.service file restart etcd service
    systemctl restart etcd


-- Check if ETCD cluster is configured correctly

   etcdctl --ca-file /etc/kubernetes/pki/etcd/ca.pem --cert-file /etc/kubernetes/pki/etcd/server.pem --key-file /etc/kubernetes/pki/etcd/server-key.pem member list

	2018-06-20 18:53:48.437921 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
	484616d98f8bbc1a: name=master03 peerURLs=https://10.0.0.53:2380 clientURLs=https://10.0.0.53:2379 isLeader=false
	72713cba360e05c0: name=master02 peerURLs=https://10.0.0.52:2380 clientURLs=https://10.0.0.52:2379 isLeader=false
	7ee96495530e23e0: name=master01 peerURLs=https://10.0.0.51:2380 clientURLs=https://10.0.0.51:2379 isLeader=true

   etcdctl --ca-file /etc/kubernetes/pki/etcd/ca.pem --cert-file /etc/kubernetes/pki/etcd/server.pem --key-file /etc/kubernetes/pki/etcd/server-key.pem cluster-health

	member 484616d98f8bbc1a is healthy: got healthy result from https://10.0.0.53:2379
	member 72713cba360e05c0 is healthy: got healthy result from https://10.0.0.52:2379
	member 7ee96495530e23e0 is healthy: got healthy result from https://10.0.0.51:2379
	cluster is healthy

- Now Setup your nginx load balancer on load balancer vm

-- install nginx on lb vm  

	apt-get install nginx nginx-extras

	cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

-- Create nginx config file as service 

cat <<__EOF__>/etc/nginx/nginx.conf
worker_processes  1;
include /etc/nginx/modules-enabled/*.conf;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
stream {
        upstream apiserver {
            server 10.0.0.51:6443 weight=5 max_fails=3 fail_timeout=30s;
            server 10.0.0.52:6443 weight=5 max_fails=3 fail_timeout=30s;
            server 10.0.0.53:6443 weight=5 max_fails=3 fail_timeout=30s;
            #server ${HOST_IP}:6443 weight=5 max_fails=3 fail_timeout=30s;
            #server ${HOST_IP}:6443 weight=5 max_fails=3 fail_timeout=30s;
        }

    server {
        listen 6443;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass apiserver;
    }
}
__EOF__

-- Start nginx service 
	systemctl restart nginx
	systemctl status nginx


- Run kubeadm init on master1 vm first

-- change directory to root user home 
	cd ~

-- Create config yaml as below

cat <<__EOF__>config.yaml
  apiVersion: kubeadm.k8s.io/v1alpha1
  kind: MasterConfiguration
  api:
    advertiseAddress: 10.0.0.58
    controlPlaneEndpoint: 10.0.0.58
  etcd:
    endpoints:
    - https://10.0.0.51:2379
    - https://10.0.0.52:2379
    - https://10.0.0.53:2379
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd/client.pem
    keyFile: /etc/kubernetes/pki/etcd/client-key.pem
  networking:
    podSubnet: 10.244.0.0/16
  apiServerCertSANs:
  - master01
  - master02
  - master03
  - lb01.home
  - lb01
  - 10.0.0.51
  - 10.0.0.52
  - 10.0.0.53
  - 10.0.0.58
  apiServerExtraArgs:
    apiserver-count: "3"
__EOF__

where 10.0.0.58 is loadbalancer IP, 10.0.0.51,10.0.0.52,10.0.0.53 etcd host IP's and 2379 is client-listner-port 

-- Run kubeadm using below command 
	kubeadm init --config config.yaml
	
	NOTE: Watch the instructions to copy the admin config and a token which minions can use to join the cluster. You can make note of the token command to use later in this documentation 

--Copy the admin config to kubeadmin user home directory

--- Create kubeadmin user and add to the docker group 
	useradd -m -s $(which bash) -G sudo kubeadmin
	sudo usermod -aG docker kubeadmin
	
--- Copy admin config to kubeadmin user home directory 

	su - kubeadmin
	rm -rf .kube
	mkdir -p $HOME/.kube
  	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  	sudo chown $(id -u):$(id -g) $HOME/.kube/config

- Now run kubeadm on other master vm's one by one

--SSH to other master VM change directory to the root user home 
	cd ~
-- SCP Cert's and config.yaml from master 1 vm 
	scp root@10.0.0.51:/root/config.yaml .
	scp root@10.0.0.51:/etc/kubernetes/pki/* /etc/kubernetes/pki 
	rm /etc/kubernetes/pki/apiserver*

-- Now run kubeadm 
	kubeadm init --config config.yaml
-- Repeat ( as mentioned above on) creating kubeadmin user and copy admin config to kubeadmin user home directory other master nodes 	
	
- Check all master nodes running kubeadm 
	kubectl get nodes
	
	NAME             STATUS     ROLES     AGE       VERSION
	master01         Ready      master    41m       v1.10.4
	master02         Ready      master    16m       v1.10.4
	master03         Ready      master    15m       v1.10.4

-- Run kube-flannel a CNI network add on on all master node one by one from kubeadmin user

   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml

-- Check kube-dns pod is up and running, you can continue by joining your nodes.
	kubectl get pods --all-namespaces -owide
	
-- Join worker vm to the cluster by running below command on worker vm  
	SSH to worker vm and switch to root user then run below command and pass the token and ca cert which was generate while running the kubeadm init command.	
	kubeadm join --token <token> master01:6443 --discovery-token-ca-cert-hash sha256:<hash>	

- Deploy dashboard UI run below command on one of the master vm from kubeadmin user

	kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
-- Edit kubernetes-dashboard service to change spec type from type: ClusterIP to type: NodePort
	kubectl -n kube-system edit service kubernetes-dashboard
	
-- Check nodePort on which Dashboard was exposed.
   kubectl -n kube-system get service kubernetes-dashboard

-- Check on which worker node dashboard pod is running (use the actual ip of that worker vm not the ip shown in below command)
	kubectl get pods --all-namespaces -owide

-- Access the dashboard on browser by using below URL 
   https://<worker-vm-ip>:<nodePort>
