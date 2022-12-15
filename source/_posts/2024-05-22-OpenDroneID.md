---
title: OpenDroneID协议研究：无人机远程识别的开放标准
date: 2024-05-22 10:00:00
tags:
  - 无人机
  - Remote ID
  - 协议
categories:
  - 技术研究
---

## 背景

<!-- more -->

全球无人机数量在过去几年快速增长，从消费级航拍无人机到工业级物流无人机，低空空域的使用密度显著提升。随之而来的问题是：当空中出现一架无人机时，地面人员如何知道它是谁的、从哪里起飞、在执行什么任务？

传统做法是依靠机场塔台的雷达和目视观察，但这些手段对小型消费级无人机基本无效——它们体积太小，雷达反射截面很小，而且很多飞行活动发生在雷达覆盖不到的区域。监管机构需要一种更直接的方式来识别这些"无名"的飞行器。

2021年，美国联邦航空管理局（FAA）正式发布了Remote ID最终规则，要求在美国注册的无人机必须具备广播身份信息的能力。欧盟航空安全局（EASA）也推进了类似的Direct Remote ID法规。日本国土交通省（MLIT）在2022年6月20日起强制要求广播式Remote ID。

这些法规催生了一个关键需求：需要一个开放、可互操作的协议来统一Remote ID的实现方式。OpenDroneID就是这个需求的产物——它是ASTM F3411和ASD-STAN prEN 4709-002标准的开源实现。

## 什么是Remote ID

Remote ID（远程识别）本质上是给无人机加了一个"电子车牌"。它通过无线电广播的方式，将无人机的身份信息、位置信息、操作者信息等数据发送给周围的接收设备。

Remote ID有两种主要实现方式：

1. **广播式Remote ID（Direct Remote ID）**：无人机直接通过蓝牙或Wi-Fi向周围广播识别信息，不依赖互联网连接。这是OpenDroneID主要覆盖的场景。

2. **网络式Remote ID（Network Remote ID）**：无人机通过蜂窝网络或互联网将识别信息发送到在线服务，接收者通过网络查询获取信息。

FAA的规则主要要求广播式Remote ID。操作员有两种合规路径：购买内置Remote ID功能的新无人机，或者为现有无人机加装独立的Remote ID广播模块。

## OpenDroneID协议架构

OpenDroneID定义了6种消息类型，外加1种用于将多条消息打包的Message Pack类型：

### 1. Basic ID（基本标识）

包含无人机的核心身份信息：

- **UAS ID**：无人机的唯一序列号，遵循ANSI/CTA-2063-A标准
- **UA Type**：无人机类型，取值包括：
  - Fixed wing（固定翼）
  - Multi-rotor（多旋翼）
  - Helicopter（直升机）
  - Hybrid Lift（混合升力）
  - Other（其他）
- **ID Type**：标识符类型（序列号、CAA注册号、UTM执行标识符等）

### 2. Location（位置）

包含无人机的实时动态位置信息：

- **Latitude**：纬度，单位为10^-7度
- **Longitude**：经度，单位为10^-7度
- **Altitude Type**：高度参考类型
  - Above Takeoff Point（起飞点以上，即AGL）
  - Above Sea Level（海拔高度，即WGS-84椭球高度）
- **Altitude**：高度值，单位为米
- **Heading**：航向角，单位为度（0-360）
- **Speed**：水平速度，单位为m/s
- **VSpeed**：垂直速度，单位为m/s（正值上升，负值下降）
- **Timestamp**：时间戳，自UTC 2022年1月1日起的秒数

### 3. Authentication（认证）

用于身份验证和防篡改。协议定义了4种认证页面类型：
- Account ID（账户ID）
- Authentication Code（认证码，基于位置/时间的散列）
- Session ID（会话ID）
- Message Set Signature（消息集签名）

认证数据在16个页面之间循环传输。

### 4. Self-ID（自标识）

操作者可选填的明文描述信息，比如"航拍摄影"、"农业喷洒"等用途说明。

### 5. System（系统）

包含操作者和运营系统的信息：

- **Operator Location Type**：操作者位置类型（固定/动态/未知）
- **Operator Latitude/Longitude**：操作者位置坐标
- **Operator Altitude**：操作者海拔高度
- **Area Count**：同时操控的无人机数量
- **Area Radius**：运营区域半径
- **Area Ceiling**：运营区域高度上限
- **Area Floor**：运营区域高度下限
- **Classification**：EASA分类（Open/Specific/Certified）
- **Timestamp**：时间戳

### 6. Operator ID（操作者ID）

操作者的注册标识符，如FAA的Remote ID编号。

### 7. Message Pack（消息包）

将上述多种消息打包为一个整体。Wi-Fi NaN、Wi-Fi Beacon和蓝牙5.0 Extended Advertising使用Message Pack来传输数据，而蓝牙4.x Legacy Advertising由于25字节的广播数据限制，只能逐条发送单个消息。

## 广播方式

OpenDroneID支持四种物理层广播方式：

