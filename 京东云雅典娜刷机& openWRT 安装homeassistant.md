# 京东云雅典娜刷 OpenWRT & 安装 Home Assistant + HomeKit

## 1. 路由器刷 OpenWRT

### 1.1 前置准备

- Windows 设备
- 下载刷机工具
  [Actions-OpenWrt - Router Flashing Files](https://github.com/lgs2007m/Actions-OpenWrt/releases/tag/Router-Flashing-Files)
- 安装高通 USB 驱动
  Qualcomm USB Driver v1.0.10061.1.exe
- TypeA 公对公的数据线一根（一般的 TypeA to TypeC 充电线可能连接不上）
- 拆机装备，六角螺丝刀

### 1.2 刷 U-Boot 和 GPT 分区表

#### 1.2.1 进入 9008 模式刷临时 U-Boot

1. 拆掉外壳和散热器，露出主板。
2. 使用数据线连接 PC 和路由器 USB 口，网线连接 PC 与路由器 LAN 口。
3. 用镊子短接 R656 和 GND 后上电，听到电脑新设备连接提示后就不用短接了（路由器亮红灯则短接失败，重试此步骤）。
4. 打开设备管理器，会出现一个【端口设备】，查看端口里面有个 Qualcomm 的设备，记下 COM(n)，我这里为 COM3，记下 3。
5. 运行【USB命令.bat】按提示输入信息，这里会用到上一步的设备号。
6. 程序分两步执行，执行第二步后路由器会亮红灯，这时候立马按住路由器 reset 键，等待 5s 左右后红灯会闪烁并变蓝，这时候就可以松开 reset 键了。

#### 1.2.2 进临时 U-Boot 刷永久 U-Boot 和 GPT 分区

1. 修改电脑 IP 地址为 `192.168.1.2`，网关地址设为 `192.168.1.1`。
2. 浏览器打开 `http://192.168.1.1/uboot.html` (注意后边一定要带上 /uboot.html)。
3. 选择文件【亚瑟雅典娜通用不死 U-Boot\uboot-JDC_AX1800_Pro-AX6600_Athena-20240510.bin】点击上传，路由器会闪烁蓝光并变绿，自动重启后会变红，这时候 U-Boot 就刷完了。
4. 刷 GPT 分区表：
   断电后按住 reset 上电，红灯闪烁变蓝之后打开浏览器访问 `http://192.168.1.1/img.html`。
   选择 `\雅典娜双分区 GPT 分区表（不带最后的大分区）\gpt-JDC_AX6600_Athena_dual-boot_rootfs2048M_no-last-partition.bin` 并上传，和上述一样的步骤，操作完之后接下来就可以刷固件了。

### 1.3 刷 OpenWRT

> OpenWRT 官方增加了对高通 CPU 的支持，如果想刷官方固件的话可以直接去 OpenWRT 官网找对应 IPQ60xx（对应雅典娜的 CPU 型号）的固件。

源码：[SAMYJ1/immortalwrt](https://github.com/SAMYJ1/immortalwrt)
云编译脚本：[SAMYJ1/OpenWRT-CI](https://github.com/SAMYJ1/OpenWRT-CI)

我这里用的是社区大佬改的 ImmortalWRT，支持雅典娜 LED 灯控制，但是自编译的版本由于 hash 与官方固件不匹配，无法在线安装官方的 `kmod` 依赖（安装 Docker 时需要用到）。这里有两种方式可以解决：

1. 修改源码中的 hash 值生成逻辑来绕过 APK 安装时的校验。（但是有个别 kmod 包安装时会强制重启，所以后边结合第二种方式来解决的）再通过指定 APK 源的方式在线安装。

   ```bash
   apk add dockerd --repository=https://downloads.immortalwrt.org/snapshots/targets/qualcommax/ipq60xx/kmods/6.6.71-1-ff75615298a003c0b4ac7104bd8904ff/packages.adb
   ```

   参考：[CSDN 博客](https://blog.csdn.net/bjr2016/article/details/107776801)

2. 在编译时指定缺少的 kmod 包。
   [SAMYJ1/OpenWRT-CI](https://github.com/SAMYJ1/OpenWRT-CI/blob/main/Config/GENERAL.txt#L27)

   用 Github Action 云编译后直接 U-Boot 刷固件即可。

## 2. 安装 Docker & Home Assistant

### 2.1 磁盘分区

安装 Docker 之前最好先给磁盘分好区，省得后边再改了。

1. 首先使用 `fdisk -l` 来查看当前系统的磁盘名称。
2. 使用磁盘分区工具对该磁盘进行分区，使用命令 `cfdisk /dev/mmcblk0`。
3. 进入界面后选择 FreeSpace，选择 New 后按回车，根据提示一步步操作就行。
4. 接着对新建的分区进行格式化，可以再次使用 `fdisk -l` 来检查一下创建情况，可以看到新分区的名字叫 `/dev/mmcblk0p27`，接着继续格式化：`mkfs.ext4 /dev/mmcblk0p27`。
5. 格式化完成后把 `/dev/mmcblk0p27` 挂载到 `/opt` 上：

   ```bash
   mount -t ext4 /dev/mmcblk0p27 /opt
   ```

### 2.2 安装 Docker

```bash
apk update
apk add luci-i18n-docker-zh-cn
apk add luci-i18n-dockerman-zh-cn
```

这里如果报错缺少依赖则参考 1.3。

### 2.3 安装 Home Assistant

1. SSH 连接路由器，在 home 目录下创建一个 Docker 配置目录，新建一个 Docker Compose 配置。

   ```bash
   cd ~
   mkdir docker && cd docker
   vim compose.yaml
   ```

   ```yaml
   services:
     homeassistant:
       image: lscr.io/linuxserver/homeassistant:latest
       container_name: homeassistant
       network_mode: host
       privileged: true
       environment:
         - TZ=Asia/Shanghai
       volumes:
         - /opt/homeassistant/data:/config
       restart: unless-stopped
   ```

2. 保存后直接在当前目录执行 `docker compose up -d`，Home Assistant 就跑起来了。

### 2.4 配置 Home Assistant 并添加 HomeKit

> 到上一步本来已经以为搞差不多了，结果添加 HomeKit 之后扫二维码一直连不上。后来才知道是苹果设备是走 mDNS 来发现设备的，需要把 mDNS 请求转发到 Docker。

#### 2.4.1 OpenWRT 跨网段解析 mDNS

1. 在路由器上安装 avahi-daemon（上边的固件自带了）。
2. 配置 avahi 跨网段解析，编辑 `/etc/avahi/avahi-daemon.conf`。

   ```ini
   [server]
   use-ipv4=yes
   use-ipv6=yes
   check-response-ttl=no
   use-iff-running=no
   # 添加这一行
   allow-interfaces=br-lan,wan

   [reflector]
   # 把这里改成 yes
   enable-reflector=yes
   reflect-ipv=no
   ```

3. 重启一下服务。

   ```bash
   /etc/init.d/avahi-daemon restart
   /etc/init.d/avahi-daemon enable # 设置开机启动
   ```

4. 配置 OpenWRT 防火墙通信规则：
   网络 -> 防火墙 -> 通信规则 -> 新增。

   <img src="/resources/firewall-mdns.png" alt="mDNS 配置" style="width: 50%;">

   > mDNS 使用 UDP 组播进行通信，组播 IPv4 地址为 `224.0.0.251` 端口为 `5353`。

5. 配置 HomeKit TCP 端口转发，一般为 `21063`，否则可能出现添加设备后无响应的问题。
   网络 -> 防火墙 -> 端口转发 -> 新增。

   <img src="/resources/firewall-hk-tcp.png" alt="HomeKit TCP 转发" style="width: 50%;">

6. 配置完端口转发应该就能连上 HomeKit 了，如果还有问题可以试着开一下 `IGMP`（ps: 我这里没验证）。

## 3. MacOS 设置自动化控制设备

这里我的需求是电脑解锁时开启屏幕挂灯，锁屏时关闭，但是 MacOS 捷径没有自动化选项，所以这里需要借助一个第三方工具 `hammerspoon` 来实现。由于电脑锁屏后再用快捷指令调用 HomeKit 会失败，所以这里就直接用 HomeKit Webhook 的形式实现了。

```bash
brew install hammerspoon --cask
# 编辑配置文件
vim ~/.hammerspoon/init.lua
```

```lua
hs.caffeinate.watcher
 .new(function(event)
  if event == hs.caffeinate.watcher.screensDidUnlock then
   hs.execute('curl -X POST "http://192.168.10.1:8123/api/webhook/turn_on_living_room_screen_light"')
  elseif event == hs.caffeinate.watcher.screensDidLock then
   hs.execute('curl -X POST "http://192.168.10.1:8123/api/webhook/turn_off_living_room_screen_light"')
  end
 end)
 :start()
```

对应 Home Assistant 里边创建好自动化就 OK 了。
