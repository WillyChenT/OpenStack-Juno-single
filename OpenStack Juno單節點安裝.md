# OpenStack Juno單節點安裝

這是一個以單台主機為節點的OpenStack實作   

[TOC]

## 環境設置

>vim /etc/hostname
 

```
controller
```

>vim /etc/hosts

```
140.127.194.130 controller
```

>vim /etc/network/interfaces 
  
``` python
auto eth0
iface eth0 inet static
        address 140.127.194.130
        netmask 255.255.255.0
        gateway 140.127.194.253
        dns-nameservers 192.83.191.8 192.83.191.9

auot eth1
iface eth1 inet manual

auto br100
iface br100 inet static
        address 140.127.194.129
        pre-up ifconfig eth1
        bridge-ports eth1 0.0.0.0
        bridge_ports eth1
        bridge_stp off

```
##安裝openstack資料庫
>echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu" \
>  "trusty-updates/juno main" > /etc/apt/sources.list.d/cloudarchive-juno.list

更新系統資料
>apt-get update
>
>apt-get dist-upgrade

##安裝database
>apt-get install mariadb-server python-mysqldb
>
>vim /etc/mysql/my.cnf

```
[mysqld]
#修改bind-address 
bind-address = 0.0.0.0

#最末端新增下列
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```

重新啟動mysql
>service mysql restart

##安裝Messaging server
>apt-get install rabbitmq-server

```
# rabbitmqctl change_password guest 07927743
Changing password for user "guest" ...
...done.
```

##安裝 Identity service
####配置環境
>mysql -u root -p

```
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '07927743';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '07927743';
```
生產一個隨機密鑰給Administration token使用
>openssl rand -hex 10


```
1006a51f77aa5f342b10
```

####安裝與設定
>apt-get install keystone python-keystoneclient
>
>vim /etc/keystone/keystone.conf


```
[DEFAULT]
# 這裡要修改成剛剛產生的隨機密鑰
admin_token = 1006a51f77aa5f342b10

[database]
# 修改資料庫的來源跟密碼
connection = mysql://keystone:07927743@controller/keystone

[token]
# 新增UUID的來源和SQL的驅動
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.sql.Token

```
同步資料庫
>su -s /bin/sh -c "keystone-manage db_sync" keystone

####完成安裝
重新啟動 keystone
>service keystone restart

刪除ubuntu預設的SQLite
>rm -f /var/lib/keystone/keystone.db

設置每小時刪除一次無效token 避免累計過大增加server負擔

```
(crontab -l -u keystone 2>&1 | grep -q token_flush) || echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/crontabs/keystone
```

##Identity service建置
####配置環境
設置Administration token與Endpoint
>export OS_SERVICE_TOKEN=1006a51f77aa5f342b10
>export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0

####新增tenants users和roles
新增admin tenant
>keystone tenant-create --name admin --description "Admin Tenant"

![Alt text](./1.png)

新增admin user
>keystone user-create --name admin --pass 07927743 --email admin@test.com

![Alt text](./2.png)


新增admin role
>keystone role-create --name admin

![Alt text](./3.png)


將admin tenant和user添加到admin role
>keystone user-role-add --user admin --tenant admin --role admin

*不會顯示任何信息


新增service tenant
>keystone tenant-create --name service --description "Service Tenant"

![Alt text](./4.png)

新增Identity service entity
>keystone service-create --name keystone --type identity --description "OpenStack Identity"

![Alt text](./5.png)

新增Identity service API endpoints
>keystone endpoint-create --service-id $(keystone service-list | awk '/ identity / {print $2}') --publicurl http://controller:5000/v2.0 --internalurl http://controller:5000/v2.0 --adminurl http://controller:35357/v2.0 --region regionOne

![Alt text](./6.png)

####驗證操作
取消臨時設置的OS_SERVICE_TOKEN與OS_SERVICE_ENDPOINT
>unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT

