# -*- mode: ruby -*- 
# vi: set ft=ruby : vsa
Vagrant.configure("2") do |config| 
 config.vm.box = "generic/ubuntu2204" 
 config.vm.provider "virtualbox" do |v| 
 v.memory = 2048 
 v.cpus = 2 
 end 
 config.vm.define "backup" do |backup| 
 backup.vm.network "private_network", ip: "192.168.50.10",  virtualbox__intnet: "net1" 
 backup.vm.hostname = "backup" 
 backup.vm.disk :disk, name: "backup", size: "2GB", file: "./backup.vdi"
 end 
 config.vm.define "client" do |client| 
 client.vm.network "private_network", ip: "192.168.50.11",  virtualbox__intnet: "net1" 
 client.vm.hostname = "client" 
 end 
# config.vm.provision :ansible do |ansible|
#    ansible.limit = "all"
#    ansible.playbook = "main.yaml"
#  end
end
