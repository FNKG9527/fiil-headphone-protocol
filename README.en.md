[简体中文](README.md) | **[English](README.en.md)**

# FIIL Headphone Bluetooth Protocol Reverse-Engineering Notes

> A write-up of the static reverse-engineering results for FIIL (Feier)'s official control app **FIIL+ 3.5.2**, together with a command-by-command on-device verification record on one real pair of headphones.
> There is only one goal: **to let third parties speak FIIL's protocol correctly on their own devices.**

![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)
![Docs](https://img.shields.io/badge/docs-3%20%E4%BB%BD-blue.svg)
![Transport](https://img.shields.io/badge/transport-BLE%20GATT%20%7C%20SPP-orange.svg)
![Status](https://img.shields.io/badge/%E7%8A%B6%E6%80%81-%E5%AE%9E%E6%9C%BA%E9%AA%8C%E8%AF%81-green.svg)

---

## What This Is

FIIL headphones ship no public developer protocol documentation. Everything the official app can do — noise-cancellation levels, EQ, cinema mode, low latency, multipoint, voice-prompt volume — runs on one **private binary protocol**, and within FIIL's own product line that protocol has split into **three coexisting command namespaces**.

This repository takes that link apart and writes it down clearly:

| What you get | Description |
|---|---|
| **Frame format** | The complete byte layout of the three command namespaces, the three meanings of the length field, framing and stream reassembly |
| **Command table** | 190 `FFCOMMAND`s (GAIA), Mate's `classifyId + cmdId` pair, and the Doubao `0x55AA` frame |
| **Semantics** | The value range of every writable item, the original wording of the level labels, and the mapping between wire values and labels (several of them are **reversed**) |
| **Timing** | The real take-effect latency of each command type, the order of ACK and status read-back, and link-drop/reconnect behaviour |
| **Pitfalls** | Places that the documentation widely copies wrong, plus the conclusions this project has disproved by measurement |

Every conclusion carries its source: either a `file:line` in the decompiled artifacts, or a raw capture log under `docs/evidence/`. **No source, no conclusion.**

---

## Documentation Map

| Document | For | Contents |
|---|---|---|
| [**PROTOCOL_FIIL.en.md**](docs/PROTOCOL_FIIL.en.md) | People who want to **understand** the protocol | The complete specification of the three command namespaces: GAIA / Mate / Doubao, model routing, the semantic difference between GET and SET, and the list of unconfirmed items |
| [**PROTOCOL_PC.en.md**](docs/PROTOCOL_PC.en.md) | People who want to **send commands themselves** | A BLE GATT field manual measured on one real pair of headphones: UUIDs, handshake, the full table of 18 GETs, five noise-cancellation states, battery parsing, take-effect latency |
| [**REVERSE_ENGINEERING_REPORT.en.md**](docs/REVERSE_ENGINEERING_REPORT.en.md) | People who want to **know how the app does it** | An unpacking analysis of FIIL+ 3.5.2: four transport links, the 190-command classification, the model table, and the anti-reversing design points |

To get hands-on right away: read **§0 Conclusions at a Glance** and **§4 Connection Initialization Sequence** in `PROTOCOL_PC.md` first — ten minutes is enough to send your first command.
To understand the whole picture: read **§0 Key findings at a glance** and **§5 Models and routing** in `PROTOCOL_FIIL.md`.

---

## Protocol at a Glance

```
Frame layout (both command namespaces share the same transport channel)

  Byte  0     1     2      3       4         5        6          7..
       seq   cid   type   00   lenField   cmdId   payloadLen   payload

  seq       1-byte rolling counter (low 4 bits)
  cid       Command family id. GET fixes it at 39 (0x27); SET uses the command family id
  type      0x01 REQUEST / 0x02 RESPONSE / 0x03 NOTIFY
  lenField  0 → empty frame (the headset does not recognise this command)
            1 → 6-byte short frame, the 6th byte is the success flag (the SET reply)
            ≥2 → total length = 5 + lenField (all GET replies and requests)
```

A few of the easiest things to get wrong, stated up front:

- **`lenField == 1` is not an "empty frame of length 1"** — the 6th byte is the success flag. Slicing frames on the rule "<2 means a 5-byte empty frame" swallows the SET success byte entirely.
- **`seq` cannot be used to correlate a request with its response**. The headset replies with its own rolling counter.
- **`type` must accept `0x01/0x02/0x03` all at once**. Accepting only `0x01` drops every reply — the classic reason why "the UI always shows not connected".
- **The first frame after connecting must be `setUnlock`**, otherwise the headset stays silent for every command.
- **A SET takes effect asynchronously**: noise cancelling about 1.75–2.0 s, cinema 0.5 s, low latency 0.45 s. Read back immediately after writing and what you read is always the old value.
- **Toggling multipoint always drops the link**. This is firmware behaviour, not a bug. `setUnlock` must be redone after reconnecting.

---

## Coverage

| Dimension | Covered |
|---|---|
| Command namespaces | GAIA (16-bit command word), Mate (`classifyId + cmdId`), Doubao `0x55AA` (structure not expanded) |
| Transport layer | Classic Bluetooth SPP/RFCOMM, BLE GATT (`fdb3` / `ff17` write / `ff18` receive) |
| On-device verification | One Mate model (earType 46 / Key Max, firmware 3.0.1.1): the full BLE GATT link, 18 GETs, 5 noise-cancellation states, 3 voice-prompt volume levels, multipoint-switch link-drop timing |
| Static derivation | Coverage differences across the 7 model command classes, the model → earType mapping, the four parallel transport stacks |
| **Not covered** | On-device verification of other models, writing custom EQ curves, the payload structure of the Doubao voice frame, the OTA flow |

Anything unconfirmed is always marked `Unconfirmed` in the documentation and collected in each document's "Unconfirmed Items" list. **Please do not use unconfirmed items as conclusions.**

---

## Repository Layout

```
.
├── README.md                        # This file
├── LICENSE                          # CC BY-SA 4.0
├── NOTICE                           # Trademark and asset attribution notice
├── DISCLAIMER.md                    # Usage boundaries and compliance statement
├── CONTRIBUTING.md                  # How to submit errata
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── CHANGELOG.md
└── docs/
    ├── README.md                    # Documentation index (with reading order)
    ├── PROTOCOL_FIIL.md             # Protocol specification overview
    ├── PROTOCOL_PC.md               # BLE GATT field manual
    ├── REVERSE_ENGINEERING_REPORT.md# Unpacking analysis report
    ├── evidence/                    # Raw capture logs (redacted)
    └── reference/
        └── _cmd_table.txt           # Raw export of the 190 FFCOMMANDs
```

---

## Data Redaction Notes

The documentation in this repository comes from the debugging process on one real device, and an **irreversible replacement** was carried out before publication:

| Category | Treatment |
|---|---|
| Bluetooth address / MAC | All replaced with documentation-reserved ranges (`B0:F1:A3:00:00:0x`, `D8:80:83:00:00:0x`, `16:F3:78:00:00:0x`, `62:F8:9E:00:00:0x`); the OUI and the relationship between the last bytes are kept so that the original reasoning still holds |
| Third-party devices in scan records | Treated the same way: only the address structure is kept, no value that could be tied to a real device |
| Absolute paths on this machine | Rewritten as repository-relative paths (`jadx_out/…`, `apktool_out/…`, `docs/evidence/…`) |
| Build environment paths | Replaced with placeholders such as `<JAVA_HOME>` |
| Personal identifiers such as package names | Private module and package names removed and replaced with neutral descriptions such as "a third-party Android controller" |
| First-person phrasing | Rewritten neutrally ("this measurement", "this model") |

**Not included**: the original APK file, decompilation output, app resources, any keys or certificates, any account or personal information.
The APK hash is kept in the sample-information section of `REVERSE_ENGINEERING_REPORT.md`, purely to identify the version that was analysed.

If you still find information in this repository that can be tied to a specific individual, please get in touch as described in [SECURITY.en.md](SECURITY.en.md); we will deal with it immediately.

---

## Usage Boundaries

The purpose of these documents is **interoperability research and local control**. Permitted: writing a controller for your own headphones, learning the protocol, writing test tools.

Not permitted: bypassing firmware checks or performing unauthorised OTA, redistributing FIIL's private implementation or resources as your own work, or claiming official FIIL compatibility in a commercial product.

The full terms are in [DISCLAIMER.en.md](DISCLAIMER.en.md).

---

## License

The documentation is released under **[CC BY-SA 4.0](LICENSE)** — you may copy, modify and use it commercially, provided you give attribution and distribute it under the same license.

FIIL, FIIL+ and related names are trademarks of Feier Technology. This repository has no affiliation with Feier Technology, has not been authorised or endorsed by it, and exists for interoperability research only. See [NOTICE](NOTICE) for details.

---

## Contributing

Found an error, have measured data for another model, or have already verified an "Unconfirmed" item — issues and PRs are welcome; the format is in [CONTRIBUTING.en.md](CONTRIBUTING.en.md). **Errata with capture logs are merged first.**
