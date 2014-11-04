---
layout: post
title: "Install OpenStack Icehouse Offline"
date: 2014-06-17 16:21:14 +0800
comments: true
categories:
---

# 概述

这是一个可以提供测试或者实验的手册，可以在虚拟机和物理机上使用，本指南将不再提供虚拟机使用方面的指导，仅仅叙述安装过程。安装过程将在一台双网卡的机器上安装OpenStack中的Keysotn，Glance，nova和Neutron组件。如果需要将服务运行在多个节点就更改对应服务里的IP设置即可。


# 安装需求


## 系统准备


下载[CentOS 6.5][id] 镜像

[id]:http://mirrors.163.com/centos/6.5/isos/x86_64/CentOS-6.5-x86_64-bin-DVD1.iso


安装时选择 **Virtualization Host** ，同时勾选 **custom_now** ,选择 **Virtualization** 的全部安装包



## 网络准备


### 修改网卡名称为eth*

由于很多机器的网卡名称不一样，为此需要统一网卡名称，进入

    vim /etc/udev/rules.d/70-persistent-net.rules


    #修改网卡名称,仅仅需要修改NAME
    SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="你的网卡MAC地址", ATTR{type}=="1", KERNEL=="eth*", NAME="eth1
    SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="你的网卡MAC地址", ATTR{type}=="1", KERNEL=="eth*", NAME="eth0"

### 配置网卡的IP地址
在network中添加网卡eth1

    cat > /etc/sysconfig/network-scripts/ifcfg-eth1<<EOF
    DEVICE=eth1
    TYPE=Ethernet
    ONBOOT=yes
    NM_CONTROLLED=no
    BOOTPROTO=static
    IPADDR=192.168.138.77
    NETMASK=255.255.255.0
    EOF

在network中添加网卡eth0

    cat > /etc/sysconfig/network-scripts/ifcfg-eth0 << EOF
    DEVICE=eth0
    TYPE=Ethernet
    ONBOOT=yes
    NM_CONTROLLED=no
    BOOTPROTO=static
    EOF

**重启系统**

    reboot




# 安装前系统配置


## 系统配置

修改主机名

    hostname controller


使能IP路由转发和桥上的iptables

    vim /etc/sysctl.conf

    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-arptables = 1


打开文件最大数

    vim /etc/security/limits.conf

    *               soft     nproc           65535
    *               hard    nproc           65535
    *               soft    nofile            65535
    *               hard    nofile           65535
    *               soft     core            ulimit
    *               hard    core            ulimit


关闭selinux:

    vim /etc/selinux/config

    SELINUX=disabled

**重启系统**:

    reboot

删除libvirt自带的bridge，准备使用openvswitch

     mv /etc/libvirt/qemu/networks/default.xml /etc/libvirt/qemu/networks/default.xml.bak
     modprobe  -r bridge


##软件仓库配置

### 获取离线安装包
拷贝openstack_centos 到工作目录（默认为root）


