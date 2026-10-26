[简体中文](PROTOCOL_PC.md) | **[English](PROTOCOL_PC.en.md)**

# PROTOCOL_PC.md — FIIL Headphone BLE GATT Protocol Specification (PC Probe Measured Edition)

> Every UUID and byte in this document comes from **real device captures taken for this document**, not from guesswork.
> Each conclusion is annotated with an evidence file (`evidence/*.txt`) or a line number in the official decompiled FIIL+ source.
> Anything not verified on a real device is marked `unconfirmed` and is not written into control code.

- **Device under test**: FIIL Key Max, BLE address `B0:F1:A3:00:00:01`
- **Firmware version**: 3.0.1.1 (`GET_VERSION` → payload `03 00 01 01`)
- **Test time**: 2026-10-25 00:16 — 01:50
- **Test platform**: Windows 11 + Python 3.12.6 + bleak (WinRT backend)
- **Probe programs**: the author's own `app.py` (GUI) / `ble_worker.py` (BLE thread) / `fiil_protocol.py` (protocol layer)
- **Unit self-test**: `test_protocol.py` → **67 passed / 0 failed**

> **About the .py files mentioned in this document**: `app.py` / `ble_worker.py` / `fiil_protocol.py` / `test_*.py` / `exp_*.py` are all probe scripts the author wrote ad hoc to verify this protocol and **are not distributed with this repository**. They are mentioned only so that the conclusions stay traceable and to show how a particular pitfall was worked around at the implementation layer. If you write your own controller, these names need not be copied.

---

## 0. Conclusions at a Glance

| Item | Conclusion |
|---|---|
| Transport layer | **BLE GATT**, service `0000fdb3`, write `ff17`, notify `ff18` (`ff16` is writable but the headset does not respond) |
| Required after connecting | **Send `setUnlock` first**, otherwise the headset is silent for all commands |
| Frame format | `seq cid type 00 [lenField] [cmdId] [payloadLen] payload` (6–N bytes, see §2) |
| GET semantics | `cid=39` is fixed, `cmdId` = query code (all 18 verified on a real device) |
| SET semantics | `cid` = command family id, **`cmdId=0`**, and the length field is payloadLen directly |
| seq | **Cannot be used to match requests with responses** (the headset uses its own rolling counter, see §2.5) |
| type | Must accept `0x01/0x02/0x03` together; accepting only `0x01` drops every reply |
| SET take-effect latency | MAF about **1.75~2.0 s**; cinema **0.5 s**; low latency **0.45 s** (reported via NOTIFY; GET polling cannot keep up) |
| Status read-back | Trust `cid=40` (NOTIFY); do not rely on the SET ACK |
| MAF levels | 5 levels defined in the source; **in this measurement all of 0/1/2/3/4 are usable** (see §6.4) |
| Low-latency levels | Four levels on GAIA (off/game/music/video); **Mate models (this Key Max) have only 0/1**, and SET 2/3 is explicitly rejected by the headset ACK (see §8.2) |
| Bluetooth multipoint | GET `cmdId=128` / SET `cid=1 cmdId=1`; **toggling drops the BLE link immediately**; reconnect and redo `setUnlock` (see §9) |
| Voice prompt volume | GET `cmdId=130` / SET `cid=1 cmdId=3`; **the wire values are reversed**: 0 high / 1 medium / 2 low / 3 mute (see §8bis) |
| SET success criterion | 6-byte short-frame ACK `xx cc 02 00 01 <00 success / 01 reject>` — it must be checked, otherwise the UI will pretend to succeed |

---

## 1. Service / Characteristic UUID Table (Confirmed)

Measured enumeration result, evidence: `evidence/gatt_20261025_001610.txt`

```
Service 0000fdb3-0000-1000-8000-00805f9b34fb  [Vendor specific]
  Char 0000ff18-0000-1000-8000-00805f9b34fb handle=2 props=[notify,read]
    Desc 00002902-0000-1000-8000-00805f9b34fb handle=4      ← CCCD
  Char 0000ff16-0000-1000-8000-00805f9b34fb handle=5 props=[write,read]
  Char 0000ff17-0000-1000-8000-00805f9b34fb handle=7 props=[write-without-response,write]
```

| Role | UUID | handle | Properties | Description |
|---|---|---|---|---|
| Service | `0000fdb3-0000-1000-8000-00805f9b34fb` | — | — | The only custom service |
| **Write characteristic** | `0000ff17-0000-1000-8000-00805f9b34fb` | 7 | write / write-no-rsp | **All commands are sent here** |
| **Notify characteristic** | `0000ff18-0000-1000-8000-00805f9b34fb` | 2 | notify / read | **All replies come from here** |
| CCCD | `00002902-0000-1000-8000-00805f9b34fb` | 4 | — | bleak's `start_notify` handles it automatically |
| Decoy | `0000ff16-0000-1000-8000-00805f9b34fb` | 5 | write / read | **Writable, but the headset ignores it completely** (`evidence/log_unlock16_20261025_001943.txt` shows no reply at all) |

### 1.1 Three Measured Pitfalls (Written into the Code Comments)

1. **Online references generally list ff16 as the write characteristic — that is wrong for this model.** Writing `setUnlock` to ff16 produces no response whatsoever; the same bytes written to ff17 respond immediately.
2. **Write with `response=False` (Write Without Response).** Using Write With Response occasionally triggers `GATT_BUSY` / timeouts under high-frequency sending.
3. **notify still arrives even without writing the CCCD** (measured on the Android side as well). For uniformity, `start_notify` is still recommended.

### 1.2 MTU

