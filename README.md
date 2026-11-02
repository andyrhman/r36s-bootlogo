# Custom Boot Logo for R36S Clone – GA36-MB V1.1 (20251025) (Stock "ArkOS 2.0" & goodluckOS)

<img src="https://i.imgur.com/tV5YQIj.jpeg" alt="Boot Image" width="40%" />

Device: R36S clone, board GA36 V1.1 (Allwinner, kernel `sun8i`).
Original stock card: 16 GB (shows up as 14.8 GB).

---

## 1. Quick summary

| OS | Stage | File | Location | How to change it |
|----|-------|------|----------|------------------|
| Stock | 1 (loader) | `bootlogo.bmp` | `Volumn` partition (drive **G:**) | Directly from Windows |
| Stock | 2 (after Linux starts) | `splash-1080.raw` | **inside `SYSTEM`** (squashfs), path `/usr/config/splash/` | Rebuild `SYSTEM` (VM + WSL) |
| goodluckOS | 1 | `bootlogo.bmp` | Hidden FAT volume inside the card (offset `37748736` = 36 MiB), not in the partition table | Through a VM, mount with `offset=` |

Stock has **two different logos**. If you only replace `G:\bootlogo.bmp`, the old
"R36S Open Source Console" logo still shows up after your own logo. That is why
the stock logo kept "coming back" at first.

> `/storage/.config/splash/splash-1080.raw` (on the ext4 partition) is **not** the
> boot logo. Replacing it was tried and had no effect. What is displayed at boot
> is the copy inside `SYSTEM`.

---

## 2. Image file formats

| For | Format | File size |
|-----|--------|-----------|
| `bootlogo.bmp` | 24-bit BMP, 640x480 (Windows 3.x format) | exactly 921,654 bytes |
| `splash-1080.raw` | Raw RGB (rgb24), 640x480, no header | exactly 921,600 bytes |

