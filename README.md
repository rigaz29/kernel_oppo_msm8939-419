# Forward-port kernel OPPO A37 — dari 3.10.108

Rencana kerja. Ditulis 5 September 2026.

Semua angka hasil pemeriksaan langsung terhadap kode atau pengukuran di
perangkat, bukan perkiraan. Sumbernya dicantumkan supaya bisa diperiksa ulang.

---

## 0. Vonis

**Sasarannya 4.19, bukan 4.4.** Dan jalurnya spesifik:

```
LineageOS/android_kernel_xiaomi_msm8937   cabang lineage-23.2
Linux 4.19.325 — commit terakhir 16 Agustus 2026
```

Bukti yang menentukan, semuanya diverifikasi langsung di pohon itu:

```
drivers/gpu/msm/adreno-gpulist.h : ADRENO_REV(ADRENO_REV_A306, 3, 0, 6, 0)
drivers/clk/msm/                 : clock-cpu-8939.c
drivers/video/fbdev/msm/         : 106 berkas
techpack/                        : camera-legacy, camera-legacy-m, audio, display
cabang                           : lineage-23.2   <- versi ROM kita persis
```

`ADRENO_REV_A306` adalah GPU A37 tepat. `clock-cpu-8939.c` adalah clock CPU
keluarga SoC kita — pohon kernel kita sendiri literal bernama `msm8939`.

**Kenapa yang lebih baru justru lebih mudah.** Pekerjaannya bukan "memindahkan
3.10 maju sembilan tahun". Pekerjaannya *"menambahkan selisih msm8916 ke pohon
msm8937 yang sudah boot di ponsel sungguhan"*. Seluruh adaptasi API sembilan
tahun itu sudah dikerjakan orang lain dan sudah terbukti. msm8917/8937 adalah
adik-kakak langsung msm8916: sama-sama Cortex-A53 empat inti, MDSS segenerasi,
Adreno 3xx, PMIC pm8916/pm8937 sekeluarga.

Bandingkan dengan 4.4: di sana kita mulai dari pohon yang tidak mengenal SoC
ini sama sekali, lalu mengadaptasi sendiri lima tahun perubahan API untuk
sejuta baris. **Lebih tua bukan berarti lebih dekat.**

---

## 1. Riwayat penilaian ini

Dokumen ini awalnya ditulis untuk 4.4 dan menyimpulkan 4.9 lebih baik. Itu
salah karena satu kelalaian metode: yang disurvei hanya pohon CAF mentah,
**kernel perangkat LineageOS tidak pernah diperiksa**. Begitu diperiksa,
jawabannya berubah total.

Pelajaran yang layak dicatat: untuk pertanyaan "versi mana yang paling mudah",
sumber terbaik bukan rilis vendor, melainkan **pohon yang sudah boot di
perangkat nyata dengan userspace yang sama**.

---

## 2. Titik berangkat

```
versi          : 3.10.108
riwayat git    : 445.894 commit, tertua Linux-2.6.12-rc2 (2005)
                 leluhur mainline ADA (commit "Linux 3.10.108")
                 270 merge CAF, termasuk LA.BR.1.2.9.1-02310-8x16.0
remote         : gh  = rigaz29/kernel_oppo_msm8939
DTS perangkat  : arch/arm/boot/dts/qcom/msm8916-mtp-15399.dts (852 byte)
                 board-id <8 0 15399>, dibangun lewat CONFIG_MACH_15399
                 arch/arm64/boot/dts/qcom -> symlink ke arch/arm/boot/dts/qcom
defconfig      : lineageos_a37f_defconfig, 601 opsi `=y`, 0 modul
SoC            : MSM8916, Cortex-A53 4x1,2 GHz, Adreno 306, RAM 2 GB
```