### 蓝牙4.x Legacy Advertising

最基础的广播方式，兼容性最好。但蓝牙4.x的广播数据包限制为31字节，一条Remote ID消息通常超过这个限制，因此需要将数据拆分到多个广播包中发送。接收端需要重组这些包才能还原完整消息。至少每333ms广播一次，以满足每秒3条消息的最低要求。

### 蓝牙5.0 Long Range with Extended Advertising

蓝牙5.0扩展了广播数据包的长度限制，支持发送完整的Message Pack。同时支持长距离模式（Coded PHY），可以显著扩大广播覆盖范围。欧盟法规将此作为强制要求。

### Wi-Fi Neighbor Awareness Networking (NaN)

Wi-Fi NaN（也叫Wi-Fi Aware）是一种设备间直接发现和通信的协议，不需要接入点。无人机通过NaN服务广播Remote ID数据，范围内的接收设备可以自动发现并接收这些数据。支持2.4GHz和5GHz频段。美国和欧盟法规都将此列为强制支持。

### Wi-Fi Beacon

在标准Wi-Fi Beacon帧中嵌入Remote ID数据（使用厂商自定义元素）。这种方式不需要设备之间建立连接，任何监听Wi-Fi Beacon的设备都能接收到数据。

## 与其他标准的关系

OpenDroneID不是一个孤立的标准，它处于多个标准体系的交汇点：

- **ASTM F3411**：美国的Remote ID标准，OpenDroneID的v3版本基于ASTM F3411 v1.1（F3411-22a）
- **ASD-STAN prEN 4709-002**：欧洲的Direct Remote ID标准，与ASTM F3411 v1.1保持对齐
- **IETF DRIP**：IETF的Drone Remote ID Protocol工作组定义了Remote ID的安全和信任框架。已发布的RFC包括RFC 9153（需求和术语）、RFC 9374（RID实现）、RFC 9434（架构）。DRIP在ASTM的广播基础上增加了HIT（Hash-based ID）等安全机制
- **MAVLink**：OpenDroneID定义了对应的MAVLink消息，飞控系统可以通过MAVLink将位置和身份信息发送给Remote ID广播模块

## 各区域法规差异

不同地区对Remote ID的要求存在差异：

| 特性 | FAA规则 | ASTM v1.1 | EU规则 | 日本规则 |
|------|---------|-----------|--------|----------|
| 序列号(ANSI/CTA-2063-A) | 必需 | 必需 | 必需 | 必需 |
| 动态位置 | 必需 | 必需 | 必需 | 必需 |
| WGS-84高度 | 必需 | 必需 | - | 必需 |
| AGL/起飞点高度 | - | 可选 | 必需 | 可选 |
| 操作者动态位置 | 必需 | 可选 | 必需 | 可选 |
| 广播间隔 | 1秒 | 1或3秒 | - | 1秒 |
| BT5 Long Range | - | 可选 | 必需 | 必需 |
| Wi-Fi NaN | - | 必需 | 必需 | 必需 |

## 安全问题

OpenDroneID的广播数据既没有加密也没有签名验证，这是一个已知的设计权衡。这意味着：

1. **隐私问题**：任何人都能接收和解码无人机的广播数据，包括操作者位置
2. **伪造攻击**：攻击者可以注入伪造的Remote ID数据，制造虚假的无人机存在
3. **数据篡改**：中间人可以修改广播中的位置或身份信息

IETF DRIP工作组正在通过HIT和签名机制解决部分安全问题，但这些方案在当前的OpenDroneID实现中尚未广泛部署。Nozomi Networks等安全厂商已经开发了检测异常ODID流量（如不合理的遥测数据、重复消息、异常流量激增）的能力。

## 开源实现

opendroneid-core-c是OpenDroneID的官方C语言库，提供了消息编码和解码功能。主要的硬件实现包括：

- **ArduRemoteID**：基于ESP32-S3/C3的固件，支持BT4、BT5和Wi-Fi，与ArduPilot飞控集成
- **ESP32实现**：基础ESP32仅支持BT4 Legacy Advertising，ESP32-S3/C3支持BT5
- **Linux发送器**：支持Wi-Fi NaN的Linux实现

接收端方面，Android有官方的OpenDroneID OSM应用和Dronetag的DroneScanner；iOS方面由于苹果的限制，仅支持BT4 Legacy Advertising的接收。

## 总结

OpenDroneID作为Remote ID的开源实现，解决了一个实际问题：在无人机数量快速增长的背景下，如何让监管机构和公众识别空中的飞行器。协议本身的技术设计相对简洁——基于蓝牙和Wi-Fi的广播机制，消息格式清晰，实现门槛不高。

但安全方面的缺陷是显而易见的。缺乏加密和认证意味着协议更容易被伪造和攻击，IETF DRIP的扩展方案是否能有效解决这些问题还有待观察。在法规强制推动和实际安全需求之间，OpenDroneID目前选择了先实现基本功能，安全增强留待后续版本。