Measured `client.mtu_size = 512`. Fixed-length frames such as the 12-byte EQ frame are sent in one write with no fragmentation. But **the receiving side must reassemble by length** — the headset may split one logical frame across several notify events (see §2.4).

### 1.3 Device Discovery: FIIL Usually Advertises No Name over BLE

This is why **the scan list shows only a MAC and you cannot tell which headset it is**; it is not a program bug.

Measured evidence (this FIIL Key Max):

| Item | Value |
|---|---|
| Classic Bluetooth BD_ADDR (used for pairing / in the system) | `B0:F1:A3:00:00:02` |
| BLE advertising address (as scanned by Bleak) | `B0:F1:A3:00:00:01` |
| `local_name` in the BLE advertising packet | **Empty** (`dev.name is None`) |

That is: FIIL is a dual-mode device whose main channel is classic Bluetooth; on the BLE side it exposes only a **nameless** advertisement,
and **the BLE address differs from the classic address only in the last byte** (`…:00:02` vs `…:00:01`).

`ble_worker._scan()` therefore resolves names with a three-level fallback:

1. `dev.name` / `adv.local_name` — the name carried in the advertisement (most accurate, but usually empty for FIIL)
2. Windows pairing / history cache — `FriendlyName` plus the `DEV_xxxxxxxxxxxx` in `InstanceId` from `Get-PnpDevice -Class Bluetooth`, turned into a `MAC → name` table (result cached for 30 s)
3. **Prefix fallback** — look for the same device in the cache table by the first 5 segments (`B0:F1:A3:00:00`), and **use it only when the match is unique** (so different models under the same OUI are not confused); if there is still none, show "FIIL device (model to be read after connecting)"

OUI prefixes of FIIL models (reversed from `FiilConfig.FIIL_OUI_PREFIXES`): `B0:F1:A3`, `FF:F1:A3`.
Devices that match the OUI or contain `FIIL` in their name are marked `★` in the list and sorted to the top, **so you will not pick the wrong one even without a name**.

After connecting, `GET_DEVICE_NAME` (cmdId=3) returns the real name (e.g. `FIIL Key Max`),
and the GUI rewrites that scan-list entry in place, so the real name shows from the second scan onward.

#### 1.3.1 The More Common Case: the Headset Is Not Advertising at All

More common than "no name in the advertisement" is **the address not appearing in the scan list at all**. Measured reproduction:

| Time point | Observation |
|---|---|
| Headset not connected to the PC | The scan finds `B0:F1:A3:00:00:01`, and the name comes from the pairing-cache fallback |
| Headset already occupied by the PC over classic Bluetooth (A2DP is playing) | 5 consecutive scans (8~25 s), 107~136 devices in total, **0 hits on the FIIL OUI**; 0 hits by the `ff17` / `ff18` / `fdb3` service-UUID fingerprint as well |

Reason: FIIL is a dual-mode headset; once it is active on the classic Bluetooth link, **it no longer sends connectable advertisements on the BLE side**.
It can still be **connected directly by address** over BLE, however (WinRT supports `connectGatt` to a known address).

The program therefore provides two fallbacks (`ble_worker.py`):

1. **Known-device table `known_devices.json`** — on every successful connection the BLE address is written to disk first (the model is not known yet),
   and the real name is filled in after the `GET_DEVICE_NAME` reply. **Writing an empty name does not overwrite a model already learned** (pinned by a unit test).
   After `_scan()` finishes, addresses in the table that "were not seen this time" are appended as a `★` row, and the user can still click to connect directly.
2. **GUI "direct address" input** — prefilled at startup with the last connected address; one click connects, with no dependency on scanning at all.
   The input is normalized, so forms such as `b0f1a3000001` / `b0-f1-a3-00-00-01` are both accepted.

> Note: you cannot connect to BLE directly with the classic Bluetooth address from the pairing cache — they are not the same address
> (here `…:00:02` is classic and `…:00:01` is BLE). So only **BLE addresses that were actually connected** enter the known-device table.


---

## 2. Frame Format (Confirmed)

### 2.1 Overall Layout

```
Byte  0     1     2      3       4         5        6          7..
     seq   cid   type   00   lenField   cmdId   payloadLen   payload
```

- **seq**: 1 byte, a rolling counter in the low 4 bits (0..15). Incrementing it on send is enough.
- **cid**: 1 byte, the command family id. Fixed at `39 (0x27)` for GET.
- **type**: `0x01`=REQUEST (host→headset) / `0x02`=RESPONSE (headset reply) / `0x03`=NOTIFY (headset-initiated push)
- **byte[3]**: reserved, must be `0x00`; a non-zero value is treated as an invalid frame.
- **lenField**: the **key field**; it determines the whole frame length. See §2.2 for its values.
- **Total length = 5 + lenField** (when `lenField == 1` the total is 6, which is the special case).

Source: `com/fengeek/main/i/b.java:237-262 writeMate()`.

### 2.2 The Three Cases of lenField (the Easiest Place to Get Wrong)

| lenField | Total length | Structure | Typical use |
|---|---|---|---|
| `0` | 5 | `seq cid type 00 00` | Empty frame returned when **the headset does not recognize the command** |
| `1` | 6 | `seq cid type 00 01 <ok>` | **Reply to a SET command** (no cmdId; the 6th byte is payload[0]) |
| `>=2` | `5+lenField` | `seq cid type 00 len cmd plen payload…` | All GET replies, all requests |

**`lenField == 1` is the most insidious pitfall**: it looks like an "empty frame of length 1", but the 6th byte is the **success flag** (`0x00`=success).