**Koreksi 5 September 2026.** Versi sebelumnya dokumen ini menulis "50 commit,
impor vendor ter-squash, TIDAK ADA leluhur CAF" dan menyimpulkan tidak ada yang
bisa di-`rebase`. **Itu salah.** Penyebabnya `git log` dijalankan lewat pembungkus
yang memfilter keluaran, lalu barisnya dihitung dengan `wc -l`; angka yang benar
diperoleh dengan `git rev-list --count`.

Konsekuensinya besar: pohon ini punya riwayat penuh mainline dan CAF, sehingga
**merge incremental per rilis mayor benar-benar mungkin** — pendekatan yang
dipakai pengembang kernel profesional dan yang keunggulannya dibahas di §8.4.

### Volume kode di pohon lama (batas atas)

| Jalur | Baris | Berkas |
|---|---:|---:|
| `drivers/media/platform/msm` (kamera) | 229.333 | 404 |
| `drivers/power` | 118.383 | 134 |
| `drivers/video/msm` (MDSS) | 104.827 | 107 |
| `sound/soc/msm` | 99.736 | 65 |
| `drivers/input/touchscreen` | 87.410 | 111 |
| `drivers/platform/msm` | 83.072 | 113 |
| `drivers/soc/qcom` | 62.206 | 115 |
| `drivers/gpu/msm` (KGSL) | 45.405 | 61 |
| `drivers/thermal` | 23.815 | 26 |
| `drivers/crypto/msm` | 20.241 | 18 |
| DTS `arch/arm/boot/dts/qcom` | 179.285 | 800 |
| **Jumlah kasar** | **~1.050.000** | **~1.950** |

Ini batas atas dan **bukan** ukuran pekerjaan sebenarnya. Sebagian besar sudah
tersedia dalam bentuk 4.19 di pohon sasaran; yang benar-benar dipindah hanya
selisih 8916. Penyaringan nyata dilakukan di Fase 0.

### Yang hilang dari 3.10, dan itu alasan proyek ini ada

Diperiksa langsung:

```
TIDAK ADA    eBPF            (kernel/bpf tidak ada, sys_bpf tidak terdaftar)
TIDAK ADA    cgroup v2       (CGROUP2_SUPER_MAGIC nol kemunculan)
TIDAK ADA    overlayfs
TIDAK ADA    incremental-fs
ADA          cgroup v1
ADA          PSI             <- backport kita sendiri
ADA          workingset      <- backport kita sendiri
```

Tiga yang pertama adalah sebab **60 patch BPF-less userspace** harus ada di ROM
ini. Itu imbalan utama proyek ini.

---

## 3. Survei sumber

### 3.1 Sasaran terpilih — LineageOS msm8937 @ `lineage-23.2` (4.19.325)

| Yang dibutuhkan | Status di pohon itu |
|---|---|
| Adreno 306 | `ADRENO_REV_A306` terdaftar eksplisit |
| Clock CPU keluarga 8939 | `clock-cpu-8939.c` |
| MDSS | 106 berkas |
| Kamera lawas | `techpack/camera-legacy`, `camera-legacy-m` |
| Audio | `techpack/audio`, `audio-legacy` |
| Integrasi Android 16 | **sudah** — cabangnya `lineage-23.2` |
| Perawatan | commit terakhir 16 Agustus 2026 |
| Keamanan | CIP merawat 4.19 |

Struktur akar pohonnya memakai konvensi CAF 4.19: audio, kamera, dan display
dipindah ke `techpack/` sebagai paket terpisah. Ada pula
`techpack/xiaomi-msm8937` dan `techpack/xiaomi-sdm439` sebagai contoh techpack
khusus perangkat — pola yang bisa kita tiru untuk A37.

Yang **belum** ada dan harus dikerjakan sendiri:

```
clock-gcc-8916.c        (pohon punya clock-gcc-8952.c dan 8953, bukan 8916)
DTS msm8916 + pm8916 + papan msm8916-mtp-15399
bagian 8916 di MDSS     (versi MDP, DSI PHY, panel A37)
codec WCD msm8x16       (struktur audio berbeda: techpack/audio)
prima                   (WLAN in-kernel kita)
```

