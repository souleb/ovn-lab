# DHCP

This lab will test the dhcp support in ovn. We will create a simple topology
with a network and ports connected to it. We will then demonstrate how the
ovn can be used to assign IP addresses to the ports.

This lab is reproducing the blog post [OVN and DHCP: A minimal example](https://blog.oddbit.com/post/2019-12-19-ovn-and-dhcp/).


## Setup

Use the vgrant file to create the lab environment.


# Step 1: Connect to ovn0 and configure tcp port 6642 to listen:

```shell
$ vagrant ssh ovn0
$ ovn-sbctl set-connection ptcp:6642
```

You can verify the connection by running the following command:

```shell
$ ss -tlnp | grep 6642
```

# Step 2: Connect nodes to the ovn-controller:

On all the nodes, connect the ovn-controller service to the ovn-sb:

```shell
$ ovs-vsctl set open_vswitch .  \
  external_ids:ovn-remote=tcp:192.168.122.100:6642 \
  external_ids:ovn-encap-ip=$(ip addr show eth1 | awk '$1 == "inet" {print $2}' | cut -f1 -d/) \
  external_ids:ovn-encap-type=geneve \
  external_ids:system-id=$(hostname)
```

This will set the `ovn-remote` to the address of the controller i.e. `ovn0` node and
the `ovn-encap-ip` to the `eth1` ip address of the node. The ovn-encap-type is set to
`geneve` and the `system-id` is set to the hostname of the node.


We can verify these settings by running the following command:

```shell
$ ovs-vsctl --columns external_ids list open_vswitch
external_ids        : {hostname=ovn0, ovn-encap-ip="192.168.122.100", ovn-encap-type=geneve, ovn-remote="tcp:192.168.122.100:6642", rundir="/var/run/openvswitch", system-id=ovn0}
```

or

```shell
$ ovs-vsctl show
f1219951-ffc3-44aa-85c0-cf3aac3bd860
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port ovn-ovn2-1
            Interface ovn-ovn2-1
                type: geneve
                options: {csum="true", key=flow, remote_ip="192.168.122.102"}
        Port br-int
            Interface br-int
                type: internal
        Port ovn-ovn0-0
            Interface ovn-ovn0-0
                type: geneve
                options: {csum="true", key=flow, remote_ip="192.168.122.100"}
    ovs_version: "3.3.0"
```

It is also possible to verify by directly querying the ovn-sb:

```shell
$ ovn-sbctl show
Chassis ovn1
    hostname: ovn1
    Encap geneve
        ip: "192.168.122.101"
        options: {csum="true"}
Chassis ovn2
    hostname: ovn2
    Encap geneve
        ip: "192.168.122.102"
        options: {csum="true"}
Chassis ovn0
    hostname: ovn0
    Encap geneve
        ip: "192.168.122.100"
        options: {csum="true"}
```

# Step 3: Create the network:

Create the network with the following command:

```shell
$ ovn-nbctl ls-add net0
```

This creates a logical switch called `net0`. It can be verified by running the following command:

```shell
$ ovn-nbctl show
switch 0b625493-bb0f-4600-9700-733132fbad48 (net0)
```

Configure the switch to manage the network `10.0.0/24` and exclude the first 10 addresses:

```shell
ovn-nbctl set logical_switch net0 \
  other_config:subnet="10.0.0.0/24" \
  other_config:exclude_ips="10.0.0.1..10.0.0.10"
```

Step 4: Create the dhcp options:

Each port that we want to assign an IP address to will need to have a dhcp option set. We can
set the dhcp options for the network by running the following command:

```shell
CIDR_UUID=$(ovn-nbctl create dhcp_options \
  cidr=10.0.0.0/24 \
  options='"lease_time"="3600" "router"="10.0.0.1" "server_id"="10.0.0.1" "server_mac"="c0:ff:ee:00:00:01"')
```

The options are:
- lease_time: The time in seconds that the lease will be valid for.
- router: The IP address of the router.
- server_id: The IP address of the virtual dhcp server.
- server_mac: The MAC address of the virtual dhcp server.

We can verify the dhcp options by running the following command:

```shell
$ ovn-nbctl list dhcp_options
_uuid               : f6a557e4-a0b0-431a-a9d9-fdf5f9bd193b
cidr                : "10.0.0.0/24"
external_ids        : {}
options             : {lease_time="3600", router="10.0.0.1", server_id="10.0.0.1", server_mac="c0:ff:ee:00:00:01"}
```

# Step 4: Create the logical ports:

We will create 3 logical ports on the switch `net0`:

```shell
ovn-nbctl lsp-add net0 port1
ovn-nbctl lsp-set-addresses port1 "c0:ff:ee:00:00:11 dynamic"
ovn-nbctl lsp-set-dhcpv4-options port1 $CIDR_UUID
ovn-nbctl lsp-add net0 port2
ovn-nbctl lsp-set-addresses port2 "c0:ff:ee:00:00:12 dynamic"
ovn-nbctl lsp-set-dhcpv4-options port2 $CIDR_UUID
ovn-nbctl lsp-add net0 port3
ovn-nbctl lsp-set-addresses port3 "c0:ff:ee:00:00:13 dynamic"
ovn-nbctl lsp-set-dhcpv4-options port3 $CIDR_UUID
```

For each port, we set a static MAC address and assign the dhcp options to the port.
We use dhcpv4 options for the port. By setting `dynamic`, we effectively tell the ovn
that the port should get an IP address from the dhcp server.

We can check the network state by running the following command:

```shell
$ ovn-nbctl show
switch 0b625493-bb0f-4600-9700-733132fbad48 (net0)
    port port2
        addresses: ["c0:ff:ee:00:00:12 dynamic"]
    port port3
        addresses: ["c0:ff:ee:00:00:13 dynamic"]
    port port1
        addresses: ["c0:ff:ee:00:00:11 dynamic"]
```

And list the logical ports addresses:

```shell
root@ovn0:~# ovn-nbctl --columns dynamic_addresses list logical_switch_port
dynamic_addresses   : "c0:ff:ee:00:00:11 10.0.0.11"

dynamic_addresses   : "c0:ff:ee:00:00:13 10.0.0.13"

dynamic_addresses   : "c0:ff:ee:00:00:12 10.0.0.12"
```


# Step 5: Simulate a dhcp request:

We can simulate a dhcp request by running the following command with `ovn-trace`:

```shell
ovn-trace --summary net0 '
  inport=="port1" &&
  eth.src==c0:ff:ee:00:00:11 &&
  ip4.src==0.0.0.0 &&
  ip.ttl==1 &&
  ip4.dst==255.255.255.255 &&
  udp.src==68 &&
  udp.dst==67'

# udp,reg14=0x1,vlan_tci=0x0000,dl_src=c0:ff:ee:00:00:11,dl_dst=00:00:00:00:00:00,nw_src=0.0.0.0,nw_dst=255.255.255.255,nw_tos=0,nw_ecn=0,nw_ttl=1,nw_frag=no,tp_src=68,tp_dst=67
ingress(dp="net0", inport="port1") {
    reg0[15] = check_in_port_sec();
    next;
    reg0[3] = put_dhcp_opts(offerip = 10.0.0.11, lease_time = 3600, netmask = 255.255.255.0, router = 10.0.0.1, server_id = 10.0.0.1);
    /* We assume that this packet is DHCPDISCOVER or DHCPREQUEST. */;
    next;
    eth.dst = eth.src;
    eth.src = c0:ff:ee:00:00:01;
    ip4.src = 10.0.0.1;
    udp.src = 67;
    udp.dst = 68;
    outport = inport;
    flags.loopback = 1;
    output;
    egress(dp="net0", inport="port1", outport="port1") {
        reg0[15] = check_out_port_sec();
        next;
        output;
        /* output to "port1", type "" */;
    };
};
```

> When ovn-controller receives a DHCP request packet, in order to send a DHCP reply
it needs the following:
>- the IPv4 address to be offered
>- the DHCP options to be added in the DHCP reply packet.
>
>ovn-northd then adds logical flows to send the DHCP replies for each logical port
>which has an IPv4 address and DHCP options defined.

See [native-dhcp-support-in-ovn](https://numans.blog/2016/08/09/native-dhcp-support-in-ovn/) for more details.


# Step 6: Create ports and attach network interfaces:

For each node, create a port with ovs:

**ovn0**:
```shell
ovs-vsctl add-port br-int port1 -- \
  set interface port1 \
    type=internal \
    mac='["c0:ff:ee:00:00:11"]' \
    external_ids:iface-id=port1
```

**ovn1**:
```shell
ovs-vsctl add-port br-int port2 -- \
  set interface port2 \
    type=internal \
    mac='["c0:ff:ee:00:00:12"]' \
    external_ids:iface-id=port2
```

**ovn2**:
```shell
ovs-vsctl add-port br-int port3 -- \
  set interface port3 \
    type=internal \
    mac='["c0:ff:ee:00:00:13"]' \
    external_ids:iface-id=port3
```

In order for ovn to detect the ports, the mac address and the iface-id must be set
and match the logical ports created in the previous step.

This can be verified by running the following command:

```shell
$ ovn-sbctl show
Chassis ovn1
    hostname: ovn1
    Encap geneve
        ip: "192.168.122.101"
        options: {csum="true"}
    Port_Binding port2
Chassis ovn2
    hostname: ovn2
    Encap geneve
        ip: "192.168.122.102"
        options: {csum="true"}
    Port_Binding port3
Chassis ovn0
    hostname: ovn0
    Encap geneve
        ip: "192.168.122.100"
        options: {csum="true"}
    Port_Binding port1
```

Let's now confiqure the interfaces with dhcp.

Create a namespace and move the port to the namespace:

**ovn1**:
```shell
$ ip netns add vm1
$ ip link set netns vm1 port1
$ ip -n vm1 addr add 127.0.0.1/8 dev lo
$ ip -n vm1 link set lo up
```

Configure the port to get an IP address from the dhcp server:

```shell
$ ip netns exec vm1 dhclient -v -i port1 --no-pid
Internet Systems Consortium DHCP Client 4.4.3-P1
Copyright 2004-2022 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/port1/c0:ff:ee:00:00:11
Sending on   LPF/port1/c0:ff:ee:00:00:11
Sending on   Socket/fallback
xid: warning: no netdev with useable HWADDR found for seed's uniqueness enforcement
xid: rand init seed (0x6799cf31) built using gethostid
Created duid "\000\001\000\001/y\212\263\300\377\356\000\000\021".
DHCPDISCOVER on port1 to 255.255.255.255 port 67 interval 3 (xid=0xb80b204e)
DHCPOFFER of 10.0.0.11 from 10.0.0.1
DHCPREQUEST for 10.0.0.11 on port1 to 255.255.255.255 port 67 (xid=0x4e200bb8)
DHCPACK of 10.0.0.11 from 10.0.0.1 (xid=0xb80b204e)
bound to 10.0.0.11 -- renewal in 1613 seconds.
```

We can verify the IP address by running the following command:

```shell
$ ip netns exec vm1 ip addr show port1
9: port1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether c0:ff:ee:00:00:11 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.11/24 brd 10.0.0.255 scope global dynamic port1
       valid_lft 3501sec preferred_lft 3501sec
    inet6 fe80::c2ff:eeff:fe00:11/64 scope link
       valid_lft forever preferred_lft forever
```

After configuring the interface on all 3 nodes, we can verify the network state
by running the following command:

```shell
$ ip netns exec vm1 ping -c1 10.0.0.13
PING 10.0.0.13 (10.0.0.13) 56(84) bytes of data.
64 bytes from 10.0.0.13: icmp_seq=1 ttl=64 time=2.26 ms

--- 10.0.0.13 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.256/2.256/2.256/0.000 ms
```
