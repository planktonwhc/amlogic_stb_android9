# android_build_amlogic_stb

Rencana kerja **porting LineageOS ke STB Amlogic** — dua device sekaligus:

| Device | SoC | Keluarga | Common tree |
|---|---|---|---|
| **ZTE B860H V5** | S905X2 | `g12a` | `device/amlogic/g12-common` |
| **Fiberhome HG680P** | S905X | `gxl` | `device/amlogic/gx-common` |

Semua nilai konfigurasi di dokumen ini diambil langsung dari tree LineageOS upstream
(bukan dari ingatan atau forum), dan dicantumkan sumbernya.

> **Status: rencana, belum dieksekusi.**
> Belum ada build yang dijalankan untuk kedua device ini.

## Isi

| File | Isi |
|---|---|
| [`PLAN.md`](PLAN.md) | Runbook 12 fase, dari cek kelayakan sampai rilis |
| [`docs/porting-lineageos-amlogic.html`](docs/porting-lineageos-amlogic.html) | Versi web dari runbook yang sama |

## Ringkasan matriks

| Parameter | B860H V5 | HG680P |
|---|---|---|
| SoC / keluarga | S905X2 · g12a | S905X · gxl |
| RAM / eMMC | 2 GB / 8 GB | 2 GB / 8 GB |
| GPU | Mali-G31 (Bifrost, Vulkan) | Mali-450 (Utgard, tanpa Vulkan) |
| Kernel | `kernel/amlogic/linux-4.9` | `kernel/amlogic/linux-4.9-pie` |
| Defconfig | `g12a_defconfig` | `meson64_defconfig` |
| DTS basis | `g12a_s905x2_u212.dts` | `gxl_p212_2g.dts` |
| Modul GPU | `mali-driver/bifrost` | `mali-driver/utgard` |
| Modul media | `media-4.9` | `media-pie` |
| Device acuan | `device/askey/dopinder` | — tidak ada |
| WiFi upstream | ada (`rtl8822cs`) | tidak ada (`rtl8189fs` / `rtl8188fu`) |
| Partisi | dynamic (`super` ±1,6 GB) | legacy (system 1,28 GB + vendor 256 MB + odm 128 MB) |
| Boot SD/USB | didukung upstream | harus di-backport |
| Plafon versi | `lineage-22.2` | `lineage-22.1` |
| Rekomendasi | `lineage-22.2` | `lineage-20` |

## Urutan kerja

| Fase | Isi |
|---|---|
| 00 | Kelayakan device — bootloader, burn mode, secure boot |
| 01 | Backup & jaring pengaman — dump eMMC penuh |
| 02 | Build environment |
| 03 | Sync sumber & manifest lokal |
| 04 | Device tree |
| 05 | DTS kernel |
| 06 | WiFi & Bluetooth |
| 07 | Blob proprietary |
| 08 | Build |
| 09 | Instalasi — SD/USB, USB Burning Tool, recovery |
| 10 | Bring-up & debugging |
| 11 | Finishing & rilis |

## Sub-proyek

- **HG680P LineageOS 20** — rencana terpisah yang jauh lebih rinci untuk satu device:
  `android_build_fiberhome_hg680p-20`

## Sumber

- [`LineageOS/android_device_amlogic_g12-common`](https://github.com/LineageOS/android_device_amlogic_g12-common)
- [`LineageOS/android_device_amlogic_gx-common`](https://github.com/LineageOS/android_device_amlogic_gx-common)
- [`LineageOS/android_device_askey_dopinder`](https://github.com/LineageOS/android_device_askey_dopinder)
- [`LineageOS/android_kernel_amlogic_linux-4.9`](https://github.com/LineageOS/android_kernel_amlogic_linux-4.9) · [`linux-4.9-pie`](https://github.com/LineageOS/android_kernel_amlogic_linux-4.9-pie)
- [`TheMuppets/proprietary_vendor_amlogic_gx-common`](https://github.com/TheMuppets/proprietary_vendor_amlogic_gx-common)
- [XDA — LineageOS untuk keluarga Amlogic](https://xdaforums.com/t/official-lineageos-22-for-amlogic-gxl-gxm-g12a-sm1-ne-family-devices-android-tv-tablet.4649881/)
