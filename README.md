# Pico FIDO2 - Waveshare RP2350-Zero Edition

An enterprise-grade, open-source hardware security key transforming the **Raspberry Pi RP2350** (specifically the ultra-compact **Waveshare RP2350-Zero**) into an integrated **FIDO2 / WebAuthn Passkey Authenticator**, **OATH TOTP/HOTP Token Generator**, and **OpenPGP Smartcard**.

This repository is a hardened, feature-enhanced fork of [librekeys/pico-fido2](https://github.com/librekeys/pico-fido2), fully ported and optimized for the **Waveshare RP2350-Zero** with native WS2812 RGB LED signaling, user presence sliding-window authorization, and seamless companion client integration.

---

## Target Hardware Specifications

| Component | Specification |
|---|---|
| **MCU** | Raspberry Pi RP2350A (Dual-core ARM Cortex-M33 @ 150 MHz, ARMv8-M with Security Extensions) |
| **Board** | Waveshare RP2350-Zero (`PICO_BOARD=waveshare_rp2350_zero`) |
| **Flash Memory** | 4 MB Quad-SPI NOR Flash (`0x400000` bytes) |
| **Physical Button** | Onboard `BOOT` button connected to `QSPI_SS` (used for physical User Presence / touch confirmation) |
| **Status LED** | Onboard WS2812 Addressable RGB LED connected to **GPIO 16** |
| **USB Architecture** | Full-Speed USB Composite Device (Interfaces: CTAPHID, Keyboard HID, CCID Smartcard, WinUSB WebCCID) |

---

## Key Improvements & What Changed

### 1. Native WS2812 RGB LED Engine & Color Calibration
* **Correct Byte Ordering (`urgb_u32`)**: Upstream firmware used GRB bit packing which swapped red and green on Waveshare boards. We recalibrated color encoding to native `(R << 16) | (G << 8) | B`.
* **Gentle Green Idle Heartbeat**: Configured a relaxed 300 ms ON / 4700 ms OFF green pulse when the dongle is mounted and idle.
* **Multi-State Visual Feedback**:
  * **Standby / Mounted**: Gentle Green Heartbeat (300ms ON / 4700ms OFF).
  * **User Presence Required**: Rapid Yellow Blink (200ms ON / 200ms OFF).
  * **TOTP Sequence A (Even 30s epoch)**: Purple Blink (200ms ON / 200ms OFF).
  * **TOTP Sequence B (Odd 30s epoch)**: Orange Blink (200ms ON / 200ms OFF).
  * **Processing**: Blue Flash.
  * **Success Confirmation**: Steady Green (1000ms).
  * **Error / Timeout**: Fast Red Flash (1500ms).

### 2. OATH / TOTP Sliding User Presence Window
* **Single-Touch Continuous Monitoring**: In upstream firmware, accounts flagged with `touch_required` would either block terminal watch dashboards or fail with `TAG_TOUCH_RESPONSE`.
* We implemented an intelligent **35-second sliding user presence window** (`oath_user_present_time_limit`) in `src/fido/oath.c`. Touching the physical `BOOT` button once authorizes the session, and live dashboard pollers (`INS_CALCULATE_ALL`) automatically refresh the window for uninterrupted live code display. Exiting watch mode immediately revokes user presence.

### 3. Custom APDU `INS_SET_LED` (`0xA6`)
* Added a dedicated OATH APDU (`CLA=0x00, INS=0xA6`) allowing companion tools to set active TOTP sequence indicators (`P1=1` for Purple / Sequence A, `P1=2` for Orange / Sequence B) and cleanly restore idle green (`P1=0`).

### 4. Compiler-Level APDU State Hardening
* In `pico-keys-sdk/src/usb/usb.c`, GCC `-Os` optimization previously discarded TOTP mode retention checks when APDU executions finished. We introduced `volatile` state checking to prevent the compiler from overwriting active sequence blinks.

### 5. Automated USB Bootloader Reboot
* Built-in support for software-triggered reboot into the RP2350 USB Mass Storage bootloader (`reset_usb_boot(0, 0)`) via the onboard `rescue` applet (`CLA=0x80, INS=0x1F, P1=0x01`).

---

## Features Overview

### FIDO2 / WebAuthn / Passkeys
* **CTAP 2.1 & CTAP 1 (U2F)**: Full support for Google, Apple, Microsoft, and GitHub passkeys.
* **Physical Touch Confirmation**: User presence enforced via the onboard `BOOT` button.
* **Hardware Cryptography**: Hardware-accelerated SHA-256 and P-256 (SECP256R1), SECP384R1, SECP521R1, SECP256K1, and Ed25519 curves.
* **Discoverable Credentials**: Resident key support with PIN protection (`clientPin`, `minPinLength`).
* **Extensions**: `hmac-secret` (Bitwarden, KeePassXC), `credProtect`, `largeBlobKey`.

### OATH (TOTP / HOTP)
* Yubico YKOATH compliant protocol.
* Up to 255 TOTP / HOTP credentials.
* Configurable period (e.g. 30s, 60s), digits (6 or 8), and hash algorithms (SHA-1, SHA-256, SHA-512).
* Physical touch requirement flag per credential.
* Compatible with the dedicated companion client [oolalabravo/rp2350-totp](https://github.com/oolalabravo/rp2350-totp) and standard `ykman`.

### OpenPGP Smartcard
* OpenPGP Card Specification v3.4.
* 3 Key Slots (Signature, Encryption, Authentication) supporting RSA up to 4096-bit and ECC curves (Ed25519, Curve25519, NIST P-256/384/521).
* Compatible with `gpg`, SSH (`gpg-agent`), and Git commit signing.

---

## Quick Start: Flashing Prebuilt Firmware

You do not need to install toolchains or compile code. A precompiled binary is included directly in this repository:

1. Download [`release/pico_fido2_waveshare_rp2350_zero.uf2`](release/pico_fido2_waveshare_rp2350_zero.uf2).
2. Press and hold the **BOOT** button on the Waveshare RP2350-Zero board while plugging it into your computer's USB port.
3. A mass storage drive named **`RP2350`** will appear in your file manager.
4. Drag and drop `pico_fido2_waveshare_rp2350_zero.uf2` into the `RP2350` drive.
5. The device will automatically flash, disconnect, and reboot.
6. The onboard LED will pulse **Green** (300ms pulse every 5 seconds) indicating the authenticator is ready!

---

## Building From Source

### Prerequisites
* **ARM GNU Toolchain**: GCC 14 or 15+ (`arm-none-eabi-gcc`).
* **CMake**: 3.20 or newer.
* **Ninja** or **Make**.
* **Pico SDK**: Version 2.1.0 or newer.

### Compilation Steps

1. **Clone with Submodules**:
   ```bash
   git clone --recursive https://github.com/oolalabravo/pico-fido2.git
   cd pico-fido2
   ```

2. **Configure with CMake**:
   ```bash
   # Set PICO_SDK_PATH if not exported globally
   cmake -B build -G Ninja \
     -DPICO_BOARD=waveshare_rp2350_zero \
     -DPICO_PLATFORM=rp2350-arm-s \
     -DCMAKE_BUILD_TYPE=Release
   ```

3. **Build the Firmware**:
   ```bash
   ninja -C build
   ```

4. The compiled output `build/pico_fido2.uf2` is generated and ready to flash.

---

## Companion TOTP Management Client

For managing 2FA/TOTP credentials on this security key, use the dedicated companion client:

👉 **[oolalabravo/rp2350-totp](https://github.com/oolalabravo/rp2350-totp)**

* Direct WinUSB transport (driverless, immune to Windows Code 31 smartcard bug).
* Live continuous watch mode with single-touch upfront authorization.
* Synchronized hardware LED blinking (Purple / Orange) indicating code validity.
* QR code / `otpauth://` import and full credential management.

---

## License

This project is licensed under the **GNU Affero General Public License v3 (AGPLv3)**. See the [LICENSE](LICENSE) file for complete details.
