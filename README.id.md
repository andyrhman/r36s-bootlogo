# Custom Boot Logo untuk R36S Clone – GA36-MB V1.1 (20251025) (Stock "ArkOS 2.0" & goodluckOS)

<img src="https://i.imgur.com/tV5YQIj.jpeg" alt="Boot Image" width="40%" />

Catatan pribadi supaya tidak perlu cari-cari command di chat lagi.
Perangkat: R36S clone, board GA36 V1.1 (Allwinner, kernel `sun8i`).
Kartu stock asli: 16 GB (terbaca 14,8 GB).

---

## 1. Ringkasan cepat

| OS | Tahap | File | Lokasi | Cara ganti |
|----|-------|------|--------|-----------|
| Stock | 1 (loader) | `bootlogo.bmp` | partisi `Volumn` (drive **G:**) | Langsung dari Windows |
| Stock | 2 (setelah Linux jalan) | `splash-1080.raw` | **di dalam `SYSTEM`** (squashfs), path `/usr/config/splash/` | Bangun ulang `SYSTEM` (VM + WSL) |
| goodluckOS | 1 | `bootlogo.bmp` | volume FAT tersembunyi di dalam kartu (offset `37748736` = 36 MiB), tidak ada di tabel partisi | Lewat VM, mount dengan `offset=` |

Stock punya **dua logo berbeda**. Kalau cuma `G:\bootlogo.bmp` yang diganti, logo
"R36S Open Source Console" yang lama tetap muncul setelah logo Pikachu. Itu
sebabnya dulu logonya "balik lagi".

> `/storage/.config/splash/splash-1080.raw` (di partisi ext4) **bukan** logo boot.
> Sudah dicoba diganti dan tidak berpengaruh. Yang dipakai saat boot adalah salinan
> di dalam `SYSTEM`.

---

## 2. Format file gambar

| Untuk | Format | Ukuran file |
|-------|--------|-------------|
| `bootlogo.bmp` | BMP 24-bit, 640x480 (Windows 3.x format) | tepat 921.654 byte |
| `splash-1080.raw` | RGB mentah (rgb24), 640x480, tanpa header | tepat 921.600 byte |

Konversi dengan Python + Pillow (`pip install pillow` kalau belum ada):

```powershell
# PNG/JPG -> bootlogo.bmp (24-bit)
python -c "from PIL import Image; Image.open('logo.png').convert('RGB').resize((640,480)).save('bootlogo.bmp')"

# bootlogo.bmp -> splash-1080.raw (urutan warna RGB, jangan BGR)
python -c "from PIL import Image; open('splash-1080.raw','wb').write(Image.open('bootlogo.bmp').convert('RGB').resize((640,480)).tobytes())"
```

Cek ukuran: BMP harus 921.654 byte, RAW harus 921.600 byte. Kalau beda, ada yang salah.

File hasil desain (gambar Pikachu + teks, font pixel Jersey 10):

| File | Dipakai untuk |
|------|---------------|
| `bootlogo_stock_r36s.bmp` | Stock tahap 1 (ganti nama jadi `bootlogo.bmp`) |
| `splash-1080_stock_r36s.raw` | Stock tahap 2 (ganti nama jadi `splash-1080.raw`) |
| `bootlogo_goodluckos.bmp` | goodluckOS (ganti nama jadi `my-logo.bmp`, taruh di drive `HOME` kartu) |

---

## 3. Persiapan: akses kartu SD lewat VM

`wsl --mount` **tidak bisa** untuk kartu SD ini (error `ERROR_FILE_NOT_FOUND`).
Pakai GParted Live di VMware Workstation (sudah terbukti jalan) atau VirtualBox
(belum diuji).

### Setup pertama kali (sekali saja)

Yang perlu disiapkan:

- ISO **GParted Live** (amd64) dari <https://gparted.org/download.php>. ISO yang dipakai
  di sini ±687 MB.
- VMware Workstation **atau** VirtualBox sudah terpasang. Virtualisasi (VT-x/AMD-V)
  harus aktif di BIOS PC, karena ISO-nya 64-bit.
- Card reader USB. Reader bawaan laptop kadang tidak muncul sebagai perangkat USB;
  kalau begitu pakai card reader USB, atau lihat "Alternatif" di bawah.

GParted Live berjalan penuh dari RAM, jadi VM **tidak butuh disk virtual** dan tidak
ada yang tersimpan di VM. Setup ini cukup sekali; selanjutnya tinggal menyalakan VM
dan menyambungkan kartu.