Convert with Python + Pillow (`pip install pillow` if you don't have it):

```powershell
# PNG/JPG -> bootlogo.bmp (24-bit)
python -c "from PIL import Image; Image.open('logo.png').convert('RGB').resize((640,480)).save('bootlogo.bmp')"

# bootlogo.bmp -> splash-1080.raw (RGB byte order, not BGR)
python -c "from PIL import Image; open('splash-1080.raw','wb').write(Image.open('bootlogo.bmp').convert('RGB').resize((640,480)).tobytes())"
```

Check the sizes: the BMP must be 921,654 bytes and the RAW must be 921,600 bytes.
If they differ, something is wrong.

Designed files (Pikachu image + text, Jersey 10 pixel font):

| File | Used for |
|------|----------|
| `bootlogo_stock_r36s.bmp` | Stock stage 1 (rename to `bootlogo.bmp`) |
| `splash-1080_stock_r36s.raw` | Stock stage 2 (rename to `splash-1080.raw`) |
| `bootlogo_goodluckos.bmp` | goodluckOS (rename to `my-logo.bmp`, put it on the card's `HOME` drive) |

---

## 3. Preparation: accessing the SD card through a VM

`wsl --mount` **does not work** for this SD card (error `ERROR_FILE_NOT_FOUND`).
Use GParted Live in VMware Workstation (confirmed working) or VirtualBox (not tested).

### First-time setup (once)

What you need:

- The **GParted Live** ISO (amd64) from <https://gparted.org/download.php>. The ISO
  used here is about 687 MB.
- VMware Workstation **or** VirtualBox installed. Virtualization (VT-x/AMD-V) must be
  enabled in the PC's BIOS, because the ISO is 64-bit.
- A USB card reader. A laptop's built-in reader sometimes doesn't show up as a USB
  device; if so, use a USB card reader or see "Alternative" below.

GParted Live runs entirely from RAM, so the VM **doesn't need a virtual disk** and
nothing is stored inside the VM. This setup is needed once; afterwards you just start
the VM and attach the card.

#### VMware Workstation (confirmed working)

1. **File > New Virtual Machine > Typical > Next**.
2. Choose **I will install the operating system later**.
3. Guest OS: **Linux**, version **Debian 12.x 64-bit** (or the newest 64-bit Debian in
   the list, or *Other Linux 5.x kernel 64-bit*). Any name, e.g. `GParted Live`.
4. The wizard asks for a virtual disk: use 1 GB, **Store virtual disk as a single
   file**. It isn't used (you can delete it later).
5. On the last page click **Customize Hardware** and set:
   - **Memory**: 2048 MB (1 GB minimum). **Processors**: 2.
   - **New CD/DVD**: choose **Use ISO image file**, point it to the GParted Live ISO,
     and make sure **Connect at power on** is ticked.
   - **USB Controller**: must be present. Start with USB compatibility **3.1**; if the
     card doesn't show up in the VM, change it to **2.0**.
   - The Network Adapter, Sound Card, and Printer can be removed (unused).
6. **Close > Finish**, then **Power On**.

If the VM doesn't boot from the ISO, make sure the CD/DVD is first in the boot order
(**VM > Power > Power On to Firmware**, or press **Esc** when the VMware logo appears).

#### VirtualBox (not tested on this setup)

1. **Machine > New**. Name `GParted Live`, Type **Linux**, Version **Debian (64-bit)**.
   In the **ISO Image** field pick the GParted Live ISO and tick **Skip Unattended
   Installation**.
2. Hardware: **Base Memory** 2048 MB, **Processors** 2. (EFI isn't needed.)
3. Hard Disk: **Do Not Add a Virtual Hard Disk** > **Finish** (ignore the disk warning).
4. **Settings > Storage**: make sure the optical drive holds the GParted Live ISO.
   **Settings > System > Boot Order**: Optical at the top.
5. **Settings > USB**: tick **Enable USB Controller** and choose **USB 3.0 (xHCI)**.
   If the card isn't detected inside the VM, switch to **USB 2.0 (EHCI)**.
   VirtualBox 7.0 and later ship the USB 2.0/3.0 controllers in the base package. On
   older versions (6.x and below), first install the *Oracle VM VirtualBox Extension
   Pack* matching your VirtualBox version.
6. (Optional) A USB filter so the card is forwarded to the VM automatically: plug in
   the card first, then in **Settings > USB** click the USB icon with the green plus
   (*Add Filter From Device*) and pick the card reader.
7. **Start**.

#### First boot of GParted Live (same for VMware and VirtualBox)

1. Blue menu: choose **GParted Live (Default settings)**.
2. Keymap: **Don't touch keymap**.
3. Language: press **Enter** (default).
4. Mode: `0` for GUI (stock flow), `2` for command line (goodluckOS flow) in this
   version.
5. Once the card is attached (see "Every time" below), check it with `lsblk`.

#### Alternative if the card reader can't be forwarded over USB (not tested on this setup)

Pass the card's **whole physical disk** to the VM:

- **VMware**: run VMware as Administrator, set the card **Offline** in Disk
  Management, then in the VM settings: **Add > Hard Disk > SATA > Use a physical
  disk > PhysicalDriveN > Use entire disk**.
- **VirtualBox**: PowerShell as Administrator, card **Offline** in Disk Management,
  then:

  ```powershell
  & "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" internalcommands createrawvmdk -filename "$env:USERPROFILE\sd.vmdk" -rawdisk "\\.\PHYSICALDRIVEN"
  ```

  Add `sd.vmdk` as a SATA disk under **Settings > Storage**.

**Careful:** a wrong `PHYSICALDRIVEN` number means another disk (SSD/HDD) can be
written to from the VM. Check with `Get-CimInstance Win32_DiskDrive` first and make
sure the Model/Size are right (the 16 GB card = 15,841,889,280 bytes).

### Every time

1. The VM and the GParted Live ISO are already set up (see "First-time setup" above). If
   the card doesn't show up later, change the VM's USB compatibility (VMware: 2.0 or
   3.1; VirtualBox: EHCI or xHCI).
2. Close every Explorer window that has G: or H: open.
3. Start the VM. In the GParted Live menu: pick the default option, keymap
   "Don't touch keymap", press Enter for the language, mode `0` (GUI).
   For the command-line-only flow (goodluckOS), choose mode **2** in this version (not 1).
4. Plug in the card, then attach it to the VM:
   - VMware: **VM > Removable Devices > Mass Storage Device USB Device > Connect
     (Disconnect from Host)**.
   - VirtualBox: **Devices > USB > Mass Storage Device USB Device** (not needed if you
     use a USB filter).
5. Open a Terminal in GParted Live and become root:

```bash
sudo -i
lsblk -o NAME,SIZE,TRAN,MODEL,FSTYPE,LABEL
```

**The disk name (`sda`, `sdb`, `sdc` ...) changes every time the card is
reconnected.** Always run `lsblk` first and replace `sdX` in every command below.
To release the card from the VM (so G:/H: show up in Windows again):
VMware: **VM > Removable Devices > ... > Disconnect**. VirtualBox: **Devices > USB**,
untick the device. Shutting down the VM also returns the card to Windows
automatically.

### Stock card layout (16 GB)

| Partition | Size | FS | Contents |
|-----------|------|----|----------|
| `sdX1` | 13 GB | vfat | ROMs / EASYROMS (**H:**) |
| `sdX2` | 32 MB | vfat, label `Volumn` | `bootlogo.bmp`, `font24.sft`, `font32.sft`, `magic.bin`, `bat` folder (**G:**) |
| `sdX3` | 1 KB | - | extended partition container |
| `sdX5` | 16 MB | raw | U-Boot environment. **Don't touch** |
| `sdX6` | 32 MB | raw | Android boot image (kernel + ramdisk, `sun8i`). **Don't touch** |
| `sdX7` | 512 MB | vfat, label `ARKOS` | `SYSTEM` (squashfs, ~422 MB), `low_pwr.bmp` |
| `sdX8` | 1 GB | ext4 | `/storage`: `.config/splash`, etc. |

Windows only shows G: and H:. The rest (~1.61 GB) isn't listed because it sits
inside the extended partition.

Mount everything you need (in the VM, as root):

```bash
mkdir -p /mnt/g /mnt/h /mnt/ark /mnt/root /mnt/sys
mount /dev/sdX2 /mnt/g      # Volumn  (G:)
mount /dev/sdX1 /mnt/h      # ROMs    (H:)
mount /dev/sdX7 /mnt/ark    # ARKOS   (SYSTEM)
mount /dev/sdX8 /mnt/root   # ext4    (/storage)
```

If you only want to look without any risk of writing, add `-o ro`.
If ext4 refuses to mount because of the journal: `mount -o ro,noload /dev/sdX8 /mnt/root`.

---

## 4. STOCK

### Back up first (once)

Keep the backup on `H:` (the ROM partition) so it's visible from Windows:

```bash
mkdir -p /mnt/h/backup_logo
cp /mnt/g/bootlogo.bmp /mnt/h/backup_logo/bootlogo.orig.bmp
cp /mnt/ark/low_pwr.bmp /mnt/h/backup_logo/
cp -r /mnt/root/.config/splash /mnt/h/backup_logo/
cp /mnt/ark/SYSTEM /mnt/h/backup_logo/SYSTEM.orig
sync
md5sum /mnt/ark/SYSTEM /mnt/h/backup_logo/SYSTEM.orig    # must match
```

The `H:\backup_logo` folder contains: `splash/` (including the original
`splash-1080.raw`), `low_pwr.bmp`, `SYSTEM.orig`. **Don't delete it** as long as
you plan to keep tinkering.

Hash of the original `splash-1080.raw` (stock build of 3 Dec 2025): `b90963148764b4e538858fc7165bc86f`.

### A. Stage 1: `G:\bootlogo.bmp` (easy)

From Windows (card connected to Windows, not the VM):

1. Copy `bootlogo_stock_r36s.bmp` to `G:\` and rename it to `bootlogo.bmp`
   (overwrite the old one; the original is already in the backup).
2. Eject, boot. The first logo changes.

(Alternative from the VM: `cp new_bootlogo.bmp /mnt/g/bootlogo.bmp && sync`.)

### B. Stage 2: `splash-1080.raw` inside `SYSTEM`

This logo is shown by `/usr/bin/lsb_release`, a script the vendor modified, via
this line:

```
ffplay -f rawvideo -autoexit -video_size 640x480 -pixel_format rgb24 /usr/config/splash/splash-1080.raw
```

`SYSTEM` is a read-only squashfs, so it has to be unpacked, the file replaced,
and the image rebuilt.

**B1. Prepare `splash-1080.raw`** (Windows), see section 2. Put it in
`H:\new_splash\splash-1080.raw` (card connected to Windows).

**B2. Back up `SYSTEM`** (already covered in "Back up first" above).

**B3. Rebuild in WSL (Windows)**

Copy `H:\backup_logo\SYSTEM.orig` and `H:\new_splash\splash-1080.raw` to
`C:\work`, then in the Ubuntu (WSL) terminal:

```bash
sudo apt install -y squashfs-tools
mkdir -p ~/r36s && cd ~/r36s
cp /mnt/c/work/SYSTEM.orig /mnt/c/work/splash-1080.raw .
sudo unsquashfs -d root SYSTEM.orig
sudo cp splash-1080.raw root/usr/config/splash/splash-1080.raw
sudo mksquashfs root SYSTEM.new -comp lzo -b 512K -noappend
file SYSTEM.new
cp SYSTEM.new /mnt/c/work/
```

- Work inside `~` (not `/mnt/c`) and use `sudo` so file permissions and symlinks
  stay intact.
- The options `-comp lzo -b 512K` match the original file. `file SYSTEM.new`
  should say roughly: `Squashfs filesystem, little endian, version 4.0, lzo compressed, ... blocksize: 524288 bytes`.
- The `ARKOS` partition is only 512 MB and the original `SYSTEM` is ~422 MB, so
  `SYSTEM.new` must not be much bigger than the original.

**B4. Write it to the card (VM)**

Copy `SYSTEM.new` to `H:\new_splash\`, connect the card to the VM, then:

```bash
lsblk
mount /dev/sdX7 /mnt/ark
mount /dev/sdX1 /mnt/h
cp /mnt/h/new_splash/SYSTEM.new /mnt/ark/SYSTEM
sync
md5sum /mnt/h/new_splash/SYSTEM.new /mnt/ark/SYSTEM    # MUST match
umount /mnt/ark /mnt/h
```

If `cp` fails with `No space left on device` (it happened once): `umount`
everything, run `lsblk` to see the current disk name (most likely the old mount
was still attached to a disk name that has since changed), mount again with the
correct name, and repeat `cp`. Last time, after remounting, `md5sum` matched and
it booted fine. **Don't unplug the card until the md5 matches.**

**B5. Test boot.** If the console won't boot, restore:

```bash
cp /mnt/h/backup_logo/SYSTEM.orig /mnt/ark/SYSTEM && sync
md5sum /mnt/h/backup_logo/SYSTEM.orig /mnt/ark/SYSTEM
```

### What you don't need / must not touch

- `low_pwr.bmp` on `ARKOS`: the low-battery screen, not the boot logo. Not changed.
- `sdX5` and `sdX6`: U-Boot environment and kernel. Writing to them wrongly can
  leave the console unable to boot.
- Don't resize or move partitions on the stock card. Extending `H:` with
  MiniTool Partition Wizard once made the console hang at the boot logo.
- The "R36S Please Wait....." images in `/storage/.config/splash/*.png` are used
  by `show_splash.sh` for intro/exit/game loading (not the boot logo). They can
  be replaced directly on the ext4 partition (640x480 PNG) if you want. Optional.

---

## 5. goodluckOS

On goodluckOS, `bootlogo.bmp` lives on a small FAT partition that Windows doesn't
show (see issue [#111](https://github.com/CodeZombie/goodluckOS/issues/111) in the goodluckOS repo). At byte offset `37748736` (= 36 MiB = sector 73728 x 512).
That volume isn't in the partition table, which is why Windows doesn't show it. To
reach it, mount straight from the raw disk (`/dev/sda`, not `/dev/sda2`) with the
`offset=` option.

Card used: 64 GB SanDisk running goodluckOS. In the VM it shows up as `sda`
(59.5G, TRAN `usb`). The FAT partition `sda2` labeled `HOME` is also visible in
Windows, and is used as a "bridge" for the new logo file.

### Preparation

1. Windows: prepare a 24-bit 640x480 BMP (section 2) named `my-logo.bmp` and put it
   on the card's `HOME` drive (designed file: rename `bootlogo_goodluckos.bmp` to
   `my-logo.bmp`).
2. Close every Explorer window, then connect the card to the VM (section 3). In the
   GParted Live menu choose command-line mode: in this version it is number **2**
   (not 1). The message `cannot set terminal process group` is harmless, ignore it.
   In this mode you're already logged in as root, so no `sudo` is needed.
3. Run `lsblk` to confirm the card is the 59.5G disk with TRAN `usb` and that
   there is a FAT partition labeled `HOME`. If the names aren't `sda`/`sda2`, adjust
   `CARD` and `DATA` below.

### Block 1: check first (read-only, changes nothing)

```bash
CARD=/dev/sda
DATA=/dev/sda2
mkdir -p /mnt/data /mnt/bootres
mount $DATA /mnt/data
ls /mnt/data
mount -t vfat -o ro,offset=37748736 $CARD /mnt/bootres
ls -l /mnt/bootres
od -An -tu4 -j18 -N8 /mnt/bootres/bootlogo.bmp
```

Expected results:

- `ls /mnt/data` shows `my-logo.bmp`.
- `ls -l /mnt/bootres` shows `bootlogo.bmp`.
- `od` prints `640 480`.

If all three match, continue to Block 2. If not, **do not continue**. If the mount
fails, run `fdisk -l /dev/sda` to see the card's partition layout.

### Block 2: replace the logo (only if Block 1 matched)

```bash
cp /mnt/bootres/bootlogo.bmp /mnt/data/bootlogo-stock.bmp
umount /mnt/bootres

mount -t vfat -o offset=37748736 $CARD /mnt/bootres
cp /mnt/data/my-logo.bmp /mnt/bootres/bootlogo.bmp
ls -l /mnt/bootres/bootlogo.bmp
sync
umount /mnt/bootres
umount /mnt/data
```

After `cp`, `ls -l` must show 921654 bytes. If `cp` reports `No space left`, stop.

### When done

After both `umount` commands finish without errors, shut the VM down with `poweroff`
(or the VMware menu VM > Power > Shut Down Guest). The card returns to Windows
automatically; if "You need to format the disk" appears, click **Cancel**.

1. Copy `bootlogo-stock.bmp` from the card drive to the PC as a backup, then delete
   `bootlogo-stock.bmp` and `my-logo.bmp` from the card.
2. Put the card in the console and power on. The Pikachu logo shows at boot.

The offset `37748736` is specific to the goodluckOS image in use now. If goodluckOS
is updated or reflashed and the layout changes, Block 1 will fail or its results
won't match. Don't continue to Block 2; check with `fdisk -l /dev/sda` first.

---

## 6. When the OS is updated or the card is reflashed

An update/reflash puts the default logos back. Repeat:

- **Stock:** sections 4A (stage 1) and 4B (stage 2).
- **goodluckOS:** section 5 (run Block 1 first; offset `37748736` may change in a new version).

A newer version may change the location or name of the logo files. To track them
down again (all read-only, from the VM):

```bash
mount -o ro /dev/sdX7 /mnt/ark
file /mnt/ark/SYSTEM        # note the compression (lzo?) and blocksize for mksquashfs
mount -t squashfs -o loop,ro /mnt/ark/SYSTEM /mnt/sys

# scripts that show the splash / logo
grep -rl 'splash' /mnt/sys/usr/bin /mnt/sys/usr/sbin /mnt/sys/etc /mnt/sys/usr/lib/systemd /mnt/sys/usr/lib/autostart /mnt/sys/usr/config 2>/dev/null
grep -n 'ffplay' /mnt/sys/usr/bin/lsb_release

# files of exactly 640x480 rgb24 size (raw image candidates)
find /mnt/sys -type f \( -iname '*splash*' -o -iname '*logo*' -o -size 921600c \) 2>/dev/null | head -40
ls -l /mnt/sys/usr/config/splash/
```

If the image shown differs from what's in `splash-1080.raw`, read the script
(`cat -n /mnt/sys/usr/bin/lsb_release`) to see which file is actually called. A
raw image can be inspected on Windows/Linux with:

```bash
ffmpeg -f rawvideo -pix_fmt rgb24 -s 640x480 -i splash-1080.raw check.png
```

---

## 7. Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `wsl --mount` error `WSL_E_ELEVATION_NEEDED_TO_MOUNT_DISK` | PowerShell/Terminal must be run as administrator |
| `wsl --mount` error `ERROR_FILE_NOT_FOUND` / disk in use | The SD card can't be attached through WSL. Use the VM |
| `PHYSICALDRIVEn` number turned out to be another disk | Check `Get-CimInstance Win32_DiskDrive`; the 16 GB card is 15,841,889,280 bytes |
| `mount: can't lookup blockdev` | The disk name changed after reconnecting. Run `lsblk` and use the new name |
| Card doesn't appear under Removable Devices | Set the VM's USB compatibility to 2.0/3.1, or close any Explorer window using G:/H: |
| Stage 1 logo changed, stage 2 is still the old logo | `splash-1080.raw` inside `SYSTEM` hasn't been replaced yet (4B) |
| Logo colors swapped (red/blue) | The RAW was made in BGR order. Use `convert('RGB')` + `tobytes()` |
| Black screen / no boot after replacing `SYSTEM` | Restore `SYSTEM.orig` (step B5) |
| `cp` to `ARKOS`: `No space left on device` | See the note in B4 (remount with the correct disk name, check the size of `SYSTEM.new`) |
| goodluckOS: `mount ... offset=...` fails or `bootlogo.bmp` doesn't show | The card layout differs from what's expected. Don't continue; run `fdisk -l /dev/sda` and re-check the offset |
| goodluckOS: `cannot set terminal process group` | Harmless, ignore it |
| Windows asks "You need to format the disk" after the card returns | Click **Cancel**. Don't format |

---

## 8. Ultra-short flow (stock)

1. Back up (`H:\backup_logo`).
2. Windows: copy the new `bootlogo.bmp` to `G:\`.
3. Make `splash-1080.raw` (921,600 bytes).
4. WSL: `unsquashfs` > replace `/usr/config/splash/splash-1080.raw` > `mksquashfs -comp lzo -b 512K`.
5. VM: copy `SYSTEM.new` to `ARKOS/SYSTEM`, `md5sum` must match, `umount`.
6. Boot, check both logo stages.

### goodluckOS (ultra-short)

1. Windows: put `my-logo.bmp` (24-bit 640x480 BMP) on the card's `HOME` drive.
2. VM (command-line mode **2**): run Block 1 and make sure all three results match.
3. Run Block 2 (back up the old logo to `bootlogo-stock.bmp`, copy `my-logo.bmp` into
   the hidden volume at offset `37748736`), `sync`, `umount` twice.
4. `poweroff`, click Cancel if Windows asks to format, clean up the files on the card, boot.