> Measured evidence: SET_MAF returns `02 2C 02 00 01 00` — 6 bytes, `lenField=1`, `payload[0]=0x00` for success.
> An early implementation sliced frames as `lenField<2 → 5-byte empty frame`, swallowing the success byte entirely and thus misjudging every SET as a failure. **Fixed**, pinned by 3 test cases in `test_protocol.py`.

### 2.3 Frame Examples (All from a Real Device)

```
setUnlock        00 27 01 00 04 FF 02 02 34        # cid=39 cmd=0xFF payload=02 34
GET_BATTERY      02 27 01 00 02 01 00              # cid=39 cmd=1  no payload
SET_MAF=1        0A 2C 01 00 01 01                 # cid=44 cmdId=0 payload=01
SET_SPATIAL=1    0B 32 01 00 01 01                 # cid=50 cmdId=0 payload=01
SET_LATENCY=1    0C 25 01 00 01 01                 # cid=37 cmdId=0 payload=01
SET_CHANGLAN=1   0D 01 01 00 03 01 01 01           # cid=1  cmdId=1 payload=01
```

Note that `SET_CHANGLAN` is the **only SET that carries a cmdId** (cmdId=1).

### 2.4 The Receiving Side Must Reassemble the Stream

Three real constraints (each with measured test cases):

1. A frame may be split across several notify events (**a partial frame must be kept**, never dropped)
2. One notify may contain several frames (**coalescing**)
3. Garbage bytes must be **resynchronized byte by byte** until a legal frame header is found (`type ∈{1,2,3}` and `byte[3]==0`)

`drain()` implements three branches: `lenField=0→need 5`, `=1→need 6`, `>=2→need 5+lenField`; when `avail < need` it **breaks and keeps** the data, and only an illegal header causes `pos += 1`.

### 2.5 ⚠️ seq Is Not a Request Correlation Id (Important)

**Measured**: TX `seq=04` → RX `seq=05`. The headset increments **its own rolling counter** independently, and RESPONSE and NOTIFY **share the same counter**.

```
[TX] GET_MAF @2.0s  04 27 01 00 02 0C 00
[RX]                05 27 02 00 03 0C 01 00      ← not 04
[TX] GET_MAF @3.0s  05 27 01 00 02 0C 00
[RX]                06 27 02 00 03 0C 01 00
```

**Conclusion: seq cannot be used to match requests with responses.** Dispatch can only be done by `cid + cmdId` semantics.

This matches the official implementation as well: for replies whose "cid/cmd does not match the pending queue", the stateHandler in `com.fengeek.main.i.d.*` **still goes down the same state-update path** — that is, cid=40 unsolicited reports and GET replies are equally trustworthy sources of state.

### 2.6 The Measured Meaning of the Three type Values

| type | Meaning | Example |
|---|---|---|
| `0x01` | REQUEST, sent by the host | `0A 27 01 00 02 0C 00` |
| `0x02` | RESPONSE, the headset's reply to a command | `0A 27 02 00 03 0C 01 01` |
| `0x03` | NOTIFY, pushed **unsolicited** by the headset (cid is always 40) | `0B 28 03 00 03 0C 01 00` |

An old implementation hard-coded `byte[2]` to `0x01` before parsing → **every reply was discarded**. That was the direct cause of "connected, but state is never received". **Fixed**.

---

## 3. Complete GET Command Table (18 Commands + Handshake, All Verified on a Real Device)

Evidence: `evidence/unverified_cmdid_20261025_014438.txt`, `evidence/table_20261025_002133.txt`

| cmdId | Name | Source method (`com/fengeek/main/i/d/c.java`) | Measured reply payload | Parsing |
|---|---|---|---|---|
| 1 | Battery | `getElectricity` :75 | `00 4D 00` | F046 single battery, `payload[1]=0x4D=77%` (see §5.2) |
| 2 | Version | `getVersion` :267 | `03 00 01 01` | Firmware 3.0.1.1 |
| 3 | Device name | (measured) | `46 49 49 4C 20 4B 65 79 20 4D 61 78` | **ASCII "FIIL Key Max"** (12 bytes), not a MAC |
| 4 | EQ | `getEQStatus` :59 | `0A 0E 08 06 04 02 00 01 FA FC FE 02` | 12-byte signed curve |
| 5 | Key configuration | `getKey` :91 | `01 01 07 02 01 07 03 01 00 …` (30 bytes) | 10 groups of "key number + type + value" |
| 7 | Playback state | `getMusicStatus` :187 | `00` | 0=paused 1=playing |
| 8 | Low latency | `getLowDelayStatus` :155 | `00` | 0=off 1=on |
| 9 | Wear detection | (no corresponding getter) | **empty frame `xx 27 02 00 00`** | **not supported on this model** |
| 12 | MAF noise cancellation | `getMAFStatus` :171 | `01` | 0..4, five states |
| 24 | Cinema / spatial audio | `getSpatialAudioStatus` :235 | `00` | 0=off 1=on |
| 128 | Multipoint | `getChangLanStatus` :27 | `01` | 0=off 1=on |
| 130 | Tone | `getToneStatus` :251 (NetError -126) | `02` | Tone level |
| 132 | PID | `getPID` :203 | `00 2E 01` | Product identifier |
| 138 | Left earbud UUID | `getLeftHeadsetUUID` :139 (NetError -118) | `10 42 50 32 36 32 30 35 12 00 3B 51 45 06 20 BB 78` | 16 bytes |
| 139 | Right earbud UUID | `getRightHeadsetUUID` :219 (NetError -117) | same as above | 16 bytes |
| 142 | LDAC unlock | `getLdacUnlockStatus` :123 (NetError -114) | `01` | — |
| 143 | LDAC switch | `getLdacSwitchStatus` :107 (NetError -113) | `00` | — |
| 144 | Codec | `getCodecStatus` :43 (NetError -112) | `02` | 0/1/2 codec level |
| 255 | Unlock handshake | `setUnlock` (cid=39 cmdId=-1) | `FF` | see §4 |

