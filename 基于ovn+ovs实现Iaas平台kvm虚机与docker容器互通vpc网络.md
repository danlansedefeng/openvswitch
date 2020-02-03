### <center>基于ovn+ovs实现与IaaS平台kvm虚机与docker容器互通vpc网络<center/>

[参考链接](https://www.cnblogs.com/silvermagic/p/7666117.html)

* 环境搭建信息:

``` json
Controller 		122.114.103.184	   
Node 	  		116.255.251.58
```

#### 创建逻辑交换机和路由
* Controller节点

``` bash
# 创建逻辑交换机和路由器
ovn-nbctl ls-add inside
ovn-nbctl ls-add dmz
ovn-nbctl lr-add tenant1
```

#### 创建路由端口
* Controller节点

``` bash
# 创建路由器端口用于连接dmz交换机
ovn-nbctl lrp-add tenant1 tenant1-dmz 02:d4:1d:8c:d9:9f 20.0.0.1/24
# 创建交换机接口用于连接tenant1路由器
ovn-nbctl lsp-add dmz dmz-tenant1
ovn-nbctl lsp-set-type dmz-tenant1 router
ovn-nbctl lsp-set-addresses dmz-tenant1 02:d4:1d:8c:d9:9f
ovn-nbctl lsp-set-options dmz-tenant1 router-port=tenant1-dmz

# 创建路由器端口用于连接inside交换机
ovn-nbctl lrp-add tenant1 tenant1-inside 02:d4:1d:8c:d9:9e 10.0.0.1/24
# 创建交换机接口用于连接tenant1路由器
ovn-nbctl lsp-add inside inside-tenant1
ovn-nbctl lsp-set-type inside-tenant1 router
ovn-nbctl lsp-set-addresses inside-tenant1 02:d4:1d:8c:d9:9e
ovn-nbctl lsp-set-options inside-tenant1 router-port=tenant1-inside

```

#### 创建交换机
* Controller节点

``` bash
# 创建交换机接口用于连接虚拟机,(不加IP的话，后面dhclient会超时，分配不了IP,这里不采用dhcp方式获取ip)
ovn-nbctl lsp-add dmz dmz-vm1
ovn-nbctl lsp-set-addresses dmz-vm1 02:d4:1d:8c:d9:9d
ovn-nbctl lsp-set-port-security dmz-vm1 02:d4:1d:8c:d9:9d
ovn-nbctl lsp-add dmz dmz-vm2
ovn-nbctl lsp-set-addresses dmz-vm2 02:d4:1d:8c:d9:9c
ovn-nbctl lsp-set-port-security dmz-vm2 02:d4:1d:8c:d9:9c

# 创建交换机端口连接容器网络
ovn-nbctl lsp-add dmz dmz-docker
ovn-nbctl lsp-set-addresses dmz-docker 02:d4:1d:8c:c9:1c
ovn-nbctl lsp-set-port-security dmz-docker 02:d4:1d:8c:c9:1c

# 创建交换机接口用于连接虚拟机
ovn-nbctl lsp-add inside inside-vm3
ovn-nbctl lsp-set-addresses inside-vm3 02:d4:1d:8c:d9:9b
ovn-nbctl lsp-set-port-security inside-vm3 02:d4:1d:8c:d9:9b
ovn-nbctl lsp-add inside inside-vm4
ovn-nbctl lsp-set-addresses inside-vm4 02:d4:1d:8c:d9:9a
ovn-nbctl lsp-set-port-security inside-vm4 02:d4:1d:8c:d9:9a

```

#### 创建虚拟机
* Controller节点

``` bash
ip netns add vm1
ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
ip link set vm1 address 02:d4:1d:8c:d9:9d
ip link set vm1 netns vm1
ovs-vsctl set Interface vm1 external_ids:iface-id=dmz-vm1
ip netns exec vm1 ip addr add 20.0.0.10/24 dev vm1
ip netns exec vm1 ip link set vm1 up
ip netns exec vm1 ip route add default via  20.0.0.1 dev vm1

ip netns add vm2
ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
ip link set vm2 address 02:d4:1d:8c:d9:9c
ip link set vm2 netns vm2
ovs-vsctl set Interface vm2 external_ids:iface-id=dmz-vm2
ip netns exec vm2 ip addr add 20.0.0.20/24 dev vm2
ip netns exec vm2 ip link set vm2 up
ip netns exec vm2 ip route add default via  20.0.0.1 dev vm2

```

* Node节点

``` bash
ip netns add vm3
ovs-vsctl add-port br-int vm3 -- set interface vm3 type=internal
ip link set vm3 address 02:d4:1d:8c:d9:9b
ip link set vm3 netns vm3
ovs-vsctl set Interface vm3 external_ids:iface-id=inside-vm3
ip netns exec vm3 ip addr add 10.0.0.10/24 dev vm3
ip netns exec vm3 ip link set vm3 up
ip netns exec vm3 ip route add default via  10.0.0.1 dev vm3


ip netns add vm4
ovs-vsctl add-port br-int vm4 -- set interface vm4 type=internal
ip link set vm4 address 02:d4:1d:8c:d9:9a
ip link set vm4 netns vm4
ovs-vsctl set Interface vm4 external_ids:iface-id=inside-vm4
ip netns exec vm4 ip addr add 10.0.0.10/24 dev vm4
ip netns exec vm4 ip link set vm4 up
ip netns exec vm4 ip route add default via  10.0.0.1 dev vm4
```

* docker容器，在任意节点上都可以执行

``` bash
docker pull busybox	
# 创建容器，设置网络为none
docker run -d --name con1 --net=none --mac-address 02:d4:1d:8c:c9:1c --privileged=true busybox top
# 将容器网络加入ovs当中，并设置mac，ip及网关
ovs-docker add-port br-int eth0 con20 --ipaddress=20.0.0.30/24 --macaddress="02:d4:1d:8c:c9:1c" --gateway=20.0.0.1
# 将添加的容器端口指定至创建的vpc交换机端口中
ovs-vsctl set Interface 897aec08653c4_l  external_ids:iface-id=dmz-docker

```

#### 测试
``` bash
[root@ovn-node ~]# docker exec con20 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
32: eth0@if33: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue qlen 1000
    link/ether 02:d4:1d:8c:c9:1c brd ff:ff:ff:ff:ff:ff
    inet 20.0.0.30/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::b8e6:e2ff:fe50:33e1/64 scope link
       valid_lft forever preferred_lft forever
[root@ovn-node ~]# docker exec con20 ip route sh
default via 20.0.0.1 dev eth0
20.0.0.0/24 dev eth0 scope link  src 20.0.0.30

# 测试到dmz 网关
[root@ovn-node ~]# docker exec con20 ping -c 2 20.0.0.1
PING 20.0.0.1 (20.0.0.1): 56 data bytes
64 bytes from 20.0.0.1: seq=0 ttl=254 time=1.112 ms
64 bytes from 20.0.0.1: seq=1 ttl=254 time=0.368 ms

# 测试到 inside网关及虚机
[root@ovn-node ~]# docker exec con20 ping -c 2 10.0.0.1
PING 10.0.0.1 (10.0.0.1): 56 data bytes
64 bytes from 10.0.0.1: seq=0 ttl=254 time=1.612 ms
64 bytes from 10.0.0.1: seq=1 ttl=254 time=0.448 ms

--- 10.0.0.1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.448/1.030/1.612 ms
[root@ovn-node ~]# docker exec con20 ping -c 2 10.0.0.10
PING 10.0.0.10 (10.0.0.10): 56 data bytes
64 bytes from 10.0.0.10: seq=0 ttl=63 time=1.366 ms
64 bytes from 10.0.0.10: seq=1 ttl=63 time=0.106 ms

```

```
由于 OVN 最初是为 OpenStack 网络功能设计的，提供了大量 Kubernetes 网络目前不存在的功能：

L2/L3 网络虚拟化包括：

• 分布式交换机，分布式路由器

• L2到L4的ACL

• 内部和外部负载均衡器

• QoS，NAT，分布式DNS

• Gateway

• IPv4/IPv6 支持

https://www.jianshu.com/p/0e59a1bd7eb2

```
