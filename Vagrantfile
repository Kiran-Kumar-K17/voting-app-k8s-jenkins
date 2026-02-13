Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/jammy64"
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # ---------------- MASTER ----------------
  config.vm.define "master" do |master|
    master.vm.hostname = "master"
    master.vm.network "private_network", ip: "192.168.56.10"
    master.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus = 2
    end
  end

  # ---------------- WORKER 1 ----------------
  config.vm.define "worker1" do |node|
    node.vm.hostname = "worker1"
    node.vm.network "private_network", ip: "192.168.56.11"
    node.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 1
    end
  end

  # ---------------- WORKER 2 ----------------
  config.vm.define "worker2" do |node|
    node.vm.hostname = "worker2"
    node.vm.network "private_network", ip: "192.168.56.12"
    node.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 1
    end
  end

  # ---------------- JENKINS ----------------
  config.vm.define "jenkins" do |jenkins|
    jenkins.vm.hostname = "jenkins"
    jenkins.vm.network "private_network", ip: "192.168.56.13"
    jenkins.vm.network "forwarded_port", guest: 8080, host: 8080
    jenkins.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus = 3
    end
  end

end