### 3.1 ⚠️ A Disproved Misconception: cmdId=5 is **not** earType

`cmdId=5` was once labeled "GET_EARTYPE / model earType", because the first byte of the reply, `0x1E = 30`, looked like a model number.

**Disproved**: the official `getKey` (`c.java:91`) uses `cmdId=5`, whose meaning is the **touch-key configuration table**. The reply structure is:

```
payloadLen = 0x1E = 30 bytes (byte 6 of the frame header, not payload content)
payload = 01 01 07 | 02 01 07 | 03 01 00 | 04 01 00 | 05 01 00
        | 06 01 00 | 07 01 00 | 08 01 00 | 09 01 00 | 0A 01 00
```

That is 10 groups of "key number + type + value" triples, 30 bytes in total = payloadLen. **There is no earType field at all.**

**There is no single-byte earType query command on this model.** The model can only be determined from the device name (cmdId=3) or the BLE advertising name.

> Lesson for the phone module: if `FiilMate` uses cid=39 cmd=5 as earType, it gets 30 and misjudges the model as CC2. **Device-name matching must be used instead.**

---

## 4. Connection Initialization Sequence (Confirmed)

### 4.1 Required Handshake

```
1. connect GATT
2. start_notify(ff18)
3. TX setUnlock ──► 00 27 01 00 04 FF 02 02 34
   ◄──            00 27 02 00 03 FF 01 FF
4. only then can any GET/SET be sent
```

**Without sending setUnlock, the headset is completely silent for all commands.** This is the most critical initialization step.

Evidence: `evidence/log_unlock16_20261025_001943.txt` shows no response vs `evidence/log_unlock17_20261025_001932.txt` replies immediately.

### 4.2 Where setUnlock Comes From

`com/fengeek/main/i/d/h.java:712-727`:

```java
public void setUnlock(infoListener) {
    c.a aVar = new c.a();
    aVar.setCommandClassifyId(39);          // cid = 39
    aVar.setCommandId(-1);                  // cmdId = -1 → 0xFF
    aVar.setBuffer(new byte[]{2, 52});      // payload = 02 34  (hex 0x34 = decimal 52)
    a.getInstance().addCommand(aVar);
}
```

Trigger point: `case 5:` in `F053MainFragment.java:275-279` ("message: Bluetooth connected") → sent **immediately** after a successful connection.

### 4.3 Recommended Full Initialization Sequence

```
setUnlock
→ GET_VERSION(2)      confirm communication is OK
→ GET_BATTERY(1)
→ GET_MAF(12)
→ GET_SPATIAL(24)
→ GET_LOW_LATENCY(8)
→ GET_CHANGLAN(128)
→ GET_TONE(130)
```

**A gap of ≥250 ms between commands is recommended**.

> ⚠️ The strength of the evidence for this point must be stated honestly: **it is not fully proven to be a timing problem**.
>
> Observed: after one successful connection, all three of `GET_LOW_LATENCY` / `GET_CHANGLAN` / `GET_TONE` returned nothing,
> while `GET_BATTERY` / `GET_MAF` worked in that same connection; the very next connection attempt failed outright,
> and after that the headset disappeared from BLE scans entirely (`BleakDeviceNotFoundError`). In other words,
> **"0/3" is more likely a prelude to the headset going offline than a result of sending too densely**.
>
> Defenses already in place (not contingent on the above judgment being correct):
> 1. The interval went from 120 ms to 250 ms — more conservative, costing only 1 extra second after connecting;
> 2. After the handshake, **re-read once** any missing item (`_got_ll/_got_mp/_got_tone` in `ble_worker.py`).
>
> Status: **the re-read logic has been verified offline by `test_worker_ack.py` (16/16); the 250 ms interval has not been re-checked on a real device** —
> the headset was already offline when the re-check was attempted. Both items need confirmation the next time a real device is connected.

### 4.4 Disconnect and Reconnect

After a disconnect, **setUnlock must be sent again** (the headset re-locks). `start_notify` must also be redone after reconnecting.

**Multipoint toggling is a special case**: `SET_CHANGLAN` makes the headset drop BLE immediately (primary/secondary role swap),
and `setUnlock` must likewise be redone after reconnecting — confirmed by measurement in `test_new3_live.py`. The program must reconnect automatically,
otherwise users will think "once multipoint is on, the headset can no longer be connected".

---

## 5. Battery Parsing Rules (Confirmed)

### 5.1 Frame

```
TX  xx 27 01 00 02 01 00
RX  xx 27 02 00 05 01 03 00 4D 00
                    │  │  │  │  │  └ payload[2] = 0x00
                    │  │  │  │  └── payload[1] = 0x4D = 77%   ← valid value on this model
                    │  │  │  └──── payload[0] = 0x00
                    │  │  └─────── payloadLen = 3
                    │  └────────── cmdId = 1
                    └───────────── lenField = 5
```

### 5.2 Two Semantics by Model (**Not "Only payload[1] Is Valid"**)

The same frame `00 4D 00` means different things on different models, so it must be branched by FIIL model (earType):