#### VMware Workstation (sudah terbukti jalan)

1. **File > New Virtual Machine > Typical > Next**.
2. Pilih **I will install the operating system later**.
3. Guest OS: **Linux**, versi **Debian 12.x 64-bit** (atau Debian 64-bit terbaru yang
   ada di daftar, atau *Other Linux 5.x kernel 64-bit*). Nama bebas, misalnya
   `GParted Live`.
4. Wizard akan meminta disk virtual: isi 1 GB, **Store virtual disk as a single
   file**. Disk ini tidak dipakai (boleh dihapus nanti).
5. Di langkah terakhir klik **Customize Hardware** dan atur:
   - **Memory**: 2048 MB (minimal 1 GB). **Processors**: 2.
   - **New CD/DVD**: pilih **Use ISO image file**, arahkan ke ISO GParted Live,
     pastikan **Connect at power on** tercentang.
   - **USB Controller**: harus ada. USB compatibility **3.1** dulu; kalau kartu tidak
     muncul di VM, ubah ke **2.0**.
   - Network Adapter, Sound Card, dan Printer boleh dihapus (tidak dipakai).
6. **Close > Finish**, lalu **Power On**.

Kalau VM tidak boot dari ISO, pastikan CD/DVD berada di urutan boot pertama
(**VM > Power > Power On to Firmware**, atau tekan **Esc** saat logo VMware muncul).

#### VirtualBox (belum diuji di setup ini)

1. **Machine > New**. Name `GParted Live`, Type **Linux**, Version **Debian (64-bit)**.
   Di kolom **ISO Image** pilih ISO GParted Live dan centang **Skip Unattended
   Installation**.
2. Hardware: **Base Memory** 2048 MB, **Processors** 2. (EFI tidak perlu dicentang.)
3. Hard Disk: **Do Not Add a Virtual Hard Disk** > **Finish** (abaikan peringatan disk).
4. **Settings > Storage**: pastikan drive optik berisi ISO GParted Live.
   **Settings > System > Boot Order**: Optical paling atas.
5. **Settings > USB**: centang **Enable USB Controller**, pilih **USB 3.0 (xHCI)**.
   Kalau kartu tidak terbaca di dalam VM, ganti ke **USB 2.0 (EHCI)**.
   VirtualBox 7.0 ke atas sudah membawa kontroler USB 2.0/3.0 di paket utamanya. Di
   versi lama (6.x ke bawah) install dulu *Oracle VM VirtualBox Extension Pack* yang
   versinya sama dengan VirtualBox.
6. (Opsional) USB filter supaya kartu otomatis diteruskan ke VM: colok kartu dulu, di
   **Settings > USB** klik ikon USB dengan tanda plus hijau (*Add Filter From
   Device*), lalu pilih card reader-nya.
7. **Start**.

#### Boot pertama GParted Live (sama untuk VMware dan VirtualBox)

1. Menu biru: pilih **GParted Live (Default settings)**.
2. Keymap: **Don't touch keymap**.
3. Bahasa: tekan **Enter** (default).
4. Mode: `0` untuk GUI (alur stock), `2` untuk command line (alur goodluckOS) di
   versi ini.
5. Setelah kartu tersambung (lihat "Setiap kali pakai" di bawah), cek dengan `lsblk`.

#### Alternatif kalau card reader tidak bisa diteruskan lewat USB (belum diuji di setup ini)

Teruskan **seluruh disk fisik** kartu ke VM:

- **VMware**: jalankan VMware sebagai Administrator, set kartu **Offline** di Disk
  Management, lalu di Settings VM: **Add > Hard Disk > SATA > Use a physical disk >
  PhysicalDriveN > Use entire disk**.
- **VirtualBox**: PowerShell sebagai Administrator, kartu **Offline** di Disk
  Management, lalu:

  ```powershell
  & "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" internalcommands createrawvmdk -filename "$env:USERPROFILE\sd.vmdk" -rawdisk "\\.\PHYSICALDRIVEN"
  ```

  Tambahkan `sd.vmdk` sebagai disk SATA di **Settings > Storage**.

**Hati-hati:** nomor `PHYSICALDRIVEN` yang salah berarti disk lain (SSD/HDD) bisa
ditulis dari VM. Cek dulu dengan `Get-CimInstance Win32_DiskDrive` dan pastikan
Model/Size-nya benar (kartu 16 GB = 15.841.889.280 byte).

