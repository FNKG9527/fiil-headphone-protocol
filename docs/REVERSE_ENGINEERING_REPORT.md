# FIIL+ 3.5.2 逆向拆包分析报告

> 目标：搞清楚 FIIL+（斐耳）App **怎么连上自家耳机**、**怎么控制**、**有哪些功能**、**怎么发命令**。
> 性质：学习 / 研究用途。全部结论来自对 APK 的静态反编译阅读（jadx + apktool），无线上攻击行为。

---

## 0. 被测样本信息

| 项 | 值 |
|---|---|
| 文件 | `fiil+_3.5.2.apk`（94,576,645 字节） |
| 包名 | `com.fengeek.f002` |
| versionName | `3.5.2` |
| versionCode | `175` |
| minSdk / targetSdk / compileSdk | 26 / 33 / 33 |
| 反编译产物 | `jadx_out\sources`（Java）、`apktool_out`（资源+smali） |
| 关键 SDK 包 | `com.fiil.sdk`（自家 GAIA 协议栈）、`com.bluetrum`（中科蓝讯芯片栈）、`com.airoha`（络达芯片栈）、`com.fengeek.main.ble`（F042 系列 BLE 私有栈） |
| 第三方 | 豆包语音 SDK（`com.startburst.sdk`）、Firebase Crashlytics、FastBle |

---

## 1. 一句话总览

App **不是只走一条链路**。它内部并列了 **4 套完全不同的传输/协议栈**，按「耳机芯片方案 + earType 机型号」分流：

| # | 链路 | 传输方式 | 适用机型 | 代码入口 |
|---|---|---|---|---|
| A | **GAIA over SPP** | 经典蓝牙 SPP（RFCOMM） | 大多数老/主力机型（FIIL、Diva、Carat、T1、CC、CG、Driifter…） | `com.fiil.sdk.gaia.father.b`（GaiaLink）+ `FiilManager` |
| B | **Bluetrum 私有 over SPP** | 经典蓝牙 SPP，**手机开服务端**等耳机连入 | 中科蓝讯方案机型（AB53xx 系列） | `com.bluetrum.devicemanager` |
| C | **Airoha 私有** | BLE GATT 自定义服务 + SPP 双通道 | 络达方案机型（CC Pro2、F042/F043 等） | `com.airoha.*` + `F042MainFragment` |
| D | **F042 BLE 私有** | 纯 BLE GATT（自定义 128 位 UUID） | F042 / CC Pro2 等新机型 | `com.fengeek.main.ble.FIILPeripheral` |

另外还有一条**第 5 类业务通道**：**豆包语音**（`0x55AA` 帧）只跑在 earType 47/48 的机型上（Atom / GS Links），帧格式见 §3.4，机型见 §5。

> 补充一个容易踩的坑：**A/B/C/D 不是"按机型严格一一对应"的**。CC Pro2 就是个混合体——它用 Bluetrum 的 Beacon 结构做发现（§2.2），建链时却调 `FIILCCPro2Connecting()` 借用 Bluetrum 的私有 SPP UUID（`d52b47bc-…`）当客户端，但上层跑的仍是 GAIA 帧（§2.3 B-2）。所以判断某台具体设备走哪条路，**只能看它的机型页调了哪个 `connecting` 重载**。

---

## 2. 连接流程：App 是怎么"找到并连上"耳机的

### 2.1 第一步：BLE 广播扫描发现设备

`com.fengeek.main.ble.b`（`FIILCentralManager`，该类本身 `extends ScanCallback`）负责扫描（`com/fengeek/main/ble/b.java`）：

- `startScan()` @L109 → 直接 `BluetoothAdapter.getDefaultAdapter().getBluetoothLeScanner().startScan(f27219b)` @L131，**不传 ScanFilter，也不传 ScanSettings**（走系统默认的 SCAN_MODE_LOW_LATENCY 语义）；
- 30 秒超时：常量 `f27220c = 30000` @L25，`c()` @L78 起 `new CountDownTimer(30000L, 1000L).start()`，`onFinish()` @L46 → `stopScan()` + 回调 `onScanFinished(new ArrayList(hashSet))`；
- 结果去重：`HashSet<com.fengeek.main.ble.e.a>` @L28，`onScanResult` @L104 `if (this.f27221d.add(aVar))` 才回调 `onScanning`；
- 设备包装类 `com.fengeek.main.ble.e.a`（`BluetoothDevice` + `rssi` + `scanRecord.getBytes()` 原始广播 + 名字）@L101。

> 结论：**发现阶段不做任何过滤**，所有 BLE 设备都会进回调；"是不是 FIIL 的耳机"完全靠下一步解析广播里的厂商数据来判断。

### 2.2 第二步：解析厂商广播数据（Beacon）

这是识别"这是不是我家耳机"的关键。`com.bluetrum.devicemanager.models.DeviceBeacon`：

```java
static boolean isDeviceBeacon(byte[] msd) {
    if (msd.length == 5  && msd[0] == 39) return true;               // V3
    if (msd.length == 13) {
        int v = msd[0] & 0x0F;
        return v == 1 || v == 2;                                     // V1 / V2
    }
    return false;
}

static DeviceBeacon getDeviceBeacon(byte[] msd) {
    if (msd.length == 5 && msd[0] == 39) return new DeviceBeaconV3(msd);
    int v = msd[0] & 0x0F;
    if (v == 1) return new DeviceBeaconV1(msd);
    if (v == 2) return new DeviceBeaconV2(msd);
    return null;
}
```

三种版本的载荷布局（都已逐字节核对）：

**V1（13 字节，`DeviceBeaconV1.java`）**

| 偏移 | 长度 | 含义 |
|---|---|---|
| 0 | 1 | version（低 4 位）+ 其它标志位 |
| 1–2 | 2 | productId（**小端** short，`byteBuffer.getShort() & 0xFFFF`，基类 `DeviceBeacon.java` 构造器） |
| 3–8 | 6 | 耳机蓝牙 MAC（**正序**原始字节，`BluetoothUtils.getAddressStringFromByte` 直接 `%02X:%02X:...` 顺序拼，不做反转） |
| 9 | 1 | bit0 = needAuth；bits2–3 = 连接状态（`(b2>>2)&3`） |
| 10 | 1 | 左耳：bit7 = 充电中，bit0–6 = 电量%（`b3 & 127`） |
| 11 | 1 | 右耳：同上 |
| 12 | 1 | 充电盒：同上 |

**V2（13 字节，`DeviceBeaconV2.java`）**

| 偏移 | 长度 | 含义 |
|---|---|---|
| 0 | 1 | version（低 4 位） |
| 1–2 | 2 | productId（小端，同 V1，先被基类吃掉） |
| 3–8 | 6 | 耳机 MAC，**每字节 XOR 0xAD** 后才还原（`btAddress[i] ^= (byte)0xAD`，`-83` 即 0xAD） |
| 9 | 1 | bit0 = needAuth；bits2–3 = 连接状态 |
| 10–12 | 3 | brandId（小端 24 位：`get() \| (get()<<8) \| (get()<<16)`）；**但如果 `data.length >= 12`，brandId 被覆写为单字节 `data[11]`** |

**V3（5 字节，最新一代）**

```java
public DeviceBeaconV3(byte[] data) {
    super(data, 0);                       // 基类先吃掉 data[0]、data[1]
    this.f16440h = "LE-FIIL CC Pro2";
    byte[] bArr = new byte[3];
    this.byteBuffer.get(bArr);            // data[2..4]
    StringBuilder sb = new StringBuilder();
    sb.append("B0:F1:A3");                // 前缀硬编码
    for (int i = 2; i >= 0; i--) {        // 倒序拼接 → MAC 是反着放的
        sb.append(':');
        sb.append(convertToHexString(bArr[i] & 255));
    }
    this.mBtAddress = sb.toString();
    setOtherName("LE-FIIL CC Pro2");
    setOtherAddress(this.mBtAddress);
    if (data.length == 5) this.brandId = data[0];
}
```

| 偏移 | 长度 | 含义 |
|---|---|---|
| 0 | 1 | brandId，固定 `39`（0x27，即 FIIL 的厂商号）—— 基类 `DeviceBeacon(data, 0)` 把 `data[0]` 同时塞进 `productId` 和 `brandId` |
| 1 | 1 | version（**低 4 位**，由基类 `& 15` 取出） |
| 2–4 | 3 | MAC 后三字节，**倒序**拼：`B0:F1:A3` + `data[4]` + `data[3]` + `data[2]`（循环 `for(i=2;i>=0;i--)` 逆序追加） |

