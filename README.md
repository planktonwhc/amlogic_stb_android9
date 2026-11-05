# Android ROM untuk ZTE B860H V2

Proyek ini fokus **hanya pada ZTE B860H V2 berbasis Amlogic S905X (GXL)**. Targetnya menyiapkan sistem Android TV yang bisa menjalankan prototype Orbit dan menyediakan USB gadget/AOA pada port OTG. **B860H V5 ditunda** dan tidak menjadi acuan konfigurasi maupun build.

## Target versi

Mulai dari **Android 9 / LineageOS berbasis Android 9** sebagai target bring-up. Common tree LineageOS untuk Amlogic GXL menyatakan dukungan S905X dan memakai kernel Amlogic 4.9 berbasis Android 9. Ini membuat Android 9 titik awal yang lebih berdasar daripada Android 10. Dukungan upstream tersebut bukan bukti bahwa image siap untuk B860H V2: device tree, partisi, vendor blobs, boot chain, dan port OTG tetap perlu dicocokkan dengan unit.

Android 10 menjadi target lanjutan hanya jika ditemukan basis BSP/device tree GXL yang cocok dengan board ini. Jangan menganggap upgrade dari Android 6 sebagai perubahan framework saja: kernel, vendor HAL/blobs, DTB, dan layout partisi harus serasi.

## Status

- Belum ada ROM Android 9/10 yang dibangun atau diverifikasi dalam folder ini.
- B860H V2 dilaporkan memakai S905X/GXL; verifikasi unit melalui `getprop`, DTB, dan foto PCB sebelum memilih konfigurasi final.
- Catatan Armbian lokal untuk B860H V2 melaporkan UDC `c9100000.usb` dan pergantian role ke `device`, tetapi README-nya juga memiliki pernyataan yang saling bertentangan tentang status verifikasi. Perlakukan ini sebagai petunjuk awal; ulangi pengecekan di unit dan kernel yang dipakai ROM.
- UDC aktif belum membuktikan port fisik, profil AOA DJI, atau video stream berfungsi.

## Tahapan kerja

1. Identifikasi revisi V2, RAM/eMMC, Wi-Fi, bootloader, partisi, kernel, DTB, dan port OTG.
2. Backup penuh eMMC dan pastikan metode pemulihan sebelum mengubah partisi.
3. Validasi boot Linux dari SD/USB dan kumpulkan DTB/serial log sebagai referensi hardware.
4. Siapkan device tree B860H V2 berdasarkan GXL, lalu bangun image Android 9 minimal.
5. Bring-up HDMI, storage, Ethernet/Wi-Fi, audio, dan remote.
6. Aktifkan serta uji USB gadget di kernel Android: UDC, role peripheral, configfs/FunctionFS, lalu enumerasi dan bulk transfer dengan host uji.
7. Baru integrasikan AOA Orbit dan uji RC-N; Goggles tetap tahap berikutnya.

Runbook: [PLAN.md](PLAN.md). Panduan host Debian Trixie: [build.md](build.md). Jalur pemulihan dan no-write eMMC: [rescue.md](rescue.md). Referensi proyek Armbian B860H V2: `/Users/macpro/orbit-stream/armbian-b860h`.

## Sumber teknis awal

- [LineageOS Amlogic GX common tree](https://github.com/LineageOS/android_device_amlogic_gx-common) — mendata keluarga GXL/S905X dan basis kernel Android 9 / Linux 4.9.
- [DTS B860H V1/V2 di pmaports](https://git.mrlokop.dev/MRLokop/pmaports/commit/bac841dcaee3dfb4fc6da384f78a6cb0e2f191be) — referensi device tree Linux; bukan DTS Android siap pakai.
- [Linux USB Gadget API](https://docs.kernel.org/driver-api/usb/gadget.html) — konsep UDC dan gadget driver.

## Di luar cakupan saat ini

- B860H V5/S905X2.
- Fiberhome HG680P.
- Raspberry Pi sebagai perangkat produk.
- USB host/RNDIS pada TV Box; transport yang dituju hanya USB gadget/AOA.
