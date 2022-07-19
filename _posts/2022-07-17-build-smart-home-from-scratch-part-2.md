---
layout: post
title: "(草稿WIP) 从头打造全屋智能家庭 (持续更新), Part II - 硬件选择, 实操以及注意事项"
---


# 硬件选择

> 具体的智能硬件方面目前是 Aqara (绝大部分) 和 Tuya, 可以完美的通过 Zigbee 配对, 离线控制. 而且 Zigbee2MQTT 里面可以直接 OTA 升级硬件官方固件. 下面介绍的都是一些周边的硬件.

#### Uunifi Dream Machine PRO Special Edition, UDM-PRO-SE
核心网络设备, 因为有各个房间ap面板(目前还没安装), 和自己托管的监控方案的需求. 直接一步到位购入了 UDM-PRO-SE, 其中 SE 是升级版, 多了POE和POE+供电, 正好适用于摄像头和AP. 不用再买一台POE的交换机(如果设备不多的话)

Unifi 的产品最方便的是可以无缝接入 Home Assistant, 包括客户端信息 (是否联网, 所在AP). Unifi Protect 也可以无缝接入, 不仅仅是有视频源, 还有摄像头自带的动作侦测, 开启红外等功能均已接入

入坑指南: https://www.chiphell.com/thread-2421885-1-1.html

#### SONOFF ZBDongle-Plus 
Zigbee天线, 大概100左右. 支持 `Zigbee 3.0` 协议. 即插即用, 固件更新也很方便, 可以直接使用 PC/树莓派 等设备直接通过 usb 更新. `Zigbee2MQTT` 完美支持.

#### N5105 NUC 11 PC
此 NUC 只花费了1000元, 有4核8G, 用来运行 Home Assistant 再合适不过 (因为没预算了😂). Zigbee Dongle 便插在此机器上


#### 智能硬件
> 目前已购买使用的有如下这些, 已经全部通过 Zigbee 接入(没有使用 Aqara 或米家 app)
 
* Aqara系的
 * 智能电动窗帘电机zigbee版
 * 门窗传感器 E1
 * 水浸传感器 E1
 * 温湿度传感器
 * 烟雾报警器
 * 智能天然气报警器
 * 智能无线墙壁开关 D1
 * 墙壁开关D1版-零火三键


# 实操
 
 #### 安装 Home Assistant
 > https://www.home-assistant.io/installation/
 
 如果是使用 RaspberryPi, 可以直接使用官方 image 直接烧录. 
 
 我由于还要在机器上单独装 Prometheus, Grafana 等服务, 所以是在 Debian 的基础上安装的. 如果你也是在现有系统上装 Home Assistant 的话, 强烈建议使用 Supervised 的方式安装, 这样后续安装 Addon 的时候可以网页 GUI 一键安装. 步骤大致如下:
 
```sh
#1 如果没有安装 Docker 的话
sudo apt-get update -y
sudo apt-get install ca-certificates curl gnupg lsb-release -y
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y

#2 安装 Home Assistant os-agent
wget https://github.com/home-assistant/os-agent/releases/download/1.2.2/os-agent_1.2.2_linux_x86_64.deb
sudo dpkg -i os-agent_1.2.2_linux_x86_64.deb
gdbus introspect --system --dest io.hass.os --object-path /io/hass/os #确认安装成功

#3 安装 Home Assistant Supervised
apt-get install jq wget curl udisks2 libglib2.0-bin network-manager dbus -y
wget https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb
dpkg -i homeassistant-supervised.deb
```
 具体还可以参考 [这里](https://github.com/home-assistant/supervised-installer) [这里](https://www.home-assistant.io/installation/linux#install-home-assistant-supervised) 还有 [这里](https://github.com/home-assistant/os-agent/tree/main#using-home-assistant-supervised-on-debian=)
 
 #### NodeRed && Zigbee2MQTT
 
 #### NodeRed 自动化
 


# 注意事项

#### 如果还在装修或者还没有装修, __一定要在开关盒里加零线__
如果开关盒里没有零线的话, 就只能用单火的智能开关 (原理图如下). 单火的智能开关在物理上是一直有电 (没有零线, 可以理解为DC里面的地线/0V Ref), 一般是一个可调电阻来通断. 所以终端 --- 灯泡会在开关断开后也会微弱的闪 (取决于灯泡和开关质量).
而零火的开关是真正的通断, 控制电路是通过继电器来进行物理的通断.
![](/images/img-build-smart-home-from-scratch-part-2-1.png)

#### 双 16A 插座 (或者 16A + 10A)
因为智能窗帘的电机一般是插空调的插座上, 但是一般只留了一个 16A 三孔插座, 窗帘电机(两台电机可以串联)也需要一个三孔插座. 所以我自己买了一个双三孔(兼容16A和10A)的86面板替换.

![](/images/img-build-smart-home-from-scratch-part-2-2.jpg)

#### 升级 SONOFF ZBDongle

部分智能硬件产品对 Zigbee Dongle 插件版本有要求, 需要升级, 如下是我用到的方法

视频: https://www.youtube.com/watch?v=iCE5Z43EKpk

固件: https://github.com/Koenkk/Z-Stack-firmware/tree/master/coordinator/Z-Stack_3.x.0/bin  一定要下载 `CC1352P2_CC2652P_launchpad_*.zip`

脚本: https://github.com/JelmerT/cc2538-bsl 如果脚本没跑成功, 多半是依赖没装, __记得要把 Zigbee2MQTT Addon 停用__.

文本教程: https://sonoff.tech/wp-content/uploads/2022/01/SONOFF-Zigbee-3.0-USB-dongle-plus-firmware-flashing-.pdf


_(未完待续...)_