### Setiap kali pakai

1. VM dan ISO GParted Live sudah terpasang (lihat "Setup pertama kali" di atas). Kalau
   kartu tidak muncul nanti, ubah USB compatibility VM (VMware: 2.0 atau 3.1;
   VirtualBox: EHCI atau xHCI).
2. Tutup semua jendela Explorer yang membuka G: atau H:.
3. Nyalakan VM. Menu GParted Live: pilih opsi default, keymap
   "Don't touch keymap", bahasa tekan Enter, mode `0` (GUI).
   Untuk alur command line saja (goodluckOS), pilih mode **2** di versi ini (bukan 1).
4. Colok kartu, lalu sambungkan ke VM:
   - VMware: **VM > Removable Devices > Mass Storage Device USB Device > Connect
     (Disconnect from Host)**.
   - VirtualBox: **Devices > USB > Mass Storage Device USB Device** (tidak perlu kalau
     sudah memakai USB filter).
5. Buka Terminal di GParted Live, lalu jadi root:

```bash
sudo -i
lsblk -o NAME,SIZE,TRAN,MODEL,FSTYPE,LABEL
```

**Nama disk (`sda`, `sdb`, `sdc` ...) berubah setiap kartu di-connect ulang.**
Selalu jalankan `lsblk` dulu, dan ganti `sdX` di semua command di bawah.
Kalau perlu memutuskan kartu dari VM (supaya G:/H: muncul di Windows lagi):
VMware: **VM > Removable Devices > ... > Disconnect**. VirtualBox: **Devices > USB**,
hilangkan centang pada perangkatnya. Mematikan VM juga otomatis mengembalikan kartu
ke Windows.

### Layout kartu stock (16 GB)

| Partisi | Ukuran | FS | Isi |
|---------|--------|----|-----|
| `sdX1` | 13 GB | vfat | ROM / EASYROMS (**H:**) |
| `sdX2` | 32 MB | vfat, label `Volumn` | `bootlogo.bmp`, `font24.sft`, `font32.sft`, `magic.bin`, folder `bat` (**G:**) |
| `sdX3` | 1 KB | - | wadah extended partition |
| `sdX5` | 16 MB | raw | environment U-Boot. **Jangan disentuh** |
| `sdX6` | 32 MB | raw | Android boot image (kernel + ramdisk, `sun8i`). **Jangan disentuh** |
| `sdX7` | 512 MB | vfat, label `ARKOS` | `SYSTEM` (squashfs, ±422 MB), `low_pwr.bmp` |
| `sdX8` | 1 GB | ext4 | `/storage`: `.config/splash`, dll |

Windows hanya menampilkan G: dan H:. Sisanya (±1,61 GB) tidak terdaftar karena
berada di extended partition.

Mount semua yang dibutuhkan (di VM, sebagai root):

```bash
mkdir -p /mnt/g /mnt/h /mnt/ark /mnt/root /mnt/sys
mount /dev/sdX2 /mnt/g      # Volumn  (G:)
mount /dev/sdX1 /mnt/h      # ROM     (H:)
mount /dev/sdX7 /mnt/ark    # ARKOS   (SYSTEM)
mount /dev/sdX8 /mnt/root   # ext4    (/storage)
```

Kalau mau sekadar melihat tanpa risiko menulis, tambahkan `-o ro`.
Kalau ext4 menolak karena journal: `mount -o ro,noload /dev/sdX8 /mnt/root`.

---

## 4. STOCK

### Backup dulu (sekali saja)

Simpan backup di `H:` (partisi ROM) supaya terlihat dari Windows:

```bash
mkdir -p /mnt/h/backup_logo
cp /mnt/g/bootlogo.bmp /mnt/h/backup_logo/bootlogo.orig.bmp
cp /mnt/ark/low_pwr.bmp /mnt/h/backup_logo/
cp -r /mnt/root/.config/splash /mnt/h/backup_logo/
cp /mnt/ark/SYSTEM /mnt/h/backup_logo/SYSTEM.orig
sync
md5sum /mnt/ark/SYSTEM /mnt/h/backup_logo/SYSTEM.orig    # harus sama
```

Folder `H:\backup_logo` berisi: `splash/` (termasuk `splash-1080.raw` asli),
`low_pwr.bmp`, `SYSTEM.orig`. **Jangan dihapus** selama masih berniat utak-atik.

Hash `splash-1080.raw` asli (build stock 3 Des 2025): `b90963148764b4e538858fc7165bc86f`.

