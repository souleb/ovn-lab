# OVN Lab

OVN Lab is a simple lab environment for testing OVN. It is based on Vagrant and
Vmware Fusion. It is a simple way to test OVN without having to install it on
your own machine.

## Requirements

- Vagrant
- Vmware Fusion
- Vagrant VMWare plugin

## Usage

To start the lab, simply run `vagrant up`. This will start 3 VMs: ovn0, ovn1 and
ovn2. ovn0 is the OVN controller, ovn1 and ovn2 are ovn compute nodes.

Once the VMs are up, you can ssh into ovn0 and start playing with OVN.

```
$ vagrant ssh ovn0
```

## labs

The lab directory contains a few labs that you can use to test OVN. Each lab
is a simple shell script that sets up a simple network topology and tests it.