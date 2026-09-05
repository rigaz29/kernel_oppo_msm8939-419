# Fase 1 — kernel 4.19 pertama untuk A37

Dikerjakan 5 September 2026. Status: **boot image siap uji, belum dijalankan di
perangkat.**

---

## 0. Koreksi atas kesimpulan Fase 0

Fase 0 menyimpulkan port clock "nyaris mekanis karena kerangka CAF hanya
bergeser 4%". **Kesimpulan itu salah sasaran.**

Yang diukur adalah `drivers/clk/msm/` — kerangka clock CAF lama. Pohon sasaran
memakai `CONFIG_COMMON_CLK_QCOM`, yaitu kerangka **mainline** di
`drivers/clk/qcom/`. Berkas CAF itu ada di pohon tetapi tidak pernah dibangun:

```
CONFIG_COMMON_CLK_QCOM=y
CONFIG_USE_COMMON_CLK_QCOM=y
objek yang benar-benar dibangun: gcc-sdm439.o, gcc-sdm429w.o
```

Kekeliruan itu berujung kabar jauh lebih baik.

---

## 1. Temuan: dukungan msm8916 sudah ada, tidak perlu di-port

```
drivers/clk/qcom/gcc-msm8916.c          83.428 byte
drivers/clk/qcom/apcs-msm8916.c
drivers/clk/qcom/a53-pll.c
drivers/pinctrl/qcom/pinctrl-msm8916.c
arch/arm64/boot/dts/qcom/msm8916.dtsi   34.554 byte
arch/arm64/boot/dts/qcom/pm8916.dtsi
arch/arm64/boot/dts/qcom/msm8916-mtp.dts + .dtsi
```

Semuanya driver mainline yang sudah masuk Linux sejak 4.0. Impor
`clock-gcc-8916.c` dari donor 3.18 yang sempat dilakukan **dibatalkan** —
jalur yang salah.

Struktur DTS pohon sasaran memisahkan keduanya dengan rapi:

```
arch/arm64/boot/dts/qcom/           DTS mainline (direktori nyata)
arch/arm64/boot/dts/vendor-legacy/  DTS CAF
```

Di 3.10 dan 4.4, `arch/arm64/boot/dts/qcom` adalah symlink ke
`arch/arm/boot/dts/qcom`. Di 4.19 tidak lagi.

---

## 2. A37 adalah turunan MTP

`compatible` di DTS CAF 3.10 kita dan DTS MTP mainline hampir identik:

```
CAF 3.10 A37 : "qcom,msm8916-mtp", "qcom,msm8916", "qcom,mtp"
mainline MTP : "qcom,msm8916-mtp", "qcom,msm8916-mtp/1", "qcom,msm8916", "qcom,mtp"
```

Jadi `msm8916-mtp.dtsi` mainline adalah titik berangkat yang benar.

---

## 3. `msm8916-oppo-a37.dts` — 26 baris

Berbasis `msm8916-mtp.dtsi`, ditambah dua properti yang mainline tidak punya
tetapi **bootloader LK pada perangkat ini membutuhkannya** untuk memilih DTB:

```
qcom,msm-id   = <206 0>, <248 0>, <249 0>, <250 0>;
qcom,board-id = <8 0 15399>;
```

Terverifikasi di DTB hasil kompilasi:

```
qcom,board-id = 8 0 3c27          (0x3c27 = 15399)
qcom,msm-id   = ce 0 f8 0 f9 0 fa 0
model         = OPPO A37
stdout-path   = serial0
serial@78b0000 status = okay
```

---

## 4. Yang diaktifkan di konfigurasi

Konfigurasi dasar `vendor/msm8937-perf_defconfig` tidak menyalakan driver
msm8916 sama sekali. Ditambahkan:

```
MSM_GCC_8916  QCOM_CLK_APCS_MSM8916  QCOM_APCS_IPC  PINCTRL_MSM8916
SERIAL_MSM  SERIAL_MSM_CONSOLE  QCOM_TSENS  QCOM_SMEM  QCOM_SMD
PSTORE  PSTORE_RAM  PSTORE_CONSOLE  PSTORE_PMSG
```

`QCOM_CLK_APCS_MSM8916` menolak menyala sampai `QCOM_APCS_IPC` dinyalakan
lebih dulu (dependensi mailbox).

