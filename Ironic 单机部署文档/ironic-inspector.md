```bash
# 安装ironic-inspector
yum install openstack-ironic-inspector python2-ironic-inspector-client -y

# 创建inspector数据库
mysql -u root
mysql> CREATE DATABASE ironic_inspector CHARACTER SET utf8;
mysql> GRANT ALL PRIVILEGES ON ironic_inspector.* TO 'inspector'@'localhost' \
IDENTIFIED BY 'inspector';
mysql> GRANT ALL PRIVILEGES ON ironic_inspector.* TO 'inspector'@'%' \
IDENTIFIED BY 'inspector';

# 修改ironic-inspector配置文件
vi /etc/ironic-inspector/inspector.conf
[DEFAULT]
listen_address = 0.0.0.0
listen_port = 5050
auth_strategy = noauth

[database]
connection =  mysql+pymysql://inspector:inspector@192.168.104.15/ironic_inspector?charset=utf8

[dnsmasq_pxe_filter]
[ironic]
auth_url = http://192.168.104.15:6385
auth_strategy = noauth
ironic_url = http://192.168.104.15:6385/

[processing]
add_ports = all
keep_ports = present
store_data = none
driver = dnsmasq

# 初始化ironic-inspector数据库
ironic-inspector-dbsync --config-file /etc/ironic-inspector/inspector.conf upgrade

# ironic-inspector 修改
cd /usr/lib/python2.7/site-packages/ironic_inspector
替换plugins/standard.py和common/i18n.py二个文件

# 修改/etc/ironic-inspector/dnsmasq.conf配置文件为如下内容
vi /etc/ironic-inspector/dnsmasq.conf
port=0
interface=ens32
bind-interfaces
dhcp-range=192.168.104.210,192.168.104.220
enable-tftp
tftp-root=/tftpboot
dhcp-boot=pxelinux.0
dhcp-sequential-ip

# 启动ironic-inspector
systemctl restart openstack-ironic-inspector 
systemctl restart openstack-ironic-inspector-dnsmasq
systemctl enable openstack-ironic-inspector
systemctl enable openstack-ironic-inspector-dnsmasq
```