### 3.2 CAF `msm-4.9` — `kernel.lnx.4.9.r11-rel` (4.9.250)

Kode SoC paling dekat dari seluruh pohon CAF:

```
clock : clock-cpu-8939.c  clock-gcc-8909.c  clock-rpm-8909.c
        clock-gcc-8952.c  clock-gcc-8953.c
DTS   : msm8917 (51)  msm8937 (32)  msm8909 (31)  sdm439 (30)  msm8953 (59)
defconfig arm64 : msm8937_defconfig  msm8953_defconfig  sdm670  sdm845
```

`clock-gcc-8909.c` adalah yang terdekat dengan `clock-gcc-8916.c` yang kita
butuhkan — msm8909 praktis msm8916 dipangkas.

**Peringkat dua.** Kalah karena tanpa integrasi LineageOS, tanpa perawatan
aktif, tanpa CIP, dan sudah EOL Januari 2023. Tetapi **pohon ini tetap wajib
di-clone sebagai donor**: `clock-gcc-8909.c` dan DTS msm8909-nya adalah rujukan
terbaik saat menulis `clock-gcc-8916.c` untuk 4.19.

### 3.3 CAF `msm-4.4` — `kernel.lnx.4.4.r40-rel` (4.4.250)

Kerangka driver CAF lengkap (MDSS 120 berkas, KGSL 77 termasuk
`adreno_a3xx.c`, `soc/qcom` 143), tetapi dukungan SoC salah keluarga:

```
clock : 8996, 8998 saja
DTS   : apq8016 apq8096 apq8098 msm8916 msm8996 msm8998 sdm455 sdm6xx sda6xx
```

`msm8916` di daftar itu menyesatkan. Hanya lima berkas, dan penamaannya
**mainline**, bukan CAF:

```
msm8916.dtsi  msm8916-pins.dtsi  msm8916-mtp.dts  msm8916-mtp.dtsi  pm8916.dtsi
```

CAF memakai `msm8916-pinctrl.dtsi`; mainline memakai `msm8916-pins.dtsi`. BSP
CAF 3.10 kita punya **800** berkas DTS, bukan lima. Yang ada di msm-4.4 adalah
dukungan upstream tingkat Dragonboard 410c — tanpa MDSS 8916, tanpa KGSL 8916,
tanpa kamera, tanpa audio WCD.

**Ditolak.** Satu-satunya keunggulannya CIP, dan 4.19 juga punya itu.

### 3.4 CAF `msm-3.18` — `kernel.lnx.3.18.r34-rel`

Punya `clock-gcc-8916.c`, `clock-rpm-8916.c`, `clock-cpu-8939.c` — menjanjikan
di permukaan. Tetapi DTS-nya hanya menyisakan lima berkas 8916/8939
(`msm8916-regulator.dtsi`, `msm8939-common.dtsi`, `msm8939-cpu.dtsi`,
`msm8916-mdss-panels.dtsi`, `msm8939-mdss-panels.dtsi`) — sisa yang dipakai
turunan msm8937 — dan tidak ada defconfig 8916/8939.

**Ditolak sebagai sasaran**, tetapi `clock-gcc-8916.c`-nya adalah donor paling
tepat untuk pekerjaan clock di Fase 1.

### 3.5 CAF `msm-4.19` mentah

Menggoda karena CIP merawat 4.19. Tetapi SoC sasarannya generasi 2019 ke atas:
`sm8150` (msmnile), `sm6150` (talos/trinket), `kona`, `lito`, `atoll`,
`bengal`. **Tidak ada keluarga 8916/8917/8937.**

Kalah oleh pohon LineageOS msm8937 yang berbasis 4.19 yang sama tetapi sudah
membawa SoC sekeluarga.

### 3.6 CIP 4.4 SLTS

