---
layout: post
title: "从头打造全屋智能家庭 (持续更新), Part I - 网关平台/协议及软件的选择"
---

先上个全家福
![](/images/img-build-smart-home-from-scratch-part-1-1.jpg)


## 序

在此之前, 我已经在租来的房间中尝试了多种智能家居以及网关系统. 此次借搬入新买的房子的之机, 我将从头打造一个全屋智能的智慧家庭. 然后也记录下来, 与君共享. 一个是分享当中一些误区, 坑点. 二是以后改进时有据可循.

## 内容规划

 * 网关平台/协议选择
 * 软件选择
 * 硬件选择
 * 实操
 * 电路改造?
 * 待续

## 网关平台/协议选择

# 平台

目前市面上每个厂商都有自己的智能家居平台, `米家`, `涂鸦`, `博联`, `HomeKit`, `Google Home`, `Alexa` 等一众平台厂商可供选择. 

# 协议

通讯协议分别又有 `Wif`, `蓝牙`, `Zigbee,` `Z-wave` 等各类协议. 其中 `Z-wave` 因为使用的 _865-926 MHz_ 波段在国内不属于公共领域, 使用起来是违法的, 所以我们不能考虑. `Wifi` 和`蓝牙`的大部分设备都不支持完全离线运行, 需要定时和云端的服务器通信来刷新令牌. 当断开网络过久, 令牌过期时便不受控制. 

所以, 我选择了使用 `Zigbee` 的智能设备. 一是因为支持 `Zigbee`的设备市面比较多, 我买过的有 `Aqara (绿米)`, `涂鸦系`两大厂牌的设备均是支持 `Zigbee` 协议的. 二是使用 `Zigbee` 协议的设备不需要使用厂家的任何网关, App, 云服务. 只需本地一个支持 `Zigbee` 协议的任意网关设备即可. 而且 `Zigbee` 协议是可以 _Mesh_ 的, 任意接通有线电源的设备均可作为一个 __路由节点__(`Zigbee` 协议中的 __信号中继节点__ ).

唯一有可能有影响的是 `Zigbee` 使用的是 _2.4 GHz_ 波段, 和 `2.4G Wifi` 有冲突的可能. 但是根据我的使用经验, 影响可以说是有限的. 除非你有很多的 AP 重叠在一块.

## 软件选择

# 前端

所谓前端, 便是与用户打交道的这一块. 由于我是重度苹果用户, 自然而然的便选择了 `HomeKit` 作为交互界面. 可以通过 _HomePod_  和 _Apple TV_ 来进行语音交互. 而且手机中的 _Home App_ 天然的支持在外网访问 (不在家中时), 安全也是相对的有保障.

# 后端

虽然可以直接 All In `HomeKit`, 但是使用体验下来其自动化方面, 还有支持的设备方面可以说是比较弱的. 所以很早我便开始了 `Home Assistant` 的尝试, 中间只是作为第三方硬件通过 _HomeKit Accessory Protocol_ (_HAP_)  接入到 `HomeKit` 生态. 然后在一年的使用当中发现, `HomeKit` 对于自动化的支持并不是很全面和易用 (例如夜晚分时段调整空调温度, 在 iOS 15之前只能通过 `Home+` 这个第三方 App 来编写).

虽然说后端选择了 `Home Assistant`, 但是由于其强大的扩展性, 我们相当于只是选择了一个智能家居的操作系统. 还有很多 _软件_(Addons & Integrations) 需要我们选择.

我接下来会介绍几个非常核心的必备 _软件_(__Addons__ & __Integrations__).

#### Zigbee2Mqtt (Addon) 和 mosquitto (Addon & Integration)

在上面说到, 我们选择了 `Zigbee` 作为通讯协议. 但要将其接入到智能家居系统中, 我们还需要一些额外的操作. 其中 `Mosquitto` 是一个 _MQTT Broker_,  消息队列. 我们需要同时装 `Mosquitto`的 __Addon__ 和 __Integration__. __Addon__ 是 `Mosquitto` 本身. __Integration__ 是连接 _MQTT_ 和 `Home Assistant`的桥梁, 将设备注册消息转化为 `Home Assistant` 的 __Entity__, 及相关的设备读写操作.

`Zigbee2Mqtt` 则是将 `Zigbee` 协议的设备通信同步到 _MQTT_(接下来硬件篇也会提及), 这样完成 `Zigbee` 设备到 `Home Assistant` 的闭环.

#### Node-RED (Addon)

`Home Assistant` 虽然支持自动化脚本编写, 但是由于其 yaml 的组织方式, 编辑起来并不是特别的好理解, 调试.  `Node-RED` 是一个图形化节点编程工具. 而这个 __Addon__ 可以让你通过图形化节点编程的方式来组织你的自动化流程, 而且可以方便的调试, debug.

#### HomeKit (Integration)

`HomeKit` __Integration__ 是打通 `HomeKit` 和 `Home Assistant` 的桥梁.

#### Apple TV (Integration)

虽然这个 __Integration__ 叫做 `Apple TV`, 但是其也可以整合 `HomePod`. 这个 __Integration__ 可以让 `Apple TV` 或 `HomePod` 作为一个媒体播放器, 可以将监控视频通过 _Airplay_ 协议投屏, 也可以使用 _TTS_ 实时生成语音投放.



_(未完待续...)_
