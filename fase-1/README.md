# Fase 1 — kernel 4.19 pertama untuk A37

Dikerjakan 5 September 2026.

**Status: BELUM LULUS, penyelidikan buntu tanpa konsol serial.** Kernel
terbangun dan ter-flash, tetapi tidak pernah menulis sebaris pun. Lima
percobaan di perangkat, dua belas percobaan build.

Tujuh tersangka disingkirkan dengan bukti eksperimental (§3). Yang tersisa
menuntut melihat apa yang terjadi sebelum pstore hidup, dan A37 tidak
menyediakan jalan ke sana — karena itu rekomendasinya adalah mendapatkan
konsol lebih dulu lewat `lk2nd` (§7), bukan melanjutkan tebakan konfigurasi.

Dokumen ini mencatat apa yang berhasil, apa yang tersingkir sebagai penyebab,
dan satu temuan struktural yang mengubah penilaian biaya seluruh proyek.

---

## 0. Koreksi atas kesimpulan Fase 0

Fase 0 menyimpulkan port clock "nyaris mekanis karena kerangka CAF hanya
bergeser 4%". **Salah sasaran.** Yang diukur `drivers/clk/msm/` (kerangka CAF
lama); pohon sasaran memakai `CONFIG_COMMON_CLK_QCOM`, kerangka mainline di
`drivers/clk/qcom/`. Berkas CAF itu ada tetapi tidak pernah dibangun — objek
yang dihasilkan `gcc-sdm439.o` dan `gcc-sdm429w.o`.

Kekeliruan itu berujung kabar baik (§1).

---

## 1. Yang berhasil

### Dukungan msm8916 sudah ada, tidak perlu di-port

```
drivers/clk/qcom/gcc-msm8916.c          83.428 byte
drivers/clk/qcom/apcs-msm8916.c
drivers/clk/qcom/a53-pll.c
drivers/pinctrl/qcom/pinctrl-msm8916.c
arch/arm64/boot/dts/qcom/msm8916.dtsi   34.554 byte
arch/arm64/boot/dts/qcom/pm8916.dtsi
```

Semuanya driver mainline sejak Linux 4.0. Impor `clock-gcc-8916.c` dari donor
3.18 yang sempat dilakukan dibatalkan.

### A37 adalah turunan MTP

```
CAF 3.10 A37 : "qcom,msm8916-mtp", "qcom,msm8916", "qcom,mtp"
mainline MTP : "qcom,msm8916-mtp", "qcom,msm8916-mtp/1", "qcom,msm8916", "qcom,mtp"
```

### `msm8916-oppo-a37.dts` — 62 baris

Berbasis `msm8916-mtp.dtsi`, ditambah:

- `qcom,msm-id` dan `qcom,board-id = <8 0 15399>` — bootloader LK memilih DTB
  dengan mencocokkan keduanya; mainline tidak memakainya.
- `memory@80000000` eksplisit 2 GB — mainline memakai `reg = <0 0 0 0>` dengan
  komentar "We expect the bootloader to fill in the reg".
- `reserved-memory` untuk ramoops di 0x9ff00000 dengan layout persis sama
  dengan cmdline ROM/TWRP.

DTB terverifikasi: `board-id = 8 0 3c27` (15399), `msm-id = ce/f8/f9/fa`,
`model = OPPO A37`, `stdout-path = serial0`.

### Kernel terbangun dan tertaut benar

```
Image                        33.464.832 byte
msm8916-oppo-a37.dtb             36.156 byte
dt.img                           38.912 byte  -> "Found 4 unique DTB(s)"
boot.img                     33.509.376 byte  (partisi 33.554.432, sisa 45.056)

objek : gcc-msm8916.o 272.912  pinctrl-msm8916.o 246.448  msm_serial.o 339.568
        apcs-msm8916.o 153.680  a53-pll.o 154.640
System.map : gcc_msm8916_probe  msm8916_pinctrl_probe  msm_serial_probe
             qcom_apcs_ipc_probe  ramoops_probe  pstore_register
```

---

## 2. Kegagalan di perangkat

Tiga percobaan, semuanya menggantung di logo OPPO, **nol jejak di ramoops**.

