# ESP Bin Patch
[![GitHub release](https://img.shields.io/github/v/release/sagdusmir/ESP-bin-patch?style=flat-square&logo=github&color=blue)](https://github.com/sagdusmir/ESP-bin-patch/releases/latest)
[![GitHub Release Date](https://img.shields.io/github/release-date/sagdusmir/ESP-bin-patch?style=flat-square&logo=github&color=blue)](https://github.com/sagdusmir/ESP-bin-patch/releases)
[![GitHub commit activity (branch)](https://img.shields.io/github/commit-activity/t/sagdusmir/ESP-bin-patch/main?style=flat-square&logo=github&color=blue)](https://github.com/sagdusmir/ESP-bin-patch/commits/main/)
![GitHub license](https://img.shields.io/github/license/sagdusmir/ESP-bin-patch?style=flat-square&logo=gnu&color=green&ts=5)

`espbinpatch` replaces bytes in a compiled ESPHome / ESP-IDF firmware image and repairs the XOR checksum and SHA-256 so the device will still boot.


Focus: **patch secrets in an already-built `.bin`** (Wi-Fi SSID, sistant API encryption key, …) without shifting offsets or leaving a checksum that the ROM bootloader will reject. However, there are some limitations.


# Table of Contents

1. [Features](#features)
2. [Encodings](#encodings)
3. [Usage](#usage)
4. [Web demo](#web-demo)
5. [Examples](#examples)
6. [Limitations](#limitations)
7. [Disclaimer](#disclaimer)

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

## Usage

```bash
python3 espbinpatch.py --self-test
python3 espbinpatch.py --help
python3 espbinpatch.py firmware.bin --verify
```

Pass `-o OUTPUT` or `--in-place` to write. `--dry-run` prints hits and repairs without writing.

When `--new` is shorter than `--old`, leftover bytes are padded. `--pad` is a hex byte (`00` default; `20` for ASCII space).

## Web demo

Check out [`https://sagdusmir.github.io/ESP-bin-patch/`](https://sagdusmir.github.io/ESP-bin-patch/). This is a static page (with a JavaScript port of the patcher) that also allows **installing the patched image over USB**  using ESP Web Tools.

- Runs in the browser only — the `.bin` is never uploaded to a server
- Requires Chrome or Edge browser
- Several replacements, each with mode **auto / utf-8 / hex / base64**
- Errors (missing needle, invalid hex/base64, replacement too long, …) are shown on the page
- "**Keep saved settings**" writes only the program (app partition), so Wi-Fi and other saved values stay. "**Erase everything**" writes a merged factory image from the start of flash.

### Prefilled links

Pass query parameters so a README can open the demo already filled in. Choose the firmware file on the page (a local `.bin` or `.espbinpatch`). The user still clicks **Install** (USB needs a click).

| Param | Meaning |
|---|---|
| `chip` | `ESP32-C6`, `ESP32`, … |
| `flash` | `keep` or `erase` |
| `pad` | pad byte (`00`) |
| `offset` | app offset hex (`10000`); keep-settings only |
| `old`, `new`, `enc` | one replacement; repeat the trio for more (`enc`: `auto`, `utf-8`, `hex`, `base64`) |
| `lock` | `1` simplified recipient form (Old read-only; mode, pad, chip family, add/remove hidden) |

Example:

```
https://sagdusmir.github.io/ESP-bin-patch/?chip=ESP32-C6&flash=erase&old=PLACEHOLDER_KEY&new=YOUR_KEY&enc=auto
```

The page has "**Copy backup link**" that copies a link with all current values to your clipboard for later use. Check "**Lock placeholders**" to add `lock=1` so the link will open the tool width simplified options to mess with (Old is visible but not editable; the lock checkbox is hidden for them, chip is fixed, pad is hidden…). Copying the link from a `lock=1` page keeps `lock=1`. That link includes replacement values (Wi-Fi names, API keys, passwords) — do not publish it unless those secrets are meant to be public.

Locally:

```bash
python3 -m http.server --directory docs 8000
```

Then open `http://localhost:8000/index.html` and use **Run self-test**.

## Examples

sistant API key — Old may be the 32-byte placeholder **or** its Base64. New is the sistant / ESPHome key:

```bash
python3 espbinpatch.py firmware.factory.bin \
    --old-auto 'ESPBINPATCH_API_ENCRYPTION_KEY__' \
    --new-auto 'A1fyywUUE1DWzu0OzhDkyc4yAnfGwyEfsVNvhytrU6k=' \
    -o patched.factory.bin
```

```bash
python3 espbinpatch.py firmware.factory.bin \
    --old-auto 'RVNQQklOUEFUQ0hfQVBJX0VOQ1JZUFRJT05fS0VZX18=' \
    --new-auto 'A1fyywUUE1DWzu0OzhDkyc4yAnfGwyEfsVNvhytrU6k=' \
    -o patched.factory.bin
```

Same patch with the encoding forced (Old must be Base64 of the 32 bytes in the image):

```bash
python3 espbinpatch.py firmware.factory.bin \
    --old-b64 'RVNQQklOUEFUQ0hfQVBJX0VOQ1JZUFRJT05fS0VZX18=' \
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
- `--new-auto` uses Old's encoding when the replacement fits. If that New is too long, auto tries the other encoding when it is valid and fits
- If the UTF-8 form **and** the decoded Base64 form both exist in the file, `--old-auto` refuses to guess
- The needle is searched in the **whole file**, not only inside ESP-IDF image payloads
- Patching the wrong span, or changing an API key without updating sistant, will leave a device that boots but cannot connect — or one that does not boot at all

## Disclaimer

This is third-party tooling. It is not affiliated with Espressif, ESPHome, or sistant. Use at your own risk.

__This project is provided for educational and experimental purposes only.__

All code, instructions, and documentation are offered AS IS without any warranty of any kind, express or implied.
YOU FLASH AND USE PATCHED FIRMWARE AT YOUR OWN RISK.
The author(s) and any contributors are not responsible for bricked devices, data loss, security issues, voided warranties, or any other consequences — direct, indirect, incidental, or consequential — that may result from using this tool or any derivative work.

Proceed only if you accept full personal responsibility.
