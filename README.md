### Create VM's for 3 master, 4 Worker, 1 loadbalancer :

- Install Virtual box , Vagrant in local machine.
- create directories for 3 master, 4 Worker, 1 loadbalancer VM 
- Place below VagrantFile in all the directories  with different static private ip and VM name 

```
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
```
- Go to each directory and start the VM using below command 
```
vagrant up 
```

- In this example below is the VM configuration. copy this on each vm in /etc/hosts file.
```
10.0.0.51    master01
10.0.0.52    master02
10.0.0.53    master03
10.0.0.54    worker01
10.0.0.55    worker02
10.0.0.56    worker03
10.0.0.57    worker04
10.0.0.58    lb01
```

#### Note:
- Login as root user to all your machines and not switch unless it is required in the document

### Install pre-requisites softwares :

- Install docker on all the machine (master, worker, load balancer)
```
apt-get update
apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
```

- Installing kubeadm, kubelet and kubectl on all the machine (master, worker, load balancer)
```
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```
- Export environment variable on all master nodes 
```
export PEER_NAME=$(hostname)
export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+')
```
- Install etcd only on master vm's
```
touch /etc/etcd.env
echo "PEER_NAME=${PEER_NAME}" >> /etc/etcd.env
echo "PRIVATE_IP=${PRIVATE_IP}" >> /etc/etcd.env
mkdir -p /var/lib/etcd
ETCD_VERSION="v3.1.12"
curl -sSL https://github.com/coreos/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz | tar -xzv --strip-components=1 -C /usr/local/bin/
```
- Install cfssl and cfssljson on all master vm's:
```
curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x /usr/local/bin/cfssl*
```
### Generate Certificates :
- Fist generate etcd certs on Master1 vm only
  - Create /etc/kubernetes/pki/etcd directory
```
mkdir -p /etc/kubernetes/pki/etcd
cd /etc/kubernetes/pki/etcd
```
  - Create below json files
```
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
```
  - Now generate the CA certs and etcd client certs
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
```

  - Copy certificates to other master nodes from master1 
```
scp root@10.0.0.51:/etc/kubernetes/pki/etcd/ca.pem .
scp root@10.0.0.51:/etc/kubernetes/pki/etcd/ca-key.pem .
scp root@10.0.0.51:/etc/kubernetes/pki/etcd/client.pem .
scp root@10.0.0.51:/etc/kubernetes/pki/etcd/client-key.pem .
scp root@10.0.0.51:/etc/kubernetes/pki/etcd/ca-config.json .
```
  - Once this is done, run the following on all master nodes to generate peer.pem, peer-key.pem, server.pem, server-key.pem
```
cfssl print-defaults csr > config.json
sed -i '0,/CN/{s/example\.net/'"$PEER_NAME"'/}' config.json
sed -i 's/www\.example\.net/'"$PRIVATE_IP"'/' config.json
sed -i 's/example\.net/'"$PEER_NAME"'/' config.json
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server config.json | cfssljson -bare server
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer config.json | cfssljson -bare peer
```
### Create etcd systemd service on all the master VM's :
- Create /etc/systemd/system/etcd.service file
```
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
```
- Start etcd service once it is setup on all the master nodes.
```
systemctl daemon-reload
systemctl start etcd
```
- If there any change in the etcd.service file restart etcd service
```systemctl restart etcd ```
- Check if ETCD cluster is configured correctly
```
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
```  
