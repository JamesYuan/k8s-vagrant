
Vagrant.require_version ">= 1.7.0"

def set_vbox(vb, config)
  vb.gui = false
  vb.memory = 2024
  vb.cpus = 1

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
    sudo curl -fsSL https://get.docker.com/ | sh
    sudo systemctl enable docker && sudo systemctl start docker
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
    sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
    sudo sysctl -p /etc/sysctl.d/k8s.conf
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
    sudo cp /vagrant/amd64/kubelet /usr/local/bin/kubelet
    sudo chmod +x /usr/local/bin/kubelet

    sudo cp /vagrant/amd64/kubectl /usr/local/bin/kubectl
    sudo chmod +x /usr/local/bin/kubectl
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
    sudo yum -y install wget
    export CNI_URL=https://github.com/containernetworking/plugins/releases/download
    sudo mkdir -p /opt/cni/bin && cd /opt/cni/bin
    sudo wget -qO- "${CNI_URL}/v0.7.1/cni-plugins-amd64-v0.7.1.tgz" | sudo tar -zx
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
    export CFSSL_URL=https://pkg.cfssl.org/R1.2
    sudo wget ${CFSSL_URL}/cfssl_linux-amd64 -O /usr/local/bin/cfssl
    sudo wget ${CFSSL_URL}/cfssljson_linux-amd64 -O /usr/local/bin/cfssljson
    sudo chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
  SHELL

  private_count = 10
  (1..(master + node)).each do |mid|
    name = (mid <= node) ? "node" : "master"
    id   = (mid <= node) ? mid : (mid - node)

    config.vm.define "#{name}#{id}" do |n|
      n.vm.hostname = "#{name}#{id}"
      ip_addr = "172.16.35.#{private_count}"
      n.vm.network :private_network, ip: "#{ip_addr}",  auto_config: true

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