使用admin取得authentication token
>keystone --os-tenant-name admin --os-username admin --os-password 07927743 --os-auth-url http://controller:35357/v2.0 token-get

![Alt text](./1427514166782.png)

使用admin查看tenan列表

>keystone --os-tenant-name admin --os-username admin --os-password 07927743 \
> --os-auth-url http://controller:35357/v2.0 tenant-list

![Alt text](./1427514217600.png)

使用admin查看user列表

>keystone --os-tenant-name admin --os-username admin --os-password 07927743 --os-auth-url >http://controller:35357/v2.0 user-list

![Alt text](./1427514244627.png)

使用admin查看role列表

>keystone --os-tenant-name admin --os-username admin --os-password 07927743 --os-auth-url http://controller:35357/v2.0 role-list

![Alt text](./1427514291807.png)


####新增環境憑證腳本
>vim admin-openrc.sh

```
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=07927743
export OS_AUTH_URL=http://controller:35357/v2.0
```

>source admin-openrc.sh

##安裝Image Service
####配置環境
>apt-get install glance python-glanceclient
>
>vim /etc/glance/glance-api.conf

```
[DEFAULT]
#設置鏡像存儲位置
default_store = file

[database]
#修改資料庫的來源跟密碼
connection = mysql://glance:07927743@controller/glance

[keystone_authtoken]
#設置Identity service許可
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = glance
admin_password = 07927743
[paste_deploy]
flavor = keystone

[glance_store]
#設置鏡像的存儲路徑
filesystem_store_datadir = /var/lib/glance/images/
```

>vim /etc/glance/glance-registry.conf

```
[database]
修改資料庫的來源跟密碼
connection = mysql://glance:07927743@controller/glance

[keystone_authtoken]
設置Identity service許可
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = glance
admin_password = 07927743
[paste_deploy]
flavor = keystone
```
同步資料庫
>su -s /bin/sh -c "glance-manage db_sync" glance

####完成安裝
>service glance-registry restart

>service glance-api restart

刪除ubuntu預設的SQLite
>rm -f /var/lib/glance/glance.sqlite

####驗證操作
創建一個鏡像暫存資料夾
>mkdir /tmp/images
>cd /tmp/images

下載測試鏡像檔
>wget http://cdn.download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img

上傳鏡像檔到Image Service
>glance image-create --name "cirros-0.3.3-x86_64" --file cirros-0.3.3-x86_64-disk.img --disk-format qcow2 --container-format bare --is-public True --progress
 
![Alt text](./1427514755353.png)

查看上傳的鏡像屬性
>glance image-list
 
移除暫存資料夾
>rm -r /tmp/images

##安裝Compute service
####配置環境
>mysql -u root -p


```
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY '07927743';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY '07927743';
```
新增nova user
>keystone user-create --name nova --pass 07927743

 ![Alt text](./1427514871325.png)


將nova user添加到admin role
>keystone user-role-add --user nova --tenant service --role admin

新增nova service entity
>keystone service-create --name nova --type compute --description "OpenStack Compute"

![Alt text](./1427514876444.png)
 

新增Compute Service API endpoints
>keystone endpoint-create \
>  --service-id $(keystone service-list | awk '/ compute / {print $2}') \
>  --publicurl http://controller:8774/v2/%\(tenant_id\)s \
>  --internalurl http://controller:8774/v2/%\(tenant_id\)s \
>  --adminurl http://controller:8774/v2/%\(tenant_id\)s \
>  --region regionOne

 ![Alt text](./1427514884054.png)

####安裝與設定Controller Node
>apt-get install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy \
>nova-scheduler python-novaclient nova-compute sysfsutils

>vim /etc/nova/nova.conf

```
[DEFAULT]
#設置RabbitMQ message broker許可
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = 07927743

auth_strategy = keystone
my_ip = 140.127.194.130
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 140.127.194.130
novncproxy_base_url = http://140.127.194.130:6080/vnc_auto.html

[glance]
#設置Image Service位置
host = controller

[database]
#修改資料庫的來源跟密碼
connection = mysql://nova:07927743@controller/nova

[keystone_authtoken]
#設置Identity service許可
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = nova
admin_password = 07927743
```