### 配置本地源用于安装httpd服务
删除现有的源:

    rm /etc/yum.repos.d/*

建立本地源（**注意不要通过网络安装任何包，容易造成包的版本不一致**）

    cat > /etc/yum.repos.d/centos65.repo<< EOF
    [base]
    name=Base
    baseurl=file:///root/openstack_centos/centos65
    gpgcheck=1
    enabled=1
    gpgkey=file:///root/openstack_centos/centos65/RPM-GPG-KEY-CentOS-6
    EOF


安装httpd服务:

    yum install httpd


### 建立基于http的源用于局域网内多个节点的安装

将**openstack_centos**目录下的**rdo**包，**epel**包，**rabbitmq**，以及光盘里的源包拷贝到目录/var/www/html/

    cd /root/openstack_centos/
    tar zxvf epel-depends.tar.gz -C /var/www/html/
    tar zxvf openstack-icehouse.tar.gz -C /var/www/html
    tar zxvf rabbitmq.tar.gz -C /var/www/html
    cp -r centos65/ /var/www/html/

删除之前的本地源文件

    rm /etc/yum.repos.d/centos65.repo


建立新的repo文件

    cat > /etc/yum.repos.d/centos65.repo<< EOF
    [base]
    name=Base
    baseurl=http://192.168.138.77/centos65
    gpgcheck=1
    enabled=1
    gpgkey=http://192.168.138.77/centos65/RPM-GPG-KEY-CentOS-6
    EOF


    cat > /etc/yum.repos.d/epel.repo<< EOF
    [epel]
    name=epel
    baseurl=http://192.168.138.77/epel-depends
    enabled=1
    gpgcheck=0
    EOF


    cat > /etc/yum.repos.d/rabbitmq.repo<< EOF
    [rabbitmq]
    name=rabbitmq
    baseurl=http://192.168.138.77/rabbitmq
    enabled=1
    gpgcheck=0
    EOF


    cat > /etc/yum.repos.d/icehouse.repo<< EOF
    [openstack-icehouse]
    name=openstack-icehouse
    baseurl=http://192.168.138.77/openstack-icehouse
    enabled=1
    gpgcheck=0
    EOF


重启httpd并配置为开机启动，以保证源正常工作

    service httpd start
    chkconfig httpd on

检查是否可以访问:

    curl http://192.168.138.77/rabbitmq/


##防火墙配置


添加规则

    vim /etc/sysconfig/iptables

    -I INPUT -p tcp --dport 80 -j ACCEPT
    -I INPUT -p tcp --dport 3306 -j ACCEPT
    -I INPUT -p tcp --dport 5000 -j ACCEPT
    -I INPUT -p tcp --dport 5672 -j ACCEPT
    -I INPUT -p tcp --dport 8080 -j ACCEPT
    -I INPUT -p tcp --dport 8773 -j ACCEPT
    -I INPUT -p tcp --dport 8774 -j ACCEPT
    -I INPUT -p tcp --dport 8775 -j ACCEPT
    -I INPUT -p tcp --dport 8776 -j ACCEPT
    -I INPUT -p tcp --dport 8777 -j ACCEPT
    -I INPUT -p tcp --dport 9292 -j ACCEPT
    -I INPUT -p tcp --dport 9696 -j ACCEPT
    -I INPUT -p tcp --dport 15672 -j ACCEPT
    -I INPUT -p tcp --dport 55672 -j ACCEPT
    -I INPUT -p tcp --dport 35357 -j ACCEPT
    -I INPUT -p tcp --dport 12211 -j ACCEPT


重新启动防火墙应用规则

    /etc/init.d/iptables restart


## 安装RabbitMQ

    yum install rabbitmq-server

在hosts中添加主机名称

    vim /etc/hosts

    127.0.0.1   controller localhost.localdomain localhost4 localhost4.localdomain4


开启RabbitMQ management服务，可以通过web访问RabiitMQ

    rabbitmq-plugins enable rabbitmq_management

启动服务，并设置开机启动

    chkconfig rabbitmq-server on
    service rabbitmq-server start

验证安装是否成功

    curl  http://192.168.138.77:15672


## 安装MySQL

    yum install mysql mysql-server

修改编码格式，支持utf8，在**my.cnf**的**[mysqd]**下加入

    vim  /etc/my.cnf

    default-character-set=utf8
    default-storage-engine=InnoDB

设置开机启动，设置密码

    chkconfig mysqld on
    /etc/init.d/mysqld start
    /usr/bin/mysqladmin -u root password 'openstack'

## OpenVSwitch的安装

    rpm -ivh openvswitch-1.11.0_8ce28d-1.el6ost.x86_64.rpm

配置开机启动

    chkconfig openvswitch on
    service openvswitch start

设置默认网桥

    ovs-vsctl add-br  br-int


## 安装和升级iproute和dnsmasq

    yum install  -y iproute dnsmasq dnsmasq-utils


# 安装Keystone

    yum install -y openstack-keystone  openstack-utils


创建token，用于对于Keystone的认证

    export SERVICE_TOKEN=$(openssl rand -hex 10)
    echo $SERVICE_TOKEN >/root/ks_admin_token



## 初始化keystone
设置Keystone使用UUID认证。可以通过**openstack-config**设置，也可以直接修改**/etc/keystone/keysotne.conf**文件

    openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $SERVICE_TOKEN
    openstack-config --set /etc/keystone/keystone.conf token provider keystone.token.providers.uuid.Provider



