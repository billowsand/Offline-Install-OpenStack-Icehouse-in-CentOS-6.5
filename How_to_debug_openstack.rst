===========
OpenStack Icehouse 调试手册
===========

:版本: 1.0
:关键字: devstack,eclipse,debug.


OpenStack的从安装使用到开发是一件极富有挑战性的事情。其中，从源码调试和开发OpenStack是其中最为困难的部分。本文主要根据自己的开发经验，讲解怎样在OpenStack上搭建开发环境。


系统环境准备
===========

操作系统
------------

* 在OpenStack老的官网上提供了在mac下利用虚拟环境调试OpenStack的方法，本人经过实验，极其复杂，故不推荐在mac上直接使用，而推荐使用虚拟机的方法安装。对于开发者而言，推荐使用fedora xface版本惊醒开发，界面简单快速。当然使用xubantu也可以。不推荐使用ubuntu，应为Unity在虚拟机环境下运行效果不好。

* 为了省去之后的麻烦，注意将用户名直接设为\ **stack**\ 。

* 安装系统完成后最好升级系统

::
  
  sudo yum update
  sudo yum upgrade
  
  sudo apt-get update
  sudo apt-get upgrade


网络设置
-----------

* OpenStack的网卡设置很重要，尽量使用传统的eth0命名网卡。在fedora下使用修改网卡名称。

::
  
  ifrename -i 原网卡名 -n eth0



* 在ubuntu下通过配置udev来更改网卡名称，这方面教程很多。

* 网卡使用固定的IP配置


配置python源
-----------------

由于国内使用pypi经常无法访问，所以使用豆瓣源加速python包的安装::
  
  mkdir ~/.pip
  cat > ~/.pip/pip.conf << EOF
  [global]
  index-url = http://pypi.douban.com/simple
  EOF

配置防火墙和selinux
------------------

对于开发环境而言，最好的办法，关了::
  
   sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
   sudo setenforce 0
   sudo service iptables stop
   sudo chkconfig iptables off

安装软件包
---------------

* 安装git

::
 
  sudo yum install git
  sudo apt-get install git


* 安装eclipse
::
  
  sudo yum install eclipse
  sudo apt-get install eclipse

* 安装pydev，在eclipse中\ **帮助**\ 中的\ **安装新软件**\，中加入 http://pydev.org/update。安装\ **Pydev**\ 。


DevStack的安装和配置
===========

* 从github上安装下载devstack，新版的github不需要创建stack账户

::
  
  cd 
  git clone https://github.com/openstack-dev/devstack.git


* 定制devstack，创建一个localrc文件，写入下面内容

::
  
 # 基本配置信息
 HOST_IP=[你的主机IP]
 DATABASE_PASSWORD=password
 ADMIN_PASSWORD=password
 SERVICE_PASSWORD=password
 SERVICE_TOKEN=password
 RABBIT_PASSWORD=password
 
 #国内用户最好使用github
 GIT_BASE=https://github.com
 
 ## vnc

 #enable_service n-spice
 #enable_service n-novnc
 #enable_service n-xvnc

 # Reclone each time
 #RECLONE=yes
 RECLONE=no

 ## For Keystone
 KEYSTONE_TOKEN_FORMAT=PKI

 ## For Swift
 #SWIFT_REPLICAS=1
 #SWIFT_HASH=011688b44136573e209e
 
 # Enable Logging
 LOGFILE=/opt/stack/logs/stack.sh.log
 VERBOSE=True
 LOG_COLOR=True
 SCREEN_LOGDIR=/opt/stack/logs
 
 # Pre-requisite
 ENABLED_SERVICES=rabbit,mysql,key
 
 ## If you want ZeroMQ instead of RabbitMQ (don't forget to un-declare 'rabbit' from the pre-requesite)
 #ENABLED_SERVICES+=,-rabbit,-qpid,zeromq
 
 ## If you want Qpid instead of RabbitMQ (don't forget to un-declare 'rabbit' from the pre-requesite)
 #ENABLED_SERVICES+=,-rabbit,-zeromq,qpid
 
 # Horizon (Dashboard UI) - (always use the trunk)
 ENABLED_SERVICES+=,horizon
 HORIZON_REPO=https://github.com/openstack/horizon
 HORIZON_BRANCH=master
 
 # Nova - Compute Service
 ENABLED_SERVICES+=,n-api,n-crt,n-obj,n-cpu,n-cond,n-sch
 
 ######vnc
 ENABLED_SERVICES+=,n-novnc,n-xvnc
 
 IMAGE_URLS+=",https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img"
 
 
 # Nova Network - If you don't want to use Neutron and need a simple network setup (old good stuff!)
 #ENABLED_SERVICES+=,n-net
 
 ## Nova Cells
 ENABLED_SERVICES+=,n-cell
 
 # Glance - Image Service
 ENABLED_SERVICES+=,g-api,g-reg
 
 # Swift - Object Storage
 #ENABLED_SERVICES+=,s-proxy,s-object,s-container,s-account
 
 # Neutron - Networking Service
 # If Neutron is not declared the old good nova-network will be used
 ENABLED_SERVICES+=,q-svc,q-agt,q-dhcp,q-l3,q-meta,neutron
 
 ## Neutron - Load Balancing
 ENABLED_SERVICES+=,q-lbaas
 
 ## Neutron - VPN as a Service
 ENABLED_SERVICES+=,q-vpn
 
 ## Neutron - Firewall as a Service
 ENABLED_SERVICES+=,q-fwaas
 
 # VLAN configuration
 #Q_PLUGIN=ml2
 #ENABLE_TENANT_VLANS=True
 
 # GRE tunnel configuration
 Q_PLUGIN=ml2
 ENABLE_TENANT_TUNNELS=True
 
 # VXLAN tunnel configuration
 #Q_PLUGIN=ml2
 #Q_ML2_TENANT_NETWORK_TYPE=vxlan   
 
 # Cinder - Block Device Service
 VOLUME_GROUP="cinder-volumes"
 ENABLED_SERVICES+=,cinder,c-api,c-vol,c-sch,c-bak
 
 # Heat - Orchestration Service
 ENABLED_SERVICES+=,heat,h-api,h-api-cfn,h-api-cw,h-eng
 
 # Ceilometer - Metering Service (metering + alarming)
 ENABLED_SERVICES+=,ceilometer-acompute,ceilometer-acentral,ceilometer-collector,ceilometer-api
 ENABLED_SERVICES+=,ceilometer-alarm-notify,ceilometer-alarm-eval
 
 # Apache fronted for WSGI
 #APACHE_ENABLED_SERVICES+=keystone,swift
 APACHE_ENABLED_SERVICES+=keystone
 

* 配置环境变量，由于devstack可能不能最后创建环境变量文件

::
 
 cat >  admin_key << EOF
 export OS_USERNAME=admin
 export OS_TENANT_NAME=admin
 export OS_PASSWORD=password
 export OS_AUTH_URL=your ip
 EOF

* 运行devstack，由于网络原因，往往一次不能完全成功，需要多运行几次

::
  
  cd devstack && ./stack.sh


* 在安装完成后可以使用

::
  
  ＃停止服务
  ./unstack.sh
 
  ＃开启服务
  ./rejoin_stack.sh


在Eclipse下调试OpenStack
=====================

