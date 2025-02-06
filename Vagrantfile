# -- mode: ruby --
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
  config.vm.box = "bento/ubuntu-22.04"
  
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 1
  end

  config.vm.define "backup" do |backup|
    backup.vm.network "private_network", ip: "192.168.57.66"
    backup.vm.hostname = "backup"
    
    backup.vm.provider "virtualbox" do |vb|
      unless File.exist?('./storage/backup.vdi')
            vb.customize ['createhd', '--filename', './storage/backup.vdi', '--variant', 'Fixed', '--size', 2176]
            needsController = true
      end
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', './storage/backup.vdi']
    end

    # Provision to install sshpass
    backup.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y sshpass
    SHELL
  end

  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.57.67"
    client.vm.hostname = "client"

    # Provision to install sshpass
    client.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y sshpass
    SHELL
  end
end
