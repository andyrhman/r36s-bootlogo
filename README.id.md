# Custom Boot Logo untuk R36S Clone – GA36-MB V1.1 (20251025) (Stock "ArkOS 2.0" & goodluckOS)

Catatan pribadi supaya tidak perlu cari-cari command di chat lagi.
Perangkat: R36S clone, board GA36 V1.1 (Allwinner, kernel `sun8i`).
Kartu stock asli: 16 GB (terbaca 14,8 GB).

---

## 1. Ringkasan cepat

| OS | Tahap | File | Lokasi | Cara ganti |
|----|-------|------|--------|-----------|
| Stock | 1 (loader) | `bootlogo.bmp` | partisi `Volumn` (drive **G:**) | Langsung dari Windows |
| Stock | 2 (setelah Linux jalan) | `splash-1080.raw` | **di dalam `SYSTEM`** (squashfs), path `/usr/config/splash/` | Bangun ulang `SYSTEM` (VM + WSL) |
| goodluckOS | 1 | `bootlogo.bmp` | partisi FAT kecil yang tidak muncul di Windows | Lewat VM (GParted Live) |

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
| `bootlogo_goodluckos.bmp` | goodluckOS (ganti nama jadi `bootlogo.bmp`) |

---

## 3. Persiapan: akses kartu SD lewat VM

`wsl --mount` **tidak bisa** untuk kartu SD ini (error `ERROR_FILE_NOT_FOUND`).
Pakai VMware Workstation + GParted Live.

1. Settings VM: CD/DVD diarahkan ke ISO GParted Live. USB Controller harus ada
   (kalau kartu tidak muncul, ubah USB compatibility ke 2.0 atau 3.1).
2. Tutup semua jendela Explorer yang membuka G: atau H:.
3. Nyalakan VM. Menu GParted Live: pilih opsi default, keymap
   "Don't touch keymap", bahasa tekan Enter, mode `0` (GUI).
4. Colok kartu, lalu di VMware: **VM > Removable Devices > Mass Storage Device
   USB Device > Connect (Disconnect from Host)**.
5. Buka Terminal di GParted Live, lalu jadi root:

```bash
sudo -i
lsblk -o NAME,SIZE,TRAN,MODEL,FSTYPE,LABEL
```

**Nama disk (`sda`, `sdb`, `sdc` ...) berubah setiap kartu di-connect ulang.**
Selalu jalankan `lsblk` dulu, dan ganti `sdX` di semua command di bawah.
Kalau perlu memutuskan kartu dari VM (supaya G:/H: muncul di Windows lagi):
**VM > Removable Devices > ... > Disconnect**.

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

Pada goodluckOS, `bootlogo.bmp` ada di partisi FAT kecil yang tidak muncul di
Windows (lihat issue #111 di repo goodluckOS). Caranya sama dengan stock tahap 1,
tapi lewat VM.

1. Siapkan `bootlogo.bmp` (BMP 24-bit 640x480, lihat bagian 2). Taruh di partisi
   kartu yang bisa dibaca Windows (misalnya partisi ROM FAT32) sebagai jembatan.
2. Connect kartu goodluckOS ke VM GParted Live (bagian 3), jalankan `lsblk`.
3. Cari partisi FAT kecil yang berisi `bootlogo.bmp`. Mount satu per satu secara
   read-only dan lihat isinya:

   ```bash
   mkdir -p /mnt/b
   mount -o ro /dev/sdXn /mnt/b && ls -la /mnt/b; umount /mnt/b
   ```

   Yang benar berisi `bootlogo.bmp` (di stock partisi sejenis juga berisi
   `font24.sft`, `font32.sft`, `magic.bin`, folder `bat`).
4. Mount read-write, backup, ganti:

   ```bash
   mount /dev/sdXn /mnt/b
   cp /mnt/b/bootlogo.bmp /mnt/b/bootlogo.bmp.orig
   cp /mnt/<jembatan>/bootlogo.bmp /mnt/b/bootlogo.bmp
   sync
   ls -l /mnt/b/bootlogo.bmp          # harus 921654 byte
   umount /mnt/b
   ```
5. Disconnect kartu, pasang ke konsol, boot.

> Bagian goodluckOS ini disusun dari metode yang sama dengan stock plus issue #111.
> Kalau cara yang dulu berhasil di goodluckOS berbeda (misalnya nama/nomor
> partisinya), perbarui bagian ini.

---

## 6. Kalau OS di-update atau kartu di-reflash

Update/reflash akan menimpa logo kembali ke bawaan. Ulangi:

- **Stock:** bagian 4A (tahap 1) dan 4B (tahap 2).
- **goodluckOS:** bagian 5.

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

---

## 8. Ringkasan alur (stock, versi super singkat)

1. Backup (`H:\backup_logo`).
2. Windows: salin `bootlogo.bmp` baru ke `G:\`.
3. Buat `splash-1080.raw` (921.600 byte).
4. WSL: `unsquashfs` > ganti `/usr/config/splash/splash-1080.raw` > `mksquashfs -comp lzo -b 512K`.
5. VM: salin `SYSTEM.new` ke `ARKOS/SYSTEM`, `md5sum` harus cocok, `umount`.
6. Boot, cek dua tahap logo.
