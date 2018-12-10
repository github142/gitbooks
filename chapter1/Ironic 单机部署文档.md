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



# 四 添加节点测试
```bash
# 创建node
ironic node-create -n node001 -d ipmi -i ipmi_username=root -i ipmi_password=calvin -i ipmi_address=192.168.1.153 -i ipmi_terminal_port=5900 -i deploy_kernel=file:///tftpboot/deploy/coreos_production_pxe.vmlinuz -i deploy_ramdisk=file:///tftpboot/deploy/coreos_production_pxe_image-oem.cpio.gz 

# 配置console信息
ironic node-update node001 replace console_interface='ipmitool-shellinabox'
ironic node-update node001 add driver_info/ipmi_terminal_port='5900'
ironic node-set-console-mode node001 true

# 验证driver信息
ironic node-validate  node001
ironic node-set-provision-state node001 manage

# 开始inspect
ironic node-set-provision-state node001 inspect

# 添加deploy信息
ironic node-update  node001 add instance_info/root_gb=100
ironic node-update  node001  add instance_info/capabilities='{"boot_option": "local"}'
ironic node-update  node001  add instance_info/image_source=file:///tftpboot/user/ubuntu-trusty.qcow2

# 部署
ironic node-set-provision-state node001 provide
ironic node-set-provision-state node001 active
ironic node-show node001

```

# 五 使用disk-image-builder（DIB）制作裸金属镜像
https://docs.openstack.org/diskimage-builder/latest/

### 5.1 deploy image 
使用默认或者制作带有特定品牌raid配置驱动的deploy镜像
```bash
# 例如制作HP的RAID管理工具proliant-tools
disk-image-create ubuntu ironic-agent proliant-tools -o /tmp/ironic-deploy
```

### 5.2 user image
```bash
# 配置镜像制作环境变量
export DIB_RELEASE=trusty  # 操作系统版本
export DIB_DEV_USER_USERNAME=centos # devuser 用户名
export DIB_DEV_USER_PASSWORD=centos # devuser 密码
export DIB_DEV_USER_PWDLESS_SUDO=YES # 允许devuser 执行sudo
export DIB_CLOUD_INIT_ALLOW_SSH_PWAUTH=YES # 启用 ssh 密码登陆
export DIB_CLOUD_INIT_DATASOURCES="ConfigDrive, OpenStack" # 配置cloud init 数据源

disk-image-create -a amd64 -t qcow2 -o /tmp/ubuntu-trusty.qcow2 ubuntu vm devuser dhcp-all-interfaces enable-serial-console cloud-init-datasources cloud-init

注:
-a amd64 # 64位操作系统
-t qcow2 # 文件格式
-o /tmp/ubuntu-trusty.qcow2 # 生成文件路径
ubuntu  # 系统类型
devuser #与上面的环境变量DIB_DEV_USER_USERNAME,DIB_DEV_USER_PASSWORD,DIB_DEV_USER_PWDLESS_SUDO 相对应
dhcp-all-interfaces #在所有网卡上启用dhcp
enable-serial-console # 启用serial console
cloud-init-datasources # 配置cloud init 数据源
cloud-init # 通过cloud init 启用 ssh 密码登陆
```
# 六 物理服务器配置

1) 启用远程管理卡的IPMI协议
2) 启用BIOS设置中的远程终端重定向，重定向到COM2
3) 操作系统需要修改grub，启用serial console