### Percobaan 1 — `Image.gz`

Penyebab ditemukan dan pasti:

```
boot.img 3.10 yang bekerja : magic 10000014 -> Image arm64 MENTAH
boot.img 4.19 percobaan 1  : magic 1f8b0800 -> gzip
```

Kernel arm64 tidak punya stub dekompresi seperti `zImage` di arm32 —
bootloader yang harus mendekompresi, dan LK ini tidak melakukannya.

Data untuk mendeteksi ini sudah tersedia sejak awal (saat mengekstrak ramdisk
dari `boot.img` perangkat) dan tidak diperiksa.

### Percobaan 2 — `Image` mentah

Masih nol jejak. Dua kesalahan ditemukan setelahnya:

- `ramoops.ecc=0` yang dipakai bentrok dengan `ecc=32` milik ROM dan TWRP.
  Buffer yang ditulis dengan satu tata letak dan dibaca dengan tata letak lain
  tampak rusak.
- DTS tidak mengisi node `memory` dan tidak mem-*reserve* area ramoops.

### Percobaan 3 — DTS diperbaiki, ecc dicocokkan

Masih nol jejak.

---

## 3. Yang sudah tersingkir sebagai penyebab

| Tersangka | Bukti |
|---|---|
| Format kernel gzip | diperbaiki, `Image` mentah seperti 3.10 |
| Format `dt.img` | **identik** dengan ROM 3.10: QCDT v2, 4 entri, chipset 0xce, board 0x3c27, struktur entri sama |
| Header arm64 | valid (`ARM\x64`), `text_offset` 0x80000 sama dengan 3.10 |
| Toolchain GCC 13 | nol instruksi ARMv8.2+/LSE/BTI/PAC — kode kompatibel Cortex-A53 |
| RAM tidak terisi | diperbaiki, DTS mengisi 2 GB eksplisit |
| ramoops tidak ter-reserve | diperbaiki, `reserved-memory` + `ecc=32` |

Tidak adanya `dmesg-ramoops` sama sekali berarti **kernel berhenti sebelum
pstore hidup** — sangat awal, sebelum menulis sebaris pun.

### Ukuran DIUJI dan TERSINGKIR

Kernel 3.10 yang terbukti bekerja di-padding nol sampai 33.339.392 byte —
99,6% dari ukuran kernel 4.19 — lalu dibungkus dengan ramdisk mini dan
`dt.img` CAF asli, sehingga satu-satunya variabel yang berubah adalah ukuran.

Hasilnya boot sempurna:

```
Linux version 3.10.108-lineageos-g756cb4623341-dirty
Machine: Qualcomm Technologies, Inc. MSM 8916 MTP
Memory: 1874536K/1990656K available (11570K kernel code, ...)
[2.943203] Kernel panic - not syncing: VFS: Unable to mount root fs
           on unknown-block(0,0)
```

Panic di detik 2,94 persis seperti yang diharapkan dari ramdisk mini, dan
`dmesg-ramoops-0` muncul untuk pertama kalinya (209.672 byte).

**LK memuat dan menjalankan kernel 33,3 MB tanpa keluhan.** Ukuran bukan
penyebab, dan perang konfigurasi di §4 tidak diperlukan untuk kasus ini.

Uji ini juga membuktikan metode ramoops bekerja: ketika kernel benar-benar
jalan, lognya tercatat dan terbaca dari TWRP.

### DTB diuji dan tersingkir

Kernel 4.19 dipasangkan dengan `dt.img` CAF asli — device tree yang sama persis
yang baru saja membuktikan kernel 3.10 bisa boot di ukuran 33 MB — dan cmdline
ROM 3.10 apa adanya. Tetap nol jejak.

### KASLR, KPTI, dan header EFI diuji dan tersingkir

Perbedaan arsitektural yang tersisa antara kernel 4.19 dan kernel 3.10 yang
bekerja dimatikan sekaligus:

