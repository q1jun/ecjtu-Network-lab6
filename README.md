# ecjtu-Network-lab6
华东交通大学计算机网络实验六 Cisco Packet Tracer  V8.0.0

（一）校园网设计示例1（必做内容）

（1）某学院有教职工50人，平均分布在教学楼1和教学楼2，学生400人，平均分布在宿舍楼1和宿舍楼2，有一个数据中心服务器。
（2）每栋教学楼和宿舍楼分别设置为不同的VLAN网段，教学楼通过汇聚路由器连接到学院的核心路由器，宿舍楼通过三层交换机连接到学院的核心路由器。数据中心服务器作为一个独立网段连接到核心路由器。
（3）学院核心路由器通过serial接口连接到ISP的路由器，从而连接到互联网。
（4）采用合适的路由配置策略（静态路由、RIP协议、OSPF协议），使得学院网络内部互联互通。
（5）在学院核心路由器上配置合适的NAT，使得宿舍楼1和宿舍楼2所属子网范围能访问外网，而教学楼1和教学楼2所属子网范围不能访问外网。
（6）在学院核心路由器上设置标准ACL或扩展ACL，允许教学楼的用户只可以访问数据中心服务器的WWW服务和FTP服务。允许宿舍楼的用户可以访问外网的资源，也能访问教学楼的资源，其余的都不可以访问。

请根据以上需求，给出学院的具体网络设计方案，各楼栋分布与节点数需求，画出网络拓扑结构设计图，给出VLAN 及 所有IP 地址规划，完成设备选型，详细实验配置等。并将以上所有内容记录在实验报告上。


## 0x01:按照拓扑图添加好设备，并标明接口和IP等等

