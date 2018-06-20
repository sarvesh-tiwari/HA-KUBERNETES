### Create VM for 3 master, 3 Worker, 1 loadbalancer

- Install Virtual box , Vagrant in local machine.
- create directories for 3 master, 3 Worker, 1 loadbalancer VM 
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
