# Task - Routing Implementation

The goal of this task is to implement a secure controlled network through making one node a router and other nodes connect and communicate via the router node. The tasks are

- Configure the physical network between three nodes
  - The router will have 2 interface for each nodes
  - Each node will be connected directly to the routing node
- Configure the router to have a static ip on each interface and enable packet forwarding
- Configure each node to use the router as default gateway to communicate with each other
  
## 1. Setup the physical network

Vmware allows creation of internal network by implementing line segments. On new/modifying network adapter, simply select `Lan Segments` and select a segment which represents a wired Ethernet network. New segments can be added from `LAN Segments...` button below.

- Create two interfaces in routing node. Each node will select different LAN segments. `Internal_A` and `Internal_B`
- On each node, modify the existing interface from NAT to LAN segments. One should be connected to `Internal_A` and other one to `Internal_B`

## 2. Configure routing node

To make a linux os act as router, first we need to acquire an IP on each connection. We are to do that manually. Setting each to `192.168.10.1` and `192.168.20.1`

```bash
# List interfaces
ip -br a 

# Setup manual IP on first interface
nmcli con mod ens160 ipv4.addresses 192.168.10.1/24 ipv4.method manual

# Setup manual IP on second interface
nmcli con mod ens224 ipv4.addresses 192.168.20.1/24 ipv4.method manual
```

Verify the network changes by running `ip a` and checking IPv4 addresses

Next we need to enable IP_FORWARD in kernel, otherwise Linux kernel will drop all packages that is not sent for it self.

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

To make it permanent, add a new file in sysctl `/etc/sysctl.d/40-routing.conf`

```bash
net.ipv4.ip_forward = 1
```

## 3. Modify each node to use the routing node

First we need to figure out which interface in routing table is connected to which node. SInce nodes do not have IP, we cannot ping. Instead we use `arping`

```bash
arping 192.168.10.1
arping 192.168.20.1
```

One or both of these will return a mac address. You can match it with result of `ip a` in routing node to understand which interface is connected to which node.

Since nodes are alpine-linux, we need to configure using `networking` services which has configuration file at `/etc/networking/interfaces` and edit the line containing the interface to following

Default configuration

```bash
auto lo
iface lo inet loopback

auto eth0 inet dhcp
```

Modified static configuration

```bash
auto lo
iface lo inet loopback

auto eth0 inet static
    address 192.168.10.15 # Randomly chosen ip from subnet
    netmask 255.255.255.0
    gateway 192.168.10.1
```

And restart the service with

```bash
rc-service networking restart
```

Now we should be able to ping to the router IP or IP of the 2nd node. If not, check the following

- Make sure you are connecting to proper gateway and using proper subnet

- Make sure the configuration are complete. That there's proper IP with proper sub-net number for all nodes including the router and proper gateway for the nodes.

- Check ping with firewall temporarily disabled.

## 4. Configure the firewall

Now we need to configure the firewall so that each device can talk to other but cannot take advantage of router access.

For simpler configuration, we include the interfaces to a zone, here internal zone and allow communication of each other

```bash
# Add each interface to the zone
firewalld-cmd --permanent --zone=internal --add-interface=ens160
firewalld-cmd --permanent --zone=internal --add-interface=ens224

# Allow communication between each other
firewalld-cmd --permanent --zone=internal --add-forward

# Reload the firewall service
firewalld-cmd --reload
```

We can check out configuration by this command

```bash
firewalld-cmd --list-all --zone=internal
```

You will notice there are services allowed in the zone such as `ssh` , `cockpit`. To deny any modification access to router, we need to remove these services from allowed list.

To remove `ssh` for example

```bash
firewalld-cmd --permanent --zone=internal --remove-service=ssh
```

Now you should be able to ping each other and see the populated IP tables from the routing node