> 注意 V3 只广播 MAC 的**后 3 字节 + 硬编码前缀 `B0:F1:A3`**，不是完整 MAC（前缀来自批量生产的 MAC 段，实际设备这个前缀未必真相同）。V3 还会硬塞一个名字 `LE-FIIL CC Pro2` 和 `otherAddress`——这说明 V3 是为 CC Pro2 这一代"LE Audio / 低功耗常连"设计的，完整 MAC 要靠后续 GATT 连接里读。另外 `DeviceBeaconV3` 用的是**大端** `ByteBuffer.wrap(data)`（`super(data, 0)` 那个构造器），和 V1/V2 的小端不同，读多字节字段时别搞混。


**brandId 过滤**（`com/fengeek/main/f040/amd/viewmodels/g0.java:128-136`、`com/fengeek/main/f039new/amd/viewmodels/f0.java:125-137`，两处逻辑一致，后者写成嵌套 `if`）：

```java
byte[] msd = scanRecord.getManufacturerSpecificData(1602);   // 0x0642
if (msd == null) msd = scanRecord.getManufacturerSpecificData(564);   // 0x0234 = VENDOR_FIIL

if (DeviceBeacon.isDeviceBeacon(msd) && (beacon = DeviceBeacon.getDeviceBeacon(msd)) != null
        && beacon.getAgentId() == 0
        && beacon.getBrandId() == this.headsetId) {
    // 认定为 FIIL 设备
}
```

即三道闸门：**Beacon 结构合法** → **agentId（`brandId >> 16`）为 0** → **brandId 等于 App 内配置的 headsetId**。这就是"只认自家耳机"的过滤逻辑——**不是靠名字前缀，而是靠广播里的厂商 ID 字段**。

`headsetId` 不是写死的常量，而是**每个机型页各自注入**的（`setHeadsetId(int)` → `startScan(headsetId)` → `this.f27647i = headsetId`，`com/fengeek/main/f040/amd/viewmodels/g0.java:162-171`）：

| 注入点 | 值 |
|---|---|
| `com/fengeek/main/HeatSetPagerCCPro2Fragment.java:510` | `39`（V3 Beacon 的 `brandId` 固定为 39，正好对上） |
| `com/fengeek/main/f040/ui/fragment/F40MainFragment.java:299` | `40` |
| `com/fengeek/main/f041/fragment/F041MainFragment.java:296` | `41` |

也就是说：**同一份扫描回调，被不同机型页用不同的 brandId 过滤**。CC Pro2 页只接受 `brandId==39` 的 V3 Beacon，F40 页只接受 `brandId==40` 的，F041 只接受 `41`。这解释了为什么 V3 Beacon 里 `brandId` 要硬编码成 `data[0]=39`——它就是给 CC Pro2 那一代做的一次性设计，而不是通用厂商号。

> 抓包提示：先按 `getManufacturerSpecificData(1602)` 取，取不到再退到 `564`。1602 = 0x0642 是蓝牙 SIG 分配给某些芯片厂商的 Company ID，564 = 0x0234 恰好就是 FIIL 自己在 GAIA 帧里用的 `VENDOR_FF`。看到广播里带着 `0x0234` 开头的 AD 结构，基本就是 FIIL 的耳机。
>
> 注意 **agentId 必须为 0**：`getAgentId()` 是 `getBrandId() >> 16`。对 V3（5 字节）来说 `brandId = data[0] = 39`，右移 16 位得 0，天然满足；对 V1/V2 来说 brandId 是 24 位或单字节，也只有高字节为 0 的才放行。这是"消费级耳机"与"代理/贴牌设备"的分流位。

### 2.3 第三步：建立数据连接

#### A. GAIA over SPP（老机型主力）

`FiilManager.connecting(String address)`（`com/fiil/sdk/manager/FiilManager.java:275`）：

```java
d dVar = MyFiilReceiver.f34272h;
d.f34410a.b(UUID.fromString("00001101-0000-1000-8000-00805F9B34FB"));  // 标准 SPP
Method m = this.f34446g.getClass().getDeclaredMethod("blueConn", d.class, String.class);
m.setAccessible(true);
m.invoke(this.f34446g, this.sdkService, str);
```

走**反射**调用 `MyFiilReceiver.blueConn(...)`，把 SPP UUID、目标 MAC 传进去；真正的 socket 在 `com.fiil.sdk.gaia.father.b`（`GaiaLink`，内部名 `GaiaSocket`）：

```java
// com/fiil/sdk/gaia/father/b.java:445  socket 工厂
protected BluetoothSocket a(UUID uuid) throws IOException {
    if (g()) return f34370h.createInsecureRfcommSocketToServiceRecord(uuid);  // 不安全模式
    return f34370h.createRfcommSocketToServiceRecord(uuid);                    // 安全模式
    // 失败兜底：反射调隐藏 API createRfcommSocket(1)
}
```

- 默认 UUID：`00001101-0000-1000-8000-00805F9B34FB`（标准 SPP，`b.java:236`）；
- 也支持 `00001107-D102-11E1-9B23-00025B00A5A5`（`b.java:29` 常量 `w`，部分老设备的私有 SPP 通道），通过 `b(UUID uuid)`（`b.java:305`）在连接前替换；
- 还有一种 **UDP 模拟器模式**（`c(String str)`，`b.java:309`）：`DatagramSocket` 连 `10.0.2.2`（Android 模拟器里指向宿主机的地址），用于无真机时联调——说明开发期做过纯软模拟；
- 连上后由 `com.fiil.sdk.gaia.father.a` 做**帧重组**（处理 SPP 的粘包/拆包）。

连上后的握手时序（`com/fiil/sdk/c/a.java:116-149`，`handleMessage` 收到"连接成功"消息后）：

```java
// 1) 反射把标志位置真
Method m1 = FiilManager.getInstance().getDeviceInfo().getClass()
        .getDeclaredMethod("setGaiaConnect", Boolean.TYPE);
m1.setAccessible(true);
m1.invoke(FiilManager.getInstance().getDeviceInfo(), Boolean.TRUE);

// 2) 拿固件版本（F039 机型跳过）
if (!FiilManager.isF039) FiilManager.getInstance().getVersion(null);

// 3) 拿机型号 —— 关键的一步
FiilManager.getInstance().getEarType(new C0338a());

// 4) 反射取出所有 ConnectionListener，逐个回调 onConnectionSuccess()
Method m2 = ConnectionUtils.getConnectionUtils().getClass()
        .getDeclaredMethod("getConnectionListeners", new Class[0]);
m2.setAccessible(true);
for (ConnectionListener l : (ArrayList<ConnectionListener>) m2.invoke(...)) l.onConnectionSuccess();
```

`C0338a`（`com/fiil/sdk/c/a.java:27-64`）是 earType 的回调：**成功时什么都不做**；**失败时降级去读 `getLedTemp`**（判断是不是老机型）。真正用到 earType 的地方是 `a()`（`com/fiil/sdk/c/a.java:245`）：

```java
public void a() {
    com.fiil.sdk.command.b bVar = new com.fiil.sdk.command.b();   // F002 专用命令类
    a("setEarType", 2, Integer.TYPE);                              // 反射写 DeviceInfo.setEarType(2)
    for (Class<?> c = com.fiil.sdk.command.b.class; c != com.fiil.sdk.command.a.class; c = c.getSuperclass()) {
        Method m = c.getDeclaredMethod("initOrder", new Class[0]); // 逐级找 initOrder 并调用
        m.setAccessible(true); m.invoke(bVar); break;
    }
    for (CommandEventListener l : b()) l.earType(2);               // 通知所有监听者"机型号=2"
}
```

也就是说：**连接成功 → 读版本 + 读 earType → 回调 UI 的 `onConnectionSuccess()`**，UI 自己再按 earType 建对应的 Fragment、按需拉电量/ANC/EQ 等状态。**不是** SDK 一次性把所有状态都读一遍。

#### B. Bluetrum 私有 SPP（手机当服务端！）

`com/bluetrum/devicemanager/bluetooth/BluetoothSppService.java`：

```java
private static final UUID f16296c = UUID.fromString("d52b47bc-2dea-42c4-b26d-20252fceeeb3");

// 关键：listenUsingInsecureRfcommWithServiceRecord —— 手机是服务端
BluetoothAdapter.getDefaultAdapter()
    .listenUsingInsecureRfcommWithServiceRecord("BluetoothSppService", f16296c);
```

**方向是反的**：手机开一个 SPP 服务端 socket 监听，耳机主动连进来（`accept()`）。这是中科蓝讯方案常见的做法（耳机侧做 SPP client，省电、可重连）。

#### B-2. CC Pro2 的混合路径（同一 UUID，两个方向）

`FiilManager` 里还有第三个连接入口，容易和老机型那条混淆：

