# 第3天-Keepalived 高可用集群

## 一、Keepalived 简介

Keepalived是集群管理中保证集群高可用的一个服务软件，它的作用是检测web服务器的状态，如果有一台web服务器死机，或工作出现故障，Keepalived将检测到，并将有故障的web服务器从系统中剔除，当web服务器工作正常后，自动将web服务器加入到服务器集群中。解决了静态路由的单点故障问题。

## 二、Keepalived 工作原理

Keepalived是以VRRP协议为实现基础的，VRRP全称Virtual Router Redundancy Protocol ,即虚拟路由冗余协议。

虚拟路由冗余协议，可以认为是实现路由器高可用的协议。也就是说N台提供相同功能的路由器组成一个路由器组，这个组里面有一个master和多个backup，master上面有一个对外提供服务的vip，master不断向backup发送心跳信息，告诉backup自己还活着，当backup收不到心跳消息时就认为master已经宕机啦，这时就需要根据VRRP的优先级来选举一个backup当master。从而保证高可用。

  Keepalived主要有三个模块，分别是 core、check 和 vrrp。

- core 模块为 keepalived 的核心，负责主进程的启动、维护、以及全局配置文件的加载和解析。

- check 负责健康检查，包括常见的各种检查方式。

- vrrp 模块是来实现 VRRP 协议的。

## 三、Keepalived 配置文件

  Keepalived 只有一个配置文件 keepalived.conf，里面主要包括以下几个配置区域，分别是 

- global_defs

- static_ipaddress

- vrrp_script

- vrrp_instance 

- virtual_server

### 1、global_defs 区域

主要是配置故障发生时的通知对象以及机器标志

```shell
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id 192.168.224.206
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
```

- notification_email  故障发生时给谁发邮件通知
- notification_email_from  通知邮件从哪个地址发出
- smtp_server 通知邮件的smtp地址
- smtp_connect_timeout 连接smtp服务器的超时时间
- enable_traps开启SNMP（Simple Network Management Protocol）陷阱
- router_id 标志本节点的字符串，通常为ip地址，故障发生时邮件会通知到

### 2、vrrp_script 区域  

 用来做健康检查的，当检查失败时会将 vrrp_instance 的 priority 减少相应的值

```shell
vrrp_script chk_nginx {
       script "/usr/local/keepalived-1.3.4/nginx_check.sh"
       interval 2 
       weight -20
}
```

-    script: 自己写的监测脚本。


-    interval 2: 每2s监测一次


-    weight -20：监测失败，则相应的vrrp_instance的优先级会减少20个点


### 3、vrrp_instance 区域

```shell
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    mcast_src_ip 192.168.224.206
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.224.208
    }
    track_script{
    chk_nginx
  }
}
```

- state:只有BACKUP和MASTER。MASTER为工作状态，BACKUP是备用状态

- interface:为网卡接口：可通过ip addr查看自己的网卡接口


- virtual_router_id:虚拟路由标志。同组的virtual_router_id应该保持一致。它将决定多播的MAC地址


- priority：设置本节点的优先级，优先级高的为master


- advert_int: MASTER与BACKUP同步检查的时间间隔


- virtual_ipaddress：这就是传说中的虚拟ip

## 四、Keepalived实战项目

### 1、Haproxy_Director + Keepalived

