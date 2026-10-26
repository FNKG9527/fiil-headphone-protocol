[简体中文](PROTOCOL_FIIL.md) | **[English](PROTOCOL_FIIL.en.md)**

# FIIL Earbud Communication Protocol Specification (PROTOCOL_FIIL.md)

> This document is the **master protocol specification**. It grew out of the implementation of a third-party Android control module, and every entry comes from the decompiled source of **FIIL+ 3.5.2 (`com.fengeek.f002`, versionCode 175, minSdk 26, compileSdk 33)**, with the file and line number cited for each item.
>
> The companion full reverse-engineering report is `docs/REVERSE_ENGINEERING_REPORT.en.md`; this document keeps only the parts "required to implement a controller", and adds the **Mate protocol family (the third command namespace)** found in the second round.
>
> **About the word "module" in this document**: it means a third-party Android controller that the author wrote to verify this protocol (experimental, not distributed with this repository). Wherever "module implementation / module strategy" appears, it means **the trade-offs made by that controller**, not a requirement of the protocol itself — you may decide differently in your own architecture.

---

## 0. Key findings at a glance

| Fact | Finding |
|---|---|
| Transport layer | **Dual channel**: classic Bluetooth SPP (RFCOMM) + BLE GATT (`ff17` write / `ff18` receive). **The frame format is exactly the same on both**; only the carrying method differs |
| Why BLE is needed | Measured on FIIL Key Max: standard SPP connects and then goes **silent** (zero responses), and a custom UUID connection is **kicked repeatedly** (`scn:6 opened=21 leaked=21`). BLE GATT is stable and usable |
| SPP UUID | Older models `00001101-0000-1000-8000-00805F9B34FB`; newer models (including Mate) `d52b47bc-2dea-42c4-b26d-20252fceeeb3` |
| BLE UUID | Write `0000ff17-0000-1000-8000-00805F9B34FB` (Write No Response); receive `0000ff18-…`. **Not ff16** |
| Frame format | **Two sets coexist**: GAIA (starts with `0xFF`) and Mate (starts with `0x00`). They are demultiplexed by first byte on the same channel |
| Command namespaces | Three: the GAIA 16-bit command word (`0x02xx`/`0x07xx`), Mate's classifyId + cmdId, and Doubao `0x55AA` (a few models) |
| **GET and SET have different semantics** | GET: `cid=39` fixed + cmdId holds the query code; SET: `cid=` command family + **cmdId is always 0**. See §3.12 |
| Connection direction | The phone is the **RFCOMM client** (`createRfcommSocketToServiceRecord`) / **GATT client** (`connectGatt`) |
| Handshake | After a successful connection **the first frame must be `setUnlock`** (`00 27 01 00 04 FF 02 02 34`), otherwise the earbuds stay silent for every query |
| Response classification | The `type` byte (offset 2): `0x01`=REQ / `0x02`=RESP / `0x03`=NOTIFY. **The old implementation mistook it for a version number and hard-coded `discard if !=0x01`, so every response was thrown away** |

---

## 1. Transport layer and UUIDs

### 1.1 Connection methods

FIIL earbuds have **two completely independent transport layers**; the frame encoding is identical and only the carrying method differs:

| Transport layer | Medium | Applicable | Module implementation |
|---|---|---|---|
| **RFCOMM / SPP** | Classic Bluetooth serial port | All models (the App uses only this one) | `FiilRfcommController.openSocket()` |
| **BLE GATT** | BLE characteristic read/write | Mate models (the only stable one in testing) | `FiilGatt` |

**SPP path**: the App actively connects to the earbuds with `BluetoothDevice.createRfcommSocketToServiceRecord(UUID)` (as a client), then does raw reads and writes through the `BluetoothSocket`'s `InputStream`/`OutputStream`.

Three-level fallback on the module side (**key point: `createRfcommSocketToServiceRecord()` only creates a socket object; it performs no SDP and reports no error** — the real link is established in `connect()`):

| Order | Method | Applicable |
|---|---|---|
| 1 | `createRfcommSocketToServiceRecord(UUID_SPP)` | Older models |
| 2 | `createRfcommSocketToServiceRecord(UUID_CC_PRO2)` | Newer models such as F045~F053 |
| 3 | Reflection `createRfcommSocket(1)` (channel 1) | When both of the above fail |

> **Hands-on lesson**: the old implementation "returned as soon as the socket object was created", so the standard SPP path always returned first, `connect()` was bound to fail on Mate models, and the custom-UUID path was never reached. The correct approach is to **create all three candidates first, then `connect()` them one by one**, keep the first that succeeds and close the rest. See `FiilRfcommController.openSocket()`.

**Why BLE GATT is still needed** (measured on 2026-10-25, connecting directly from a PC with Bleak to FIIL Key Max `B0:F1:A3:00:00:01`):

1. Standard SPP (`00001101-…`) **can `connect()` successfully but the earbuds stay completely silent** — zero responses to every query;
2. The custom UUID (`d52b47bc-…`) connects and does answer, but on the Android side RFCOMM gets kicked by the earbuds over and over (`dumpsys bluetooth_manager` shows `scn:6 opened=21 closed=0 leaked=21`);
3. BLE GATT (ff17 write / ff18 receive) **is stable, and the Mate frame format is byte-for-byte identical to SPP**.

So the module's strategy is: try SPP first → if all three RFCOMM attempts fail, or 3.5 seconds pass with zero responses after connecting → switch to GATT automatically.

### 1.2 Full UUID table

| Constant | Value | Notes |
|---|---|---|
| `UUID_SPP` | `00001101-0000-1000-8000-00805F9B34FB` | Standard serial port service, older models |
| `UUID_CC_PRO2` | `d52b47bc-2dea-42c4-b26d-20252fceeeb3` | FIIL custom SPP, used by all of F045~F053 (all 16 `createRfcommSocketToServiceRecord` call sites in `F045~F053MainFragment` use it) |
| `FiilGatt.CHAR_WRITE` | `0000ff17-0000-1000-8000-00805F9B34FB` | **BLE GATT write characteristic** (App → earbuds), using Write Without Response |
| `FiilGatt.CHAR_NOTIFY` | `0000ff18-0000-1000-8000-00805F9B34FB` | **BLE GATT notify characteristic** (earbuds → App) |
| F042 BLE Service | `13fa60a2-429a-4a90-a3e8-d4c2b9a17655` | F042 series only (Airoha chipset), uses a private BLE channel |
| F042 Write/Notify | `3d4dac58-7d68-4734-8a71-21…` | One characteristic both writes and receives |

