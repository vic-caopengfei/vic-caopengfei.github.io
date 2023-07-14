---
title: Greenplum 安装教程
author: vic
date: 2023-03-22 00:34:00 +0800
categories: [Blogging, DB]
tags: [favicon]
typora-root-url: ../../
---

## 一、安装步骤
###  准备
1. github地址：https://gp-docs-cn.github.io/docs/
2. 下载地址：
 * 商业版：https://network.pivotal.io/products/pivotal-gpdb/
 * 社区版： https://github.com/greenplum-db/gpdb/releases

### 安装


###  第一步：修改host文件，所有节点机器都要修改
 **vi /etc/hosts**
```
172.17.13.160 v1 gpmaster
172.17.13.161 v2 gpsegment1
172.17.13.162 v3 gpsegment2
```
###  第二步：创建用户和用户组（所有机器都要修改）
```
 创建用户组命令：groupadd -g 530 gpadmin 
 创建用户命令：useradd -g 530 -u530 -m -d /home/gpadmin -s /bin/bash gpadmin 
 修改密码命令：passwd gpadmin

```
###  第三步：修改系统内核（所有机器都要修改）
 **vi /etc/sysctl.conf** 
```
kernel.shmmax = 500000000
kernel.shmmni = 4096
kernel.shmall = 4000000000
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ip_local_port_range = 1025 65535
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.overcommit_memory = 2

```
 **立马生效** 
`sysctl -p`

###  第四步：修改文件打开限制: （每台机子都要修改）
 **vi /etc/security/limits.conf** 
```
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 13107

```

###  第五步：关闭防火墙：（每台机子都要修改）
```
启动： systemctl start firewalld
关闭： systemctl stop firewalld
查看状态： systemctl status firewalld 
开机禁用  ： systemctl disable firewalld
开机启用  ： systemctl enable firewalld
```

###  第六步：关闭SELINUX （每台机子都要修改）
 **vi /etc/selinux/config** 
```
SELINUX=disabled
```

###  第七步：设置时区 （每台机子都要修改）
```
timedatectl set-timezone Asia/Shanghai 
```

###  第八步：创建安装文件目录（每台机子都要修改）
```
mkdir /opt/greenplum
chown -R gpadmin:gpadmin /opt/greenplum 
```

###  第九步：上传安装包到master（master上执行）
```

```

###  第10步：在master上安装greenplum（master上执行，用root用户装）
```
赋权命令:   chmod +x greenplum-db-5.11.3-rhel7-x86_64.bin
执行安装命令： ./greenplum-db-5.11.3-rhel7-x86_64.bin
安装过程中修改安装目录：/opt/greenplum/greenplum-db
安装成功后：安装目录的权限修改为gpadmin  命令如下：
命令： chown -R gpadmin:gpadmin /opt/greenplum
```
 **![成功截图如下](/assets/img/post_image/143508_a6582583_1625231.webp"TIM图片20190331143456.webp")** 

###  第11步：创建配置文件（master上执行，用gpadmin用户）
```
vi ./conf/hostlist(新增文件)    （在这个目录/opt/greenplum/conf，需要创建conf文件夹）

gpmaster
gpsegment1
gpsegment2

 vi ./conf/seg_hosts(新增文件)

gpsegment1
gpsegment2

```

###  第12步：打通所有节点 (master上执行，用gpadmin用户，注意：此步骤如果打通失败，需要重启机器后再执行下面命令)
```
source /opt/greenplum/greenplum-db/greenplum_path.sh
gpssh-exkeys -f /opt/greenplum/conf/hostlist   （注意当前路径）
```
 **显示   [INFO] completed successfully  即打通成功** 
![输入图片说明](/assets/img/post_image/144329_82b2a60c_1625231.webp "22228.webp")

 **测试节点是否打通成功**

```
gpssh -f /opt/greenplum/conf/hostlist
pwd

```
成功截图如下：

![输入图片说明](/assets/img/post_image/144508_801a5c58_1625231.webp "333331.webp")

###  第13步：将安装包分发到每个子节点（master上执行，用gpadmin用户）
```
cd /opt/greenplum
tar -cf gp.tar greenplum-db/
gpscp -f /opt/greenplum/conf/hostlist gp.tar =:/opt/greenplum/   （复制到每台机器命令） 

批量复制成功后去segment系统查看文件是否存在 ，如果存在执行以下命令解压

 gpssh -f /opt/greenplum/conf/hostlist
     => cd /opt/greenplum
     => tar -xf gp.tar
     => ll （可以查看是否安装成功）
     => exit
到此所有节点安装完成

```
成功截图如下
![输入图片说明](/assets/img/post_image/145811_f58098fc_1625231.webp "屏幕截图.webp")



###  第14步：初始化数据库（master上执行，用gpadmin用户）

