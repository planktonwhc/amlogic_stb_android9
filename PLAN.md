# Porting LineageOS ke STB Amlogic — Runbook 12 Fase

Berlaku untuk **ZTE B860H V5** (S905X2 · g12a) dan **Fiberhome HG680P** (S905X · gxl).
Yang berbeda antar device ditandai per bagian.

---

## Fase 00 — Kelayakan device

**Tujuan:** membuktikan STB mau menjalankan image yang bukan dari operator.

Fase paling penting dan paling sering dilewati. Kalau bootloader terkunci rapat,
semua kerja device tree jadi sia-sia.

### Baca identitas board

```bash
adb shell getprop | grep -E "ro.board.platform|ro.product.device|ro.build.version|ro.treble.enabled"
adb shell ls -l /dev/block/by-name/     # layout partisi: ada 'super'?
adb shell cat /proc/meminfo | head -1
adb shell cat /proc/cpuinfo | tail -20
```

`ro.board.platform` harus `g12a` (B860H V5) atau `gxl` (HG680P).

### Uji burn mode

Kabel **USB A-to-A male** ke port USB dekat catu daya. Tekan tombol reset di lubang
pinhole pakai tusuk gigi sambil menyalakan daya. USB Burning Tool harus menampilkan
*"Connect success"*.

### Uji boot dari luar eMMC

Tulis **Armbian** ke microSD atau flashdisk dan coba boot. Kalau Armbian jalan,
`aml_autoscript` dieksekusi dan u-boot tidak memaksa signature — lampu hijau.

> **Secure boot.** Sebagian unit di-fuse dengan kunci vendor. Gejalanya: burn mode
> terdeteksi tapi flash selalu gagal di tahap bootloader, dan tidak ada image SD yang
> mau boot. Kalau ini kasusnya, hentikan proyek di unit tersebut.

**Checkpoint:** burn mode terdeteksi, Armbian dari SD/USB boot, `ro.board.platform`
dan daftar `by-name` sudah dicatat.

---

## Fase 01 — Backup & jaring pengaman

**Tujuan:** apa pun yang rusak nanti, bisa dikembalikan ke kondisi pabrik.

> **Jangan dilewati.** Tanpa dump eMMC, satu kesalahan flash bootloader mengubah STB
> jadi barang mati permanen. Ini satu-satunya asuransi yang benar-benar bekerja —
> bukan SD boot, bukan "hati-hati".

```bash
# dari Armbian yang sudah boot di Fase 00
lsblk                                  # cari device eMMC
dd if=/dev/mmcblk1 of=/mnt/usb/stock-emmc-full.img bs=8M status=progress
sha256sum /mnt/usb/stock-emmc-full.img > /mnt/usb/stock-emmc-full.sha256

dd if=/dev/mmcblk1boot0 of=/mnt/usb/bootloader-boot0.img
cat /proc/partitions > /mnt/usb/partitions.txt
```

Simpan juga:

- **DTB stock** — bahan baku Fase 05
- **Firmware pabrik** `aml_upgrade_package.img` kalau tersedia untuk versi OTA unitmu
- **Foto PCB dua sisi** resolusi tinggi — untuk pad UART, test point eMMC, revisi board
- **Catatan chip WiFi** — baca tulisan di modul kaleng, menentukan Fase 06

Siapkan sebelum dibutuhkan: kabel USB A-to-A male, tusuk gigi, pinset, USB Burning
Tool dengan driver terpasang, adaptor USB-TTL 3.3V.

**Checkpoint:** dump eMMC penuh tersimpan di dua lokasi dengan checksum, dan sudah
pernah sekali berhasil di-restore.

---

## Fase 02 — Build environment

| Sumber daya | Minimum | Nyaman |
|---|---|---|
| Disk | 300 GB | 500 GB SSD NVMe |
| RAM | 16 GB + 16 GB swap | 32 GB |
| CPU | 4 core | 12 core+ |
| OS | Ubuntu 22.04 | Ubuntu 24.04 |
| Build pertama | ± 6 jam | ± 1,5 jam |