| Model family (earType) | Official implementation | Semantics |
|---|---|---|
| **F046 (earType 46) / F042 (42)** | `F046HomeFragment:623` → `m(buff[1])`<br>`F042HomeFragment:440` → `D(buff[1])` | **Single battery**: reads only `payload[1]`, one battery level for the whole device |
| F045 / F047 / F049 / F051 / F053 | `setElectricity` → `o(buff[0]); r(buff[1]); m(buff[2])` | **Three values**: `payload[0]=left earbud`, `payload[1]=right earbud`, `payload[2]=charging case` |
| F048 | `buff[0]==0 \|\| buff[1]==0` determines whether the left/right earbuds are online, then `getEarSurplus()` | Two values, left and right (case battery is queried separately) |

**The Key Max (earType 46) is an F046 → single battery**: payload `00 4D 00` should be displayed as "headset 77%",
**not as "right earbud 77%"** — that is the semantics for three-battery models and would mislead users into thinking only the right earbud has charge.
Values change across repeated measurements (another round read `00 4C 00` = 76%), which matches real battery fluctuation
(`exp_battery.py` → `evidence/battery_20261025.txt`: 20 samples all a stable `00 4D 00`).

### 5.2.1 Short-Frame Layouts (Fallback When earType Is Unknown)

| payload length | Layout |
|---|---|
| 3 | Branch by the table above; when earType is unknown, interpret as `[left][right][case]` |
| 2 | `[left][right]` |
| 1 | `[right]` |

### 5.3 Handling Invalid Values

Both `0` and `255 (0xFF)` are treated as "no data" and displayed as `--`. **Do not display 0% or 255%**. Short and empty frames must be defended against; a single abnormal frame must not crash anything.

### 5.4 Official Basis

- `jadx_out/sources/com\fengeek\main\f046\fragment\F046HomeFragment.java:623`
  `public void setElectricity(byte[] buff) { m(buff[1]); }` — single battery
- `...\f045\fragment\F045HomeFragment.java:546`
  `o(buff[0]); r(buff[1]); m(buff[2]);` — left/right/case
- `...\f047\...\F047HomeFragment.java:390`, `...\f049\...\F049HomeFragment.java:457`,
  `...\f051\...\F051HomeFragment.java:723`, `...\f053\...\F053HomeFragment.java:722` — all three values
- `...\f042\fragment\F042HomeFragment.java:440` — reads `buff[1]` after validating with `buff[0] != 0`
- Routing: `...\f002\MainActivity.java:1032-1049` `case 42→F042 / 46→F046 / 45→F045 / 47→F047 / 48→F048 / 49→F049 / 51→F051 / 53→F053`

---

## 6. MAF Noise-Cancellation Modes (**All Five States Verified on a Real Device**)

### 6.1 Enumeration

| Value | Meaning | Label (verbatim from FIIL+ `fragment_f46_style.xml`) | This measurement |
|---|---|---|---|
| 0 | Off | Off | ✓ write and read-back agree |
| 1 | Noise cancellation | Noise cancellation | ✓ write and read-back agree |
| 2 | Open | Open | ✓ write and read-back agree |
| 3 | Awareness | Awareness | ✓ write and read-back agree |
| 4 | Wind | Wind | ✓ write and read-back agree |

**All five levels are usable** (`exp_maf_five.py` → `evidence/maf_five_20261025.txt`: SET 0→4 level by level, the ACK for every level was `OK`,
and the read-back at @3.0 s always equaled the written value). The earlier judgment that "only 0/1/2 are covered" was a **sampling bias** — at the time only the values reported by the headset were read passively,
3/4 were never sent, so it could not be inferred from that that the firmware does not support them.

> ⚠️ Note the alignment with the target description: the target says "Noise cancellation / **Adaptive** / Wind / Monitor / Off", whereas the FIIL+ original is
> "Off / Noise cancellation / Open / **Awareness** / Wind". That is, "Adaptive" in the target has **no corresponding wire value** in this protocol;
> it is actually 4 independent "adaptive switches" (cid=1, cmdId=14/15/16/17), not a 6th MAF level;
> "Monitor" in the target = "Awareness" (3) in FIIL's original.

### 6.2 Read

```
TX  xx 27 01 00 02 0C 00
RX  xx 27 02 00 03 0C 01 <mode>
```

### 6.3 Write

```
TX  xx 2C 01 00 01 <mode>          # cid=44, cmdId=0, payload=mode
RX  xx 2C 02 00 01 00              # 6 bytes, payload[0]=00 means success
```

### 6.4 Write Matrix (All Five States Measured)

`exp_maf_five.py` output (device `B0:F1:A3:00:00:01`, firmware 3.0.1.1):

| Written | ACK | @0.6 s | @2.0 s | @3.0 s | Verdict |
|---|---|---|---|---|---|
| 0 Off | OK | 0 | 0 | 0 | ✓ |
| 1 Noise cancellation | OK | 0 | 1 | 1 | ✓ |
| 2 Open | OK | 1 | 2 | 2 | ✓ |
| 3 Awareness | OK | 2 | 3 | 3 | ✓ |
| 4 Wind | OK | 3 | 4 | 4 | ✓ |

Mode switching **must step up one level at a time** (only 0→1→2→3→4 could be measured; skipping levels was not tested).
Note that the value read at @0.6 s is still the **previous level**, which corroborates the latency conclusion in §6.5.

### 6.5 ⚠️ Take-Effect Latency of About 1.75–2.0 Seconds

Measured (`exp_maf_latency.py`, 400 ms polling + NOTIFY timing):

```
SET_MAF=2  →  ACK returns immediately (payload=00)
GET @0.42s  →  1  ✗ (old value)
GET @1.09s  →  1  ✗ (old value)
GET @1.75s  →  2  ✓  ← first GET to read the new value
NOTIFY      →  1.95s  ← headset pushes the new value
GET @3.0s   →  2  ✓
```