> **Note on the ff16 / ff17 / ff18 trio**: in practice **writes go to ff17**, not ff16. Assuming ff16 by analogy with common Nordic designs made every write return false and the earbuds did not react.
>
> **Unconfirmed item**: the full 128-bit value of the F042 BLE characteristic is truncated in the decompiled artifacts, so the module does not implement that link for now (F042 takes the `BLE_ONLY` branch and is not handled yet).

### 1.3 BLE GATT channel details (measured)

| Item | Value |
|---|---|
| Write characteristic | `0000ff17-…` (`PROPERTY_WRITE_NO_RESPONSE`, must be used; Write With Response often returns `GATT_BUSY`) |
| Notify characteristic | `0000ff18-…` |
| CCCD | `00002902-…`; in practice **notifications arrive even without writing it** (`setCharacteristicNotification(true)` is enough), but the module still tries to write it for other ROMs |
| Frame format | Exactly the same as SPP (see §3.2), no encapsulation needed |
| Handshake | `setUnlock` must be sent first (cid=39 / cmd=0xFF / payload `02 34`), otherwise the earbuds stay silent |
| Write interval | 25 ms (writing too fast drops frames) |
| Notification fragmentation | A single frame may be split into several packets and must be reassembled by the length field described in §3.2 — see `FiilGatt.handleRx()` |

---

## 2. GAIA frame format (the first command namespace)

### 2.1 Frame structure

Source: `com\fiil\sdk\command\FiilCommandUtil.java:517 getSendBuffer(int vendor, int cmd, boolean newProto, byte... payload)`
together with the offset constants in `com\fiil\sdk\gaia\father\Gaia.java:344-359` (`OFFS_SOF=0`, `OFFS_VERSION=1`, `OFFS_FLAGS=2`, `OFFS_PAYLOAD_LENGTH=3`, `OFFS_VENDOR_ID_H=4`, `OFFS_VENDOR_ID_L=5`, `OFFS_COMMAND_ID_H=6`, `OFFS_COMMAND_ID_L=7`, `OFFS_PAYLOAD=8`, `SOF=-1`, `VENDOR_FF=564=0x0234`, `PROTOCOL_VERSION=1`, `NEW_PROTOCOL_VERSION=241=0xF1`).

```
offset len field         value / notes
 0     1   SOF           0xFF
 1     1   version       0x01 (legacy protocol) / 0xF1 (new protocol)
 2     1   flags         0x00
 3     1   payload_len   = payload.length (**payload only, excluding vendor and command**)
 4     2   vendor_id     0x0234 (big-endian, FIIL vendor ID)
 6     2   command       (cmd & 0x7FFF), high bit 0x8000 = ACK frame
 8     n   payload       command payload
```

> Key point: `payload_len` equals **the payload length only**; inside `getSendBuffer` it is `bArr2[OFFS_PAYLOAD_LENGTH] = bArr.length`, while the total frame length = `8 + payload.length`. This is very easy to get wrong (it was once mis-remembered as "including vendor + cmd"); trust the source.

### 2.2 Common frame examples

| Operation | Complete bytes | Notes |
|---|---|---|
| Get version | `FF 01 00 00 02 34 03 01` | cmd = 769 = `0x0301`, empty payload |
| Get model | `FF 01 00 00 02 34 03 09` | cmd = 777 = `0x0309` |
| Get battery | `FF 01 00 00 02 34 03 03` | cmd = 771 = `0x0303` |
| Get ANC | `FF 01 00 00 02 34 02 81` | cmd = 641 = `0x0281` |
| Set ANC | `FF 01 00 01 02 34 02 01 [mode]` | cmd = 513 = `0x0201` |
| Set EQ (legacy) | `FF 01 00 01 02 34 02 03 [idx]` | cmd = 515 = `0x0203` |
| Set EQ (V2) | `FF F1 00 0C 02 34 07 0C [0][type][10 bands]` | cmd = 1804 = `0x070C`, 12-byte payload |
| Set 3D | `FF 01 00 01 02 34 02 04 [0/1]` | cmd = 516 = `0x0204` |
| Low latency | `FF 01 00 01 02 34 07 22 [level]` | cmd = 1826 = `0x0722`, level 0=music / 1=game / 2=video (see §3.7) |
| Multipoint (Smart Connect) | `FF 01 00 01 02 34 01 02 [0/1]` | cmd = 258 = `0x0102` |
| Find earbuds | `FF 01 00 01 02 34 01 09 01` | cmd = 265 = `0x0109` |

> The 4th byte in the table is exactly `payload_len`; cross-check it against the formula in §2.1 (for example, the payload of Set ANC is `[mode]`, 1 byte, so the 4th byte = `01`).

### 2.3 Response parsing

Source: the `analysisXxx` methods in `com\fiil\sdk\gaia\GaiaSdkCommand.java`.

- For the vast majority of commands, **`payload[1]` is the effective value** (`payload[0]` is a status/reserved field).
- **Battery level is the exception**: `payload[2]` is a **decimal digit string** (e.g. `"85"`), and the App adds `+1` before displaying it. See `FiilGaia.parseBatteryLevel()`.
- Charging-case battery uses a different command (`GET_BATTERY_OF_BOX = 1915`); there `payload[1]` is the percentage directly, **with no +1**.
- Wear state: left earbud `payload[2]`, right earbud `payload[1]`.

### 2.4 Full table of GAIA command values

The full set of 190 entries is in `docs/reference/_cmd_table.txt`; the module uses only about 60 of them from `FiilCommands.kt`. Key values:

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

## 3. Mate frame format (the third command namespace, F045~F053 only)

### 3.1 Why a third set exists

Models after F045 (GS Lite / Key Max / Atom / GS Links / Key Pro2 / CC3 / CC Pro3) use a **completely new command layer** under the `com.fengeek.main.i` package: the command is no longer a 16-bit `0x02xx` value but two 8-bit values, "classifyId (category) + commandId (sub-command)".

### 3.2 Frame structure

Source: `com\fengeek\main\i\b.java:237 writeMate(FiilBaseCommand, handle)`

```
no sub-command (commandId == 0):
  [0]=0x00  [1]=classifyId  [2]=0x01  [3]=0x00  [4]=payloadLen  [5..]=payload
  total frame length = 5 + payloadLen

with sub-command (commandId != 0):
  [0]=0x00  [1]=classifyId  [2]=0x01  [3]=0x00
  [4]=payloadLen+2  [5]=commandId  [6]=payloadLen  [7..]=payload
  total frame length = 7 + payloadLen
```

- **No checksum beyond SOF**, no vendor, no ACK bit.
- Written straight to SPP with a bare `write(byte[])`, **not wrapped in a GAIA shell**.
- Distinguished from GAIA frames by the first byte: `0xFF` = GAIA, `0x00` = Mate.

Module implementation: `FiilMate.buildFrame()` / `FiilMate.parse()` / `FiilMate.drain()`.

