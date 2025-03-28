# -*- mode: ruby -*-
# vi: set ft=ruby :

# Based on https://gist.github.com/flavio-fernandes/862747708512c1967c7b412f500fb56d

# Vagrantfile to launch a 3-node OVN cluster based on Ubuntu 24.04
# It uses the bento/ubuntu-24.04 box and the ovn packages from the Ubuntu repository
# The supported provider is VMware Desktop

VAGRANTFILE_API_VERSION = "2"
Vagrant.require_version ">=2.4.3"

$bootstrap_ovn = <<SCRIPT

apt-get update -y
apt-get install -y rdma-core
apt-get install -y openvswitch-common openvswitch-switch
apt-get install -y ovn-host

for n in openvswitch ovn-controller ; do
    systemctl enable --now $n
    systemctl status $n
done

apt install isc-dhcp-client -y

SCRIPT

$bootstrap_ovn_central = <<SCRIPT
apt-get update -y
apt-get install -y ovn-central
systemctl enable --now ovn-northd

SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "bento/ubuntu-24.04"
  config.vm.synced_folder ".", "/vagrant", type: "rsync"
  config.vm.provision "bootstrap_ovn", type: "shell", inline: $bootstrap_ovn

  config.vm.define "ovn0", primary: true, autostart: true do |ovn0|
    ovn0.vm.hostname = "ovn0"
    ovn0.vm.network "private_network", ip: "192.168.122.100",
                    :mac => 'decaff000064'
    ovn0.vm.provision "bootstrap_ovn_central", type: "shell", inline: $bootstrap_ovn_central
    config.vm.provider "vmware_desktop" do |v|
      v.vmx["ethernet0.virtualDev"] = "vmxnet3"
      v.vmx["ethernet1.virtualDev"] = "vmxnet3" 
    end
  end
  config.vm.define "ovn1", primary: false, autostart: true do |ovn1|
    ovn1.vm.hostname = "ovn1"
    ovn1.vm.network "private_network", ip: "192.168.122.101",
                    :mac => 'decaff000065'
    config.vm.provider "vmware_desktop" do |v|
      v.vmx["ethernet0.virtualDev"] = "vmxnet3"
      v.vmx["ethernet1.virtualDev"] = "vmxnet3" 
    end
  end
  config.vm.define "ovn2", primary: false, autostart: true do |ovn2|
    ovn2.vm.hostname = "ovn2"
    ovn2.vm.network "private_network", ip: "192.168.122.102",
                    :mac => 'decaff000066'
    config.vm.provider "vmware_desktop" do |v|
      v.vmx["ethernet0.virtualDev"] = "vmxnet3"
      v.vmx["ethernet1.virtualDev"] = "vmxnet3" 
    end
  end
end