The earlier record of "1.5 s" was an underestimate caused by too coarse a sampling interval; **the real value falls in 1.7–2.0 s**.

**Therefore: after writing, you cannot read back in the same tick and judge success from it.** The correct approach:
1. Use `payload[0]==0` in the ACK to decide that **the command was accepted**
2. Use a `cid=40` NOTIFY or a **GET ≥2 s later** to decide that **the state has taken effect**
3. Reading the old value within 2 s ≠ failure

### 6.6 Verified Write Matrix

`evidence/set_matrix_20261025_015030.txt`:

| Item | ACK | @0.3s | @2.0s | @3.0s |
|---|---|---|---|---|
| MAF noise cancellation | OK | old value | new value | new value |
| Cinema | OK | new value | new value | new value |
| Low latency | OK | new value | new value | new value |

---

## 7. Cinema Mode (Spatial Audio)

### 7.1 Read

```
TX  xx 27 01 00 02 18 00          # cmdId=24 (0x18)
RX  xx 27 02 00 03 18 01 <0/1>
```

### 7.2 Write

```
TX  xx 32 01 00 01 <0/1>          # cid=50 (0x32), cmdId=0
RX  xx 32 02 00 01 00
```

Source: `com/fengeek/main/i/d/c.java:504-518 setSpatialAudio`.

### 7.3 ⚠️ `cmdId=11 (0x0B)` Is a Trap

Some online / published material gives cinema mode as `cmdId=0x0B`. **Measurement returns an empty frame `xx 27 02 00 00` (not supported)**. The real cinema bit is **24 (0x18)**.

### 7.4 Take-Effect Speed (0.5 s Measured)

Precise timing experiment (`exp_cinema_latency`): after sending `SET_SPATIAL=1`, 250 ms polling + a 220 ms read; **the first GET to read the new value is at 0.5 s**.
This is much faster than MAF (1.75 s), but **not "within 0.3 s"** — the "≤0.3 s" from an early single 500 ms sample was an underestimate due to insufficient resolution.

**There is no cid=40 unsolicited push at any point during a cinema toggle** (NOTIFY=None), so cinema **can only be confirmed by GET read-back**; NOTIFY cannot be used for read-back.

---

## 8. Low-Latency Mode

### 8.1 Read / Write

```
TX  xx 27 01 00 02 08 00          # GET, cmdId=8
RX  xx 27 02 00 03 08 01 <level>

TX  xx 25 01 00 01 <level>        # SET, cid=37 (0x25), cmdId=0
RX  xx 25 02 00 01 00
```

### 8.2 Level Semantics (⚠️ Model Branching: the Easiest Section in This Document to Get Wrong)

| Value | Older GAIA models | **Mate models (this Key Max / earType 46)** |
|---|---|---|
| 0 | Off | Off |
| 1 | Game | On (equivalent to "Game") |
| 2 | Music | **Rejected by the headset** |
| 3 | Video | **Rejected by the headset** |

**Mate models have only two levels, 0/1. There is hard evidence for this — it is not "not yet verified"; the headset explicitly refused:**

```
TX  xx 25 01 00 01 02        # SET_LOW_LATENCY=2 (music)
RX  xx 25 02 00 01 01        # ACK payload[0] = 0x01 → failure
TX  xx 25 01 00 01 03        # SET_LOW_LATENCY=3 (video)
RX  xx 25 02 00 01 01        # ACK payload[0] = 0x01 → failure
TX  xx 27 01 00 02 08 00     # GET to re-check
RX  xx 27 02 00 03 08 01 01  # read-back is still 1 → it really did not take effect
```

Evidence file: `evidence/ll4_20261025.txt`.

**So any upper-layer implementation must check the SET ACK byte** and must not "treat it as success once sent" — otherwise the UI will pretend to have switched to "Music / Video" while the headset has not moved at all. `_interpret_set_ack()` in this repository's `ble_worker.py` does exactly this; on receiving `setack(ok=False)` the GUI shows an explicit notice that "Mate models support only two levels".

> Side note: the earlier document's "2/3 unconfirmed, not exposed as semantic buttons" was a conservative but correct approach.
> It is now upgraded to "explicitly unsupported on Mate" based on the ACK-failure evidence; GAIA models still keep all four levels,
> and the GUI automatically adds the "Music / Video" buttons when it reads 2/3 (`app.py:_ensure_ll_buttons`).

### 8.3 Take-Effect Speed (0.45 s Measured, Reported via NOTIFY)

