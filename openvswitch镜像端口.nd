#!/bin/bash
# Security Bureau data syrvey
# 通过使用ovs 端口镜像功能监控源服务器数据
# 通过ovs将源虚机ovs端口 镜像至目标虚机

Source_server=$1
Target_server=$2

# 添加一个虚拟网卡
cat << EOF > /tmp/NET.xml
  <interface type='bridge'>
      <source bridge='br-wan'/>
      <virtualport type='openvswitch'>
      </virtualport>
      <model type='virtio'/>
    </interface>
EOF

# 添加
virsh attach-device ${Target_server} /tmp/NET.xml --config --live


Mirror_Port=`virsh domiflist ${Target_server}|awk 'NR==5{print $1}'|grep vnet`
ovs-vsctl -- set Bridge br-wan mirrors=@m \
-- --id=@wan${Source_server} get Port wan${Source_server} \
-- --id=@${Mirror_Port} get Port ${Mirror_Port} \
-- --id=@m create Mirror name=mirror_ceshi select-dst-port=@wan${Source_server} select-src-port=@wan${Source_server} output-port=@${Mirror_Port}
