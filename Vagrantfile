Vagrant.configure("2") do |config|
    # Define VM for master
    config.vm.define "master" do |master|
      master.vm.box = "bento/rockylinux-8"
  
      # Set Host-Only Network
      master.vm.network "private_network", type: "dhcp"
  
      # Set NAT for internet access
      master.vm.network "public_network", type: "dhcp"
  
      master.vm.provider "virtualbox" do |vb|
        vb.memory = "2048"
        vb.cpus = 2
      end
  
      master.vm.hostname = "master"
  
    end
  
    # Define VM for node01
    config.vm.define "node01" do |node01|
      node01.vm.box = "bento/rockylinux-8"
  
      # Set Host-Only Network
      node01.vm.network "private_network", type: "dhcp"
  
      # Set NAT for internet access
      node01.vm.network "public_network", type: "dhcp"
  
      node01.vm.provider "virtualbox" do |vb|
        vb.memory = "2048"
        vb.cpus = 2
      end
  
      node01.vm.hostname = "node01"
  
    end
  
    # Define VM for node02
    config.vm.define "node02" do |node02|
      node02.vm.box = "bento/rockylinux-8"
  
      # Set Host-Only Network
      node02.vm.network "private_network", type: "dhcp"
  
      # Set NAT for internet access
      node02.vm.network "public_network", type: "dhcp"
  
      node02.vm.provider "virtualbox" do |vb|
        vb.memory = "2048"
        vb.cpus = 2
      end
  
      node02.vm.hostname = "node02"
  
    end
  end