设置数据库路径并同步数据

    openstack-config --set /etc/keystone/keystone.conf sql connection mysql://keystone:keystone@192.168.138.77/keystone
    openstack-db --init --service keystone --password keystone --rootpw openstack

修改目录访问权限并启动服务

    chown -R keystone:keystone /etc/keystone
    chkconfig openstack-keystone on
    service openstack-keystone start

## 配置Keystone服务
导入keystone service和endpoint的环境变量访问

    export  SERVICE_TOKEN=`cat /root/ks_admin_token`
    export SERVICE_ENDPOINT=http://192.168.138.77:35357/v2.0

创建keystone service和endpoint供其余组件访问

     keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
     keystone endpoint-create --service  keystone   --publicurl 'http://192.168.138.77:5000/v2.0' --adminurl 'http://192.168.138.77:35357/v2.0' --internalurl 'http://192.168.138.77:5000/v2.0' --region changsha


创建Keystone用户，Admin角色和Admin租户

   keystone user-create --name admin --pass openstack
   keystone role-create --name admin
   keystone tenant-create --name admin
   keystone user-role-add --user admin  --role admin  --tenant admin


建立环境变量用于命令行的访问

    cat >/root/keystone_admin<<EOF
    export OS_USERNAME=admin
    export OS_TENANT_NAME=admin
    export OS_PASSWORD=openstack
    export OS_AUTH_URL=http://192.168.138.77:35357/v2.0/
    export PS1='[\u@\h \W(keystone_admin)]\$ '
    EOF


创建Member角色和普通用户

    source /root/keystone_admin
    keystone role-create --name Member
    keystone user-create --name usera --pass openstack
    keystone tenant-create --name tenanta
    keystone user-role-add --user  usera --role Member --tenant tenanta

    keystone user-create --name userb --pass openstack
    keystone tenant-create --name tenantb
    keystone user-role-add --user  userb --role Member --tenant tenantb

##验证Keyston
显示创建的用户则表示成功

    keystone user-list

# Glance的安装

安装Glance服务

    yum install -y openstack-glance  openstack-utils  python-kombu python-anyjson


在keystone中创建Glance服务

    keystone service-create --name glance --type image  --description "Glance Image Service"
    keystone endpoint-create --service glance --publicurl  "http://192.168.138.77:9292" --adminurl "http://192.168.138.77:9292" --internalurl "http://192.168.138.77:9292" --region changsha


## Glance的配置
主要配置Glance与MySQL和Keystone的接口

    openstack-config --set /etc/glance/glance-api.conf  DEFAULT sql_connection mysql://glance:glance@192.168.138.77/glance
    openstack-config --set /etc/glance/glance-registry.conf  DEFAULT sql_connection mysql://glance:glance@192.168.138.77/glance
    openstack-config --set /etc/glance/glance-api.conf  paste_deploy flavor keystone
    openstack-config --set /etc/glance/glance-api.conf  keystone_authtoken auth_host 192.168.138.77
    openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_port 35357
    openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_protocol http
    openstack-config --set /etc/glance/glance-api.conf keystone_authtoken admin_tenant_name admin
    openstack-config --set /etc/glance/glance-api.conf  keystone_authtoken admin_user admin
    openstack-config --set /etc/glance/glance-api.conf  keystone_authtoken admin_password openstack
    openstack-config --set /etc/glance/glance-registry.conf  paste_deploy flavor keystone
    openstack-config --set /etc/glance/glance-registry.conf  keystone_authtoken auth_host 192.168.138.77
    openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_port 35357
    openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_protocol http
    openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken admin_tenant_name admin
    openstack-config --set /etc/glance/glance-registry.conf  keystone_authtoken admin_user admin
    openstack-config --set /etc/glance/glance-registry.conf  keystone_authtoken admin_password openstack
    openstack-config --set /etc/glance/glance-api.conf DEFAULT notifier_strategy noop