```shell
一、Haproxy负载均衡
主/备调度器均能够实现正常调度

二、Keepalived实现调度器HA
注：主/备调度器均能够实现正常调度
1. 主/备调度器安装软件
yum安装的方式：
[root@master ~]# yum -y install keepalived 
[root@backup ~]# yum -y install keepalived 
编译安装的方式：
[root@qfedu.com ~]# yum -y install ipvsadm kernel-headers kernel-devel openssl-devel popt-devel
[root@qfedu.com ~]# wget http://www.keepalived.org/software/keepalived-1.2.2.tar.gz
[root@qfedu.com ~]# tar zxvf keepalived-1.2.2.tar.gz
[root@qfedu.com ~]# cd keepalived-1.2.2
[root@qfedu.com ~]# ./configure --prefix=/
[root@qfedu.com ~]# make 
[root@qfedu.com ~]# make install

2. Keepalived
Master 
[root@qfedu.com ~]# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id director1	        # 辅助改为director2
}

vrrp_instance VI_1 {
    state BACKUP
    nopreempt				
    interface eth0				# VIP绑定接口
    virtual_router_id 80		# MASTER,BACKUP一致
    priority 100			    # 辅助改为50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.122.100
    }
}

 


3. 启动KeepAlived（主备均启动）
[root@qfedu.com ~]# chkconfig keepalived on
[root@qfedu.com ~]# service keepalived start
[root@qfedu.com ~]# ip addr


4. 扩展对调度器Haproxy健康检查（可选）
思路：
让Keepalived以一定时间间隔执行一个外部脚本，脚本的功能是当Haproxy失败，则关闭本机的Keepalived
a. script
[root@master ~]# cat /etc/keepalived/check_haproxy_status.sh
#!/bin/bash											        	
/usr/bin/curl -I http://localhost &>/dev/null	
if [ $? -ne 0 ];then									    	
	/etc/init.d/keepalived stop					    	
fi															        	
[root@master ~]# chmod a+x /etc/keepalived/check_haproxy_status.sh

b. keepalived使用script
! Configuration File for keepalived

global_defs {
   router_id director1
}

vrrp_script check_haproxy {
   script "/etc/keepalived/check_haproxy_status.sh"
   interval 5
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    nopreempt
    virtual_router_id 90
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass qfedu
    }
    virtual_ipaddress {
        192.168.122.100
    }

    track_script {
        check_haproxy
    }
}

```

### 2、Nginx_Director + Keepalived

```shell
一、Nginx负载均衡
主/备调度器均能够实现正常调度


二、Keepalived实现调度器HA
1. 主/备调度器安装软件
[root@master ~]# yum -y install keepalived 
[root@backup ~]# yum -y install keepalived

2. Keepalived
BACKUP1
[root@qfedu.com ~]# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     root@localhost     
   }
   notification_email_from root@local.domain
   smtp_server 10.8.16.10
   smtp_connect_timeout 30
   router_id SCRM-MySQL-11
}
vrrp_instance VI_1 {
    state BACKUP
    nopreempt
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.8.16.50
    }
}

BACKUP2


3. 启动KeepAlived（主备均启动）
[root@qfedu.com ~]# chkconfig keepalived on
[root@qfedu.com ~]# service keepalived start
[root@qfedu.com ~]# ip addr

到此：
可以解决心跳故障  keepalived
不能解决Nginx服务故障

4. 扩展对调度器Nginx健康检查（可选）
思路：
让Keepalived以一定时间间隔执行一个外部脚本，脚本的功能是当Nginx失败，则关闭本机的Keepalived
a. script
[root@master ~]# cat /etc/keepalived/check_nginx_status.sh
#!/bin/bash											        	
/usr/bin/curl -I http://localhost &>/dev/null	
if [ $? -ne 0 ];then									    	
	/etc/init.d/keepalived stop					    	
fi															        	
[root@master ~]# chmod a+x /etc/keepalived/check_nginx_status.sh

b. keepalived使用script
! Configuration File for keepalived

global_defs {
   router_id director1
}

vrrp_script check_nginx {
   script "/etc/keepalived/check_nginx_status.sh"
   interval 5
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    nopreempt
    virtual_router_id 90
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass qfedu
    }
    
    virtual_ipaddress {
        192.168.1.80
    }

    track_script {
        check_nginx
    }
}

注：必须先启动nginx，再启动keepalived
```

### 3、MySQL + Keepalived

