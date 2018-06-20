### Create VM's for 3 master, 4 Worker, 1 loadbalancer

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

####Note:#### Login as root user to all your machines and not switch unless it is required in the document