```bash
sudo apt update && sudo apt install -y \
  bc bison build-essential ccache curl flex g++-multilib gcc-multilib \
  git git-lfs gnupg gperf imagemagick lib32readline-dev lib32z1-dev \
  libelf-dev liblz4-tool libsdl1.2-dev libssl-dev libxml2 libxml2-utils \
  lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev \
  python3 python-is-python3 device-tree-compiler

mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=~/bin:$PATH

export USE_CCACHE=1 CCACHE_EXEC=/usr/bin/ccache CCACHE_DIR=~/.ccache
ccache -M 100G && ccache -o compression=true
```

```bash
# B860H V5
repo init --git-lfs -u https://github.com/LineageOS/android.git -b lineage-22.2
# HG680P
repo init --git-lfs -u https://github.com/LineageOS/android.git -b lineage-20
```

---

## Fase 03 — Sync sumber & manifest lokal

`breakfast` otomatis menarik turunan dari `lineage.dependencies` tiap tree:

| Tree | Menarik |
|---|---|
| `askey/dopinder` | `amlogic/g12-common` · `kernel-modules/rtl8822cs` · `hardware/realtek` |
| `amlogic/g12-common` | `amlogic/common` · `mali-driver` · `media-4.9` · `optee` · `linux-4.9` |
| `amlogic/gx-common` | `amlogic/common` · `mali-driver` · `media-pie` · `optee-pie` · `linux-4.9-pie` |

`.repo/local_manifests/amlogic.xml` — contoh B860H V5:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project name="rigaz29/android_device_zte_b860hv5"
           path="device/zte/b860hv5"
           remote="github" revision="lineage-22.2" />

  <project name="TheMuppets/proprietary_vendor_amlogic_g12-common"
           path="vendor/amlogic/g12-common"
           remote="github" revision="lineage-22.2" />
  <project name="TheMuppets/proprietary_vendor_askey_dopinder"
           path="vendor/askey/dopinder"
           remote="github" revision="lineage-22.2" />
</manifest>
```

```bash
repo sync -c -j$(nproc --all) --force-sync --no-clone-bundle --no-tags
```

> Branch di repo TheMuppets tidak selalu sama persis dengan branch LineageOS.
> Cek dulu branch yang benar-benar ada sebelum menulis `revision=`.

---

## Fase 04 — Device tree

Salin struktur `device/askey/dopinder` — device S905X2 nyata yang resmi didukung.

```
device/zte/b860hv5/
├── AndroidProducts.mk        # daftar product makefile
├── BoardConfig.mk            # DTB, partisi, WiFi, modul kernel
├── lineage_b860hv5.mk        # identitas produk + inherit ATV
├── device.mk                 # paket, HAL, permission
├── extract-files.py
├── setup-makefiles.py
├── proprietary-files.txt
├── proprietary-firmware.txt
├── init-files/               # fstab + init.*.rc
├── keylayout/                # pemetaan tombol remote
├── rro_overlays/             # overlay resolusi, overscan
└── sepolicy/vendor/
```

### BoardConfig.mk — B860H V5 (g12a)

```make
DEVICE_PATH := device/zte/b860hv5

TARGET_BOOTLOADER_BOARD_NAME := b860hv5
TARGET_DTB_NAME := g12a_s905x2_b860hv5

TARGET_KERNEL_EXT_MODULES := \
    rtl8822cs/rtl88x2CS:kbuild

BOARD_SUPER_PARTITION_SIZE := 1677721600

BOARD_WLAN_DEVICE := realtek
BOARD_WPA_SUPPLICANT_DRIVER := NL80211
WPA_SUPPLICANT_VERSION := VER_0_8_X
BOARD_HOSTAPD_PRIVATE_LIB := lib_driver_cmd_rtl
BOARD_WPA_SUPPLICANT_PRIVATE_LIB := lib_driver_cmd_rtl
WIFI_DRIVER_SOCKET_IFACE := wlan0

SELINUX_IGNORE_NEVERALLOWS := true
BOARD_VENDOR_SEPOLICY_DIRS += $(DEVICE_PATH)/sepolicy/vendor

