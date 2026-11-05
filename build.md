# Panduan build Android untuk B860H V2

Panduan ini menyiapkan **mini PC Debian 13 (Trixie)** sebagai mesin build untuk **ZTE B860H V2 / S905X / GXL**. Target pengembangan adalah Android 9 terlebih dulu. Build host bisa disiapkan sekarang; build ROM yang bisa boot belum dapat dimulai sampai source tree khusus B860H V2, branch Android, kernel, vendor blobs, dan layout partisi sudah dikunci di [PLAN.md](PLAN.md).

## Status penting sebelum mulai

Folder ini belum berisi source tree Android, `device/zte/b860hv2`, manifest lokal final, atau vendor blobs. Karena itu, perintah `lunch lineage_b860hv2-userdebug` **belum akan berhasil sekarang**. Jangan sync atau compile branch secara acak: common tree GXL menyebut S905X dan kernel Amlogic 4.9 berbasis Android 9, tetapi itu bukan target ROM B860H yang siap dibangun.

Android 9 dipilih sebagai titik awal karena kesesuaian kernel GXL yang terdokumentasi. Android 10 tetap opsi lanjutan setelah dependency Android 10 yang cocok tersedia. [LineageOS Amlogic GX common tree](https://github.com/LineageOS/android_device_amlogic_gx-common)

## 1. Periksa kemampuan mini PC

Jalankan di Debian Trixie:

```bash
uname -m
nproc
free -h
df -h /
lsblk
```

Host harus **x86_64**. Sediakan disk kerja khusus dengan **setidaknya 300 GB kosong; 400 GB lebih nyaman**, idealnya SSD. Untuk build pertama, **32 GB RAM disarankan**; 16 GB bisa dicoba dengan swap besar dan parallelism rendah, tetapi proses akan lambat atau dapat kehabisan memori. AOSP terbaru mencantumkan 400 GB dan 64 GB RAM sebagai kebutuhan workstation saat ini; tree LineageOS Android 9 yang lebih kecil mungkin muat di bawah itu, tetapi ruang lega mencegah build berhenti di tengah jalan. [Persyaratan build AOSP](https://source.android.com/docs/setup/start/requirements)

Pastikan ruang ada di volume yang akan dipasang ke container, bukan hanya partisi root Debian. Contoh di bawah memakai `/srv/android`; sesuaikan dengan mount SSD milikmu.

## 2. Siapkan Docker di Debian

```bash
sudo apt update
sudo apt install -y docker.io ca-certificates
sudo systemctl enable --now docker
sudo docker version
sudo mkdir -p /srv/android
sudo chmod 0777 /srv/android
```

Mode permission di atas memudahkan container menulis ke direktori build khusus. Jangan gunakan direktori ini untuk file pribadi.

## 3. Buat container Ubuntu 20.04

Untuk source Android lama, Ubuntu container mengurangi perbedaan paket/tool host dari Debian Trixie. Build dilakukan sebagai root **di dalam container**; file akan tersimpan di `/srv/android` pada host.

```bash
sudo docker run -it \
  --name b860h-v2-build \
  --hostname b860h-builder \
  --mount type=bind,src=/srv/android,dst=/work \
  --workdir /work \
  ubuntu:20.04 bash
```

Jika container sudah pernah dibuat dan terminal terputus:

```bash
sudo docker start -ai b860h-v2-build
```

## 4. Pasang paket build di container

Jalankan di shell Ubuntu container:

```bash
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  bc bison build-essential ccache curl flex g++-multilib gcc-multilib \
  git git-lfs gnupg gperf imagemagick lib32readline-dev lib32z1-dev \
  libelf-dev liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev \
  libssl-dev libxml2 libxml2-utils lzop openjdk-8-jdk pngcrush \
  python-is-python3 rsync schedtool squashfs-tools unzip xsltproc zip \
  zlib1g-dev

git lfs install
java -version
```

Jika paket `openjdk-8-jdk` tidak tersedia pada mirror Ubuntu yang dipakai, jangan menggantinya dengan Java acak. Beberapa branch Android membawa JDK prebuilt; ikuti kebutuhan branch yang dipilih dan perbaiki sumber paket container lebih dulu.

## 5. Pasang `repo` dan siapkan cache

```bash
mkdir -p /root/bin /root/.cache/ccache
curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo \
  -o /root/bin/repo
chmod 0755 /root/bin/repo
export PATH="/root/bin:$PATH"

export USE_CCACHE=1
export CCACHE_DIR=/root/.cache/ccache
ccache -M 50G
ccache -o compression=true
```

Tambahkan `PATH`, `USE_CCACHE`, dan `CCACHE_DIR` ke `/root/.bashrc` bila ingin tetap tersedia setelah container direstart.

## 6. Kunci source sebelum `repo sync`

Sebelum menarik ratusan gigabyte source, pastikan jawaban ini sudah ada:

- branch manifest yang menyediakan framework Android 9;
- branch common tree GXL dan dependency-nya yang benar-benar ada;
- kernel GXL yang sesuai dengan branch tersebut;
- device tree **B860H V2** (atau rencana porting dari DTS yang sudah dibuktikan cocok);
- vendor blobs/HAL yang serasi dengan framework dan kernel;
- nama product/codename, jenis partisi, dan metode packaging image;
- dukungan UDC/USB gadget pada kernel target.

Jangan gunakan konfigurasi B860H V5/S905X2 atau HG680P sebagai pengganti. Untuk B860H V2, kernel Linux/Armbian dan DTS Linux dapat membantu memahami hardware, tetapi tidak otomatis menjadi kernel/device tree Android.

Setelah source dan branch tersebut dicatat di [PLAN.md](PLAN.md), gunakan template berikut dengan URL dan branch yang sudah diverifikasi:

```bash
mkdir -p /work/lineage-b860h-v2
cd /work/lineage-b860h-v2

# Isi dari manifest/branch yang sudah diverifikasi untuk target Android 9.
repo init -u <MANIFEST_URL> -b <ANDROID_9_BRANCH>
repo sync -c --no-clone-bundle --no-tags -j2
```

`<MANIFEST_URL>` dan `<ANDROID_9_BRANCH>` sengaja belum diisi: saat dokumen ini dibuat belum ada kombinasi source yang terverifikasi untuk product B860H V2. Jangan menyalin URL/branch contoh dari device Amlogic lain tanpa mengaudit dependency.

## 7. Tambahkan product B860H V2

Setelah device tree dan vendor tree tersedia, pastikan product terdaftar di `AndroidProducts.mk`, `BoardConfig.mk` menunjuk kernel/DTB/partisi yang benar, dan dependency manifest memiliki revisi yang nyata. Kemudian:

```bash
cd /work/lineage-b860h-v2
source build/envsetup.sh
breakfast b860hv2
```

Jika `breakfast` belum mengenali product, berhenti dan lengkapi device tree; jangan memilih target device lain untuk menghasilkan image yang akan di-flash ke B860H.

Ketika target dikenali dan konfigurasi build siap:

```bash
lunch lineage_b860hv2-userdebug
mka bacon -j2
```

`-j2` adalah nilai awal yang konservatif untuk mini PC. Naikkan bertahap sambil memantau `free -h`; compile dapat gagal jika RAM habis. Nama lunch dan target paket di atas adalah **rencana nama product**, bukan target yang sudah ada di repo saat ini.

Hasil biasanya berada di:

```text
out/target/product/b860hv2/
```

Sebelum menyentuh STB, arsipkan log build, checksum setiap image, source manifest (`repo manifest -r`), config kernel, DTB, dan cara pemulihan. Tahap boot/flash harus mengikuti Fase 1 dan Fase 4–5 di [PLAN.md](PLAN.md); jangan flash eMMC sebelum backup dan jalur unbrick terbukti.

## 8. Build ulang dan bersihkan

Container dapat dihentikan tanpa kehilangan source/cache yang ada di volume `/srv/android`:

```bash
exit
sudo docker start -ai b860h-v2-build
```

Hapus output build jika perlu membebaskan ruang (pastikan folder yang akan dihapus benar):

```bash
cd /work/lineage-b860h-v2
rm -rf out
```

## Urutan bring-up setelah image tersedia

1. Boot tanpa menimpa eMMC bila metode SD/USB terbukti aman.
2. Tangkap serial log; pastikan kernel, DTB, partisi, dan HDMI benar.
3. Uji Ethernet/Wi-Fi, remote/input, audio, dan HEVC playback.
4. Di kernel Android, cek `/sys/class/udc`, role port, configfs/FunctionFS, lalu enumerasi gadget uji dengan komputer host.
5. Setelah transfer bulk stabil, integrasikan AOA RC-N dan tampilkan clean stream ke HDMI.

USB OTG/ADB atau UDC terdaftar saja belum membuktikan AOA DJI. Prosedur validasi gadget ada di [PLAN.md](PLAN.md).

## Referensi

- [AOSP: persyaratan workstation dan paket build](https://source.android.com/docs/setup/start/requirements)
- [AOSP: alur sync dan build](https://source.android.com/docs/setup/start)
- [LineageOS Amlogic GX common tree](https://github.com/LineageOS/android_device_amlogic_gx-common)
- [Docker Engine di Debian](https://docs.docker.com/engine/install/debian/)
