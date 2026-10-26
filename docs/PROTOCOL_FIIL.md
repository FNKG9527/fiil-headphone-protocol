**[简体中文](PROTOCOL_FIIL.md)** | [English](PROTOCOL_FIIL.en.md)

# FIIL 耳机通信协议规格（PROTOCOL_FIIL.md）

> 本文档是**协议规格总纲**，源自一个第三方 Android 控制模块的实现过程，所有条目均来自对 **FIIL+ 3.5.2（`com.fengeek.f002`，versionCode 175，minSdk 26，compileSdk 33）** 的反编译源码，逐条标注文件与行号。
>
> 配套的完整逆向报告见 `docs/REVERSE_ENGINEERING_REPORT.md`；本文档只保留「实现控制端必需」的部分，并补充第二轮新发现的 **Mate 协议族（第三套命令空间）**。
>
> **关于文中的「模块」**：指作者为验证本协议而编写的一个第三方 Android 控制端（实验性，未随本仓库分发）。凡是写「模块实现 / 模块策略」的地方，指的是**该控制端的取舍**，不是协议本身的要求——你可以按自己的架构重新决定。

---

## 0. 结论速览

| 事实 | 结论 |
|---|---|
| 传输层 | **双通道**：经典蓝牙 SPP（RFCOMM）+ BLE GATT（`ff17` 写 / `ff18` 收）。**两者帧格式完全相同**，只有搬运方式不同 |
| 为什么需要 BLE | FIIL Key Max 实测：标准 SPP 连上即**静默**（零回包），自定义 UUID 连上被**反复踢**（`scn:6 opened=21 leaked=21`）。BLE GATT 稳定可用 |
| SPP UUID | 老机型 `00001101-0000-1000-8000-00805F9B34FB`；新机型（含 Mate）`d52b47bc-2dea-42c4-b26d-20252fceeeb3` |
| BLE UUID | 写 `0000ff17-0000-1000-8000-00805F9B34FB`（Write No Response）；收 `0000ff18-…`。**不是 ff16** |
| 帧格式 | **两套并存**：GAIA（`0xFF` 打头）与 Mate（`0x00` 打头）。同一通道上按首字节分流 |
| 命令空间 | 三套：GAIA 16 位命令字（`0x02xx`/`0x07xx`）、Mate 的 classifyId + cmdId、豆包 `0x55AA`（少数机型） |
| **GET 与 SET 语义不同** | GET：`cid=39` 固定 + cmdId 填查询码；SET：`cid=`命令族 + **cmdId 恒为 0**。详见 §3.12 |
| 连接方向 | 手机是 **RFCOMM 客户端**（`createRfcommSocketToServiceRecord`）／**GATT 客户端**（`connectGatt`） |
| 握手 | 连接成功后**第一帧必须是 `setUnlock`**（`00 27 01 00 04 FF 02 02 34`），否则耳机对所有查询静默 |
| 应答判定 | `type` 字节（偏移 2）：`0x01`=REQ / `0x02`=RESP / `0x03`=NOTIFY。**旧实现误把它当版本号并写死 `!=0x01 丢弃`，导致所有应答被扔掉** |

---

## 1. 传输层与 UUID

### 1.1 连接方式

FIIL 耳机有**两条完全独立的传输层**，帧编码完全相同，只有搬运方式不同：

| 传输层 | 媒介 | 适用 | 模块实现 |
|---|---|---|---|
| **RFCOMM / SPP** | 经典蓝牙串口 | 全系（APP 只用这一条） | `FiilRfcommController.openSocket()` |
| **BLE GATT** | BLE 特征读写 | Mate 机型（实测唯一稳定） | `FiilGatt` |

**SPP 路径**：APP 用 `BluetoothDevice.createRfcommSocketToServiceRecord(UUID)` 主动连耳机（客户端），拿 `BluetoothSocket` 的 `InputStream`/`OutputStream` 裸读写。

模块侧三级兜底（**关键：`createRfcommSocketToServiceRecord()` 只创建 socket 对象、不做 SDP 也不报错**，真正建链发生在 `connect()`）：

| 顺序 | 方式 | 适用 |
|---|---|---|
| 1 | `createRfcommSocketToServiceRecord(UUID_SPP)` | 老机型 |
| 2 | `createRfcommSocketToServiceRecord(UUID_CC_PRO2)` | F045~F053 等新机型 |
| 3 | 反射 `createRfcommSocket(1)`（channel 1） | 前两者都失败时 |

> **实证教训**：旧实现「socket 对象创建成功就 return」，导致标准 SPP 永远先返回、`connect()` 对 Mate 机型必然失败，自定义 UUID 那条路根本执行不到。正确做法是**三个候选全部先建好，再逐个 `connect()`**，成功即用、失败关掉换下一个。见 `FiilRfcommController.openSocket()`。

**为什么还需要 BLE GATT**（2026-10-25 PC 端 Bleak 直连 FIIL Key Max `B0:F1:A3:00:00:01` 实测）：

1. 标准 SPP（`00001101-…`）**能 `connect()` 成功但耳机完全静默** —— 所有查询零回包；
2. 自定义 UUID（`d52b47bc-…`）能连上也有回包，但 Android 侧 RFCOMM 会被耳机反复踢掉（`dumpsys bluetooth_manager` 里 `scn:6 opened=21 closed=0 leaked=21`）；
3. BLE GATT（ff17 写 / ff18 收）**稳定、且 Mate 帧格式与 SPP 逐字节相同**。

所以模块的策略是：先走 SPP → 三条 RFCOMM 全失败或连上后 3.5 秒零回包 → 自动改走 GATT。

### 1.2 UUID 全表

| 常量 | 值 | 说明 |
|---|---|---|
| `UUID_SPP` | `00001101-0000-1000-8000-00805F9B34FB` | 标准串口服务，老机型 |
| `UUID_CC_PRO2` | `d52b47bc-2dea-42c4-b26d-20252fceeeb3` | FIIL 自定义 SPP，F045~F053 全部使用（`F045~F053MainFragment` 的 `createRfcommSocketToServiceRecord` 共 16 处调用点均为它） |
| `FiilGatt.CHAR_WRITE` | `0000ff17-0000-1000-8000-00805F9B34FB` | **BLE GATT 写入特征**（App → 耳机），用 Write Without Response |
| `FiilGatt.CHAR_NOTIFY` | `0000ff18-0000-1000-8000-00805F9B34FB` | **BLE GATT 通知特征**（耳机 → App） |
| F042 BLE Service | `13fa60a2-429a-4a90-a3e8-d4c2b9a17655` | 仅 F042 系列（Airoha 芯片）走 BLE 私有通道 |
| F042 Write/Notify | `3d4dac58-7d68-4734-8a71-21…` | 同一特征既写又收 |

> **注意 ff16 / ff17 / ff18 三兄弟**：实测**写入是 ff17**，不是 ff16。曾按常见北欧方案假设成 ff16，导致写入全部返回 false、耳机无反应。
>
> **未确认项**：F042 BLE 特征的完整 128 位值在反编译产物中被截断，模块当前不实现该链路（F042 走 `BLE_ONLY` 分支，暂不接管）。