include device/amlogic/g12-common/BoardConfigCommon.mk
```

### BoardConfig.mk — HG680P (gxl)

```make
DEVICE_PATH := device/fiberhome/hg680p

TARGET_BOOTLOADER_BOARD_NAME := hg680p
TARGET_DTB_NAME := gxl_p212_2g_hg680p

TARGET_KERNEL_EXT_MODULES := \
    rtl8189fs:kbuild

# partisi legacy, bukan super
BOARD_SYSTEMIMAGE_PARTITION_SIZE := 1342177280
BOARD_VENDORIMAGE_PARTITION_SIZE := 268435456
BOARD_ODMIMAGE_PARTITION_SIZE    := 134217728

SELINUX_IGNORE_NEVERALLOWS := true
BOARD_VENDOR_SEPOLICY_DIRS += $(DEVICE_PATH)/sepolicy/vendor

include device/amlogic/gx-common/BoardConfigCommon.mk
```

### Product makefile

Dua baris inherit pertama menentukan varian Android TV. Tanpa keduanya kamu dapat
UI tablet.

```make
# harus paling atas, dibaca sebelum apa pun
PRODUCT_IS_ATV := true

$(call inherit-product, device/google/atv/products/atv_base.mk)
$(call inherit-product, vendor/lineage/config/common_full_tv.mk)
$(call inherit-product, $(LOCAL_PATH)/device.mk)

PRODUCT_BRAND        := ZTE
PRODUCT_DEVICE       := b860hv5
PRODUCT_MANUFACTURER := zte
PRODUCT_MODEL        := B860H V5
PRODUCT_NAME         := lineage_b860hv5
```

**Checkpoint:** `source build/envsetup.sh && breakfast <codename>` selesai tanpa error.

---

## Fase 05 — DTS kernel

Bagian yang paling menentukan apakah device boot atau layar tetap gelap.

```bash
adb shell su -c 'dd if=/dev/block/dtb of=/sdcard/dtb.img'
adb pull /sdcard/dtb.img

# partisi dtb Amlogic biasanya container multi-DTB (magic 'AML_DT'),
# berisi beberapa DTB untuk varian RAM berbeda — pisahkan dulu
dtc -I dtb -O dts -o stock.dts single.dtb
```

| Device | DTS basis di kernel LineageOS |
|---|---|
| B860H V5 | `arch/arm64/boot/dts/amlogic/g12a_s905x2_u212.dts` |
| HG680P | `arch/arm64/boot/dts/amlogic/gxl_p212_2g.dts` |

Yang hampir selalu berbeda dari board referensi, urut dari yang paling sering
bikin gagal boot:

1. **Tabel partisi** — node `partitions` menentukan seluruh `by-name` yang dilihat Android
2. **Timing dan kapasitas DDR** — salah di sini berarti panic sangat awal atau layar mati
3. **PHY ethernet** — alamat MDIO dan reset GPIO berbeda antar vendor
4. **Node SDIO WiFi** — `sd_emmc_a` di GXL (`sdio@d0070000`), lengkap dengan GPIO power enable
5. **IR receiver, LED, tombol reset** — nomor GPIO khas tiap board

> **Alamat controller berbeda per keluarga.**
> GXL: `d0074000.emmc` (eMMC) dan `d0072000.sd` (kartu SD).
> G12A: `ffe07000.emmc` dan `ffe05000.sd2`.
> Salah nilai berarti Android tidak menemukan satu partisi pun.

---

## Fase 06 — WiFi & Bluetooth

Upstream hanya menyediakan `rtl8822bs`, `rtl8822cs`, Broadcom `dhd-driver`, dan
`rtk_btusb`. Di luar itu, kamu yang menyediakan.

**B860H V5** — kalau modulnya memang RTL8822CS, cukup daftarkan modul yang sudah ada.
Buka casing dan baca tulisan di modul WiFi sebelum mengasumsikan ini; revisi board V5
tidak seragam.

**HG680P** — dua varian: casing **hitam** RTL8189FS (SDIO), casing **putih** RTL8188FU
(USB). Keduanya harus ditambahkan sebagai modul kernel eksternal.

> **Strategi build pertama.** Jangan tunggu WiFi beres. Build pertama sebaiknya tanpa
> WiFi sama sekali dan mengandalkan ethernet. Debugging WiFi di atas sistem yang belum
> pernah boot berarti menebak dua variabel sekaligus.

---

## Fase 07 — Blob proprietary

### Jalur A — blob generic TheMuppets (disarankan)

| Repo | Path | Untuk |
|---|---|---|
| `proprietary_vendor_amlogic_g12-common` | `vendor/amlogic/g12-common` | B860H V5 |
| `proprietary_vendor_askey_dopinder` | `vendor/askey/dopinder` | B860H V5 |
| `proprietary_vendor_amlogic_gx-common` | `vendor/amlogic/gx-common` | HG680P |

Untuk HG680P ini keuntungan besar: `gx-common` berisi blob generic Amlogic era Pie,
jadi **tidak perlu** mengekstrak apa pun dari firmware Android 6 bawaan Fiberhome —
yang memang tidak akan cocok dengan kernel 4.9.

### Jalur B — extract dari dump sendiri

```bash
./extract-files.py adb                    # dari device yang menyala dan sudah di-root
./extract-files.py /path/ke/dump/stock    # dari dump Fase 01
```

Pakai ini hanya untuk blob yang benar-benar khas unitmu — firmware WiFi, kalibrasi,
kunci HDCP.

> **Kecocokan versi.** Kalau muncul crash berulang di `hwcomposer` atau
> `media.amlogic`, penyebabnya hampir selalu campuran blob dari dua era berbeda —
> bukan device tree kamu.

---

## Fase 08 — Build

```bash
source build/envsetup.sh
breakfast <codename>