```
rilis terbaru : linux-cip-4.4.302-cip114        19 Agustus 2026
varian RT     : linux-cip-4.4.302-cip113-rt62   30 Juli 2026
versi SLTS    : 4.4  4.19  5.10  6.1  6.12
```

Ini membatalkan anggapan umum bahwa 4.4 mati sejak Februari 2022 — yang mati
LTS biasa. Yang relevan bagi kita: **CIP juga merawat 4.19**, jadi sasaran
terpilih tetap mendapat perbaikan keamanan berkelanjutan.

### 3.7 Android common `android-4.4`

Seluruh cabangnya bertanda `deprecated/` (`android-4.4`, `-o`, `-o-mr1`,
`-p`, `-llvm`, `-4.4.y`, dan varian `-release`). Yang aktif sekarang
`android14-6.1`, `android15-6.6`, `android16-6.12`, `android17-6.18`.

Patch Android yang kita butuhkan (binder, ashmem, ION, sync) sudah menyatu di
pohon CAF dan LineageOS, jadi cabang ini tidak diperlukan.

### 3.8 Ringkasan peringkat

| Peringkat | Sasaran | Keluarga SoC | Integrasi LOS | CIP | Dirawat |
|---|---|---|---|---|---|
| **1** | **LOS msm8937 4.19.325** | **A306, clock-8939** | **lineage-23.2** | **ya** | **Agu 2026** |
| 2 | CAF msm-4.9 | 8909/8917/8937 | tidak | tidak | tidak |
| 3 | CAF msm-4.4 | tidak ada | tidak | ya | tidak |
| 4 | CAF msm-3.18 | sebagian | tidak | tidak | tidak |
| 5 | mainline 6.x | penuh, tapi non-CAF | tumpukan lain | ya | ya |

---

## 4. Analisis kesenjangan

### 4.1 API internal (3.10 → 4.19, sembilan tahun)

Ini yang **tidak** perlu kita kerjakan, karena pohon sasaran sudah melewatinya.
Didaftar hanya supaya jelas apa yang kita hindari:

| Perubahan | Rentang |
|---|---|
| `clk` framework: `struct clk` buram, `clk_hw` wajib | 3.15–4.0 |
| `fbdev` CAF pindah ke `drivers/video/fbdev/msm` | 3.18 |
| V4L2 `media_entity` refactor, `vb2` | 3.16–4.0 |
| IOMMU `iommu_group`, `arm-smmu` ditulis ulang | 3.16–4.2 |
| ASoC `snd_soc_codec` → `component` | 4.4–4.18 |
| `setup_timer` → `timer_setup` | 4.15 |
| `refcount_t`, `access_ok` argumen berubah | 4.11–4.19 |
| audio/kamera/display pindah ke `techpack/` | CAF 4.19 |

**Yang tetap kena kita** hanyalah kode khas 8916 yang kita bawa sendiri, dan
itu sebagian besar driver kecil plus DTS.

### 4.2 ABI ke userspace — risiko yang paling sering diremehkan

433 berkas proprietary dipakai A37:

```
kamera : 66 blob     <- risiko tertinggi
radio  : 18
gpu    : 14          <- driver userspace Adreno, bicara ke KGSL lewat ioctl
audio  :  6
```

Blob ini dibangun untuk kernel 3.10.

- **KGSL ioctl** — risiko sedang. Diringankan karena A306 didukung resmi di
  pohon sasaran; kalau Xiaomi msm8917 memakai blob Adreno segenerasi, ABI-nya
  kemungkinan besar cocok.
- **MDSS fb ioctl** (`MSMFB_*`) — dipakai gralloc/hwcomposer, risiko sedang.
- **Kamera** — `msm_cam` ioctl dan subdev V4L2 berubah banyak. **Risiko
  tertinggi dan berpotensi menghentikan proyek.** Kehadiran
  `techpack/camera-legacy` memberi harapan, tetapi belum diverifikasi.