```java
// com/fiil/sdk/manager/FiilManager.java:259
public void FIILCCPro2Connecting(String str) {
    isF039 = true;                                                   // 全局静态标志（:73）
    d dVar = MyFiilReceiver.f34272h;
    d.f34410a.b(UUID.fromString("d52b47bc-2dea-42c4-b26d-20252fceeeb3")); // ★ 换 SPP UUID
    this.f34446g.a(this.sdkService, str);                            // ★ 直接调用，不走反射
}
```

对比 `connecting(String)`（`:275`）就能看清分叉点：

| | `connecting()` (:275) | `FIILCCPro2Connecting()` (:259) | `quickConnect()` (:1036) |
|---|---|---|---|
| `isF039` | `false` | **`true`** | `false` |
| SPP UUID | `00001101-…`（标准 SPP） | **`d52b47bc-…`**（Bluetrum 私有） | `00001101-…` |
| 调用方式 | 反射 `blueConn(d, String)` | 直接 `a(service, str)` | 反射 `blueConnQuick()` |

**这里有个容易看漏的点**：`d52b47bc-2dea-42c4-b26d-20252fceeeb3` 这个 UUID 在包里出现了**两次、两个方向**——

- `BluetoothSppService.java:35`：手机 `listenUsingInsecureRfcommWithServiceRecord(...)` **当服务端**；
- `FiilManager.java:266`：手机 `createRfcommSocketToServiceRecord(...)` **当客户端**。

同一个 UUID、同一个进程、两个方向。老机型走标准 SPP（`0x1101`）当客户端；CC Pro2 则改用这个私有 UUID 主动去连耳机——**传输层借了 Bluetrum 那套私有通道号，上层跑的却仍是 GAIA 帧**（`d`→`b` 那个 `GaiaLink` 对象没变）。所以判断"某机型走哪条链路"不能只看 UUID，要看它调用的是 `connecting()` 还是 `FIILCCPro2Connecting()`。

`isF039`（`public static boolean`，`FiilManager.java:73`）全包只有 6 处出现，其中唯一读取点是握手里的
`if (!FiilManager.isF039) { FiilManager.getInstance().getVersion(null); }`（`com/fiil/sdk/c/a.java:132`）——
即 **earType=39（CC Pro2）这一类设备在握手时会跳过 GAIA 版本查询，直接去读 earType**。
这是逆向时判断机型代数的副作用信号；至于设备本身是否不响应版本查询，静态代码看不出来，只能确定 App 主动不问。

#### C. Airoha 双通道

`com/airoha/liblinker/constant/a.java`（内部名 `UuidTable`，整个 Airoha 协议栈共用的 UUID 表）：

```java
public static final UUID f10186a = UUID.fromString("00002902-0000-1000-8000-00805F9B34FB"); // CCCD 通知开关
public static final UUID f10187b = UUID.fromString("00000000-0000-0000-0099-AABBCCDDEEFF"); // 私有服务
public static final UUID f10188c = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB"); // 标准 SPP
public static final UUID f10189d = UUID.fromString("5052494D-2DAB-0341-6972-6F6861424C45"); // "PRIM" 主服务
public static final UUID f10190e = UUID.fromString("43484152-2DAB-3241-6972-6F6861424C45"); // "CHAR"-2A
public static final UUID f10191f = UUID.fromString("43484152-2DAB-3141-6972-6F6861424C45"); // "CHAR"-1A
public static final UUID f10192g = UUID.fromString("49535343-FE7D-4AE5-8FA9-9FAFD205E455"); // 透传服务（微芯 BM7x）
public static final UUID f10193h = UUID.fromString("49535343-8841-43F4-A8D4-ECBE34729BB3"); // 透传写
public static final UUID f10194i = UUID.fromString("49535343-1E4D-4BD9-BA61-23C647249616"); // 透传通知
```

UUID 里的 ASCII 很有意思：`PRIM` = primary（主服务），`CHAR` = characteristic（特征），把语义直接编进了 128 位 UUID。

连接入口：`com/fengeek/main/f042/fragment/F042MainFragment.java:363 connectAirohaSDK()`，先取 `com.fengeek.main.ble.c.getInstance().getmBLEDevice()`，再判断 `F042HomeFragment` 是否已就绪，然后 `DeviceControl.connect(...)`。

#### D. F042 BLE 私有（新机型）

`com/fengeek/main/f042/fragment/F042HomeFragment.java:62-68`：

```java
public static final String f28317d = "13fa60a2-429a-4a90-a3e8-d4c2b9a17655";  // Service
public static final String f28318e = "3d4dac58-7d68-4734-8a71-21bc659df11c";  // Write 特征
public static final String f28319f = "3d4dac58-7d68-4734-8a71-21bc659df11c";  // Notify 特征（与写同一个）
```

注意 `f28318e` 和 `f28319f` **是同一个 UUID** —— 这个特征同时支持 Write 和 Notify，收发共用一个特征值（这是很多国产 BLE 方案的省事做法）。

```java
// 建链（F042HomeFragment.java:1619）
FIILPeripheral.getInstance().connectGatt(getContext(), bleDevice,
        f28317d, "3d4dac58-7d68-4734-8a71-21bc659df11c", new x());

// 订阅通知（F042HomeFragment.java:1215）
FIILPeripheral.getInstance().notify("3d4dac58-7d68-4734-8a71-21bc659df11c", new C0254a());
```

`FIILPeripheral.connectGatt(Context, DeviceWrapper, serviceUUID, writeCharUUID, callback)` 内部会 `discoverServices` → 找服务 → 找特征 → `writeDescriptor(CCCD)` 打开 notify，然后才能收发。

---

## 3. 协议帧格式（核心）

### 3.1 GAIA 帧（链路 A）—— 最重要

构造函数 `FiilCommandUtil.getSendBuffer(int vendor, int cmd, boolean newProto, byte... payload)`
（`com/fiil/sdk/command/FiilCommandUtil.java:517`）：

```java
int off = getGaiaIntValue("OFFS_PAYLOAD");          // = 8
byte[] out = new byte[payload.length + off];
out[0] = 0xFF;                                       // SOF
out[1] = newProto ? (byte)0xF1 : 0x01;               // 协议版本
out[2] = 0;                                          // FLAGS
out[3] = (byte) payload.length;                      // PAYLOAD_LENGTH
out[4] = (byte)(vendor >> 8);                        // VENDOR_ID_H
out[5] = (byte) vendor;                              // VENDOR_ID_L
out[6] = (byte)(cmd >> 8);                           // COMMAND_ID_H
out[7] = (byte) cmd;                                 // COMMAND_ID_L
// out[8..] = payload
```

常量全部来自 `com/fiil/sdk/gaia/father/Gaia.java`（通过 `ReflectUtils` 反射读取，说明作者故意做了轻度防逆向）：

| 常量 | 值 | 含义 |
|---|---|---|
| `SOF` | `-1` = 0xFF | 帧起始 |
| `PROTOCOL_VERSION` | `1` | 老协议版本字节 |
| `NEW_PROTOCOL_VERSION` | `241` = 0xF1 | 新协议版本字节 |
| `DEFAULT_FLAGS` | `0` | 默认标志 |
| `FLAG_CHECK` | `1` | 置位则帧尾带 1 字节校验 |
| `OFFS_SOF` | 0 | |
| `OFFS_VERSION` | 1 | |
| `OFFS_FLAGS` | 2 | |
| `OFFS_PAYLOAD_LENGTH` | 3 | |
| `OFFS_VENDOR_ID` / `_H` / `_L` | 4 / 4 / 5 | |
| `OFFS_COMMAND_ID` / `_H` / `_L` | 6 / 6 / 7 | |
| `OFFS_PAYLOAD` | 8 | 载荷起点 |
| `VENDOR_FF` | `564` = 0x0234 | FIIL 厂商 ID |
| `COMMAND_MASK` | 32767 = 0x7FFF | 取低 15 位得真实命令号 |
| `ACK_MASK` | 32768 = 0x8000 | 置位表示这是"响应/ACK" |

**所以最小帧长 = 8 字节**（无 payload），长度 = `8 + payloadLen (+1 if FLAG_CHECK)`。

`Gaia.frame(...)`（`Gaia.java:456`）/ `Gaia.newFrame(...)`（`Gaia.java:544`）是同一套帧的另一份实现——两者唯一的差别是**版本字节**：`frame()` 硬写 `bArr2[1] = 1`，`newFrame()` 用 `bArr2[1] = i.f9426g`（即新协议版本）。其余完全一致，包括：

```java
if (i4 > 254) throw new IOException("GAIA frame full");   // 长度上限 254（1 字节长度字段）
int i7 = (b2 & 1) != 0 ? 1 : 0;                            // flags bit0 = 带校验
int i8 = i4 + 8 + i7;                                      // 总长
...
if (i7 != 0) {                                             // 带校验时：前面所有字节 XOR 后追加到帧尾
    byte b3 = 0;
    for (int i6 = 0; i6 < i8 - 1; i6++) b3 ^= bArr2[i6];
    bArr2[i8 - 1] = b3;
}
```