##在MySQL中初始化Glance数据库

    openstack-db --init --service glance --password glance --rootpw openstack



##设置权限并启动服务

    chown -R glance:glance /etc/glance
    chown -R glance:glance /var/lib/glance
    chown -R glance:glance /var/log/glance

    chkconfig openstack-glance-api on
    chkconfig openstack-glance-registry on

    service openstack-glance-api start
    service openstack-glance-registry start

##验证Glance服务

    source /root/keystone_admin
    glance image-list

此时不出错，无镜像列表

上传测试镜像

    glance image-create --name "cirros-0.3.1-x86_64" --disk-format qcow2 --container-format bare --is-public true --file cirros-0.3.1-x86_64-disk.img
    glance image-list

成功则显示当前上传的镜像信息

上传镜像文件存放位置： **/var/lib/glance/images**


#安装Horizon Dashboard

    yum install -y mod_wsgi httpd mod_ssl memcached python-memcached openstack-dashboard


## 配置Horizon
修改配置文件

    vim /etc/openstack-dashboard/local_settings

注释如下几行，注意**缩进**

    #CACHES = {
      #    'default': {
      #        'BACKEND' : 'django.core.cache.backends.locmem.LocMemCache'
      #    }
      #}

    #打开下面几行的注释
    CACHES = {
        'default': {
            'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
            'LOCATION' : '127.0.0.1:11211',
         }
    }


修改如下几行

    ALLOWED_HOSTS = ['*']
    OPENSTACK_HOST = "192.168.138.77"

设置权限并启动服务

    chown -R apache:apache /etc/openstack-dashboard/ /var/lib/openstack-dashboard/
    chkconfig httpd on
    chkconfig memcached on
    service httpd start
    service memcached start

##验证
在局域网内任意一台节点访问http://192.168.138.77/dashboard





#Nova的安装

    yum install  -y  openstack-nova openstack-utils python-kombu python-amqplib openstack-neutron-openvswitch dnsmasq-utils python-stevedore


##创建Nova数据库

    mysql -u root -popenstack
    CREATE DATABASE nova;
    GRANT ALL ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova';
    GRANT ALL ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'nova';
    FLUSH PRIVILEGES;
    quit;

重新启动

    service mysqld restart


##在Keystonec创建Nova服务

    keystone service-create --name compute  --type compute --description "OpenStack Compute Service"
    keystone endpoint-create --service compute --publicurl "http://192.168.138.77:8774/v2/%(tenant_id)s" --adminurl "http://192.168.138.77:8774/v2/%(tenant_id)s"  --internalurl "http://192.168.138.77:8774/v2/%(tenant_id)s" --region changsha

