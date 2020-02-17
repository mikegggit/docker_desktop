Docker Desktop 
==================
A general design and architectural notes regarding Docker Desktop architecture and networking.  

Goal
----
Document my understanding of various docker desktop aspects suitable for future reference. 

Bring together and summarize material likely better stated elsewhere simply for reference and context to the goal of my own understanding.


Scope
-----
This document is not intended to be thorough, rather simply as a helpful overview and thumbnail of commands and procedures for understanding how a model installation works under the covers, at least aspects of it. 

What focus there is on networking is limited to a specific network setup and isn't exhaustive in discussing every network type or every aspect of, for example, bridge networks.  In the future, I might discuss the other networks (host, overlay).

I intend to follow up on this with a separate repository focussed exclusively on troubleshooting and SOP's.


Background
----------
Generally speaking, Docker Desktop refers to an installation including a variety of parts.  

The most obvious parts are arguably...
 - hyperkit
 - vpnkit
 - dockerd and associated processes running within the virtualized environment
 - networking

Both hyperkit and vpnkit run on the mac host as services.


Hyperkit
--------
Hyperkit is an alternative virtualization infrastructure to VirtualBox.

Docker runs within hyperkit.


VPNKit
------
VPNKit constructs native socket API calls from docker daemon ethernet traffic.  

It avoids the need to perform network bridging on the host and manipulate host routing tables.

Docker manages all the networking required to deliver packets from within the VM to vpnkit.  All connections to the outside works are done by vpnkit.  In theory, this makes for a virtualization solution which is doesn't depend on any host rules or setup to establish and maintain the vm -> vpnkit connection.


Accessing the virtualized environment
-------------------------------------
dockerd runs within the hyperkit environment. 


To open a terminal into the virtualized environment:
```
screen ~/Library/Containers/com.docker.docker/Data/vms/0/tty
```

Dockerd can be seen running here...
```
docker-desktop:~# ps -ef
...
/usr/bin/vpnkit-bridge --addr connect://2/1999 guest
 1017 root      0:00 /usr/local/bin/docker-init /usr/bin/entrypoint.sh
 1042 root      0:00 {entrypoint.sh} /bin/sh /usr/bin/entrypoint.sh
 1373 root      0:00 /vpnkit-forwarder -data-connect /run/host-services/vpnkit-d
ata.sock
 1406 root      0:00 {start-docker.sh} /bin/sh -x /usr/bin/start-docker.sh /run/
config/docker/daemon.json
 1411 root      0:21 /usr/local/bin/dockerd -H unix:///var/run/docker.sock -H unix:///run/guest-services/docker.sock --pidfile /run/desktop/docker.pid --config-file /run/config/docker/daemon.json --swarm-default-advertise-addr=eth0
 1422 root      0:18 containerd --config /var/run/docker/containerd/containerd.t
oml --log-level debug
...
```

Dockerd is the daemon.  It is also referred to as docker engine and is the server aspect of docker.

The process is configured to listen to various sockets for communicating with clients, in other words, listening for engine API requests.

The configuration it uses to startup maps directly to the Docker for Mac options.

```
docker-desktop:~# cat /run/config/docker/daemon.json
{"debug":true,"experimental":false}
docker-desktop:~#
```


You can see the listening sockets here:
```
docker-desktop:/tmp# netstat -l|grep docker
unix  2      [ ACC ]     STREAM     LISTENING      17666 /var/run/docker/metrics.sock
unix  2      [ ACC ]     STREAM     LISTENING      16833 /var/run/docker/libnetwork/678f8de01ebff230fdcfa60d06c6bb2b7b5790cf2b7af3d007c6859c846e5331.sock
unix  2      [ ACC ]     STREAM     LISTENING      17614 /var/run/docker.sock
unix  2      [ ACC ]     STREAM     LISTENING      17618 /run/guest-services/docker.sock
unix  2      [ ACC ]     STREAM     LISTENING      14769 @/containerd-shim/services.linuxkit/docker/shim.sock@
unix  2      [ ACC ]     STREAM     LISTENING      17654 /var/run/docker/containerd/containerd-debug.sock
unix  2      [ ACC ]     STREAM     LISTENING      17658 /var/run/docker/containerd/containerd.sock
```

Notice especially /var/run/docker.sock, the socket over which containers communicate w/ the daemon.


All commands available via the docker API can be invoked using curl against /var/run/docker.sock.  Normally, interaction w/ the daemon will be via a higher-level client, however it is possible to demonstrate the low-level mechanics and API using something like curl.


Networking
----------
With no other containers running, you can see the initial setup created by docker on the daemon vm.

```
docker-desktop:~# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:58:64:A1:03
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:58ff:fe64:a103/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:15 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:1226 (1.1 KiB)

eth0      Link encap:Ethernet  HWaddr 02:50:00:00:00:01
          inet addr:192.168.65.3  Bcast:192.168.65.255  Mask:255.255.255.0
          inet6 addr: fe80::50:ff:fe00:1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:5169 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2510 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:6434686 (6.1 MiB)  TX bytes:192856 (188.3 KiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:2 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:140 (140.0 B)  TX bytes:140 (140.0 B)
```

