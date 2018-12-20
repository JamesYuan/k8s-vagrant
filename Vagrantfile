
Vagrant.require_version ">= 1.7.0"

def set_vbox(vb, config)
  vb.gui = false
  vb.memory = 3072
  vb.cpus = 2

  case $os_image
  when :centos7
    config.vm.box = "centos/7"
  when :ubuntu16
    config.vm.box = "bento/ubuntu-16.04"
  end
end

def set_libvirt(lv, config)
  lv.nested = true
  lv.volume_cache = 'none'
  lv.uri = 'qemu+unix:///system'
  lv.memory = $system_memory
  lv.cpus = $system_vcpus

  case $os_image
  when :centos7
    config.vm.box = "centos/7"
  when :ubuntu16
    config.vm.box = "yk0/ubuntu-xenial"
  end
end

$os_image = (ENV['OS_IMAGE'] || "centos7").to_sym

Vagrant.configure("2") do |config|
  config.vm.provider "libvirt"
  config.vm.provider "virtualbox"

  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

  master = 1
  node = 2

  config.vm.provision "shell", inline: <<-SHELL
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
  SHELL

  config.vm.provision "shell", inline: "sudo swapoff -a && sudo sysctl -w vm.swappiness=0"

  config.vm.provision "shell", inline: <<-SHELL
    sudo echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
    sudo sysctl -p
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
   sudo yum install -y epel-release python-pip python34 python34-pip
   sudo yum install -y ansible
   sudo pip install netaddr
   sudo pip install --upgrade jinja2
  SHELL

  private_count = 10
  (1..(master + node)).each do |mid|
    name = (mid <= node) ? "node" : "master"
    id   = (mid <= node) ? mid : (mid - node)

    config.vm.define "#{name}#{id}" do |n|
      n.vm.hostname = "#{name}#{id}"
      ip_addr = "172.16.35.#{private_count}"
      n.vm.network :private_network, ip: "#{ip_addr}",  auto_config: true
      n.vm.network "forwarded_port", guest: 6443, host: "64#{private_count}"
      n.vm.network "forwarded_port", guest: 31390, host: "313#{private_count}"

      n.vm.provider :virtualbox do |vb, override|
        vb.name = "kube-#{n.vm.hostname}"
        set_vbox(vb, override)
      end

      n.vm.provider :libvirt do |lv, override|
        lv.host = "kube-#{n.vm.hostname}"
        set_libvirt(lv, override)
      end
      private_count += 1
    end
  end
end
