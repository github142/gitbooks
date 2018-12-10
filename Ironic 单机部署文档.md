# 一 系统环境准备
操作系统: CentOS 7.5 64位

硬件配置: 4C/4G/100G

网卡1： 192.168.104.15

```bash
# 添加epel 源
yum install epel-release -y

# 关闭防火墙
systemctl disable firewalld
systemctl stop firewalld

# 关闭selinux
sed -i 's/=enforcing/=disabled/1' /etc/selinux/config
reboot
```
# 二 ironic和依赖组件
### 2.1 ironic和依赖组件安装
```bash
# 添加openstack 源
yum install centos-release-openstack-rocky -y

# 安装mysql rabbitmq 和其他依赖软件
yum install dnsmasq mysql-server rabbitmq-server shellinabox qemu-img iscsi-initiator-utils psmisc openssl -y

# 安装ironic-api 和 ironic-conductor
yum install openstack-ironic-api openstack-ironic-conductor python-ironicclient -y

# 启动iscis deploy相关服务，iscsid
systemctl start iscsid
systemctl enable iscsid

```
### 2.2 PXE 配置
 
```bash
# 创建tftpboot目录
mkdir -p /tftpboot

# 安装 tftp-server
yum install tftp-server syslinux syslinux-tftpboot xinetd -y

# 复制系统 pxe 引导文件到 tftp 目录
cp /usr/share/syslinux/pxelinux.0 /tftpboot/
cp /usr/share/syslinux/chain.c32 /tftpboot/

# 创建 map-file 文件
echo 're ^(/tftpboot/) /tftpboot/\2' > /tftpboot/map-file
echo 're ^/tftpboot/ /tftpboot/' >> /tftpboot/map-file
echo 're ^(^/) /tftpboot/\1' >> /tftpboot/map-file
echo 're ^([^/]) /tftpboot/\1' >> /tftpboot/map-file

# 修改 tftp 配置文件
vi /etc/xinetd.d/tftp 
service tftp
{
  protocol        = udp
  port            = 69
  socket_type     = dgram
  wait            = yes
  user            = root
  server          = /usr/sbin/in.tftpd
  server_args     = -v -v -v -v -v --map-file /tftpboot/map-file /tftpboot
  disable         = no
  flags           = IPv4
}

# 创建inspector pxe default文件
mkdir /tftpboot/pxelinux.cfg/
vi /tftpboot/pxelinux.cfg/default
default introspect

label introspect
kernel inspector/ironic-deploy-new.kernel
append initrd=inspector/ironic-deploy-new.initramfs ipa-inspection-callback-url=http://192.168.104.15:5050/v1/continue ipa-inspection-collectors=default,logs ipa-inspection-dhcp-all-interfaces=true systemd.journald.forward_to_console=yes

ipappend 3

# 拷贝镜像文件并修改权限
mkdir /tftpboot/inspector
/tftpboot/inspector/ironic-deploy-new.kernel
/tftpboot/inspector/ironic-deploy-new.initramfs

mkdir /tftpboot/deploy
/tftpboot/deploy/coreos_production_pxe.vmlinuz
/tftpboot/deploy/coreos_production_pxe_image-oem.cpio.gz 

mkdir /tftpboot/user
/tftpboot/user/ubuntu-trusty.qcow2

chown -R ironic. /tftpboot

# 启动xinetd
systemctl restart xinetd
systemctl enable xinetd
```
### 2.3 生成 web 控制台ssl证书
```bash
#生成证书
mkdir -p /opt/ca
cd /opt/ca
openssl genrsa -des3 -out my.key 1024
openssl req -new -key my.key  -out my.csr
cp my.key my.key.org
openssl rsa -in my.key.org -out my.key
openssl x509 -req -days 3650 -in my.csr -signkey my.key -out my.crt
cat my.crt my.key > certificate.pem
``` 

### 2.4 配置rabbitmq 
```bash
# 启动rabbitmq
systemctl start rabbitmq-server.service
systemctl enable rabbitmq-server.service

# 添加用户
rabbitmqctl add_user RPC_USER RPC_PASSWORD
rabbitmqctl set_permissions RPC_USER ".*" ".*" ".*"
```
### 2.5 Ironic 配置

```bash
# 启动数据库 
systemctl start mariadb
systemctl enable mariadb

# 创建ironic数据库 
mysql -u root
mysql> CREATE DATABASE ironic CHARACTER SET utf8;
mysql> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost' \
       IDENTIFIED BY 'IRONIC_DBPASSWORD';
mysql> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' \
       IDENTIFIED BY 'IRONIC_DBPASSWORD';

# 修改ironic配置文件
vi /etc/ironic/ironic.conf 
[DEFAULT]
auth_strategy = noauth
enabled_hardware_types = ipmi
enabled_boot_interfaces = pxe
enabled_console_interfaces = ipmitool-shellinabox
enabled_deploy_interfaces = iscsi,direct
enabled_inspect_interfaces = inspector
enabled_management_interfaces = ipmitool
enabled_network_interfaces = noop
enabled_power_interfaces = ipmitool
my_ip = 192.168.104.15
notification_level = debug
transport_url = rabbit://RPC_USER:RPC_PASSWORD@192.168.104.15:5672/

[api]
host_ip = 0.0.0.0
port = 6385

[conductor]
my_ip= 192.168.104.15
automated_clean = false
api_url = http://192.168.104.15:6385
send_sensor_data = true
send_sensor_data_types = ALL

[console]
terminal_cert_dir=/opt/ca

[database]
connection=mysql+pymysql://ironic:IRONIC_DBPASSWORD@192.168.104.15/ironic?charset=utf8

[dhcp]
dhcp_provider = none

[inspector]
enabled = true
service_url = http://192.168.104.15:5050

[pxe]
pxe_append_params = nofb nomodeset vga=normal console=ttyS0,115200n8

# 初始化ironic数据库
ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema

# 启动ironic服务
systemctl restart openstack-ironic-api
systemctl restart openstack-ironic-conductor
systemctl enable openstack-ironic-api
systemctl enable openstack-ironic-conductor


# 创建用户环境变量文件
vi /root/adminrc
export OS_TOKEN=fake-token
export IRONIC_URL=http://localhost:6385/

# 验证
source /root/adminrc
ironic driver-list

```

# 三 Ironic-inspector
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