##配置nova

    openstack-config --set /etc/nova/nova.conf database connection mysql://nova:nova@192.168.138.77/nova
    openstack-config --set /etc/nova/nova.conf DEFAULT rabbit_host 192.168.138.77
    openstack-config --set /etc/nova/nova.conf DEFAULT my_ip 192.168.138.77
    openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_listen 0.0.0.0
    openstack-config --set /etc/nova/nova.conf  DEFAULT vnc_enabled True
    openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_proxyclient_address 192.168.138.77
    openstack-config --set /etc/nova/nova.conf DEFAULT novncproxy_base_url http://192.168.138.77:6080/vnc_auto.html
    openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
    openstack-config --set /etc/nova/nova.conf DEFAULT rpc_backend nova.openstack.common.rpc.impl_kombu
    openstack-config --set /etc/nova/nova.conf DEFAULT glance_host 192.168.137.231
    openstack-config --set /etc/nova/nova.conf DEFAULT api_paste_config /etc/nova/api-paste.ini
    openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_host 192.168.138.77
    openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_port 5000
    openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_protocol http
    openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_version v2.0
    openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_user admin
    openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_tenant_name admin
    openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_password openstack
    openstack-config --set /etc/nova/nova.conf DEFAULT enabled_apis ec2,osapi_compute,metadata
    openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
    openstack-config --set /etc/nova/nova.conf DEFAULT network_manager nova.network.neutron.manager.NeutronManager
    openstack-config --set /etc/nova/nova.conf DEFAULT service_neutron_metadata_proxy True;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_metadata_proxy_shared_secret awcloud;
    openstack-config --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_use_dhcp True;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_url http://192.168.138.77:9696;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_username admin;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_password openstack;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_tenant_name admin;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_region_name changsha;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_auth_url http://192.168.138.77:5000/v2.0;
    openstack-config --set /etc/nova/nova.conf DEFAULT neutron_auth_strategy keystone;
    openstack-config --set /etc/nova/nova.conf DEFAULT security_group_api neutron;
    openstack-config --set /etc/nova/nova.conf DEFAULT linuxnet_interface_driver nova.network.linux_net.LinuxOVSInterfaceDriver;
    openstack-config --set /etc/nova/nova.conf libvirt vif_driver nova.virt.libvirt.vif.LibvirtGenericVIFDriver;

## 配置nova启动项::

    chkconfig openstack-nova-consoleauth on
    chkconfig openstack-nova-api on
    chkconfig openstack-nova-scheduler on
    chkconfig openstack-nova-conductor on
    chkconfig openstack-nova-compute on
    chkconfig openstack-nova-novncproxy on


## 启动nova

    service openstack-nova-conductor start
    service openstack-nova-api start
    service openstack-nova-scheduler start
    service openstack-nova-compute start
    service openstack-nova-consoleauth start
    service openstack-nova-novncproxy start
注意启动的顺序

## 同步nova数据库

    nova-manage db sync


# 安装Neutron

    yum -y install openstack-neutron  python-kombu python-amqplib  python-pyudev python-stevedore openstack-utils openstack-neutron-openvswitch openvswitch

## 在数据库创建Neutron

    mysql  -u root  -popenstack
    CREATE DATABASE neutron;
    GRANT ALL ON neutron.* TO neutron @'%' IDENTIFIED BY 'neutron';
    GRANT ALL ON neutron.* TO neutron @'localhost'  IDENTIFIED BY 'neutron';
    FLUSH PRIVILEGES;
    quit;

## 在keystone创建Neutron::

    keystone service-create --name neutron  --type network --description "Neutron Networking Service"
    keystone endpoint-create --service neutron --publicurl "http://192.168.138.77:9696" --adminurl "http://192.168.138.77:9696"  --internalurl "http://192.168.138.77:9696" --region changsha



## 配置Neutron

    openstack-config --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
    openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_host 192.168.138.77
    openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_tenant_name admin
    openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_user admin
    openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_password openstack
    openstack-config --set /etc/neutron/neutron.conf DEFAULT rpc_backend neutron.openstack.common.rpc.impl_kombu
    openstack-config --set /etc/neutron/neutron.conf DEFAULT rabbit_host 192.168.138.77
    openstack-config --set  /etc/neutron/neutron.conf  DEFAULT   core_plugin neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2
    openstack-config --set  /etc/neutron/neutron.conf  DEFAULT   control_exchange neutron
    openstack-config --set  /etc/neutron/neutron.conf  database   connection  mysql://neutron:neutron@192.168.138.77/neutron
    openstack-config --set  /etc/neutron/neutron.conf  DEFAULT  allow_overlapping_ips True

## 配置Neutron插件

### OpenVSwitch
链接插件

     ln -s /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini /etc/neutron/plugin.ini