Precise timing experiment: after sending `SET_LOW_LATENCY=1`, **the cid=40 NOTIFY reports the new value at 0.45 s**.
But GET polling in the same round **always read the old value** (first GET to read the new value = None) — which means low latency **must be read back via NOTIFY**;
GET polling cannot keep up within this time window (there is a race between the headset's state switch and its GET replies).

**Practical advice**: to judge success after a low-latency write, prefer the `cid=40 cmd=8` NOTIFY, and otherwise wait ≥1 s before GET.
This repository's GUI uses the latter (`after(1200, ...)`), which reliably reads the new value in measurement.

---

## 8bis. Voice Prompt Volume (Tone Volume)

The **volume level** of built-in prompts and voice announcements; unrelated to media volume.

### 8bis.1 Read / Write

```
TX  xx 27 01 00 02 82 00          # GET, cmdId=130 (0x82)
RX  xx 27 02 00 03 82 01 <level>  # payload[0]=01 is the "field present" flag; the volume is in payload[1]

TX  xx 01 01 00 03 03 01 <level>  # SET, cid=1, cmdId=3, payloadLen=1
RX  xx 01 02 00 03 03 01 00       # last byte 00 = success
```

### 8bis.2 ⚠️ Wire Values and Labels Are **Reversed**

| Wire value | UI label |
|---|---|
| 0 | Volume high (default) |
| 1 | Volume medium |
| 2 | Volume low |
| 3 | Mute |

**Do not implement it on the intuition that "a larger number is louder"**. Basis: the FIIL+ `res/values/arrays.xml` array
`f43_select_operate_volume` (volume high / volume medium / volume low / mute), and
`setVolume(int)` in `com/fengeek/main/f046/fragment/F046HomeFragment.java:524-538`:
`==3 → "mute"`, `==2 → "prompt tone-low"`, `==1 → "prompt tone-medium"`, `==0 → "prompt tone-high"`.

### 8bis.3 Measurement Record

On 2026-10-25, levels were written one by one on a real device as 2→1→0; **all three ACKs succeeded and all read-backs agreed**. Evidence: `evidence/new3_20261025.txt`.
Regression script: `test_new3_live.py` (closed-loop verification of SET → ACK → read-back).

---

## 9. Bluetooth Multipoint (Multi-Device Connection)

### 9.1 Read / Write

```
TX  xx 27 01 00 02 80 00          # GET, cmdId=128 (0x80)
RX  xx 27 02 00 03 80 01 <0/1>

TX  xx 01 01 00 03 01 01 <0/1>    # SET, cid=1, cmdId=1, payloadLen=1
RX  xx 01 02 00 03 01 01 00
```

### 9.2 ⚠️ Toggling Inevitably Drops the BLE Link (Not a Bug, It Is Firmware Behavior)

**This must be stated clearly: whenever `SET_MP` is sent (on or off alike), the headset always drops BLE within a few milliseconds of returning the ACK.** Measured three times, with intervals of 3 ms / 46 ms / 16 ms:

| Timestamp | Event | After ACK |
|---|---|---|
| 15:49:55.645 | `TX SET_MULTIPOINT=1` → `00 01 01 00 03 01 01 01` | — |
| 15:49:55.814 | `RX RESP cid=1 cmd=1 len=1` → `03 01 02 00 03 01 01 00` (success) | — |
| 15:49:55.817 | Connection dropped | **+3 ms** |
| 15:50:24.419 | `SET_MULTIPOINT=0` ACK success | — |
| 15:50:24.465 | Connection dropped | **+46 ms** |
| 15:51:49.029 | `SET_MULTIPOINT=1` ACK success | — |
| 15:51:49.045 | Connection dropped | **+16 ms** |

Log original: `evidence/fiil_log_20261025_155218.txt` (383 lines).

**Two misjudgments that can be ruled out**:
- It is not caused by our writes — all three SET frames are perfectly well-formed and the headset returned a success ACK (`payload=00`) for each.
- It is not that the setting did not take effect — after reconnecting, the read-back did change (`15:49:26.833` read "off" → `15:50:09.679` read "on" → `15:50:40.456` read "off").

**Mechanism**: multipoint changes the headset's primary/secondary role (which device is primary), so the headset must rebuild its BLE-side connection parameters and therefore kicks the current link. This is consistent with the official FIIL+ app's behavior (its `case 6:` merely puts the UI into the disconnected state upon receiving the disconnect message).

**Recovery time**: after dropping, the headset **does not come back on its own**. Measured time to become connectable again is **8~13 seconds** (`15:49:55.817` dropped → `15:50:07.054` connected = 11.2 s; `15:50:24.465` dropped → `15:50:37.891` connected = 13.4 s).

**Three things the program must do** (already implemented in `ble_worker.py`):
1. **Set the flag before sending**: when `method == "set_multipoint"`, `_handle()` first sets `_reconnect_after_multipoint = True`, then sends the frame. It must be set before sending — the disconnect callback may arrive earlier than any assignment made afterwards.
2. **Route the disconnect callback through the "switching" branch**: instead of raising a red "connection lost" error, emit a neutral notice and start `_reconnect_loop()`.
3. **Backoff reconnection + redo the handshake**: `delays = (2, 3, 5, 8, 12)` seconds, 5 attempts in total; on success it runs `_connect_inner(keep_state=True)`, which includes `setUnlock` and a full state re-read. **`setUnlock` must be sent again after reconnecting**, otherwise the headset is silent for all queries (see §6). Only when every attempt fails is the user prompted to intervene manually.

**UI side**: for "switching", the `conn` event in `app.py` keeps the old readings and does not clear the screen, so the UI does not turn entirely into `--` during the 10-second reconnect window; it also does not send a verification GET after 1.5 s (the connection is necessarily down by then) — verification is instead completed by the handshake re-read after a successful reconnect.

---

## 10. EQ

### 10.1 Read

```
TX  xx 27 01 00 02 04 00
RX  xx 27 02 00 0E 04 0A 0E 08 06 04 02 00 01 FA FC FE 02
                     │  │  └────── 12-byte curve (signed) ──────
                     │  └ payloadLen = 0x0C = 12 (frame-header field)
                     └ cmdId = 4
```

**The payload is 12 bytes in total with no internal length prefix** (`0x0C` is already in the frame header as payloadLen).

The curve values are signed: `0A=+10, 0E=+14, 08=+8, 06=+6, 04=+4, 02=+2, 00=0, 01=+1, FA=-6, FC=-4, FE=-2, 02=+2`. The range is about **-6 ~ +14** (corresponding to the ±12 dB step).

### 10.2 Write (⚠️ Take-Effect Unconfirmed)

```
TX  xx 20 01 00 0C <12 bytes>     # cid=32 (0x20), cmdId=0
RX  xx 20 02 00 01 00             # ACK success
```

But **the read-back is still the old value** ("write consistency: inconsistent ✗" in `evidence/set4_20261025_003607.txt`).

**Conclusion: the payload structure for EQ writes is still not right; it is marked `unconfirmed`, is not exposed as a semantic button, and remains only in the raw Hex console.**

---

## 11. Media Control

| Function | Frame | Notes |
|---|---|---|
| Play/Pause | `xx 21 01 00 03 02 01 <0/1>` | cid=33, cmdId=2 |
| Next track | `xx 21 01 00 02 05 00` | cid=33, cmdId=5 |
| Previous track | `xx 21 01 00 02 04 00` | cid=33, cmdId=4 |
| Volume + | `xx 21 01 00 03 01 01 01` | cid=33, cmdId=1 |
| Volume − | `xx 21 01 00 03 01 01 00` | cid=33, cmdId=1 |

All of them returned an ACK in measurement, but **whether the headset actually executes them is unverified** (whether playback state changes needs to be confirmed by ear).

---

## 12. Unconfirmed Items

| # | Item | Status | Risk |
|---|---|---|---|
| 1 | EQ write | ACK succeeds but the read-back does not change | Medium — do not expose in the semantic UI |
| 2 | ~~Low-latency levels 2/3 (music/video)~~ | **Settled**: the headset on Mate models explicitly rejects them (ACK payload=01, read-back unchanged); this is not "unverified". The GUI offers only levels 0/1 and adds the buttons automatically when a GAIA model reports them | None (the user is told truthfully) |
| 3 | ~~MAF values 3 (awareness) / 4 (wind)~~ | **Confirmed usable** (`exp_maf_five.py`: SET 0→4 with a successful ACK at every level and a consistent read-back at @3.0 s); removed from the risk list | None |
| 4 | Wear detection (cmdId=9) | Empty frame; not supported on this model | None — hidden outright |
| 5 | Actual effect of media control | ACK present; not confirmed by ear | Low |
| 6 | The NOTIFY for `cid=40 cmd=0x83` | Structure not decoded (`0B 28 03 00 0E 83 0C D8 80 83 00 00 01 00 C3 0A B3 7F 60`) | Low — safe to ignore |
| 7 | ~~Exact attribution of the three battery values in this measurement~~ | **Confirmed**: F046/F042 are single-battery and read only `payload[1]`; F045/F047/F049/F051/F053 use three values (left/right/case) (verified model by model against the source + 20 samples). Removed from the risk list | None |
| 8 | Business meaning of PID / left and right earbud UUID | Replies exist; semantics not decoded | None — read-only, no control |
| 9 | Connecting SPP and BLE GATT at the same time | Not tested | Medium — the phone module must take care |
| 10 | Command differences on other models (non-Key Max) | Not tested | Medium — adaptation per model table is needed |
| 11 | ~~Voice prompt volume~~ | **Confirmed**: cid=1 cmd=3, four levels (0 high / 1 medium / 2 low / 3 mute, wire values reversed); three levels measured with ACK + read-back all consistent | None |
| 12 | Reconnect timing after a multipoint toggle | BLE drops immediately on toggle and can reconnect within 2~3 s; **"whether setUnlock must be redone after reconnecting" has been proven by `test_new3_live.py` to be required** | Low — the program already redoes the handshake automatically |

---

## 13. Porting Notes for the Android Module

The GATT implementation of the phone-side module (`FiilGatt.kt`) corresponds exactly to this document:

| This document | Android implementation |
|---|---|
| §1 write `ff17` / notify `ff18` | `FiilGatt.CHAR_WRITE` / `CHAR_NOTIFY` |
| §2 frame format | `FiilMate.buildFrame` / `parse` / `drain` |
| §4 setUnlock | `FIILProtocol.unlock()` → the first entry in `handshake()` |
| §5 battery | `FiilGaia.parseBatteryLevel` |
| §6 MAF | `FIILProtocol.setMaf` |
| §7 cinema | `FIILProtocol.set_cinema` corresponds to `setCinemaMode` |
| §2.5 seq cannot correlate | **Neither side may match by seq** |
| §2.2 lenField==1 | **The drain on both sides must support 6-byte short frames** |

**The three easiest things to miss when porting**: ① the setUnlock handshake; ② the 6-byte SET reply for lenField==1; ③ the type byte must accept all three of 0x01/0x02/0x03.

---

## 14. Evidence File Index

| File | Content |
|---|---|
| `evidence/gatt_20261025_001610.txt` | GATT service / characteristic enumeration |
| `evidence/log_unlock17_20261025_001932.txt` | ff17 write of setUnlock succeeded |
| `evidence/log_unlock16_20261025_001943.txt` | ff16 write of setUnlock **no response** (decoy evidence) |
| `evidence/table_20261025_002133.txt` | First round of the full GET table |
| `evidence/unverified_cmdid_20261025_014438.txt` | **Full on-device verification of all 18 cmdIds** |
| `evidence/set4_20261025_003607.txt` | EQ write / media control / multipoint link drop |
| `evidence/maf_timing_20261025_012651.txt` | MAF take-effect latency (coarse sampling, underestimated as 1.5 s; see §6.5 for the precise value) |
| (experiment script) | **Precise timing**: 400 ms polling + NOTIFY timing; noise cancellation takes effect at 1.75 s / 1.95 s (the script is not distributed with the repository) |
| `evidence/set_matrix_20261025_015030.txt` | SET three-command timing matrix |

