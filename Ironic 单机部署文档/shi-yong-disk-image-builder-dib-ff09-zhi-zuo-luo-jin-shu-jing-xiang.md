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