- **Audio** — nama kontrol mixer ALSA harus dipertahankan persis. Kita sudah
  tahu betapa sensitifnya ini dari pekerjaan `RX1 Digital Volume`.

### 4.3 Integrasi Android

Yang jadi lebih mudah:

- **60 patch BPF-less bisa dipensiunkan** — eBPF tersedia.
- cgroup v2 dan overlayfs tersedia.
- **PSI sudah masuk mainline di 4.20**, jadi di 4.19 tinggal cherry-pick satu
  rilis — bukan backport tangan seperti yang kita kerjakan di 3.10.
- Pohon sasaran sudah bercabang `lineage-23.2`, jadi seluruh perekat build
  LineageOS sudah beres.

Yang jadi pekerjaan baru:

- `sepolicy` perlu ditinjau ulang.
- Seluruh setelan sysfs yang kita hafal harus divalidasi ulang:
  `default_pwrlevel`, `process_reclaim`, `RX1 Digital Volume`,
  `current_quality` hwrng.

---

## 5. Kriteria lulus tiap fase

Satu ukuran objektif per fase. Tidak lanjut sebelum terpenuhi.

| Fase | Kriteria lulus |
|---|---|
| 0 | Pohon sasaran terbangun apa adanya untuk msm8937 |
| 1 | Konsol serial/ramoops hidup di A37, `console_init` tercapai |
| 2 | `adb shell` jalan, `/data` termount |
| 3 | Bootanimation terlihat |
| 4 | `usesDeviceComposition=true`, tanpa fallback GPU |
| 5 | Audio, WiFi, sensor, telepon berfungsi |
| 6 | LOS 23 boot penuh, 60 patch BPF-less dicabut, PSI aktif |
| 7 | Stabil 72 jam, tidak ada regresi vs 3.10 |

---

## 6. Rencana kerja

### Fase 0 — Persiapan (20–40 jam)

1. Clone pohon sasaran, bangun apa adanya untuk msm8937. Ini memastikan
   toolchain benar **sebelum** menyentuh apa pun.
2. Clone tiga donor: CAF msm-4.9 (`clock-gcc-8909.c`, DTS msm8909),
   CAF msm-3.18 (`clock-gcc-8916.c`), dan pohon 3.10 kita sendiri.
3. **Saring DTS.** Dari 800 berkas di pohon lama, tentukan yang benar-benar
   dirujuk `msm8916-mtp-15399.dts` secara transitif. Perkiraan 60–100 berkas.
4. **Petakan 601 opsi `=y`** ke berkas sumber; tandai mana yang sudah ada di
   4.19 dan mana yang harus dipindah.

Keluaran: daftar pekerjaan terukur. **Fase ini yang menentukan apakah sisa
rencana realistis.** Murah, jadi tidak ada alasan melewatinya.

### Fase 1 — Boot ke konsol (40–80 jam)

1. DTS SoC msm8916 minimal: CPU, memori, timer, PSCI, GIC.
2. `clock-gcc-8916.c` — tulis untuk 4.19 dengan `clock-gcc-8909.c` (msm-4.9)
   dan `clock-gcc-8916.c` (msm-3.18) sebagai rujukan berdampingan.
3. `clock-cpu-8939.c` sudah ada di pohon sasaran — verifikasi cocok.
4. pinctrl msm8916, adaptasi dari msm8917 yang sudah ada.
5. UART + `earlycon`.

Titik gagal paling umum: clock. Kalau GCC salah, tidak ada yang hidup dan tidak
ada log. **Siapkan `earlycon` dan ramoops sebelum apa pun** — cmdline kita
sudah mengonfigurasi ramoops, manfaatkan.

### Fase 2 — Penyimpanan dan init (30–60 jam)

`spmi-pmic-arb` + `pm8916` regulator → `sdhci-msm` → USB gadget + ADB → boot
ramdisk LOS.