注意 `frame()` 里 `bArr2[2] = b2`（flags 直接用入参），而 `getSendBuffer()` 里 `out[2] = 0`（永远是 0）。所以**两条路径的 flags 处理不同**：走 `Gaia.frame(...)` 才能开校验位。

**帧示例**（读 ANC 当前值，老款）：

```
FF 01 00 00 02 34 02 8A
│  │  │  │  └┬─┘ └┬─┘
│  │  │  │   │    └─ cmd = 0x028A = 650 = FFCOMMAND_GET_MAF（老款用 MAF 表示 ANC）
│  │  │  │   └────── vendor = 0x0234 = 564
│  │  │  └────────── payload 长度 0
│  │  └───────────── flags 0
│  └──────────────── version 1（新协议用 F1）
└─────────────────── SOF
```

**解析回包**（`com.fiil.sdk.gaia.father.a`）：

```
vendor   = u16(frame[4..5])
cmdRaw   = u16(frame[6..7])
isAck    = (cmdRaw & 0x8000) != 0
cmd      = cmdRaw & 0x7FFF
payload  = frame[8 .. 8+len]
```

拿到 `cmd` 后，`com.fiil.sdk.command.a`（`FiilBaseCommand`）用 `Config.sMap`（`com/fiil/sdk/config/Config.java:155` 声明，`:185` 起逐条 `put`，共 **145 条** `commandId → "analysisXxx"` 映射）**反射调用**对应的 `analysisXxx(byte[])` 做业务解析。这就是整套 SDK 的分发中枢。

> 注意区分两个数字：**190** 是 `Gaia.java` 里定义的 `FFCOMMAND_*` 常量总数；**145** 是 `Config.sMap` 里真正挂了处理方法的条数。剩下那 45 条只定义未处理（或由别的模块直接比较数值处理），逆向时如果按 `sMap` 找会发现"少了一些命令"，原因就在这里。

### 3.2 Bluetrum 帧（链路 B）

组包器 `com/bluetrum/devicemanager/z.java`（内部名 `RequestHandler`），方法 `b(Request)` @L36：

```
[seq:1][command:1][commandType:1][packInfo:1][len:1][payload:len]
```

- `seq`：**低 4 位**滚动计数，每发一个分包 `this.f16473e = (byte)((this.f16473e + 1) & 15)`（`z.java:49`、`:64`），复位用 `c()` @L70。注意**每个分包各自占一个 seq**，不是"一条 Request 一个 seq"；
- `packInfo`：`(byte)((((iB-1) << 4) & 0xF0) | (i2 & 15))`（`z.java:55`，掩码常量 `com/bluetrum/fota/i.java:14` → `f16511c = 240` = 0xF0）—— 高 4 位"总包数-1"，低 4 位"第几包"；
- 空 payload 走特判（`z.java:41-49`）：直接 `ByteBuffer.allocate(5)`，`packInfo=0`、`len=0`，帧长固定 5；
- 分包器 `a0`（`com/bluetrum/devicemanager/a0.java`，内部名 `RequestSplitter`）默认 `new a0(15)`（`z.java:29`）—— 每包净载荷 15 字节；`d(int maxPacketSize)` @L74 里 `this.f16472d.d(maxPacketSize - 5)` 会把 5 字节头扣掉，所以协商 MTU 20 时净载荷正好 15。`b(int payloadLength)` 返回 `(payloadLength + i2 - 1) / i2`，即向上取整的总包数。

命令类型常量：`com.bluetrum.devicemanager.cmd.Request`，`commandType` 区分 GET/SET/NOTIFY/RESPONSE。

### 3.3 F042 BLE 帧（链路 D）

组包在 `com/fengeek/main/ble/FIILPeripheral.java:444 write(BaseCommand, Handler)`：

```java
arrayList.add((byte) -1);                                   // 0xFF
arrayList.add((byte) 1);                                    // version 1
arrayList.add((byte) 0);                                    // flags 0
arrayList.add((byte) (payload.length + 2));                 // payload 长度（+2 = cmd 字节 + len 字节）
arrayList.add((byte) 2);                                    // vendor H
arrayList.add((byte) 52);                                   // vendor L → 0x0234
arrayList.add((byte) ((cid >> 8) & 255));                   // commandClassifyId 高字节
arrayList.add((byte) (cid & 255));                          // commandClassifyId 低字节
arrayList.add((byte) cmd);                                  // commandId（1 字节）
arrayList.add((byte) payload.length);                       // payload 长度（1 字节）
// payload...
```

即：

```
FF 01 00 (payloadLen+2) 02 34 CID_H CID_L CMD LEN payload...
```

**注意**：它和 GAIA 帧长得很像（同样 `FF .. 02 34`），但语义不同——GAIA 的 6/7 位是 2 字节 commandId；F042 的 6/7 位是 2 字节 **commandClassifyId**，第 8 位才是 commandId，第 9 位再来一个 payload 长度。写解析器时**别把两条链路混为一谈**。

### 3.4 豆包 0x55AA 帧（语音业务）

`com/fengeek/main/i/d/e.java` 的 `setDouBao(byte[] data, listener)`：

```java
arrayList.add((byte) 85);                    // 0x55
arrayList.add((byte) -86);                   // 0xAA
arrayList.add((byte) 0);                     // 类型/序号
arrayList.add((byte) (len & 255));           // 长度 低字节
arrayList.add((byte) ((len >> 8) & 255));    // 长度 高字节
// data...
bArr[arrayList.size()]     = -91;            // 0xA5
bArr[arrayList.size() + 1] = 90;             // 0x5A
```

```
55 AA 00 LEN_LO LEN_HI payload... A5 5A
```

**收包方向**（`com/fengeek/main/i/b.java` 的 `c(byte[])`）逻辑是状态机：

- 若 `data[0]==0x55 && data[1]==0xAA` → 这是**"发给豆包"**的帧（`isApp=false`），长度取 `d(new byte[]{data[4], data[3]})`。`d()`（`com/fengeek/main/i/b.java:145`）是 `(acc<<8)|byte` 的累加，所以传 `{data[4], data[3]}` 进去算出来的是 **`data[4]<<8 | data[3]`**，即**小端 16 位**（低字节在前）；切段 `copyOfRange(data, i3+5, i6)`，之后游标 `i3 = i6 + 2`（跳过尾部 `A5 5A`）；
- 否则 → 这是**"发给 App"**的普通帧（`isApp=true`），长度取 `data[4]`（单字节），按 `i4 + i2 + 5` 算段尾，游标 `i3 = i5`（不额外跳）；
- 外层是 `while(true)`，每轮重新取 5 字节头重判方向，直到剩余不足 5 字节才 `return`（日志 "组包数据处理完毕"）。

```java
// com/fengeek/main/i/b.java:91  c(byte[] data)  —— 收包状态机（逐行核对后的等价伪码）
boolean isApp = !((data[0]&255) == 85 && (data[1]&255) == 170);
byte[] head = copyOfRange(data, 0, 5);
int segStart = 0, consumed = 0;
for (;;) {
    if (isApp) {                                   // 普通帧：头 5 字节 + data[4] 长度
        int len = head[4] & 255;
        int end = consumed + len + 5;
        if (end > data.length) return;             // "App数据错误"
        dispatch(copyOfRange(data, segStart, end));
        segStart = end; consumed = end;
    } else {                                       // 豆包帧：55 AA 00 len_lo len_hi + 尾部 A5 5A
        int len = d(new byte[]{head[4], head[3]}); // = data[4]<<8 | data[3]
        int end = consumed + len + 5;
        if (end > data.length) return;             // "豆包数据错误"
        dispatch(copyOfRange(data, segStart + 5, end));
        segStart = end + 2; consumed = end;        // +2 跳过 A5 5A
    }
    if (segStart + 5 > data.length) return;        // "组包数据处理完毕"
    head = copyOfRange(data, segStart, segStart + 5);
    isApp = !((head[0]&255) == 85 && (head[1]&255) == 170);
}
```

`setDouBao` 写长度用 `len&255`→索引 3、`len>>8`→索引 4，`d()` 又把 `data[4]<<8|data[3]` 拼回来，一写一读正好对上，是**小端**的。

这段代码还写死了：**只有 earType == 47 或 48 的机型**才走这条拆包逻辑（`com/fengeek/main/i/b.java:187` → `if (this.f31855k == 48 || this.f31855k == 47)`），其它机型直接把原始字节 `obtainMessage(1, ...)` 丢给上层。

### 3.5 Airoha Race 包