### 1.3 BLE GATT 通道细节（实测）

| 项 | 值 |
|---|---|
| 写入特征 | `0000ff17-…`（`PROPERTY_WRITE_NO_RESPONSE`，必须用它；Write With Response 常回 `GATT_BUSY`） |
| 通知特征 | `0000ff18-…` |
| CCCD | `00002902-…`；实测**不写也能收到通知**（`setCharacteristicNotification(true)` 已足够），但模块仍尝试写以兼容其他 ROM |
| 帧格式 | 与 SPP 完全相同（见 §3.2），无需任何封装 |
| 握手 | 必须先发 `setUnlock`（cid=39 / cmd=0xFF / payload `02 34`），否则耳机静默 |
| 写间隔 | 25 ms（连续写太快会丢帧） |
| 通知分包 | 一帧可能被拆成多包，必须按 §3.2 的长度字段重组 —— 见 `FiilGatt.handleRx()` |

---

## 2. GAIA 帧格式（第一套命令空间）

### 2.1 帧结构

来源：`com\fiil\sdk\command\FiilCommandUtil.java:517 getSendBuffer(int vendor, int cmd, boolean newProto, byte... payload)`
配合 `com\fiil\sdk\gaia\father\Gaia.java:344-359` 的偏移常量（`OFFS_SOF=0`、`OFFS_VERSION=1`、`OFFS_FLAGS=2`、`OFFS_PAYLOAD_LENGTH=3`、`OFFS_VENDOR_ID_H=4`、`OFFS_VENDOR_ID_L=5`、`OFFS_COMMAND_ID_H=6`、`OFFS_COMMAND_ID_L=7`、`OFFS_PAYLOAD=8`、`SOF=-1`、`VENDOR_FF=564=0x0234`、`PROTOCOL_VERSION=1`、`NEW_PROTOCOL_VERSION=241=0xF1`）。

```
偏移  长度  字段          值 / 说明
 0     1   SOF           0xFF
 1     1   version       0x01（老协议） / 0xF1（新协议）
 2     1   flags         0x00
 3     1   payload_len   = payload.length（**只算 payload，不含 vendor 与 command**）
 4     2   vendor_id     0x0234（大端，FIIL 厂商号）
 6     2   command       (cmd & 0x7FFF)，高位 0x8000 = ACK 帧
 8     n   payload       命令负载
```

> 关键点：`payload_len` 只等于**负载长度**，`getSendBuffer` 里 `bArr2[OFFS_PAYLOAD_LENGTH] = bArr.length`，而帧总长 = `8 + payload.length`。这一点极易写错（曾误记为「含 vendor + cmd」），以源码为准。

### 2.2 常用帧示例

| 操作 | 完整字节 | 说明 |
|---|---|---|
| 查版本 | `FF 01 00 00 02 34 03 01` | cmd = 769 = `0x0301`，空负载 |
| 查机型 | `FF 01 00 00 02 34 03 09` | cmd = 777 = `0x0309` |
| 查电量 | `FF 01 00 00 02 34 03 03` | cmd = 771 = `0x0303` |
| 查 ANC | `FF 01 00 00 02 34 02 81` | cmd = 641 = `0x0281` |
| 设 ANC | `FF 01 00 01 02 34 02 01 [mode]` | cmd = 513 = `0x0201` |
| 设 EQ（老） | `FF 01 00 01 02 34 02 03 [idx]` | cmd = 515 = `0x0203` |
| 设 EQ（V2） | `FF F1 00 0C 02 34 07 0C [0][类型][10 段]` | cmd = 1804 = `0x070C`，12 字节负载 |
| 设 3D | `FF 01 00 01 02 34 02 04 [0/1]` | cmd = 516 = `0x0204` |
| 低延迟 | `FF 01 00 01 02 34 07 22 [档位]` | cmd = 1826 = `0x0722`，档位 0=音乐 / 1=游戏 / 2=视频（见 §3.7） |
| 多点（畅连） | `FF 01 00 01 02 34 01 02 [0/1]` | cmd = 258 = `0x0102` |
| 寻找耳机 | `FF 01 00 01 02 34 01 09 01` | cmd = 265 = `0x0109` |

> 表中 `payload_len` 才是第 4 字节；请与 §2.1 的公式核对（例：设 ANC 的 payload 是 `[mode]` 1 字节，故第 4 字节 = `01`）。

### 2.3 回包解析

来源：`com\fiil\sdk\gaia\GaiaSdkCommand.java` 各 `analysisXxx` 方法。

- 绝大多数命令：**`payload[1]` 是有效值**（`payload[0]` 是状态/保留位）。
- **电量是例外**：`payload[2]` 是一个**十进制数字字符串**（如 `"85"`），且 APP 会 `+1` 后显示。见 `FiilGaia.parseBatteryLevel()`。
- 充电盒电量走另一条命令（`GET_BATTERY_OF_BOX = 1915`），`payload[1]` 直接是百分比，**不加 1**。
- 佩戴状态：左耳 `payload[2]`、右耳 `payload[1]`。

### 2.4 GAIA 命令值全表

完整 190 条见 `docs/reference/_cmd_table.txt`；模块只用 `FiilCommands.kt` 里的约 60 条。关键值：

```
GET_VERSION 769(0x0301)  GET_ADDRESS 770  GET_BATTERY_LEVEL 771(0x0303)
GET_EARTYPE 777(0x0309)  GET_ANC 641(0x0281)  GET_MAF 650(0x028A)
GET_EQ 643(0x0283)       GET_3D 644(0x0284)   GET_WEAR 391(0x0187)
GET_WEARING_STATE 1912   GET_LOW_LATENCY 1825(0x0721)  GET_MULTIPOINT 386(0x0182)
GET_BATTERY_OF_BOX 1915  GET_BATTERY_LEVEL_V2 1800(0x0708)  GET_EQ_V2 1803(0x070B)
SET_ANC 513(0x0201)      SET_EQ 515(0x0203)   SET_3D 516(0x0204)
SET_MAF 522(0x020A)      SET_WEAR 263(0x0107) SET_MULTIPOINT 258(0x0102)
SET_VOLUME 262           UP_VOLUME 517        DOWN_VOLUME 518
SET_SEARCH 265(0x0109)   SET_RESET 521        RESTORE_FACTORY 16398
SET_LOW_LATENCY 1826(0x0722)  SET_EQ_V2 1804(0x070C)  SET_MAF_V2 1824(0x0720)
PLAY 8193(0x2001)  PAUSE 8194(0x2002)  NEXT 8195(0x2003)  PREVIOUS 8196(0x2004)
```

---

## 3. Mate 帧格式（第三套命令空间，F045~F053 专用）

### 3.1 为什么会有第三套

F045 之后的机型（GS Lite / Key Max / Atom / GS Links / Key Pro2 / CC3 / CC Pro3）走的是 `com.fengeek.main.i` 包下的一套**全新命令层**：命令不再是 16 位的 `0x02xx`，而是「classifyId（分类）+ commandId（子命令）」两个 8 位值。

