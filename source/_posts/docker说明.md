---
title: 安装docker说明
date: 2023-09-23 10:00:00
categories:
  - 代码命令
---
之前讲过玩客云可以安装armbian与openwrt，但是并没有安装docker，因为玩客云的s905是32位的，可用的docker镜像比较少，所以没有折腾，最近发现了一个刷机即可使用docker的armbian镜像，就玩了一下，可以实现大部分的斐讯n1的功能。

有了omv5与docker丝毫不逊色N1的玩客云

让玩客云与N1一样出色不仅要omv还要docker

可以安装docker的armbian镜像docker_Armbian_20.05.4_Aml-s805_focal_current_5.7.0-rc5
https://pan.baidu.com/s/132mbd05DE9wqYoRiw4dA4Q
提取码：5wxi

镜像通过写盘工具写到u盘，插入到靠近网口的usb上，插上电源方可加载系统

第一次启动会很慢，稍安勿躁
启动以后，去路由器里拿到ip
用ssh工具登录
用户名是root
密码是1234
登录后要求修改root密码
输入密码1234
然后输入两次新密码，有复杂度要求，数字密码大写特殊符号等，你看着输吧

设置成功会要求新建用户，这里取消就可以了，不用新用户了

按ctrl c会中断

这时候系统还是在u盘了的

cd /boot/install
./install.sh
会把系统写入到emmc中

等待写入完毕会自动退出
拔下u盘，拔电重启

耐心等待启动完毕，ip大概率还是之前的ip，如果登陆不了就去路由器里确认。

再次登录成功，系统就刷好了

## 安装samba实现共享

插上移动硬盘或者u盘，我这里是一个u盘

需要分区格式化，我这里就不演示了，用fdisk或者parted分区，挂载

我挂在到了/data目录上

安装samba

```bash
apt update 
apt install samba
# 设置用户
useradd zhangxiaoqi
smbpasswd -a zhangxiaoqi
# 123456
```

配置共享的目录
nano  /etc/samba/smb.conf
```bash
[wkyshare]
   comment = wky shares
   path = /data
   create mask = 0664
   directory mask = 0775
   read only = No
   valid users = zhangxiaoqi
```

重启服务

```bash
systemctl restart smbd
systemctl enable smbd
# 可以在局域网的win机器访问
# \\ip 查看共享是否正常
```

就可以了安装docker

```bash
apt update
apt install docker.io
docker info
```

## 下面是必装镜像

安装portainer

```bash
docker run -d \
  -p 8000:8000 \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /data/docker/portainer/data:/data \
  --name=portainer \
  --restart=always \
  portainer/portainer-ce

```

安装openwrt
docker pull dreamwsbg/openwrt:7.0
docker run -d --name="openwrt" --restart unless-stopped --network macnet --privileged dreamwsbg/openwrt:7.0 /sbin/init


docker pull virking/openwrt:20.04
ip link set eth0 promisc on
docker network create -d macvlan --subnet=192.168.188.0/24 --gateway=192.168.188.1 -o parent=eth0 macnet

启动一个openwrt容器
docker run -i -t -d --name=openwrt --restart=always --network=macnet --privileged=true virking/openwrt:20.04 /sbin/init
docker run -d --name="openwrt" --restart unless-stopped --network macnet --privileged dreamwsbg/openwrt:7.0 /sbin/init
编辑网卡
vi /etc/config/network
内容如下，注意修改网段

config interface 'loopback'
        option ifname 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

config globals 'globals'
        option ula_prefix 'fdd3:60d7:a044::/48'

config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '192.168.188.254' #openwrt的IP地址，可自定义网段内任意IP，不发生冲突即可
        option netmask '255.255.255.0'
        option gateway '192.168.188.1' #填写主路由的IP地址
        option dns '192.168.188.1' #填写主路由的IP地址

config interface 'vpn0'
        option ifname 'tun0'
        option proto 'none'


重启玩客云等待openwrt启动后
账密：root-password



回到玩客云shell
 /etc/network/interfaces /etc/network/interfaces.bak
 vi /etc/network/interfaces

添加如下内容
auto macvlan
iface macvlan inet static
  address 192.168.188.8
  netmask 255.255.255.0
  gateway 192.168.188.1
  dns-nameservers 192.168.188.1
  pre-up ip link add macvlan link eth0 type macvlan mode bridge
  post-down ip link del macvlan link eth0 type macvlan mode bridge





Bushu transmission

```bash

docker run \
  -d \
  --name=transmission \
  -e USERNAME=admin \
  -e PASSWORD=admin \
  -e PUID=1000 \
  -e PGID=1000 \
  -p 9091:9091 \
  -p 51413:51413 \
  -p 51413:51413/udp \
  -v /data/transmission/config:/config \
  -v /data:/downloads \
  -v /data/transmission/watch:/watch \
  --restart unless-stopped \
lsioarmhf/transmission

```

给transmission安装界面
到docker的shell中
wget https://gitee.com/culturist/transmission-web-control/raw/master/release/install-tr-control-gitee.sh
bash install-tr-control-cn.sh



网盘nextcloud

```bash
docker run -d \
--name nextcloud \
-p 8085:80 \
-v /data/docker/nextcloud/html:/var/www/html \
-e TZ=Asia/Shanghai \
--restart unless-stopped \
nextcloud

```

blog

```bash
docker run -d -p 80:80 --name typecho -v "/data/tyepcho":/var/www/html yangxuan8282/typecho:php-arm
```

aria2

```bash
docker run -d \
    --name aria2-pro \
    --restart unless-stopped \
    --log-opt max-size=1m \
    --network host \
    -e PUID=1000 \
    -e PGID=1000 \
    -e RPC_SECRET=12345 \
    -e RPC_PORT=6800 \
    -e LISTEN_PORT=6888 \
    -v /data/docker/aria2/config:/config \
    -v /data/downloads:/downloads \
    p3terx/aria2-pro
```

浏览器安装插件连接 6800端口即可

web ui

[https://github.com/mayswind/AriaNg](https://github.com/mayswind/AriaNg)

同步工具

verysync

```bash
docker run -d \
--restart=unless-stopped \
--name verysync \
-v /data/verysync:/data \
-p 8886:8886 \
unwenliu/verysync:armv7
```

qbittorrent

```bash
docker run -d \
  --name=qbittorrent \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Aisa/Shanghai \
  -e WEBUI_PORT=8080 \
  --network host \
  -v /data/docker/qb/config:/config \
  -v /data/qb_downloads:/downloads \
  --restart unless-stopped \
  ghcr.io/linuxserver/qbittorrent
```

有了docker，玩客云确实一个不错的nas了

相比直接安装openwrt，可用性会高很多，也经得起折腾

希望大家不要瞧不起玩客云，哈

玩客云的终点就在这里了，已经是极限了

本期视频就到这里了，我是张大七，我们下期见。