批量创建greenplum数据存放目录  如：/home/gpadmin/gpdata/gpmaster 
```
 命令:  gpssh -f  /opt/greenplum/conf/hostlist
    => cd /home/gpadmin
    => mkdir gpdata
    => cd gpdata
    => mkdir gpmaster gpdatap1 gpdatap2 gpdatam1 gpdatam2 
    => ll
    => exit
 
```
成功截图如下
![输入图片说明](/assets/img/post_image/150132_890bdbd6_1625231.webp "屏幕截图.webp")


###  第15步： 配置.bash_profile环境变量（每台机器都需要修改）

 **vi /home/gpadmin/.bash_profile** 
```
 新增以下内容：
source /opt/greenplum/greenplum-db/greenplum_path.sh
export MASTER_DATA_DIRECTORY=/home/gpadmin/gpdata/gpmaster/gpseg-1
export PGPORT=5432

```
 **立马生效** 

`source /home/gpadmin/.bash_profile`


###  第16步： 初始化配置文件（master上执行，用gpadmin用户）

 **vi  /opt/greenplum/conf/gpinitsystem_config** 
```
ARRAY_NAME="Greenplum"
SEG_PREFIX=gpseg
PORT_BASE=33000
declare -a DATA_DIRECTORY=(/home/gpadmin/gpdata/gpdatap1 /home/gpadmin/gpdata/gpdatap2)
MASTER_HOSTNAME=gpmaster
MASTER_DIRECTORY=/home/gpadmin/gpdata/gpmaster
MASTER_PORT=5432
TRUSTED_SHELL=/usr/bin/ssh
MIRROR_PORT_BASE=43000
REPLICATION_PORT_BASE=34000
MIRROR_REPLICATION_PORT_BASE=44000
declare -a MIRROR_DATA_DIRECTORY=(/home/gpadmin/gpdata/gpdatam1 /home/gpadmin/gpdata/gpdatam2)
MACHINE_LIST_FILE=/opt/greenplum/conf/seg_hosts


```

###  第17步： 初始化数据库（master上执行，用gpadmin用户）
 **批量初始化命令** 
```
gpinitsystem -c /opt/greenplum/conf/gpinitsystem_config -h /opt/greenplum/conf/hostlist

```
 **单库初始化命令** 
```
gpinitsystem -c /opt/greenplum/conf/gpinitsystem_config -s gpmaster

```


###  第18步： 设置访问白名单（master上执行，用gpadmin用户）
vi /home/gpadmin/gpdata/gpmaster/gpseg-1/pg_hba.conf

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
 host    all            all             10.10.56.17/24             trust

```
修改配置生效
（master上执行，用gpadmin用户）
gpstop -u  
### 常用命令
停止数据库  gpstop -M fast -a
启动数据库  gpstart -m

 **备注** 

 **1. 需要安装命令ifconfig、netstat** 


## 2、 常见问题处理
### 2.1维护问题
#### 2.1.1  白名单pg_hba.conf文件配置错误，导致数据库无法重启。
* 现象： 数据库启动，在这一句卡住了，不动。
```
20200413:11:13:08:006443 gpstart:gpmaster:gpadmin-[INFO]:-Starting Master instance in admin mode

```
* 排查步骤： 参考：https://blog.csdn.net/csdnhsh/article/details/95789004 进行排查。
* 原因： /home/gpadmin/gpdata/gpmaster/gpseg-1/pg_hba.conf 中的白名单IP有问题导致，删掉有问题的IP即可

#### 2.1.1  gp某个seg启动失败
* 现象：gp某个segment启动异常。
gpstate -m
![输入图片说明](/assets/img/post_image/191910_0de18b0a_5021824.webp "屏幕截图.webp")
gpstate -m
![输入图片说明](/assets/img/post_image/191943_5891beb3_5021824.webp "屏幕截图.webp")

* 原因：服务器宕机重启后，启动异常，经查看，硬盘存储不够，segment恢复失败
* 解决：增加一部分内存，然后手动恢复seg
 产生一个恢复文件：gprecoverseg -o ./recov
  查看需要恢复的seg： cat recov
  进行恢复： gprecoverseg -i ./recov
  查看恢复状态： gpstate -m
#### 2.1.2 备份恢复
```
 使用 gprecoverseg -F
```
* 错误：
![输入图片说明](/assets/img/post_image/134029_385199e1_303502.webp "屏幕截图.webp")
* 提示 ： perl: command not found
  *解决办法：
  
   > yum -y install perl perl-devel 安装依赖即可 所有的服务器都要安装
* 解决mirror和primary 互换的问题  
`
gprecoverseg -r
`
* 报错：-gprecoverseg failed. (Reason='Some segments are not yet synchronized.  All segments must be synchronized to rebalance.') exiting...
![输入图片说明](/assets/img/post_image/135921_c98ef2f4_303502.webp "屏幕截图.webp")
* 原因 这时候节点正在恢复。需要等到恢复完成即可