openvswitch agent的配置

    openstack-config --set  /etc/neutron/plugin.ini  OVS  tenant_network_type gre
    openstack-config --set  /etc/neutron/plugin.ini  OVS  tunnel_id_ranges 1:1000
    openstack-config --set  /etc/neutron/plugin.ini  OVS  enable_tunneling True
    openstack-config --set  /etc/neutron/plugin.ini  OVS  local_ip 192.168.138.77
    openstack-config --set  /etc/neutron/plugin.ini  OVS  integration_bridge br-int
    openstack-config --set  /etc/neutron/plugin.ini  OVS  tunnel_bridge br-tun
    openstack-config --set  /etc/neutron/plugin.ini  SECURITYGROUP  firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver


### Neutron dhcp agent

    openstack-config --set  /etc/neutron/dhcp_agent.ini  DEFAULT  interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
    openstack-config --set  /etc/neutron/dhcp_agent.ini  DEFAULT  dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
    openstack-config --set  /etc/neutron/dhcp_agent.ini  DEFAULT  use_namespaces True


### Neutron L3 agent::

配置网桥

    ovs-vsctl add-br br-ex
    ovs-vsctl add-port br-ex eth1
    ip addr add 192.168.137.231/24 dev br-ex
    ip link set br-ex up
    echo  "ip addr add 192.168.137.231/24 dev br-ex" >> /etc/rc.local

配置L3

    openstack-config --set  /etc/neutron/l3_agent.ini DEFAULT  interface_driver  neutron.agent.linux.interface.OVSInterfaceDriver
    openstack-config --set  /etc/neutron/l3_agent.ini DEFAULT  user_namespaces True
    openstack-config --set /etc/neutron/l3_agent.ini DEFAULT external_network_bridge br-ex
    openstack-config --set /etc/neutron/l3_agent.ini DEFAULT enable_metadata_proxy True;



### Neutron metadata


    vim /etc/neutron/metadata_agent.ini
注释**auth_region**这一行

    openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  auth_region changsha
    openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  auth_url http://192.168.138.77:35357/v2.0
    openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  admin_tenant_name admin
    openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  admin_user admin
    openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  admin_password openstack
    openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  nova_metadata_ip 192.168.138.77
    openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  metadata_proxy_shared_secret  dashanshu




## 配置Neutron启动项

    chkconfig neutron-openvswitch-agent on
    chkconfig neutron-server on
    chkconfig neutron-openvswitch-agent on
    chkconfig neutron-dhcp-agent on
    chkconfig neutron-l3-agent on
    chkconfig  neutron-metadata-agent  on


## 启动Neutron

    service neutron-openvswitch-agent start
    service neutron-dhcp-agent start
    service neutron-l3-agent start
    service neutron-metadata-agent start
    service neutron-server start
    service neutron-openvswitch-agent start



## 同步Nuetron数据库

    neutron-db-manage --config-file /usr/share/neutron/neutron-dist.conf --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head;




## 重新启动服务::

    service openstack-nova-conductor restart
    service openstack-nova-api restart
    service openstack-nova-scheduler restart
    service openstack-nova-compute restart
    service openstack-nova-consoleauth restart
    service openstack-nova-novncproxy restart

    service neutron-openvswitch-agent restart
    service neutron-dhcp-agent restart
    service neutron-l3-agent restart
    service neutron-metadata-agent restart
    service neutron-server restart
    service neutron-openvswitch-agent restart


## 验证

执行

    nova service list
    neutron service list

不报错，且每个服务显示一个**笑脸**即可


如果报错，重新同步nova和Nuetron数据库::



# 在Dashboard中使用OpenStack


在浏览器中访问http://192.168.138.77/dashboard, 用户名**admin**，密码**openstack**


#错误排查


## 日志文件存放位置

*  **horizon**: /var/log/horizon /var/log/httpd
*  **nova**: /var/log/nova
*  **glance**:/var/log/glance
*  **neutron**:/var/log/neutron

## nova boot VIF creation fails

修改nova.conf文件

    vif_plugging_timeout=10
    vif_plugging_is_fatal=False

## libvrit无法连通

重启massagebus服务

    service massagebus restart

##Live Migration
