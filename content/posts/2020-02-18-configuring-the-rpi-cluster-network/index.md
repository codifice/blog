---
title: "RPi Cluster (Part 3) - Networking"
date: 2020-02-18T11:27:56Z
draft: false
authors: ["martincjarvis"]
comments: true
thumbnail: "/networking-gallery"
series: ["rpi"]
---
# Configuring the RPi Cluster Network

Now the that the cluster is configured and all the RPi's are up and running, it's time to sort out the networking so that the cluster operates on its own, predictable network and external connectivity is through a single Gateway.

{{< imgproc simple-network Resize "650x472" >}}
Simple Network Diagram
{{< / imgproc>}}

We're going to configure the gateway RPi (which is doing double duty as the cache RPi in my case) to be a DHCP server using `dnsmasq` for all the devices connected to the 5 Port Switch which is serving as our backplane.  We'll also need a second ethernet port to be our "hotplug" network port which will get it's IP Address assigned by the host networks DHCP server, and similarly with the RPi's built-in WIFI.  All the other RPi's on the cluster will have their WiFi interfaces disabled.

## Steps to Configure

### On the Gateway RPi

Install the dnsmasq software, but stop the service until we've configured it correctly:

```bash
sudo apt install dnsmasq
sudo systemctl stop dnsmasq
```

Assign a static IP address (`10.0.1.1`) to `eth0` (the onboard ethernet port - connected to the 5 port switch) by adding the following to the end of `/etc/dhcpcd.conf` (`sudo nano /etc/dhcpcd.conf`):

```plain
interface eth0
        static ip_address=10.0.1.1/24
```

Now we need to configure the DHCP server for the cluster, first backup the default config and create a new one:

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```

Add the following config for `eth0` to assign IP's using the `10.0.1.100` - `10.0.1.200` pool:

```plain
# global options
domain-needed
bogus-priv
filterwin2k
expand-hosts
domain=cluster.lan
local=/cluster.lan/
listen-address=127.0.0.1
listen-address=10.0.1.1


interface=eth0
dhcp-range=10.0.1.100,10.0.1.200,255.255.255.0,24h
dhcp-option=option:router,10.0.1.1

dhcp-host=master,10.0.1.2
```

> The last line will assign a static IP of 10.0.1.2 to any host named `master` 

We also need to update the RPi `/etc/hosts` to use the static IP address for the cluster/gateway/cache RPis

`sudo nano /etc/hosts`

```plain

# replace 127.0.1.1 with 10.0.1.1 (static IP for `cache` RPi) 

10.0.1.1        cluster cache gateway
```

Finally restart the dnsmasq service with `sudo systemctl start dnsmasq`

You should be able to see the RPi's of the cluster get assigned their IP's by using:
`tail -f /var/lib/misc/dnsmasq.leases`:

```plain
1582215976 dc:a6:32:66:e4:6c 10.0.1.107 node3 01:dc:a6:32:66:e4:6c
tail: /var/lib/misc/dnsmasq.leases: file truncated
1582215987 dc:a6:32:66:e4:b7 10.0.1.119 node1 01:dc:a6:32:66:e4:b7
1582215984 dc:a6:32:66:e4:7c 10.0.1.2 master 01:dc:a6:32:66:e4:7c
1582215976 dc:a6:32:66:e4:6c 10.0.1.107 node3 01:dc:a6:32:66:e4:6c
tail: /var/lib/misc/dnsmasq.leases: file truncated
1582215991 dc:a6:32:66:e4:2d 10.0.1.108 node2 01:dc:a6:32:66:e4:2d
1582215987 dc:a6:32:66:e4:b7 10.0.1.119 node1 01:dc:a6:32:66:e4:b7
1582215984 dc:a6:32:66:e4:7c 10.0.1.2 master 01:dc:a6:32:66:e4:7c
1582215976 dc:a6:32:66:e4:6c 10.0.1.107 node3 01:dc:a6:32:66:e4:6c
```

You should also be able to ping all the nodes from the gateway/cache RPi:

```bash
pi@cluster:~ $ ping node1
PING node1 (10.0.1.119) 56(84) bytes of data.
64 bytes from node1 (10.0.1.119): icmp_seq=1 ttl=64 time=0.264 ms
64 bytes from node1 (10.0.1.119): icmp_seq=2 ttl=64 time=0.228 ms
64 bytes from node1 (10.0.1.119): icmp_seq=3 ttl=64 time=0.213 ms
64 bytes from node1 (10.0.1.119): icmp_seq=4 ttl=64 time=0.215 ms
```

Correspondingly you can ping the gateway/cache RPi from each of the cluster nodes:

```bashpi@master:~ $ ping cache
PING cache (10.0.1.1) 56(84) bytes of data.
64 bytes from cluster.cluster.local (10.0.1.1): icmp_seq=1 ttl=64 time=0.220 ms
64 bytes from cluster.cluster.local (10.0.1.1): icmp_seq=2 ttl=64 time=0.212 ms
64 bytes from cluster.cluster.local (10.0.1.1): icmp_seq=3 ttl=64 time=0.208 ms
64 bytes from cluster.cluster.local (10.0.1.1): icmp_seq=4 ttl=64 time=0.208 ms