```shell
Keepalived+mysql 自动切换

项目环境：
VIP 192.168.122.100
mysql1 192.168.122.10
mysql2 192.168.122.20
 

实施步骤：
一、keepalived 主备配置文件
192.168.122.10 Master配置
[root@qfedu.com ~]# vim /etc/keepalived/keepalived.conf
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝
! Configuration File for keepalived

global_defs {
   router_id mysql1
}

vrrp_script check_run {
   script "/root/keepalived_check_mysql.sh"
   interval 5
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 88
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass qfedu
    }

    track_script {
        check_run
    }

    virtual_ipaddress {
        192.168.122.100
    }
}

＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝



192.168.122.20 Slave配置
[root@qfedu.com ~]# vim /etc/keepalived/keepalived.conf
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝
! Configuration File for keepalived

global_defs {
   router_id mysql2
}

vrrp_script check_run {
   script "/root/keepalived_check_mysql.sh"
   interval 5
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 88
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass qfedu
    }

    track_script {
        check_run
    }

    virtual_ipaddress {
        192.168.122.100
    }
}

1. 注意空格
2. 日志查看脚本是否被执行
[root@xen2 ~]# tail -f /var/log/messages 
Jun 19 15:20:19 xen1 Keepalived_vrrp[6341]: Using LinkWatch kernel netlink reflector...
Jun 19 15:20:19 xen1 Keepalived_vrrp[6341]: VRRP sockpool: [ifindex(2), proto(112), fd(11,12)]
Jun 19 15:20:19 xen1 Keepalived_vrrp[6341]: VRRP_Script(check_run) succeeded

＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝


二、mysql状态检测脚本/root/keepalived_check_mysql.sh（两台MySQL同样的脚本）
版本一：简单使用：
#!/bin/bash
/usr/bin/mysql -uroot -p123 -e "show status" &>/dev/null 
if [ $? -ne 0 ] ;then 
	service keepalived stop
fi 


版本二：检查多次：
[root@qfedu.com ~]#  vim  /root/keepalived_check_mysql.sh
#!/bin/bash 
MYSQL=/usr/local/mysql/bin/mysql
MYSQL_HOST=localhost 
MYSQL_USER=root 
MYSQL_PASSWORD=qfedu
CHECK_TIME=3

#mysql  is working MYSQL_OK is 1 , mysql down MYSQL_OK is 0 
MYSQL_OK=1
 
check_mysql_helth (){ 
    $MYSQL -h $MYSQL_HOST -u $MYSQL_USER -p${MYSQL_PASSWORD} -e "show status" &>/dev/null 
    if [ $? -eq 0 ] ;then 
    	MYSQL_OK=1
    else 
    	MYSQL_OK=0 
    fi 
    return $MYSQL_OK 
}
 
while [ $CHECK_TIME -ne 0 ]
do 
    check_mysql_helth 
	if [ $MYSQL_OK -eq 1 ] ; then 
    		exit 0 
	fi
 
	if [ $MYSQL_OK -eq 0 ] &&  [ $CHECK_TIME -eq 1 ];then
     		/etc/init.d/keepalived stop					
     		exit 1								
 	fi										
	let CHECK_TIME--
 	sleep 1 
done

版本三：检查多次：
[root@qfedu.com ~]#  vim  /root/keepalived_check_mysql.sh
#!/bin/bash 
MYSQL=/usr/local/mysql/bin/mysql
MYSQL_HOST=localhost 
MYSQL_USER=root 
MYSQL_PASSWORD=qfedu
CHECK_TIME=3

#mysql  is working MYSQL_OK is 1 , mysql down MYSQL_OK is 0 
MYSQL_OK=1
 
check_mysql_helth (){ 
    $MYSQL -h $MYSQL_HOST -u $MYSQL_USER -p${MYSQL_PASSWORD} -e "show status" &>/dev/null 
    if [ $? -eq 0 ] ;then 
    	MYSQL_OK=1
    else 
    	MYSQL_OK=0 
    fi 
    return $MYSQL_OK 
}
 
while [ $CHECK_TIME -ne 0 ]
do 
    check_mysql_helth 
	if [ $MYSQL_OK -eq 1 ] ; then 
    		exit 0 
	fi
 
	let CHECK_TIME--
 	sleep 1 
done

/etc/init.d/keepalived stop					
exit 1				
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝

[root@qfedu.com ~]# chmod 755 /root/keepalived_check_mysql.sh

两边均启动keepalived
[root@qfedu.com ~]# /etc/init.d/keepalived start
[root@qfedu.com ~]# /etc/init.d/keepalived start
[root@qfedu.com ~]# chkconfig --add keepalived
[root@qfedu.com ~]# chkconfig keepalived on
```