Docker has a variety of network options available for establishing how containers communicate with each other and across networks.  Docker associates each different network type with networking rules that enable the connectivity expected of each network type.  By default, docker creates a bridge network on the daemon host (indicated by the docker0 interface above).  Any container running on the same network (and same host) can communicate with each other with this network mode.  Additional details are beyond the scope of this document.

There are three addressible networks setup on the docker host.  

Docker assigns 192.168.65.1 as the nameserver for DNS lookups (port 53).  In reality, vpnkit does some magic to route traffic targeting 192.168.65.1 to the host gateway, which is in this case 192.168.1.1 (FIOS router) on my LAN.

```
docker-desktop:/tmp# cat /etc/resolv.conf
# This file is included on the metadata iso
nameserver 192.168.65.1
```

This is almost exactly the same as my host dns config except that 192.168.1.1 is replaced here by 192.168.65.1.

External routing
----------------
Routing from the daemon to the internet if managed up to the point of the vpnkit implementation by 
```
docker-desktop:/tmp# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.65.1    0.0.0.0         UG    0      0        0 eth0
127.0.0.0       *               255.0.0.0       U     0      0        0 lo
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
192.168.65.0    *               255.255.255.0   U     0      0        0 eth0
```

The locally hosted ip's:
```
docker-desktop:/tmp# ip route show table local type local
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
local 172.17.0.1 dev docker0 proto kernel scope host src 172.17.0.1
local 192.168.65.3 dev eth0 proto kernel scope host src 192.168.65.3
```

192.168.65.1 is typically given to a router.  Given this is a virtual network and not mapped to a physical device, connectivity to the external network requires some vpnkit magic not discussed here.


```
docker-desktop:/tmp# iptables -t nat -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere            !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        anywhere

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
```

Packets not targeting the loopback interface are modified to have their source ip's changed to the value of the routing device, in this case 192.168.65.3.  TODO: Confirm this isn't actually the ip address of the physical gateway router on the host network (192.168.1.1), or perhaps the vpnkit gateway ip (192.168.65.1).



Show the bridge network
```
docker-desktop:~# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02425864a103       no              

docker-desktop:~# ifconfig docker0
docker0   Link encap:Ethernet  HWaddr 02:42:58:64:A1:03
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:58ff:fe64:a103/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:15 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:1226 (1.1 KiB)



```
On the host, you can see the default bridge network:
```
Michaels-Air:spark grudkowm$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
c9182ea8f9c2        bridge              bridge              local
879bb207aa19        host                host                local
34b10956512b        none                null                local
Michaels-Air:spark grudkowm$

Michaels-Air:spark grudkowm$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
264f956fe306        bash                "docker-entrypoint.s…"   2 hours ago         Up 2 hours                              epic_yonath

Michaels-Air:spark grudkowm$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "c9182ea8f9c2bfcc767740acbc5468caa7806560bc97e41f468075810bb518fc",
        "Created": "2020-02-16T19:38:28.780551656Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
                    "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]

docker-desktop:/tmp# ip addr show docker0
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:58:64:a1:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:58ff:fe64:a103/64 scope link
       valid_lft forever preferred_lft forever

```

Above, docker host has been assigned 172.17.0.1 on the docker0 interface. 


Routing example involving 2 containers
--------------------------------------
Let's start two simple bash containers on the bridge network:

```
Michaels-Air:spark grudkowm$ docker container run  -dit -p 9000:9000 --privileged -v /var/run/docker.sock:/var/run/docker.sock bash
6926692714c9c45ede8563807e23bb0bbb8cd4338b2d3d9a6e3262f3db3c39c3

Michaels-Air:spark grudkowm$ docker container run  -dit -p 9001:9001 --privileged -v /var/run/docker.sock:/var/run/docker.sock bash
95b799d4b84f0dffbfafc3bfe56f2eb09bd2f4bfed1919b8455e71909678c6e3

Michaels-Air:spark grudkowm$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
95b799d4b84f        bash                "docker-entrypoint.s…"   13 minutes ago      Up 13 minutes       0.0.0.0:9001->9001/tcp   silly_ishizaka
6926692714c9        bash                "docker-entrypoint.s…"   14 minutes ago      Up 14 minutes       0.0.0.0:9000->9000/tcp   frosty_curran
```

Notice two virtual interfaces have been created on the bridge network, and the new containers have been assigned ip's in the docker0 network.