Empat opsi PSTORE penting karena **A37 tidak punya port serial yang bisa
diakses**. Ramoops satu-satunya cara membaca log kernel, dan cmdline A37 sudah
mengonfigurasinya (`ramoops.mem_address=0x9ff00000`).

---

## 5. Hasil build

```
Image.gz                     15.783.275 byte
msm8916-oppo-a37.dtb             35.888 byte
dt.img                           38.912 byte
boot-a37-419-fase1.img       17.383.424 byte
sha256 35788e4582058a1cdd5307d86ff55a494366dd568e9629eb343b44005d0e5741
```

Objek yang terkompilasi dan simbol yang tertaut:

```
gcc-msm8916.o 272.912   pinctrl-msm8916.o 246.448   msm_serial.o 339.568
apcs-msm8916.o 153.680  a53-pll.o 154.640

System.map: gcc_msm8916_probe  msm8916_pinctrl_probe  msm_serial_probe
            qcom_apcs_ipc_probe  ramoops_probe  pstore_register
```

Header boot image terverifikasi: kernel @ 0x80008000, ramdisk @ 0x81000000,
tags 0x80000100, page_size 2048, dt 38.912.

---

## 6. Toolchain

GCC 13.3.0 (`/usr/bin/aarch64-linux-gnu-`). GCC 4.9 dari pohon LineageOS
ditolak kernel 4.19 (butuh 5.1+), lihat laporan Fase 0.

Perbaikan IPA dari Fase 0 tetap diperlukan.

---

## 7. Dua jebakan alat yang memakan waktu

**`dtbToolOppo -p` menerima DIREKTORI, bukan biner.** Kode menyambung
`dtc_path + "dtc -I dtb -O dts"`, jadi memberi jalur ke biner menghasilkan
`.../dtc/dtcdtc` yang gagal senyap. Akibatnya deteksi versi jatuh ke v1, dan v1
mengharapkan triplet `<x y z>` sementara `msm-id` kita punya 8 nilai — pesan
galatnya menyesatkan ("incorrect 'qcom,msm-id = <' format") padahal formatnya
benar.

Setelah diperbaiki:

```
Version:2
chipset: 206, rev: 0, platform: 8, subtype: 0, oppoId: 15399
additional chipset: 248, 249, 250
=> Found 4 unique DTB(s)
```

Identik dengan pola log build ROM 3.10 kita.

**Flag `-2`/`-3` tidak memengaruhi deteksi versi**, hanya format keluaran.
Deteksi versi murni dari hasil dekompilasi: v2 bila ada `qcom,board-id`, v3
bila ada `qcom,pmic-id`.

---

## 8. Cara uji — AMAN, tidak menulis ke partisi

```sh
adb reboot bootloader
fastboot boot boot-a37-419-fase1.img
```

**`fastboot boot`, bukan `fastboot flash`.** Perintah ini menjalankan image dari
RAM tanpa menyentuh partisi `boot`. Kalau gagal, cabut baterai dan perangkat
kembali persis seperti semula. TWRP dan ROM 3.10 tetap utuh.

Kemungkinan hasil:

| Yang terlihat | Artinya |
|---|---|
| Layar tetap mati, tidak ada reaksi | kernel tidak jalan; baca ramoops |
| Reboot sendiri setelah beberapa detik | panic; baca ramoops |
| Bootanimation / logo | jauh melampaui target Fase 1 |

Membaca ramoops setelah gagal, lewat TWRP:

```sh
adb shell "cat /sys/fs/pstore/console-ramoops-0" > fase1-console.txt
adb shell "cat /sys/fs/pstore/dmesg-ramoops-0"   > fase1-panic.txt
```

Kriteria lulus Fase 1: `console-ramoops` berisi baris `Booting Linux on
physical CPU` dan seterusnya sampai `console_init`.

---

## 9. Yang belum dikerjakan

Ramdisk yang dipakai diambil dari `boot.img` ROM 3.10 yang ada. Isinya `init`
dan sepolicy untuk kernel 3.10, jadi **tidak diharapkan boot sampai Android** —
tujuannya hanya membuktikan kernel hidup dan konsol menulis.

Belum disentuh sama sekali: MDSS, KGSL, audio, WLAN (`prima`), kamera, dan
seluruh DTS CAF untuk perangkat keras itu.