`com/airoha/android/lib/e/b/a.java`（`RacePacket`）：`SOF=5`、类型=6、总长字段……整套是 Airoha 官方的 RACE 协议封装，业务层在 `com.airoha.libmmi`（MMI = 设备交互消息）与 `com.airoha.libbase.RaceCommand`（AppId / Recipient / RaceId / RaceType 四元组路由）。

---

## 4. 命令集：190 条 FFCOMMAND

全部常量集中在 `com/fiil/sdk/gaia/father/Gaia.java`（`static final int FFCOMMAND_xxx`），完整清单见附带的 `_cmd_table.txt`（190 行，`名字|十进制值`）。这里按功能归类挑重点：

### 4.1 连接 / 信息

| 命令 | 值(hex) | 说明 |
|---|---|---|
| `FFCOMMAND_GET_EARTYPE` | 777 (0x0309) | **取机型号**，决定 UI 与命令集 |
| `FFCOMMAND_GET_VERSION` | 769 (0x0301) | 固件版本 |
| `FFCOMMAND_GetTWSVersion` | 1819 (0x071B) | TWS 版本 |
| `FFCOMMAND_GET_ADDRESS` | 770 (0x0302) | 设备地址 |
| `FFCOMMAND_GET_SEQCODE` | 772 (0x0304) | 序列号 |
| `FFCOMMAND_GET_STATUS` | 16394 (0x400A) | 连接状态 |
| `FFCOMMAND_GET_BATTERY_LEVEL` | 771 (0x0303) | 电量（主） |
| `FFCOMMAND_GetBatteryLevel` | 1800 (0x0708) | 电量（另一套，新机型） |
| `FFCOMMAND_GetBoxBatter` | 1915 (0x077B) | 充电盒电量 |
| `FFCOMMAND_GET_BATTERY_CHARGE` | 774 (0x0306) | 是否在充电 |
| `FFCOMMAND_GET_USETIME` | 773 (0x0305) | 累计使用时长 |

### 4.2 降噪（ANC / MAF）

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_ANC` | 641 (0x0281) | 读 ANC（F002 专用类） |
| `FFCOMMAND_SET_ANC` | 513 (0x0201) | 写 ANC（F002 专用类） |
| `FFCOMMAND_GET_MAF` | 650 (0x028A) | 读 MAF（老枚举名，实际被 `getAnc()` 用） |
| `FFCOMMAND_SET_MAF` | 522 (0x020A) | 写 MAF（老枚举名，实际被 `setAnc()` 用） |
| `FFCOMMAND_GetMAF` | 1823 (0x071F) | 读 MAF（新机型，被 `getMaf()` 用） |
| `FFCOMMAND_SetMAF` | 1824 (0x0720) | 写 MAF（新机型，被 `setMaf()` 用） |

**坑点 1：命令名和 API 名是错位的。** `Gaia.java` 里同时存在 `FFCOMMAND_GET_MAF`(650) 和 `FFCOMMAND_GetMAF`(1823) —— 只差一个字母大小写，是两组完全不同的命令。而 `FiilCommandUtil` 的对应关系是交叉的：

| API | 实际下发的命令 | 值 |
|---|---|---|
| `setAnc(int)` @L759 | `FFCOMMAND_SET_MAF` | 522 |
| `getAnc()` @L102 | `FFCOMMAND_GET_MAF` | 650 |
| `setMaf(int)` @L1030 | `FFCOMMAND_SetMAF` | 1824 |
| `getMaf()` @L403 | `FFCOMMAND_GetMAF` | 1823 |

也就是说 **`setAnc()` 发的并不是 ANC 命令，而是 MAF 命令**；真正发 `FFCOMMAND_SET_ANC`(513)/`GET_ANC`(641) 的是另一套类 `FiilCommandF002`（`com/fiil/sdk/command/b.java`）。两套并存，按机型选择：老机型走 650/522，CC Pro2 等新机型走 1823/1824，F002 专用页走 641/513。

**坑点 2：ANC 和 MAF 在耳机端是同一个东西。** 从 UI 看，"降噪档位"和"音效模式（MAF = Mode Adjustment Function）"是两个面板，但底层都落在同一组命令上，payload 都只有 1 字节。

档位语义（`adjust_zao_fragment.setZaoInfo(boolean)`，`com/fengeek/main/heat_info_fragment/adjust_zao_fragment.java:509`，逐档核对）：

| 值 | 含义 | 代码行为 |
|---|---|---|
| 0 | 关闭 | `k0=1`；收起面板 `closeItem()`；禁用 `U` 控件；埋点 `X("20923","关")`（earType 8）/ `X("21023","关")`（earType 5） |
| 1 | 降噪（ANC） | 选中 `T`；禁用 `S`；埋点 `"降噪"` |
| 2 | 监听 / 通透 | 选中 `T`；禁用 `R`；埋点 `"监听"` |
| 3 | 开放 | 选中 `T`；禁用 `Q`；埋点 `"开放"` |
| 4 | 风中（抗风噪） | 选中 `T`；禁用其它；埋点 `"风中"` |

代码里 `anc` 来源：`FiilManager.getInstance().getDeviceInfo().getAnc()`；若未连上或 `getEarType() != this.a1`（当前 Fragment 绑定的机型号不匹配）则回退到保存值 `p0`，避免把别的机型的状态画到当前页上。不同机型可用档位不同（老款只有 0/1/2，CC Pro2 有 4 档），UI 会按 `earType` 禁用不支持的按钮。

### 4.3 EQ / 音效

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_EQ` | 643 (0x0283) | 读 EQ（通用） |
| `FFCOMMAND_SET_EQ` | 515 (0x0203) | 写 EQ（通用，1 字节索引） |
| `FFCOMMAND_GetEQ` | 1803 (0x070B) | 读 EQ（新机型） |
| `FFCOMMAND_SetEQ` | 1804 (0x070C) | 写 EQ（新机型，1 字节） |
| `FFCOMMAND_GET_3D` / `SET_3D` | 644 / 516 | 3D 音效开关 |
| `FFCOMMAND_GET_MAF` / `SET_MAF` | 650 / 522 | 音效模式（与 ANC 复用） |
| `FFCOMMAND_GetDSP` / `SetDSP` | 1811 / 1812 | DSP 预设 |

写 EQ 就是 `payload = [索引]`；索引表由 App 端 `Style` 常量决定（0=默认/原声，1..N 对应 App 内预设列表顺序）。**耳机端不做名称映射**，只认数字。

### 4.4 音量 / 播放

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_VOLUME` | 390 | 读音量 |
| `FFCOMMAND_SET_VOLUME` | 262 | 写音量，payload `[1]`=加 / `[2]`=减 |
| `FFCOMMAND_UP_VOLUME` / `DOWN_VOLUME` | 517 / 518 | 加 / 减 |
| `FFCOMMAND_SET_VOLUME_UP` / `_DOWM` | 8197 / 8198 | 绝对音量（新机型，带值） |
| `FFCOMMAND_PALY` / `PAUSE` | 8193 / 8194 | 播放 / 暂停 |
| `FFCOMMAND_NEXT` / `PREVIOUSE` | 8195 / 8196 | 上/下一曲 |
| `FFCOMMAND_SET_PLAY_MODE` / `GET_PLAY_MODE` | 8199 / 8327 | 循环模式 |
| `FFCOMMAND_GetPlayStatus` / `SetPlayStatus` | 1945 / 1946 | 播放状态 |

### 4.5 触控 / 按键自定义

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GetTouchSensitivity` / `SetTouchSensitivity` | 1919 / 1918 | 触控灵敏度 |
| `FFCOMMAND_GetCustomButtonFunction` / `Set...` | 1922 / 1921 | 单击/双击/三击功能 |
| `FFCOMMAND_GetCustomButtonFourHitFunction` / `Set...` | 1936 / 1935 | 四击功能 |
| `FFCOMMAND_GetCustomBtn` / `SetCustomBtn` | 1963 / 1962 | 自定义按键 |
| `FFCOMMAND_GetPressCustom` / `SetPressCustom` | 1965 / 1964 | 长按自定义 |
| `FFCOMMAND_GetBeep` / `SetBeep` | 1973 / 1972 | 提示音开关 |

灵敏度 payload 实现（`setTouchSensitivity`，`FiilCommandUtil.java:1274-1293`）：**两字节 = `[组别, 阈值]`**，共 6 个合法取值，其余值一律回落成 `{1,37}`：

| 入参 | payload | 入参 | payload |
|---|---|---|---|
| 0 | `1, 44` | 3 | `0, 44` |
| 1 | `1, 37` | 4 | `0, 37` |
| 2 | `1, 31` | 5 | `0, 31` |

