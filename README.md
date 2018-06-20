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