```
RANDOMIZE_BASE, RELOCATABLE       KASLR: kernel merelokasi diri saat boot,
                                  butuh kaslr-seed yang LK 2015 tidak sediakan
UNMAP_KERNEL_AT_EL0               KPTI, tidak dibutuhkan Cortex-A53
ARM64_UAO, ARM64_PAN, SW_TTBR0_PAN  fitur ARMv8.1
EFI, EFI_STUB                     agar code0 jadi branch murni
```

Hasilnya `code0` berubah dari `0x91005a4d` (MZ/EFI) menjadi `0x145e0000`
(instruksi branch), sejenis dengan kernel 3.10 yang bekerja (`0x14000010`).
`-Os` juga dipakai, menurunkan `Image` menjadi 29.261.832 byte dengan sisa
partisi 4,2 MB — tidak lagi mepet.

Terpasang dan terverifikasi di perangkat (byte pertama kernel di partisi boot
`00 00 5e 14`). **Tetap nol jejak.**

### Kesimpulan penyelidikan

Kernel 4.19 mati sebelum pstore hidup, konsisten di seluruh konfigurasi.
Tersangka yang tersingkir dengan bukti eksperimental:

| Tersangka | Cara diuji | Hasil |
|---|---|---|
| Format gzip | ganti ke `Image` mentah | tetap gagal |
| Ukuran kernel | kernel 3.10 di-padding 33,3 MB | **boot sempurna** |
| DTB buatan sendiri | pakai `dt.img` CAF asli | tetap nol |
| KASLR / relokasi | `RANDOMIZE_BASE` mati | tetap nol |
| KPTI / ARMv8.1 | `UNMAP_KERNEL_AT_EL0` dll mati | tetap nol |
| Header MZ/EFI | `code0` jadi branch murni | tetap nol |
| Toolchain | nol instruksi ARMv8.2+ | bukan penyebab |

**Batas metode.** Tanpa konsol serial fisik, tidak ada cara mempersempit lebih
jauh: setiap tersangka berikutnya butuh melihat apa yang terjadi *sebelum*
pstore hidup, dan A37 tidak menyediakan jalan ke sana.

Lima percobaan flash, dua belas percobaan build.

---

## 4. Temuan struktural: pohon CAF menolak dikonfigurasi ulang

Ini temuan terpenting Fase 1, dan mengubah penilaian biaya seluruh proyek.

Sepuluh percobaan build. Tujuh gagal, semuanya pola yang sama: pohon ini punya
kode ber-`obj-y` **tanpa penjaga config**, sehingga mematikan hampir apa pun
memutus penautan di tempat yang tidak terduga.

| Yang dimatikan | Yang putus |
|---|---|
| `ARCH_MSM8937` | techpack mencari `msm8909auto.conf`, struct `clk_regmap_mux_div` kehilangan `clk_lpm` |
| `QSEECOM` | `qseecom_send_command` (driver kamera) |
| `FB_MSM` | `msm_dss_config_vreg`, `msm_dss_get_clk` (mdss-pll di `drivers/clk/msm/mdss/`, `obj-y`) |
| `FTRACE` | `tracing_prog_func_proto` (bpf_lsm) |
| `MSM_VIDC` | `vb2_mmap`, `vb2_streamoff` (v4l2-mem2mem) |
| `NET_SCHED` | `tc_qdisc_flow_control` (rmnet) |
| `MSM_CAMERA` | nama salah — yang benar `MSMB_CAMERA` |

Dan jalur mainline tertutup total:

```
defconfig arm64 generik -> mm_struct has no member named mmap_sem  (kode THP)
THP dimatikan           -> mmu_notifier.h: parameter 'event' has incomplete type
MMU_NOTIFIER dimatikan  -> galat sama tetap muncul
```

`mmap_sem` sudah dihapus dari `mm_struct` di pohon ini tetapi kode THP masih
merujuknya; `mmu_notifier.h` rusak tanpa bergantung config. Artinya kombinasi
config di luar yang disediakan vendor **tidak pernah diuji dan tidak bisa
dibangun**.

**Konsekuensi untuk rencana:** setiap fase yang menuntut konfigurasi berbeda
dari `vendor/msm8937-perf_defconfig` akan menabrak tembok ini. Ongkosnya tidak
ada di perkiraan Fase 0 maupun rencana induk.

