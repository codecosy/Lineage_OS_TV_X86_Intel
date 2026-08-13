### This repo exists to provide a built ISO of BigShoots's repo. It's not exact. My main objective was to get HDMI audio working on HP mini PCs.
**If you're still having issues with audio, make sure that the internal speakers are disabled in the BIOS.**

- Another fire tv remote keylayout has been added.
- The audio output switch can be accessed as an app. Though, I saw that the upstream dev intends on moving it into settings.
<br>

- This build doesn't have GApps.
- I couldn't figure out USB storage "adoptable" support.
- I found that the Settings menu resource overlay was messing with Settings labels. This has also been omitted.
<br>

#### Download:
https://mega.nz/file/4JhnWJKB#pcicHVND-8UexlhtM-wad66nOyrly-qyNjjItMvyjms  
c8f6756edc0b99f90e98d1ce4e56b45bae4d13ffed48014c83203eea91066cd8  
sha256
#### Alternative Build (which should function exactly the same, but I haven't tested it. It's here just in case. It was built more recently):
https://mega.nz/file/VJACmICI#raJYCn9YhanomudAwoNo8a_GAbol5hWNJAr1C3Gd3zY  
e1cc278b5744112bc8089bb0b27bb14e189bcdbeb461d953d64966af8959b4f8  
sha256 
<br>

#### **Known issues that also exist upstream when using an HP mini PC:**
- ~25% chance of the screen flickering black for a second instead of android going to sleep when `sleep` is clicked.
- Restarting android results in a connected Fire TV remote disconnecting until the remote is put into pairing mode. I believe this is a BLE issue. I'm not sure. I gave up on it.  
<br>

#### This repo will be obsolete when the upstream project implements an HDMI audio fix.

### Build instructions can be found in [BUILD.MD](https://github.com/codecosy/Lineage_OS_TV_X86_Intel/blob/main/BUILD.md)
### ORIGINAL README TEXT BELOW

***

# LineageOS TV x86 Intel Patches

This repository contains the source files, image patch scripts, validation scripts, and test notes used to build a patched LineageOS 21 Android TV x86_64 ISO for Intel mini PCs.

The working target was `lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-signed.iso` on a 7th generation Intel mini PC connected over DisplayPort-to-HDMI.

## Improvements Over Stock

- Adds a privileged Android TV audio output switcher under Display & Sound > Sound.
- Persists the selected media output device and reapplies it after boot.
- Filters unsafe/bogus policy routes such as telephony and bus outputs.
- Exposes the primary HDMI/DisplayPort PCM path to Android audio policy so media apps can route to HDMI audio.
- Makes public USB storage app-visible on Android TV x86 by enabling adoptable-style vold visibility.
- Adds a Fire TV remote keylayout for the observed Bluetooth HID remote (`0171:0413`) so the menu button opens Android Settings.
- Adds a boot helper that requests saved Wi-Fi reconnect and bonded Bluetooth HID reconnect after startup.
- Includes scripts to integrate a user-supplied MindTheGapps Android TV x86_64 package into the ISO.
- Includes ADB probes for audio routing, HDMI PCM, USB storage, CEC, HDR, and app-level playback debugging.

## Important Caveats

- The GApps ZIP is not included in this repository. Place your own compatible MindTheGapps ATV x86_64 Android 14 ZIP in the workspace before running the integration script.
- The tested Intel setup did not expose Android HDMI-CEC support.
- The tested Intel setup did not advertise HDR output support.
- Fire TV remote microphone input was not exposed as a standard Android audio input during testing; the voice button can launch search, but speech capture depends on a usable microphone device.
- Wi-Fi and Bluetooth reconnect helpers are best-effort image-side fixes and should be validated on the exact mini PC after flashing.
- Material Files should use normal file access, not root-only mode, because this build has ADB root but not a normal `su` provider.

## Final Tested ISO

The local build produced:

`lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-audio-output-gapps.iso`

SHA-256:

`9a568035817b9cdad3067e038ae69197152b9d4c617ab13b6526f197ec6bb62b`

Size:

`2812338176` bytes

GitHub release assets must be under 2 GiB each, so the ISO should be uploaded as multipart archives.

## Repository Layout

- `audio-output-switch/app`: privileged audio switcher app source.
- `audio-output-switch/overlay`: static TV Settings overlay that places the switcher in Display & Sound > Sound.
- `audio-output-switch/keylayout`: Fire TV remote keylayout used by the image patch.
- `audio-output-switch/diagnostics`: ADB test and probe scripts used during bring-up.
- `audio-output-switch/*.xml`: privapp and hidden API allowlist files for the switcher.
- `scripts`: image patch, packaging, GApps integration, and validation scripts.
- `release`: checksums and release asset metadata.

## Build Flow

The scripts assume the workspace layout used during development:

- `work/iso-root`
- `work/system/system.img`
- `audio-output-switch`
- `out`

High-level flow:

```bash
bash scripts/install_audio_switcher_to_system_img.sh
bash scripts/install_primary_hdmi_policy_to_system_img.sh
bash scripts/install_usb_storage_visibility_to_system_img.sh
bash scripts/install_fire_remote_keylayout_to_system_img.sh
bash scripts/integrate_mindthegapps.sh /path/to/MindTheGapps-14.0.0-x86_64-ATV-full.zip
bash scripts/package_gapps_iso.sh
bash scripts/validate_gapps_iso.sh
```