### A. Tahap 1: `G:\bootlogo.bmp` (mudah)

Dari Windows (kartu terhubung ke Windows, bukan VM):

1. Salin `bootlogo_stock_r36s.bmp` ke `G:\`, ganti nama jadi `bootlogo.bmp`
   (timpa yang lama; yang asli sudah ada di backup).
2. Eject, boot. Logo pertama berubah.

(Alternatif dari VM: `cp bootlogo_baru.bmp /mnt/g/bootlogo.bmp && sync`.)

### B. Tahap 2: `splash-1080.raw` di dalam `SYSTEM`

Logo ini ditampilkan skrip `/usr/bin/lsb_release` yang sudah dimodifikasi vendor,
lewat baris:

```
ffplay -f rawvideo -autoexit -video_size 640x480 -pixel_format rgb24 /usr/config/splash/splash-1080.raw
```

`SYSTEM` itu squashfs read-only, jadi harus dibongkar, diganti filenya, lalu
dibangun ulang.

**B1. Siapkan `splash-1080.raw`** (Windows), lihat bagian 2. Taruh di
`H:\new_splash\splash-1080.raw` (kartu terhubung ke Windows).

**B2. Backup `SYSTEM`** (sudah ada di bagian "Backup dulu" di atas).

**B3. Bangun ulang di WSL (Windows)**

Salin `H:\backup_logo\SYSTEM.orig` dan `H:\new_splash\splash-1080.raw` ke
`C:\work`, lalu di terminal Ubuntu (WSL):

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

- Kerjakan di `~` (bukan di `/mnt/c`) dan pakai `sudo` supaya izin file dan
  symlink tetap utuh.
- Opsi `-comp lzo -b 512K` menyamakan dengan file aslinya. Hasil `file SYSTEM.new`
  harus kira-kira: `Squashfs filesystem, little endian, version 4.0, lzo compressed, ... blocksize: 524288 bytes`.
- Partisi `ARKOS` cuma 512 MB dan `SYSTEM` asli ±422 MB, jadi `SYSTEM.new` tidak
  boleh jauh lebih besar dari aslinya.

**B4. Pasang ke kartu (VM)**

Salin `SYSTEM.new` ke `H:\new_splash\`, connect kartu ke VM, lalu:

```bash
lsblk
mount /dev/sdX7 /mnt/ark
mount /dev/sdX1 /mnt/h
cp /mnt/h/new_splash/SYSTEM.new /mnt/ark/SYSTEM
sync
md5sum /mnt/h/new_splash/SYSTEM.new /mnt/ark/SYSTEM    # WAJIB sama
umount /mnt/ark /mnt/h
```

Kalau `cp` gagal dengan `No space left on device` (pernah terjadi): `umount` semua,
jalankan `lsblk` untuk melihat nama disk yang sekarang (kemungkinan mount lama
masih menempel ke nama disk yang sudah berubah), mount ulang dengan nama yang
benar, ulangi `cp`. Pada kejadian dulu, setelah mount ulang `md5sum` cocok dan
bootnya berhasil. **Jangan cabut kartu sebelum md5 cocok.**

**B5. Tes boot.** Kalau konsol tidak mau boot, kembalikan:

```bash
cp /mnt/h/backup_logo/SYSTEM.orig /mnt/ark/SYSTEM && sync
md5sum /mnt/h/backup_logo/SYSTEM.orig /mnt/ark/SYSTEM
```

### Yang tidak perlu / jangan disentuh

- `low_pwr.bmp` di `ARKOS`: layar baterai lemah, bukan logo boot. Belum diubah.
- `sdX5` dan `sdX6`: U-Boot env dan kernel. Salah tulis di sini bisa membuat
  konsol tidak mau boot.
- Jangan resize/geser partisi kartu stock. Dulu memperbesar `H:` dengan
  MiniTool Partition Wizard membuat konsol hang di logo boot.
- Gambar "R36S Please Wait....." di `/storage/.config/splash/*.png` dipakai
  `show_splash.sh` untuk intro/exit/loading game (bukan logo boot). Bisa diganti
  langsung di ext4 (PNG 640x480) kalau mau, opsional.

---

## 5. goodluckOS

Di goodluckOS, `bootlogo.bmp` **tidak** ada di partisi biasa (lihat issue [#111](https://github.com/CodeZombie/goodluckOS/issues/111) di repo goodluckOS). File itu ada di volume
FAT tersembunyi di dalam kartu, pada offset byte `37748736` (= 36 MiB = sektor 73728
x 512). Volume itu tidak terdaftar di tabel partisi, makanya tidak muncul di Windows.
Cara mengaksesnya: mount langsung dari disk mentah (`/dev/sda`, bukan `/dev/sda2`)
dengan opsi `offset=`.

Kartu yang dipakai: SanDisk 64 GB dengan goodluckOS. Di VM terbaca sebagai `sda`
(59.5G, TRAN `usb`). Partisi FAT `sda2` berlabel `HOME` juga terlihat di Windows,
dan dipakai sebagai "jembatan" untuk file logo baru.

### Persiapan

1. Windows: siapkan BMP 24-bit 640x480 (bagian 2) dengan nama `my-logo.bmp`, simpan
   di drive `HOME` milik kartu (hasil desain: `bootlogo_goodluckos.bmp` diganti nama
   jadi `my-logo.bmp`).
2. Tutup semua jendela Explorer, lalu connect kartu ke VM (bagian 3). Di menu
   GParted Live pilih mode command line: di versi ini nomornya **2** (bukan 1).
   Pesan `cannot set terminal process group` aman, abaikan. Di mode ini sudah login
   sebagai root, jadi tidak perlu `sudo`.
3. Jalankan `lsblk` untuk memastikan kartu = disk 59.5G dengan TRAN `usb`, dan ada
   partisi FAT berlabel `HOME`. Kalau namanya bukan `sda`/`sda2`, sesuaikan `CARD`
   dan `DATA` di bawah.

### Blok 1: cek dulu (read-only, tidak mengubah apa pun)

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

Hasil yang diharapkan:

- `ls /mnt/data` menampilkan `my-logo.bmp`.
- `ls -l /mnt/bootres` menampilkan `bootlogo.bmp`.
- `od` mencetak `640 480`.

Kalau ketiganya cocok, lanjut ke Blok 2. Kalau tidak, **jangan lanjut**. Kalau mount
gagal, jalankan `fdisk -l /dev/sda` untuk melihat tata letak partisi kartu.

### Blok 2: ganti logo (hanya jika Blok 1 cocok)

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

Setelah `cp`, `ls -l` harus menampilkan 921654 byte. Kalau `cp` menampilkan
`No space left`, berhenti.

### Setelah selesai

Setelah dua `umount` selesai tanpa error, matikan VM dengan `poweroff` (atau menu
VMware VM > Power > Shut Down Guest). Kartu otomatis kembali ke Windows; kalau muncul
"You need to format the disk", klik **Cancel**.

1. Salin `bootlogo-stock.bmp` dari drive kartu ke PC sebagai cadangan, lalu hapus
   `bootlogo-stock.bmp` dan `my-logo.bmp` dari kartu.
2. Pasang kartu ke konsol dan nyalakan. Logo Pikachu muncul saat boot.

Offset `37748736` khusus untuk image goodluckOS yang dipakai sekarang. Kalau suatu
saat goodluckOS di-update atau di-reflash dan layout-nya berubah, Blok 1 akan gagal
atau hasilnya tidak cocok. Jangan lanjut ke Blok 2; cek dulu dengan `fdisk -l /dev/sda`.

---

## 6. Kalau OS di-update atau kartu di-reflash

Update/reflash akan menimpa logo kembali ke bawaan. Ulangi:

- **Stock:** bagian 4A (tahap 1) dan 4B (tahap 2).
- **goodluckOS:** bagian 5 (jalankan Blok 1 dulu; offset `37748736` bisa berubah di versi baru).

Versi baru bisa mengubah lokasi atau nama file logo. Untuk melacak ulang (semua
read-only, dari VM):

```bash
mount -o ro /dev/sdX7 /mnt/ark
file /mnt/ark/SYSTEM        # catat kompresi (lzo?) dan blocksize untuk mksquashfs
mount -t squashfs -o loop,ro /mnt/ark/SYSTEM /mnt/sys

# skrip yang menampilkan splash / logo
grep -rl 'splash' /mnt/sys/usr/bin /mnt/sys/usr/sbin /mnt/sys/etc /mnt/sys/usr/lib/systemd /mnt/sys/usr/lib/autostart /mnt/sys/usr/config 2>/dev/null
grep -n 'ffplay' /mnt/sys/usr/bin/lsb_release

# file berukuran persis 640x480 rgb24 (kandidat gambar mentah)
find /mnt/sys -type f \( -iname '*splash*' -o -iname '*logo*' -o -size 921600c \) 2>/dev/null | head -40
ls -l /mnt/sys/usr/config/splash/
```

Kalau gambar yang tampil berbeda dari yang ada di `splash-1080.raw`, baca skripnya
(`cat -n /mnt/sys/usr/bin/lsb_release`) untuk melihat file mana yang sebenarnya
dipanggil. Gambar mentah bisa diperiksa di Windows/Linux dengan:

```bash
ffmpeg -f rawvideo -pix_fmt rgb24 -s 640x480 -i splash-1080.raw cek.png
```

---

## 7. Troubleshooting

| Gejala | Penyebab / solusi |
|--------|-------------------|
| `wsl --mount` error `WSL_E_ELEVATION_NEEDED_TO_MOUNT_DISK` | PowerShell/Terminal harus Run as administrator |
| `wsl --mount` error `ERROR_FILE_NOT_FOUND` / disk in use | Kartu SD tidak bisa dipasang lewat WSL. Pakai VM |
| Nomor `PHYSICALDRIVEn` ternyata disk lain | Cek `Get-CimInstance Win32_DiskDrive`; kartu 16 GB berukuran 15.841.889.280 byte |
| `mount: can't lookup blockdev` | Nama disk berubah setelah connect ulang. `lsblk` lalu pakai nama baru |
| Kartu tidak muncul di Removable Devices | Ubah USB compatibility VM ke 2.0/3.1, atau tutup Explorer yang memakai G:/H: |
| Logo tahap 1 berubah, tahap 2 tetap logo lama | `splash-1080.raw` di dalam `SYSTEM` belum diganti (4B) |
| Warna logo terbalik (merah/biru tertukar) | RAW dibuat dengan urutan BGR. Pakai `convert('RGB')` + `tobytes()` |
| Layar hitam / tidak boot setelah ganti `SYSTEM` | Kembalikan `SYSTEM.orig` (langkah B5) |
| `cp` ke `ARKOS`: `No space left on device` | Lihat catatan di B4 (mount ulang dengan nama disk yang benar, cek ukuran `SYSTEM.new`) |
| goodluckOS: `mount ... offset=...` gagal atau `bootlogo.bmp` tidak muncul | Layout kartu berbeda dari yang diharapkan. Jangan lanjut; jalankan `fdisk -l /dev/sda` dan cek ulang offset |
| goodluckOS: `cannot set terminal process group` | Aman, abaikan |
| Windows minta "You need to format the disk" setelah kartu kembali | Klik **Cancel**. Jangan format |
| VM tidak boot dari ISO | Pastikan drive CD/DVD berisi ISO dan berada di urutan boot pertama (VMware: VM > Power > Power On to Firmware; VirtualBox: Settings > System > Boot Order) |
| Error VT-x/AMD-V, VM 64-bit tidak mau jalan | Aktifkan virtualisasi di BIOS PC (ISO GParted Live yang dipakai 64-bit) |
| VirtualBox: kartu tidak ada di Devices > USB atau tidak terbaca di VM | Settings > USB: aktifkan controller, coba xHCI lalu EHCI; di VirtualBox 6.x ke bawah install Extension Pack yang versinya sama; tutup Explorer yang memakai G:/H: |

---

## 8. Ringkasan alur (stock, versi super singkat)

1. Backup (`H:\backup_logo`).
2. Windows: salin `bootlogo.bmp` baru ke `G:\`.
3. Buat `splash-1080.raw` (921.600 byte).
4. WSL: `unsquashfs` > ganti `/usr/config/splash/splash-1080.raw` > `mksquashfs -comp lzo -b 512K`.
5. VM: salin `SYSTEM.new` ke `ARKOS/SYSTEM`, `md5sum` harus cocok, `umount`.
6. Boot, cek dua tahap logo.

### goodluckOS (versi super singkat)

1. Windows: taruh `my-logo.bmp` (BMP 24-bit 640x480) di drive `HOME` kartu.
2. VM (mode command line **2**): jalankan Blok 1, pastikan tiga hasilnya cocok.
3. Jalankan Blok 2 (backup logo lama ke `bootlogo-stock.bmp`, salin `my-logo.bmp` ke
   volume tersembunyi offset `37748736`), `sync`, `umount` dua kali.
4. `poweroff`, Cancel kalau Windows minta format, bersihkan file di kartu, boot.