### 4、Lvs_Director + Keepalived

由于lvs的无法监控后端的real server是否宕机，故我们采用keepalived+LVS DR的方式，来监控后端real server的服务，当real server宕机时，不再将请求转发至已经宕机的real server。由于LVS的功能已经嵌套进了keepalived软件里，故我们只需要在调度器（director）上安装keepalived即可，不用安装ipvsadm包，也不需要写lvs_dr.sh脚本，只需要写keepalived的脚本即可。

**为了节省时间，这里的高可用集群我只做master主机，不做backup备用机**

**三台服务器A、B、C：**

#### 1、A: load balancer

（调度器dir，分发器）

内网网卡：192.168.31.128，网关保持不变（192.168.31.2）

外网网卡：192.168.229.128，先不用理会，这里用不到

##### 1、备份之前的 keepalived 配置脚本（nginx高可用）

```shell
[root@a.qfedu.com  ~]# yum -y install ipvsadm net-tools keepalived
[root@a.qfedu.com  ~]# mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf-bak
```

##### 2、清空 ipvsadm 规则

```shell
[root@a.qfedu.com  ~]# ipvsadm -C
```

##### 3、创建 keepalived 的配置脚本

```shell
[root@a.qfedu.com  ~]# vim /etc/keepalived/keepalived.conf
#写入如下内容：
vrrp_instance VI_1 {
state MASTER  # 角色为master
interface eth0 # 访问接口为eth0，vip绑定的网卡名称
virtual_router_id 51 # 虚拟路由id为51，须和backup保持一致
priority 100 # 权重为100，backup稍低一些
advert_int 1
authentication {
auth_type PASS # 验证类型密码
auth_pass 123456 # 验证密码为123456
}
virtual_ipaddress {
192.168.31.200 #vip
}
}
virtual_server 192.168.31.200 80 { # 绑定访问ip及端口
delay_loop 10 # 每隔10秒查询realserver状态
lb_algo wlc lvs # 调度算法
lb_kind DR # lvs转发模式
persistence_timeout 60 # 登陆保持时限为60秒
protocol TCP # 用TCP协议检查realserver状态
real_server 192.168.31.129 80 { # reala server设置
weight 100 # 权重
TCP_CHECK { # 用tcp协议检测
connect_timeout 10 # 连接超时时限为10秒
nb_get_retry 3
delay_before_retry 3
connect_port 80 # 连接端口80
}
}
real_server 192.168.31.130 80 { # 另一台real server配置，同上
weight 100
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 80
}
}
}

例子：
vrrp_instance VI_1 {
state MASTER
interface eth0
virtual_router_id 51
priority 100
advert_int 1
authentication {
auth_type PASS
auth_pass 123456
}
virtual_ipaddress {
192.168.152.200
}
}
virtual_server 192.168.152.200 80 {
delay_loop 10
lb_algo rr
lb_kind DR
persistence_timeout 60
protocol TCP
real_server 192.168.152.132 80 {
weight 110
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 80
}
}
real_server 192.168.152.133 80 {
weight 100
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 80
}
}
}
```

##### 4、查看 ipvsadm 转发规则

```shell
[root@a.qfedu.com  ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
-> RemoteAddress:Port Forward Weight ActiveConn InActConn
```

##### 5、启动 keepalived

```shell
[root@a.qfedu.com ~]# systemctl start keepalived
```

##### 6、再次查看 ipvsadm 规则,会发现有了转发规则

```shell
[root@a.qfedu.com ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
-> RemoteAddress:Port Forward Weight ActiveConn InActConn
TCP 192.168.31.200:80 wlc persistent 60
-> 192.168.31.129:80 Route 100 0 0
-> 192.168.31.130:80 Route 100 0 0
```





#### 2、B: real server

（web服务器） 内网网卡：192.168.31.129 网关改回129.168.31.2

安装nginx，并启动，在默认主页里写入，real server 1 关闭selinux，清空防火墙规则

##### 1、创建转发脚本

