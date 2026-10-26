[简体中文](../CONTRIBUTING.md) | **[English](CONTRIBUTING.md)**

# Contributing Guide

This repository is a documentation project. The most valuable contributions are **evidence-backed errata** and **measured data for new models**.

---

## Before You Submit

### 1. First Check That This Is Not a Known Unconfirmed Item

Every document ends with an "Unconfirmed Items" list. If your question is already in that list, the answer is "not yet confirmed" and there is no need to open an issue.

### 2. Check That Your Conclusion Has Evidence

The hard rule of this repository: **conclusions without a source are not merged.**

There are only two legitimate sources for a conclusion:

| Source | Requirement |
|---|---|
| **Static analysis** | Give a `file:line`. The file path is relative to the decompilation output root (`jadx_out/sources/...` or `apktool_out/...`); do not write absolute paths |
| **On-device capture** | Attach the raw log. The log must contain the complete TX/RX bytes, the timestamps and the command name |

"I remember it being this way", "someone on the internet said so" and "the AI said so" are not sources.

### 3. One Conclusion, One Piece of Evidence

Do not stuff five mutually independent conclusions into one PR. Split them up, so that each one can be reviewed and rolled back on its own.

---

## What to Submit

### Errata (Most Welcome)

If you find a byte written incorrectly in the docs, a level whose semantics are reversed, or a file line number that has drifted — just open a PR to fix it, but please observe the following:

- **Minimal change**: fix only the one thing that is wrong; do not reformat, change punctuation or reword along the way.
- **Preserve the evidence citation**: if the original text has a `file:line` or a log citation, your change must keep it, or provide a new one.
- **If the original text has been disproven**: do not simply delete it. Mark it **`Disproven`** instead and state what evidence overturned it — leaving a trace is more valuable than erasing it, so that later readers do not repeat the same mistake.

### Measured Data for New Models

If you have got this working on another FIIL model, contributions are welcome:

1. **Model information**: model name, earType (the `GET_EARTYPE` reply packet or `GET_DEVICE_NAME`), firmware version
2. **Transport layer**: whether SPP or BLE GATT, which UUID works and which does not
3. **Command coverage differences**: which commands the model accepts, which return an empty frame, and which the ACK rejects
4. **Raw logs**: the complete TX/RX, redacted and placed under `docs/evidence/`

For the format, please copy the verification table in `PROTOCOL_PC.md` — one record per model, which makes side-by-side comparison easy.

### Turning "Unconfirmed" into "Confirmed"

The three highest-priority items:

- `EQ custom curve writing` (`PROTOCOL_PC.md` §10.2 / `PROTOCOL_FIIL.md` §6, item 11)
- `payload structure of the Doubao 0x55AA frame` (`PROTOCOL_FIIL.md` §6, item 3)
- `command differences between other models and Key Max` (`PROTOCOL_PC.md` §12, item 10)

If you have verified one of these, please provide evidence for **both the successes and the failures** — reporting only the successes makes it easy to mistake "it happened to work" for "the protocol supports it".

---

## Redaction Requirements (Mandatory)

Before opening a PR, you **must** check that your logs and documents do not contain any of the following:

| Category | How to handle |
|---|---|
| Bluetooth MAC addresses | Replace with a documentation-reserved address such as `B0:F1:A3:00:00:01`, keeping the OUI prefix |
| Third-party devices found while scanning | Replace in the same way; do not include the device name |
| Absolute paths on your machine | Change to relative paths or placeholders |
| Accounts, serial numbers, IMEI, location | **Delete outright**; do not replace |
| Personal identifiers (real name, email, username) | Remove from both documents and logs |

If you are not sure whether a value counts as sensitive, treat it as sensitive.

---

## Formatting Conventions

### Language

Chinese is the primary language, and technical terms keep their original English form (`payload`, `lenField`, `ACK` and the like are not translated).

### Byte Strings

```
TX  xx 27 01 00 02 0C 00          # GET, cmdId=12
RX  xx 27 02 00 03 0C 01 <mode>
```

- `xx` denotes the `seq` field; in an actual transmission it is filled with the rolling counter
- End each line with a comment explaining its meaning
- For multi-byte fields where byte order matters, state explicitly whether it is big-endian or little-endian

### Tables

Use a table for verification results, with the columns fixed as: **Item / Command / Expected / Measured / Verdict**. The verdict is one of `✓`, `✗` or `Unconfirmed`.

### Headings

`##` is level one, `###` is level two and `####` is level three. Do not skip levels.

---

## PR Description

```markdown
## What changed
(one sentence)

## Why
(what the problem is, or what data this adds)

## Evidence
(file:line, or the log filename under evidence/)

## Redaction self-check
- [ ] No real MAC / no third-party device address
- [ ] No absolute path
- [ ] No personal identifier
```

---

## Code of Conduct

Participation in this project means you agree to abide by [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

---

## License

By submitting a contribution, you agree to license it under **[CC BY-SA 4.0](../LICENSE)**.
