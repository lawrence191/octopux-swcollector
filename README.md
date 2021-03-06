
感谢[gaochao1](https://github.com/gaochao1)开源 swcollector

我们基于swcollector根据自己需求做了部分修改。主要有这四点。
* 启动方式支持supervisor启动，保证程序持续运行。
* 添加了采集本机网卡和iptables统计的内外网流量信息。之前我们通过snmp方式采集服务信息，小运营商存在比较多丢包情况。现在由本机的swcollector采集，采集数据上报比较完整。并且依赖软件发布机制，和swcollector主动上报无需配置服务流量采集。
* 实际使用中存在交换机采集超时问题较为严重。修改交换机采集方式为按交换机端口独立发起采集。减少交换机采集超时问题。
* 添加采集交换机的丢包数

*PS*：


本机内外网采集依赖iptables统计功能。下面是我们的配置脚本片段：

```
dev=$(route -n|egrep ^0.0.0.0|egrep -o "[a-z0-9.]+$"|sed 1p -n)
net=`ifconfig $dev|fgrep Bcast|awk '{print $2}'|cut -d: -f2|sed -r 's@[0-9]+$@0/24@'|sed 1p -n`

iptables -N traffic_in_$dev
iptables -N traffic_out_$dev
iptables -N traffic_lan_in_$dev
iptables -N traffic_lan_out_$dev

iptables -I INPUT -i $dev -j traffic_in_$dev
iptables -I OUTPUT -o $dev -j traffic_out_$dev
iptables -I INPUT -i $dev -j traffic_lan_in_$dev
iptables -I OUTPUT -o $dev -j traffic_lan_out_$dev

iptables -A traffic_in_$dev -i $dev
iptables -A traffic_out_$dev -o $dev
iptables -A traffic_lan_in_$dev -i $dev -s $net
iptables -A traffic_lan_out_$dev -o $dev -d $net
```


－－－－－－－－－

基于小米运维开源的[open-falcon](http://open-falcon.com)，交换机专用agent。

感谢小米运维同学的杰出工作、开源、良好的文档和热心。

感谢[来炜](https://github.com/laiwei)的宝贵建议。

##简介
采集的metric列表：

* CPU利用率
* 内存利用率
* Ping延时
* IfHCInOctets
* IfHCOutOctets
* IfHCInUcastPkts
* IfHCOutUcastPkts

CPU和内存的OID私有，根据设备厂家和OS版本可能不同。目前测试过的设备：

* Cisco IOS(Version 12)
* Cisco NX-OS(Version 6)
* Huawei VRP(Version 8)
* H3C(Version 5)
* H3C(Version 7)

##源码安装
	依赖$GOPATH/src/github.com/gaochao1/sw
	cd $GOPATH/src/github.com/gaochao1/swcollector
	go get ./...
	./control build
	./control pack
	最后一步会pack出一个tar.gz的安装包，拿着这个包去部署服务即可。

##部署说明

swcollector需要部署到有交换机SNMP访问权限的服务器上。

使用Go原生的ICMP协议进行Ping探测，swcollector需要root权限运行。

Huawei交换机使用Go原生SNMP协议会报超时或乱序错误。暂时解决方法是SNMP接口流量查询前先判断设备型号，对Huawei设备，调用snmpwalk命令进行数据收集。


##配置说明

配置文件请参照cfg.example.json，修改该文件名为cfg.json，将该文件里的IP换成实际使用的IP。

switch配置项说明：

	"switch":{
	   "enabled": true,
		"ipRange":[						#交换机IP地址段，对该网段有效IP，先发Ping包探测，对存活IP发SNMP请求
           "192.168.1.0/24",
           "192.168.56.102/32",
           "172.16.114.233"
 		],
 		"pingTimeout":300, 			   #Ping超时时间，单位毫秒
		"pingRetry":3,				   #Ping探测重试次数
		"community":"public",			#SNMP认证字符串
		"snmpTimeout":1000,				#SNMP超时时间，单位毫秒
		"snmpRetry":5,					#SNMP重试次数
		"ignoreIface": ["Nu","NU","Vlan","Vl"],    #忽略的接口，如Nu匹配ifName为*Nu*的接口
		"ignorePkt": true,            #不采集IfHCInUcastPkts和IfHCOutUcastPkts
 		"limitConcur": 1000           #限制SNMP请求并发数
    }


