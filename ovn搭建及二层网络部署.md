# <center>OVN环境搭建及二层网络部署<center/>

#### 环境搭建信息:

``` json
Controller 		122.114.103.184	   
Node 	  		116.255.251.58
```

#### 1. 设置yum 源信息

* master和node节点上执行

      
``` bash
systemctl stop firewalld && systemctl disable firewalld
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

# 修改CentOS的yum基础源, 如果无法访问外网请自行搭建并修改
cat >/etc/yum.repos.d/CentOS-Base.repo <<'EOF'
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#

[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
EOF

yum makecache fast
# 安装CentOS的yum epel源
yum install -y epel-release

# 修改CentOS的yum epel源, 如果无法访问外网请自行搭建并修改
cat >/etc/yum.repos.d/epel.repo <<'EOF'
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=https://mirrors.tuna.tsinghua.edu.cn/epel/7/$basearch
#mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=$basearch
failovermethod=priority
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

[epel-debuginfo]
name=Extra Packages for Enterprise Linux 7 - $basearch - Debug
baseurl=https://mirrors.tuna.tsinghua.edu.cn/epel/7/$basearch/debug
#mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-debug-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1

[epel-source]
name=Extra Packages for Enterprise Linux 7 - $basearch - Source
baseurl=https://mirrors.tuna.tsinghua.edu.cn/epel/7/SRPMS
#mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-source-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1
EOF

yum makecache
```

#### 2. 安装软件包 
* master和node节点上执行

``` bash
yum -y install firewalld-filesystem net-tools
rpm -ivh firewalld-filesystem-0.4.4.5-5.fc28.noarch.rpm
rpm -ivh openvswitch-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-debuginfo-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-devel-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-ovn-central-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-ovn-common-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-ovn-central-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-ovn-docker-2.8.1-1.el7.centos.x86_64.rpm
yum -y install python-openvswitch
rpm -ivh openvswitch-ovn-docker-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-ovn-host-2.8.1-1.el7.centos.x86_64.rpm
rpm -ivh openvswitch-ovn-vtep-2.8.1-1.el7.centos.x86_64.rpm
# 启动服务
systemctl enable ovn-northd openvswitch ovn-controller
systemctl start ovn-northd openvswitch ovn-controller
```

#### 3. 配置ovn
* OVN Controller节点

``` bash
ovn-nbctl set-connection ptcp:6641:122.114.103.184
ovn-sbctl set-connection ptcp:6642:122.114.103.184

ovs-vsctl set open . external-ids:ovn-remote=tcp:122.114.103.184:6642
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
ovs-vsctl set open . external-ids:ovn-encap-ip=122.114.103.184
```

* OVN Node节点

```
ovs-vsctl set open . external-ids:ovn-remote=tcp:122.114.103.184:6642
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
ovs-vsctl set open . external-ids:ovn-encap-ip=116.255.251.58
```

#### 4. 定义逻辑交换机
* 创建一个逻辑交换机，然后添加两个交换机端口，并为端口设置物理地址
* Controller节点

``` bash
# central节点
ovn-nbctl ls-add ls1
ovn-nbctl lsp-add ls1 ls1-vm1
ovn-nbctl lsp-set-addresses ls1-vm1 02:d4:1d:8c:d9:8f
ovn-nbctl lsp-set-port-security ls1-vm1 02:d4:1d:8c:d9:8f
ovn-nbctl lsp-add ls1 ls1-vm2
ovn-nbctl lsp-set-addresses ls1-vm2 02:d4:1d:8c:d9:8e
ovn-nbctl lsp-set-port-security ls1-vm2 02:d4:1d:8c:d9:8e
```

#### 5. 模拟虚拟机
* 创建网络命名空间，并在br-int上添加端口，然后将端口添加到命名空间，最后通过设置端口的MAC地址和网卡名完成和交换机端口的映射

``` bash

# Central节点
ip netns add vm1
ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
ip link set vm1 netns vm1
ip netns exec vm1 ip link set vm1 address 02:d4:1d:8c:d9:8f
ip netns exec vm1 ip addr add 172.16.255.11/24 dev vm1
ip netns exec vm1 ip link set vm1 up
ovs-vsctl set Interface vm1 external_ids:iface-id=ls1-vm1
ip netns exec vm1 ip addr show

# Node节点
ip netns add vm2
ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
ip link set vm2 netns vm2
ip netns exec vm2 ip link set vm2 address 02:d4:1d:8c:d9:8e
ip netns exec vm2 ip addr add 172.16.255.22/24 dev vm2
ip netns exec vm2 ip link set vm2 up
ovs-vsctl set Interface vm2 external_ids:iface-id=ls1-vm2
ip netns exec vm2 ip addr show
```

#### 6. 测试

``` bash
# Central节点 
# ip netns exec vm1 ping -c 2 172.16.255.22
PING 172.16.255.22 (172.16.255.22) 56(84) bytes of data.
64 bytes from 172.16.255.22: icmp_seq=1 ttl=64 time=0.507 ms
64 bytes from 172.16.255.22: icmp_seq=2 ttl=64 time=0.448 ms

--- 172.16.255.22 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.448/0.477/0.507/0.036 ms
```
