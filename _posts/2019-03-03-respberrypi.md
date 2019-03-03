---
layout: post
title:  'Respberry Pi 3B!'
subtitle:  '树莓派的安装和配置'
date:    '2019-03-03 18:35:00'
background:  /image/post2.jpg
---
![respberry](/image/posts/2019-03-03/respberrypi.png)

按照我的想法，我是想用Arduino+RespberryPi完成机器人的制作，Arduino主控运动控制，RespberryPi作为大脑。Respberry其实就是一个低配版的笔记本，使用它我可以最大化地运用各种软件资源去做更多的事情，而不用受限于嵌入式系统。

### 一、准备
我入手的树莓派是Respberry Pi 3代B型，除了树莓派本身，它的启动还需要一些其他设备：
- 电源
    
    树莓派使用的是5V2A的电源，电源线是Micro USB接口的，类似于以前大部分安卓手机使用的mini usb口的充电线就行，USB端可以接移动电源或者电脑USB口。
- 显示器
    
    树莓派第一次安装系统需要显示器来配置，安装完系统之后就可以ssh登录，或者windows远程桌面登录，之后就不需要显示器了。显示器需要支持HDMI接口，因为树莓派的接口就是HDMI接口，除此之外还需要HDMI线。如果只有VGA接口的显示器，那可以用HDMI转VGA线。
- MicroSD卡
    
    树莓派没有内置的硬盘，需要插入MicroSD卡作为它的硬盘，建议至少配备16G的卡，我是从旧的手机里面找到了一张32G的卡。
- 键鼠
    
    键鼠也只是在第一次安装系统的时候需要使用，等到可以远程登录的时候，键鼠也不需要了。

另外，树莓派本身是有内置的网卡的，所以不需要额外购买无线网卡。所以，树莓派的初始化只需要以上的这些，其他外壳、散热片之类的，以后用到了再说，可能大部分人需要买的也就是树莓派本身+MicroSD卡。

### 二、安装
安装树莓派系统比较简单，按照[官方教程](https://www.raspberrypi.org/downloads/noobs/)一步一步操作即可，简单来说就是：
- 格式化MicroSD卡为FAT格式
- 下载[NOOBS](https://downloads.raspberrypi.org/NOOBS_latest)，并解压到SD卡
- 将SD卡插入卡槽，插上显示器，插上键鼠，插上电源

由于没有开机按钮，树莓派在插上电源的时候就自动启动了，然后下面的操作类似于安装linux系统的操作，一步一步就可以安装完成。

### 三、配置
1. 网络配置

    无线网络的连接在桌面右上角，点击、输入密码即可。

    为了防止没有显示器，没有办法连接之前配置过的无线网，需要配置有线网络，这样就可以在无线连接不上树莓派的时候，通过网线连接到树莓派，然后让树莓派连接到可连接的无线网。具体操作就是给eth0配置固定局域网ip，然后在需要的时候将电脑的有线网络ip配置到同一网段就可以连接了。

    正常配置都是在/etc/network/interfaces文件，进入后发现注释要求修改/etc/dhcpcd.conf
    ```conf
    # interfaces(5) file used by ifup(8) and ifdown(8)

    # Please note that this file is written to be used with dhcpcd
    # For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

    # Include files from /etc/network/interfaces.d:
    source-directory /etc/network/interfaces.d
    ```
    然后进入修改/etc/dhcpcd.conf文件，将# Example static IP configuration下面的内容修改为如下内容：
    ```conf
    # Example static IP configuration:
    interface eth0
    static ip_address=10.6.6.6/24
    #static ip6_address=fd51:42f8:caae:d92e::ff/64
    static routers=10.6.6.1
    static domain_name_servers=10.6.6.1 8.8.8.8
    ```
    这里是将树莓派的有线网络固定为10.6.6.6。

    然后配置一下电脑的以太网，测试一下有线连接：将电脑的以太网的ip固定为10.6.6.7，子网掩码设置为255.255.255.0。此时树莓派和电脑就处于局域网的同一网段，然后连上网线，重启树莓派，如果启动后电脑可以ping通树莓派(10.6.6.6)，则配置成功。

2. 用户密码
    
    ![respberry](/image/posts/2019-03-03/change-passwd.png)

    系统的默认用户pi没有设置密码，需要在首选项->Respberry Pi Configration中修改默认用户pi的密码，然后再使用命令行

    ```shell
    sudo passwd root
    ```

    设置root用户的密码。

3. 远程登录
    
    ![respberry](/image/posts/2019-03-03/enable-ssh.png)
    
    首先打开ssh登录的权限，在Respberry Pi Configration中打开ssh的权限，此时就可以从外部使用ssh登录树莓派了。
    
    然后安装windows远程桌面需要的软件xrdp：

    ```shell
    sudo apt install tightvncserver xrdp
    ```

    安装完成后执行命令sudo /etc/init.d/xrdp restart 重启xrdp服务，此时就可以使用windows的远程桌面登录到树莓派的桌面了。

-----
到此，如果一切正常，那么就可以将显示器和键鼠拔掉，只留下电源和MicroSD卡就可以了，之后只需要远程连接即可，无论是从无线网络连接还是从10.6.6.6连接，都可以操作树莓派。