Sejak `adb shell` menyala, seluruh alat diagnosis yang kita pakai sepanjang
proyek ini bisa dipakai lagi.

### Fase 3 — Tampilan (80–150 jam) — blok terbesar

MDSS 4.19 sudah ada (106 berkas). Ini **adaptasi**, bukan penulisan ulang:
pindahkan bagian khusus 8916 (versi MDP, konfigurasi DSI PHY, panel A37).

Paling mungkin meleset dari perkiraan. Pengalaman kita dengan
`mdss_dsi_event` menunjukkan subsistem ini penuh kejutan.

### Fase 4 — GPU (20–40 jam)

Paling murah dari semua fase karena `ADRENO_REV_A306` sudah terdaftar.
Pekerjaannya: DT KGSL, clock GPU, IOMMU. Uji paling awal: `kgsl-3d0` muncul di
sysfs dan `gpuclk` terbaca.

**Uji ioctl blob Adreno di sini, sebelum investasi besar berikutnya.**

### Fase 5 — Sisanya (120–250 jam)

Berurutan menurut risiko naik: audio (`techpack/audio`, codec WCD msm8x16) →
WiFi (`prima`) → sensor → BT → modem → **kamera terakhir**.

Kamera adalah kandidat terkuat untuk dinyatakan gagal. Rencanakan kemungkinan
A37 kehilangan kamera.

### Fase 6 — Integrasi Android (30–60 jam)

Jauh lebih ringan daripada rencana 4.4, karena cabang `lineage-23.2` sudah ada:

1. Cabut 60 patch BPF-less, aktifkan eBPF di userspace.
2. Cherry-pick PSI dari 4.20.
3. sepolicy, fstab, `init.target.rc`.
4. Validasi ulang seluruh setelan sysfs kita.

### Fase 7 — Stabilisasi (60+ jam)

Ulangi seluruh pengukuran yang sudah jadi patokan proyek ini dan bandingkan
langsung dengan angka 3.10:

```
FPS timestats     : baseline 41,3 fps (adegan ringan), biaya GPU 22,1 ms/frame
PSI memori idle   : 0,18%
lmkd kills        : 0
allocstall idle   : 16 per 54 menit
zram              : lz4, rasio 2,75x, overhead zsmalloc 8%
suhu beban penuh  : pm8916_tz 64C, tanpa throttling
```

**Total kasar: 400–740 jam.** Lebih rendah daripada perkiraan 530–1.100 jam
untuk jalur 4.4, karena Fase 1, 2, 4, dan 6 semuanya lebih murah bila memulai
dari pohon yang sudah boot.

---

## 7. Register risiko

| Risiko | Peluang | Dampak | Mitigasi |
|---|---|---|---|
| Blob kamera tidak kompatibel | tinggi | fitur hilang permanen | Terima kemungkinan A37 tanpa kamera; kerjakan paling akhir |
| MDSS lebih sulit dari perkiraan | sedang | jadwal molor 2x | Fase 3 dianggarkan terbesar; gagal cepat lebih baik |
| Blob GPU menolak ioctl 4.19 | rendah–sedang | tidak ada akselerasi | Uji di Fase 4 sebelum investasi besar |
| Clock salah, tidak ada log | sedang | buntu di Fase 1 | `earlycon` + ramoops disiapkan lebih dulu |
| Pohon hulu berubah arah | rendah | rebase mahal | Pin ke commit tertentu, jangan ikuti `lineage-23.2` buta |
| Proyek ditinggalkan separuh | tinggi | 3.10 tetap dipakai | Tiap fase berdiri sendiri |

Mitigasi terpenting: **pohon 3.10 yang sekarang berfungsi tidak boleh
disentuh.** ROM yang jalan sekarang tetap jalan apa pun yang terjadi di sini.
Itu sebabnya proyek ini hidup di repo terpisah.

---

## 8. Alternatif

### 8.1 CAF msm-4.9

