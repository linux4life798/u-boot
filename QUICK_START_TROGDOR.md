# Quick Start

On Debian, do the following:

```bash
sudo apt install crossbuild-essential-arm64 libgnutls28-dev

export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64

make clean -j
make chromebook_trogdor_defconfig
# You must change .config to have:
# CONFIG_DEFAULT_DEVICE_TREE="qcom/sc7180-trogdor-wormdingler-rev1-boe"
time make -j$(nproc)
```


## Resources

* MOST IMPORTANT - https://wiki.postmarketos.org/wiki/Lenovo_IdeaPad_Duet_3_(google-wormdingler)

* [Chromium OS Support in U-Boot](https://docs.u-boot.org/en/stable/chromium/overview.html)
* https://docs.u-boot.org/en/stable/chromium/chainload.html
* IMPORTANT - https://chromium.googlesource.com/chromiumos/docs/+/51293c61c9cd09e4a55b698a22eae28f4ea2d703/developer_mode.md#alt-firmware
    - https://chatgpt.com/g/g-p-691168df77c48191b6807f3f2164c218-arm-linux/c/69123258-8cd4-8325-87db-ce008b764f80

* https://docs.mrchromebox.tech/docs/boot-modes/legacy.html
* https://github.com/MrChromebox/scripts/blob/398b65fd52198f42e1ea040eb9ee98f4ae33eafc/firmware.sh#L176

  ```bash
  # Backup
  flashrom -p internal:boardmismatch=force --fmap -i RW_LEGACY:rw_legacy_backup.bin -r
  flashrom --fmap -i RW_LEGACY:rw_legacy_backup.bin -r

  ${flashromcmd} -w $FMAP -i RW_LEGACY:${rwlegacy_file} ${noverify} -o /tmp/flashrom.log >/dev/null 2>&1
  #crossystem dev_boot_legacy=1 > /dev/null 2>&1
  crossystem dev_boot_altfw=1 > /dev/null 2>&1
  ```

* https://source.chromium.org/
* Qualcomm RMTFS Service
    - https://chromium.googlesource.com/chromiumos/overlays/board-overlays/+/refs/heads/main/baseboard-trogdor/chromeos-base/chromeos-bsp-baseboard-trogdor/files/verify_fsg.sh
        - /dev/mmcblk1boot0
        - `sudo blockdev --setro /dev/mmcblk1boot0`
    - https://source.chromium.org/chromiumos/chromiumos/codesearch/+/main:src/third_party/chromiumos-overlay/net-misc/rmtfs/files/rmtfs.conf
    - rmtfs options:
        - -o <path>: points storage logic to a directory containing EFS images or raw partitions when combined with -P (rmtfs.c:521-527).
        - -P: switches storage backend to scan for raw EFS partitions by-name instead of image files (rmtfs.c:529-532).
        - -r: opens storage read-only and skips writes to the backing store (rmtfs.c:534-536).
        - -s: enables MSS remoteproc synchronization by calling rproc_init() and later controlling it (rmtfs.c:538-548).
        - -v: enables verbose debugging through dbgprintf() (rmtfs.c:550-552).
    - `sudo /usr/bin/rmtfs -vso /var/lib/rmtfs/boot`

* https://wiki.postmarketos.org/wiki/Qualcomm_Snapdragon_7c_(SC7180)



# Device

```bash
flashrom -r /tmp/bios.bin

# Backup stock/unmodified binary
cp /tmp/bios.bin /tmp/chromebook-bios-wormdingler-nov-10-2025-stock.bin
md5sum /tmp/chromebook-bios-wormdingler-nov-10-2025-stock.bin >/tmp/checksum.md5

cbfstool /tmp/bios.bin print -r RW_LEGACY
cbfstool /tmp/bios.bin remove -r RW_LEGACY -n altfw/u-boot-uefi
cbfstool /tmp/bios.bin add-payload -r RW_LEGACY -c lzma -n altfw/u-boot-uefi -f /tmp/u-boot.elf
cbfstool /tmp/bios.bin print -r RW_LEGACY

# Add this bootloader to the altfw list
# This file exists, but was 0 bytes on wormdingler.
cbfstool /tmp/bios.bin extract -r RW_LEGACY -n altfw/list -f /tmp/altfw.txt
# The 0 index simply sets the default for "crossystem dev_default_boot=altfw"
# and will not show up as a selectable option.
echo "0;altfw/u-boot-uefi;U-Boot UEFI;From Stephen u-boot mod. Installed Nov 10, 2025" >/tmp/altfw.txt
echo "1;altfw/u-boot-uefi;U-Boot UEFI;From Stephen u-boot mod. Installed Nov 10, 2025" >>/tmp/altfw.txt
cbfstool /tmp/bios.bin remove -r RW_LEGACY -n altfw/list
cbfstool /tmp/bios.bin add -r RW_LEGACY -n altfw/list -f /tmp/altfw.txt -t raw

# Checkout the cros_allow_auto_update. It is just a raw file with the 64bit word for 1
# "01 00 00 00 00 00 00 00"
cbfstool /tmp/bios.bin extract -r RW_LEGACY -n cros_allow_auto_update -f /tmp/cros_allow_auto_update
hd /tmp/cros_allow_auto_update
cbfstool /tmp/bios.bin remove -r RW_LEGACY -n cros_allow_auto_update

#
## DO NOT RUN "cbfstool /tmp/bios.bin compact -r RW_LEGACY"!
##
## It causes coreboot to not even launch the alt firmware menu. Just hangs.
## Had to restore stock firmware.
#

cbfstool /tmp/bios.bin print -r RW_LEGACY

flashrom -w /tmp/bios.bin -i RW_LEGACY
#crossystem dev_boot_legacy
#crossystem dev_boot_legacy=1
crossystem dev_boot_altfw
crossystem dev_boot_altfw=1

crossystem dev_default_boot=altfw
```


Attempt 2

```bash
cbfst() {
    cbfstool bios.bin "$@"
}

flashrom -r bios.bin

cbfst print -r RW_LEGACY
cbfst remove -r RW_LEGACY -n altfw/u-boot
# You need to use the "u-boot.elf" output file, not the "u-boot" (elf) file
cbfst add-payload -r RW_LEGACY -c lzma -n altfw/u-boot-uefi -f ./u-boot.elf
cbfst print -r RW_LEGACY

# Add this bootloader to the altfw list
cbfst extract -r RW_LEGACY -n altfw/list -f ./altfw.txt
# echo '0;altfw/u-boot;U-Boot;U-Boot bootloader' >./altfw.txt
echo '1;altfw/u-boot-uefi;U-Boot UEFI;From Stephen u-boot mod. Installed Nov 10, 2025' > ./altfw.txt
# This file exists, but was 0 bytes on wormdingler.
cbfst remove -r RW_LEGACY -n altfw/list
cbfst add -r RW_LEGACY -n altfw/list -f ./altfw.txt -t raw
cbfst print -r RW_LEGACY

flashrom -w ./bios.bin -i RW_LEGACY
```


cbfstool bios.bin layout



# U-Boot Console

```bash
usb start
usb storage

fatls usb 0:2 /EFI/BOOT
            ./
            ../
   971654   bootaa64.efi
   971654   bootx64.efi
      141   grub.cfg
  4222448   grubaa64.efi
   886296   mmaa64.efi


ls usb 0:2 EFI/BOOT

# This worked:
bootefi bootmgr

# In GRUB:

ls (hd1,gpt2)/craig/chromeos-fdt.dtb
# Use the `normal` command to go back to menu entries.
        devicetree (hd1,gpt2)/craig/chromeos-fdt.dtb
# Edit
console=ttyMSM0,115200n8

Ctrl-x
```

---

## ChromeOS Setup

```bash
wormdingler-rev3 ~ # cat /proc/device-tree/compatible; echo
google,wormdingler-sku1024qcom,sc7180

wormdingler-rev3 ~ # cat /proc/device-tree/model; echo
Google Wormdingler rev1+ BOE panel board
```

Using https://chromium.googlesource.com/chromiumos/third_party/kernel/+/refs/heads/chromeos-6.12/arch/arm64/boot/dts/qcom/sc7180-trogdor-wormdingler-rev1-boe.dts

## Errors on boot:

```
[  OK  ] Mounted sys-kernel-config.mount - Kernel Configuration File System.
aux_bridge.aux_bridge aux_bridge.aux_bridge.0: error -ENODEV: failed to acquire drm_bridge
[  OK  ] Stopped systemd-vconsole-setup.service - Virtual Console Setup.
         Stopping systemd-vconsole-setup.service - Virtual Console Setup...
         Starting systemd-vconsole-setup.service - Virtual Console Setup...
[  OK  ] Finished systemd-vconsole-setup.service - Virtual Console Setup.
[  OK  ] Reached target tpm2.target - Trusted Platform Module.
pwm-backlight backlight: error -EINVAL: unable to request PWM
pwm-backlight backlight: probe with driver pwm-backlight failed with error -22
ti_sn65dsi86 2-002d: error -ETIMEDOUT: failed to read device id
ti_sn65dsi86 2-002d: probe with driver ti_sn65dsi86 failed with error -110
[  OK  ] Finished dracut-initqueue.service - dracut initqueue hook.
[  OK  ] Reached target remote-fs-pre.target - Preparation for Remote File Systems.


Sep 17 00:00:01 fedora kernel: qcom_scm firmware:scm: qseecom: scm call failed with error -5
Sep 17 00:00:07 fedora kernel: aux_bridge.aux_bridge aux_bridge.aux_bridge.0: error -ENODEV: failed to acquire drm_bridge
Sep 17 00:00:08 fedora kernel: pwm-backlight backlight: error -EINVAL: unable to request PWM
Sep 17 00:00:08 fedora kernel: pwm-backlight backlight: probe with driver pwm-backlight failed with error -22
Nov 10 22:06:34 fedora kernel: ti_sn65dsi86 2-002d: error -ETIMEDOUT: failed to read device id
Nov 10 22:06:34 fedora kernel: ti_sn65dsi86 2-002d: probe with driver ti_sn65dsi86 failed with error -110
Nov 10 22:07:17 fedora kernel: sx9310 5-0028: error -ETIMEDOUT: error reading WHOAMI
Nov 10 22:07:17 fedora kernel: sx9310 5-0028: probe with driver sx9310 failed with error -110
Nov 10 22:07:44 localhost-live gdm[1616]: Gdm: GdmSession: no session desktop files installed, aborting...
Nov 10 22:07:45 localhost-live systemd-coredump[2040]: [🡕] Process 1616 (gdm) of user 0 dumped core.

                                                       Module /usr/bin/gdm from rpm gdm-49.1-1.fc43.aarch64

```

```
[  OK  ] Mounted sys-kernel-config.mount - Kernel Configuration File System.
aux_bridge.aux_bridge aux_bridge.aux_bridge.0: error -ENODEV: failed to acquire drm_bridge
mmc1: Reset 0x1 never completed.
mmc1: sdhci: ============ SDHCI REGISTER DUMP ===========
mmc1: sdhci: Sys addr:  0x00000000 | Version:  0x00007202
mmc1: sdhci: Blk size:  0x00000000 | Blk cnt:  0x00000000
mmc1: sdhci: Argument:  0x00000000 | Trn mode: 0x00000000
mmc1: sdhci: Present:   0x01f800f0 | Host ctl: 0x00000000
mmc1: sdhci: Power:     0x00000000 | Blk gap:  0x00000000
mmc1: sdhci: Wake-up:   0x00000000 | Clock:    0x00000003
mmc1: sdhci: Timeout:   0x00000000 | Int stat: 0x00000000
mmc1: sdhci: Int enab:  0x00000000 | Sig enab: 0x00000000
mmc1: sdhci: ACmd stat: 0x00000000 | Slot int: 0x00000000
mmc1: sdhci: Caps:      0x322dc8b2 | Caps_1:   0x0000808f
mmc1: sdhci: Cmd:       0x00000000 | Max curr: 0x00000000
mmc1: sdhci: Resp[0]:   0x00000000 | Resp[1]:  0x00000000
mmc1: sdhci: Resp[2]:   0x00000000 | Resp[3]:  0x00000000
mmc1: sdhci: Host ctl2: 0x00000000
mmc1: sdhci_msm: ----------- VENDOR REGISTER DUMP -----------
mmc1: sdhci_msm: DLL sts: 0x00000000 | DLL cfg:  0x6000642c | DLL cfg2: 0x0020a000
mmc1: sdhci_msm: DLL cfg3: 0x00000000 | DLL usr ctl:  0x00000000 | DDR cfg: 0x80040873
mmc1: sdhci_msm: Vndr func: 0x00018a9c | Vndr func2 : 0xf88218a8 Vndr func3: 0x02626040
mmc1: sdhci: ============================================
[  OK  ] Stopped systemd-vconsole-setup.service - Virtual Console Setup.
         Stopping systemd-vconsole-setup.service - Virtual Console Setup...
         Starting systemd-vconsole-setup.service - Virtual Console Setup...
[  OK  ] Finished systemd-vconsole-setup.service - Virtual Console Setup.
[  OK  ] Reached target tpm2.target - Trusted Platform Module.
msm_dpu ae01000.display-controller: [drm:adreno_request_fw [msm]] *ERROR* failed to load a630_sqe.fw
[  OK  ] Finished dracut-initqueue.service - dracut initqueue hook.
```
