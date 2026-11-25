


> 局域网环境下，用 **MQTT over WebSocket** 传输摄像头视频流。
> 服务端 C++/Qt 6，客户端 uni-app（Android/iOS），端到端延迟 200ms 级。

先看效果：手机端app和电脑端画面几乎同步
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4fe28a34a6d440a0b5f8fb5adb5c6a08.png#pic_center)

## 目录

- [一、项目简介](#一项目简介)
- [二、核心功能](#二核心功能)
- [三、技术选型](#三技术选型)
- [四、系统架构](#四系统架构)
- [五、通讯协议](#五通讯协议)
- [六、关键实现](#六关键实现)
- [七、界面预览](#七界面预览)
- [八、快速上手](#八快速上手)
- [九、性能实测](#九性能实测)
- [十、常见问题](#十常见问题)
- [十一、项目结构](#十一项目结构)
- [十二、从源码编译](#十二从源码编译)
- [十三、为什么不用 WebRTC](#十三为什么不用-webrtc)
- [十四、扩展方向](#十四扩展方向)

---

## 一、项目简介

MJ Camera 是一套**局域网视频传输系统**，实现「摄像头采集 → 编码 → 跨设备传输 → 实时解码渲染」的完整链路。

核心目标是用 **MQTT over WebSocket** 在局域网内传输实时视频流：

```
摄像头 ──采集──▶ JPEG 编码 ──▶ MQTT(over WebSocket) ──▶ 接收端解码渲染
                                    │
                              局域网 192.168.x.x
```

### 解决的技术问题

在局域网里做视频传输，可选方案不多：

| 方案 | 问题 |
|---|---|
| **HTTP MJPEG 轮询** | 每个请求传一帧，HTTP 头开销大，帧率上不去，并发一多就崩 |
| **HLS / DASH** | 切片延迟以秒计，完全不适合实时监看 |
| **RTSP/RTP** | 客户端要处理 RTCP、丢包重传、NAT 穿透，实现复杂 |
| **WebRTC** | 需要信令服务器 + STUN/TURN 部署，客户端体积大，局域网场景收益低 |
| **裸 TCP 粘包** | 要自己设计分帧、粘包拆包、心跳、断线重连 |
| **HTTP-FLV** | 依赖 flv.js，流媒体生态不通用 |
| ✅ **MQTT over WebSocket** | **协议成熟、自带 QoS/心跳/重连、跨平台、客户端可手写** |

**选 MQTT 的理由**：它是**为低延迟消息传输设计**的成熟协议，已经解决了心跳（`PINGREQ/PINGRESP`）、
连接保活、订阅管理、QoS 等级这些工程难题。视频帧本质上是「带时间戳的大消息」，
用 MQTT 传非常契合。而且它只需要 **WebSocket 握手 + 5 种控制报文**，客户端可以手写得很轻。

### 适用场景

- 📷 局域网内的摄像头实时监看 / 视频回传
- 🖥️ 自研客户端接入（本文档提供了完整协议说明，可自行实现）
- 🏠 家庭 / 门店 / 实验室的简易视频监控
- 🔧 设备画面远程查看（产线、机房、老家店铺）

---

## 二、核心功能

### 服务端（PC 端）

| 功能 | 说明 |
|---|---|
| **多设备枚举** | 通过 Media Foundation 枚举系统内全部视频设备（USB 摄像头、采集卡、虚拟摄像头…） |
| **运行时切换** | 界面下拉框随时切换采集设备，无需重启 |
| **五档画质** | 超清 / 高清 / 均衡 / 低画质 / 极速，切换后立即生效，手机端同步 |
| **自适应缩放** | 按原始宽高比换算目标宽度，**不放大、不变形** |
| **实时状态面板** | FPS、推流分辨率、已推帧数、连接设备数 |
| **独立日志浮窗** | 独立顶层窗口，贴主界面右侧随主界面移动，显示 MQTT 连接详情 |
| **IP 自动检测** | 监听网卡变化，Wi-Fi 网卡优先于虚拟/有线网卡，支持手动刷新 |
| **停止推流** | 只停止向客户端推帧，**本地采集与预览保持运行** |
| **设备黑名单** | 配置文件中屏蔽指定设备名（如虚拟摄像头、系统假设备） |
| **HTTP REST API** | 状态查询、画质切换、设备枚举、快照抓拍、推流开关 |

### 客户端（移动端）

| 功能 | 说明 |
|---|---|
| **自研 MQTT 客户端** | 手写 MQTT 3.1.1 报文编解码，不依赖 `mqtt.js`（省 ~200KB） |
| **三级服务发现** | UDP 广播 → 高概率优先扫描 → 全网段兜底，换网络也能自动重连 |
| **时钟偏差校准** | 最小值滤波估计器，修正 PC / 手机系统时间不同步导致的延迟失真 |
| **Canvas 渲染** | 帧队列 + 懒 base64 编码 + 丢弃过期帧，避免消息堆积 |
| **画质档位面板** | 与服务端配置共用同一份档位定义 |
| **实时指标显示** | 通道类型、实际帧率、端到端延迟 |
| **地址持久化** | 记住上次成功连接的地址 |

### 通用特性

- 🟢 **绿色免安装**，无 Python / OpenCV / FFmpeg 运行时依赖
- 🟢 **~31MB** 完整分发包，单 exe 仅 0.4MB
- 🟢 **~60MB** 运行时内存占用
- 🟢 源码统一 UTF-8，中文零乱码
- 🟢 `PerMonitorV2` 高 DPI 感知，4K / 150% 缩放下窗口尺寸不跳变

---

## 三、技术选型

### 服务端

| 模块 | 选型 | 理由 |
|---|---|---|
| 语言 / 框架 | **C++17 + Qt 6.11** | 原生性能、跨平台、QtNetwork 自带完整 TCP/WebSocket 栈 |
| 采集 | **Media Foundation** | Windows 官方采集 API，零依赖，支持所有标准视频设备 |
| 编码 | **WIC (Windows Imaging Component)** | 系统自带 JPEG 编码器，无需 OpenCV/libjpeg |
| HTTP 服务 | **QtHttpServer** | Qt 6 内置，声明式路由 |
| 传输协议 | **MQTT 3.1.1 over WebSocket** | 自实现 Broker，复用 Qt 的 QTcpSocket |
| 界面 | **Qt Widgets + QSS** | 原生观感，样式表定制 |

**为什么不用 OpenCV / FFmpeg？**

- OpenCV 会引入 ~80MB 依赖，且它的视频采集后端在 Windows 上仍绕不开 Media Foundation
- FFmpeg 需要外带 `ffmpeg.exe`（~80MB），且要处理子进程管道、僵尸进程、编码器参数差异
- Media Foundation + WIC 都是 **Windows 系统自带 API**，真正做到零外部依赖

### 客户端

| 模块 | 选型 | 理由 |
|---|---|---|
| 框架 | **uni-app (Vue3)** | 一套代码输出 Android / iOS |
| MQTT | **手写实现** | `mqtt.js` 的 wx transport 在 App 端不可用（`socketTask.onOpen is not a function`） |
| 渲染 | **Canvas 2D** | 比 `<image>` 元素可控，能精确控制丢帧策略 |
| 服务发现 | **plus.android + 原生 DatagramSocket** | App-Plus 运行时没有 `uni.createUDPSocket` |

---

## 四、系统架构

```
┌──────────────────────────┐         局域网         ┌──────────────────────┐
│      服务端 (Windows)     │                        │     客户端 (Android/iOS)│
│                          │   HTTP  :8080          │                      │
│  MJ_CAMERA_PC.exe        │ ─────────────────────▶ │  ┌────────────────┐  │
│                          │                        │  │  UI (uni-app)  │  │
│  ┌────────────────────┐  │   MQTT  :1883          │  └───────┬────────┘  │
│  │ CaptureThread      │  │ ─────────────────────▶ │          │           │
│  │  MFSourceReader    │  │                        │  ┌───────▼────────┐  │
│  │   ↓ CVPixelBuffer  │  │   WebSocket  :8081     │  │ MqttStream     │  │
│  │  JpegEncoder (WIC) │  │ ─────────────────────▶ │  │ (手写 MQTT)     │  │
│  │   ↓ JPEG           │  │                        │  └───────┬────────┘  │
│  │  FrameHub          │  │                        │          │           │
│  └─────────┬──────────┘  │                        │  ┌───────▼────────┐  │
│            │             │                        │  │ ClockSync      │  │
│  ┌─────────▼──────────┐  │                        │  │ (时钟校准)      │  │
│  │ StreamPumpThread   │  │                        │  └───────┬────────┘  │
│  │  组装 MJF1 帧      │  │                        │          │           │
│  └─────────┬──────────┘  │                        │  ┌───────▼────────┐  │
│            │             │                        │  │ CanvasRenderer │  │
│  ┌─────────▼──────────┐  │                        │  └────────────────┘  │
│  │ MqttBroker         │  │                        └──────────────────────┘
│  │  topic 匹配分发    │  │
│  └────────────────────┘  │
└──────────────────────────┘
```

### 线程模型

服务端是**多线程**的，每个 MQTT 连接独立一个 socket 线程：

| 线程 | 职责 |
|---|---|
| **主线程 (GUI)** | UI 渲染、HTTP 服务、Broker 控制、设备枚举 |
| **CaptureThread** | 设备采集 → 缩放 → JPEG 编码 → 写入 FrameHub |
| **StreamPumpThread** | 从 FrameHub 取最新帧 → 组装 MJF1 → 推给 Broker |
| **SocketThread (per session)** | 每个 MQTT 连接一个线程，负责网络读写 |

---

## 五、通讯协议

### 端口分配

| 端口 | 协议 | 用途 |
|---|---|---|
| **8080** | HTTP/REST | 状态查询、控制指令、快照 |
| **1883** | MQTT over WebSocket | **主推流通道** |
| **8081** | 原生 WebSocket | 备用推流通道（自定义帧格式） |

### MQTT over WebSocket 握手

MQTT 报文跑在 WebSocket 之上，所以先做 WebSocket 升级：

```http
GET /mqtt HTTP/1.1
Host: 192.168.1.100:1883
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: <base64 编码的 16 字节随机数>
Sec-WebSocket-Protocol: mqtt
Sec-WebSocket-Version: 13
```

服务端响应：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: <base64(SHA1(key + GUID))>
Sec-WebSocket-Protocol: mqtt
```

> **Sec-WebSocket-Protocol 必须带上 `mqtt`** —— 这是 MQTT over WebSocket 的标准约定，
> 客户端和服务端通过子协议协商确认双方都支持 MQTT 语义。

### 报文流程

```
Client                                Server
  │────── WebSocket 握手 (101) ──────────▶│
  │                                     │
  │──── CONNECT (0x10) ─────────────────▶│   协议名 "MQTT", 版本 4, clean session
  │◀─── CONNACK (0x20) ──────────────────│   Session Present=0, ReturnCode=0
  │                                     │
  │──── SUBSCRIBE topic=mj/camera/frame ▶│   PacketId=1, QoS=0
  │◀─── SUBACK (0x90) ───────────────────│   Granted QoS=0
  │                                     │
  │◀─── PUBLISH (0x30) ──────────────────│   ← 持续推送视频帧
  │◀─── PUBLISH (0x30) ──────────────────│
  │      ...                            │
  │                                     │
  │──── PINGREQ (0xC0) ────────────────▶│   每 25s 一次，keepalive
  │◀─── PINGRESP (0xD0) ─────────────────│
```

### 服务端使用的 MQTT 报文类型

| 类型 | 值 | 方向 | 用途 |
|---|---|---|---|
| CONNECT | `0x10` | C→S | 建立连接，携带协议名/版本/keepalive/clientId |
| CONNACK | `0x20` | S→C | 连接确认，返回 Session Present 和 ReturnCode |
| PUBLISH | `0x30` | S→C | **推送视频帧**（Topic: `mj/camera/frame`） |
| SUBSCRIBE | `0x82` | C→S | 订阅视频主题 |
| SUBACK | `0x90` | S→C | 订阅确认 |
| PINGREQ | `0xC0` | C→S | 心跳请求 |
| PINGRESP | `0xD0` | S→C | 心跳响应 |

**Remaining Length 字段**采用 MQTT 的变长整数编码（每字节低 7 位是数据，最高位是 continuation flag），
因为单帧 JPEG 可能超过 127 字节，必须支持多字节长度。

### MJF1 视频帧格式

PUBLISH 报文的 payload（应用数据）使用自定义二进制帧：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Magic 'MJF1'                         |  0x4D 0x4A 0x46 0x31
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                   Timestamp (Unix ms, Big-Endian)             |  8 bytes
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                     Sequence Number (Big-Endian)               |  4 bytes
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                                                               |
|                        JPEG Payload                            |  N bytes
|                        (variable length)                       |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                     Sequence Number (Big-Endian)               |  4 bytes (重复)
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
|---|---|---|
| **Magic** | 4B | 固定 `MJF1`（`0x4D 0x4A 0x46 0x31`），用于快速识别帧起始，非法数据直接丢弃 |
| **Timestamp** | 8B | 服务端采集时刻，Unix 毫秒，**大端**，用于端到端延迟计算 |
| **Sequence** | 4B | 帧序号 `uint32`，**大端**，头尾各一份，可用于丢帧检测与乱序检测 |
| **JPEG** | N | 原始 JPEG 字节流，无额外封装 |

> **为什么时间戳和序号都用大端？**
> 大端（网络字节序）是协议惯例，且 JavaScript 的 `DataView.getUint32(offset, false)`
> 默认就是大端，客户端解析时零成本。
>
> **为什么序号写两遍？**
> 尾部序号让接收端可以**校验**帧是否完整——如果头尾序号不一致，说明这帧被截断了。
> 在 WebSocket 之上其实有 TCP 保证，但这个冗余在 UDP 场景或跨代理转发时很有价值。

### HTTP API

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/status` | 服务端状态快照 |
| GET | `/api/config` | 当前配置 + 全部画质档位定义 |
| GET | `/api/quality` | 查询当前画质档位 |
| POST | `/api/quality` | 切换画质，body: `{"preset": "high"}` |
| GET | `/api/cameras` | 枚举可用摄像头 |
| POST | `/api/cameras/refresh` | 重新枚举设备 |
| POST | `/api/camera/select` | 选择摄像头，body: `{"name": "..."}` |
| POST | `/api/start` | 开始推流 |
| POST | `/api/stop` | 停止推流 |
| GET | `/api/frame` | 获取当前帧（JPEG 二进制） |
| GET | `/api/snapshot` | 抓拍（JPEG 二进制） |
| GET | `/api/timing` | 采集 / 编码耗时统计 |
| GET | `/api/latency` | 服务端内部延迟 |
| GET | `/api/mqtt-dbg` | Broker 调试信息 |

`/api/status` 响应示例：

```json
{
  "alive": true,
  "streaming": true,
  "deviceName": "Integrated Camera",
  "resolution": "852x480",
  "streamResolution": "852x480",
  "fps": 30,
  "preset": "high",
  "frameSeq": 1864,
  "hasFrame": true,
  "serverTimeMs": 1795530550584,
  "wsClients": 0,
  "error": ""
}
```

> **`alive` 与 `streaming` 的语义区别**
>
> - `streaming` = **意图** —— 「我打算推流」
> - `alive` = **实际** —— 「最近 3 秒内确实产出了新帧」
>
> 摄像头被其他程序（会议软件）独占时，`streaming=true` 但 `alive=false`。
> 客户端用 `alive` 判断画面是否真的在动；若两者相等，客户端会在黑屏时仍认为画面正常。

### 画质档位

档位定义在服务端配置里，客户端通过 `/api/quality` 拉取，两端共用同一份定义：

| 档位 key | 高度 | JPEG 质量 | 目标帧率 |
|---|---|---|---|
| `ultrafast` | 180 | 40 | 15 fps |
| `low` | 270 | 45 | 20 fps |
| `balanced` | 360 | 55 | 25 fps |
| `high` | 480 | 65 | 30 fps |
| `ultra` | 720 | 72 | 30 fps |

缩放遵循**不放大、不变形**：按原始宽高比换算目标宽度，并收敛到配置的上下限内。
摄像头原生 720p 时选 `ultra` 不会被拉伸变形。

---

## 六、关键实现

这一节记录实现过程中真正花时间的几个点。

### 6.1 跨线程写 Qt Socket 被静默丢弃

**现象**：手机端显示「已连接」，连接数正常增长，但**永远黑屏**。

**原因**：`StreamPumpThread` 里直接调用 `socket->write()`，而 `QTcpSocket` 是在主线程创建的。
`QTcpSocket` **不是线程安全的**，跨线程调用 `write()` 时 Qt 会**静默丢弃**数据——
不报错、不抛异常。所以：

- `CONNACK` / `SUBACK` 是主线程发的 → 正常
- `PUBLISH` 视频帧是 StreamPump 线程发的 → **全部丢失**

**修复**：在 `writeRaw()` 里检查线程归属，跨线程时投递回 socket 所属线程。

```cpp
void MqttSession::writeRaw(const QByteArray &data)
{
    if (!m_alive.load() || !m_socket)
        return;

    // 跨线程写保护：QTcpSocket 必须在它自己的线程里操作，
    // 跨线程 write() 会被 Qt 静默丢弃（不报错！）
    if (QThread::currentThread() != m_socket->thread()) {
        QMetaObject::invokeMethod(this, [this, data]() {
            writeRaw(data);
        }, Qt::QueuedConnection);
        return;
    }

    const qint64 written = m_socket->write(data);
    if (written < 0) {
        m_lastError = m_socket->errorString();
        m_alive.store(false);
    }
}
```

发送队列用 `QMutex` 保护，队列深度限制为 **2**：
网络跟不上时**丢弃最旧的帧**而不是堆积——实时传输里堆积的旧帧毫无价值，只会累积延迟。

### 6.2 会话泄漏导致连接数只增不减

**现象**：手机端看门狗超时后重连，电脑端「已连接手机」从 1 涨到 2、3、4… 不再回落。

**原因**：握手失败或 EOF 时 `close()` 提前 return（因为 `!m_alive`），socket 没有真正 disconnect，
死连接一直留在 `m_sessions` 里。

**修复**：
- `close()` **无条件** disconnect + close socket
- 增加 **2 秒周期**的会话清理（`reapDeadSessions()`）
- `clientCount()` **只统计 `isAlive() && !topics().isEmpty()`** 的会话（已订阅才算数）

### 6.3 时钟偏差导致延迟永远显示 0ms

**现象**：手机端延迟恒为 `0ms`。

**原因**：PC 和手机的系统时间不同步。朴素算法：

```
延迟 = Date.now() - 采集时刻
```

当 PC 时钟比手机快时，这个差值是**负数**，再被 `Math.max(0, ...)` 钳制成 0。
所以显示的 0ms 是**假象**，不是真实延迟。

**修复**：**最小值滤波估计器**。

设 `raw = 收到时刻 - 采集时刻`，则 `raw = d - skew`，
其中 `d ≥ 0` 是网络传输延迟，`skew = T_PC - T_phone` 是时钟偏差。

关键洞察：**局域网内的传输延迟下限就是 0**（同一时刻收发），
所以所有 `raw` 的**历史最小值** `rawMin ≈ d_min - skew ≈ -skew`。

于是：

```
skew        = -rawMin
采集时刻(手机时钟) = ts + rawMin
真实延迟     = Date.now() - (ts + rawMin)
```

用滑动窗口维护 `rawMin` 即可。**实测在 ±30000ms 的时钟偏差下，校准误差 < 6.4ms**。

> 踩坑：最初把 `_rawMin` 初始化为 `Number.MAX_SAFE_INTEGER`，
> 首次采样时 `skew += (MAX_SAFE_INTEGER - raw)` 直接算出 9×10¹⁵ 的天文数字。
> 正确做法是初始化为 `null`，首次采样直接赋值。

### 6.4 服务发现：用出口 IP 推断网段是错的

**现象**：在公司 Wi-Fi 下扫描永远找不到服务端。

**原因**：原实现请求 `api.ipify.org` 拿公网出口 IP，取前两段当作手机所在网段。
这只在**单层 NAT**（家用路由器）时成立。

实测（公司 Wi-Fi `10.14.33.86/23`，出口 IP `123.153.97.130`）：

- 候选列表里只有 `123.153.x.x`，和手机真实网段 `10.14.33.x` 完全无关
- 更糟的是拼出来的 `123.153.1` 只有 3 段，**不是合法 IPv4**
- 4318 个候选里 **2032 个（47%）是废串**，全部会走到超时

**修复**：直接读手机自己的网卡信息。

- 用 `plus.android` 调 Android 原生 `java.net.NetworkInterface.getNetworkInterfaces()`
  拿到**真实网卡 IP**，100% 准确
- 再用 `NetworkInterface.getInterfaceAddresses()` 拿到**真实子网掩码**
- 按掩码算出完整网段范围（`/23` 这种跨字节网段也要完整枚举，只按 `/24` 扫会漏掉一半地址）
- 出口 IP 推断降级为**最后的兜底**

另外修掉：全局广播 + 网段定向广播**同时发**（很多路由器转发全局广播但不转发定向广播）；
刚发现 IP 的探测超时用 3 秒而不是 700ms（刚换网时 TCP 要重新建连 + ARP 解析）。

### 6.5 base64 懒编码

客户端每收到一帧立刻 base64 编码，30fps 下每秒做 30 次十几 KB 的字符串编码，
而且**大部分帧还没渲染就被新帧覆盖了**——那些 base64 白算了。

改成 **getter 懒编码**，真正要渲染这一帧时才编码：

```javascript
return {
  get data() {
    if (this._b64 === undefined) {
      this._b64 = encodeBase64(jpeg)
    }
    return this._b64
  },
  jpeg: jpeg,
  ts: ts,
  seq: seq
}
```

调用方 `msg.data` 完全不用改。

### 6.6 接收缓冲区用 Uint8Array 手动搬运

uni-app App 端（weex 运行时）里，对 11KB 的数组做 `concat` / `slice` 非常慢，
30fps 下会直接堵死消息队列。

改为预分配 `Uint8Array` + `copyWithin` 手工搬移，容量按 2 倍递增。

### 6.7 高 DPI 窗口尺寸跳变

**现象**：启动时窗口是 1180×760（物理像素），之后突然变成 1792×1196。

**原因**：进程启动时没有 DPI 感知（Windows 按 96 DPI 虚拟化），
后续创建第二个顶层窗口时 Windows 切换 DPI 上下文，把所有窗口重新缩放一次。

**修复**：`resources/app.manifest` 声明 `PerMonitorV2`，并嵌入 exe：

```xml
<dpiAwareness>PerMonitorV2, PerMonitor</dpiAwareness>
<dpiAware>true</dpiAware>
```

另外 `activateWindow()` 在 Windows 上会把 owner 从「最大化」拽回「还原」，
所以日志浮窗只用 `show() + raise()`，不抢焦点。

---

## 七、界面预览

> 📸 以下位置放置实际截图

### 服务端主界面

<!-- 建议：截图放 docs/ 目录，然后改成 <img src="docs/desktop.png" width="800"> -->

### 日志浮窗

> 独立顶层窗口，贴主界面右侧，高度对齐，移动主界面时自动跟随。

<!-- <img src="docs/desktop-log.png" width="800"> -->

### 客户端实时监看界面

<!-- <img src="docs/mobile.png" width="300"> -->

---

## 八、快速上手

### 环境要求

| 依赖 | 版本 |
|---|---|
| 服务端 | Windows 10 / 11（64 位） |
| 服务端依赖 | 无（绿色免安装） |
| 客户端 | Android / iOS |
| 网络 | 两者在同一局域网 |

### 启动服务端

解压 `MJ_CAMERA_PC.zip`，双击 `MJ_CAMERA_PC.exe`。

首次运行会在 exe 同目录生成 `config.json`。程序自动检测本机 IP 并显示在界面上。

### 连接客户端

打开 App，客户端会自动搜索服务端（UDP 广播 → 网段扫描）。
若自动发现失败（部分路由器禁止 UDP 广播），手动填写服务端界面显示的 IP，App 会记住。

### 验证服务端状态

```powershell
curl http://127.0.0.1:8080/api/status
```

```json
{"alive":true,"streaming":true,"fps":30,"frameSeq":1864,"resolution":"852x480", ...}
```

---

## 九、性能实测

测试环境：Intel 集成摄像头，Windows 11，1920×1080 采集，局域网 Wi-Fi。

| 指标 | 实测值 |
|---|---|
| 推流帧率 | **30 fps**（稳定无抖动） |
| 单帧大小 | **32.7 KB**（852×480, quality 65） |
| 端到端延迟 | **200 ms**（含网络传输 + 解码 + 渲染） |
| 服务端内部延迟 | **33 ms**（帧龄） |
| 时钟校准误差 | **< 6.4 ms**（±30000ms 偏差下测试） |
| 内存占用 | **~60 MB** |
| 分发包体积 | **~31 MB**（19 个文件） |
| 单 exe 体积 | **0.4 MB** |
| 会话泄漏 | **0**（反复连断 5 次，session 数稳定归零） |

### 协议验证脚本

项目附带 Node.js 脚本，可直接用来压测：

```bash
# 连接 MQTT over WebSocket，验证握手、订阅、帧格式与帧率
node tools/mqtt_probe.js

# 模拟时钟偏差，验证延迟校准算法
node tools/test_skew.js
```

`mqtt_probe.js` 输出示例：

```
[probe] handshake OK
[probe] CONNECT sent
[probe] CONNACK -> subscribing
[probe] SUBACK -> subscribed, waiting for frames...
[probe] FIRST FRAME seq=416 ts=1795542550243 jpeg=14170B

========== RESULT ==========
frames received : 150
elapsed        : 10.0 s
fps            : 15.0
jpeg size      : 14170 bytes
frame age      : 17 ms
subscribed     : true
RESULT: PASS
```

---

## 十、常见问题

<details>
<summary><b>Q: 延迟显示 0ms，是真的 0 延迟吗？</b></summary>

不是，是**时钟偏差**造成的假象——PC 时钟比手机快时原始延迟算出来是负数，被钳制成了 0。

已通过最小值滤波估计器修复，实测 ±30000ms 偏差下校准误差 < 6.4ms。详见 [6.3](#63-时钟偏差导致延迟永远显示-0ms)。
</details>

<details>
<summary><b>Q: 手机端连不上 / 一直「正在查找服务器」</b></summary>

1. 确认手机与 PC 在**同一网段**（同一路由器）
2. 检查 Windows 防火墙，允许「专用网络」访问 8080 / 1883 / 8081
3. 手动填写 PC 界面显示的 IP
4. 部分公司/学校 Wi-Fi 开启了 **AP 隔离**，设备间不互通，需换热点
</details>

<details>
<summary><b>Q: 换 Wi-Fi 后连不上（IP 变了）</b></summary>

服务端会自动监听网卡变化并更新显示的 IP。客户端点「刷新状态」触发重新发现。
路由器禁止 UDP 广播时会退化为网段扫描，可能需要十几秒；此时手动填一次 IP 更快。
</details>

<details>
<summary><b>Q: 摄像头列表是空的 / 提示 no camera found</b></summary>

1. 打开 Windows「设备管理器 → 摄像头」确认设备存在且无黄色感叹号
2. 安装摄像头厂商驱动（部分 USB 免驱摄像头仍需额外驱动）
3. 点界面「刷新」重新扫描
4. 检查 `config.json` 的 `deviceBlacklist` 是否误屏蔽
5. 设备可能被会议软件独占，先关掉那些程序
</details>

<details>
<summary><b>Q: 能接网络摄像机（海康/大华）吗？</b></summary>

**不能直接接。** 本项目走 Windows 标准视频设备接口（Media Foundation）。
网络摄像机需先转接：DVR/NVR 的 HDMI 或 USB 输出 → HDMI 采集卡 → 电脑；
或用工具把 RTSP 流转成 Windows 虚拟摄像头。转接后在设备管理器能看到即可。
</details>

<details>
<summary><b>Q: 画面卡住 / 客户端黑屏但显示已连接</b></summary>

1. 看 `/api/status` 的 `alive` 字段，`false` 说明摄像头被其他程序独占
2. 看服务端日志浮窗的 `[MQTT]` 输出
3. 访问 `/api/mqtt-dbg` 查看各连接的 `sentFrames` / `droppedFrames`
</details>

<details>
<summary><b>Q: 点「停止推流」后摄像头灯还亮着？</b></summary>

**有意设计**。「停止推流」只停止向客户端推帧，本地采集和预览继续运行，
方便确认摄像头仍在工作。需要完全关闭请退出程序。
</details>

<details>
<summary><b>Q: 能在 Linux / macOS 上跑吗？</b></summary>

当前针对 Windows（依赖 Media Foundation + WIC）。
Qt 代码本身可移植，但采集层需替换为 V4L2 / AVFoundation。
</details>

---

## 十一、项目结构

```
MJ_CAMERA_001/
├── src/                              # uni-app 客户端
│   ├── pages/index/index.vue         # 主页面
│   ├── components/multi-player/      # 播放器组件（Canvas 渲染 + 时钟校准）
│   └── utils/
│       ├── mqtt-client.js            # 手写 MQTT 3.1.1 客户端
│       ├── discovery.js              # 三级服务发现
│       ├── canvas-renderer.js        # Canvas 帧渲染
│       ├── clock-sync.js             # 时钟偏差校准
│       └── base64.js
│
└── MJ_CAMERA_PC/                     # Qt 6 / C++ 服务端
    ├── main.cpp
    ├── src/
    │   ├── capture/
    │   │   ├── camera_device.cpp     # MF 设备枚举
    │   │   ├── camera_capture.cpp    # MF 采集线程
    │   │   └── jpeg_encoder.cpp      # WIC JPEG 编码 + 缩放
    │   ├── core/
    │   │   ├── config.cpp            # 配置加载（兼容两种 JSON 格式）
    │   │   └── utils.cpp             # IP 检测、缩放计算
    │   ├── server/
    │   │   ├── http_server.cpp       # QtHttpServer 封装
    │   │   └── api_handlers.cpp      # REST 路由
    │   ├── stream/
    │   │   ├── frame_hub.cpp         # 最新帧槽
    │   │   ├── stream_pump.cpp       # 组帧 + 推送线程
    │   │   ├── mqtt_broker.cpp       # 会话管理 + 主题分发
    │   │   ├── mqtt_session.cpp      # 单连接会话（MQTT 报文解析）
    │   │   └── ws_server.cpp         # 原生 WebSocket 通道
    │   └── ui/                       # StatCard、样式表
    ├── resources/
    │   ├── app.manifest              # PerMonitorV2 DPI 声明
    │   └── icons/                    # 图标源文件
    ├── build.bat                     # 一键编译 + 打包
    └── tools/                        # 协议验证脚本
```

---

## 十二、从源码编译

### 环境要求

| 依赖 | 版本 |
|---|---|
| Qt | 6.11（需 `Network`、`Widgets`、`Svg`、`HttpServer` 模块） |
| CMake | 3.21+ |
| MSVC | 2022 (v143) 或更新 |
| 编译器 | C++17 |

### 编译命令

```bat
:: Release 编译
build.bat

:: 清理后重新编译
build.bat clean

:: 编译 + 打包成可分发目录（含 windeployqt）
build.bat deploy
```

产物位于 `dist/`。

### 配置文件

服务端在 exe 同目录生成 `config.json`，支持两种格式：

```jsonc
// A) 嵌套格式
{
  "server":  { "port": 8080, "mqttPort": 1883, "autostart": false },
  "camera":  { "requestWidth": 1280, "requestHeight": 720,
               "preferredNames": ["USB Camera"] },
  "quality": { "preset": "high" },
  "stream":  { "maxWidth": 1280, "minWidth": 320,
               "maxHeight": 720, "minHeight": 180 }
}

// B) 扁平格式
{
  "httpPort": 8080,
  "mqttPort": 1883,
  "preset": "high",
  "forcedDeviceName": "USB2.0 HD Webcam",
  "deviceBlacklist": ["OBS Virtual Camera"],
  "allowLanAccess": true
}
```

### build.bat 的两个坑

1. **文件必须是纯 ASCII**
   `cmd.exe` 用 OEM 代码页（中文系统是 936）读取 `.bat`，
   UTF-8 中文注释会被解析成乱码命令。所以注释一律用英文。

2. **不要用 `"%OUT_DIR%\"` 这种写法**
   尾部反斜杠会把闭合引号转义掉，`cmd` 看到畸形命令而**静默失败**——
   曾因此让 `dist/` 里的 exe 一直是旧版本，而构建却报告成功。

---

## 十三、为什么不用 WebRTC

| 对比 | WebRTC | 本方案 |
|---|---|---|
| 延迟 | 50~100ms | 200ms |
| 部署依赖 | 信令服务器 + STUN/TURN | **无** |
| 客户端体积 | 完整 WebRTC 栈（移动端数 MB） | **手写 MQTT 客户端，约 10KB** |
| 实现复杂度 | ICE 候选收集、NAT 穿透、编解码协商、JSEP | **MQTT 5 种控制报文** |
| 硬件编码要求 | 高 | **CPU 软编码即可** |
| 协议通用性 | 私有协商流程 | **标准协议，任意客户端可接入** |

局域网场景下 WebRTC 的核心优势（NAT 穿透、拥塞控制、抖动缓冲）都用不上，
反而要背信令服务器和 STUN/TURN 的部署成本。

选择 MQTT 的另一个理由：**协议通用**。本文档给出了完整帧格式，
任何语言都能按规范实现客户端——这正是做视频传输开发时更需要的。

---

## 十四、扩展方向

当前实现留出了这些扩展点：

- **多路摄像头** —— MQTT 的主题机制天然支持，用 `mj/camera/{id}/frame` 区分即可，
  Broker 侧只需在 `publish()` 时按主题过滤
- **音频传输** —— 复用 `MJF1` 帧格式，新增 `MJF2` magic 标记音频帧
- **录像 / 回放** —— `/api/snapshot` 已在提供单帧抓取，可扩展为服务端写文件
- **H.264 编码** —— 把 WIC 换成 `MFVideoEncoder`（系统自带 H.264 编码器），
  帧格式改为 Annex-B，复用同一套 MQTT 通道
- **跨平台服务端** —— 采集层替换为 V4L2 / AVFoundation，传输层完全可复用
- **鉴权** —— MQTT 3.1.1 的 CONNECT 报文支持 Username/Password 字段，
  服务端 `handleHandshake()` 里校验即可

---

## 许可协议

MIT License

---

## 相关链接

- **Qt 6** — https://www.qt.io/
- **Media Foundation** — https://learn.microsoft.com/windows/win32/medfound/
- **MQTT 3.1.1 规范** — https://mqtt.org/mqtt-specification/
- **RFC 6455 (WebSocket)** — https://datatracker.ietf.org/doc/html/rfc6455
- **OASIS MQTT Version 3.1.1** — https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html

---

<div align="center">

**MJ Camera** —— 局域网 MQTT 视频传输方案

如果需要获取该软件及详细内容，欢迎私信留言，也欢迎基于本文档的协议说明实现自己的客户端。

</div>
