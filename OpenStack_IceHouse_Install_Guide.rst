==========================================================
  OpenStack Icehouse 安装指南
==========================================================

:Version: 1.0
:Source: https://github.com/billowsand/Offline-Install-OpenStack-Icehouse-in-CentOS-6.5/blob/master/OpenStack_IceHouse_Install_Guide.rst
:Keywords: Multi node, Single node,Icehouse, Neutron, Nova, Keystone, Glance, Horizon, OpenVSwitch, KVM, CentOS 6.5 (64 bits).

Authors
==========

Siyang Li


Table of Contents
=================

::

  0. What is it?
  1. Requirements
  2. Controller Node
  3. Network Node
  4. Compute Node
  5. Your first VM
  6. Licensing
  7. Contacts
  8. Credits
  9. To do

概述
==============

这是一个可以提供测试或者实验的手册，可以在虚拟机和物理机上使用，本指南将不再提供虚拟机使用方面的指导，仅仅叙述安装过程。安装过程将在一台双网卡的机器上安装OpenStack的组建。如果需要将服务运行在多个节点就更改对应服务里的IP设置即可。


安装需求
====================

系统准备
-----------------

* 下载CentOS 6.5 镜像 <http://mirrors.163.com/centos/6.5/isos/x86_64/CentOS-6.5-x86_64-bin-DVD1.iso>
* 安装时选择\ *Virtualization Host*\，同时勾选\ *custom_now*\ ,选择\ *Virtualization*\ 的全部安装包



网络准备
-------------------

对于很多机器的网卡名称不一样，为此需要统一网卡名称，进入::
 
 vim /etc/udev/rules.d/70-persistent-net.rules
 #修改网卡名称,仅仅需要修改NAME
 SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="你的网卡地址", ATTR{type}=="1", KERNEL=="eth*", NAME="eth1
 SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="你的网卡地址", ATTR{type}=="1", KERNEL=="eth*", NAME="eth0"

在network中添加网卡::
 
 vim /etc/sysconfig/network-scripts/ifcfg-eth1

 ＃配置如下
 DEVICE=eth0
 TYPE=Ethernet
 ONBOOT=yes
 NM_CONTROLLED=no
 BOOTPROTO=static
 IPADDR=192.168.138.77
 NETMASK=255.255.255.0

::
 
 vim /etc/sysconfig/network-scripts/ifcfg-eth0
 DEVICE=eth1
 TYPE=Ethernet
 ONBOOT=yes
 NM_CONTROLLED=no
 BOOTPROTO=static

**重启系统**

* 网卡配置如下

:Node Role: NICs
:Allinone Node:  eth1 (192.168.137.77)

离线安装包准备
-----------------------

rdo 包下载::
 
 wget -S http://rdo.fedorapeople.org/openstack/openstack-havana/rdo-release-havana.rpm
 wget -S -c -r -np -L http://repos.fedorapeople.org/repos/openstack/openstack-icehouse/epel-6/


epel 包下载::

 wget -S -c -r -np -Lhttp://dl.fedoraproject.org/pub/epel/6/x86_64/


RabbitMQ 包下载::

 wget khlkh


OpenVSwich 包下载::

 wget gohk


测试镜像下载::

 wget hlhjkh



安装前系统配置
=======

系统配置
------------

修改主机名::

 hostname controller


使能IP路由转发和桥上的iptables::
 
 vim /etc/sysctl.conf

 net.ipv4.ip_forward = 1
 net.bridge.bridge-nf-call-ip6tables = 1
 net.bridge.bridge-nf-call-iptables = 1
 net.bridge.bridge-nf-call-arptables = 1


打开文件最大数::
 
 vim /etc/security/limits.conf

 *               soft     nproc           65535 
 *               hard    nproc           65535
 *               soft    nofile            65535
 *               hard    nofile           65535
 *               soft     core            ulimit 
 *               hard    core            ulimit


关闭selinux::

 vim /etc/selinux/config

 SELINUX=disabled