lunch lineage_<codename>-userdebug          # LineageOS 20 dan 21
lunch lineage_<codename>-bp1a-userdebug     # LineageOS 22.x (release config)

mka bacon -j$(nproc --all)
mka recoveryimage
```

Kalau nama target ditolak, jalankan `lunch` tanpa argumen dan pilih dari daftar.

| Gejala | Penyebab umum |
|---|---|
| Device tidak muncul di `lunch` | `AndroidProducts.mk` tidak menunjuk product makefile, atau `PRODUCT_NAME` ≠ nama file |
| Gagal di `neverallow` SELinux | Belum set `SELINUX_IGNORE_NEVERALLOWS`, atau sepolicy vendor belum lengkap |
| DTB tidak ditemukan | `TARGET_DTB_NAME` tidak cocok dengan nama file DTS, atau DTS belum didaftarkan di Makefile kernel |
| Kehabisan RAM saat linking | `mka bacon -j4`, tambah swap |
| Image melebihi partisi | Naikkan `BOARD_*_PARTITION_SIZE` atau kurangi paket di `device.mk` |

---

## Fase 09 — Instalasi

### Jalur A — boot dari SD atau USB (non-destruktif)

Jalur bring-up yang benar: eMMC stock tetap utuh. Upstream `g12-common` sudah
menyediakannya lengkap dengan varian tanpa AVB.

```make
TARGET_BOOTDEVICE := sdcard      # atau: usb
```

Efeknya: cmdline berubah ke `androidboot.boot_devices=ffe05000.sd2`, dan fstab beralih
ke `fstab.sdcard.amlogic` yang memetakan `/data` ke `mmcblk1p5`, `/metadata` ke
`mmcblk1p2`. Varian `fstab_no_avb.sdcard.amlogic` mematikan verifikasi.

Untuk mengarahkan u-boot ke kartu tanpa menulis eMMC, taruh `aml_autoscript` di
partisi FAT kartu — mekanisme yang sama dipakai multiboot Armbian.

> **Perlu diverifikasi sendiri.** Di mode sdcard, entri `/boot`, `/dtbo`, `/misc`,
> `/recovery` masih memakai node `by-name` Amlogic. Cek ke mana node itu resolve di
> unitmu sebelum menyimpulkan eMMC tidak tersentuh sama sekali.

Untuk HG680P blok ini belum ada di `gx-common` — salin polanya dari `g12-common`
dan ganti alamat controller ke `d0072000.sd`.

### Jalur B — USB Burning Tool (permanen)

1. Ambil paket firmware stock sebagai kerangka, bongkar dengan `imagepacker` di
   `hardware/amlogic/tools/imagepacker`
2. Ganti isinya dengan hasil build: `boot.img`, `recovery.img`, `super.img` atau trio
   system/vendor/odm, dan DTB baru
3. Repack, flash lewat USB Burning Tool dengan **Erase flash** aktif

### Jalur C — recovery sideload (update berikutnya)

```bash
adb reboot recovery
adb sideload out/target/product/<codename>/lineage-*.zip
```

---

## Fase 10 — Bring-up & debugging

> **Serial console dulu.** Cari pad RX / TX / GND di PCB, pakai adaptor USB-TTL
> **3.3V**, buka pada 115200 8N1. Tanpa serial kamu buta terhadap semua kegagalan
> sebelum `adbd` hidup — dan di situlah sebagian besar masalah terjadi.

```make
WITH_CONSOLE := true
# menambahkan: console=ttyS0,115200 no_console_suspend ignore_loglevel
```

| Gejala | Kemungkinan penyebab | Langkah |
|---|---|---|
| Layar gelap, serial diam | DTB salah atau timing DDR | Kembali ke DTS referensi murni, ubah satu blok per percobaan |
| Panic saat mount | Tabel partisi DTS ≠ layout eMMC, atau `boot_devices` salah | Cocokkan node `partitions` dengan `by-name` Fase 00 |
| Bootloop di logo | SELinux atau blob tidak cocok | `logcat` dari recovery, cari `avc: denied` dan crash HAL |
| UI berkedip | hwcomposer versi tidak cocok | Samakan blob display dengan versi common tree |
| Ethernet mati | Alamat PHY atau reset GPIO | Diff blok ethernet dengan DTS stock |
| Video patah-patah | Modul `media-*` tidak termuat | Cek `lsmod` dan urutan di modules.load |
| Tidak ada audio HDMI | Node audio DTS tidak lengkap | Salin blok audio dari DTS stock |

---

## Fase 11 — Finishing & rilis

- **Remote** — buat keylayout; `dopinder` memakai pola `keylayout/Vendor_0001_Product_0001.kl`.
  Petakan minimal: arah, OK, Back, Home, Menu, volume.
- **Tampilan** — overlay resolusi, DPI, overscan lewat `rro_overlays/`. Set mode HDMI
  default 1080p60 supaya tidak ada unit yang boot ke layar kosong.
- **Signing:**

```bash
./development/tools/make_key ~/.android-certs/releasekey '/CN=Lineage/'
```

Yang perlu dijelaskan ke pemakai:

- **Widevine turun ke L3** — kunci HDCP pabrik tidak ikut, Netflix dan Disney+ hanya SD. Permanen, bukan bug.
- **GApps Android TV opsional** dan menambah beban di RAM 2 GB.
- **Sertifikasi hilang** — sebagian aplikasi TV akan menolak jalan.

---

## Estimasi waktu

| Fase | Waktu wajar | Risiko terbesar |
|---|---|---|
| 00–01 Kelayakan & backup | 1–2 hari | Secure boot terkunci — proyek berhenti di sini |
| 02–03 Environment & sync | 1 hari | Bandwidth; source penuh 200 GB lebih |
| 04 Device tree | 2–4 hari | Rendah, sebagian besar menyalin pola |
| 05 DTS | 3 hari – 2 minggu | **Tertinggi.** Di sinilah sebagian besar proyek mandek |
| 06 WiFi | 2 hari – 1 minggu | Tinggi untuk HG680P, nyaris nol untuk B860H V5 |
| 07–08 Blob & build | 2–5 hari | Ketidakcocokan versi blob |
| 09–10 Instalasi & bring-up | 1–3 minggu | Tanpa UART bisa berlipat ganda |
| 11 Finishing | 3–7 hari | Rendah |

**Yang tidak disediakan upstream** — empat hal ini murni pekerjaanmu: DTS board,
driver WiFi HG680P, keylayout remote, dan paket instalasi Amlogic. Sisanya sudah ada
tinggal dipakai.