```shell
[root@b.qfedu.com ~]# yum -y install net-tools

[root@b.qfedu.com ~]# vim /usr/local/sbin/lvs_rs.sh

写入以下内容：
#/bin/bash
vip=192.168.31.200
#把vip绑定在lo上，是为了实现rs直接把结果返回给客户端
ifdown lo
ifup lo
ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up绑定vip到虚拟网卡lo:0上
route add -host $vip lo:0为lo:0网卡添加网关
#以下操作为更改arp内核参数，目的是为了让rs顺利发送mac地址给客户端
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce


例子：
#/bin/bash
vip=192.168.152.200
ifdown lo
ifup lo
ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up
route add -host $vip lo:0
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```

##### 2、给脚本设权

```shell
[root@b.qfedu.com ~]# chmod 755 /usr/local/sbin/lvs_rs.sh
```

##### 3、执行脚本

```shell
[root@b.qfedu.com ~]# sh /usr/local/sbin/lvs_rs.sh
```

##### 4、查看路由上的 vip

```shell
[root@b.qfedu.com ~]# route -n
```

##### 5、查看网卡 lo 上的 vip

```shell
[root@b.qfedu.com ~]# ip addr
```

#### 3、C: real server

（web服务器） 内网网卡：192.168.31.130 网关改回129.168.31.2

安装nginx，并启动，在默认主页里写入，real server 2 关闭selinux，清空防火墙规则

##### 1、创建转发脚本

```shell
[root@c.qfedu.com ~]# vim /usr/local/sbin/lvs_rs.sh
```

写入以下内容：

```shell
[root@c.qfedu.com ~]# yum -y install net-tools

[root@c.qfedu.com ~]# vim /usr/local/sbin/lvs_rs.sh

写入以下内容：
#/bin/bash
vip=192.168.31.200
#把vip绑定在lo上，是为了实现rs直接把结果返回给客户端
ifdown lo
ifup lo
ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up绑定vip到虚拟网卡lo:0上
route add -host $vip lo:0为lo:0网卡添加网关
#以下操作为更改arp内核参数，目的是为了让rs顺利发送mac地址给客户端
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce


例子：
#/bin/bash
vip=192.168.152.200
ifdown lo
ifup lo
ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up
route add -host $vip lo:0
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```

##### 2、给脚本设权

```shell
[root@c.qfedu.com ~]# chmod 755 /usr/local/sbin/lvs_rs.sh
```

##### 3、执行脚本

```shell
[root@c.qfedu.com ~]# sh /usr/local/sbin/lvs_rs.sh
```

##### 4、查看路由上的 vip

```shell
[root@c.qfedu.com ~]# route -n
```

##### 5、查看网卡 lo 上的 vip

```shell
[root@c.qfedu.com ~]# ip addr
```

##### 6、测试

测试1：

浏览器里访问192.168.31.200，（vip:vitrual ip）多刷新几次看结果，服务器的切换。

浏览器上因为有本地缓存的原因，虽已经设定了登陆保持时限为1秒，但每次刷新都会保持在real server 2主机上。可以在调度机里用 curl 192.168.31.200 测试访问，调度算法采用rr，效果更明显。

测试2：

关闭其中一台real server上的nginx，再次在浏览器上查看real server的切换。

测试3：

重启开启关闭的nginx，再次在浏览器查看real server的切换。



### 5、keepalived脑裂

```
脑裂  split barin：
Keepalived的BACKUP主机在收到不MASTER主机报文后就会切换成为master，如果是它们之间的通信线路出现问题，无法接收到彼此的组播通知，但是两个节点实际都处于正常工作状态，这时两个节点均为master强行绑定虚拟IP，导致不可预料的后果，这就是脑裂。
解决方式:
1、添加更多的检测手段，比如冗余的心跳线（两块网卡做健康监测），ping对方等等。尽量减少"裂脑"发生机会。(指标不治本，只是提高了检测到的概率)；
2、设置仲裁机制。两方都不可靠，那就依赖第三方。比如启用共享磁盘锁，ping网关等。(针对不同的手段还需具体分析)；
3、爆头，将master停掉。然后检查机器之间的防火墙。网络之间的通信
```