**重启系统**

删除libvirt自带的bridge，准备使用openvswitch::

 mv /etc/libvirt/qemu/networks/default.xml /etc/libvirt/qemu/networks/default.xml.bak
 modprobe  -r bridge


软件仓库配置
------------------

将CentOS 6.5光盘挂载到服务器上::

 mount  /dev/cdrom  /mnt

删除CentOS自带的repo::
 
 rm /etc/yum.repo.d/*

建立本地源::

 vim /etc/yum.repo.d/centos65

 [base]
 name=Base
 baseurl=file:///mnt/
 gpgcheck=1
 enabled=1
 gpgkey=http://mnt/RPM-GPG-KEY-CentOS-6

安装httpd服务::
 
 yum install httpd

将下载的rdo包，epel包，rabbitmq包拷贝到目录/var/www/html/::

 mkdir /var/www/html/epel-depends/
 cp -r epel-depends/* /var/www/html/epel-depends/
 mkdir /var/www/html/openstack-icehouse
 cp -r openstack-icehouse/* /var/www/html/openstack-icehouse/
 mkdir /var/www/html/rabbitmq
 cp -r rabbitmq/* /var/www/html/rabbitmq/
 cp rdo-releas/RPM-GPG-KEY-RDO-Icehouse /var/www/html/openstack-icehouse


拷贝CentOS光盘源::

 mkdir /var/www/html/centos65
 cp -r /mnt/* /var/www/html/centos65/



建立新的repo文件::
 
 cp *.repo /etc/yum.repo.d/


重新启动httpd::

 service httpd start
 chkconfig httpd on

#检查是否可以访问
 curl http://192.168.138.77/rabbitmq/

防火墙配置
-------------------

添加规则::

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


重新启动防火墙::
 /etc/init.d/iptables restart

安装RabbitMQ
----------------------

安装RabbitMQ软件::
 
 rpm --import /var/www/html/rabbitmq rabbitmq-signing-key-public.asc
 yum install rabbitmq-server

在hosts中添加主机名称::
 
 vim /etc/hosts

 127.0.0.1   controller localhost.localdomain localhost4 localhost4.localdomain4


建立plugin文件::

 echo "[rabbitmq_management].">/etc/rabbitmq/enabled_plugins

启动服务::
 
 chkconfig rabbitmq-server on
 service rabbitmq-server start
 #验证
 curl  http://192.168.138.77:15672

安装MySQL
-----------------

安装软件::
 
 yum install mysql mysql-server

修改编码格式::
 
 vim  /etc/my.cnf

 #在[mysqd]下加入
 default-character-set=utf8
 default-storage-engine=InnoDB

启动并设置密码::
 
 chkconfig mysqld on
 /etc/init.d/mysqld start

 /usr/bin/mysqladmin -u root password 'openstack'


安装Keystone
===========

::
 
 rpm  --import  /var/www/html/rdo-icehouse-b3RPM-GPG-KEY-RDO-Icehouse 
 yum install -y openstack-keystone  openstack-utils


创建token::

 export SERVICE_TOKEN=$(openssl rand -hex 10)
 echo $SERVICE_TOKEN >/root/ks_admin_token


使用UUID认证::
 
 openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $SERVICE_TOKEN
 openstack-config --set /etc/keystone/keystone.conf token provider keystone.token.providers.uuid.Provider;
 openstack-config --set /etc/keystone/keystone.conf sql connection mysql://keystone:keystone@192.168.138.77/keystone;


同步数据::
 
 openstack-db --init --service keystone --password keystone --rootpw openstack;

修改目录属性并启动服务::
 
 chown -R keystone:keystone /etc/keystone
 chkconfig openstack-keystone on
 service openstack-keystone start

创建keystone service和endpoint::
 
 export  SERVICE_TOKEN=`cat /root/ks_admin_token`
 export SERVICE_ENDPOINT=http://192.168.138.77:35357/v2.0

 keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
 keystone endpoint-create --service  keystone   --publicurl 'http://192.168.138.77:5000/v2.0' --adminurl 'http://192.168.138.77:35357/v2.0' --internalurl 'http://192.168.138.77:5000/v2.0' --region beijing

创建Keystone用户，Admin角色和Admin租户::
 
 keystone user-create --name admin --pass openstack
 keystone role-create --name admin
 keystone tenant-create --name admin
 keystone user-role-add --user admin  --role admin  --tenant admin

建立环境变量::
 
 vim /root/keystone_admin

 export OS_USERNAME=admin
 export OS_TENANT_NAME=admin
 export OS_PASSWORD=openstack
 export OS_AUTH_URL=http://192.168.138.77:35357/v2.0/
 export PS1='[\u@\h \W(keystone_admin)]\$ '

创建Member角色和普通用户::
 
 source /root/keystone_admin
 keystone role-create --name Member
 keystone user-create --name usera --pass openstack
 keystone tenant-create --name tenanta
 keystone user-role-add --user  usera --role Member --tenant tenanta
 
 keystone user-create --name userb --pass openstack
 keystone tenant-create --name tenantb
 keystone user-role-add --user  userb --role Member --tenant tenantb

 #验证
 keystone user-list

安装Glance
=========

::
 
 yum install -y openstack-glance  openstack-utils  python-kombu python-anyjson


在keystone中创建Glance::
 
 keystone service-create --name glance --type image  --description "Glance Image Service"
 keystone endpoint-create --service glance --publicurl  "http://192.168.138.77:9292" --adminurl "http://192.168.138.77:9292" --internalurl "http://192.168.138.77:9292" --region beijing


配置Glance::
 
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
 openstack-config --set /etc/glance/glance-api.conf DEFAULT notifier_strategy noop;


配置Glance数据库::
 
 openstack-db --init --service glance --password glance --rootpw openstack;


设置权限并启动服务::
 
 chown -R glance:glance /etc/glance
 chown -R glance:glance /var/lib/glance
 chown -R glance:glance /var/log/glance
 
 chkconfig openstack-glance-api on
 chkconfig openstack-glance-registry on
 
 service openstack-glance-api start
 service openstack-glance-registry start
 
 #验证
 source /root/keystone_admin
 glance image-list

上传测试镜像::
 glance image-create --name "cirros-0.3.1-x86_64" --disk-format qcow2 --container-format bare --is-public true --file cirros-0.3.1-x86_64-disk.img 
 glance image-list

镜像文件存放位置： /var/lib/glance/images


安装Horizon Dashboard
===================

::
 
 rpm  --import  /var/www/html/epel-depends RPM-GPG-KEY-EPEL-6
 yum install -y mod_wsgi httpd mod_ssl memcached python-memcached openstack-dashboard


修改配置文件::
 
 vim /etc/openstack-dashboard/local_settings

 #注释如下几行
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
 
 #修改如下几行
 ALLOWED_HOSTS = ['*'] 
 OPENSTACK_HOST = "192.168.138.77"

设置权限并启动服务::
 
 chown -R apache:apache /etc/openstack-dashboard/ /var/lib/openstack-dashboard/;
 chkconfig httpd on 
 chkconfig memcached on
 service httpd start
 service memcached start



安装和配置OpenVSwitch
===================

::
 
 rpm -ivh openvswitch-1.11.0_8ce28d-1.el6ost.x86_64.rpm

 chkconfig openvswitch on
 service openvswitch start
 ovs-vsctl add-br  br-int


安装和升级iproute和dnsmasq
----------------------------------------

::
 
 yum install  -y iproute dnsmasq dnsmasq-utils


安装Nova
=========

安装Nova软件包::
 
 yum install  -y  openstack-nova openstack-utils python-kombu python-amqplib openstack-neutron-openvswitch dnsmasq-utils python-stevedore

安装Nova数据库::
 
 mysql -u root -popenstack
 CREATE DATABASE nova;
 GRANT ALL ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova';
 GRANT ALL ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'nova';
 FLUSH PRIVILEGES;
 quit;
 
 service mysqld restart


在Keystone安装Nova::
 
 keystone service-create --name compute  --type compute --description "OpenStack Compute Service"    
 keystone endpoint-create --service compute --publicurl "http://192.168.138.77:8774/v2/%(tenant_id)s" --adminurl "http://192.168.138.77:8774/v2/%(tenant_id)s"  --internalurl "http://192.168.138.77:8774/v2/%(tenant_id)s" --region beijing 

配置nova::
 
 openstack-config --set /etc/nova/nova.conf database connection mysql://nova:nova@192.168.138.77/nova;
 openstack-config --set /etc/nova/nova.conf DEFAULT rabbit_host 192.168.138.77;
 openstack-config --set /etc/nova/nova.conf DEFAULT my_ip 192.168.138.77;
 openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_listen 0.0.0.0;
 openstack-config --set /etc/nova/nova.conf  DEFAULT vnc_enabled True
 openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_proxyclient_address 192.168.138.77;
 openstack-config --set /etc/nova/nova.conf DEFAULT novncproxy_base_url http://192.168.138.77:6080/vnc_auto.html;
 openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone;
 openstack-config --set /etc/nova/nova.conf DEFAULT rpc_backend nova.openstack.common.rpc.impl_kombu;
 openstack-config --set /etc/nova/nova.conf DEFAULT glance_host 192.168.3.233;
 openstack-config --set /etc/nova/nova.conf DEFAULT api_paste_config /etc/nova/api-paste.ini;
 openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_host 192.168.138.77;
 openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_port 5000;
 openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_protocol http;
 openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_version v2.0;
 openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_user admin;
 openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_tenant_name admin;
 openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_password openstack;
 openstack-config --set /etc/nova/nova.conf DEFAULT enabled_apis ec2,osapi_compute,metadata;
 openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver;
 openstack-config --set /etc/nova/nova.conf DEFAULT network_manager nova.network.neutron.manager.NeutronManager;
 openstack-config --set /etc/nova/nova.conf DEFAULT service_neutron_metadata_proxy True;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_metadata_proxy_shared_secret awcloud;
 openstack-config --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_use_dhcp True;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_url http://192.168.138.77:9696;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_username admin;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_password openstack;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_tenant_name admin;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_region_name beijing;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_auth_url http://192.168.138.77:5000/v2.0;
 openstack-config --set /etc/nova/nova.conf DEFAULT neutron_auth_strategy keystone;
 openstack-config --set /etc/nova/nova.conf DEFAULT security_group_api neutron;
 openstack-config --set /etc/nova/nova.conf DEFAULT linuxnet_interface_driver nova.network.linux_net.LinuxOVSInterfaceDriver;
 openstack-config --set /etc/nova/nova.conf libvirt vif_driver nova.virt.libvirt.vif.LibvirtGenericVIFDriver;





安装Neutron
============

在数据库创建Neutron::
 
 mysql  -u root  -popenstack
 CREATE DATABASE neutron;
 GRANT ALL ON neutron.* TO neutron @'%' IDENTIFIED BY 'neutron';
 GRANT ALL ON neutron.* TO neutron @'localhost'  IDENTIFIED BY 'neutron';
 FLUSH PRIVILEGES;
 quit;

在keystone创建Neutron::
 
 keystone service-create --name neutron  --type network --description "Neutron Networking Service"
 keystone endpoint-create --service neutron --publicurl "http://192.168.138.77:9696" --adminurl "http://192.168.138.77:9696"  --internalurl "http://192.168.138.77:9696" --region beijing

安装Neutron::
 
 yum -y install openstack-neutron  python-kombu python-amqplib  python-pyudev python-stevedore openstack-utils openstack-neutron-openvswitch openvswitch

配置Neutron::
 
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

chkconfig neutron-server on
chkconfig neutron-openvswitch-agent on
chkconfig neutron-dhcp-agent on
chkconfig neutron-l3-agent on
chkconfig  neutron-metadata-agent  on

配置Neutron openvswitch agent::
   
 ln -s /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini /etc/neutron/plugin.ini

 openstack-config --set  /etc/neutron/plugin.ini  OVS  tenant_network_type gre
 openstack-config --set  /etc/neutron/plugin.ini  OVS  tunnel_id_ranges 1:1000
 openstack-config --set  /etc/neutron/plugin.ini  OVS  enable_tunneling True 
 openstack-config --set  /etc/neutron/plugin.ini  OVS  local_ip 192.168.138.77
 openstack-config --set  /etc/neutron/plugin.ini  OVS  integration_bridge br-int
 openstack-config --set  /etc/neutron/plugin.ini  OVS  tunnel_bridge br-tun
 openstack-config --set  /etc/neutron/plugin.ini  SECURITYGROUP  firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver


配置Neutron dhcp agent::
 
 openstack-config --set  /etc/neutron/dhcp_agent.ini  DEFAULT  interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
 openstack-config --set  /etc/neutron/dhcp_agent.ini  DEFAULT  dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
 openstack-config --set  /etc/neutron/dhcp_agent.ini  DEFAULT  use_namespaces True


配置Neutron L3 agent::
 
 ovs-vsctl add-br br-ex
 ovs-vsctl add-port br-ex eth1
 ip addr add 192.168.137.231/24 dev br-ex
 ip link set br-ex up
 echo  "ip addr add 192.168.137.231/24 dev br-ex" >> /etc/rc.local
 
 openstack-config --set  /etc/neutron/l3_agent.ini DEFAULT  interface_driver  neutron.agent.linux.interface.OVSInterfaceDriver
 openstack-config --set  /etc/neutron/l3_agent.ini DEFAULT  user_namespaces True
 openstack-config --set /etc/neutron/l3_agent.ini DEFAULT external_network_bridge br-ex
 openstack-config --set /etc/neutron/l3_agent.ini DEFAULT enable_metadata_proxy True;



配置Neutron metadata::
 
 vim /etc/neutron/metadata_agent.ini
 #注释auth_region这一行

 openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  auth_region beijing
 openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  auth_url http://192.168.138.77:35357/v2.0
 openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  admin_tenant_name admin
 openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  admin_user admin
 openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  admin_password openstack
 openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  nova_metadata_ip 192.168.138.77
 openstack-config --set  /etc/neutron/metadata_agent.ini DEFAULT  metadata_proxy_shared_secret  awcloud


 
启动Nova和Neutron
================

配置nova启动项::
 
 chkconfig openstack-nova-consoleauth on
 chkconfig openstack-nova-api on
 chkconfig openstack-nova-scheduler on
 chkconfig openstack-nova-conductor on
 chkconfig openstack-nova-compute on
 chkconfig openstack-nova-novncproxy on

配置Neutron启动项::
 
 chkconfig neutron-openvswitch-agent on
 chkconfig neutron-server on
 chkconfig neutron-openvswitch-agent on
 chkconfig neutron-dhcp-agent on
 chkconfig neutron-l3-agent on
 chkconfig  neutron-metadata-agent  on


启动Neutron::
 
 service neutron-openvswitch-agent start
 service neutron-dhcp-agent start
 service neutron-l3-agent start
 service neutron-metadata-agent start
 service neutron-server start
 service neutron-openvswitch-agent start


启动nova::
 service openstack-nova-conductor start
 service openstack-nova-api start
 service openstack-nova-scheduler start
 service openstack-nova-compute start
 service openstack-nova-consoleauth start
 service openstack-nova-novncproxy start


同步Nuetron数据库::
 
 neutron-db-manage --config-file /usr/share/neutron/neutron-dist.conf --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head;


同步nova数据库::
 
 nova-manage db sync

重新启动服务::
 
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

运行没有报错即可，如果报错，重新同步nova和Nuetron数据库::
 
 neutron net-list