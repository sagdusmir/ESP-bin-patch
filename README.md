# ESP Bin Patch
[![GitHub release](https://img.shields.io/github/v/release/sagdusmir/ESP-bin-patch?style=flat-square&logo=github&color=blue)](https://github.com/sagdusmir/ESP-bin-patch/releases/latest)
[![GitHub Release Date](https://img.shields.io/github/release-date/sagdusmir/ESP-bin-patch?style=flat-square&logo=github&color=blue)](https://github.com/sagdusmir/ESP-bin-patch/releases)
[![GitHub commit activity (branch)](https://img.shields.io/github/commit-activity/t/sagdusmir/ESP-bin-patch/main?style=flat-square&logo=github&color=blue)](https://github.com/sagdusmir/ESP-bin-patch/commits/main/)
![GitHub license](https://img.shields.io/github/license/sagdusmir/ESP-bin-patch?style=flat-square&logo=gnu&color=green)

`espbinpatch` replaces bytes in a compiled ESPHome / ESP-IDF firmware image and repairs the XOR checksum and SHA-256 so the device will still boot.


Focus: **patch secrets in an already-built `.bin`** (Wi-Fi SSID, Home Assistant API encryption key, …) without shifting offsets or leaving a checksum that the ROM bootloader will reject.


# Table of Contents

1. [Features](#features)
2. [Encodings](#encodings)
3. [Usage](#usage)
4. [Examples](#examples)
5. [Limitations](#limitations)
6. [Disclaimer](#disclaimer)

## Features

- **In-place replace** of every occurrence of a needle in `firmware.bin` or a merged `firmware.factory.bin`
- Replacement may be the **same length or shorter**; leftover bytes are filled with a pad byte (default `0x00`) so the image size and all offsets stay unchanged
- A **longer** replacement is rejected because it would shift the firmware
- Finds ESP-IDF **bootloader + app** images (partition table, or a 4 KiB-aligned scan as a fallback)
- Rewrites each image's **XOR checksum** and appended **SHA-256**
- `--verify` only checks whether those fields already match
- `--dry-run` shows what would be patched without writing a file
- `--self-test` runs built-in integrity tests (no firmware file required)

Python 3, **stdlib only** — no pip packages.

## Encodings

Use **exactly one** old/new pair. After decoding, every pair does the same byte replace.

| Flags | What is searched for in the `.bin` |
|---|---|
| `--old` / `--new` | UTF-8 bytes of the argument **as typed** |
| `--old-hex` / `--new-hex` | raw bytes from hex (whitespace ignored) |
| `--old-b64` / `--new-b64` | raw bytes from standard Base64 |
| `--old-auto` / `--new-auto` | UTF-8 **or** decoded Base64 — whichever needle actually exists |

`--old-auto` does **not** try hex. Short hex sequences appear constantly in firmware, so an SSID or OTA password that happens to be hex digits would collide or patch the wrong span. Use `--old-hex` when you mean hex.

### Home Assistant API key

ESPHome's `api.encryption.key` is a Base64 string in YAML / Home Assistant. The compiled image stores the **decoded 32 raw bytes**, not that 44-character ASCII string.

- `--old 'YcM9…ugZA='` looks for the **text** of the key (usually a miss)
- `--old-b64 'YcM9…ugZA='` looks for the **32-byte key**
- `--old-auto 'YcM9…ugZA='` tries both and uses the unique hit

`--new-auto` is decoded with the **same** encoding that matched `--old-auto`. If both representations are present, the script exits and tells you to pass `--old` or `--old-b64` explicitly.

## Usage

```bash
python3 espbinpatch.py --self-test
python3 espbinpatch.py --help
python3 espbinpatch.py firmware.bin --verify
```

Pass `-o OUTPUT` or `--in-place` to write. `--dry-run` prints hits and repairs without writing.

When `--new` is shorter than `--old`, leftover bytes are padded. `--pad` is a hex byte (`00` default; `20` for ASCII space).

## Examples

Home Assistant API key — paste the YAML/HA value and let auto-detect pick decoded Base64:

```bash
python3 espbinpatch.py firmware.factory.bin \
    --old-auto 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=' \
    --new-auto 'A1fyywUUE1DWzu0OzhDkyc4yAnfGwyEfsVNvhytrU6k=' \
    -o patched.factory.bin
```

Same patch with the encoding forced:

```bash
python3 espbinpatch.py firmware.factory.bin \
    --old-b64 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=' \
    --new-b64 'A1fyywUUE1DWzu0OzhDkyc4yAnfGwyEfsVNvhytrU6k=' \
    -o patched.factory.bin
```

UTF-8 text (SSID, hostname, …):

```bash
python3 espbinpatch.py firmware.bin \
    --old 'test_old' --new 'test_new' -o patched.bin
```

Shorter replacement, pad leftover bytes with `0x00`:

```bash
python3 espbinpatch.py firmware.bin \
    --old 'very_long_name' --new 'short' -o patched.bin
```

Check image integrity only:

```bash
python3 espbinpatch.py firmware.factory.bin --verify
```

## Limitations

- Everything that needs a **longer** replacement cannot be patched this way — rebuild the firmware instead
- `--old-auto` only distinguishes **UTF-8 vs Base64**. Hex is `--old-hex` only
- If the UTF-8 form **and** the decoded Base64 form both exist in the file, `--old-auto` refuses to guess
- The needle is searched in the **whole file**, not only inside ESP-IDF image payloads
- Patching the wrong span, or changing an API key without updating Home Assistant, will leave a device that boots but cannot connect — or one that does not boot at all

## Disclaimer

This is third-party tooling. It is not affiliated with Espressif, ESPHome, or Home Assistant. Use at your own risk.

__This project is provided for educational and experimental purposes only.__

All code, instructions, and documentation are offered AS IS without any warranty of any kind, express or implied.
YOU FLASH AND USE PATCHED FIRMWARE AT YOUR OWN RISK.
The author(s) and any contributors are not responsible for bricked devices, data loss, security issues, voided warranties, or any other consequences — direct, indirect, incidental, or consequential — that may result from using this tool or any derivative work.

Proceed only if you accept full personal responsibility.