```
docker-desktop:/tmp# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02425864a103       no              veth35fd58e
                                                        vethaebefa2

Michaels-Air:spark grudkowm$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "c9182ea8f9c2bfcc767740acbc5468caa7806560bc97e41f468075810bb518fc",
        "Created": "2020-02-16T19:38:28.780551656Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "6926692714c9c45ede8563807e23bb0bbb8cd4338b2d3d9a6e3262f3db3c39c3": {
                "Name": "frosty_curran",
                "EndpointID": "d2d81cdfdc870587d3874477ee021bf6b1761fb6f1bff497320e5451168e60c5",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "95b799d4b84f0dffbfafc3bfe56f2eb09bd2f4bfed1919b8455e71909678c6e3": {
                "Name": "silly_ishizaka",
                "EndpointID": "3b3025e77738c08baac647c1636116d453f2bf9c3b138ab3a445f00fca311991",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

Each container on the network is directly accessible.  Connectivity to the internet is via the gateway.

```
Michaels-Air:spark grudkowm$ docker exec -it 95b799d4b84f bash
bash-5.0# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1000
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
32: eth0@if33: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

bash-5.0# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      *               255.255.0.0     U     0      0        0 eth0    
```

This shows that the default gateway has been setup as the ip associated w/ the docker0 interface on the network.  Any traffic targeting 172.17.*.* is directly addressible, whereas anything targeting another network will be sent to the default gateway @ 172.17.0.1.

When each container is started, its /etc/hosts file is updated with an entry mapping it's container name to it's assigned ip.  In custom bridge networks, the file is updated with each container on the network as it's added / removed, allowing inter container connectivity by name.

```
bash-5.0# cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.3	95b799d4b84f
```

The container on 172.17.0.3 can connect to 172.17.0.2:

```
bash-5.0# hostname
95b799d4b84f
bash-5.0# ping `hostname`
PING 95b799d4b84f (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.225 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.073 ms
^C
--- 95b799d4b84f ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.073/0.149/0.225 ms
bash-5.0# ping -c 2 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.119 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.147 ms

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.119/0.133/0.147 ms
```

It can also ping the vpnkit gateway:

```
Michaels-Air:spark grudkowm$ docker exec -it 95b799d4b84f ping 172.17.0.1
PING 172.17.0.1 (172.17.0.1): 56 data bytes
64 bytes from arp: seq=0 ttl=64 time=0.090 ms
64 bytes from 172.17.0.1: seq=1 ttl=64 time=0.105 ms
^C
```
Traceroute from a container to www.google.com...

```
bash-5.0# ping google.com
PING google.com (172.217.10.142): 56 data bytes
64 bytes from 172.217.10.142: seq=0 ttl=37 time=15.418 ms
64 bytes from 172.217.10.142: seq=1 ttl=37 time=20.539 ms
64 bytes from 172.217.10.142: seq=2 ttl=37 time=12.642 ms
^C
--- google.com ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 12.642/16.199/20.539 ms
bash-5.0# traceroute google.com
traceroute to google.com (172.217.10.142), 30 hops max, 46 byte packets
 1  172.17.0.1 (172.17.0.1)  0.072 ms  0.105 ms  0.051 ms
 2  Fios_Quantum_Gateway.fios-router.home (192.168.1.1)  5.399 ms  2.534 ms  2.236 ms
 3  *  *  *
 4  ae1385-21.PHLAPALO-MSE01-AA-IE1.verizon-gni.net (100.41.135.78)  28.200 ms  18.174 ms  6.457 ms
 5  0.et-9-1-2.GW15.NYC1.ALTER.NET (140.222.227.25)  13.668 ms  0.et-10-0-5.GW15.NYC1.ALTER.NET (140.222.1.83)  11.465 ms  15.691 ms
 6  72.14.208.130 (72.14.208.130)  9.979 ms  18.809 ms  17.371 ms
 7  *  *  *
 8  108.170.248.1 (108.170.248.1)  26.446 ms  209.85.253.188 (209.85.253.188)  57.505 ms  108.170.248.1 (108.170.248.1)  23.115 ms
 9  108.170.248.20 (108.170.248.20)  17.839 ms  lga34s16-in-f14.1e100.net (172.217.10.142)  19.021 ms  108.170.248.66 (108.170.248.66)  21.434 ms
```

In the above demonstration, the docker0 gateway ip can be seen as the first hop on the journey to google.com.  


Reference
=========
http://collabnix.com/how-docker-for-mac-works-under-the-hood/
https://developer.ibm.com/recipes/tutorials/networking-your-docker-containers-using-docker0-bridge/
https://docs.docker.com/engine/reference/commandline/dockerd/
https://stackoverflow.com/questions/24319662/from-inside-of-a-docker-container-how-do-i-connect-to-the-localhost-of-the-machine
http://jedelman.com/home/docker-networking/
https://www.techrepublic.com/article/understand-the-basics-of-linux-routing/
https://www.karlrupp.net/en/computer/nat_tutorial
https://docs.docker.com/network/iptables/
https://stackoverflow.com/questions/26963362/dockers-nat-table-output-chain-rule
https://github.com/moby/vpnkit/blob/master/docs/ethernet.md
https://docs.docker.com/network/bridge/