####完成安裝Compute Node
>service nova-api restart
>service nova-cert restart
>service nova-consoleauth restart
>service nova-scheduler restart
>service nova-conductor restart
>service nova-novncproxy restart
>service nova-compute restart

刪除ubuntu預設的SQLite
>rm -f /var/lib/nova/nova.sqlite

####驗證操作
查看各個Compute Service狀態
>nova service-list
 
查看Image Service的鏡像列表
>nova image-list
 
##安裝Networking元件
安裝與設定Compute Node
>apt-get install nova-network 

>vim /etc/nova/nova.conf

```
auth_strategy = keystone
my_ip = 140.127.194.129
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 140.127.194.189
novncproxy_base_url = http://140.127.194.130:6080/vnc_auto.html

network_api_class = nova.network.api.API
security_group_api = nova

network_manager=nova.network.manager.FlatDHCPManager
force_dhcp_release=True
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

flat_network_bridge=br100
fixed_range=10.10.0.0/22
flat_network_dhcp_start=10.10.0.10
network_size=1022

# Floating IPs
#auto_assign_floating_ip=true
#default_floating_pool=public
public_interface=eth1
```

>service nova-api restart
>service nova-scheduler restart
>service nova-conductor restart
>service nova-network restart
>service nova-api-metadata restart

##新增Network
>nova network-create demo-net --bridge br100 --multi-host T \
>  --fixed-range-v4 10.10.0.0/24
>nova-manage floating create --ip_range 140.127.194.0/24 --pool=public

查看Network列表
>nova net-list
>
![Alt text](./1427515132815.png)

##安裝DashBoard
####安裝與設定
>apt-get install openstack-dashboard apache2 libapache2-mod-wsgi memcached python-memcache && dpkg --purge openstack-dashboard-ubuntu-theme

>vim /etc/openstack-dashboard/local_settings.py

```
OPENSTACK_HOST = "controller"
```

>service apache2 restart
>service memcached restart

##安裝Block Storage service
####配置環境
>mysql -u root -p

```
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY '07927743';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY '07927743';
```

>keystone user-create --name cinder --pass 07927743

![Alt text](./1427515304586.png)

>keystone user-role-add --user cinder --tenant service --role admin
>keystone service-create --name cinder --type volume --description "OpenStack Block Storage"

 ![Alt text](./1427515310404.png)

>keystone service-create --name cinderv2 --type volumev2 --description "OpenStack Block Storage"

 ![Alt text](./1427515316508.png)

>keystone endpoint-create --service-id $(keystone service-list | awk '/ volume / {print $2}') --publicurl http://controller:8776/v1/%\(tenant_id\)s --internalurl http://controller:8776/v1/%\(tenant_id\)s --adminurl http://controller:8776/v1/%\(tenant_id\)s --region regionOne

 ![Alt text](./1427515320323.png)


>keystone endpoint-create --service-id $(keystone service-list | awk '/ volumev2 / {print $2}') --publicurl http://controller:8776/v2/%\(tenant_id\)s --internalurl http://controller:8776/v2/%\(tenant_id\)s --adminurl http://controller:8776/v2/%\(tenant_id\)s --region regionOne

 ![Alt text](./1427515326176.png)

####安裝與設定Controller Node
>apt-get install cinder-api cinder-scheduler python-cinderclient cinder-volume python-mysqldb

>vim /etc/cinder/cinder.conf

```
[DEFAULT]
#設置RabbitMQ message broker許可
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = 07927743
my_ip = 140.127.194.129
glance_host = controller

[database]
#修改資料庫的來源跟密碼
connection = mysql://cinder:07927743@controller/cinder

[keystone_authtoken]
#設置Identity service許可
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = cinder
admin_password = 07927743
```




如有任何疑問與建議。歡迎與我聯繫
e-mail: eaischang@gmail.com