其中 `31` 来自混淆后的常量 `com.google.firebase.crashlytics.buildtools.reloc.com.google.common.base.a.I`（`a.java:17` → `public static final byte I = 31;`）。第一字节把 6 档分成两组（0/1，很可能是左右耳或"灵敏度方向"），第二字节才是三档阈值 `44 / 37 / 31`。UI 上给的是 5 档滑条，与实际下发的 6 值表并不严格一一对应。

### 4.6 佩戴检测 / 省电

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_WEAR` / `SET_WEAR` | 391 / 263 | 佩戴检测开关 |
| `FFCOMMAND_EBABLE_WEAR_DETECT` / `DIABLE_WEAR_DETECT` | 12294 / 12295 | 另一种穿戴检测 |
| `FFCOMMAND_GetWearingState` | 1912 | 当前是否佩戴 |
| `FFCOMMAND_GetWearingSwitch` / `Set...` | 1917 / 1916 | 佩戴开关（新） |
| `FFCOMMAND_GetWearingSensitivity` / `Set...` | 1914 / 1913 | 佩戴灵敏度 |
| `FFCOMMAND_GetPowerSavingStatus` / `Set...` | 1832 / 1833 | 省电模式 |
| `FFCOMMAND_GetSleepMode` / `SetSleepMode` | 1975 / 1974 | 休眠 |
| `FFCOMMAND_GET_SHUTDOWNTIME` / `SET_SHUTDOWNTIME` | 387 / 259 | 自动关机时间 |
| `FFCOMMAND_GetSimpleMode` / `SetSimpleMode` | 1808 / 1809 | 简易模式 |

### 4.7 多设备 / 编解码

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_MULTIPOINT` / `SET_MULTIPOINT` | 386 / 258 | 一拖二开关 |
| `FFCOMMAND_GetMultipointConnection` / `Set...` | 1953 / 1952 | 多点连接（新） |
| `FFCOMMAND_GetPairCount` | 1956 | 已配对数量 |
| `FFCOMMAND_SetPair` | 1889 | 主动配对 |
| `FFCOMMAND_SetMutultipointPair` | 1978 | 指定多点配对 |
| `FFCOMMAND_SetDisconnectDevice` | 1977 | 断开某台设备 |
| `FFCOMMAND_SetExchangeA2DP` / `SetTwoExchange` | 1976 / 1957 | 音频通道切换 |
| `FFCOMMAND_GET_APTX` / `SET_APTX` | 392 / 264 | aptX 开关 |
| `FFCOMMAND_GetLDACSwitchType` / `SetLDACSwitchType` | 1971 / 1970 | LDAC 开关 |
| `FFCOMMAND_GetLDACBuyType` / `SetUnlockLADC` | 1967 / 1968 | LDAC 授权解锁 |
| `FFCOMMAND_GetLowLatency` / `SetLowLatency` | 1825 / 1826 | 低延迟（游戏模式） |
| `FFCOMMAND_GetCodingAndDecodingType` | 1954 | 当前编解码 |

**LDAC 是个收费点**：`GetLDACBuyType`（1967）读"是否已购买"，`SetUnlockLADC`（1968，拼写按源码）下发解锁。这说明 LDAC 是**按机型/按账号付费解锁**的，不是纯开关。

### 4.8 语音 / 提示音

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_LANGUAGE` / `SET_LANGUAGE` | 388 / 260 | 语音提示语言 |
| `FFCOMMAND_NEW_GET_LANGUAGE` / `NEW_SET_LANGUAGE` | 4102 / 4103 | 新语言协议 |
| `FFCOMMAND_LANGUAGE_HINT` | 775 | 语言能力探测 |
| `FFCOMMAND_SET_VOIDE_CONTENT` / `GET_VOIDE_CONTENT` | 271 / 399 | 自定义语音内容 |
| `FFCOMMAND_SET_VOICE_DIAL_MODE` / `GET_..._STATE` | 272 / 400 | 语音唤醒/拨号 |
| `FFCOMMAND_PLAY_VOIDE_HINT` | 12305 | 播放指定提示音 |
| `FFCOMMAND_GetNewLanguage` / `SetNewLanguage` | 1815 / 1816 | 新语言枚举 |

### 4.9 查找 / 重置

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_SET_SEARCH` / `GET_SEARCH` | 265 / 393 | 查找耳机（响铃） |
| `FFCOMMAND_SET_RESET` / `SetReset` | 521 / 1818 | 恢复出厂 |
| `FFCOMMAND_RESTORE_FACTORY_SETTING` | 16398 | 恢复出厂（另一套） |
| `FFCOMMAND_SetNewReset` | 1818 | 重置（共用值） |

### 4.10 运动 / 健康（Runner 等）

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_SET_SPORT_START` / `STOP` | 12289 / 12290 | 开始/结束运动 |
| `FFCOMMAND_GET_TOTAL_STEP` | 12301 | 总步数 |
| `FFCOMMAND_GET_ONE_WAY_STEP` | 12302 | 单程步数 |
| `FFCOMMAND_GET_TOTAL_DETAIL_STEP` | 12304 | 分时段步数 |
| `FFCOMMAND_GET_HISTORICAL_DATA` | 12300 | 历史数据 |
| `FFCOMMAND_SET_TOTAL_ACTIVITY_GOAL` / `GET_...` | 12298 / 12426 | 运动目标 |
| `FFCOMMAND_GET_ACTIVITY_STATUS` | 12312 | 运动状态 |
| `FFCOMMAND_SET_ACTIVITY_MODE` / `GET_ACTIVITY_MODE` | 12293 / 12421 | 运动模式 |
| `FFCOMMAND_ACTIVITY_DATA` | 12296 | 运动数据推送 |
| `FFCOMMAND_BURN_IN_STATUS` | 16410 | 煲机状态 |
| `FFCOMMAND_START_TRIAL_MODE` / `END_TRIAL_MODE` | 12308 / 12309 | 试用模式（电商/线下试听） |

### 4.11 灯效 / 显示

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_LED_COLOR` / `SET_LED_COLOR` | 397 / 269 | RGB 灯色 |
| `FFCOMMAND_CONFIG_GET_FIIL_LED_MODE` / `SWITCH_...` | 395 / 267 | 灯效模式 |
| `FFCOMMAND_GET_BRIGHT` / `SET_BRIGHT` | 389 / 261 | 亮度 |
| `FFCOMMAND_GET_NECKLACE_MODE` / `SET_NECKLACE_MODE` | 398 / 270 | 项链模式（颈挂式） |