### 3.2 帧结构

来源：`com\fengeek\main\i\b.java:237 writeMate(FiilBaseCommand, handle)`

```
无子命令（commandId == 0）：
  [0]=0x00  [1]=classifyId  [2]=0x01  [3]=0x00  [4]=payloadLen  [5..]=payload
  整帧长 = 5 + payloadLen

有子命令（commandId != 0）：
  [0]=0x00  [1]=classifyId  [2]=0x01  [3]=0x00
  [4]=payloadLen+2  [5]=commandId  [6]=payloadLen  [7..]=payload
  整帧长 = 7 + payloadLen
```

- **没有 SOF 之外的校验**，没有 vendor，没有 ACK 位。
- 直接 `write(byte[])` 裸写 SPP，**不套 GAIA 外壳**。
- 与 GAIA 帧靠首字节区分：`0xFF` = GAIA，`0x00` = Mate。

模块实现：`FiilMate.buildFrame()` / `FiilMate.parse()` / `FiilMate.drain()`。

### 3.3 classifyId 表

来源：`com\fengeek\main\i\d\c.java`

| classifyId | 名称 | cmdId | payload | 行号 |
|---|---|---|---|---|
| 1 | `SET_MULTIPOINT` 畅连开关 | **1** | `[0/1]` | c.java:321 |
| 32 | `SET_EQ` EQ 写入 | 无 | `[保留][类型][N 段]` | c.java:338 |
| 33 | `SET_MUSIC` 媒体控制 | 见 §3.6 | 无 | c.java:420 |
| 37 | `SET_LOW_LATENCY` 低延迟 | 无 | `[0/1]` | c.java:388 |
| 44 | `SET_MAF` 降噪模式 | 无 | `[mode]` | c.java:404 |
| 50 | `SET_SPATIAL_AUDIO` 影院模式 | 无 | `[0/1]` | c.java:504 |
| 38 | `SET_WEAR` 佩戴检测开关 | **无** | `[0/1]` | g.java:663 / h.java:729 |
| 39 | `QUERY` 查询组 | 见 §3.4 | 无 | c.java:75~267 |
| **40** | **`NOTIFY` 耳机主动上报（只收不发）** | 见 §3.4.2 | 耳机推送 | 各机型 `*MainFragment` notify 分支 |

> **易错点 1**：`SET_MULTIPOINT` 是**唯一**同时带 `commandId` 的设置类命令（`setCommandClassifyId(1)` + `setCommandId(1)`），其余设置类都只有 `classifyId`。曾误判为「无 cmdId」。
>
> **易错点 2**：Mate 的 `commandId` 在源码里是 **signed byte**，超过 127 的值写成负数（`-128`→128、`-118`→138、`-117`→139、`-114`→142、`-113`→143、`-112`→144、`-126`→130）。组帧时务必 `and 0xFF`。
>
> **易错点 3（最易漏，直接决定状态同步是否可用）**：回包分两条通道。`cid=39` 是**查询应答**，`cid=40` 是**耳机主动上报**。全库 `setCommandClassifyId(...)` 的实参只有 1/2/3/32/33/34/36/37/38/39/41/44，**从不出现 40**；而 7 个机型 `*MainFragment` 的 notify 分支**全部判 `getCommandClassifyId() == 40`**（F045:306/310/314、F046:299/303/308/317、F047:303/307/314/323/333、F048:361/365/372/381、F049:303/307/314/323/333/343、F051:300/304/307/316/323/333/343、F053 同 F051）。**只处理 cid=39 会导致：用户在耳机上按键改 ANC、摘下耳机、电量变化全部收不到，模块状态与耳机脱节。** 模块实现见 `FiilMate.Cid.NOTIFY` 与 `FiilRfcommController.applyMateNotify()`。

### 3.4 QUERY(39) 的子命令

| cmdId | 名称 | 回包 |
|---|---|---|
| 1 | 电量 | `payload[1]` = 百分比（0/255 视为 0） |
| 2 | 版本 | 字符串 |
| 4 | EQ 状态 | 12 字节：`[0]`保留 `[1]`类型 `[2..11]`10 段增益 |
| 5 | 按键配置 | — |
| 6 / 138 | 右 / 左耳 UUID | — |
| 7 | 播放状态 | `payload[0]==1` = 播放中 |
| 8 | 低延迟状态 | `payload[0]==1` = 开 |
| 12 | MAF 状态 | `payload[0]` ∈ 0..4 |
| 24 | 影院模式状态 | `payload[0]==1` = 开 |
| 128 | 多点状态 | `payload[0]==1` = 开 |
| 130 | 音效 | — |
| 142 / 143 / 144 | LDAC 解锁 / 开关 / 编码 | — |

> **注意 cmdId=6 与 139 的关系**：源码里右耳 UUID 是 `NetError.ERR_BAD_SSL_CLIENT_AUTH_CERT` = **-117**，落在线上是 `0x8B` = **139**，不是 6。左耳 `-118` → **138**。这一处曾在模块里写错成 6，已更正。

### 3.4.1 各机型命令覆盖差异（实测 7 个机型类）

同一套 classifyId 并非所有机型都实现。差异对照（`-` = 该类无此方法）：

| 能力 | F045 | F046 | F047 | F048 | F049 | F051 | F053 |
|---|---|---|---|---|---|---|---|
| 电量/EQ/低延迟/多点/版本/按键/播放 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| MAF 五态（cid 44） | ✗ | ✓ | ✗ | ✗ | ✓ | ✓ | ✓ |
| 影院模式（cid 50） | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| EQ 自适应（1/14） | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| 噪声自适应（1/16） | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| 风噪自适应（1/17） | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |
| 佩戴检测（cid 38/39） | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 语言设置（cid 41） | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| LDAC 开关（1/13） | ✗ | ✓ | ✗ | ✓ | ✓ | ✓ | ✓ |
| 录音/存储（F048 独有） | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ |
| 主从耳 / 耳机 UUID | ✗ | ✓ | ✗ | ✓ | ✗ | ✗ | ✗ |
| 配对列表管理 | ✗ | ✗ | ✓ | ✗ | ✓ | ✓ | ✓ |
| 调音（getTuning/setTuning） | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |
| 助眠（setHelpSlep） | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| 豆包（setDouBao） | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ |

