[简体中文](../DISCLAIMER.md) | **[English](DISCLAIMER.md)**

# Usage Boundaries and Compliance Statement

## What This Repository Is

This repository is an **interoperability research documentation project**: through static analysis of a publicly distributable Android app (FIIL+ 3.5.2), it documents the Bluetooth communication protocol of FIIL headphones, with verification performed on our own devices.

The goal: to let third parties (including the owners of the headphones themselves) understand and connect to hardware they paid for, without having to rely on the official app. In many jurisdictions this constitutes **reverse engineering for interoperability purposes**, which is generally protected by law.

## What This Repository Is Not

- **It is not** official FIIL documentation. It has no affiliation whatsoever with Feier Technology and has not been authorized, endorsed or approved by it.
- **It does not contain** the official app's installation package, decompilation output, resource files, icons or any copyright-protected material.
- **It does not contain** any keys, certificates, accounts, personal information or data that could be linked to a specific device holder.
- **It does not provide** any means of bypassing device security mechanisms.

## Prohibited Uses

When using the contents of this repository, you **must not** use them to:

1. **Bypass firmware verification or signing**, or perform unauthorized OTA flashing.
2. **Redistribute** FIIL's proprietary protocol implementation, App resources or other copyright-protected material.
3. **Claim, in a commercial product or promotion,** official compatibility with, partnership with, or authorization from FIIL.
4. **Spoof a device identity** in order to access someone else's account, cloud service or pairing relationship.
5. Any activity that violates local law, infringes the rights of others, or damages another person's device.

## About the "Unconfirmed" Items

Items marked `Unconfirmed` in the documentation mean:

- they come solely from static code reading and have not been verified on a real device; or
- the on-device verification result did not match expectations, and the cause has not yet been identified.

**Any consequences of using these items as conclusions are borne by the user.** The documentation has done its best to flag the uncertainty; please do not ignore these markers.

## About Third-Party Trademarks

FIIL, FIIL+, FIIL Key Max, FIIL CC Pro2, FIIL Atom and other names and logos are trademarks or registered trademarks of Feier Technology (and its affiliates).

This repository mentions these names within the scope of **nominative fair use**, to identify the device models the protocol applies to. No sponsorship, endorsement or association is implied.

Other trademarks mentioned in this document (Bluetooth, Android, HyperOS, MIUI, LSPosed, Doubao, etc.) belong to their respective owners.

## About the Capture Data

The logs under `docs/evidence/` come from debugging sessions on a device we own, and were redacted before public release:

- all Bluetooth addresses are replaced with documentation-reserved ranges (the OUI prefix and the relationships between last bytes are kept, so that the original reasoning remains readable);
- third-party device addresses in the scan records are replaced in the same way;
- absolute paths on the local machine are rewritten as repository-relative paths;
- they contain no accounts, serial numbers, IMEI, location or any field that could be linked to an individual.

The logs contain only BLE characteristic values and frame bytes — no audio, contacts or user data.

## Disclaimer

The contents of this repository are provided "as is", without any express or implied warranty, including but not limited to the warranties of merchantability, fitness for a particular purpose and non-infringement.

The authors accept no liability for device damage, voided warranties, data loss or legal consequences arising from the use of this repository's contents.

**If you do not agree with any of the above, please do not use the contents of this repository.**