### 3.3 classifyId table

Source: `com\fengeek\main\i\d\c.java`

| classifyId | Name | cmdId | payload | Line |
|---|---|---|---|---|
| 1 | `SET_MULTIPOINT` Smart Connect switch | **1** | `[0/1]` | c.java:321 |
| 32 | `SET_EQ` EQ write | none | `[reserved][type][N bands]` | c.java:338 |
| 33 | `SET_MUSIC` media control | see §3.6 | none | c.java:420 |
| 37 | `SET_LOW_LATENCY` low latency | none | `[0/1]` | c.java:388 |
| 44 | `SET_MAF` noise-cancellation mode | none | `[mode]` | c.java:404 |
| 50 | `SET_SPATIAL_AUDIO` cinema mode | none | `[0/1]` | c.java:504 |
| 38 | `SET_WEAR` wear-detection switch | **none** | `[0/1]` | g.java:663 / h.java:729 |
| 39 | `QUERY` query group | see §3.4 | none | c.java:75~267 |
| **40** | **`NOTIFY` earbud-initiated report (receive only)** | see §3.4.2 | pushed by earbuds | `*MainFragment` notify branches of each model |

> **Pitfall 1**: `SET_MULTIPOINT` is the **only** setting command that carries a `commandId` as well (`setCommandClassifyId(1)` + `setCommandId(1)`); every other setting command has only a `classifyId`. It was once wrongly judged to have "no cmdId".
>
> **Pitfall 2**: Mate's `commandId` is a **signed byte** in the source, so values above 127 are written as negative numbers (`-128`→128, `-118`→138, `-117`→139, `-114`→142, `-113`→143, `-112`→144, `-126`→130). Always `and 0xFF` when building a frame.
>
> **Pitfall 3 (easiest to miss; it alone decides whether state sync works)**: responses come over two channels. `cid=39` is a **query response** and `cid=40` is an **earbud-initiated report**. Across the whole codebase the actual arguments to `setCommandClassifyId(...)` are only 1/2/3/32/33/34/36/37/38/39/41/44, **40 never appears**; yet the notify branches of all 7 models' `*MainFragment` **test `getCommandClassifyId() == 40` without exception** (F045:306/310/314, F046:299/303/308/317, F047:303/307/314/323/333, F048:361/365/372/381, F049:303/307/314/323/333/343, F051:300/304/307/316/323/333/343, F053 same as F051). **Handling only cid=39 means: when the user changes ANC with the earbud button, takes the earbuds off, or the battery level changes, nothing is received and the module's state drifts away from the earbuds.** See `FiilMate.Cid.NOTIFY` and `FiilRfcommController.applyMateNotify()` for the module implementation.

### 3.4 Sub-commands of QUERY(39)

| cmdId | Name | Response |
|---|---|---|
| 1 | Battery | `payload[1]` = percentage (0/255 treated as 0) |
| 2 | Version | string |
| 4 | EQ state | 12 bytes: `[0]` reserved, `[1]` type, `[2..11]` 10 band gains |
| 5 | Button configuration | — |
| 6 / 138 | Right / left earbud UUID | — |
| 7 | Playback state | `payload[0]==1` = playing |
| 8 | Low-latency state | `payload[0]==1` = on |
| 12 | MAF state | `payload[0]` ∈ 0..4 |
| 24 | Cinema mode state | `payload[0]==1` = on |
| 128 | Multipoint state | `payload[0]==1` = on |
| 130 | Sound effects | — |
| 142 / 143 / 144 | LDAC unlock / switch / codec | — |

> **Note the relationship between cmdId=6 and 139**: in the source the right-earbud UUID is `NetError.ERR_BAD_SSL_CLIENT_AUTH_CERT` = **-117**, which on the wire is `0x8B` = **139**, not 6. Left earbud `-118` → **138**. This was once written as 6 in the module and has been corrected.

### 3.4.1 Command coverage differences per model (measured across 7 model classes)

The same classifyId set is not implemented by every model. Coverage comparison (`-` = that class has no such method):

| Capability | F045 | F046 | F047 | F048 | F049 | F051 | F053 |
|---|---|---|---|---|---|---|---|
| Battery/EQ/low latency/multipoint/version/buttons/playback | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| MAF five states (cid 44) | ✗ | ✓ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Cinema mode (cid 50) | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| EQ adaptive (1/14) | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Noise adaptive (1/16) | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Wind-noise adaptive (1/17) | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |
| Wear detection (cid 38/39) | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| Language setting (cid 41) | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| LDAC switch (1/13) | ✗ | ✓ | ✗ | ✓ | ✓ | ✓ | ✓ |
| Recording/storage (F048 only) | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ |
| Primary/secondary earbud / earbud UUID | ✗ | ✓ | ✗ | ✓ | ✗ | ✗ | ✗ |
| Paired-device list management | ✗ | ✗ | ✓ | ✗ | ✓ | ✓ | ✓ |
| Tuning (getTuning/setTuning) | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |
| Sleep aid (setHelpSlep) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Doubao (setDouBao) | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ |

> Evidence method: under `jadx_out/sources/com\fengeek\main\i\d\` the files b~h.java are the 7 model command classes; the **real method declarations** were exported with `^\s*public (void|boolean|int|byte\[\]|String) (\w+)\(` (not a text grep, to avoid hitting comments and string literals) and then compared column by column.
>
> Binding between the module and the command classes (full grep of `i.d.X.getInstance`): **f045→b, f046→c, f047→d(+b), f048→e, f049→f, f051→g, f053→h**.
>
> Conclusion: **cinema mode (`setSpatialAudio` / cid 50) is in practice supported by F046 (Key Max) only** — `setSpatialAudio` is defined only in `c.java`, and its only call sites in the whole codebase are `F046StyleFragment.java:560/842/897/996`. The module's `supportsCinema()` must decide precisely by earType (`setOf(46)`) and cannot rely on `isMateDialect`.
>
> **The MAF five states are not universal across Mate models either**: `setMAF` (cid 44) exists only in `c/f/g/h.java`, i.e. **F046/F049/F051/F053**; F045 (GS Lite), F047 (Atom) and F048 (GS Links) use Mate frames but **have no noise-cancellation control command** (those three models do not have ANC at all). The module's `supportsMateMaf()` decides by `setOf(46,49,51,53)` and the UI hides the noise-cancellation card for those three models rather than sending a command that is bound to get no answer.
### 3.4.2 NOTIFY(40) sub-commands (earbuds → phone)

| cmdId | Meaning | Response parsing | Evidence |
|---|---|---|---|
| 1 | Battery | `payload[1]` = percentage | F046:299 |
| 7 | Playback state | `payload[0]` | F046:303 → `playState()` |
| 8 | Low latency | `payload[0]==1` = on | F046:308 → `setDeferState()` |
| 9 | Wear state | `payload[0]==1` = worn (single value, no left/right distinction) | F051:316 / F053:316 |
| 12 | MAF mode | `payload[0]`, with 5→3 and 6→4 needing normalization | F046:317; F051:343-349 |
| 136 | Smart Connect device list | `type=0` | F051:323 |
| 137 | Smart Connect result | `type=1`, `payload[0]==2` means timeout | F051:333 |

> **Same cmd number, different meaning**: `cmdId=8` is the response to "read low-latency state" under QUERY, and the "low-latency state changed" push under NOTIFY; `cmdId=1` is the "battery query response" under QUERY and the "battery level changed" push under NOTIFY. The cmdIds of the two channels must be **parsed from separate tables** and cannot share one `when`.
>
> **5/6 normalization for cmdId=12**: `F051MainFragment.java:344-349` rewrites `MAF == 6` to 4 and `== 5` to 3, which shows that newer models have two additional levels, 5/6, beyond the MAF five states. The module's `applyMateNotify()` already mirrors that mapping.


### 3.5 MAF levels (Mate)

From the icon bindings of `F046StyleFragment.java:1037 setMafModeUI(int)` and the label order in `apktool_out\res\layout\fragment_f49_style.xml:36-45`:

| Value | Meaning | Selected icon |
|---|---|---|
| 0 | Off | `f046_maf_off_select` |
| 1 | Noise cancelling | `f046_noise_select` |
| 2 | **Open** | `f046_open_select` (`fiil_cc_pro2_open`) |
| 3 | **Awareness (listen)** | `f046_listen_select` (`fiil_cc_pro2_listen`) |
| 4 | Wind | `f046_wind_select` |

**There is no 5th level.** FIIL calls the whole block "MAF intelligent noise tuning" (the `tv_headset_sound2` string in `fragment_f49_style.xml:35`); "intelligent" there refers to the noise-cancellation algorithm itself, not to an extra level. The `f49/f51/f53` layouts contain four other **independent switches** ("environment-adaptive noise cancelling / adaptive wind-noise reduction / ear-canal-adaptive noise cancelling / adaptive EQ tuning", cid=1, cmd=16/17/15/14, see §3.7), which sit alongside the MAF levels rather than being one of them.

**The number of levels differs by model**:

| Model | Levels | Basis |
|---|---|---|
| F046 Key Max / F049 Key Pro2 / F053 CC Pro3 | 5 | `setMafModeUI` handles 0~4 |
| F051 CC3 | **3** (off / noise cancelling / open) | `F051StyleFragment.java:800` handles only 0/1/2; `fragment_f51_style.xml` has only three labels |
| F045 GS Lite / F047 Atom / F048 GS Links | 0 | The command class has no `setMAF`; the models have no ANC |

> Note that the ANC semantics **differ** from GAIA models: GAIA is `0 off / 1 noise cancelling / 2 listen / 3 open / 4 wind` (see `heat_info_fragment\adjust_zao_fragment.java:509 setZaoInfo`, where `anc==3` lands on "open" and `anc==4` lands on "wind"). **The meanings of 2 and 3 are exactly opposite on the two sides** (Mate 2=open 3=listen; GAIA 2=listen 3=open), and the module renders them separately with `FiilModels.ancLabel()` and `mateMafLabel()`.

**5/6 normalization on responses**: `F049MainFragment.java:344-349`, `F051MainFragment.java:344-349` and `F053MainFragment.java:344-349` are structurally identical — `buffer[0]==6 → 4` (wind), `==5 → 3` (listen). This means some firmware reports these two levels as 5/6, and the module's `FiilRfcommController.applyMateNotify()` already mirrors that mapping.

### 3.6 EQ read/write (key details, easy to get wrong)

**Read** (`F046StyleFragment.java:153/219`):

```java
if (buff.length >= 12) {
    f29301i = buff;                      // save the whole block
    list = buff[1 .. length-1];          // skip [0]
    setEq(list);                         // → m(list.get(0)=buff[1], list)
}
```

**Write custom** (`s0()` @L890-903):

```java
if (f29301i.length < 12) return;         // do not write if never read
f29301i[1] = 15;                         // 15 = custom
for (i = 0; i < sliderCount; i++)
    f29301i[i + 2] = sliderValue;        // starting at [2]
setEQ(f29301i);                          // send the whole 12 bytes back
```

**Write preset** (`z0(int type)` @L986-999): only replace `f29301i[1]` with the preset number and send the whole block back.

**Conclusion (payload layout)**:

```
[0]  reserved (reuse the value from the read response; the module fills 0)
[1]  level: 3~14 = preset (Nostalgia…New Age), 15 = custom
[2..] N band gains, 0~255 offset encoding, 128 = 0 dB
```

**Preset number mapping**: `n()` @L765 builds `new c(icon, selIcon, name, i2 + 3)` → the i-th preset (0-based) = wire value `i + 3`.

**The number of bands differs by model** (`arrays.xml`):

| Model | Bands | Center frequencies | Source |
|---|---|---|---|
| Default (F046/F047/F048/F051/F053) | 10 | 31 / 63 / 125 / 250 / 500 / 1k / 2k / 4k / 8k / 16k | `customize_eq` @L87 |
| F042 / F043 | 10 | 200 / 280 / 400 / 550 / 770 / 1k / 2k / 4k / 8k / 12k | `f042_customize_eq` @L118 |
| F045 | 5 | 63 / 250 / 1k / 4k / 8k | `f045_customize_eq` @L158 |
| F049 | 8 | 125 / 250 / 500 / 1k / 2k / 4k / 8k / 16k | `f049_customize_eq` @L165 |

> Read-length check per model: F046 requires `>= 12`, while F045/F047/F048/F049/F051/F053 require only `>= 7` (lenient; the actual length comes from the response). The module takes the band count dynamically as `payload.size - 2` instead of hard-coding 10.

### 3.7 Low latency: three levels on GAIA / two states on Mate

**This is a section that this document once got wrong and has now corrected.** An earlier version asserted that "the protocol has only two states, and video is a semantic level invented by the module" — a conclusion extrapolated after reading only `F041SetFragment` (the F041 model really is a two-state switch). A closer look confirms that **the GAIA link genuinely has three levels**.

Wire values:

| Value | Meaning | Basis |
|---|---|---|
| 0 | Music mode (default) | `firstCCPro2SetFragment.I()` @:917 → `setLowLatency(0)` |
| 1 | Game mode | same @:935 → `setLowLatency(1)` |
| 2 | Video mode | same @:952 → `setLowLatency(2)` |

Links:

- Write (GAIA): `FF 01 00 01 02 34 07 22 [level]`, cmd = 1826 = `0x0722`
  —— `FiilCommandUtil.setLowLatency(int i2)` @:1022 passes the argument straight through as a **1-byte payload**:
  `getSendBuffer(VENDOR_FF, "FFCOMMAND_SetLowLatency", false, (byte) i2)`.
- Write (Mate): `classifyId 37`, payload `[0/1]` —— `com\fengeek\main\i\d\c.java:388`
  `setLowDelay(boolean)` accepts only a boolean and **has no video level**.
- Read (GAIA): `07 21` (GET_LOW_LATENCY = 1825), returns `payload[1]`, i.e. 0/1/2.
- Read (Mate): `classifyId 39, cmdId 8`, `payload[0]==1` = on.

UI corroboration: in `fragment_first_cc_pro2_set_new.xml` L35/39/43 the three `movie_textView1/2/3`
labels read "game mode / music mode / video mode", and tapping them goes through the same `setLowLatency(0/1/2)` group above.

Module behaviour: GAIA models are sent 0/1/2 as-is; Mate models have only two states, so choosing "video" downgrades to 1 and says so in the log.

**On-device re-check (2026-10-25, FIIL Key Max / firmware 3.0.1.1 / over BLE GATT)**: Mate's cid=37 cmdId=0 **accepts all three values 0/1/2** in practice; after the SET is acked,
GET_LL(39/8) reads back the same value that was written, so it is not limited to a boolean two-state. The module therefore opens Mate up to three levels as well:
FiilMate.Param.LOW_LATENCY = 37, payload [level]. **"Mate is two-state" was an inference drawn early from reading only the setLowDelay(boolean) signature, and has been overturned by the device.**

### 3.7.1 Voice prompt volume (Mate only, cid=1 cmdId=3)

| Item | Value |
|---|---|
| Write | 00 01 01 00 03 03 01 [vol] (com\fengeek\main\i\d\f.java:711-726 setTone, buffer {vol}) |
| Read | 00 27 01 00 02 82 00 (cid=39, cmdId=130 = -126 & 0xFF, c.java:251 getToneStatus) |
| Read-back sample | 01 27 02 00 03 82 01 02 |
| Wire value | **0=high volume / 1=medium / 2=low / 3=mute** (counter-intuitive: 0 is the loudest) |

**The pitfall of the read-back offset**: the response above is, byte by byte,
seq=01 | cid=27 | type=02 | reserved=00 | lenField=03 | cmd=82 | payloadLen=01 | payload=02
—— the length field 0x03 minus 2 gives payloadLen = 1, **the payload is only 1 byte**, and the volume sits in payload[0].
Early on, 01 (payloadLen) was misread as a "field presence flag" and the code therefore said "the volume is in payload[1]",
and only happened to read correctly thanks to an else branch that read payload[0] as a fallback; it has now been changed to the real offset and pinned down with a unit test.


### 3.8 Multipoint connection (Bluetooth Smart Connect)

- Write: `classifyId 1` **+ `cmdId 1`**, payload `[0/1]` → wire `00 01 01 00 03 01 [0/1]`
- Read: `classifyId 39, cmdId 128`, `payload[0]==1` = on

### 3.9 Cinema mode (spatial audio)

- Write: `classifyId 50`, payload `[0/1]` → wire `00 32 01 00 03 01 [0/1]`
- Read: `classifyId 39, cmdId 24`, `payload[0]==1` = on
- **Only F046 (Key Max) implements this command** (`setSpatialAudio` is defined only in `com\fengeek\main\i\d\c.java`, and its only call sites in the whole codebase are `F046StyleFragment.java:560/842/897/996`; see the coverage table in §3.4.1). The module does not expose this entry point for non-F046 models, rather than assuming "all Mate models support it".
- In the App, switching EQ while cinema mode is on **turns cinema off first** (`s0()` @L896-898, `z0()` @L995-997); the module does not replicate this linkage (see the unimplemented items in §6).

### 3.10 Wear detection (write cid 38 / read cid 39 cmdId 9)

| Item | Mate (F051 / F053) | GAIA |
|---|---|---|
| Write | `00 26 01 00 01 [1/0]` —— **cid=38, no cmdId**, value `bool?1:0` | cmd `SET_WEAR=263` (0x0107), value **1=on, 2=off** |
| Read | `00 27 01 00 00 09` —— cid=39, cmdId=9, no payload | cmd `GET_WEARING_SWITCH=1917` (0x077D), integer returned in `payload[1]` |
| Notify | cid=40, cmdId=9, `payload[0]==1` = worn (single value, no left/right split) | `GET_WEARING_STATE=1912`, left and right reported separately |
| Evidence | `g.java:663`/`h.java:729` (write), `g.java:267`/`h.java` (read); `F051SetFragment.java:662/745`, `F053SetFragment.java:717/783` | `FiilCommandUtil.java:1367-1375` (write, `z ? 1 : 2`), `:653-658` (read 1917); `GaiaSdkCommand.java:1199 analysisGetWearingSwitch`, `:1159 analysisGetWearStatus`, `:2305 analysisWear` (`isOpen()` = `bArr[1]==1`) |

> **The two protocols use different values, and this is the easiest thing to get wrong**: Mate's "off" is `0`, GAIA's "off" is `2`.
> The early module wrote GAIA as `0` too, which the earbuds would treat as an unknown value.
>
> **"Switch" and "state" are two different semantics**: cid39/cmdId9 asks "is wear detection enabled", cid40/cmdId9 reports "is it worn right now", and the module maps them to `wearDetect` and `wearLeft/wearRight` respectively.

---

### 3.11 Adaptive switches (query on cid=39 / write on cid=1, five independent switches)

This is a group of **independent boolean switches that sit alongside the MAF levels**, not a 5th MAF level.
In the FIIL+ UI they are scattered across the "intelligent noise tuning" and "sound quality" pages, and are easy to mistake for MAF sub-items when reverse engineering.

#### 3.11.1 Command wire values

Writes always use `cid=1` (`FiilMate.Cid.SET_PARAM`) with a fixed 1-byte payload `[1/0]`;
reads always use `cid=39` (`QUERY`) with no payload.
**The cmdIds for reads are written as `NetError.ERR_*` constants (negative numbers) in the original source and must be converted to unsigned bytes**:

| Function | Write (cid=1, cmdId) | Read (cid=39, cmdId) | Source constant | Model coverage |
|---|---|---|---|---|
| Cloud-feel tuning | 8 | 135 | `ERR_SOCKS_CONNECTION_HOST_UNREACHABLE` = -121 | F049 |
| Adaptive EQ | 14 | 137 | `ERR_HOST_RESOLVER_QUEUE_TOO_LARGE` = -119 | F049 / F051 / F053 |
| Ear-canal-adaptive noise cancelling | 15 | 138 | `ERR_CONNECTION_TIMED_OUT` = -118 | F049 / F053 |
| Environment-adaptive noise cancelling | 16 | 139 | `ERR_BAD_SSL_CLIENT_AUTH_CERT` = -117 | F049 / F051 / F053 |
| Adaptive wind-noise reduction | 17 | 140 | `ERR_CERT_ERROR_IN_SSL_RENEGOTIATION` = -116 | F049 |

Evidence: in `com\fengeek\main\i\d\f.java` (F049),
`setEQAdaptive:460` / `setEarAdaptive:477` / `setNoiseAdaptive:612` / `setTuning:728` / `setWindAdaptive:762`,
together with `getEQAdaptive:59` / `getEarAdaptive:91` / `getNoiseAdaptive:219` / `getTuning:283` / `getWindAdaptive:315`.
`g.java` (F051) has only the EQ + Noise pair, and `h.java` (F053) has only the EQ + Ear + Noise trio —
which matches the number of switches in each model's layout file `fragment_f{49,51,53}_style.xml` exactly.

#### 3.11.2 Frame examples

```
write "enable environment-adaptive noise cancelling"  → 00 01 01 00 03 10 01 01
                          │  │  │  │  │  │  └─ payload=[1]
                          │  │  │  │  │  └──── cmdId=0x10=16 (NOISE_ADAPTIVE)
                          │  │  │  │  └─────── len=3 (cmdId+payloadLen+payload)
                          │  │  │  └────────── reserved
                          │  │  └───────────── ver=1
                          │  └──────────────── cid=1 (SET_PARAM)
                          └─────────────────── SOF=0x00

read "environment-adaptive noise cancelling state" → 00 27 01 00 02 8B 00
                          │  │  │  │  │  │  └─ payloadLen=0
                          │  │  │  │  │  └──── cmdId=0x8B=139 (NOISE_ADAPTIVE)
                          │  │  │  │  └─────── len=2
                          │  │  │  └────────── reserved
                          │  │  └───────────── ver=1
                          │  └──────────────── cid=39 (QUERY)
                          └─────────────────── SOF=0x00
```

In the response `payload[0]==1` means on and `0` means off.

#### 3.11.3 Pitfalls

1. **138/139 in `Query` collide numerically with `LEFT_UUID`/`RIGHT_UUID`**:
   the former come from `c.java` (the F046 command class), the latter from `f/g/h.java` (F049/F051/F053).
   The same cmdId has different meanings in different model command classes, so **gating by earType is mandatory**;
   otherwise F046 is sent a 138 it does not understand, the earbuds silently drop it, and the module waits 2.5 s for nothing and then falsely reports failure.
2. These five switches **do not follow the MAF level**: after noise cancelling is turned off, "environment-adaptive noise cancelling" can still be set independently.
3. `setTuning` (cloud-feel tuning) and `setEQAdaptive` both affect the tone, but the former is hardware-level feedback tuning
   and the latter is software EQ compensation. They can be on at the same time.

#### 3.11.4 The echo is not the state (a very easy place to get wrong)

The response to `cid=1` **only means "the write succeeded" and carries no new state**; in practice the payload is empty or a single byte `0x00`.
In other words `[0]` is a success code, not "off". Likewise the SET echo of `cid=32/37/38/50` (EQ / low latency / wear / spatial audio)
is an empty packet.

Three wrong ways to write it, and their consequences:

| Code | Consequence |
|---|---|
| `state = payload[0] == 1` | After switching on successfully it is recorded as off (`[0]` is always 0) |
| `state = !state` (invert inference) | Correct for a single operation; but tap twice, or resend after a lost echo, and it drifts out of sync with the real state |
| Record the target on send, use it on echo | Correct — the module's `pendingTargets` does exactly this |

The real state can only come from the query response on `cid=39` (or the report on `cid=40`).
The module therefore separates the two paths: `applyAdaptiveFlag()` handles direct values from cid=39,
while `applyAdaptiveEcho()` / `applySetParam()` handle the SET echo and trust only `pendingTargets`.

### 3.12 GET and SET are **two completely different frame semantics** (the biggest trap in practice)

On 2026-10-25 a PC connecting directly to FIIL Key Max with Bleak was tested command by command, an entire evening of traps, and the conclusions are:

**GET (query): `classifyId` is fixed at 39, `commandId` holds the query code**

```
00 27 01 00 02 0C 00        query MAF
└┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘
seq  cid=39 t=1 rsv len=2 cmd=12 len=0
```

**SET (set): `classifyId` = command family number, `commandId` = 0**

```
02 2C 01 00 01 02           set MAF=2
└┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘
seq  cid=44 t=1 rsv len=1 payload
                     ↑ note: here there is **only one length field**, no cmdId
```

| | classifyId | commandId | Number of length fields |
|---|---|---|---|
| **GET** | always 39 | query code | 2 (`len+2` and `len`) |
| **SET** | command family (1/32/33/37/38/44/50…) | **always 0** | 1 (`len`) |

> **Trap 1 (which I hit)**: the first version wrote SET as `cid=39 + commandId=set code` too, and the earbuds answered every SET with an **empty frame** `xx 27 02 00 00` (`lenField=0`). An empty frame means "I don't know this command", not "executed successfully".
>
> **Trap 2 (which I hit)**: after fixing the cid it still failed, because `payloadLen` had been written as two mutually exclusive `if/else` branches. In fact `writeMate` writes `payloadLen` **unconditionally** once more after `if (commandId != 0)` — both length fields must be written when cmdId≠0, and only the latter is written when cmdId=0.

**SET acknowledgement**: `[seq][cid][02][00][01][00]` —— 5 bytes, and **`payload[0]==0` means success** (1 = failure).
**Read-back confirmation**: SET does not return the new value on its own, so another GET must be sent to confirm (some models additionally push a `cid=40` NOTIFY frame).

**SETs verified one by one on a real device (FIIL Key Max, firmware 3.0.1.1)**

| Function | Frame sent | Acknowledgement | Read-back confirmation |
|---|---|---|---|
| MAF noise-cancellation mode | `02 2C 01 00 01 02` | `02 2C 02 00 01 00` | `GET_MAF` → `0C 01 02` ✓ |
| Low latency | `07 25 01 00 01 01` | `08 25 02 00 01 00` | `GET_LL` → `08 01 01` ✓ |
| Multipoint Smart Connect | `07 01 01 00 03 01 01 01` | `07 01 02 00 03 01 01 00` | **BLE drops** after setting (normal, primary/secondary switch); read back after reconnecting ✓ |
| Cinema / spatial audio | `02 32 01 00 01 01` | `02 32 02 00 01 00` | `GET_SPATIAL` → `18 01 01` ✓ |
| EQ 12 bytes | `0C 20 01 00 0C 0A 0E …` | `03 20 02 00 01 01` | **read-back unchanged** (see below) |
| Play/pause | `0E 21 01 00 03 02 01 01` | `07 21 02 00 03 02 01 00` | — |
| Next / previous track | `0F 21 01 00 02 05 00` / `00 21 01 00 02 04 00` | both answered | — |

> **The EQ peculiarity**: `SET_EQ` (cid=32) **returns success (`payload[0]==1`… actually status code 1) but the read-back curve does not change**. The reason is that EQ on Key Max runs on a "preset + custom" dual track, so writing requires switching to the custom level first (`setEQAdaptive` cid=1 cmd=14), or the model simply does not accept arbitrary curve writes. **This is still unconfirmed**, and the module's EQ custom sliders currently only promise to work on GAIA models.

### 3.13 Earbud-initiated reports (cid=40 NOTIFY), measured samples

On the Mate channel the earbuds push spontaneously when the battery level changes or the earbuds are taken off:

```
0B 28 03 00 0E 83 0C D8 80 83 00 00 01 00 C3 0A B3 7F 60
└┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └┬┘ └─────────────────────────────┘
seq  cid=40 t=3 rsv len=14 cmd=0x83(131) len=12  payload(12B, suspected peer device MAC + state)
```

- The `type` field (offset 2) is **`0x01`=REQUEST / `0x02`=RESPONSE / `0x03`=NOTIFY**, not a version number.
  **The old implementation hard-coded `discard if !=0x01`, so every RESPONSE was thrown away as noise** — this is one of the direct reasons "the UI always showed not connected".
- Under `cid=40`, `cmd=0x83(131)` is not in any known table, so it is currently treated as "multipoint peer information" (the 12 bytes contain `D8:80:83:00:00:01` = another paired device).

---

## 4. Doubao link 0x55AA (a few models)

Source: the gate at `com\fengeek\main\i\b.java:187`; enabled only for **earType 47 / 48**.

```
55 AA | 00 | LEN_LO LEN_HI | payload... | A5 5A
```

- Interleaved with Mate frames **on the same socket**; the read loop demultiplexes by first byte (`0x55` = Doubao, `0x00` = Mate, `0xFF` = GAIA).
- The module does **not implement** the Doubao link at present; those models fall back to the Mate path (`0x00` frames can still be parsed).

---

## 5. Models and routing

### 5.1 earType → model name

| earType | Model | Link |
|---|---|---|
| 1~44 | see `FiilModels.NAMES` | GAIA |
| 45 | FIIL GS Lite | Mate |
| 46 | FIIL Key Max | Mate |
| 47 | FIIL Atom | Mate + Doubao |
| 48 | FIIL GS Links | Mate + Doubao |
| 49 | FIIL Key Pro2 | Mate |
| 51 | FIIL CC3 | Mate |
| 53 | FIIL CC Pro3 | Mate |
| 39 | CC Pro2 | Airoha BLE |
| 40 | Belt | Airoha BLE |
| 42 | Box | Airoha BLE |

> Name source: entries such as `fiil_f045`~`fiil_f053` in `apktool_out\res\values\strings.xml`.

### 5.2 Handshake sequence

**GAIA models** (`com/fiil/sdk/c/a.java:132-155`): version(769) → model(777) → battery(771) → ANC(641).

**Mate models** (no earType command): version(39/2) → battery(39/1) → MAF(39/12) → low latency(39/8) → cinema(39/24) → multipoint(39/128).

Module implementation: `FIILProtocol.handshake()`, called once inside `FiilRfcommController.connect()`.

### 5.3 Manual model override (fallback for model identification)

Neither `GET_EARTYPE` (GAIA 777) nor the Mate link has a generic "query model" command (Mate models simply do not provide an earType response; the module infers the model from the model word in the Bluetooth name). The module therefore offers a manual fallback:

| Item | Value |
|---|---|
| Storage | `ear_type_override` in `SharedPreferences("pods_config")` (0 = unspecified); read and write via `FiilConfig.readEarTypeOverride` / `writeEarTypeOverride` |
| Address binding | Key `ear_type_for_<MAC>` in the same file; a model identified through any of the three paths (manual override, Bluetooth-name inference, `GET_EARTYPE` response) is stored there, and reused directly on the next connection to the same address |
| When it takes effect | `FiilRfcommController.connect()` reads it first; if non-zero it is used directly and `earTypeOverridden = true` |
| Priority | manual override > address binding > Bluetooth **original name** inference > `-1` unknown (waiting for a response); the display name uses an alias, but model identification uses **the original name only** |
| Automatic response | while an override is in effect the `GET_EARTYPE` response **does not overwrite** the specified value (otherwise it would be changed back as soon as the device connects) |
| Protocol-family switch | once a Mate model (45/46/47/48/49/51/53) is specified, `isMateDialect` decides by model and the handshake follows the Mate sequence |
| Restore automatic | pass `-1` to clear the preference, then look up the address binding first and only then fall back to original-name inference |

Broadcast channel: `ACTION_COMMAND` + `EXTRA_CMD="setEarType"` + `EXTRA_VALUE=earType` (-1 restores automatic).

### 5.4 Device identification: three fallbacks

| Method | Basis | Scenario covered |
|---|---|---|
| Name keyword | Name contains `FIIL` / `FIIL CC` / `FIIL GS` / `FIIL KEY` / `FIIL T` / `FIIL CARAT` / `FIIL DIVA` | Factory name has not been changed |
| MAC prefix | `B0:F1:A3`, `FF:F1:A3` | Bluetooth name changed by the user beyond recognition |
| Address binding | `ear_type_for_<MAC>` preference | Model was identified before, but the name has since changed |

Reverse-engineering origin of the MAC prefix: on every OTA page FIIL+ treats `device.getAddress().startsWith("B0:F1:A3")` as the hard test for "is this one of our devices" — `jadx_out/sources/com\fengeek\main\base\BaseFragment.java:28`, plus the OTA activities of F039 / F040 / F041 / F043 / F045 / F046 / F048 / F049 / F051 / F053 respectively (15 sites in total); `F042HomeFragment.java:1014` has another `FF:F1:A3` variant. See `FiilConfig.isFiilAddress()` / `isFiilDevice(name, address)` for the module side, with call sites covering four places: the Bluetooth process hook, the settings-page hook, the fused device center hook, and paired-device enumeration.

---

## 6. Unconfirmed items and known limitations

| # | Item | Status |
|---|---|---|
| 1 | Full UUID of the F042 private BLE characteristic | Truncated in the decompiled artifacts; that link is not implemented |
| 2 | The "video" low-latency level | **Corrected**: the GAIA link really does have three levels 0/1/2 (see §3.7); only Mate models have no video level, and the module degrades it to game |
| 3 | payload structure of the Doubao 0x55AA frame | Not reverse engineered in detail; the module does not send that frame |
| 4 | Meaning of Mate's `[0]` reserved field | Only known that the App echoes it back unchanged; the meaning is unknown, and the module fills 0 |
| 5 | Mutual exclusion between cinema mode and EQ | The App has it (switching EQ turns cinema off first); the module does not replicate it |
| 6 | Wear detection (Mate models) | **Implemented**: `setWear`/`getWear` exist only in `i.d.g`(F051) and `i.d.h`(F053); the module gates on `supportsWearDetect()` = setOf(51,53) and the UI switch is wired up (see §3.10) |
| 7 | Availability of the adaptive mode (MAF=4) | Decided by `FiilModels.supportsExtendedAnc()` against a per-model allow-list; not verified on a real device |
| 8 | Simultaneous connection of multiple models | The controller is a singleton and maintains only one SPP connection |
| 9 | Per-model command differences not verified on hardware | The table above is derived from decompiled method declarations; whether each firmware actually implements everything needs per-model verification on a real device |
| 10 | Fused device center (`com.milink.service`) | **Implemented** (see §7), but depends on HyperOS internal class signatures; hook points may break after a system upgrade, and on failure it only logs and never crashes |
| 11 | Custom EQ writes on Mate models | **Do not take effect in practice**: `SET_EQ` (cid=32) returns a success code but the read-back curve is unchanged (Key Max firmware 3.0.1.1, see §3.12). It may be "must switch to the custom level first" or "this model only accepts presets"; unconfirmed |
| 12 | `cid=40` / `cmd=0x83(131)` report | A 12-byte payload can be received in practice (suspected multipoint peer MAC); the semantics are not fully reverse engineered, so the module only logs it and does not change state |
| 13 | Concurrency of SPP and BLE GATT | What happens when both channels are open at once has not been tested; the module currently makes them mutually exclusive and uses only one at a time |
| 14 | SPP behaviour on FIIL Key Max | In practice standard SPP connects and goes silent, and a custom-UUID connection is kicked repeatedly (`scn:6 opened=21 leaked=21`). **This is the direct reason for switching to BLE GATT**; whether other Mate models behave the same has not been tested |

---

## 6.1 System interfaces on the fused device center side

The fused device center is not part of the FIIL protocol; it is a layer the module **fakes for HyperOS**. It is listed here because it likewise drifts with system versions.

| System class / method | Purpose | What we return |
|---|---|---|
| `MxBluetoothService/Manager.checkIsMiTWS(BluetoothDevice)` | Whether it is a Xiaomi TWS | `1` |
| `.getDeviceId(BluetoothDevice)` | Device ID | `01010607` |
| `.getBatteryLevel(...)` | Battery | The lower of the left and right earbuds |
| `.getAncState(...)` | Noise-cancellation state | MAF five states → MIUI three states |
| `.getWearStatus(...)` | Wear | `"0,0"` |
| `.isLeAudio(...)` | LE Audio | `false` |
| `ProfileContext.getBatteryLevel(BluetoothDevice)` | Three-way breakdown | `[case, left, right, charge1, charge2, charge3]`, `-1` when not connected |
| `ProfileContext.getAudioSpatialEffectState(...)` | Spatial audio | Cinema on → `2`, off → `0` |
| `AncBatteryController.getFindRingState(...)` | Game-mode bit | On → `100`, off → `0` |
| `AncBatteryController.setAncStateBlock(device, int)` | Dragging the ANC slider | Intercepted, converted to `ACTION_ANC_SELECT` |
| `AncBatteryController.setMiAudioEffect(device, int)` | Spatial audio switch | Intercepted, converted to `ACTION_COMMAND("cinema")` |
| `AncBatteryController.setFindRing(device, int)` | Find-earbuds card | Taken over as low-latency game mode |
| `AncBatteryController.headsetPropertyChangeListener.invoke(device, int)` | Refresh the UI in reverse | 4 battery / 8 noise cancelling / 9 spatial audio / 10 game mode |

> The mapping between the MIUI three states and FIIL MAF is not one-to-one: FIIL's "listen" and "wind" both show up as "transparency" on the system side. The control center can only select three levels; the five levels are still selected in the module's own UI.

---

## 7. Module-side implementation map

| Protocol capability | Module entry point | File |
|---|---|---|
| Semantic command wrappers | `FIILProtocol.requestBattery()` / `setMAFMode()` / `setCinemaMode()` / `setLowLatencyMode()` / `setMultipoint()` / `setWearDetect()` / `readEQ()` / `writeEQ()` | `pods\FIILProtocol.kt` |
| GAIA frame encode/decode | `FiilGaia.buildFrame()` / `parse()` / `drain()` | `pods\FiilGaia.kt` |
| Mate frame encode/decode | `FiilMate.buildFrame()` / `parse()` / `drain()` / `setFrame()` | `pods\FiilMate.kt` |
| SPP connection and I/O | `FiilRfcommController.openSocket()` / `connect()` / `readLoop()` / `sendMate()` | `pods\FiilRfcommController.kt` |
| **BLE GATT connection and I/O** | `FiilGatt.connect()` / `send()` / `handleRx()` / `close()` | `pods\FiilGatt.kt` |
| **SPP→GATT automatic downgrade** | Inside `FiilRfcommController.connectBlocking()`: all SPP attempts fail or 3.5 s with zero responses → `useGatt = true` | `pods\FiilRfcommController.kt` |
| State cache | `FiilStatus` + `StatusRepo` | `pods\FiilStatus.kt`, `ui\StatusRepo.kt` |
| Settings-page injection | `SettingsHook` (including the "⋮" menu → module console entry `hookModuleEntry()`) | `hook\SettingsHook.kt` |
| Fused device center | `MiLinkServiceHook` / `MiLinkSpatialAudioHook` | `hook\milink\MiLinkServiceHook.kt`, `hook\milink\MiLinkSpatialAudioHook.kt` |
| Super Island | `MiBluetoothToastHook` | `hook\MiBluetoothToastHook.kt` |

---

## 8. Compliance notice

The purpose of this document is to **study Bluetooth protocol stack design** and to develop a local control module for one's own earbuds. All commands come from static analysis of a publicly distributable APK. Do not use it to:

- redistribute FIIL's private protocol implementation or resource files;
- bypass a device's firmware checks or perform unauthorized OTA;
- claim compatibility with FIIL official products in a commercial product.

This repository contains documentation only, no code. The upstream reference projects (HyperPods / HuaweiPods / OppoPods) are licensed under GPL-3.0, and their code has **not** been copied into this repository; see [NOTICE](../NOTICE) for the attribution and the exact boundary.