### 4.12 OTA / 文件传输

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_START_OTA_UPDATE` | 16397 (0x400D) | 开始升级 |
| `FFCOMMAND_OPEN_FILE` / `CLOSE_FILE` | 16385 / 16388 | 打开/关闭文件 |
| `FFCOMMAND_WRITE_FILE` | 16386 | 写文件（分片） |
| `FFCOMMAND_READ_FILE` | 16387 | 读文件 |
| `FFCOMMAND_DELETE_FILE` / `RESET_FILE` / `RESUME_FILE` | 16389 / 16391 / 16390 | 删除/重置/续传 |
| `FFCOMMAND_GET_DISK_SPACE` | 16399 | 剩余空间 |
| `FFCOMMAND_ENTER_SYNC_MODE` / `EXIT_SYNC_MODE` | 16392 / 16393 | 同步模式 |
| `FFCOMMAND_EVENT_NOTIFICATION` | 20484 | 事件通知 |
| `FFCOMMAND_REGISTER_NOTIFICATION` | 20481 | 订阅通知 |
| `FFCOMMAND_GET_NOTIFICATION` | 20482 | 读通知 |
| `FFCOMMAND_CANCEL_NOTIFICATION` | 20483 | 取消通知 |

GAIA 原生还有一套 DFU 命令（`COMMAND_DFU_BEGIN=1585` / `DFU_WRITE=1586` / `DFU_COMMIT=1587` / `DFU_REQUEST=1584`），以及存储分区命令（`COMMAND_OPEN_STORAGE_PARTITION=1553`、`COMMAND_WRITE_STORAGE_PARTITION=1557`、`COMMAND_MOUNT_STORAGE_PARTITION=1562`），Airoha 侧另外有 `com.airoha.libfota155x` 独立 OTA 栈。

### 4.13 用户信息 / 云端

| 命令 | 值 | 说明 |
|---|---|---|
| `FFCOMMAND_GET_DEVICE_ID` / `GET_USER_ID` | 4097 / 4226 | 设备/用户 ID |
| `FFCOMMAND_GET_USERINFO` / `SET_USERINFO` | 4229 / 4101 | 用户资料 |
| `FFCOMMAND_GET_USER_SECURITY_CODE` / `SET_...` | 4108 / 4109 | 安全码 |
| `FFCOMMAND_SET_TIME` / `SET_TIME2` / `NEW_SET_TIME` | 4104 / 4105 / 4110 | 对时 |
| `FFCOMMAND_GET_CUSTOM_DEVICE_NAME` | 4100 | 自定义设备名 |
| `FFCOMMAND_GET_DEVICE_SETTING` / `SET_DEVICE_SETTING` | 4106 / 4107 | 设备设置包 |
| `FFCOMMAND_GET_CONFIG_SDS` / `SET_CONFIG_SDS` | 396 / 268 | SDS 配置 |
| `FFCOMMAND_NETWORK_ALERT` | 16402 | 网络提醒 |
| `FFCOMMAND_VOICE_RECOGNITI` | 16403 | 语音识别 |
| `FFCOMMAND_VOIDE_PROMOTE` | 16421 | 语音提示 |

---

## 5. 机型对照表（earType → 型号）

`MainActivity.java` 私有方法 `private void x(Bundle bundle)`（第 1902 行起）与 `Config.java` 里的 `HeatSetMode` 枚举一一对应。**耳机上报的 earType 决定整个 UI 形态**。

| earType | 枚举 | 型号（strings.xml 实际文案） |
|---|---|---|
| 1 | `FIIL` | Music |
| 2 | `FIIL_WIRELESS` | FIIL Wireless |
| 3 | `FIIL_BESTIE` | Music |
| 4 | `DIVA_WIRED` | FIIL Diva Wired |
| 5 | `DIVA_PRO` | FIIL Diva Pro |
| 6 | `FIIL_CARAT` | FIIL Carat |
| 7 | `FIIL_CARAT_PRO` | FIIL Carat M |
| 8 | `FIIL_DIVA` | FIIL Diva |
| 9 | `FIIL_VOX` | FIIL VOX |
| 10 | `FIIL_DIFFERT_NC` | Music |
| 11 | `FIIL_DIFFERT_PRO` | FIIL Driifter Pro |
| 15 | `FIIL_CARAT_LITE` | Music |
| 16 | `FIIL_DIFFERT` | Music |
| 17 | `FIIL_T1` | Music |
| 18 | `FIIL_DIFFERT_DNC` | FIIL Driifter DNC |
| 19 | `FIIL_DIFFERT_DNC_PLUS` | FIIL Driifter DNC Plus |
| 20 | `FIIL_RUNNER` | FIIL Runner |
| 21 | `FIIL_CC` | FIIL CC |
| 22 | `FIIL_T1_X` | FIIL T1 X |
| 26 | `FIIL_T1_XS` | FIIL T1 XS |
| 27 | `FIIL_T1_Pro` | FIIL T1 Pro |
| 28 | `FIIL_CC_PRO` | FIIL CC Pro |
| 29 | `FIIL_T1_Lite` | FIIL T1 Lite |
| 30 | `FIIL_CC2` | FIIL CC2 |
| 31 | `FIIL_CG` | FIIL CG |
| 33 | `FIIL_CG_PRO` | FIIL CG Pro |
| 35 | `FIIL_T2_PRO` | FIIL T2 Pro |
| 37 | `FIIL_CC_nano` | FIIL CC nano |
| 38 | `FIIL_Key` | FIIL Key |
| 39 | `FIIL_CC_Pro2` | **FIIL CC Pro2** |
| 40 | `FIIL_Belt` | FIIL Belt |
| 41 | `FIIL_Key_Pro` | FIIL Key Pro |
| 42 | `FIIL_BOX` | FIIL Aviator |
| 43 | `FIIL_GS` | FIIL GS |
| 45 | `FIIL_F045` | FIIL GS Lite |
| 46 | `FIIL_F046` | FIIL Key Max |
| 47 | `FIIL_F047` | FIIL Atom |
| 48 | `FIIL_F048` | FIIL GS Links |
| 49 | `FIIL_F049` | FIIL Key Pro2 |
| 51 | `FIIL_F051` | FIIL CC3 |
| 53 | `FIIL_F053` | FIIL CC Pro3 |
| 247 | `FIIL_DIVA2` | FIIL Diva2 |
| 250 | `DIVA_PRO2` | FIIL Diva2 Pro |

> 注意 47/48（Atom / GS Links）这两个机型，就是**唯一启用豆包 0x55AA 语音帧**的机型。

---

## 6. 功能全景：App 到底能控制耳机什么

按 UI 模块整理（来自 `res/values/strings.xml` 中文文案 + 各 Fragment）：

1. **连接管理**：扫描、配对、下拉重连、断开、多设备切换、电量/充电状态、序列号、固件版本。
2. **降噪**：关 / 降噪 / 通透（监听）/ 开放 / 风中，5 档，部分机型带"自适应"。
3. **EQ 音效**：预设（原声/流行/摇滚/古典/人声/低音增强…）、自定义 EQ（多段）+ 3D 音效、DSP 预设。
4. **触控设置**：单击/双击/三击/四击/长按分别绑定（播放暂停、上下一曲、音量加减、语音助手、ANC 切换、无操作），触控灵敏度 5 档。
5. **佩戴检测**：开/关、摘下自动暂停、佩戴灵敏度。
6. **音量**：耳机侧音量加减、绝对音量、音量提示。
7. **低延迟 / 游戏模式**：`GetLowLatency` 开关。
8. **编解码**：aptX、LDAC（含"购买解锁"）、AAC/SBC 显示、连接类型。
9. **多点连接**：一拖二开关、已配对列表、指定断开、A2DP 通道互换。
10. **查找耳机**：让耳机响铃，左右耳分别响。
11. **提示音 / 语音**：语言（中/英…）、提示音开关、自定义语音内容、语音唤醒/拨号。
12. **灯效**（带灯机型）：颜色、模式、亮度、项链模式。
13. **自动关机**：15/30/60 分钟/永不。
14. **省电 / 休眠 / 简易模式**。
15. **运动健康**（Runner 等）：计步、运动开始/结束、目标步数、历史数据、实时播报、煲机（Burn-in）。
16. **OTA 升级**：整包差分、断点续传、进度上报、升级中禁控。
17. **离线媒体**（部分机型）：耳机内置存储，写/读/删音乐文件、剩余空间、同步模式（`OPEN_FILE`/`WRITE_FILE`/`CLOSE_FILE` 一族）。
18. **恢复出厂 / 重置 / 试用模式（Trial）**。
19. **豆包 AI 语音**（Atom / GS Links）：语音唤醒、ASR 上行、TTS 下行，走 `0x55AA` 帧。
20. **调试 / 日志**：`savelog` 埋点、`FFCOMMAND_GET_STATUS` 心跳。

---

## 7. 怎么自己发一条命令（可复现）

### 7.1 Python 组 GAIA 帧

```python
import struct

SOF = 0xFF
VENDOR_FIIL = 0x0234

def gaia_frame(cmd: int, payload: bytes = b"", new_proto=False) -> bytes:
    ver = 0xF1 if new_proto else 0x01          # 0x01 老协议 / 0xF1 新协议
    return bytes([SOF, ver, 0x00, len(payload)]) \
         + struct.pack(">H", VENDOR_FIIL) \
         + struct.pack(">H", cmd) \
         + payload

# 1) 读机型号（决定后续 UI/命令集）
pkt = gaia_frame(777)                       # FFCOMMAND_GET_EARTYPE
assert pkt == bytes.fromhex("ff01000002340309")   # 0x0309 = 777

# 2) 读电量
pkt = gaia_frame(771)                       # FFCOMMAND_GET_BATTERY_LEVEL  → ff01000002340303

# 3) 设降噪：老机型 / 走 setAnc() 的机型用 522；CC Pro2 等新机型改 1824
pkt = gaia_frame(522, b"\x01")              # FFCOMMAND_SET_MAF (setAnc)   降噪
# pkt = gaia_frame(1824, b"\x01")           # FFCOMMAND_SetMAF (setMaf)    新机型
# pkt = gaia_frame(513,  b"\x01")           # FFCOMMAND_SET_ANC (F002 专用)

# 4) 设 EQ = 索引 3
pkt = gaia_frame(515, b"\x03")              # FFCOMMAND_SET_EQ  → ff01000102340203 03
# 新机型：gaia_frame(1804, b"\x03")         # FFCOMMAND_SetEQ

# 5) 音量 +
pkt = gaia_frame(517)                       # FFCOMMAND_UP_VOLUME → ff01000002340205
```

> 命令号到底走「老」还是「新」，最稳的判据是**先用 777 读出 earType**（§5 机型表），再看该机型 UI 走的是 `HeatSetPagerXXXFragment` 还是 `f0xx` 新包 —— 两种页面的命令族不同。实战中更快的办法是直接抓一次 App 的 btsnoop，看你点按钮时它发的是哪一条。

### 7.2 解析回包

```python
def parse_gaia(buf: bytes):
    assert buf[0] == 0xFF
    length = buf[3]
    vendor = struct.unpack(">H", buf[4:6])[0]
    raw    = struct.unpack(">H", buf[6:8])[0]
    is_ack = bool(raw & 0x8000)
    cmd    = raw & 0x7FFF
    payload = buf[8:8+length]
    return dict(vendor=vendor, cmd=cmd, ack=is_ack, payload=payload)
