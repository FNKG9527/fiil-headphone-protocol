[简体中文](SECURITY.md) | **[English](SECURITY.en.md)**

# Security Policy

## What This Repository Is

This repository is a **documentation project**: it contains only Markdown, plain text and capture logs — no executable code, build scripts or dependencies.

Supply-chain risks in the conventional sense (dependency poisoning, execution at build time) therefore do not apply here. But there is another class of security issue that needs to be taken seriously.

---

## What to Report

### 1. Redaction Oversights (Highest Priority)

If you find **information that has not been redacted** in this repository, including but not limited to:

- Bluetooth MAC addresses that can be linked to a real device
- Scan records of third-party devices
- Absolute paths, usernames or hostnames of the local machine
- Any account, serial number, IMEI or location information
- Any personal identifier

**This is the most important class of issue to fix in this repository.** Please report it immediately.

### 2. Dangerous Advice in the Documentation

If a passage of documentation is worded in a way that could lead a user to **damage their device** or **break the law**, for example:

- A missing critical handshake order, causing commands to be sent while the device is in the wrong state
- An OTA / firmware writing procedure that lacks safeguards
- An operation sequence that will actually brick the device

### 3. Copyright and Trademark Issues

If any part of this repository infringes the rights of you or of an organization you represent, please report that as well.

---

## How to Report

**Please do not open a public issue.**

Prefer GitHub's private vulnerability reporting feature:

> Repository page → **Security** tab → **Report a vulnerability**

If you do not see that entry point, or if you need to attach large files (such as a complete log), please state **only** that "there is private material to submit" in an issue and leave your contact details; a maintainer will reply with a private channel.

---

## What to Include in a Report

| Item | Description |
|---|---|
| Location | File path + line number, or a specific section |
| Content | The original text suspected of being a leak (the smallest excerpt is enough) |
| Impact | What it can be linked to (device / person / location) |
| Suggestion | How you would like it fixed (delete / replace with a placeholder / reword) |

---

## Response Times

| Stage | Time limit |
|---|---|
| Acknowledgement of receipt | Within 3 days |
| Initial assessment | Within 7 days |
| Fix of redaction issues | Fixed **immediately** after confirmation — take it down first, discuss later |
| Other issues | We will share the plan after assessment |

For redaction issues we follow the principle of **fix first, discuss later** — we would rather over-delete than leave personal information in a public repository.

---

## Out of Scope

- Vulnerabilities in the **earphone firmware** itself (this repository does not ship firmware; please contact the manufacturer directly)
- Vulnerabilities in the **FIIL+ App** (same as above)
- Plain technical disagreements (that is what issues are for; see [CONTRIBUTING.en.md](CONTRIBUTING.en.md))

---

## Acknowledgements

Responsible disclosures are recorded in the Security section of [CHANGELOG.en.md](CHANGELOG.en.md) once fixed (unless you ask to remain anonymous).
