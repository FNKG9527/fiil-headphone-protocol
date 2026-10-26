[简体中文](../CHANGELOG.md) | **[English](CHANGELOG.md)**

# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/lang/zh-CN/).

---

## [1.0.0] — 2026-10-26

Initial public release.

### Added

- **`docs/PROTOCOL_FIIL.md`** — protocol specification overview
  - Complete frame structures of the three command spaces (GAIA / Mate / Doubao `0x55AA`)
  - Connection strategy and fallback order for the two transport layers (classic Bluetooth SPP/RFCOMM + BLE GATT)
  - The semantic difference between GET and SET frames (`cid=39` fixed vs. the `cid=` command family)
  - Coverage difference matrix for the seven model command classes
  - earType → model mapping table
  - List of unconfirmed items (14)

- **`docs/PROTOCOL_PC.md`** — BLE GATT on-device verification manual
  - Full Service / Characteristic UUID table, plus on-device disproof of the `ff16` decoy
  - Frame format and the framing rules for the three `lenField` values
  - Complete bytes and reply packets for the 18 GET commands
  - Connection initialization handshake sequence (why `setUnlock` is necessary)
  - Model-specific branching rules for battery parsing (single battery vs. three values)
  - Write matrix for the five MAF noise-cancellation states and the 1.75–2.0 s activation delay
  - Read/write semantics and activation timing for cinema mode, low latency, prompt-tone volume and multipoint connection
  - List of unconfirmed items (12)

- **`docs/REVERSE_ENGINEERING_REPORT.md`** — unpacking analysis report for FIIL+ 3.5.2
  - Routing logic of the four parallel transport links (GAIA SPP / Bluetrum / Airoha / F042 BLE)
  - Byte-by-byte layout of the three versions of the vendor broadcast data (Beacon)
  - The 190 `FFCOMMAND` entries grouped into 13 categories by function
  - Model comparison table and routing entry point
  - Anti-reverse-engineering design points (reflective reads of constants, command numbers not located at call sites)

- **`docs/evidence/`** — 15 raw capture and probing logs
- **`docs/reference/_cmd_table.txt`** — raw export of the 190 `FFCOMMAND` entries

- **Repository infrastructure**
  - CC BY-SA 4.0 license and trademark attribution notice (`NOTICE`)
  - Usage boundaries and compliance statement (`DISCLAIMER.md`)
  - Contributing guide, code of conduct, security policy

- **Chinese / English bilingual editions**
  - All 11 Markdown documents have an English counterpart, kept in `en/` and `docs/en/` (a mirror tree; Chinese is the authoritative text)
  - Each document carries a language switcher at the top; the two languages are each complete and readable on their own rather than interleaved in one file
  - Identifiers, byte sequences, UUIDs, address placeholders and section numbers inside code fences are byte-for-byte identical across both languages

### Security

- All documentation and logs have been redacted: Bluetooth addresses replaced with documentation-reserved ranges, absolute paths rewritten as relative paths, private module names and package names replaced with neutral descriptions, and first-person phrasing rewritten in neutral terms
- Contains no APK, decompilation output, App resources, keys, or any account or personal information

---

## Versioning Convention

| Change type | Version digit | Example |
|---|---|---|
| New model / new command / new documentation | MINOR | Add the measured coverage matrix for a model |
| Errata, wording fixes, formatting adjustments | PATCH | Fix an incorrect `cmdId` value |
| A fact is disproven, or the protocol understanding changes fundamentally | MAJOR | Discover a fourth value for `lenField` |

---

## About "Disproven" Entries

Conclusions that have been overturned **are not** removed from the documentation; they are marked `Disproven` and kept. This is intentional:

A wrong claim that was once widely believed is itself valuable information — it lets those who come later avoid taking the same detour.

Such changes are recorded as **MAJOR**, because the protocol understanding has changed substantially.
