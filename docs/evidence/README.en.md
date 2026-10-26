[简体中文](README.md) | **[English](README.en.md)**

# Capture Evidence Index

This directory holds the raw captures and probe logs referenced throughout the documentation. All logs come from debugging sessions on one device we own (Mate model / earType 46 / firmware 3.0.1.1 / 2026-10-25) and were uniformly redacted before publication.

**Redaction rules**: Bluetooth addresses are replaced with documentation-reserved ranges such as `B0:F1:A3:00:00:01` (the OUI and the delta between last bytes are preserved); absolute paths are rewritten as relative paths; no accounts, serial numbers or personal information are included.

---

## Index by Document Section

### `PROTOCOL_PC.md`

| Section | Conclusion | Evidence file |
|---|---|---|
| §1 UUID table | Service `fdb3`, write `ff17`, receive `ff18`, distractor `ff16` | `gatt_20261025_001610.txt` / `.json` |
| §1.1 Pitfall 1 | Writing `ff16` gets no response; writing `ff17` replies immediately | `log_unlock16_20261025_001943.txt` (no reply)<br>`log_unlock17_20261025_001932.txt` (reply received) |
| §3 Full GET table | All 18 cmdIds verified on a real device | `unverified_cmdid_20261025_014438.txt`<br>`table_20261025_002133.txt` |
| §5 Battery parsing | Stable `00 4D 00` across 20 samples (single battery) | `battery_20261025.txt` |
| §6.4 MAF five states | SET 0→4 ACKs successfully level by level, read-back matches | `maf_five_20261025.txt` |
| §6.5 Take-effect latency | 400 ms polling + NOTIFY timing, measured 1.75 s / 1.95 s | `maf_timing_20261025_012651.txt` |
| §8bis Voice-prompt volume | Written level by level 2→1→0, ACK and read-back all match | `new3_20261025.txt` |
| §9 Multipoint connection | BLE drops the link within 3 / 46 / 16 ms after switching; reconnecting takes 8–13 s | `fiil_log_20261025_155218.txt` |
| §10 EQ write | ACK succeeds but the read-back is unchanged (**Unconfirmed**) | `set4_20261025_003607.txt` |
| §12 Unconfirmed item 6 | Raw NOTIFY bytes for `cid=40 cmd=0x83` | `set4_20261025_003607.txt` |
| SET timing matrix | ACK / read-back timing for the three kinds of SET commands | `set_matrix_20261025_015030.txt` |
| SET regression | SET → ACK → read-back closed loop | `set_verify_20261025_012511.txt` |
| Low latency | Level writes and rejection behaviour | `ll4_20261025.txt` |

---

## File List

| File | Size class | Contents |
|---|---|---|
| `gatt_20261025_001610.txt` | Short | GATT service / characteristic enumeration result |
| `gatt_20261025_001610.json` | Short | Same as above, JSON structure |
| `log_unlock17_20261025_001932.txt` | Short | Writing `setUnlock` to `ff17` succeeds |
| `log_unlock16_20261025_001943.txt` | Short | Writing `setUnlock` to `ff16` gets **no response** (evidence of the distractor) |
| `table_20261025_002133.txt` | Medium | First-round probe of the full GET command table |
| `unverified_cmdid_20261025_014438.txt` | Medium | Full verification of the 18 cmdIds (with source-line cross-references) |
| `set_verify_20261025_012511.txt` | Medium | SET regression: write → ACK → read-back |
| `set_matrix_20261025_015030.txt` | Medium | Full SET timing matrix |
| `set4_20261025_003607.txt` | Medium | EQ write, media control, multipoint link drop, `cid=40 cmd=0x83` report |
| `maf_five_20261025.txt` | Short | MAF five-state level-by-level write and read-back |
| `maf_timing_20261025_012651.txt` | Short | Coarse sampling of MAF take-effect latency |
| `new3_20261025.txt` | Short | Closed-loop verification of the three voice-prompt volume levels |
| `ll4_20261025.txt` | Short | Low-latency level probe |
| `battery_20261025.txt` | Short | 20 battery samples |
| `fiil_log_20261025_155218.txt` | Long | Full session log: scan, handshake, status reads, multipoint switching, link drop and reconnect |

---

## Log Format Conventions

### Session logs (`fiil_log_*.txt`)

```
15:49:24.613  TX  setUnlock (connection handshake)  00 27 01 00 04 FF 02 02 34
15:49:24.660  RX  RESP cid=39 cmd=255 len=1         00 27 02 00 03 FF 01 FF
```

On the left is the local timestamp; `TX` means host-to-headset, `RX` means a reply from the headset.

### Probe logs (`table_*` / `set_*` / `unverified_*`)

```
[TX] setUnlock                00 27 01 00 04 FF 02 02 34
     [RX] 00 27 02 00 03 FF 01 FF            RESP cid=39 cmd=255 len=1
```

After `[TX]` comes the command name, with the hex bytes on the right; the `[RX]` line holds the reply bytes and the parsed result.

### Hex Conventions

Bytes are separated by a single space and are all uppercase. When `xx` appears, it denotes the `seq` field at that position (the actual logs contain concrete values).

---

## Reproducing This

These logs correspond to one debugging session on one specific device; they are **not a repeatable test suite**. To reproduce:

1. Prepare a FIIL headset of a Mate model (earType 45–53)
2. Handshake in the order given in `PROTOCOL_PC.md` §4
3. Send the commands in `unverified_cmdid_20261025_014438.txt` one by one

Different firmware versions may return different values. **If your results differ from the logs in this directory, that is new data, not an error** — please submit it as described in [CONTRIBUTING.en.md](../../CONTRIBUTING.en.md) and include your firmware version.