```

### 7.3 用手机端抓包验证（推荐路径）

因为 SPP 是**标准 RFCOMM**，最稳的验证方式不是模拟耳机，而是**抓手机和耳机之间的链路**：

1. Android 开发者选项 → 开启「蓝牙 HCI 信息收集日志」（`btsnoop_hci.log`）；
2. 操作 App（切 ANC、切 EQ、调音量）；
3. 导出 `/sdcard/btsnoop_hci.log`，用 Wireshark 打开；
4. 过滤 `btbrlmp` / `btspp`，直接看 RFCOMM 载荷 —— 上面所有 `FF 01 00 ...` 帧都会原样出现；
5. 对照本报告的 §3.1 和 §4 命令表，即可反推每条操作对应的字节。

BLE 链路（F042/Airoha）用同一份 btsnoop 也能看到（过滤 ATT），只是要按 `13fa60a2` / `3d4dac58` 这两个 UUID 过滤特征值写入。

### 7.4 用 PC 直接连（L2CAP/RFCOMM）

Linux 下可用：

```bash
# 查设备上的 SPP 通道号
sdptool browse XX:XX:XX:XX:XX:XX
# 通道号一般是 1 或 2
rfcomm connect /dev/rfcomm0 XX:XX:XX:XX:XX:XX 1
# 往 /dev/rfcomm0 写上面组好的字节即可
```

Windows 没有原生 rfcomm 命令行，可用 Python `pybluez` 或 `bleak`（BLE 机型）。

> ⚠️ 注意：链路 B（Bluetrum）**是手机当服务端**，所以 PC 端不能主动连耳机，要反过来让耳机连 PC —— 需要模拟那个 `d52b47bc-2dea-42c4-b26d-20252fceeeb3` 的 SPP 服务端，并且耳机得先和手机的配对记录解绑，实际很难复现。这一路更适合用抓包而非自建连接。

---

## 8. 几个值得注意的设计点

1. **用反射读常量**：`getGaiaIntValue("VENDOR_FF")` / `getGaiaByteValue("SOF")`（`FiilCommandUtil` 内，最终落到 `com.fiil.sdk.utils.ReflectUtils` 的 `Class.forName(...).getDeclaredField(...)`）。作者故意让常量和命令号不直接出现在调用点，增加了静态分析成本（但对 Frida/动态调试无效）。
2. **命令分发靠字符串表**：`Config.sMap`（`com/fiil/sdk/config/Config.java:155` 声明、`:185` 起 145 条 `put`）是 `commandId → "analysisXxx"` 的 Map，收到回包后反射调用。加新命令只要加一行 Map + 一个方法。
3. **`isGaiaConnect()` 全量守卫**：`FiilManager` 里几乎每个 setter 开头都是
   ```java
   if (!this.f34440a.isGaiaConnect() || getDeviceInfo().isUpdate()) return;   // 例：FiilManager.java:1319、:1127
   ```
   即**升级中禁止一切控制**，避免 OTA 期间发命令把耳机刷死。全工程 `isGaiaConnect()` 出现 **362 处**，可见这条守卫被贯彻到多深。
4. **命令队列**：`com.fiil.sdk.queue.CommandQueue`（注意包名是 `queue` 不是 `command`）用 `new LinkedBlockingQueue<>(64)`（`CommandQueue.java:91`）串行发送；每条命令 3 秒超时（`com/fiil/sdk/command/a.java:13` → `protected int timeOut = 3000;`），超时后走失败回调。**不要并发硬灌**，会丢包。
5. **机型决定能力**：`earType` 不只是"显示名字"，它决定 UI 里哪些开关存在、ANC 有几档、是否支持 LDAC、是否走豆包链路。**同一份命令表，不同机型可用子集完全不同**。
6. **LDAC 是付费解锁**：`GetLDACBuyType` / `SetUnlockLADC` 说明存在"买了才给开"的商业逻辑，不是纯本地开关。
7. **两套"ANC"命令并存**：`MAF(650/522)` 与 `ANC(641/513)`。逆向时如果只盯一个会漏掉一半机型。
8. **广播里带电量**：V1/V2 Beacon 直接广播左右耳+盒子的电量与充电状态，所以 App 在**还没连上**时就能在列表里显示电量 —— 这是"未连接也显示电量"的实现原理。

---

## 9. 合规与边界提醒

- 本报告仅用于**理解自己拥有的设备如何被控制**、做互操作研究（如自建替代 App、做自动化测试、写开源驱动）。
- **不要**用于：破解付费功能（如 LDAC 解锁）、伪造 OTA 包、绕过设备鉴权、批量刷写他人设备。
- 抓包只抓**自己设备与自己手机**之间的链路；BLE/SPP 抓包不涉及他人数据。
- 若要做第三方 App：FIIL 的 GAIA 实现基于 CSR GAIA 框架改造，部分命令语义与 CSR 公开文档一致，但 `FFCOMMAND_*` 是厂商私有扩展，**无公开规范**，只能靠实验确定。

---

## 10. 关键文件索引（便于继续深挖）

| 主题 | 文件 |
|---|---|
| GAIA 常量全集 | `jadx_out/sources/com/fiil/sdk/gaia/father/Gaia.java` |
| 帧构造 | `jadx_out/sources/com/fiil/sdk/command/FiilCommandUtil.java:517`（`getSendBuffer`）|
| 帧定义（另一处） | `jadx_out/sources/com/fiil/sdk/gaia/father/Gaia.java:456`（`frame(...)`）|
| 帧解析 / 分发 | `jadx_out/sources/com/fiil/sdk/gaia/father/a.java` |
| 命令方法表 | `jadx_out/sources/com/fiil/sdk/config/Config.java:155`（`sMap` 声明）、`:185-329`（145 条 `put`）|
| 命令队列 | `jadx_out/sources/com/fiil/sdk/queue/CommandQueue.java`（容量 64 在 `:91`） |
| 对外 API（200+） | `jadx_out/sources/com/fiil/sdk/manager/FiilManager.java` |
| SPP socket | `jadx_out/sources/com/fiil/sdk/gaia/father/b.java` |
| 连接入口（老机型） | `FiilManager.java:275 connecting(String)` |
| 连接入口（CC Pro2 / F039） | `FiilManager.java:259 FIILCCPro2Connecting(String)` |
| 连接入口（秒连） | `FiilManager.java:1036 quickConnect()` |
| GAIA 收发门面 | `jadx_out/sources/com/fiil/sdk/gaia/father/d.java`（`GaiaService`，静态字段 `f34410a` 就是 GaiaLink）|
| 连接成功回调 / 握手 | `jadx_out/sources/com/fiil/sdk/c/a.java:116-149`（`ConnectHandler.handleMessage`）|
| BLE 扫描 | `jadx_out/sources/com/fengeek/main/ble/b.java` |
| brandId 过滤 | `jadx_out/sources/com/fengeek/main/f040/amd/viewmodels/g0.java:133`、`f039new/amd/viewmodels/f0.java:131` |
| headsetId 注入 | `HeatSetPagerCCPro2Fragment.java:510`（39）、`F40MainFragment.java:299`（40）、`F041MainFragment.java:296`（41）|
| Beacon 解析 | `jadx_out/sources/com/bluetrum/devicemanager/models/DeviceBeacon*.java` |
| Bluetrum 组包 | `jadx_out/sources/com/bluetrum/devicemanager/z.java` |
| Bluetrum SPP 服务端 | `jadx_out/sources/com/bluetrum/devicemanager/bluetooth/BluetoothSppService.java` |
| F042 BLE 组包 | `jadx_out/sources/com/fengeek/main/ble/FIILPeripheral.java:444` |
| 豆包 0x55AA | `jadx_out/sources/com/fengeek/main/i/d/e.java`（`setDouBao`） |
| 豆包收包状态机 | `jadx_out/sources/com/fengeek/main/i/b.java`（`c(byte[])`） |
| 机型映射 | `jadx_out/sources/com/fengeek/f002/MainActivity.java:1902` |
| 型号文案 | `apktool_out/res/values/strings.xml` |
| Airoha UUID | `jadx_out/sources/com/airoha/liblinker/.../a.java` |
| Airoha Race 包 | `jadx_out/sources/com/airoha/android/lib/e/b/a.java` |

---

*报告生成依据：jadx 1.5.6 反编译 `fiil+_3.5.2.apk`，全部结论附源码行号，可逐条回查。*


