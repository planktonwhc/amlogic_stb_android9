# Port Android 9 ke ZTE B860H V2 — rencana kerja

## Sasaran

Satu perangkat: **ZTE B860H V2, Amlogic S905X/GXL**. Target pertama **Android 9** dengan UI Android TV. Sasaran khusus proyek Orbit adalah USB gadget/peripheral pada port OTG supaya box dapat menjadi sisi perangkat AOA. V5 dan HG680P ditunda.

Status saat ini: rencana riset; belum ada source tree Android yang disinkronkan atau image hasil build yang diuji. Tidak ada klaim bahwa ROM Android 9 untuk V2 sudah tersedia dan berfungsi.

## Keputusan teknis awal

- **Android 9 diprioritaskan.** LineageOS `android_device_amlogic_gx-common` mendata GXL/S905X dan menyebut kernel Amlogic 4.9 berbasis Android 9. Ini memberi titik awal yang relevan pada level SoC, bukan port B860H siap pakai.
- **Android 10 belum dipilih.** Sebelum menetapkannya, temukan manifest, device tree GXL, kernel, dan vendor blobs yang secara eksplisit cocok satu sama lain untuk S905X dan layout B860H V2. Jangan mencampur framework Android 10 dengan blobs Android 6/9 tanpa bukti kompatibilitas.
- Catatan lokal `../armbian-b860h/README.md` melaporkan `c9100000.usb` sebagai UDC dan role dapat diubah ke `device`, tetapi di file itu ada bagian lain yang menyatakan status gadget belum terkonfirmasi. Ini belum bukti untuk kernel Android. Ulangi seluruh pemeriksaan pada perangkat, lalu pada kernel ROM.
- App Android biasa tidak dapat menggantikan UDC/kernel gadget. Root membantu integrasi sistem, tetapi driver, device tree, konfigurasi kernel, dan akses ke configfs/FunctionFS tetap diperlukan.

## Fase 0 — inventarisasi unit V2

Simpan keluaran berikut sebelum melakukan perubahan:

```sh
adb shell getprop > b860h-v2-getprop.txt
adb shell uname -a
adb shell cat /proc/cmdline
adb shell cat /proc/meminfo
adb shell ls -l /dev/block/by-name
adb shell su -c 'ls -l /sys/class/udc; find /sys/class/usb_role -maxdepth 3 -type f 2>/dev/null'
adb shell su -c 'dmesg | grep -iE "dwc2|dwc3|udc|gadget|otg|role"'
```

Catat juga:

- nomor revisi yang tercetak di PCB dan foto kedua sisi;
- RAM/eMMC, chip Wi-Fi, dan port fisik yang dipakai untuk OTG;
- versi Android, kernel, build fingerprint, bootloader, dan status root;
- apakah kabel A-to-A/USB Burning Tool mendeteksi mode burn, tanpa mulai menulis image.

Identitas `ro.board.platform` atau nama pemasaran saja tidak cukup untuk memilih DTB. Jika tersedia, simpan DTB stock dan serial log boot.

**Checkpoint:** V2 teridentifikasi dengan bukti dari unit; tabel partisi dan metode recovery diketahui.

## Fase 1 — backup dan pemulihan

Sebelum flash atau mengubah bootloader:

1. Dump eMMC penuh ke media eksternal, simpan checksum, dan buat salinan kedua.
2. Simpan partisi boot, recovery, dtb, vendor, system, bootloader/env bila dapat dibaca, serta firmware resmi yang cocok dengan revisi tersebut.
3. Catat prosedur masuk recovery/burn mode dan siapkan kabel, driver, serta host flashing yang benar.
4. Jangan uji image dengan menimpa eMMC. Utamakan boot SD/USB bila U-Boot unit mendukungnya; buktikan bahwa metode tersebut tidak menyentuh eMMC.

**Checkpoint:** berkas backup dapat dibaca, ukuran dan checksum tercatat, jalur restore tersedia.

## Fase 2 — audit basis Android 9

Jangan mulai build besar sebelum setiap dependency memiliki branch yang nyata dan saling cocok.

1. Periksa branch/tag yang tersedia pada `LineageOS/android_device_amlogic_gx-common` dan dependency-nya.
2. Temukan **device-specific tree untuk B860H V2**. Jika tidak ada, perlakukan `gxl_p212_2g`/p212 sebagai referensi sementara saja, bukan board config final.
3. Pastikan kernel yang ditarik benar-benar kernel GXL Android 9 4.9, lalu cari dukungan board DTS B860H V1/V2 dan controller OTG-nya.
4. Audit vendor blobs: cocokkan versi Android, ABI, HAL display/media, audio, Wi-Fi, dan init scripts. Android 6 stock dump berguna untuk identifikasi hardware, tetapi belum tentu cocok langsung dengan userspace Android 9.
5. Periksa layout partisi V2 dari unit. Jangan memakai ukuran `super`, dynamic partition, atau fstab device lain sebagai asumsi.
6. Catat hasil sebagai tabel `komponen / sumber / branch / bukti cocok / status` sebelum menulis local manifest.

**Checkpoint:** kernel, common tree, device tree, blobs, dan layout partisi punya rantai kompatibilitas yang dapat dijelaskan. Jika komponen kritis tidak ada, berhenti di riset/bring-up Linux dan jangan mengarang kombinasi build.