Kode SoC paling dekat, tetapi tanpa integrasi LineageOS, tanpa CIP, tanpa
perawatan. Masuk akal hanya bila pohon msm8937 ternyata bermasalah di Fase 0.
Tetap wajib di-clone sebagai donor.

### 8.2 Mainline 6.x lewat `msm8916-mainline`

Proyek `msm8916-mainline` dan postmarketOS menjalankan msm8916 di kernel 6.x
dengan DRM/KMS berfungsi dan GPU lewat freedreno. Ini jalur paling hidup secara
upstream.

Tetapi arahnya berbeda: menukar seluruh tumpukan HAL Android CAF dengan
tumpukan mainline. Blob kamera, audio, dan RIL kita tidak dipakai. Untuk
LineageOS 23 yang bergantung pada HAL vendor, ini bukan forward-port melainkan
proyek yang sama sekali lain.

Layak dipertimbangkan **kalau** tujuannya distribusi Linux, bukan Android.

### 8.3 Tetap di 3.10

Pilihan yang sah, dan sejauh ini terbukti. Proyek ini sudah membuktikan 3.10
bisa menjalankan Android 16 dengan 60 patch userspace, dan setiap pengukuran
menunjukkan sistemnya sehat: PSI memori 0,18%, lmkd nol pembunuhan, komposisi
tanpa fallback GPU, zram 2,75x dengan overhead 8%.

Yang dibeli dengan 400–740 jam: eBPF, cgroup v2, overlayfs, dan pencabutan 60
patch. **Tidak satu pun akan terasa oleh pengguna A37.** Yang membatasi
perangkat ini CPU dan RAM — terukur: keempat inti mentok 1.209.600 kHz dengan
16% idle saat bermain, dan working set game 650 MB melawan RAM 2 GB. Kernel
4.19 tidak mengubah keduanya.

---

### 8.4 Merge incremental per rilis mayor — jalur yang sebelumnya dikira mustahil

Metodologi standar industri: alih-alih mengambil pohon 4.19 yang sudah jadi
lalu menambahkan dukungan msm8916 ke dalamnya (pendekatan Fase 0-1), pohon A37
sendiri dimajukan bertahap lewat `git merge` upstream:

```
3.10.108 -> 3.14 -> 3.18 -> 4.4 -> 4.9 -> 4.14 -> 4.19
```

Dokumen ini sebelumnya menyatakan jalur ini tertutup karena pohon dikira hanya
punya 50 commit tanpa leluhur. Setelah dikoreksi (§2), prasyaratnya ternyata
terpenuhi: 445.894 commit, leluhur mainline sampai 2.6.12, dan 270 merge CAF.

**Keunggulan yang menjawab kegagalan Fase 1.** Pendekatan sekali-lompat
menempatkan kita di posisi terburuk: kernel tidak boot dan tidak ada cara tahu
kenapa (fase-1 §3). Dengan merge incremental, posisi itu tidak akan terjadi —
setiap langkah diuji boot, sehingga begitu ada yang rusak, langkah perusaknya
langsung teridentifikasi. Itu keunggulan struktural, bukan sekadar kerapian.

**Yang tidak dihilangkannya:**

- Blob kamera tetap risiko terbesar (§4.2). Wrapper/shim adalah jawaban yang
  benar tetapi tetap pekerjaan besar.
- Konsol tetap prasyarat. Bahkan dengan incremental, satu langkah yang membuat
  bootloop tetap menuntut melihat sebabnya. `lk2nd` tetap langkah pertama.
- `CONFIG_PSTORE`/`PSTORE_RAM` saja **tidak cukup** — ramoops baru hidup di
  `postcore_initcall`, dan kernel yang mati sebelum itu tidak meninggalkan
  jejak. Fase 1 membuktikannya lima kali, sekaligus membuktikan metodenya
  sendiri berfungsi ketika kernel benar-benar jalan.

