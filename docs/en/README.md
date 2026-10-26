[简体中文](../README.md) | **[English](README.md)**

# Documentation Index

Three documents, three ways to read them. Pick the path that matches your goal.

This is the English translation. **The Chinese text is the authoritative version**; the originals sit one level up under `docs/`, and this `docs/en/` tree mirrors their layout one-to-one.

---

## I want to try sending a command

**Read** [`PROTOCOL_PC.md`](PROTOCOL_PC.md)

This is a BLE GATT field manual verified command by command on one real pair of headphones (Mate model / earType 46 / firmware 3.0.1.1).

| Section | What you get |
|---|---|
| §0 Conclusions at a Glance | Every key conclusion on a single page |
| §1 UUID table | Service, write characteristic, notify characteristic, and three measured pitfalls |
| §2 Frame format | Framing, frame slicing, stream reassembly, and the `lenField == 1` trap |
| §3 Full GET table | The complete bytes and replies for 18 query commands |
| §4 Connection initialization | The handshake order, and why `setUnlock` must be sent first |
| §6 Noise cancellation | The five-state write matrix and the 1.75–2.0 s take-effect latency |
| §9 Multipoint connection | Why switching always drops the link, and the reconnect timing |

The matching raw captures are in [`evidence/`](../evidence/).

---

## I want to understand the whole protocol

**Read** [`PROTOCOL_FIIL.md`](PROTOCOL_FIIL.md)

This is the specification overview. It lays out the **three command namespaces** that coexist on the same pair of headphones:

| Command namespace | Characteristics | Models it applies to |
|---|---|---|
| **GAIA** | 16-bit command word (`0x02xx` / `0x07xx`), frames start with `0xFF` | Older mainstream models |
| **Mate** | `classifyId + cmdId`, two 8-bit values, frames start with `0x00` | The F045 ~ F053 generation |
| **Doubao 0x55AA** | A separate voice-service frame | earType 47 / 48 |

Key sections: **§0 Key findings at a glance**, **§3.12 GET and SET are two completely different frame semantics** (the biggest trap in practice), **§5 Models and routing**.

---

## I want to know how the official App does it

**Read** [`REVERSE_ENGINEERING_REPORT.md`](REVERSE_ENGINEERING_REPORT.md)

A static unpacking analysis of FIIL+ 3.5.2. This document answers "what does the App look like on the inside":

- **§2** Connection flow: broadcast discovery → vendor-data parsing → link establishment, taken apart layer by layer
- **§3** The five parallel frame formats (GAIA / Bluetrum / F042 BLE / Doubao / Airoha)
- **§4** 190 `FFCOMMAND`s split into 13 categories by function
- **§5** earType → model mapping table
- **§8** Anti-reversing design points: constants read via reflection, command ids never at the call site

The complete command table is in [`reference/_cmd_table.txt`](../reference/_cmd_table.txt) (190 lines, `name|decimal value`).

---

## Reference Material

| Path | Contents |
|---|---|
| [`evidence/`](../evidence/) | Raw captures and probe logs, named by timestamp; the body text names the file it references |
| [`reference/_cmd_table.txt`](../reference/_cmd_table.txt) | The full table of 190 `FFCOMMAND`s exported from the decompiled artifacts |

---

## Reading Conventions

- Items marked **`Unconfirmed`** have not yet been verified on a real device, or the verification result did not match expectations. Please do not use them as conclusions.
- Items marked **`Disproved`** are claims that were once popular or adopted early by this project, but have since been overturned by measurement. They are kept so that later readers do not repeat the same mistake.
- The `xx` in a byte string denotes the `seq` field at that position; fill it in from the rolling counter when actually sending.
- All `file:line` references point at the decompiled artifacts of FIIL+ 3.5.2 (`jadx_out/` holds the Java decompilation, `apktool_out/` holds resources and smali). **These artifacts are not distributed with this repository.**
