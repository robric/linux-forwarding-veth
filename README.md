# linux-forwarding-veth
Investigation on linux native forwarding for pod routing.

## l3-only forwarding 

Model networking as a collection of point-to-point links and /32 routing.
All routing between pods is based on L3 forwarding even when in a same subnet. This is equivalent to the contrial l3-only forwarding mode.

## Test environment

Linux ubuntu 20.04.1 LTS.

## Test 1 - local pod networking

### Diagream

The topology is the following
- define two namespaces (these ns plays the role of pods): ns1 and ns2
- built veth to connect these namespaces to the default network namespace
- Configure interfaces in same IP network (172.30.30.0/24), with a unique gateway 172.30.30.1/32 and different pod IPs (.11 for ns1 and .12 for ns2).

![image](https://user-images.githubusercontent.com/21667569/133264432-8ec98d8e-6b2e-47d4-b83e-cd8a5a8655fa.png)

By default in such configuration the connectivity between ns is not granted because the Layer 2 domain is not extended between ns.

```
ip netns add ns1
ip netns add ns2
ip link add veth-ns1 type veth peer name veth-ns1-pod
ip link add veth-ns2 type veth peer name veth-ns2-pod
ip link set up dev veth-ns1 
ip link set up dev veth-ns2
ip link set up dev veth-ns2-pod
ip link set up dev veth-ns1-pod
```
You should have something like this at this point

```console
root@ubuntu-ns:~# ip link
[...]
5: veth-ns1-pod@veth-ns1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether e6:d3:f6:ab:e0:09 brd ff:ff:ff:ff:ff:ff
6: veth-ns1@veth-ns1-pod: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 5a:9a:10:bb:cf:7c brd ff:ff:ff:ff:ff:ff
7: veth-ns2-pod@veth-ns2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether de:2d:a1:2a:9e:7e brd ff:ff:ff:ff:ff:ff
8: veth-ns2@veth-ns2-pod: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 02:8d:f5:15:9b:01 brd ff:ff:ff:ff:ff:ff
```
Add IP addresses with ns assignements
```
ip addr add 172.30.30.1/24 dev veth-ns1
ip addr add 172.30.30.1/24 dev veth-ns2
ip link set veth-ns1-pod netns ns1
ip link set veth-ns2-pod netns ns2
ip netns exec ns1 ip link set up dev veth-ns1-pod
ip netns exec ns2 ip link set up dev veth-ns2-pod
ip netns exec ns1 ip addr add 172.30.30.11/24 dev veth-ns1-pod
ip netns exec ns2 ip addr add 172.30.30.12/24 dev veth-ns2-pod
```
From now on, you should have the basic setup represented above.
```console
##### Routing tables from default namespace
root@ubuntu-ns:~# ip route show
default via 10.57.89.1 dev ens4 proto dhcp src 10.57.89.146 metric 100 
10.57.89.0/24 dev ens4 proto kernel scope link src 10.57.89.146 
10.57.89.1 dev ens4 proto dhcp scope link src 10.57.89.146 metric 100 
172.30.30.0/24 dev veth-ns1 proto kernel scope link src 172.30.30.1 
172.30.30.0/24 dev veth-ns2 proto kernel scope link src 172.30.30.1 
root@ubuntu-ns:~# ip netns exec ns1 ip route show
172.30.30.0/24 dev veth-ns1-pod proto kernel scope link src 172.30.30.11
root@ubuntu-ns:~# ip netns exec ns1 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth-ns1-pod@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether e6:d3:f6:ab:e0:09 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.30.30.11/24 scope global veth-ns1-pod
       valid_lft forever preferred_lft forever
    inet6 fe80::e4d3:f6ff:feab:e009/64 scope link 
       valid_lft forever preferred_lft forever
root@ubuntu-ns:~# ip netns exec ns2 ip route show
172.30.30.0/24 dev veth-ns2-pod proto kernel scope link src 172.30.30.12 
root@ubuntu-ns:~# ip netns exec ns2 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
7: veth-ns2-pod@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether de:2d:a1:2a:9e:7e brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.30.30.12/24 scope global veth-ns2-pod
       valid_lft forever preferred_lft forever
    inet6 fe80::dc2d:a1ff:fe2a:9e7e/64 scope link 
       valid_lft forever preferred_lft forever

```

Here pings between NS won't work, simply because ARP request can't be resolved: we're have two separate L2 domains - one over each veth connection.