**Biaya terukur, bukan perkiraan.** Satu loncatan terkecil diukur langsung
pada 5 September 2026 dengan `git merge v3.11` di branch percobaan:

```
merge-base   : commit "Linux 3.10" (pohon A37 bercabang tepat di sana)
berkas konflik : 676
  dokumentasi  :  12
  arsitektur lain (x86/powerpc/mips/...) : 0
  benar-benar relevan (arm, drivers, fs, kernel, mm, include) : 619
  menyentuh kode MSM/QCOM : 40
```

Nol konflik di arsitektur lain berarti hampir semuanya relevan — tidak bisa
diabaikan begitu saja.

676 konflik itu untuk **satu** loncatan minor (3.10 → 3.11). Jalur ke 4.19
melewati kira-kira 3.11, 3.12, 3.13, 3.14, 3.16, 3.18, 4.1, 4.4, 4.9, 4.14,
4.19 — sebelas loncatan. Bila 3.11 mewakili, ordenya beberapa ribu konflik
total, dan yang menyentuh kode MSM (bagian tersulit karena tidak ada rujukan
upstream) sekitar 40 per loncatan.

Sebagai kalibrasi: pekerjaan Fase 0-1 seluruhnya menghasilkan **nol** konflik
merge, karena pendekatannya menambahkan berkas ke pohon yang sudah 4.19.
Pertukarannya jelas — incremental jauh lebih mahal di muka, tetapi setiap
langkah bisa diuji boot, dan itu yang menyelamatkan dari kebuntuan Fase 1.

## 9. Gerbang keputusan

Tiga pertanyaan, berurutan. Jangan lanjut kalau jawabannya tidak.

1. **Tujuannya belajar atau hasil?** Kalau hasil untuk pengguna A37, §8.3
   menang telak. Kalau belajar forward-port kernel, proyek ini sangat berharga
   dan sasarannya sekarang jauh lebih realistis daripada rencana 4.4 semula.
2. **Bersedia kehilangan kamera?** Kalau tidak, hentikan sekarang — §4.2
   menempatkan ini sebagai risiko tertinggi tanpa mitigasi murah.
3. **Ada 400+ jam?** Fase 0 saja 20–40 jam, dan itu murah. Kalau ragu,
   kerjakan Fase 0 lalu putuskan ulang dengan data yang jauh lebih baik.

Kalau ketiganya lolos, mulai dari **Fase 0** dan jangan lewati penyaringannya.

---

## 10. Rujukan

Sasaran dan donor:

- Sasaran — https://github.com/LineageOS/android_kernel_xiaomi_msm8937 (`lineage-23.2`, 4.19.325)
- Donor clock 8909 — https://github.com/android-linux-stable/msm-4.9 (`kernel.lnx.4.9.r11-rel`)
- Donor clock 8916 — https://github.com/android-linux-stable/msm-3.18 (`kernel.lnx.3.18.r34-rel`)
- Pohon kita — https://github.com/rigaz29/kernel_oppo_msm8939 (`lineage-23`)

Disurvei dan ditolak:

- CAF msm-4.4 — https://github.com/android-linux-stable/msm-4.4 (`kernel.lnx.4.4.r40-rel`)
- CodeLinaro — https://git.codelinaro.org/explore
- CIP SLTS — https://www.kernel.org/pub/linux/kernel/projects/cip/4.4/
- Android common — https://android.googlesource.com/kernel/common/
- msm8916-mainline — https://github.com/msm8916-mainline
- postmarketOS MSM8916 — https://wiki.postmarketos.org/wiki/Qualcomm_Snapdragon_410_(MSM8916)

Dokumen terkait di repo kit `rigaz29/a37-23`:

- `PLAN-BACKPORT-KERNEL.md` — backport fitur ke 3.10 (arah sebaliknya)
- `PLAN-LOS23.md` — 60 patch BPF-less dan alasannya
- `RILIS.md` — jebakan build rilis
