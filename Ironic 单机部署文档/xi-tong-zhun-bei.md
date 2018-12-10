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
