[简体中文](REVERSE_ENGINEERING_REPORT.md) | **[English](REVERSE_ENGINEERING_REPORT.en.md)**

# FIIL+ 3.5.2 Reverse Engineering Analysis Report

> Objective: figure out exactly **how the FIIL+ (Feier) App connects to its own earbuds**, **how it controls them**, **what features it has**, and **how it sends commands**.
> Nature: learning / research use. All conclusions come from static decompilation reading of the APK (jadx + apktool); no live attack activity.

---

## 0. Sample under test

| Item | Value |
|---|---|
| File | `fiil+_3.5.2.apk` (94,576,645 bytes) |
| Package name | `com.fengeek.f002` |
| versionName | `3.5.2` |
| versionCode | `175` |
| minSdk / targetSdk / compileSdk | 26 / 33 / 33 |
| Decompiled artifacts | `jadx_out\sources` (Java), `apktool_out` (resources + smali) |
| Key SDK packages | `com.fiil.sdk` (in-house GAIA protocol stack), `com.bluetrum` (Bluetrum chipset stack), `com.airoha` (Airoha chipset stack), `com.fengeek.main.ble` (F042 series BLE private stack) |
| Third-party | Doubao voice SDK (`com.startburst.sdk`), Firebase Crashlytics, FastBle |

---

## 1. Overview in one sentence

The App **does not run over a single link**. It runs **4 completely different transport/protocol stacks** side by side, dispatched by "earbud chipset + earType model number":

| # | Link | Transport | Applicable models | Code entry point |
|---|---|---|---|---|
| A | **GAIA over SPP** | Classic Bluetooth SPP (RFCOMM) | Most older / mainstream models (FIIL, Diva, Carat, T1, CC, CG, Driifter…) | `com.fiil.sdk.gaia.father.b` (GaiaLink) + `FiilManager` |
| B | **Bluetrum private over SPP** | Classic Bluetooth SPP, **the phone opens the server** and waits for the earbuds to connect in | Bluetrum-chipset models (AB53xx series) | `com.bluetrum.devicemanager` |
| C | **Airoha private** | BLE GATT custom service + SPP dual channel | Airoha-chipset models (CC Pro2, F042/F043, etc.) | `com.airoha.*` + `F042MainFragment` |
| D | **F042 BLE private** | Pure BLE GATT (custom 128-bit UUID) | Newer models such as F042 / CC Pro2 | `com.fengeek.main.ble.FIILPeripheral` |

There is also a **fifth, business-level channel**: **Doubao voice** (`0x55AA` frames) runs only on earType 47/48 models (Atom / GS Links); the frame format is in §3.4 and the models are in §5.

> One easy-to-miss caveat: **A/B/C/D are not "strictly one-to-one with model numbers"**. CC Pro2 is a hybrid — it uses Bluetrum's Beacon structure for discovery (§2.2), but when establishing the link it calls `FIILCCPro2Connecting()` and borrows Bluetrum's private SPP UUID (`d52b47bc-…`) as a client, while the upper layer still runs GAIA frames (§2.3 B-2). So to decide which path a given device takes, **the only reliable signal is which `connecting` overload its model page calls**.

---

## 2. Connection flow: how the App "finds and connects to" the earbuds

### 2.1 Step 1: BLE advertising scan discovers devices

`com.fengeek.main.ble.b` (`FIILCentralManager`, which itself `extends ScanCallback`) handles scanning (`com/fengeek/main/ble/b.java`):

- `startScan()` @L109 → directly `BluetoothAdapter.getDefaultAdapter().getBluetoothLeScanner().startScan(f27219b)` @L131, **no ScanFilter and no ScanSettings are passed** (system-default SCAN_MODE_LOW_LATENCY semantics);
- 30-second timeout: constant `f27220c = 30000` @L25, `c()` @L78 starts `new CountDownTimer(30000L, 1000L).start()`, `onFinish()` @L46 → `stopScan()` + callback `onScanFinished(new ArrayList(hashSet))`;
- Result de-duplication: `HashSet<com.fengeek.main.ble.e.a>` @L28; `onScanResult` @L104 only invokes `onScanning` when `if (this.f27221d.add(aVar))` succeeds;
- Device wrapper class `com.fengeek.main.ble.e.a` (`BluetoothDevice` + `rssi` + raw advertising `scanRecord.getBytes()` + name) @L101.

> Conclusion: **the discovery stage applies no filtering whatsoever** — every BLE device reaches the callback. Whether a device "is a FIIL earbud" is decided entirely in the next step, by parsing the manufacturer data in the advertisement.

### 2.2 Step 2: parsing the manufacturer advertising data (Beacon)

This is the key to deciding "is this one of our earbuds". `com.bluetrum.devicemanager.models.DeviceBeacon`:

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

Payload layouts of the three versions (all verified byte by byte):

**V1 (13 bytes, `DeviceBeaconV1.java`)**

| Offset | Length | Meaning |
|---|---|---|
| 0 | 1 | version (low 4 bits) + other flag bits |
| 1–2 | 2 | productId (**little-endian** short, `byteBuffer.getShort() & 0xFFFF`, base-class `DeviceBeacon.java` constructor) |
| 3–8 | 6 | earbud Bluetooth MAC (**in-order** raw bytes; `BluetoothUtils.getAddressStringFromByte` concatenates directly in `%02X:%02X:...` order, no reversal) |
| 9 | 1 | bit0 = needAuth; bits2–3 = connection state (`(b2>>2)&3`) |
| 10 | 1 | left earbud: bit7 = charging, bit0–6 = battery % (`b3 & 127`) |
| 11 | 1 | right earbud: same as above |
| 12 | 1 | charging case: same as above |

**V2 (13 bytes, `DeviceBeaconV2.java`)**

| Offset | Length | Meaning |
|---|---|---|
| 0 | 1 | version (low 4 bits) |
| 1–2 | 2 | productId (little-endian, same as V1, consumed by the base class first) |
| 3–8 | 6 | earbud MAC, restored only after **XOR 0xAD on every byte** (`btAddress[i] ^= (byte)0xAD`, `-83` is 0xAD) |
| 9 | 1 | bit0 = needAuth; bits2–3 = connection state |
| 10–12 | 3 | brandId (little-endian 24-bit: `get() \| (get()<<8) \| (get()<<16)`); **but if `data.length >= 12`, brandId is overwritten with the single byte `data[11]`** |

**V3 (5 bytes, latest generation)**

```java
public DeviceBeaconV3(byte[] data) {
    super(data, 0);                       // base class consumes data[0], data[1] first
    this.f16440h = "LE-FIIL CC Pro2";
    byte[] bArr = new byte[3];
    this.byteBuffer.get(bArr);            // data[2..4]
    StringBuilder sb = new StringBuilder();
    sb.append("B0:F1:A3");                // prefix is hard-coded
    for (int i = 2; i >= 0; i--) {        // appended in reverse order → the MAC is stored backwards
        sb.append(':');
        sb.append(convertToHexString(bArr[i] & 255));
    }
    this.mBtAddress = sb.toString();
    setOtherName("LE-FIIL CC Pro2");
    setOtherAddress(this.mBtAddress);
    if (data.length == 5) this.brandId = data[0];
}
```