![image](https://user-images.githubusercontent.com/57565901/121010883-8abd9900-c7c8-11eb-9a8b-440e22b6dc99.png)

>注意其中核心路由器Router1，我们要先关闭电源，添加接口模块 `NM-2FE2W` 和 `WIC-1T` ：
![image](https://user-images.githubusercontent.com/57565901/121011001-b17bcf80-c7c8-11eb-9774-01a7cd08fca4.png)

## 0x02:下面便是配置计算机IP、网关、掩码和路由器端口了

![image](https://user-images.githubusercontent.com/57565901/121011872-b0976d80-c7c9-11eb-85b6-3f496f9f864b.png)

>注意右上角三层交换机这里的节点IP地址是没有给出的，
>这里我们根据下面的网关命名习惯推算出此处网关为`10.1.1.254/24`
>![image](https://user-images.githubusercontent.com/57565901/121012526-619e0800-c7ca-11eb-97b4-0892eb1e345c.png)

其他的PC和server简单的配置一下IP、网关、掩码就行，没有什么特别的地方。

配置[Router2]的f0/1.1和f0/1.2两个子端口的命令：
```shell
Router>en
Router#conf t
Router(config)#int fa0/1  //开启端口fa0/1
Router(config-if)#no shut
Router(config-if)#int fa0/1.1
Router(config-subif)#encapsulation dot1q 2 
//重要指令，dot1Q为这个接口配置802.1Q协议 ，最后的2是设置为vlan2的意思
Router(config-subif)#ip add 192.168.2.254 255.255.255.0
Router(config-subif)#no shut
Router(config-subif)#int fa0/1.2
Router(config-subif)#encapsulation dot1q 3
Router(config-subif)#ip add 192.168.3.254 255.255.255.0
Router(config-subif)#no shut 
Router(config)#int fa0/0
Router(config-if)#ip address 192.168.4.100 255.255.255.0
Router(config-if)#no shut
```
设置交换机[Switch 1]的f0/1为trunk模式(Switch 2同理)，划分VLAN:
```shell
Switch>en
Switch#conf t
Switch(config)#int fa0/1
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk allowed vlan all
Switch(config)#int fa0/2
Switch(config-if)#switchport access vlan 2
Switch(config-if)#no shut
Switch(config-if)#int fa0/3
Switch(config-if)#switchport access vlan 3
Switch(config-if)#no shut
```
配置三层交换机MS1:
```shell
Switch>en
Switch#conf t
Switch(config)#vlan 10
Switch(config-vlan)#exit
Switch(config)#vlan 20
Switch(config-vlan)#exit
Switch(config)#int fa0/2 
Switch(config-if)#switchport access vlan 20
Switch(config-if)#int fa0/1
Switch(config-if)#switchport access vlan 10
Switch(config-if)#exit
Switch(config)#int vlan 10
Switch(config-if)#ip address 10.1.2.100 255.255.255.0
Switch(config-if)#no shut
Switch(config-if)#int vlan 20
Switch(config-if)#ip address 10.1.1.254 255.255.255.0
Switch(config-if)#no shut
```
配置三层交换机MS2:
```shell
Switch>en
Switch#conf t
Switch(config)#int fa0/1
Switch(config-if)#switchport access vlan 5 //这里写vlan 4或者vlan 5都可以,虚接口
Switch(config-if)#exit
Switch(config)#int vlan 4
Switch(config-if)#ip add 172.16.4.254 255.255.255.0
Switch(config-if)#no shut
Switch(config-if)#int vlan 5
Switch(config-if)#ip add 172.16.5.254 255.255.255.0
Switch(config-if)#no shut
Switch(config)#int vlan 10
Switch(config)#int fa0/2
Switch(config-if)#switchport access vlan 10
Switch(config-if)#exit
Switch(config)#int vlan 10 //配置一个vlan10虚接口作为路由出口
Switch(config-if)#ip add 172.16.1.100 255.255.255.0
Switch(config-if)#no shut
Switch(config-if)#exit
```
## 0x03:动态路由-OSPF协议：

关于ospf、bgp中的Router-ID和loopback接口：
> 动态路由协议OSPF 、BGP 在运行过程中需要为该协议指定一个Router id，作为此路由器的唯一标识，并要求在整个自治系统内唯一。由于router id 是一个32 位的无符号整数，这一点与IP地址十分相像。而且IP 地址是不会出现重复现象的，所以通常将路由器的router id指定为与该设备上的某个接口的地址相同。由于loopback 接口的IP 地址通常被视为路由器的标识，所以也就成了router id的最佳选择。

### 0x0301:关于ospf中区域``area``的划分如下,中间的终端路由器为``area 0``作为骨干网络连接其他网络:

![image](https://user-images.githubusercontent.com/57565901/121023330-c3b03a80-c7d5-11eb-84b3-2dcc540634aa.png)

### 0x0301:动态路由配置命令：
Router ISP:
```shell
Router(config)#int loopback0 //设置回环接口
Router(config-if)#ip address 1.1.1.1 255.255.255.0
Router(config-if)#no shut
Router(config-if)#exit
Router(config)#router ospf 1
Router(config-router)#router-id 1.1.1.1 //设置路由ID
Router(config-router)#network 202.121.241.0 0.0.0.255 area 1
Router(config-router)#network 219.220.240.0 0.0.0.255 area 1
Router(config-router)#network 1.1.1.0 0.0.0.255 area 1
Router(config-router)#network 5.5.5.0 0.0.0.255 area 0 //5.5.5.0为核心网络号
Router(config-router)#log-adjacency-changes  //开启日志提示
Router(config-router)#exit
Router(config)#ip route 0.0.0.0 0.0.0.0 202.121.241.8
```
Router1:
```shell
Router>en
Router#conf t
Router(config)#int loopback0
Router(config-if)#ip add 5.5.5.5 255.255.255.0
Router(config-if)#no shut
Router(config-if)#exit
Router(config)#router ospf 5
Router(config-router)#router-id 5.5.5.5
Router(config-router)#network 5.5.5.0 0.0.0.255 area 0
Router(config-router)#network 202.121.241.0 0.0.0.255 area 1
Router(config-router)#network 10.1.2.0 0.0.0.255 area 2
Router(config-router)#network 192.168.4.0 0.0.0.255 area 3
Router(config-router)#network 172.16.1.0 0.0.0.255 area 4
Router(config-router)#log-adjacency-changes //开启日志功能
```
Router2:
```shell
Router(config)#int loopback0
Router(config-if)#ip add 3.3.3.3 255.255.255.0
Router(config-if)#no shut
Router(config-if)#exit
Router(config)#router ospf 3
Router(config-router)#router-id 3.3.3.3
Router(config-router)#network 5.5.5.0 0.0.0.255 area 0
Router(config-router)#network 192.168.4.0 0.0.0.255 area 3
Router(config-router)#network 192.168.2.0 0.0.0.255 area 3
Router(config-router)#network 192.168.3.0 0.0.0.255 area 3
Router(config-router)#network 3.3.3.0 0.0.0.255 area 3
Router(config-router)#log-adjacency-changes //开启日志功能
Router(config-router)#exit
Router(config)#ip route 0.0.0.0 0.0.0.0 192.168.4.101
Router(config)#end
Router#
```
MS1:
```shell
Switch>en
Switch#conf t
Switch(config)#int loopback0
Switch(config-if)#ip add 2.2.2.2 255.255.255.0
Switch(config-if)#no shut
Switch(config-if)#exit
Switch(config)#ip routing
Switch(config)#router ospf 2
Switch(config-router)#router-id 2.2.2.2
Switch(config-router)#network 2.2.2.0 0.0.0.255 area 2
Switch(config-router)#network 10.1.2.0 0.0.0.255 area 2
Switch(config-router)#network 10.1.1.0 0.0.0.255 area 2
Switch(config-router)#network 5.5.5.0 0.0.0.255 area 0
Switch(config-router)#log-adjacency-changes 
Switch(config-router)#exit
Switch(config)#ip route 0.0.0.0 0.0.0.0 10.1.2.101
```
MS2:
```shell
Switch#conf t
Switch(config)#int loopback0
Switch(config-if)#ip add 4.4.4.4 255.255.255.0
Switch(config-if)#no shut
Switch(config-if)#exit
Switch(config)#ip routing
Switch(config)#router ospf 4
Switch(config-router)#router-id 4.4.4.4
Switch(config-router)#network 4.4.4.0 0.0.0.255 area 4
Switch(config-router)#network 172.16.1.0 0.0.0.255 area 4
Switch(config-router)#network 5.5.5.0 0.0.0.255 area 0
Switch(config-router)#network 172.16.4.0 0.0.0.255 area 4
Switch(config-router)#network 172.16.5.0 0.0.0.255 area 4
Switch(config-router)#log-adjacency-changes 
Switch(config-router)#exit
Switch(config)#
Switch(config)#ip route 0.0.0.0 0.0.0.0 172.16.1.101 //缺省路由
```

## 0x04:至此所有设备均可以完成通讯,剩下的就是NAT协议和ACL策略组的布置了

连通效果图：
![image](https://user-images.githubusercontent.com/57565901/121059622-40eaa800-c7f4-11eb-8a56-9187653606e0.png)

### 0x0401 访问控制列表ACL

  标准IP访问控制列表，一个标准IP访问控制列表匹配IP协议包钟的源地址或源地址中的一部分，
可对匹配的协议包采取拒绝或允许两个操作。编号范围是从1~99.

  扩展IP访问控制列表，扩展IP访问控制列表比标准IP访问控制列表具有更多的匹配项，
包括协议类型、源地址、目的地址、源端口、目的端口、建立连接的和IP优先级等。
编号范围是从100~199
```
这里我们再看看该实验的要求：
  （5）在学院核心路由器上配置合适的NAT，使得宿舍楼1和宿舍楼2所属子网范围能访问外网，而教学楼1和教学楼2所属子网范围不能访问外网。
  （6）在学院核心路由器上设置标准ACL或扩展ACL，允许教学楼的用户只可以访问数据中心服务器的WWW服务和FTP服务。
允许宿舍楼的用户可以访问外网的资源，也能访问教学楼的资源，其余的都不可以访问。
```
