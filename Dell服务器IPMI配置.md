# Dell 服务器配置 
### 1 启用远程管理卡的IPMI
![ironic-bios](ironic1.jpg)
```
 配置 BMC
1）系统启动 BIOS POST 画面，提示按 Ctrl+E 配置 Remote Access Configuration Utility
1) Remote Access Configuration Utility---IPMI Over LAN---On
```


### 2 启用BIOS远程终端重定向
![ironic-bios](ironic2.jpg)
```
系统启动，安 F2 进入 BIOS 设置, 设置重定向到 com2
• Serial Communication ……. On with Console Redirection via COM2
• External Serial Connector .. COM2
• Failsafe Baud Rate ……… 115200
• Remote Terminal Type ……. VT100/VT220
• Redirection After Boot ….. Enabled
```

### 3 操作系统相关设置
```
1) 修改 grub
在 grub 中加入 串口,Baud Rate 以及在系统中终端
即如下：


default=2
timeout=15

#注掉 splashimage 图形显示行，否则在字符模式下无法显示菜单
#splashimage=(hd0,0)/grub/splash.xpm.gz
………..
kernel /boot/kernel ro root=/dev/sda1 console=tty0 console=ttyS1,57600

在当前 kernel 后面添加蓝色字符部分.


2)修改/etc/inittab
修改或加入下列一行在第 50 行的位置


# Console Redirection via COM1/COM2
s0:2345:respawn:/sbin/agetty -h -L 57600 ttyS0 vt100
s1:2345:respawn:/sbin/agetty -h -L 57600 ttyS1 vt100

# init –q


3)修改字符终端能够登陆
修改/etc/securetty 添加
ttyS1
ttyS0
```