--- cache ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 96ms
rtt min/avg/max/mdev = 0.208/0.212/0.220/0.005 ms
```

### Configure the gateway to allow outwards communication

Back on the the gateway RPi to activate the IP Forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Add a masquarade rules for `eth1` and `wlan0` and then save the rules:

```bash
sudo iptables -t nat -A  POSTROUTING -o eth1 -j MASQUERADE
sudo iptables -t nat -A  POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -j ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

To restore these settings on reboot, we need to enable IP forwarding, edit `/etc/sysctl.conf` witht `sudo nano /etc/sysctl.conf`:

```plain
net.ipv4.ip_forward=1
```

and edit `/etc/rc.local` with `sudo nano /etc/rc.local` and add the line below above "exit 0" to install reload the routing rules on boot:

```plain
iptables-restore < /etc/iptables.ipv4.nat
```

At this point, we should remove any static IP addresses assigned in `/etc/hosts` on any of the non-gateway nodes as name resolution should all be done via the dnsmasq DNS server which is configured during DHCP.

You should now be able to SSH onto any of the RPi's and ping an existing domain/IP (such as `ping www.google.com` or `ping 8.8.8.8`) and get a ping response back via the gateway.

## Disable WIFI on cluster RPis and SSH via Jumpbox

If you SSH onto one of the cluster RPi's (master, node1-3) and run `ifconfig` you'll see three adapters:

* `eth0` - wired ethernet
* `lo` - loop back (essentially 127.0.0.1)
* `wlan0` - Wifi

```bash
pi@node1:~ $ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.1.119  netmask 255.255.255.0  broadcast 10.0.1.255
        inet6 fe80::bb69:a323:deeb:d359  prefixlen 64  scopeid 0x20<link>
        ether dc:a6:32:66:e4:b7  txqueuelen 1000  (Ethernet)
        RX packets 1018  bytes 301535 (294.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 369  bytes 81865 (79.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.16  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::eec4:4889:689d:b863  prefixlen 64  scopeid 0x20<link>
        ether dc:a6:32:66:e4:b8  txqueuelen 1000  (Ethernet)
        RX packets 42480  bytes 6225458 (5.9 MiB)
        RX errors 0  dropped 1  overruns 0  frame 0
        TX packets 535  bytes 71124 (69.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

We need to disable the WiFi adapter to isolate the cluster RPi's...but if we do that we can no longer SSH onto the box directly.  

So before we disable the adapter we will need to SSH into the node via the gateway RPi as a jumpbox:

```bash
ssh -J pi@<gateway-ip> pi@master
```

> For Windows users, try using `Windows Subsystem for Linux` (WSL) to use linux SSH tools.  However, if for `putty` instructions see [putty/plink config](https://jamesd3142.wordpress.com/2018/02/05/jump-box-config-for-putty/)

You will need to enter the SSH Passwords first for the Jumpbox, and then for `master`.

If you a WSL/Linux/Mac you can configure SSH to automatically use the jumpbox so that you don't need to enter the `-J pi@<gateway-ip>` each time.  Edit your local `~/.ssh/config` file and add entries similar to:

```plain
Host master
User          pi
HostName      master
ProxyCommand  ssh pi@<gateway-ip> nc %h %p 2> /dev/null

Host node1
User          pi
HostName      node1
ProxyCommand  ssh pi@<gateway-ip> nc %h %p 2> /dev/null

Host node2
User          pi
HostName      node2
ProxyCommand  ssh pi@<gateway-ip> nc %h %p 2> /dev/null

Host node3
User          pi
HostName      node3
ProxyCommand  ssh pi@<gateway-ip> nc %h %p 2> /dev/null
```

This will allow you to `ssh` / `scp` onto the boxes behind the jumpbox transparently using `ssh pi@master`

So...to disable the `wlan0` adapter:

```bash
rfkill block wifi
```

> You can unblock wifi again with `rfkill unblock wifi`

Repeating the `ifconfig` command will show that the `wlan0` adapter has been disabled and no longer appears on the list.

To double-check connectivity try `ping www.google.com`

## Shutting down the cluster cleanly

Up until now, we've had to shut down the Raspberry PI cluster by logging onto each machine and running `sudo shutdown -h now` or `sudo poweroff`.  If we configure [SSH Key Authentication](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server) with the private key on the cache/gateway RPi and install the public key on each of the other nodes (in `~/.ssh/authorized_keys`) we can parse the DHCP Leases file and shutdown each node in turn and finally the cache/gateway RPi.

```bash
#!/bin/bash

while IFS="" read -r p || [ -n "$p" ]
do
  IFS=' ' read -r -a array <<< "$p"
  echo "Shutting down ${array[3]}"
  ssh -n pi@${array[3]} 'sudo poweroff' >> /dev/null

done < /var/lib/misc/dnsmasq.leases

sudo poweroff
```

After the nodes have safely shutdown (the flashing green LED next to the red power LED stays dark) the power can safely be turned off.

```bash
pi@cluster:~ $ ./shutdown_cluster.sh
Shutting down node2
Connection to node2 closed by remote host.
Shutting down node1
Connection to node1 closed by remote host.
Shutting down master
Connection to master closed by remote host.
Shutting down node3
Connection to node3 closed by remote host.
```

## Summary

We've now configured the RPi cluster network to isolate the RPi's behind a single gateway and enable the cluster-as-an appliance network isolation.  We can add the cluster to a network by either joining the gateway RPi to the host WIFI network or by patching in via the `eth1` port and DHCP on the host network will assign the cluster an IP address which can be used to SSH into the gateway RPi or use it as a jump box to the internal RPi's.
