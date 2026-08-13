# Building the patched ISO

These instructions will create a patched ISO file based on the [BigShoots/Lineage_OS_TV_X86_Intel](https://github.com/BigShoots/Lineage_OS_TV_X86_Intel) repo and a stock `lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-signed.iso`.
We're just extracting the existing ISO's system image, copying files into it, and repackaging it. It's just an image patch.

I don't care for GApps, couldn't figure out USB storage "adoptable" support, and found that the Settings-menu resource overlay was messing with Settings labels. These are being omitted in the following instructions.

## Prerequisites

I tested this on Linux Mint / Ubuntu. Package names will differ on other distros.

```bash
sudo apt update
sudo apt install -y \
  git xorriso genisoimage \
  erofs-utils \
  e2fsprogs \
  cpio p7zip-full \
  openjdk-17-jdk openjdk-21-jdk \
  android-sdk-platform-23 \
  google-android-build-tools-34.0.0-installer \
  google-android-build-tools-33.0.2-installer \
  android-sdk-platform-tools adb
```

I installed two build-tools versions because the app needs to be compiled against an old Android API stub (`android-23`) and the `d8` dexer that ships in newer `build-tools` pacakges on Mint crashes with an internal `NullPointerException` on certain ordinary Java constructs when fed bytecode compiled by JDK 21.
Compiling with JDK 17 avoids the crash.

## 1. Clone this repo and set up a workspace in a dedicated directory

```bash
mkdir -p ~/lineage-tv-patch
cd ~/lineage-tv-patch
git clone https://github.com/codecosy/Lineage_OS_TV_X86_Intel.git
```

## 2. Copy in the original ISO from 2026-03-31

PLEASE check to make sure that a newer ISO that already contains the HDMI audio fix doesn't already exist.

You can grab the 2026-03-31 lineage os tv x86 ISO from [here](https://sourceforge.net/projects/lineageos-tv-x86/files/lineage-21.0/x86_64_tv/lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-signed.iso/download).\
If you are not using the correct ISO file, this guide will not work.
Your file should have a SHA256 hash of `29c44bb7bb0cb6531a11e3778377c985c4c96b881b2666fba3901e4c21d67bc2`

**Below, replace `/path/to/` with the real path to your lineage os tv x86 iso**

```bash
cp /path/to/lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-signed.iso ~/lineage-tv-patch/
cd ~/lineage-tv-patch
sha256sum lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-signed.iso
```

## 3. Extract the ISO and pull out `system.img`

The layout is nested as such:\
ISO -> `system.efs` (an EROFS filesystem image) -> `system.img` (an ext4 filesystem, which will be patched)

```bash
cd ~/lineage-tv-patch

mkdir -p work/iso-root work/iso-mnt
sudo mount -o loop,ro lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-signed.iso work/iso-mnt
cp -a work/iso-mnt/. work/iso-root/
sudo umount work/iso-mnt

mkdir -p work/system/efs-mnt
sudo mount -o loop,ro -t erofs work/iso-root/system.efs work/system/efs-mnt
cp --sparse=always work/system/efs-mnt/system.img work/system/system.img
sudo umount work/system/efs-mnt

# Keep a clean, unpatched copy. This is useful in the event you ever want to start over
cp --sparse=always work/system/system.img work/system/system.img.orig-backup
```

## 4. Identify your Fire TV remote's Bluetooth product ID (OPTIONAL)

*(This is optional in the event you have a Fire TV remote and want to change its key mappings using a keylayout file. From my experience, my remote worked just fine without doing this. I just wanted to disable power, volume, and mute in android so they would only affect my TV.)*

If you have a Fire TV remote, its Bluetooth product ID might be different. The original repo's keylayout file targets `Product_0413` and this fork adds support for `Product_0421`.\
Verify your own remote's ID by pairing your remote to an install of android.\
You'll then need to connect over adb. In order to do this on the 2026-03-31 version of Lineage OS TV x86, go to settings, get to developer options (you'll need to enable this by going to "About" and then by clicking "Build Number" 7 times.), enable adb, then enable adb over network.\
You'll see an ip address pop up.

Use that IP address to connect over adb and check the remote's information:

```bash
adb connect <box-ip>:5555
adb root
adb connect <box-ip>:5555
adb shell cat /proc/bus/input/devices | grep -B2 -A15 -i "0171"
```

Be on the lookout for a line like `I: Bus=0005 Vendor=0171 Product=XXXX` next to `N: Name="Amazon Remote Keyboard"`.  

If your `Product` differs from `0421` or `0413`, rename one of the keylayout files to match your device (`Vendor_0171_Product_<yours>.kl`).  

## 5. Build the audio output switcher app

Copy the app project into the workspace:

```bash
cd ~/lineage-tv-patch
cp -r Lineage_OS_TV_X86_Intel/audio-output-switch work/audio-output-switch
chmod +x work/audio-output-switch/build.sh
```

Pull the two framework APKs the build needs out of the stock system image:

```bash
mkdir -p work/apk/original work/system/inspect-mnt
sudo mount -o loop,ro work/system/system.img work/system/inspect-mnt
sudo cp work/system/inspect-mnt/system/framework/framework-res.apk work/apk/original/
sudo cp work/system/inspect-mnt/system/system_ext/priv-app/TvSettingsTwoPanel/TvSettingsTwoPanel.apk work/apk/original/
sudo umount work/system/inspect-mnt
sudo chown "$USER:$USER" work/apk/original/*.apk
```

Build, using JDK 17 and d8:

```bash
cd ~/lineage-tv-patch/work/audio-output-switch
PATH="/usr/lib/jvm/java-17-openjdk-amd64/bin:$PATH" \
AAPT2=/usr/bin/aapt2 \
D8=/usr/lib/android-sdk/build-tools/33.0.2/d8 \
./build.sh
cd ~/lineage-tv-patch
```

This should create `work/audio-output-switch/build/dist/AudioOutputSwitch.apk` and `AudioOutputSettingsOverlay.apk`.\
Only the first one will be installed due to the issue with the overlay previously mentioned.  

## 6. Add the keylayout and a default-permissions file that fixed some Bluetooth reconnect logic

If you skipped step 4, leave the filename below as `Vendor_0171_Product_0421.kl`. If you identified a different product ID, replace `0421` with your product ID wherever this filename appears below.

```bash
cat > work/audio-output-switch/keylayout/Vendor_0171_Product_0421.kl << 'EOF'
# Amazon Fire TV remote paired as "AR Keyboard" (Bluetooth LE HID).
# Confirmed via /proc/bus/input/devices as Bus=0005 Vendor=0171 Product=0421.
#
# Keep the normal TV remote controls aligned with Generic.kl where practical,
# but make the remote menu button open Android Settings on this TV build.

key 28    DPAD_CENTER
key 96    DPAD_CENTER
key 102   HOME
key 103   DPAD_UP
key 105   DPAD_LEFT
key 106   DPAD_RIGHT
key 108   DPAD_DOWN

key 113   VOLUME_MUTE
key 114   VOLUME_DOWN
key 115   VOLUME_UP
key 116   POWER

key 127   SETTINGS
key 139   SETTINGS
key 158   BACK
key 164   MEDIA_PLAY_PAUSE
key 172   HOME
key 217   SEARCH
key 231   CALL
key 353   DPAD_CENTER
key 582   VOICE_ASSIST
key 583   ASSIST
EOF

cat > work/audio-output-switch/default-permissions-org.lineageos.tv.audiooutput.xml << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<exceptions>
    <exception package="org.lineageos.tv.audiooutput">
        <permission name="android.permission.BLUETOOTH_CONNECT" fixed="false"/>
    </exception>
</exceptions>
EOF
```

## 7. Patch `system.img` to include the Audio output switcher app

```bash
cd ~/lineage-tv-patch

SYSTEM_IMG=work/system/system.img
MNT=work/system/audio-switcher-install-mnt
sudo mkdir -p "$MNT"
sudo umount -l "$MNT" 2>/dev/null || true
sudo mount -o loop,rw "$SYSTEM_IMG" "$MNT"

SYSTEM_OUT="$MNT/system"
APP_DIR="$SYSTEM_OUT/system_ext/priv-app/AudioOutputSwitch"
sudo mkdir -p "$APP_DIR"
sudo cp work/audio-output-switch/build/dist/AudioOutputSwitch.apk "$APP_DIR/AudioOutputSwitch.apk"

sudo install -m 0644 -o 0 -g 0 \
    work/audio-output-switch/privapp-permissions-org.lineageos.tv.audiooutput.xml \
    "$SYSTEM_OUT/system_ext/etc/permissions/privapp-permissions-org.lineageos.tv.audiooutput.xml"
sudo install -m 0644 -o 0 -g 0 \
    work/audio-output-switch/hiddenapi-package-whitelist-org.lineageos.tv.audiooutput.xml \
    "$SYSTEM_OUT/system_ext/etc/sysconfig/hiddenapi-package-whitelist-org.lineageos.tv.audiooutput.xml"

sudo chown -R 0:0 "$APP_DIR"
sudo find "$APP_DIR" -type d -exec chmod 0755 {} +
sudo find "$APP_DIR" -type f -exec chmod 0644 {} +
sudo chcon -hR u:object_r:system_file:s0 "$APP_DIR" || true
sudo chcon -h u:object_r:system_file:s0 \
    "$SYSTEM_OUT/system_ext/etc/permissions/privapp-permissions-org.lineageos.tv.audiooutput.xml" \
    "$SYSTEM_OUT/system_ext/etc/sysconfig/hiddenapi-package-whitelist-org.lineageos.tv.audiooutput.xml" || true

sudo sync
sudo umount "$MNT"
sudo e2fsck -fy "$SYSTEM_IMG"
```

### HDMI audio policy

We're just using the original repo's script:

```bash
sudo bash Lineage_OS_TV_X86_Intel/scripts/install_primary_hdmi_policy_to_system_img.sh
```

### Fire TV remote keylayout

The original repo's `install_fire_remote_keylayout_to_system_img.sh` is hardcoded to the `0413` filename, so it won't pick up a differently-named file. This installs the corrected one directly.

If your filename differs from `0421`, once again, substitute your own filename.

```bash
cd ~/lineage-tv-patch

SYSTEM_IMG=work/system/system.img
MNT=work/system/keylayout-mnt
sudo mkdir -p "$MNT"
sudo umount -l "$MNT" 2>/dev/null || true
sudo mount -o loop,rw "$SYSTEM_IMG" "$MNT"

sudo install -m 0644 -o 0 -g 0 \
  work/audio-output-switch/keylayout/Vendor_0171_Product_0421.kl \
  "$MNT/system/usr/keylayout/Vendor_0171_Product_0421.kl"
sudo chcon -h u:object_r:system_file:s0 "$MNT/system/usr/keylayout/Vendor_0171_Product_0421.kl" || true

sudo sync
sudo umount "$MNT"
sudo e2fsck -fy "$SYSTEM_IMG"
```

### Default-permissions (Bluetooth reconnect fix)

```bash
cd ~/lineage-tv-patch

SYSTEM_IMG=work/system/system.img
MNT=work/system/default-perms-mnt
sudo mkdir -p "$MNT"
sudo umount -l "$MNT" 2>/dev/null || true
sudo mount -o loop,rw "$SYSTEM_IMG" "$MNT"

for DIR in "$MNT/system/etc/default-permissions" "$MNT/system/system_ext/etc/default-permissions"; do
  sudo mkdir -p "$DIR"
  sudo install -m 0644 -o 0 -g 0 \
    work/audio-output-switch/default-permissions-org.lineageos.tv.audiooutput.xml \
    "$DIR/default-permissions-org.lineageos.tv.audiooutput.xml"
  sudo chcon -h u:object_r:system_file:s0 "$DIR/default-permissions-org.lineageos.tv.audiooutput.xml" || true
done

sudo sync
sudo umount "$MNT"
sudo e2fsck -fy "$SYSTEM_IMG"
```

## 8. Repack `system.img` into `system.efs` and rebuild the ISO

At this point, the patched `system.img` is finished, but it is not the final ISO yet. The ISO contains `system.efs`, which is an EROFS container holding `system.img`. The following commands rebuild that container and then run `package_iso.sh` to put the new `system.efs` back into a copy of the original ISO. The result is the finished patched ISO.

`package_iso.sh` resizes a copy of the new `system.efs` to the original's exact reserved size and writes it into a copy of the ISO at a fixed byte offset. If the new image is smaller, the file is extended with sparse/zero-filled space. If it is larger, `truncate` discards the excess data.\
The size check below stops the build before `package_iso.sh` can truncate an oversized filesystem.

```bash
cd ~/lineage-tv-patch

rm -rf work/system/erofs-build-root work/system/system.efs.new
mkdir -p work/system/erofs-build-root
cp --sparse=always work/system/system.img work/system/erofs-build-root/system.img

mkfs.erofs -zlz4hc,12 -C65536 \
  -U f94efea1-cebe-4f16-b2ee-340f2ca1f8e4 \
  work/system/system.efs.new \
  work/system/erofs-build-root

fsck.erofs -p work/system/system.efs.new

orig_size=$(stat -c%s work/iso-root/system.efs)
new_size=$(stat -c%s work/system/system.efs.new)
echo "original: $orig_size   new: $new_size"
if [ "$new_size" -gt "$orig_size" ]; then
    echo "*** STOP: new image is larger than the original, do not proceed ***"
else
    echo "OK: fits"
    mkdir -p out
    bash Lineage_OS_TV_X86_Intel/scripts/package_iso.sh
fi
```

The finished ISO is written to:

```
~/lineage-tv-patch/out/lineage-21.0-20260331-UNOFFICIAL-x86_64_tv-audio-output.iso
```

The ISO's hash will be different every build.

## 9. Validate

```bash
sudo bash Lineage_OS_TV_X86_Intel/scripts/validate_iso.sh
```

This will mount the finished ISO read-only and confirm that the audio output switcher app and its permission files are present. It also verifies the APK's signature and prints the boot layout to ensure the partition tables weren't disturbed.

# Good to Know

- **The HDMI audio policy patch is just a workaround, not a root-layer fix.**
  - The primary audio HAL doesn't honor this named route. Rather, it falls back to its own internal ALSA device selection.
    - Audio does reach HDMI, but through that fallback and not through the route this patch defines.
