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