> 取证方法：`jadx_out/sources/com\fengeek\main\i\d\` 下 b~h.java 是 7 个机型命令类，用 `^\s*public (void|boolean|int|byte\[\]|String) (\w+)\(` 导出**真实方法声明**（不是文本 grep，避免命中注释/字符串）后逐列比对。
>
> 模块与命令类的绑定（全量 grep `i.d.X.getInstance`）：**f045→b、f046→c、f047→d(+b)、f048→e、f049→f、f051→g、f053→h**。
>
> 结论：**影院模式（`setSpatialAudio` / cid 50）实际只有 F046（Key Max）支持** —— `setSpatialAudio` 仅在 `c.java` 定义，全库唯一调用点是 `F046StyleFragment.java:560/842/897/996`。模块 `supportsCinema()` 必须按 earType 精确判定（`setOf(46)`），不能用 `isMateDialect`。
>
> **MAF 五态也不是 Mate 机型通用**：`setMAF`（cid 44）只存在于 `c/f/g/h.java`，即 **F046/F049/F051/F053**；F045（GS Lite）、F047（Atom）、F048（GS Links）虽然走 Mate 帧，但**没有降噪控制命令**（这三款本身不带 ANC）。模块 `supportsMateMaf()` 按 `setOf(46,49,51,53)` 判定，UI 对这三款机型隐藏降噪卡片，而不是发一条必然无人应答的命令。
### 3.4.2 NOTIFY(40) 子命令（耳机 → 手机）

| cmdId | 含义 | 回包解析 | 取证 |
|---|---|---|---|
| 1 | 电量 | `payload[1]` = 百分比 | F046:299 |
| 7 | 播放状态 | `payload[0]` | F046:303 → `playState()` |
| 8 | 低延迟 | `payload[0]==1` = 开 | F046:308 → `setDeferState()` |
| 9 | 佩戴状态 | `payload[0]==1` = 已佩戴（单值，不区分左右） | F051:316 / F053:316 |
| 12 | MAF 模式 | `payload[0]`，且 5→3、6→4 需归一化 | F046:317；F051:343-349 |
| 136 | 畅连设备列表 | `type=0` | F051:323 |
| 137 | 畅连连接结果 | `type=1`，`payload[0]==2` 表示超时 | F051:333 |

> **cmd 号同号不同义**：`cmdId=8` 在 QUERY 里是「读低延迟状态」的应答，在 NOTIFY 里是「低延迟状态变化推送」；`cmdId=1` 在 QUERY 里是「电量查询应答」，在 NOTIFY 里是「电量变化推送」。两条通道的 cmdId 必须**分表解析**，不能共用一个 `when`。
>
> **cmdId=12 的 5/6 归一化**：`F051MainFragment.java:344-349` 在 `MAF == 6` 时改写为 4、`== 5` 时改写为 3，说明新机型在 MAF 五态之外还有 5/6 两个附加档位。模块 `applyMateNotify()` 已同步该映射。


### 3.5 MAF 档位（Mate）

`F046StyleFragment.java:1037 setMafModeUI(int)` 的图标绑定与 `apktool_out\res\layout\fragment_f49_style.xml:36-45` 的标签顺序：

| 值 | 语义 | 选中图标 |
|---|---|---|
| 0 | 关闭 | `f046_maf_off_select` |
| 1 | 降噪 | `f046_noise_select` |
| 2 | **开放** | `f046_open_select`（`fiil_cc_pro2_open`） |
| 3 | **兼听** | `f046_listen_select`（`fiil_cc_pro2_listen`） |
| 4 | 风中 | `f046_wind_select` |

**没有第 5 档。** FIIL 把整块叫「MAF 智能调噪」（`fragment_f49_style.xml:35` 的 `tv_headset_sound2` 文案），这里的「智能」指降噪算法本身，不是额外档位。`f49/f51/f53` 布局里另有「环境自适应降噪 / 自适应抗风噪 / 耳道自适应降噪 / 自适应 EQ 调节」四个**独立开关**（cid=1，cmd=16/17/15/14，见 §3.7），与 MAF 档位并列而非其中一档。

**机型档位数量不同**：

| 机型 | 档位数 | 依据 |
|---|---|---|
| F046 Key Max / F049 Key Pro2 / F053 CC Pro3 | 5 | `setMafModeUI` 处理 0~4 |
| F051 CC3 | **3**（关 / 降噪 / 开放） | `F051StyleFragment.java:800` 只处理 0/1/2；`fragment_f51_style.xml` 只有三个标签 |
| F045 GS Lite / F047 Atom / F048 GS Links | 0 | 命令类无 `setMAF`，本身不带 ANC |

> 注意与 GAIA 机型的 ANC 语义**不同**：GAIA 是 `0 关 / 1 降噪 / 2 监听 / 3 开放 / 4 风中`（见 `heat_info_fragment\adjust_zao_fragment.java:509 setZaoInfo`，其中 `anc==3` 落「开放」、`anc==4` 落「风中」）。**两边 2 和 3 的语义正好相反**（Mate 2=开放 3=兼听；GAIA 2=监听 3=开放），模块用 `FiilModels.ancLabel()` 与 `mateMafLabel()` 分别渲染。

**回包 5/6 归一化**：`F049MainFragment.java:344-349`、`F051MainFragment.java:344-349`、`F053MainFragment.java:344-349` 三处结构完全相同 —— `buffer[0]==6 → 4`（风中）、`==5 → 3`（兼听）。说明部分固件会把这两档上报为 5/6，模块 `FiilRfcommController.applyMateNotify()` 已同步该映射。

### 3.6 EQ 读写（关键细节，易错）

**读**（`F046StyleFragment.java:153/219`）：

```java
if (buff.length >= 12) {
    f29301i = buff;                      // 整块保存
    list = buff[1 .. length-1];          // 跳过 [0]
    setEq(list);                         // → m(list.get(0)=buff[1], list)
}
```

**写自定义**（`s0()` @L890-903）：

```java
if (f29301i.length < 12) return;         // 没读到过就不写
f29301i[1] = 15;                         // 15 = 自定义
for (i = 0; i < 滑块数; i++)
    f29301i[i + 2] = 滑块值;             // 从 [2] 开始
setEQ(f29301i);                          // 整块 12 字节发回
```

**写预设**（`z0(int type)` @L986-999）：只把 `f29301i[1]` 换成预设号，整块发回。

**结论（payload 布局）**：

```
[0]  保留位（沿用读取回包的值，模块填 0）
[1]  档位：3~14 = 预设（Nostalgia…New Age），15 = 自定义
[2..] N 段增益，0~255 偏移编码，128 = 0 dB
```

**预设编号映射**：`n()` @L765 构造 `new c(icon, selIcon, name, i2 + 3)` → 第 i 个预设（0 起）= 线值 `i + 3`。

**频段数量按机型不同**（`arrays.xml`）：

| 机型 | 段数 | 中心频率 | 来源 |
|---|---|---|---|
| 默认（F046/F047/F048/F051/F053） | 10 | 31 / 63 / 125 / 250 / 500 / 1k / 2k / 4k / 8k / 16k | `customize_eq` @L87 |
| F042 / F043 | 10 | 200 / 280 / 400 / 550 / 770 / 1k / 2k / 4k / 8k / 12k | `f042_customize_eq` @L118 |
| F045 | 5 | 63 / 250 / 1k / 4k / 8k | `f045_customize_eq` @L158 |
| F049 | 8 | 125 / 250 / 500 / 1k / 2k / 4k / 8k / 16k | `f049_customize_eq` @L165 |

> 各机型的读校验长度：F046 要求 `>= 12`，F045/F047/F048/F049/F051/F053 只要求 `>= 7`（宽松，实际长度以回包为准）。模块侧按 `payload.size - 2` 动态取段数，不写死 10。

### 3.7 低延迟：GAIA 三档 / Mate 二态

**这是本文档曾经写错、现已纠正的一节。** 早前版本断言「协议只有二态、视频是模块自造的语义档」，那是只读了 `F041SetFragment`（F041 机型确实是二态开关）就外推的结论。补查后确认：**GAIA 链路是真正的三档**。

线值：

| 值 | 语义 | 依据 |
|---|---|---|
| 0 | 音乐模式（默认） | `firstCCPro2SetFragment.I()` @:917 → `setLowLatency(0)` |
| 1 | 游戏模式 | 同上 @:935 → `setLowLatency(1)` |
| 2 | 视频模式 | 同上 @:952 → `setLowLatency(2)` |

链路：

- 写（GAIA）：`FF 01 00 01 02 34 07 22 [level]`，cmd = 1826 = `0x0722`
  —— `FiilCommandUtil.setLowLatency(int i2)` @:1022 把入参原样当 **1 字节 payload** 下发：
  `getSendBuffer(VENDOR_FF, "FFCOMMAND_SetLowLatency", false, (byte) i2)`。
- 写（Mate）：`classifyId 37`，payload `[0/1]` —— `com\fengeek\main\i\d\c.java:388`
  `setLowDelay(boolean)` 只接受布尔，**没有视频档**。
- 读（GAIA）：`07 21`（GET_LOW_LATENCY = 1825），返回 `payload[1]` 即 0/1/2。
- 读（Mate）：`classifyId 39, cmdId 8`，`payload[0]==1` = 开。

UI 旁证：`fragment_first_cc_pro2_set_new.xml` L35/39/43 三个 `movie_textView1/2/3`
文案分别是「游戏模式 / 音乐模式 / 视频模式」，点击后走上面同一组 `setLowLatency(0/1/2)`。

模块行为：GAIA 机型原样下发 0/1/2；Mate 机型只有二态，选「视频」时降级下发 1 并在日志里说明。

**真机复核（2026-10-25，FIIL Key Max / 固件 3.0.1.1 / 走 BLE GATT）**：Mate 的 cid=37 cmdId=0 实测**接受 0/1/2 三个值**，SET 回 ack 后
GET_LL(39/8) 读回与写入值一致，并非只有布尔二态。因此模块侧把 Mate 也放开为三档：
FiilMate.Param.LOW_LATENCY = 37、payload [level]。**「Mate 二态」是早期只读 setLowDelay(boolean) 签名得出的推断，已被真机推翻。**

### 3.7.1 语音提示音量（Mate 专有，cid=1 cmdId=3）

| 项 | 值 |
|---|---|
| 写 | 00 01 01 00 03 03 01 [vol]（com\fengeek\main\i\d\f.java:711-726 setTone，buffer {vol}） |
| 读 | 00 27 01 00 02 82 00（cid=39，cmdId=130 = -126 & 0xFF，c.java:251 getToneStatus） |
| 读回样本 | 01 27 02 00 03 82 01 02 |
| 线值 | **0=音量高 / 1=中 / 2=低 / 3=静音**（与直觉相反，0 最响） |

**读回偏移的坑**：上面的回包逐字节是
seq=01 | cid=27 | type=02 | reserved=00 | lenField=03 | cmd=82 | payloadLen=01 | payload=02
—— 长度字段 0x03 减 2 得 payloadLen = 1，**payload 只有 1 字节**，音量就在 payload[0]。
早期把 01（payloadLen）误读成「字段存在标志」，据此写成「音量在 payload[1]」，
靠 else 读 payload[0] 的容错才侥幸读对；现已改成真实偏移并加单测钉死。


### 3.8 多点连接（蓝牙畅连）

- 写：`classifyId 1` **+ `cmdId 1`**，payload `[0/1]` → 线值 `00 01 01 00 03 01 [0/1]`
- 读：`classifyId 39, cmdId 128`，`payload[0]==1` = 开

### 3.9 影院模式（空间音频）

- 写：`classifyId 50`，payload `[0/1]` → 线值 `00 32 01 00 03 01 [0/1]`
- 读：`classifyId 39, cmdId 24`，`payload[0]==1` = 开
- **仅 F046（Key Max）机型实现了该命令**（`setSpatialAudio` 只在 `com\fengeek\main\i\d\c.java` 定义，全库唯一调用点是 `F046StyleFragment.java:560/842/897/996`；见 §3.4.1 覆盖差异表）。模块对非 F046 机型不做该入口，而不是「Mate 机型全都支持」。
- APP 里影院模式开启时切 EQ 会**先自动关影院**（`s0()` @L896-898、`z0()` @L995-997），模块未复刻这一联动（见 §6 未实现项）。

### 3.10 佩戴检测（写 cid 38 / 读 cid 39 cmdId 9）

| 项目 | Mate（F051 / F053） | GAIA |
|---|---|---|
| 写 | `00 26 01 00 01 [1/0]` —— **cid=38，无 cmdId**，值 `bool?1:0` | cmd `SET_WEAR=263`（0x0107），值 **1=开、2=关** |
| 读 | `00 27 01 00 00 09` —— cid=39，cmdId=9，无 payload | cmd `GET_WEARING_SWITCH=1917`（0x077D），integer 回 `payload[1]` |
| 通知 | cid=40，cmdId=9，`payload[0]==1` = 已佩戴（单值，不分左右） | `GET_WEARING_STATE=1912`，左右分列 |
| 取证 | `g.java:663`/`h.java:729`（写）、`g.java:267`/`h.java`（读）；`F051SetFragment.java:662/745`、`F053SetFragment.java:717/783` | `FiilCommandUtil.java:1367-1375`（写，`z ? 1 : 2`）、`:653-658`（读 1917）；`GaiaSdkCommand.java:1199 analysisGetWearingSwitch`、`:1159 analysisGetWearStatus`、`:2305 analysisWear`（`isOpen()` = `bArr[1]==1`） |

> **两套协议取值不同，这是最容易写错的一条**：Mate 的「关」是 `0`，GAIA 的「关」是 `2`。
> 早期模块把 GAIA 也写成 `0`，会被耳机当作未知值。
>
> **「开关」与「状态」是两条不同语义**：cid39/cmdId9 问「摘戴检测功能开没开」，cid40/cmdId9 报「当前戴没戴」，模块分别映射到 `wearDetect` 与 `wearLeft/wearRight`。

---

### 3.11 自适应开关（cid=39 查 / cid=1 写，五个独立开关）

这是一组**与 MAF 档位并列的独立布尔开关**，不是 MAF 的第 5 档。
FIIL+ 界面里它们散落在「智能调噪」「音质」两页，逆向时容易被误当成 MAF 的子项。

#### 3.11.1 命令线值

写入统一是 `cid=1`（`FiilMate.Cid.SET_PARAM`），payload 恒为 1 字节 `[1/0]`；
读取统一是 `cid=39`（`QUERY`），无 payload。
**读取的 cmdId 在原码里写成 `NetError.ERR_*` 常量（负数），必须换算成无符号字节**：

| 功能 | 写（cid=1, cmdId） | 读（cid=39, cmdId） | 原码常量 | 机型覆盖 |
|---|---|---|---|---|
| 云感调音 Tuning | 8 | 135 | `ERR_SOCKS_CONNECTION_HOST_UNREACHABLE` = -121 | F049 |
| 自适应 EQ | 14 | 137 | `ERR_HOST_RESOLVER_QUEUE_TOO_LARGE` = -119 | F049 / F051 / F053 |
| 耳道自适应降噪 | 15 | 138 | `ERR_CONNECTION_TIMED_OUT` = -118 | F049 / F053 |
| 环境自适应降噪 | 16 | 139 | `ERR_BAD_SSL_CLIENT_AUTH_CERT` = -117 | F049 / F051 / F053 |
| 自适应抗风噪 | 17 | 140 | `ERR_CERT_ERROR_IN_SSL_RENEGOTIATION` = -116 | F049 |

取证：`com\fengeek\main\i\d\f.java`（F049）的
`setEQAdaptive:460` / `setEarAdaptive:477` / `setNoiseAdaptive:612` / `setTuning:728` / `setWindAdaptive:762`，
与 `getEQAdaptive:59` / `getEarAdaptive:91` / `getNoiseAdaptive:219` / `getTuning:283` / `getWindAdaptive:315`。
`g.java`（F051）只有 EQ + Noise 两组，`h.java`（F053）只有 EQ + Ear + Noise 三组——
与三个机型的布局文件 `fragment_f{49,51,53}_style.xml` 里的开关数量完全吻合。

#### 3.11.2 帧示例

```
写「开环境自适应降噪」  → 00 01 01 00 03 10 01 01
                          │  │  │  │  │  │  └─ payload=[1]
                          │  │  │  │  │  └──── cmdId=0x10=16 (NOISE_ADAPTIVE)
                          │  │  │  │  └─────── len=3 (cmdId+payloadLen+payload)
                          │  │  │  └────────── 保留
                          │  │  └───────────── ver=1
                          │  └──────────────── cid=1 (SET_PARAM)
                          └─────────────────── SOF=0x00

读「环境自适应降噪状态」 → 00 27 01 00 02 8B 00
                          │  │  │  │  │  │  └─ payloadLen=0
                          │  │  │  │  │  └──── cmdId=0x8B=139 (NOISE_ADAPTIVE)
                          │  │  │  │  └─────── len=2
                          │  │  │  └────────── 保留
                          │  │  └───────────── ver=1
                          │  └──────────────── cid=39 (QUERY)
                          └─────────────────── SOF=0x00
```

回包 `payload[0]==1` 为开、`0` 为关。

#### 3.11.3 易错点

1. **`Query` 里 138/139 与 `LEFT_UUID`/`RIGHT_UUID` 数值撞车**：
   前者来自 `c.java`（F046 命令类），后者来自 `f/g/h.java`（F049/F051/F053）。
   同一 cmdId 在不同机型命令类里语义不同，**必须按 earType 门控**，
   否则会给 F046 发一条它不认识的 138，耳机静默丢弃、模块白等 2.5s 再误报失败。
2. 这五个开关**不跟随 MAF 档位**：关掉降噪后「环境自适应降噪」仍可独立置位。
3. `setTuning`（云感调音）与 `setEQAdaptive` 都会影响音色，但前者是硬件级反馈调音，
   后者是软件 EQ 补偿。二者可同时开。

#### 3.11.4 回显不是状态（极易写错的地方）

`cid=1` 的回包**只表示「写成功」，不带新状态**，实测 payload 为空或单字节 `0x00`。
换言之 `[0]` 是成功码，不是「关」。同理 `cid=32/37/38/50`（EQ / 低延迟 / 佩戴 / 空间音频）
的 SET 回显也是空包。

三种错误写法及后果：

| 写法 | 后果 |
|---|---|
| `state = payload[0] == 1` | 开成功后被记成关闭（`[0]` 恒为 0） |
| `state = !state`（取反推定） | 单次操作正确；连点两次、或回显丢失后重发，就与真实状态错位 |
| 发送时记目标值、回显时取用 | 正确 —— 模块 `pendingTargets` 即此 |

真实状态只能来自 `cid=39` 的查询回包（或 `cid=40` 的主动上报）。
模块里因此把两条路径分开：`applyAdaptiveFlag()` 处理 cid=39 的直接取值，
`applyAdaptiveEcho()` / `applySetParam()` 处理 SET 回显，只认 `pendingTargets`。

### 3.12 GET 与 SET 是**两套完全不同的帧语义**（实战最大坑）

2026-10-25 用 PC 端 Bleak 直连 FIIL Key Max 逐条实测，踩了一整晚的坑，结论如下：

**GET（查询）：`classifyId` 固定 39，`commandId` 填查询码**

```
00 27 01 00 02 0C 00        查 MAF
└┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘
seq  cid=39 t=1 rsv len=2 cmd=12 len=0
```

**SET（设置）：`classifyId` = 命令族编号，`commandId` = 0**

```
02 2C 01 00 01 02           设 MAF=2
└┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘
seq  cid=44 t=1 rsv len=1 payload
                     ↑ 注意：这里**只有一层长度**，没有 cmdId
```

| | classifyId | commandId | 长度字段个数 |
|---|---|---|---|
| **GET** | 恒为 39 | 查询码 | 2 个（`len+2` 与 `len`） |
| **SET** | 命令族（1/32/33/37/38/44/50…） | **恒为 0** | 1 个（`len`） |

> **坑 1（我踩了）**：第一版把 SET 也写成 `cid=39 + commandId=设置码`，耳机对所有 SET 都回一个**空帧** `xx 27 02 00 00`（`lenField=0`）。空帧 = 「这条命令我不认识」，不是「执行成功」。
>
> **坑 2（我踩了）**：修正 cid 后仍失败，因为把 `payloadLen` 写成了 `if/else` 互斥分支。实际上 `writeMate` 在 `if (commandId != 0)` 之后**无条件**再写一次 `payloadLen` —— 两个长度字段在 cmdId≠0 时都要写，在 cmdId=0 时只写后者。

**SET 应答**：`[seq][cid][02][00][01][00]` —— 5 字节，**`payload[0]==0` 才是成功**（1 = 失败）。
**读回确认**：SET 不会主动回新值，必须再发一次 GET 才能确认（部分机型会额外推一帧 `cid=40` NOTIFY）。

**已逐条实测通过的 SET（FIIL Key Max，固件 3.0.1.1）**

| 功能 | 发送帧 | 应答 | 读回确认 |
|---|---|---|---|
| MAF 降噪模式 | `02 2C 01 00 01 02` | `02 2C 02 00 01 00` | `GET_MAF` → `0C 01 02` ✓ |
| 低延迟 | `07 25 01 00 01 01` | `08 25 02 00 01 00` | `GET_LL` → `08 01 01` ✓ |
| 多点畅连 | `07 01 01 00 03 01 01 01` | `07 01 02 00 03 01 01 00` | 设完**BLE 会掉线**（正常，切换主从），重连后读回 ✓ |
| 影院/空间音频 | `02 32 01 00 01 01` | `02 32 02 00 01 00` | `GET_SPATIAL` → `18 01 01` ✓ |
| EQ 12 字节 | `0C 20 01 00 0C 0A 0E …` | `03 20 02 00 01 01` | **读回不变**（见下） |
| 播放/暂停 | `0E 21 01 00 03 02 01 01` | `07 21 02 00 03 02 01 00` | — |
| 下一曲 / 上一曲 | `0F 21 01 00 02 05 00` / `00 21 01 00 02 04 00` | 均有应答 | — |

> **EQ 的特殊性**：`SET_EQ`（cid=32）**返回成功（`payload[0]==1`… 实为状态码 1）但读回曲线不变**。原因是 Key Max 的 EQ 走的是「预设 + 自定义」双轨，写入需要先切到自定义档（`setEQAdaptive` cid=1 cmd=14），或者该机型不接受任意曲线写入。**这条仍是未确认项**，模块的 EQ 自定义滑块暂只对 GAIA 机型承诺生效。

### 3.13 耳机主动上报（cid=40 NOTIFY）实测样本

Mate 通道上耳机在电量变化/摘下时主动推：

```
0B 28 03 00 0E 83 0C D8 80 83 00 00 01 00 C3 0A B3 7F 60
└┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └─────────────────────────────┘
seq  cid=40 t=3 rsv len=14 cmd=0x83(131) len=12  payload(12B，疑似对端设备 MAC + 状态)
```

- `type` 字段（偏移 2）是 **`0x01`=REQUEST / `0x02`=RESPONSE / `0x03`=NOTIFY**，不是版本号。
  **旧实现把它写死成 `!=0x01 就丢弃`，导致所有 RESPONSE 被当噪声扔掉** —— 这是「UI 一直显示未连接」的直接原因之一。
- `cid=40` 下 `cmd=0x83(131)` 不在已知表内，暂按「多点对端信息」处理（12 字节里能读出 `D8:80:83:00:00:01` = 另一台已配对设备）。

---

## 4. 豆包链路 0x55AA（少数机型）

来源：`com\fengeek\main\i\b.java:187` 的闸门，仅 **earType 47 / 48** 启用。

```
55 AA | 00 | LEN_LO LEN_HI | payload... | A5 5A
```

- 与 Mate 帧在**同一 socket** 上交错，读取循环按首字节分流（`0x55` = 豆包，`0x00` = Mate，`0xFF` = GAIA）。
- 模块当前**未实现**豆包链路；这些机型回落到 Mate 路径（`0x00` 帧仍可解析）。

---

## 5. 机型与路由

### 5.1 earType → 型号名

| earType | 型号 | 链路 |
|---|---|---|
| 1~44 | 见 `FiilModels.NAMES` | GAIA |
| 45 | FIIL GS Lite | Mate |
| 46 | FIIL Key Max | Mate |
| 47 | FIIL Atom | Mate + 豆包 |
| 48 | FIIL GS Links | Mate + 豆包 |
| 49 | FIIL Key Pro2 | Mate |
| 51 | FIIL CC3 | Mate |
| 53 | FIIL CC Pro3 | Mate |
| 39 | CC Pro2 | Airoha BLE |
| 40 | Belt | Airoha BLE |
| 42 | Box | Airoha BLE |

> 名称来源：`apktool_out\res\values\strings.xml` 的 `fiil_f045`~`fiil_f053` 等条目。

### 5.2 握手序列

**GAIA 机型**（`com/fiil/sdk/c/a.java:132-155`）：版本(769) → 机型(777) → 电量(771) → ANC(641)。

**Mate 机型**（无 earType 命令）：版本(39/2) → 电量(39/1) → MAF(39/12) → 低延迟(39/8) → 影院(39/24) → 多点(39/128)。

模块实现：`FIILProtocol.handshake()`，`FiilRfcommController.connect()` 里调用一次。

### 5.3 手动指定型号（型号识别兜底）

`GET_EARTYPE`（GAIA 777）与 Mate 链路都没有「查询型号」的通用命令（Mate 机型根本不提供 earType 回包，模块靠蓝牙名里的型号词推断）。因此模块提供一条人工兜底：

| 项 | 值 |
|---|---|
| 存储 | `SharedPreferences("pods_config")` 的 `ear_type_override`（0 = 未指定）；读写见 `FiilConfig.readEarTypeOverride` / `writeEarTypeOverride` |
| 地址绑定 | 同文件键 `ear_type_for_<MAC>`；手动指定、蓝牙名推断、`GET_EARTYPE` 回包三条路径识别出的型号都会落一份，下次同地址连接直接复用 |
| 生效时机 | `FiilRfcommController.connect()` 开头先读它；非 0 时直接采用，且 `earTypeOverridden = true` |
| 优先级 | 手动指定 > 地址绑定 > 蓝牙**原始名**推断 > `-1` 未知（等回包）；显示名用别名，型号识别**只用原始名** |
| 自动回包 | override 生效时，`GET_EARTYPE` 的回包**不覆盖**已指定的值（否则连上就被改回去） |
| 协议族切换 | 指定为 Mate 机型（45/46/47/48/49/51/53）后，`isMateDialect` 按机型判断，握手走 Mate 序列 |
| 恢复自动 | 传 `-1`，清空偏好并先查地址绑定、再退回原始名推断 |

广播通道：`ACTION_COMMAND` + `EXTRA_CMD="setEarType"` + `EXTRA_VALUE=earType`（-1 为恢复自动）。

### 5.4 设备识别：三路兜底

| 手段 | 依据 | 覆盖场景 |
|---|---|---|
| 名字关键字 | 名字含 `FIIL` / `FIIL CC` / `FIIL GS` / `FIIL KEY` / `FIIL T` / `FIIL CARAT` / `FIIL DIVA` | 出厂名未被改 |
| MAC 前缀 | `B0:F1:A3`、`FF:F1:A3` | 蓝牙名被用户改得看不出品牌 |
| 地址绑定 | `ear_type_for_<MAC>` 偏好项 | 曾识别过型号、但名字已改 |

MAC 前缀的逆向出处：FIIL+ 在所有 OTA 页都把 `device.getAddress().startsWith("B0:F1:A3")` 当作「是不是自家设备」的硬判定 —— `jadx_out/sources/com\fengeek\main\base\BaseFragment.java:28`，以及 F039 / F040 / F041 / F043 / F045 / F046 / F048 / F049 / F051 / F053 各自的 OTA Activity（共 15 处）；`F042HomeFragment.java:1014` 另有一处 `FF:F1:A3` 变体。模块侧实现见 `FiilConfig.isFiilAddress()` / `isFiilDevice(name, address)`，调用点覆盖蓝牙进程 Hook、设置页 Hook、融合设备中心 Hook、已配对设备枚举四处。

---

## 6. 未确认项与已知限制

| # | 项 | 状态 |
|---|---|---|
| 1 | F042 BLE 私有特征完整 UUID | 反编译产物被截断，未实现该链路 |
| 2 | 「视频」低延迟档 | **已纠正**：GAIA 链路真实存在 0/1/2 三档（见 §3.7）；仅 Mate 机型无视频档，模块降级为游戏 |
| 3 | 豆包 0x55AA 帧的 payload 结构 | 未展开逆向，模块不主动发该帧 |
| 4 | Mate 的 `[0]` 保留位语义 | 只知道 APP 原样回传，不知其含义；模块填 0 |
| 5 | 影院模式与 EQ 的互斥联动 | APP 有（切 EQ 先关影院），模块未复刻 |
| 6 | 佩戴检测（Mate 机型） | **已实现**：`setWear`/`getWear` 只在 `i.d.g`(F051)、`i.d.h`(F053) 存在；模块 `supportsWearDetect()` = setOf(51,53) 门控，UI 开关已接（见 §3.10） |
| 7 | 自适应（MAF=4）的可用性 | 由 `FiilModels.supportsExtendedAnc()` 按机型白名单判断，未经真机验证 |
| 8 | 多机型同时连接 | 控制器是单例，仅维护一条 SPP 连接 |
| 9 | 各机型命令差异未真机验证 | 上表由反编译方法声明推导，实际固件是否全部实现需真机逐型号验证 |
| 10 | 融合设备中心（`com.milink.service`） | **已实现**（见 §7），但依赖 HyperOS 内部类签名；系统版本升级后 hook 点可能失效，失败时只打日志不崩溃 |
| 11 | Mate 机型自定义 EQ 写入 | **实测不生效**：`SET_EQ`（cid=32）返回成功码但读回曲线不变（Key Max 固件 3.0.1.1，见 §3.12）。可能是「必须先切自定义档」或「该机型只接受预设」，未确认 |
| 12 | `cid=40` / `cmd=0x83(131)` 上报 | 实测能收到 12 字节 payload（疑似多点对端 MAC），语义未完全逆向，模块只记日志不改状态 |
| 13 | SPP 与 BLE GATT 的并发 | 两条通道同时开会怎样未测；模块当前互斥，同一时刻只走一条 |
| 14 | FIIL Key Max 的 SPP 表现 | 实测标准 SPP 连上即静默、自定义 UUID 连上被反复踢（`scn:6 opened=21 leaked=21`）。**这是换用 BLE GATT 的直接原因**；其他 Mate 机型是否同样，未测 |

---

## 6.1 融合设备中心侧的系统接口

融合设备中心不是 FIIL 协议的一部分，而是模块**伪造给 HyperOS** 的一层。列在这里是因为它同样随系统版本漂移。

| 系统类 / 方法 | 作用 | 我方返回 |
|---|---|---|
| `MxBluetoothService/Manager.checkIsMiTWS(BluetoothDevice)` | 是否米 TWS | `1` |
| `.getDeviceId(BluetoothDevice)` | 设备 ID | `01010607` |
| `.getBatteryLevel(...)` | 电量 | 取左右耳较低者 |
| `.getAncState(...)` | 降噪状态 | MAF 五态 → MIUI 三态 |
| `.getWearStatus(...)` | 佩戴 | `"0,0"` |
| `.isLeAudio(...)` | LE Audio | `false` |
| `ProfileContext.getBatteryLevel(BluetoothDevice)` | 三路明细 | `[盒, 左, 右, 充1, 充2, 充3]`，未连接 `-1` |
| `ProfileContext.getAudioSpatialEffectState(...)` | 空间音频 | 影院开 → `2`，关 → `0` |
| `AncBatteryController.getFindRingState(...)` | 游戏模式位 | 开 → `100`，关 → `0` |
| `AncBatteryController.setAncStateBlock(device, int)` | 拖 ANC 滑块 | 拦截，转 `ACTION_ANC_SELECT` |
| `AncBatteryController.setMiAudioEffect(device, int)` | 空间音频开关 | 拦截，转 `ACTION_COMMAND("cinema")` |
| `AncBatteryController.setFindRing(device, int)` | 寻找耳机卡片 | 接管为低延迟游戏模式 |
| `AncBatteryController.headsetPropertyChangeListener.invoke(device, int)` | 反向刷新 UI | 4 电量 / 8 降噪 / 9 空间音频 / 10 游戏模式 |

> MIUI 三态与 FIIL MAF 的映射不是一一对应：FIIL 的「监听」和「风中」在系统侧都显示为「通透」。控制中心只能选三档，五档仍然在模块自己的界面里选。

---

## 7. 模块侧实现对照

| 协议能力 | 模块入口 | 文件 |
|---|---|---|
| 语义化命令封装 | `FIILProtocol.requestBattery()` / `setMAFMode()` / `setCinemaMode()` / `setLowLatencyMode()` / `setMultipoint()` / `setWearDetect()` / `readEQ()` / `writeEQ()` | `pods\FIILProtocol.kt` |
| GAIA 帧编解码 | `FiilGaia.buildFrame()` / `parse()` / `drain()` | `pods\FiilGaia.kt` |
| Mate 帧编解码 | `FiilMate.buildFrame()` / `parse()` / `drain()` / `setFrame()` | `pods\FiilMate.kt` |
| SPP 连接与收发 | `FiilRfcommController.openSocket()` / `connect()` / `readLoop()` / `sendMate()` | `pods\FiilRfcommController.kt` |
| **BLE GATT 连接与收发** | `FiilGatt.connect()` / `send()` / `handleRx()` / `close()` | `pods\FiilGatt.kt` |
| **SPP→GATT 自动降级** | `FiilRfcommController.connectBlocking()` 内：SPP 全失败或 3.5 s 零回包 → `useGatt = true` | `pods\FiilRfcommController.kt` |
| 状态缓存 | `FiilStatus` + `StatusRepo` | `pods\FiilStatus.kt`、`ui\StatusRepo.kt` |
| 设置页注入 | `SettingsHook`（含「⋮」菜单 → 模块控制台入口 `hookModuleEntry()`） | `hook\SettingsHook.kt` |
| 融合设备中心 | `MiLinkServiceHook` / `MiLinkSpatialAudioHook` | `hook\milink\MiLinkServiceHook.kt`、`hook\milink\MiLinkSpatialAudioHook.kt` |
| 超级岛 | `MiBluetoothToastHook` | `hook\MiBluetoothToastHook.kt` |

---

## 8. 合规提醒

本文档的目的是**学习蓝牙协议栈设计**与为自己的耳机开发本地控制模块。所有命令均来自公开可分发的 APK 静态分析。请勿用于：

- 重新分发 FIIL 的私有协议实现或资源文件；
- 绕过设备的固件校验或进行未授权的 OTA；
- 商业产品中声称与 FIIL 官方兼容。

本仓库只包含文档，不含任何代码。上游参考项目（HyperPods / HuaweiPods / OppoPods）以 GPL-3.0 授权，其代码**未**被复制进本仓库；相关方法论的引用与边界见 [NOTICE](../NOTICE)。