| Offset | Length | Meaning |
|---|---|---|
| 0 | 1 | brandId, fixed to `39` (0x27, i.e. FIIL's vendor number) — the base-class `DeviceBeacon(data, 0)` stuffs `data[0]` into both `productId` and `brandId` |
| 1 | 1 | version (**low 4 bits**, extracted by the base class with `& 15`) |
| 2–4 | 3 | last three MAC bytes, concatenated **in reverse**: `B0:F1:A3` + `data[4]` + `data[3]` + `data[2]` (loop `for(i=2;i>=0;i--)` appends in descending order) |

> Note that V3 advertises only the **last 3 bytes of the MAC plus the hard-coded prefix `B0:F1:A3`**, not the full MAC (the prefix comes from a mass-production MAC range; real devices may not actually share that prefix). V3 also forcibly injects a name `LE-FIIL CC Pro2` and an `otherAddress` — which indicates V3 was designed for the CC Pro2 generation's "LE Audio / low-power always-on connection", and the full MAC must be read later over a GATT connection. In addition, `DeviceBeaconV3` uses a **big-endian** `ByteBuffer.wrap(data)` (the `super(data, 0)` constructor), unlike the little-endian V1/V2 — don't mix them up when reading multi-byte fields.


**brandId filtering** (`com/fengeek/main/f040/amd/viewmodels/g0.java:128-136`, `com/fengeek/main/f039new/amd/viewmodels/f0.java:125-137`; the two are logically identical, the latter written as nested `if`s):

```java
byte[] msd = scanRecord.getManufacturerSpecificData(1602);   // 0x0642
if (msd == null) msd = scanRecord.getManufacturerSpecificData(564);   // 0x0234 = VENDOR_FIIL

if (DeviceBeacon.isDeviceBeacon(msd) && (beacon = DeviceBeacon.getDeviceBeacon(msd)) != null
        && beacon.getAgentId() == 0
        && beacon.getBrandId() == this.headsetId) {
    // treated as a FIIL device
}
```

That is, three gates: **Beacon structure is valid** → **agentId (`brandId >> 16`) is 0** → **brandId equals the headsetId configured in the App**. This is the "only accept our own earbuds" filtering logic — **it does not rely on a name prefix, but on the vendor ID field in the advertisement**.

`headsetId` is not a hard-coded constant; it is **injected separately by each model page** (`setHeadsetId(int)` → `startScan(headsetId)` → `this.f27647i = headsetId`, `com/fengeek/main/f040/amd/viewmodels/g0.java:162-171`):

| Injection point | Value |
|---|---|
| `com/fengeek/main/HeatSetPagerCCPro2Fragment.java:510` | `39` (the V3 Beacon's `brandId` is fixed to 39, which matches exactly) |
| `com/fengeek/main/f040/ui/fragment/F40MainFragment.java:299` | `40` |
| `com/fengeek/main/f041/fragment/F041MainFragment.java:296` | `41` |

In other words: **the same scan callback is filtered with different brandIds by different model pages**. The CC Pro2 page accepts only V3 Beacons with `brandId==39`, the F40 page accepts only `brandId==40`, and F041 only `41`. This explains why the `brandId` in the V3 Beacon is hard-coded to `data[0]=39` — it is a one-off design for the CC Pro2 generation, not a general vendor number.

> Packet-capture tip: first try `getManufacturerSpecificData(1602)`, and fall back to `564` if that returns null. 1602 = 0x0642 is a Company ID that Bluetooth SIG assigned to certain chip vendors, while 564 = 0x0234 happens to be exactly the `VENDOR_FF` FIIL itself uses inside GAIA frames. If an advertisement carries an AD structure starting with `0x0234`, it is almost certainly a FIIL earbud.
>
> Note that **agentId must be 0**: `getAgentId()` is `getBrandId() >> 16`. For V3 (5 bytes) `brandId = data[0] = 39`, and shifting right by 16 yields 0, so it passes naturally; for V1/V2, brandId is 24-bit or a single byte, and only those whose high byte is 0 are let through. This is the split between "consumer earbuds" and "agent / ODM devices".

### 2.3 Step 3: establishing the data connection

#### A. GAIA over SPP (mainstay of older models)

`FiilManager.connecting(String address)` (`com/fiil/sdk/manager/FiilManager.java:275`):

```java
d dVar = MyFiilReceiver.f34272h;
d.f34410a.b(UUID.fromString("00001101-0000-1000-8000-00805F9B34FB"));  // standard SPP
Method m = this.f34446g.getClass().getDeclaredMethod("blueConn", d.class, String.class);
m.setAccessible(true);
m.invoke(this.f34446g, this.sdkService, str);
```

It uses **reflection** to call `MyFiilReceiver.blueConn(...)`, passing in the SPP UUID and the target MAC; the actual socket lives in `com.fiil.sdk.gaia.father.b` (`GaiaLink`, internal name `GaiaSocket`):

```java
// com/fiil/sdk/gaia/father/b.java:445  socket factory
protected BluetoothSocket a(UUID uuid) throws IOException {
    if (g()) return f34370h.createInsecureRfcommSocketToServiceRecord(uuid);  // insecure mode
    return f34370h.createRfcommSocketToServiceRecord(uuid);                    // secure mode
    // fallback on failure: reflectively call the hidden API createRfcommSocket(1)
}
```

- Default UUID: `00001101-0000-1000-8000-00805F9B34FB` (standard SPP, `b.java:236`);
- Also supports `00001107-D102-11E1-9B23-00025B00A5A5` (constant `w` at `b.java:29`, the private SPP channel of some older devices), swapped in before connecting via `b(UUID uuid)` (`b.java:305`);
- There is also a **UDP emulator mode** (`c(String str)`, `b.java:309`): `DatagramSocket` connects to `10.0.2.2` (the address that points to the host machine inside the Android emulator), used for debugging without real hardware — which shows pure software simulation was done during development;
- Once connected, `com.fiil.sdk.gaia.father.a` performs **frame reassembly** (handling SPP packet coalescing / fragmentation).

Handshake sequence after connecting (`com/fiil/sdk/c/a.java:116-149`, inside `handleMessage` after receiving the "connection succeeded" message):

```java
// 1) reflectively set the flag to true
Method m1 = FiilManager.getInstance().getDeviceInfo().getClass()
        .getDeclaredMethod("setGaiaConnect", Boolean.TYPE);
m1.setAccessible(true);
m1.invoke(FiilManager.getInstance().getDeviceInfo(), Boolean.TRUE);

// 2) get the firmware version (skipped for F039 models)
if (!FiilManager.isF039) FiilManager.getInstance().getVersion(null);

// 3) get the model number -- the critical step
FiilManager.getInstance().getEarType(new C0338a());

// 4) reflectively obtain all ConnectionListeners and call onConnectionSuccess() on each
Method m2 = ConnectionUtils.getConnectionUtils().getClass()
        .getDeclaredMethod("getConnectionListeners", new Class[0]);
m2.setAccessible(true);
for (ConnectionListener l : (ArrayList<ConnectionListener>) m2.invoke(...)) l.onConnectionSuccess();
```

`C0338a` (`com/fiil/sdk/c/a.java:27-64`) is the earType callback: **on success it does nothing**; **on failure it degrades to reading `getLedTemp`** (to determine whether this is an older model). The place that actually consumes earType is `a()` (`com/fiil/sdk/c/a.java:245`):

```java
public void a() {
    com.fiil.sdk.command.b bVar = new com.fiil.sdk.command.b();   // F002-specific command class
    a("setEarType", 2, Integer.TYPE);                              // reflectively write DeviceInfo.setEarType(2)
    for (Class<?> c = com.fiil.sdk.command.b.class; c != com.fiil.sdk.command.a.class; c = c.getSuperclass()) {
        Method m = c.getDeclaredMethod("initOrder", new Class[0]); // walk up the hierarchy to find initOrder and call it
        m.setAccessible(true); m.invoke(bVar); break;
    }
    for (CommandEventListener l : b()) l.earType(2);               // notify all listeners that "model number = 2"
}
```

In other words: **connection succeeds → read version + read earType → call back into the UI's `onConnectionSuccess()`**, and the UI then builds the matching Fragment for that earType and pulls battery/ANC/EQ state as needed. It is **not** the case that the SDK reads every piece of state in one pass.

#### B. Bluetrum private SPP (the phone acts as the server!)

`com/bluetrum/devicemanager/bluetooth/BluetoothSppService.java`:

```java
private static final UUID f16296c = UUID.fromString("d52b47bc-2dea-42c4-b26d-20252fceeeb3");

// key point: listenUsingInsecureRfcommWithServiceRecord -- the phone is the server
BluetoothAdapter.getDefaultAdapter()
    .listenUsingInsecureRfcommWithServiceRecord("BluetoothSppService", f16296c);
```

**The direction is reversed**: the phone opens an SPP server socket and listens, and the earbuds actively connect in (`accept()`). This is a common approach on Bluetrum chipsets (the earbud side acts as the SPP client, which saves power and allows reconnection).

#### B-2. The CC Pro2 hybrid path (same UUID, two directions)

`FiilManager` contains a third connection entry point, easy to confuse with the older-model one:

```java
// com/fiil/sdk/manager/FiilManager.java:259
public void FIILCCPro2Connecting(String str) {
    isF039 = true;                                                   // global static flag (:73)
    d dVar = MyFiilReceiver.f34272h;
    d.f34410a.b(UUID.fromString("d52b47bc-2dea-42c4-b26d-20252fceeeb3")); // ★ switch the SPP UUID
    this.f34446g.a(this.sdkService, str);                            // ★ direct call, no reflection
}
```

Comparing it with `connecting(String)` (`:275`) makes the fork point obvious:

| | `connecting()` (:275) | `FIILCCPro2Connecting()` (:259) | `quickConnect()` (:1036) |
|---|---|---|---|
| `isF039` | `false` | **`true`** | `false` |
| SPP UUID | `00001101-…` (standard SPP) | **`d52b47bc-…`** (Bluetrum private) | `00001101-…` |
| Call style | reflective `blueConn(d, String)` | direct `a(service, str)` | reflective `blueConnQuick()` |

**Here is something easy to overlook**: the UUID `d52b47bc-2dea-42c4-b26d-20252fceeeb3` appears **twice, in two directions**, in the package —

- `BluetoothSppService.java:35`: the phone calls `listenUsingInsecureRfcommWithServiceRecord(...)` **acting as server**;
- `FiilManager.java:266`: the phone calls `createRfcommSocketToServiceRecord(...)` **acting as client**.

Same UUID, same process, two directions. Older models use standard SPP (`0x1101`) as client; CC Pro2 instead uses this private UUID to actively connect to the earbuds — **the transport layer borrows Bluetrum's private channel number, while the upper layer still runs GAIA frames** (the `d`→`b` `GaiaLink` object is unchanged). So deciding "which link a model takes" cannot be done from the UUID alone; you have to check whether it calls `connecting()` or `FIILCCPro2Connecting()`.

`isF039` (`public static boolean`, `FiilManager.java:73`) appears in only 6 places across the whole package, and the only read site is in the handshake:
`if (!FiilManager.isF039) { FiilManager.getInstance().getVersion(null); }` (`com/fiil/sdk/c/a.java:132`) —
i.e. **devices in the earType=39 (CC Pro2) class skip the GAIA version query during handshake and go straight to reading earType**.
This is a side-effect signal for determining the model generation during reverse engineering; whether the device itself fails to answer a version query cannot be determined from static code — all that can be established is that the App deliberately does not ask.

#### C. Airoha dual channel

`com/airoha/liblinker/constant/a.java` (internal name `UuidTable`, the UUID table shared by the entire Airoha protocol stack):

```java
public static final UUID f10186a = UUID.fromString("00002902-0000-1000-8000-00805F9B34FB"); // CCCD notify toggle
public static final UUID f10187b = UUID.fromString("00000000-0000-0000-0099-AABBCCDDEEFF"); // private service
public static final UUID f10188c = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB"); // standard SPP
public static final UUID f10189d = UUID.fromString("5052494D-2DAB-0341-6972-6F6861424C45"); // "PRIM" primary service
public static final UUID f10190e = UUID.fromString("43484152-2DAB-3241-6972-6F6861424C45"); // "CHAR"-2A
public static final UUID f10191f = UUID.fromString("43484152-2DAB-3141-6972-6F6861424C45"); // "CHAR"-1A
public static final UUID f10192g = UUID.fromString("49535343-FE7D-4AE5-8FA9-9FAFD205E455"); // transparent service (Microchip BM7x)
public static final UUID f10193h = UUID.fromString("49535343-8841-43F4-A8D4-ECBE34729BB3"); // transparent write
public static final UUID f10194i = UUID.fromString("49535343-1E4D-4BD9-BA61-23C647249616"); // transparent notify
```

The ASCII inside the UUIDs is interesting: `PRIM` = primary (primary service), `CHAR` = characteristic — the semantics are encoded directly into the 128-bit UUID.

Connection entry point: `com/fengeek/main/f042/fragment/F042MainFragment.java:363 connectAirohaSDK()`, which first fetches `com.fengeek.main.ble.c.getInstance().getmBLEDevice()`, then checks whether `F042HomeFragment` is ready, and then calls `DeviceControl.connect(...)`.

#### D. F042 BLE private (newer models)

`com/fengeek/main/f042/fragment/F042HomeFragment.java:62-68`:

```java
public static final String f28317d = "13fa60a2-429a-4a90-a3e8-d4c2b9a17655";  // Service
public static final String f28318e = "3d4dac58-7d68-4734-8a71-21bc659df11c";  // Write characteristic
public static final String f28319f = "3d4dac58-7d68-4734-8a71-21bc659df11c";  // Notify characteristic (same as write)
```

Note that `f28318e` and `f28319f` **are the same UUID** — this characteristic supports both Write and Notify, with a single characteristic value for both directions (a convenient shortcut common in domestic BLE designs).

```java
// link setup (F042HomeFragment.java:1619)
FIILPeripheral.getInstance().connectGatt(getContext(), bleDevice,
        f28317d, "3d4dac58-7d68-4734-8a71-21bc659df11c", new x());

// subscribe to notifications (F042HomeFragment.java:1215)
FIILPeripheral.getInstance().notify("3d4dac58-7d68-4734-8a71-21bc659df11c", new C0254a());
```

Internally, `FIILPeripheral.connectGatt(Context, DeviceWrapper, serviceUUID, writeCharUUID, callback)` performs `discoverServices` → find the service → find the characteristic → `writeDescriptor(CCCD)` to enable notify, and only then can data be sent and received.

---

## 3. Protocol frame formats (core)

### 3.1 GAIA frame (link A) — the most important one

Constructor `FiilCommandUtil.getSendBuffer(int vendor, int cmd, boolean newProto, byte... payload)`
(`com/fiil/sdk/command/FiilCommandUtil.java:517`):

```java
int off = getGaiaIntValue("OFFS_PAYLOAD");          // = 8
byte[] out = new byte[payload.length + off];
out[0] = 0xFF;                                       // SOF
out[1] = newProto ? (byte)0xF1 : 0x01;               // protocol version
out[2] = 0;                                          // FLAGS
out[3] = (byte) payload.length;                      // PAYLOAD_LENGTH
out[4] = (byte)(vendor >> 8);                        // VENDOR_ID_H
out[5] = (byte) vendor;                              // VENDOR_ID_L
out[6] = (byte)(cmd >> 8);                           // COMMAND_ID_H
out[7] = (byte) cmd;                                 // COMMAND_ID_L
// out[8..] = payload
```

All constants come from `com/fiil/sdk/gaia/father/Gaia.java` (read via `ReflectUtils` reflection, which shows the author deliberately applied light anti-reverse-engineering):

| Constant | Value | Meaning |
|---|---|---|
| `SOF` | `-1` = 0xFF | start of frame |
| `PROTOCOL_VERSION` | `1` | legacy protocol version byte |
| `NEW_PROTOCOL_VERSION` | `241` = 0xF1 | new protocol version byte |
| `DEFAULT_FLAGS` | `0` | default flags |
| `FLAG_CHECK` | `1` | when set, a 1-byte checksum is appended to the frame |
| `OFFS_SOF` | 0 | |
| `OFFS_VERSION` | 1 | |
| `OFFS_FLAGS` | 2 | |
| `OFFS_PAYLOAD_LENGTH` | 3 | |
| `OFFS_VENDOR_ID` / `_H` / `_L` | 4 / 4 / 5 | |
| `OFFS_COMMAND_ID` / `_H` / `_L` | 6 / 6 / 7 | |
| `OFFS_PAYLOAD` | 8 | payload start |
| `VENDOR_FF` | `564` = 0x0234 | FIIL vendor ID |
| `COMMAND_MASK` | 32767 = 0x7FFF | take the low 15 bits to get the real command number |
| `ACK_MASK` | 32768 = 0x8000 | when set, this is a "response/ACK" |

**So the minimum frame length is 8 bytes** (no payload); total length = `8 + payloadLen (+1 if FLAG_CHECK)`.

`Gaia.frame(...)` (`Gaia.java:456`) / `Gaia.newFrame(...)` (`Gaia.java:544`) are another implementation of the same frame — the only difference between them is the **version byte**: `frame()` hard-codes `bArr2[1] = 1`, while `newFrame()` uses `bArr2[1] = i.f9426g` (i.e. the new protocol version). Everything else is identical, including:

```java
if (i4 > 254) throw new IOException("GAIA frame full");   // length cap 254 (1-byte length field)
int i7 = (b2 & 1) != 0 ? 1 : 0;                            // flags bit0 = checksum present
int i8 = i4 + 8 + i7;                                      // total length
...
if (i7 != 0) {                                             // with checksum: XOR all preceding bytes and append to the frame tail
    byte b3 = 0;
    for (int i6 = 0; i6 < i8 - 1; i6++) b3 ^= bArr2[i6];
    bArr2[i8 - 1] = b3;
}
```

Note that in `frame()` we have `bArr2[2] = b2` (flags come straight from the argument), whereas in `getSendBuffer()` we have `out[2] = 0` (always 0). So **the two paths handle flags differently**: only `Gaia.frame(...)` can enable the checksum bit.

**Frame example** (read the current ANC value, older models):

```
FF 01 00 00 02 34 02 8A
│  │  │  │  └┬─┘ └┬─┘
│  │  │  │   │    └─ cmd = 0x028A = 650 = FFCOMMAND_GET_MAF (older models express ANC as MAF)
│  │  │  │   └────── vendor = 0x0234 = 564
│  │  │  └────────── payload length 0
│  │  └───────────── flags 0
│  └──────────────── version 1 (the new protocol uses F1)
└─────────────────── SOF
```

**Parsing a response** (`com.fiil.sdk.gaia.father.a`):

```
vendor   = u16(frame[4..5])
cmdRaw   = u16(frame[6..7])
isAck    = (cmdRaw & 0x8000) != 0
cmd      = cmdRaw & 0x7FFF
payload  = frame[8 .. 8+len]
```

Once `cmd` is known, `com.fiil.sdk.command.a` (`FiilBaseCommand`) uses `Config.sMap` (declared at `com/fiil/sdk/config/Config.java:155`, populated one `put` at a time from `:185` onward, **145 entries** of `commandId → "analysisXxx"`) to **reflectively invoke** the corresponding `analysisXxx(byte[])` for business-level parsing. This is the dispatch hub of the entire SDK.

> Note the distinction between two numbers: **190** is the total number of `FFCOMMAND_*` constants defined in `Gaia.java`; **145** is the number of entries in `Config.sMap` that actually have a handler attached. The remaining 45 are defined but unhandled (or handled by other modules comparing the numeric value directly); if you go looking through `sMap` during reverse engineering you will find "some commands are missing" — this is why.

### 3.2 Bluetrum frame (link B)

Assembler `com/bluetrum/devicemanager/z.java` (internal name `RequestHandler`), method `b(Request)` @L36:

```
[seq:1][command:1][commandType:1][packInfo:1][len:1][payload:len]
```

- `seq`: a **low 4-bit** rolling counter; each transmitted fragment does `this.f16473e = (byte)((this.f16473e + 1) & 15)` (`z.java:49`, `:64`), reset via `c()` @L70. Note that **each fragment takes its own seq** — it is not "one seq per Request";
- `packInfo`: `(byte)((((iB-1) << 4) & 0xF0) | (i2 & 15))` (`z.java:55`, mask constant `com/bluetrum/fota/i.java:14` → `f16511c = 240` = 0xF0) — high 4 bits "total packets - 1", low 4 bits "which packet this is";
- An empty payload takes a special case (`z.java:41-49`): `ByteBuffer.allocate(5)` directly, `packInfo=0`, `len=0`, frame length fixed at 5;
- Splitter `a0` (`com/bluetrum/devicemanager/a0.java`, internal name `RequestSplitter`) defaults to `new a0(15)` (`z.java:29`) — 15 bytes of net payload per packet; inside `d(int maxPacketSize)` @L74, `this.f16472d.d(maxPacketSize - 5)` subtracts the 5-byte header, so with a negotiated MTU of 20 the net payload is exactly 15. `b(int payloadLength)` returns `(payloadLength + i2 - 1) / i2`, i.e. the total packet count rounded up.

Command type constants: `com.bluetrum.devicemanager.cmd.Request`; `commandType` distinguishes GET/SET/NOTIFY/RESPONSE.

### 3.3 F042 BLE frame (link D)

Assembly happens in `com/fengeek/main/ble/FIILPeripheral.java:444 write(BaseCommand, Handler)`:

```java
arrayList.add((byte) -1);                                   // 0xFF
arrayList.add((byte) 1);                                    // version 1
arrayList.add((byte) 0);                                    // flags 0
arrayList.add((byte) (payload.length + 2));                 // payload length (+2 = cmd byte + len byte)
arrayList.add((byte) 2);                                    // vendor H
arrayList.add((byte) 52);                                   // vendor L → 0x0234
arrayList.add((byte) ((cid >> 8) & 255));                   // commandClassifyId high byte
arrayList.add((byte) (cid & 255));                          // commandClassifyId low byte
arrayList.add((byte) cmd);                                  // commandId (1 byte)
arrayList.add((byte) payload.length);                       // payload length (1 byte)
// payload...
```

That is:

```
FF 01 00 (payloadLen+2) 02 34 CID_H CID_L CMD LEN payload...
```

**Note**: it looks very similar to the GAIA frame (the same `FF .. 02 34`), but the semantics differ — in GAIA bytes 6/7 are a 2-byte commandId; in F042 bytes 6/7 are a 2-byte **commandClassifyId**, byte 8 is the commandId, and byte 9 is yet another payload length. When writing a parser, **do not conflate the two links**.

### 3.4 Doubao 0x55AA frame (voice service)

`setDouBao(byte[] data, listener)` in `com/fengeek/main/i/d/e.java`:

```java
arrayList.add((byte) 85);                    // 0x55
arrayList.add((byte) -86);                   // 0xAA
arrayList.add((byte) 0);                     // type/sequence
arrayList.add((byte) (len & 255));           // length low byte
arrayList.add((byte) ((len >> 8) & 255));    // length high byte
// data...
bArr[arrayList.size()]     = -91;            // 0xA5
bArr[arrayList.size() + 1] = 90;             // 0x5A
```

```
55 AA 00 LEN_LO LEN_HI payload... A5 5A
```

**Receive direction** (`c(byte[])` in `com/fengeek/main/i/b.java`) is a state machine:

- If `data[0]==0x55 && data[1]==0xAA` → this is a frame **"sent to Doubao"** (`isApp=false`), and the length is taken from `d(new byte[]{data[4], data[3]})`. `d()` (`com/fengeek/main/i/b.java:145`) accumulates as `(acc<<8)|byte`, so passing `{data[4], data[3]}` yields **`data[4]<<8 | data[3]`**, i.e. a **little-endian 16-bit** value (low byte first); the segment is `copyOfRange(data, i3+5, i6)` and afterwards the cursor becomes `i3 = i6 + 2` (skipping the trailing `A5 5A`);
- Otherwise → this is an ordinary frame **"sent to the App"** (`isApp=true`), the length is taken from `data[4]` (single byte), the segment end is computed as `i4 + i2 + 5`, and the cursor becomes `i3 = i5` (no extra skip);
- The outer loop is `while(true)`; each round re-reads a 5-byte header and re-decides the direction, returning only when fewer than 5 bytes remain (log line "组包数据处理完毕").

```java
// com/fengeek/main/i/b.java:91  c(byte[] data)  -- receive state machine (equivalent pseudocode after line-by-line verification)
boolean isApp = !((data[0]&255) == 85 && (data[1]&255) == 170);
byte[] head = copyOfRange(data, 0, 5);
int segStart = 0, consumed = 0;
for (;;) {
    if (isApp) {                                   // ordinary frame: 5-byte header + data[4] length
        int len = head[4] & 255;
        int end = consumed + len + 5;
        if (end > data.length) return;             // "App数据错误"
        dispatch(copyOfRange(data, segStart, end));
        segStart = end; consumed = end;
    } else {                                       // Doubao frame: 55 AA 00 len_lo len_hi + trailing A5 5A
        int len = d(new byte[]{head[4], head[3]}); // = data[4]<<8 | data[3]
        int end = consumed + len + 5;
        if (end > data.length) return;             // "豆包数据错误"
        dispatch(copyOfRange(data, segStart + 5, end));
        segStart = end + 2; consumed = end;        // +2 skips A5 5A
    }
    if (segStart + 5 > data.length) return;        // "组包数据处理完毕"
    head = copyOfRange(data, segStart, segStart + 5);
    isApp = !((head[0]&255) == 85 && (head[1]&255) == 170);
}
```

`setDouBao` writes the length as `len&255`→index 3 and `len>>8`→index 4, while `d()` reassembles `data[4]<<8|data[3]` — the writer and the reader match exactly, and it is **little-endian**.

This code also hard-codes: **only models with earType == 47 or 48** take this unpacking path (`com/fengeek/main/i/b.java:187` → `if (this.f31855k == 48 || this.f31855k == 47)`); all other models hand the raw bytes straight up via `obtainMessage(1, ...)`.

### 3.5 Airoha Race packet

`com/airoha/android/lib/e/b/a.java` (`RacePacket`): `SOF=5`, type=6, total-length field… the whole thing is the encapsulation of Airoha's official RACE protocol; the business layer is in `com.airoha.libmmi` (MMI = device interaction messages) and `com.airoha.libbase.RaceCommand` (routing by the four-tuple AppId / Recipient / RaceId / RaceType).

---

## 4. Command set: 190 FFCOMMANDs

All the constants are concentrated in `com/fiil/sdk/gaia/father/Gaia.java` (`static final int FFCOMMAND_xxx`); the full list is in the bundled `_cmd_table.txt` (190 lines, `name|decimal value`). Here they are grouped by function, with the highlights selected:

### 4.1 Connection / information

| Command | Value (hex) | Description |
|---|---|---|
| `FFCOMMAND_GET_EARTYPE` | 777 (0x0309) | **get the model number**; determines the UI and the command set |
| `FFCOMMAND_GET_VERSION` | 769 (0x0301) | firmware version |
| `FFCOMMAND_GetTWSVersion` | 1819 (0x071B) | TWS version |
| `FFCOMMAND_GET_ADDRESS` | 770 (0x0302) | device address |
| `FFCOMMAND_GET_SEQCODE` | 772 (0x0304) | serial number |
| `FFCOMMAND_GET_STATUS` | 16394 (0x400A) | connection status |
| `FFCOMMAND_GET_BATTERY_LEVEL` | 771 (0x0303) | battery (primary) |
| `FFCOMMAND_GetBatteryLevel` | 1800 (0x0708) | battery (the other set, newer models) |
| `FFCOMMAND_GetBoxBatter` | 1915 (0x077B) | charging-case battery |
| `FFCOMMAND_GET_BATTERY_CHARGE` | 774 (0x0306) | whether charging |
| `FFCOMMAND_GET_USETIME` | 773 (0x0305) | cumulative usage time |

### 4.2 Noise cancellation (ANC / MAF)

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_ANC` | 641 (0x0281) | read ANC (F002-specific class) |
| `FFCOMMAND_SET_ANC` | 513 (0x0201) | write ANC (F002-specific class) |
| `FFCOMMAND_GET_MAF` | 650 (0x028A) | read MAF (legacy enum name, actually used by `getAnc()`) |
| `FFCOMMAND_SET_MAF` | 522 (0x020A) | write MAF (legacy enum name, actually used by `setAnc()`) |
| `FFCOMMAND_GetMAF` | 1823 (0x071F) | read MAF (newer models, used by `getMaf()`) |
| `FFCOMMAND_SetMAF` | 1824 (0x0720) | write MAF (newer models, used by `setMaf()`) |

**Pitfall 1: command names and API names are misaligned.** `Gaia.java` contains both `FFCOMMAND_GET_MAF`(650) and `FFCOMMAND_GetMAF`(1823) — differing only in the case of one letter, they are two completely different command groups. And the mapping in `FiilCommandUtil` is crossed:

| API | Command actually sent | Value |
|---|---|---|
| `setAnc(int)` @L759 | `FFCOMMAND_SET_MAF` | 522 |
| `getAnc()` @L102 | `FFCOMMAND_GET_MAF` | 650 |
| `setMaf(int)` @L1030 | `FFCOMMAND_SetMAF` | 1824 |
| `getMaf()` @L403 | `FFCOMMAND_GetMAF` | 1823 |

In other words, **`setAnc()` does not send an ANC command — it sends a MAF command**; the ones that really send `FFCOMMAND_SET_ANC`(513)/`GET_ANC`(641) are another class, `FiilCommandF002` (`com/fiil/sdk/command/b.java`). The two sets coexist and are selected per model: older models use 650/522, newer models such as CC Pro2 use 1823/1824, and the F002-specific page uses 641/513.

**Pitfall 2: ANC and MAF are the same thing on the earbud side.** From the UI, "noise-cancellation level" and "sound-effect mode (MAF = Mode Adjustment Function)" are two panels, but underneath they land on the same command group, and the payload is only 1 byte in both cases.

Level semantics (`adjust_zao_fragment.setZaoInfo(boolean)`, `com/fengeek/main/heat_info_fragment/adjust_zao_fragment.java:509`, verified level by level):

| Value | Meaning | Code behaviour |
|---|---|---|
| 0 | Off | `k0=1`; collapse the panel via `closeItem()`; disable the `U` control; analytics `X("20923","关")` (earType 8) / `X("21023","关")` (earType 5) |
| 1 | Noise cancellation (ANC) | select `T`; disable `S`; analytics `"降噪"` |
| 2 | Monitor / transparency | select `T`; disable `R`; analytics `"监听"` |
| 3 | Open | select `T`; disable `Q`; analytics `"开放"` |
| 4 | Wind (wind-noise reduction) | select `T`; disable the others; analytics `"风中"` |

Where `anc` comes from in the code: `FiilManager.getInstance().getDeviceInfo().getAnc()`; if not connected, or if `getEarType() != this.a1` (the model number bound to the current Fragment does not match), it falls back to the saved value `p0`, to avoid painting another model's state onto the current page. The levels available differ per model (older models only have 0/1/2, CC Pro2 has 4 levels), and the UI disables unsupported buttons according to `earType`.

### 4.3 EQ / sound effects

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_EQ` | 643 (0x0283) | read EQ (generic) |
| `FFCOMMAND_SET_EQ` | 515 (0x0203) | write EQ (generic, 1-byte index) |
| `FFCOMMAND_GetEQ` | 1803 (0x070B) | read EQ (newer models) |
| `FFCOMMAND_SetEQ` | 1804 (0x070C) | write EQ (newer models, 1 byte) |
| `FFCOMMAND_GET_3D` / `SET_3D` | 644 / 516 | 3D sound-effect toggle |
| `FFCOMMAND_GET_MAF` / `SET_MAF` | 650 / 522 | sound-effect mode (shared with ANC) |
| `FFCOMMAND_GetDSP` / `SetDSP` | 1811 / 1812 | DSP presets |

Writing EQ is simply `payload = [index]`; the index table is determined by the `Style` constants on the App side (0 = default/original, 1..N follow the order of the preset list in the App). **The earbud side performs no name mapping** — it only understands numbers.

### 4.4 Volume / playback

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_VOLUME` | 390 | read volume |
| `FFCOMMAND_SET_VOLUME` | 262 | write volume, payload `[1]`=up / `[2]`=down |
| `FFCOMMAND_UP_VOLUME` / `DOWN_VOLUME` | 517 / 518 | up / down |
| `FFCOMMAND_SET_VOLUME_UP` / `_DOWM` | 8197 / 8198 | absolute volume (newer models, with a value) |
| `FFCOMMAND_PALY` / `PAUSE` | 8193 / 8194 | play / pause |
| `FFCOMMAND_NEXT` / `PREVIOUSE` | 8195 / 8196 | next / previous track |
| `FFCOMMAND_SET_PLAY_MODE` / `GET_PLAY_MODE` | 8199 / 8327 | loop mode |
| `FFCOMMAND_GetPlayStatus` / `SetPlayStatus` | 1945 / 1946 | playback status |

### 4.5 Touch / button customization

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GetTouchSensitivity` / `SetTouchSensitivity` | 1919 / 1918 | touch sensitivity |
| `FFCOMMAND_GetCustomButtonFunction` / `Set...` | 1922 / 1921 | single/double/triple tap functions |
| `FFCOMMAND_GetCustomButtonFourHitFunction` / `Set...` | 1936 / 1935 | quadruple-tap function |
| `FFCOMMAND_GetCustomBtn` / `SetCustomBtn` | 1963 / 1962 | custom button |
| `FFCOMMAND_GetPressCustom` / `SetPressCustom` | 1965 / 1964 | long-press customization |
| `FFCOMMAND_GetBeep` / `SetBeep` | 1973 / 1972 | prompt-tone toggle |

Sensitivity payload implementation (`setTouchSensitivity`, `FiilCommandUtil.java:1274-1293`): **two bytes = `[group, threshold]`**, with 6 legal values in total; any other value falls back to `{1,37}`:

| Input | payload | Input | payload |
|---|---|---|---|
| 0 | `1, 44` | 3 | `0, 44` |
| 1 | `1, 37` | 4 | `0, 37` |
| 2 | `1, 31` | 5 | `0, 31` |

Here `31` comes from the obfuscated constant `com.google.firebase.crashlytics.buildtools.reloc.com.google.common.base.a.I` (`a.java:17` → `public static final byte I = 31;`). The first byte splits the 6 levels into two groups (0/1 — most likely left/right earbud or "sensitivity direction"), and only the second byte is the three-level threshold `44 / 37 / 31`. The UI offers a 5-level slider, which does not map strictly one-to-one onto the 6 values actually sent.

### 4.6 Wear detection / power saving

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_WEAR` / `SET_WEAR` | 391 / 263 | wear-detection toggle |
| `FFCOMMAND_EBABLE_WEAR_DETECT` / `DIABLE_WEAR_DETECT` | 12294 / 12295 | the other kind of wear detection |
| `FFCOMMAND_GetWearingState` | 1912 | whether currently worn |
| `FFCOMMAND_GetWearingSwitch` / `Set...` | 1917 / 1916 | wear toggle (new) |
| `FFCOMMAND_GetWearingSensitivity` / `Set...` | 1914 / 1913 | wear sensitivity |
| `FFCOMMAND_GetPowerSavingStatus` / `Set...` | 1832 / 1833 | power-saving mode |
| `FFCOMMAND_GetSleepMode` / `SetSleepMode` | 1975 / 1974 | sleep |
| `FFCOMMAND_GET_SHUTDOWNTIME` / `SET_SHUTDOWNTIME` | 387 / 259 | auto power-off time |
| `FFCOMMAND_GetSimpleMode` / `SetSimpleMode` | 1808 / 1809 | simple mode |

### 4.7 Multi-device / codecs

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_MULTIPOINT` / `SET_MULTIPOINT` | 386 / 258 | one-to-two toggle |
| `FFCOMMAND_GetMultipointConnection` / `Set...` | 1953 / 1952 | multipoint connection (new) |
| `FFCOMMAND_GetPairCount` | 1956 | number of paired devices |
| `FFCOMMAND_SetPair` | 1889 | initiate pairing |
| `FFCOMMAND_SetMutultipointPair` | 1978 | pair a specific multipoint device |
| `FFCOMMAND_SetDisconnectDevice` | 1977 | disconnect a given device |
| `FFCOMMAND_SetExchangeA2DP` / `SetTwoExchange` | 1976 / 1957 | audio-channel switching |
| `FFCOMMAND_GET_APTX` / `SET_APTX` | 392 / 264 | aptX toggle |
| `FFCOMMAND_GetLDACSwitchType` / `SetLDACSwitchType` | 1971 / 1970 | LDAC toggle |
| `FFCOMMAND_GetLDACBuyType` / `SetUnlockLADC` | 1967 / 1968 | LDAC license unlock |
| `FFCOMMAND_GetLowLatency` / `SetLowLatency` | 1825 / 1826 | low latency (game mode) |
| `FFCOMMAND_GetCodingAndDecodingType` | 1954 | current codec |

**LDAC is a paid feature**: `GetLDACBuyType` (1967) reads "whether already purchased", and `SetUnlockLADC` (1968, spelling as in the source) sends the unlock. This shows LDAC is **unlocked by payment per model / per account**, not a plain toggle.

### 4.8 Voice / prompt tones

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_LANGUAGE` / `SET_LANGUAGE` | 388 / 260 | voice-prompt language |
| `FFCOMMAND_NEW_GET_LANGUAGE` / `NEW_SET_LANGUAGE` | 4102 / 4103 | new language protocol |
| `FFCOMMAND_LANGUAGE_HINT` | 775 | language-capability probe |
| `FFCOMMAND_SET_VOIDE_CONTENT` / `GET_VOIDE_CONTENT` | 271 / 399 | custom voice content |
| `FFCOMMAND_SET_VOICE_DIAL_MODE` / `GET_..._STATE` | 272 / 400 | voice wake-up / dialing |
| `FFCOMMAND_PLAY_VOIDE_HINT` | 12305 | play a specific prompt tone |
| `FFCOMMAND_GetNewLanguage` / `SetNewLanguage` | 1815 / 1816 | new language enumeration |

### 4.9 Find / reset

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_SET_SEARCH` / `GET_SEARCH` | 265 / 393 | find earbuds (ring) |
| `FFCOMMAND_SET_RESET` / `SetReset` | 521 / 1818 | factory reset |
| `FFCOMMAND_RESTORE_FACTORY_SETTING` | 16398 | factory reset (the other set) |
| `FFCOMMAND_SetNewReset` | 1818 | reset (shared value) |

### 4.10 Sports / health (Runner etc.)

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_SET_SPORT_START` / `STOP` | 12289 / 12290 | start / end workout |
| `FFCOMMAND_GET_TOTAL_STEP` | 12301 | total steps |
| `FFCOMMAND_GET_ONE_WAY_STEP` | 12302 | single-trip steps |
| `FFCOMMAND_GET_TOTAL_DETAIL_STEP` | 12304 | steps by time period |
| `FFCOMMAND_GET_HISTORICAL_DATA` | 12300 | historical data |
| `FFCOMMAND_SET_TOTAL_ACTIVITY_GOAL` / `GET_...` | 12298 / 12426 | activity goal |
| `FFCOMMAND_GET_ACTIVITY_STATUS` | 12312 | activity status |
| `FFCOMMAND_SET_ACTIVITY_MODE` / `GET_ACTIVITY_MODE` | 12293 / 12421 | activity mode |
| `FFCOMMAND_ACTIVITY_DATA` | 12296 | activity data push |
| `FFCOMMAND_BURN_IN_STATUS` | 16410 | burn-in status |
| `FFCOMMAND_START_TRIAL_MODE` / `END_TRIAL_MODE` | 12308 / 12309 | trial mode (e-commerce / in-store demo) |

### 4.11 Lighting / display

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_LED_COLOR` / `SET_LED_COLOR` | 397 / 269 | RGB light colour |
| `FFCOMMAND_CONFIG_GET_FIIL_LED_MODE` / `SWITCH_...` | 395 / 267 | lighting mode |
| `FFCOMMAND_GET_BRIGHT` / `SET_BRIGHT` | 389 / 261 | brightness |
| `FFCOMMAND_GET_NECKLACE_MODE` / `SET_NECKLACE_MODE` | 398 / 270 | necklace mode (neckband) |

### 4.12 OTA / file transfer

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_START_OTA_UPDATE` | 16397 (0x400D) | start the upgrade |
| `FFCOMMAND_OPEN_FILE` / `CLOSE_FILE` | 16385 / 16388 | open / close file |
| `FFCOMMAND_WRITE_FILE` | 16386 | write file (chunked) |
| `FFCOMMAND_READ_FILE` | 16387 | read file |
| `FFCOMMAND_DELETE_FILE` / `RESET_FILE` / `RESUME_FILE` | 16389 / 16391 / 16390 | delete / reset / resume |
| `FFCOMMAND_GET_DISK_SPACE` | 16399 | free space |
| `FFCOMMAND_ENTER_SYNC_MODE` / `EXIT_SYNC_MODE` | 16392 / 16393 | sync mode |
| `FFCOMMAND_EVENT_NOTIFICATION` | 20484 | event notification |
| `FFCOMMAND_REGISTER_NOTIFICATION` | 20481 | subscribe to notifications |
| `FFCOMMAND_GET_NOTIFICATION` | 20482 | read notification |
| `FFCOMMAND_CANCEL_NOTIFICATION` | 20483 | cancel notification |

GAIA natively also has a DFU command set (`COMMAND_DFU_BEGIN=1585` / `DFU_WRITE=1586` / `DFU_COMMIT=1587` / `DFU_REQUEST=1584`), plus storage-partition commands (`COMMAND_OPEN_STORAGE_PARTITION=1553`, `COMMAND_WRITE_STORAGE_PARTITION=1557`, `COMMAND_MOUNT_STORAGE_PARTITION=1562`); on the Airoha side there is additionally a separate OTA stack, `com.airoha.libfota155x`.

### 4.13 User information / cloud

| Command | Value | Description |
|---|---|---|
| `FFCOMMAND_GET_DEVICE_ID` / `GET_USER_ID` | 4097 / 4226 | device / user ID |
| `FFCOMMAND_GET_USERINFO` / `SET_USERINFO` | 4229 / 4101 | user profile |
| `FFCOMMAND_GET_USER_SECURITY_CODE` / `SET_...` | 4108 / 4109 | security code |
| `FFCOMMAND_SET_TIME` / `SET_TIME2` / `NEW_SET_TIME` | 4104 / 4105 / 4110 | time sync |
| `FFCOMMAND_GET_CUSTOM_DEVICE_NAME` | 4100 | custom device name |
| `FFCOMMAND_GET_DEVICE_SETTING` / `SET_DEVICE_SETTING` | 4106 / 4107 | device-setting package |
| `FFCOMMAND_GET_CONFIG_SDS` / `SET_CONFIG_SDS` | 396 / 268 | SDS configuration |
| `FFCOMMAND_NETWORK_ALERT` | 16402 | network alert |
| `FFCOMMAND_VOICE_RECOGNITI` | 16403 | voice recognition |
| `FFCOMMAND_VOIDE_PROMOTE` | 16421 | voice prompt |

---

## 5. Model table (earType → model)

The private method `private void x(Bundle bundle)` in `MainActivity.java` (starting at line 1902) corresponds one-to-one with the `HeatSetMode` enum in `Config.java`. **The earType reported by the earbuds determines the entire UI shape**.

| earType | Enum | Model (actual text from strings.xml) |
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

> Note that models 47/48 (Atom / GS Links) are the **only ones that enable the Doubao 0x55AA voice frames**.

---

## 6. Feature landscape: what the App can actually control on the earbuds

Organized by UI module (from the Chinese strings in `res/values/strings.xml` + the individual Fragments):

1. **Connection management**: scanning, pairing, pull-to-reconnect, disconnect, multi-device switching, battery/charging state, serial number, firmware version.
2. **Noise cancellation**: off / ANC / transparency (monitor) / open / wind, 5 levels, with "adaptive" on some models.
3. **EQ sound effects**: presets (original/pop/rock/classical/vocal/bass boost…), custom EQ (multi-band) + 3D sound effects, DSP presets.
4. **Touch settings**: single/double/triple/quadruple tap and long press bound individually (play-pause, next/previous track, volume up/down, voice assistant, ANC switching, no action), touch sensitivity in 5 levels.
5. **Wear detection**: on/off, auto-pause when removed, wear sensitivity.
6. **Volume**: earbud-side volume up/down, absolute volume, volume prompts.
7. **Low latency / game mode**: the `GetLowLatency` toggle.
8. **Codecs**: aptX, LDAC (including "purchase to unlock"), AAC/SBC display, connection type.
9. **Multipoint connection**: one-to-two toggle, paired-device list, targeted disconnect, A2DP channel swap.
10. **Find earbuds**: make the earbuds ring, left and right separately.
11. **Prompt tones / voice**: language (Chinese/English…), prompt-tone toggle, custom voice content, voice wake-up / dialing.
12. **Lighting** (models with lights): colour, mode, brightness, necklace mode.
13. **Auto power-off**: 15/30/60 minutes / never.
14. **Power saving / sleep / simple mode**.
15. **Sports and health** (Runner etc.): step counting, workout start/end, step goal, historical data, real-time announcements, burn-in.
16. **OTA upgrade**: full-package differential, resumable transfer, progress reporting, controls disabled during upgrade.
17. **Offline media** (some models): built-in earbud storage; write/read/delete music files, free space, sync mode (the `OPEN_FILE`/`WRITE_FILE`/`CLOSE_FILE` family).
18. **Factory reset / reset / trial mode (Trial)**.
19. **Doubao AI voice** (Atom / GS Links): voice wake-up, ASR uplink, TTS downlink, over `0x55AA` frames.
20. **Debug / logging**: `savelog` analytics, `FFCOMMAND_GET_STATUS` heartbeat.

---

## 7. How to send a command yourself (reproducible)

### 7.1 Building a GAIA frame in Python

```python
import struct

SOF = 0xFF
VENDOR_FIIL = 0x0234

def gaia_frame(cmd: int, payload: bytes = b"", new_proto=False) -> bytes:
    ver = 0xF1 if new_proto else 0x01          # 0x01 legacy protocol / 0xF1 new protocol
    return bytes([SOF, ver, 0x00, len(payload)]) \
         + struct.pack(">H", VENDOR_FIIL) \
         + struct.pack(">H", cmd) \
         + payload

# 1) read the model number (determines the subsequent UI/command set)
pkt = gaia_frame(777)                       # FFCOMMAND_GET_EARTYPE
assert pkt == bytes.fromhex("ff01000002340309")   # 0x0309 = 777

# 2) read battery level
pkt = gaia_frame(771)                       # FFCOMMAND_GET_BATTERY_LEVEL  → ff01000002340303

# 3) set noise cancellation: 522 for older models / models that route through setAnc(); 1824 for newer models such as CC Pro2
pkt = gaia_frame(522, b"\x01")              # FFCOMMAND_SET_MAF (setAnc)   noise cancellation
# pkt = gaia_frame(1824, b"\x01")           # FFCOMMAND_SetMAF (setMaf)    newer models
# pkt = gaia_frame(513,  b"\x01")           # FFCOMMAND_SET_ANC (F002-specific)

# 4) set EQ = index 3
pkt = gaia_frame(515, b"\x03")              # FFCOMMAND_SET_EQ  → ff01000102340203 03
# newer models: gaia_frame(1804, b"\x03")   # FFCOMMAND_SetEQ

# 5) volume +
pkt = gaia_frame(517)                       # FFCOMMAND_UP_VOLUME → ff01000002340205
```

> The most reliable criterion for whether a command number should take the "old" or the "new" route is to **first read out the earType with 777** (§5 model table), and then check whether that model's UI goes through the `HeatSetPagerXXXFragment` pages or the new `f0xx` package — the two page families use different command families. In practice, the faster approach is to capture a btsnoop of the App once and see which command it sends when you press the button.

### 7.2 Parsing a response

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

### 7.3 Verifying with a phone-side packet capture (recommended path)

Because SPP is **standard RFCOMM**, the most reliable way to verify is not to emulate the earbuds but to **capture the link between the phone and the earbuds**:

1. Android developer options → enable "Bluetooth HCI snoop log" (`btsnoop_hci.log`);
2. Operate the App (switch ANC, switch EQ, adjust volume);
3. Export `/sdcard/btsnoop_hci.log` and open it in Wireshark;
4. Filter on `btbrlmp` / `btspp` and look directly at the RFCOMM payload — every `FF 01 00 ...` frame above will appear verbatim;
5. Cross-check against §3.1 and the §4 command tables in this report to derive the bytes behind each operation.

BLE links (F042/Airoha) show up in the same btsnoop capture too (filter on ATT); you just need to filter characteristic-value writes by the two UUIDs `13fa60a2` / `3d4dac58`.

### 7.4 Connecting directly from a PC (L2CAP/RFCOMM)

On Linux you can use:

```bash
# look up the SPP channel number on the device
sdptool browse XX:XX:XX:XX:XX:XX
# the channel number is usually 1 or 2
rfcomm connect /dev/rfcomm0 XX:XX:XX:XX:XX:XX 1
# write the assembled bytes above to /dev/rfcomm0
```

Windows has no native rfcomm command line; you can use Python `pybluez` or `bleak` (for BLE models).

> ⚠️ Note: link B (Bluetrum) **has the phone acting as the server**, so the PC cannot actively connect to the earbuds — you have to flip it around and let the earbuds connect to the PC, which requires emulating the SPP server for `d52b47bc-2dea-42c4-b26d-20252fceeeb3`, and the earbuds must first be unbound from their pairing record with the phone; in practice this is very hard to reproduce. This path is better suited to packet capture than to building your own connection.

---

## 8. A few design points worth noting

1. **Reading constants via reflection**: `getGaiaIntValue("VENDOR_FF")` / `getGaiaByteValue("SOF")` (inside `FiilCommandUtil`, ultimately landing in `com.fiil.sdk.utils.ReflectUtils`'s `Class.forName(...).getDeclaredField(...)`). The author deliberately keeps constants and command numbers out of the call sites, raising the cost of static analysis (though it is useless against Frida/dynamic debugging).
2. **Command dispatch via a string table**: `Config.sMap` (declared at `com/fiil/sdk/config/Config.java:155`, 145 `put`s from `:185` onward) is a map of `commandId → "analysisXxx"`, reflectively invoked after a response is received. Adding a new command just means adding one map line + one method.
3. **`isGaiaConnect()` blanket guard**: in `FiilManager` almost every setter starts with
   ```java
   if (!this.f34440a.isGaiaConnect() || getDeviceInfo().isUpdate()) return;   // e.g. FiilManager.java:1319, :1127
   ```
   i.e. **all control is forbidden during an upgrade**, to avoid bricking the earbuds by sending commands during OTA. `isGaiaConnect()` appears **362 times** across the whole project, which shows how deeply this guard is applied.
4. **Command queue**: `com.fiil.sdk.queue.CommandQueue` (note the package is `queue`, not `command`) uses `new LinkedBlockingQueue<>(64)` (`CommandQueue.java:91`) for serial sending; each command has a 3-second timeout (`com/fiil/sdk/command/a.java:13` → `protected int timeOut = 3000;`), and on timeout the failure callback runs. **Do not blast commands concurrently** — packets will be dropped.
5. **The model determines the capabilities**: `earType` is not just "a name to display"; it determines which toggles exist in the UI, how many ANC levels there are, whether LDAC is supported, and whether the Doubao link is used. **The same command table exposes completely different usable subsets on different models.**
6. **LDAC is a paid unlock**: `GetLDACBuyType` / `SetUnlockLADC` show that there is a commercial "buy it before you can enable it" logic, not a purely local toggle.
7. **Two "ANC" command sets coexist**: `MAF(650/522)` and `ANC(641/513)`. If you only look at one during reverse engineering, you will miss half the models.
8. **Battery level travels in the advertisement**: V1/V2 Beacons directly advertise the battery level and charging state of both earbuds plus the case, which is why the App can show battery levels in the list **before it has connected** — this is how "show battery even when not connected" is implemented.

---

## 9. Compliance and boundary notes

- This report is solely for **understanding how devices you own are controlled**, and for interoperability research (e.g. building an alternative App, doing automated testing, writing an open-source driver).
- **Do not** use it for: cracking paid features (such as the LDAC unlock), forging OTA packages, bypassing device authentication, or mass-flashing other people's devices.
- Capture only the link between **your own devices and your own phone**; BLE/SPP capture does not involve other people's data.
- If you want to build a third-party App: FIIL's GAIA implementation is a modification of the CSR GAIA framework, and some command semantics match the public CSR documentation, but `FFCOMMAND_*` are vendor-private extensions with **no public specification** — they can only be determined experimentally.

---

## 10. Index of key files (for further digging)

| Topic | File |
|---|---|
| Complete GAIA constant set | `jadx_out/sources/com/fiil/sdk/gaia/father/Gaia.java` |
| Frame construction | `jadx_out/sources/com/fiil/sdk/command/FiilCommandUtil.java:517` (`getSendBuffer`)|
| Frame definition (elsewhere) | `jadx_out/sources/com/fiil/sdk/gaia/father/Gaia.java:456` (`frame(...)`)|
| Frame parsing / dispatch | `jadx_out/sources/com/fiil/sdk/gaia/father/a.java` |
| Command method table | `jadx_out/sources/com/fiil/sdk/config/Config.java:155` (`sMap` declaration), `:185-329` (145 `put`s)|
| Command queue | `jadx_out/sources/com/fiil/sdk/queue/CommandQueue.java` (capacity 64 at `:91`) |
| Public API (200+) | `jadx_out/sources/com/fiil/sdk/manager/FiilManager.java` |
| SPP socket | `jadx_out/sources/com/fiil/sdk/gaia/father/b.java` |
| Connection entry (older models) | `FiilManager.java:275 connecting(String)` |
| Connection entry (CC Pro2 / F039) | `FiilManager.java:259 FIILCCPro2Connecting(String)` |
| Connection entry (quick connect) | `FiilManager.java:1036 quickConnect()` |
| GAIA send/receive facade | `jadx_out/sources/com/fiil/sdk/gaia/father/d.java` (`GaiaService`, static field `f34410a` is the GaiaLink)|
| Connection-success callback / handshake | `jadx_out/sources/com/fiil/sdk/c/a.java:116-149` (`ConnectHandler.handleMessage`)|
| BLE scanning | `jadx_out/sources/com/fengeek/main/ble/b.java` |
| brandId filtering | `jadx_out/sources/com/fengeek/main/f040/amd/viewmodels/g0.java:133`, `f039new/amd/viewmodels/f0.java:131` |
| headsetId injection | `HeatSetPagerCCPro2Fragment.java:510` (39), `F40MainFragment.java:299` (40), `F041MainFragment.java:296` (41)|
| Beacon parsing | `jadx_out/sources/com/bluetrum/devicemanager/models/DeviceBeacon*.java` |
| Bluetrum assembly | `jadx_out/sources/com/bluetrum/devicemanager/z.java` |
| Bluetrum SPP server | `jadx_out/sources/com/bluetrum/devicemanager/bluetooth/BluetoothSppService.java` |
| F042 BLE assembly | `jadx_out/sources/com/fengeek/main/ble/FIILPeripheral.java:444` |
| Doubao 0x55AA | `jadx_out/sources/com/fengeek/main/i/d/e.java` (`setDouBao`) |
| Doubao receive state machine | `jadx_out/sources/com/fengeek/main/i/b.java` (`c(byte[])`) |
| Model mapping | `jadx_out/sources/com/fengeek/f002/MainActivity.java:1902` |
| Model display text | `apktool_out/res/values/strings.xml` |
| Airoha UUIDs | `jadx_out/sources/com/airoha/liblinker/.../a.java` |
| Airoha Race packet | `jadx_out/sources/com/airoha/android/lib/e/b/a.java` |

---

*Basis for this report: jadx 1.5.6 decompilation of `fiil+_3.5.2.apk`; every conclusion carries source line numbers and can be checked item by item.*
