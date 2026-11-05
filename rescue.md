# Rescue dan boot SD tanpa menulis eMMC — B860H V2

Dokumen ini khusus **ZTE B860H V2 / Amlogic S905X (GXL)**. Tujuannya mempertahankan Android dan bootloader bawaan di eMMC sambil menguji sistem custom dari microSD.

## Jawaban singkat

Idenya benar dengan syarat, tetapi **`u-boot.bin` saja tidak menjamin eMMC aman**. Sistem custom dapat dijalankan dari SD tanpa memasang ROM ke eMMC jika rantai boot benar-benar mengambil bootloader/kernel/root dari SD dan tidak ada skrip installer yang menulis ke eMMC. Perlu dibuktikan pada unit V2.

Amlogic GX BootROM memiliki urutan boot perangkat keras. Untuk keluarga S905X, dokumentasi U-Boot mencantumkan eMMC sebelum SD pada urutan default; ada metode memaksa boot SD melalui pin/strap atau HDMI recovery dongle yang didukung SoC. Jadi pada jalur biasa, U-Boot di eMMC bisa tetap menjadi tahap pertama, lalu ia memuat sistem dari SD. Opsi dan wiring persisnya bergantung pada board. [Amlogic SoC Boot Flow — U-Boot](https://docs.u-boot-project.org/en/stable/board/amlogic/boot-flow.html)

## Tentang `UBOOT.USB`

File `/Users/macpro/Downloads/UBOOT.USB` yang tersedia berukuran sekitar 1.1 MiB dan terdeteksi sebagai data biner. Panduan Amlogic menyebut `DDR.USB` dan `UBOOT.USB` sebagai komponen SPL/TPL yang dipakai untuk memuat U-Boot melalui protokol USB dalam alur inisialisasi/flash. Nama tersebut **tidak membuktikan file itu dapat diletakkan di root SD lalu dieksekusi oleh box**. [Panduan Amlogic USB Update Tool](https://raw.githubusercontent.com/osmc/aml-flash-tool/master/docs/Amlogic_update_usb_tool_user_guide.pdf)

Untuk saat ini, perlakukan file itu sebagai **bahan recovery/bahan analisis**:

```bash
shasum -a 256 /Users/macpro/Downloads/UBOOT.USB
file /Users/macpro/Downloads/UBOOT.USB
```

Jangan flash `UBOOT.USB` ke eMMC atau kartu SD sebelum format, target board, offset, dan metode boot-nya dipastikan dari paket firmware serta dokumentasi yang cocok dengan revisi V2. Salah bootloader/DDR setup dapat membuat box tidak bisa boot normal.

## Skema yang dituju

```text
BootROM / mekanisme boot SD yang terverifikasi
                  │
                  ▼
       U-Boot yang cocok dengan B860H V2
                  │ membaca dari SD
                  ▼
       kernel + DTB + ramdisk + system
                  │
                  ▼
       Android custom berjalan dari SD

eMMC stock: hanya dibaca; tidak di-flash atau dipakai sebagai target instalasi
```

Jika BootROM memulai U-Boot dari eMMC, eMMC tetap **dibaca** saat start. “Tidak menulis eMMC” bukan berarti eMMC tidak disentuh sama sekali.

## Jalur uji yang disarankan

### 1. Simpan jalur kembali ke stock

- Catat identitas board/revisi, nomor build Android, kernel, dan layout partisi.
- Buat dump eMMC penuh dan dump partisi bootloader/boot/dtb/recovery yang relevan ke media eksternal; hitung SHA-256 dan simpan salinan kedua.
- Simpan firmware resmi yang benar-benar cocok dengan B860H V2 dan pastikan host dapat mendeteksi mode recovery/burn sebelum percobaan.
- Foto PCB dan catat UART 3.3 V jika tersedia. UART berguna untuk melihat apakah U-Boot memuat SD atau jatuh kembali ke eMMC.

USB Burning Tool atau pemulihan firmware dapat **menulis ulang eMMC**. Itu jalur pemulihan jika sesuatu rusak, bukan bagian dari jalur no-write.

### 2. Buktikan metode boot SD tanpa mengubah eMMC

Pilih hanya metode yang sudah diverifikasi untuk PCB V2 milikmu:

1. **BootROM dipaksa memilih SD** melalui mekanisme hardware yang didukung board/SoC, misalnya HDMI DDC recovery dongle yang memaksa `boot@SDC`, jika terbukti dikenali unit tersebut.
2. **U-Boot stock membaca SD**, misalnya lewat recovery/toothpick dan skrip boot SD yang memang sudah ada di firmware V2. Dalam jalur ini U-Boot dapat berasal dari eMMC, tetapi tidak perlu di-flash ulang.
3. Jika metode di atas tidak terbukti, berhenti di serial/UART dan kumpulkan log. Jangan mencoba bootloader generik atau instruksi untuk V5/HG680P.

Dokumentasi BootROM Amlogic menjelaskan bahwa USB protocol dapat memuat U-Boot ke RAM untuk recovery, tetapi itu berbeda dari menaruh berkas `UBOOT.USB` di SD. [U-Boot Amlogic boot flow](https://docs.u-boot-project.org/en/stable/board/amlogic/boot-flow.html)

**Syarat lulus:** serial log menunjukkan U-Boot/kernel/DTB/root filesystem berasal dari kartu; Android stock pada eMMC tetap dapat diboot setelah kartu dilepas.

### 3. Siapkan kartu terpisah untuk uji

- Gunakan microSD khusus yang dapat dihapus; jangan gunakan satu-satunya salinan backup.
- Simpan salinan image mentah kartu sebelum setiap eksperimen.
- Periksa skrip boot dan initramfs sebelum boot. Tolak image yang menjalankan perintah seperti `amlmmc`, `mmc write`, `dd of=/dev/mmcblk...`, `nand`, `install-to-emmc`, atau installer otomatis.
- Pastikan root filesystem berada di SD, bukan di eMMC. Periksa `root=`, `fstab`, init scripts, dan symlink `/dev/block/by-name`.
- Mulai dari image Linux/diagnostik minimal yang dapat memberi serial log. Setelah sumber boot dan akses storage jelas, baru uji image Android.

Jangan menjalankan `armbian-install`, `install-aml.sh`, USB Burning Tool, `dd` ke device block, atau skrip “install to internal” selama pengujian no-write.

### 4. Verifikasi setelah boot

Di sistem uji, identifikasi storage berdasarkan ukuran, model, dan hubungan device tree. Jangan mengasumsikan `mmcblk0` adalah eMMC atau SD:

```sh
cat /proc/cmdline
lsblk -o NAME,TYPE,SIZE,RO,MOUNTPOINTS,MODEL
cat /proc/mounts
for d in /sys/block/mmcblk*; do
  echo "== $d =="
  cat "$d/device/type" 2>/dev/null
  cat "$d/device/name" 2>/dev/null
done
```

Pastikan root filesystem dan partisi Android yang sedang dipakai berasal dari SD. Jangan mount partisi eMMC read-write. Untuk verifikasi kuat, buat hash/dump eMMC sebelum dan sesudah dari lingkungan recovery yang hanya membaca storage, lalu bandingkan. Dump harus ditulis ke media eksternal yang sudah dipastikan bukan eMMC.

Mengatur block device menjadi read-only dari Linux dapat menjadi lapisan tambahan setelah device eMMC teridentifikasi, tetapi tidak mencegah BootROM/U-Boot menulis sebelum kernel hidup, tidak mengalahkan akses tulis langsung dari firmware lain, dan dapat mengganggu operasi yang belum dipahami. Jangan aktifkan secara membabi buta pada perangkat block yang belum diidentifikasi.

## Model perlindungan

| Lapisan | Apa yang dilindungi | Batasnya |
|---|---|---|
| Boot dari SD | Image custom di SD terpisah | Tidak otomatis mencegah skrip/OS menulis eMMC |
| Tidak menjalankan installer internal | Menghindari operasi instalasi umum | Kernel/root masih dapat menulis block device jika diakses |
| Mount eMMC read-only / block device read-only | Mengurangi risiko write dari Linux yang berjalan | Tidak mengontrol BootROM/U-Boot atau akses sebelum mode RO dipasang |
| Isolasi fisik eMMC | Perlindungan paling kuat terhadap write | B860H memakai eMMC tersolder; pembongkaran/pemutusan bukan langkah awal yang disarankan |

Untuk proyek ini, definisikan “no-write” sebagai **tidak mem-flash bootloader/ROM ke eMMC, tidak menginstal sistem ke eMMC, dan membuktikan tidak ada perubahan pada eMMC selama sesi boot SD**. SD boot saja tidak cukup sebagai bukti.

## Jika box tidak boot

1. Matikan daya, keluarkan SD, lalu coba boot Android stock.
2. Jika stock boot, jangan mengubah eMMC; periksa ulang SD image, DTB, dan UART log.
3. Jika stock tidak boot, masuk recovery/burn mode dengan prosedur PCB V2 yang sudah diketahui dan pulihkan firmware V2 dari backup.
4. Jika USB Burning Tool mendeteksi SoC tetapi firmware gagal, jangan mencoba bootloader acak; rekam pesan error dan simpan log UART.
5. Jika mode burn tidak terdeteksi, hentikan percobaan flash. Verifikasi kabel/port/driver dan cari test point berdasarkan foto board V2 yang tepat.

## Keputusan untuk `UBOOT.USB`

**Belum dipakai pada prosedur boot SD.** Pertahankan file asli dan checksum-nya. Sebelum memanfaatkannya, identifikasi asal paket, versi firmware, keluarga/revisi board, dan apakah itu payload untuk USB Burning Tool atau binary boot yang memang diformat untuk SD. Nama file dan ekstensi saja tidak cukup.

## Referensi

- [Amlogic SoC Boot Flow — U-Boot](https://docs.u-boot-project.org/en/stable/board/amlogic/boot-flow.html)
- [Panduan Amlogic USB Update Tool](https://raw.githubusercontent.com/osmc/aml-flash-tool/master/docs/Amlogic_update_usb_tool_user_guide.pdf)
- [Proyek lokal Armbian B860H V2](../armbian-b860h/README.md) — catatan eksperimen lokal; cek kembali setiap instruksi sebelum menerapkannya pada jalur rescue.
