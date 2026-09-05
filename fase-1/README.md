# Fase 1 — kernel 4.19 pertama untuk A37

Dikerjakan 5 September 2026.

**Status: BELUM LULUS.** Kernel terbangun dan ter-flash, tetapi tidak pernah
menulis sebaris pun ke konsol. Tiga percobaan di perangkat, sepuluh percobaan
build.

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

### Tersangka yang tersisa, belum diuji

**Ukuran.** Kernel 3.10 yang bekerja 18.562.488 byte; kernel 4.19 ini
33.464.832 byte, 80% lebih besar, dan `boot.img` menyisakan hanya 45 KB dari
partisi. LK lama sering punya batas yang tidak terdokumentasi.

Menguji ini butuh kernel 4.19 yang jauh lebih kecil — dan §4 menjelaskan
kenapa itu tidak tercapai.

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

Berurutan menurut nilai per usaha:

1. **Uji hipotesis ukuran tanpa membangun kernel kecil.** Ambil kernel 3.10
   yang bekerja, bungkus dalam `boot.img` dengan padding sampai ~33 MB, dan
   lihat apakah masih boot. Kalau tidak, ukuran terbukti jadi penyebab tanpa
   perlu memenangkan perang konfigurasi.

2. **Cari LK OPPO A37 atau sumber `lk` msm8916** untuk membaca batas ukuran
   kernel dan cara ia menghitung alamat muat.

3. **Pertimbangkan pohon lain.** `msm8916-mainline` menjalankan msm8916 di
   kernel 6.x tanpa beban CAF, dan konfigurasinya bisa diatur bebas. Itu
   menghindari §4 sepenuhnya, dengan konsekuensi yang sudah dicatat di rencana
   induk §8.2 — tumpukan HAL Android CAF tidak dipakai.

4. Ramdisk kosong 129 byte terbukti cukup dan benar untuk uji ini: kernel
   diharapkan panic `no init found` setelah konsol hidup, dan itu justru bukti
   yang dicari.