Pemangkasan maksimum yang berhasil dibangun: **1.545 → 1.358 opsi `=y`**,
`Image` 37.900.800 → 33.464.832 byte (turun 11,7%), dengan mematikan
`MSMB_CAMERA`, `IPA`, `PRIMA_WLAN`, `DIAG_CHAR`, `KALLSYMS_ALL`, `DEBUG_LIST`,
netfilter, IPv6, wireless, dan Bluetooth.

---

## 5. Dua jebakan alat

**`dtbToolOppo -p` menerima DIREKTORI, bukan biner.** Kode menyambung
`dtc_path + "dtc -I dtb -O dts"`, jadi memberi jalur ke biner menghasilkan
`.../dtc/dtcdtc` yang gagal senyap. Deteksi versi lalu jatuh ke v1, dan v1
mengharapkan triplet `<x y z>` sementara `msm-id` kita punya 8 nilai — pesan
galatnya menyesatkan ("incorrect 'qcom,msm-id = <' format") padahal formatnya
benar. Flag `-2`/`-3` hanya mengubah format keluaran, bukan deteksi versi.

**GCC 4.9 dari pohon LineageOS ditolak 4.19** ("please use 5.1 or newer").
GCC 13.3.0 sistem dipakai, setelah perbaikan IPA dari Fase 0.

---

## 6. Berkas

```
msm8916-oppo-a37.dts          DTS v2, 62 baris
config-a37-caf-trimmed.txt    konfigurasi terpangkas yang build bersih
patches/0001-a37-dts-pertama.patch
```

Boot image yang diuji (tidak disimpan di repo, ukurannya 33 MB):

```
boot-a37-419-fase1b.img  sha256 0eff0fbed40bb45373dfdc273671c7d06825ab3abc2a1ca37927d7c04c6415b0
boot-a37-419-fase1c.img  sha256 2c49f57ec3a8d63373b69626dd417b172152e1dbf822ece7148569332ff72580
```

---

## 7. Langkah berikutnya bila dilanjutkan

**Rekomendasi: mulai dari `lk2nd`, bukan dari menebak konfigurasi kernel.**

Proyek `msm8916-mainline` membuat `lk2nd` persis untuk situasi ini: bootloader
kedua yang dimuat oleh LK stock, lalu menyediakan penanganan DTB modern **dan
konsol via USB**. Konsol itu yang selama ini kita kekurangan, dan tanpanya
penyelidikan ini buntu.

Mereka sudah memecahkan masalah yang sama untuk puluhan perangkat msm8916,
termasuk perangkat dengan LK sekelas A37.

Urutan yang masuk akal bila dilanjutkan:

1. **`lk2nd`** — dapatkan konsol lebih dulu. Semua penyelidikan berikutnya
   menjadi murah begitu kernel bisa bicara.
2. Dengan konsol tersedia, ulangi kernel 4.19 ini apa adanya; penyebab
   kematiannya akan langsung terlihat.
3. Baru pertimbangkan apakah tetap di 4.19 CAF (dengan beban §4) atau pindah
   ke mainline 6.x yang dipakai `msm8916-mainline` — konsekuensinya dicatat di
   rencana induk §8.2, tumpukan HAL Android CAF tidak dipakai.

### Yang terbukti berguna dan layak dipertahankan

- **Ramdisk kosong 129 byte** — kernel diharapkan panic `no init found` setelah
  konsol hidup, dan itu justru bukti yang dicari. Terbukti bekerja pada uji
  padding.
- **Uji padding sebagai kontrol.** Membungkus kernel yang *terbukti bekerja*
  agar menyerupai kandidat yang gagal adalah cara termurah menyingkirkan
  variabel. Satu flash menggugurkan hipotesis yang sudah menelan sepuluh build.
- **`ramoops.ecc` harus sama** dengan ROM/TWRP (32). Nilai berbeda membuat
  buffer terbaca sebagai sampah.
- **Jangan biarkan TWRP menyala lama** sebelum menarik log — TWRP menulis ke
  buffer ramoops yang sama dan menimpa jejak kernel sebelumnya.