## Fase 3 — device tree dan product tree

Buat `device/zte/b860hv2/` hanya setelah fase 2 mengunci nama DTS, codename, dan partisi. Kerangka yang nantinya dibutuhkan:

```text
device/zte/b860hv2/
├── AndroidProducts.mk
├── BoardConfig.mk
├── lineage_b860hv2.mk
├── device.mk
├── proprietary-files.txt
├── init-files/
├── fstab/
├── keylayout/
└── sepolicy/
```

Jangan salin nilai dari B860H V5 (G12A/S905X2), HG680P, atau board p212 generik tanpa membandingkan dengan DTB/partisi V2. Khususnya validasi:

- RAM, DDR timing, eMMC, pin reset dan power;
- node HDMI/display dan audio;
- Ethernet PHY dan Wi-Fi SDIO/power sequence;
- port USB OTG, controller, `dr_mode`/role switch, VBUS, dan pinctrl;
- fstab serta nama dan offset partisi aktual.

## Fase 4 — build image Android 9 minimal

Lakukan build hanya setelah manifest dan tree sudah pasti. Gunakan host Linux dengan ruang disk dan RAM memadai; build AOSP/LineageOS penuh tidak praktis dikerjakan pada TV Box.

Tahap pertama hanya perlu menghasilkan image yang dapat boot dan memberi log serial/ADB. Jangan tambahkan GApps, optimasi, atau banyak aplikasi sebelum HDMI, storage, dan init stabil.

Validasi berurutan:

1. U-Boot memuat kernel dan DTB yang dimaksud.
2. Kernel mencapai userspace; serial log tidak menunjukkan panic.
3. Mount partisi tepat dan data tidak ditulis ke partisi yang salah.
4. HDMI menghasilkan gambar 1080p yang stabil.
5. ADB, Ethernet, Wi-Fi, remote/input, audio, suspend/reboot berfungsi.
6. Video decode HEVC yang dibutuhkan prototype berjalan pada resolusi dan bitrate target.

## Fase 5 — USB gadget pada Android

Validasi hardware dan kernel secara terpisah dari AOA DJI:

1. Pastikan UDC terdaftar (`/sys/class/udc` tidak kosong).
2. Identifikasi node role-switch atau mode controller; pastikan port OTG fisik dengan uji aman.
3. Pastikan configfs/libcomposite dan FunctionFS tersedia, atau dokumentasikan backend gadget lain yang disediakan kernel.
4. Atur peripheral mode melalui mekanisme yang didukung kernel dan pastikan bertahan setelah reboot.
5. Enumerasikan gadget uji dengan komputer host; verifikasi descriptor, endpoint bulk IN/OUT, transfer berulang, dan log kernel.
6. Uji stabilitas listrik/VBUS dan suhu dalam sesi panjang.
7. Setelah uji generik lulus, integrasikan `orbit-rs`/AOA dan lakukan handshake DJI.

Jangan menyimpulkan AOA tersedia hanya dari USB OTG, ADB, MTP, atau nama UDC. Kriteria berhasil untuk proyek adalah host mengenali profil AOA yang diperlukan dan data diterima konsisten.

## Fase 6 — integrasi viewer Orbit

- Pasang service native untuk konfigurasi gadget dan lifecycle koneksi.
- Terapkan profil RC-N/AOA terlebih dulu; Goggles menyusul setelah profil pertama stabil.
- Baca stream DUML/frame video sesuai implementasi Orbit yang telah direferensikan.
- Decode HEVC dan render clean stream ke HDMI tanpa OSD.
- Ukur latensi ujung ke ujung, frame drop, reconnect, suhu, dan pemakaian RAM.

## Kriteria selesai prototype V2

- ROM Android 9 boot stabil pada B860H V2 dan output HDMI berfungsi.
- Port yang benar berjalan sebagai USB peripheral, UDC/configfs/FunctionFS tersedia.
- Gadget uji menyelesaikan transfer bulk berulang tanpa reset/error.
- Profil RC-N/AOA melakukan handshake dan stream tampil clean di HDMI.
- Backup serta prosedur pemulihan V2 terdokumentasi.

Android 10 dan B860H V5 baru dibuka sebagai pekerjaan berikutnya setelah kriteria V2 tercapai atau ditemukan BSP Android 10 yang lebih cocok tanpa mengorbankan dukungan gadget.

## Referensi

- [LineageOS Amlogic GX common tree](https://github.com/LineageOS/android_device_amlogic_gx-common) — menyebut GXL/S905X dan kernel Amlogic 4.9 berbasis Android 9.
- [Referensi DTS Linux B860H V1/V2](https://git.mrlokop.dev/MRLokop/pmaports/commit/bac841dcaee3dfb4fc6da384f78a6cb0e2f191be) — berguna untuk perbandingan hardware, bukan pengganti DTS Android.
- [Armbian B860H V2 (lokal)](/Users/macpro/orbit-stream/armbian-b860h/README.md) — catatan eksperimen gadget dengan klaim status yang perlu direkonsiliasi dan diuji ulang.
- [Linux USB Gadget API](https://docs.kernel.org/driver-api/usb/gadget.html).
