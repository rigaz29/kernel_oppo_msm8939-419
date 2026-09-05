# Fase 0 — hasil penyaringan

Dikerjakan 5 September 2026. Semua angka hasil pengukuran, bukan perkiraan.

Tujuan fase ini menurut rencana: mengubah tebakan "sekitar sejuta baris" menjadi
daftar pekerjaan yang terukur. Hasilnya di bawah, dan **beberapa di antaranya
mengubah perkiraan biaya fase berikutnya secara mendasar.**

---

## 1. Penyaringan DTS — pengurangan 94%

Telusur `#include` transitif dari titik masuk `msm8916-mtp-15399.dts`:

```
DTS terpakai : 46 dari 800 berkas
baris        : 11.864 dari 179.285
include gagal: 0
```

Rencana memperkirakan 60–100 berkas. Kenyataannya **46**.

Daftar lengkap: `dts-wajib.txt`. Sepuluh terbesar:

| Baris | Berkas |
|---:|---|
| 2.081 | `msm8916.dtsi` |
| 1.447 | `msm8916-pinctrl.dtsi` |
| 879 | `msm8916-bus.dtsi` |
| 637 | `msm-pm8916.dtsi` |
| 630 | `msm8916-mtp-15399.dtsi` |
| 595 | `msm8916-coresight.dtsi` |
| 486 | `msm8916-mtp.dtsi` |
| 477 | `msm8916-camera-sensor-mtp.dtsi` |
| 422 | `msm8916-camera-sensor-mtp-15399.dtsi` |
| 384 | `msm8916-regulator.dtsi` |

---

## 2. Pemetaan defconfig — 81% sudah tersedia

597 opsi `=y` di `lineageos_a37f_defconfig` diadu dengan 17.692 simbol Kconfig
di pohon sasaran:

```
SUDAH ADA di 4.19 : 483  (81%)
HILANG            : 114  (19%)
yatim (tak terlacak): 0
```

Seluruh 114 berhasil dilacak ke berkas Kconfig di pohon 3.10 kita — tidak ada
satu pun yang menggantung. Sebarannya terpusat:

| Jumlah | Subsistem |
|---:|---|
| 14 | `drivers/soc/qcom` |
| 8 | `drivers/staging/prima` (WLAN) |
| 7 | `drivers/platform/msm` |
| 5 | `drivers/input/misc` |
| 4 | `init` |
| 3 | `drivers/staging/android` |
| 3 | `arch/arm64` |
| 3 | `drivers/mmc/core` |
| 3 | `drivers/thermal` |
| 2 | masing-masing: `ion`, `cpufreq`, `gpu/msm`, `pinctrl`, `ipc_router`, netfilter v4/v6 |

Berkas: `config-hilang.tsv` (simbol → Kconfig penyedia), `config-sudah-ada.txt`.

Sebagian dari 114 itu bukan pekerjaan porting melainkan **penerjemahan nama**
yang berubah di hulu — `CC_STACKPROTECTOR` menjadi `STACKPROTECTOR` (4.18),
`IPV6_PRIVACY`, `MMC_UNSAFE_RESUME`, `ARMV7_COMPAT`. Yang benar-benar butuh
kode adalah kelompok Qualcomm: `ARCH_MSM8916`, `MACH_15399`, `ION_MSM`,
`GPIO_QPNP_PIN`, `KGSL_PER_PROCESS_PAGE_TABLE`, `IPC_ROUTER`, dan `prima`.

---

## 3. Toolchain — GCC 4.9 TIDAK BISA dipakai

Temuan pertama yang menghentikan pekerjaan, dan persis alasan fase ini ada:

```
include/linux/compiler-gcc.h:20:3: error:
  Sorry, your version of GCC is too old - please use 5.1 or newer.
```

GCC 4.9 — toolchain yang dipakai kernel 3.10 kita dan tersedia di pohon
LineageOS (`prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9`) —
**ditolak kernel 4.19**.

Yang tersedia dan memenuhi syarat:

| Toolchain | Versi | Catatan |
|---|---|---|
| `/usr/bin/aarch64-linux-gnu-gcc` | 13.3.0 | dipakai untuk uji build fase ini |
| `prebuilts/clang/.../clang-r563880` | clang 21 | jauh lebih baru dari r416183b yang diminta pohon |
| `prebuilts/clang/.../clang-r574158` | clang 21 | idem |

Pohon sasaran sendiri meminta `LLVM=1` dengan `clang-r416183b`
(`build.config.common`), yang **tidak ada** di lingkungan ini.

### Hasil akhir: GCC 13 berhasil, setelah satu perbaikan

```
Image.gz-dtb  20.083.207 byte   EXIT=0, nol galat
```

Tetapi tidak langsung. GCC 13 berhenti di 2.302 objek dengan:

```
include/linux/thread_info.h:160:25: error: call to '__bad_copy_to'
  declared with attribute error: copy destination size is too small
```

Sumbernya `drivers/platform/msm/ipa/ipa_v2/ipa_debugfs.c:1924`, dan itu **bug
nyata**, bukan artefak kompilator:

```c
if (sizeof(dbg_buff) < count + 1)   /* count bertipe size_t */
    return -EFAULT;
if (copy_from_user(dbg_buff, ubuf, count))
```

Saat `count == SIZE_MAX`, `count + 1` melimpah menjadi 0, penjaganya lolos, dan
`copy_from_user` dipanggil dengan ukuran raksasa. Penjaga yang benar
`count >= sizeof(dbg_buff)`.

Polanya **sistematis: 10 kemunculan di 3 berkas** driver IPA
(`ipa_v2/ipa_debugfs.c` 8x, `ipa_v2/ipa_dma.c`, `ipa_v3/ipa_hw_stats.c`).
Semuanya diperbaiki, lalu build tuntas.

Bug ini sudah lama ada di pohon hulu dan tidak pernah terdeteksi karena GCC 4.9
dan clang lama tidak menganalisis sejauh itu. A37 memakai `CONFIG_IPA=y`, jadi
driver ini memang dipakai.

**Pelajaran untuk perkiraan biaya:** memakai toolchain modern pada kode CAF lama
akan memunculkan bug laten. Itu ongkos tersembunyi yang belum masuk hitungan
rencana, dan besarnya belum diketahui — 10 ini baru yang menghalangi build.

### Ukuran kernel — kekhawatiran partisi tidak beralasan

```
Image (mentah)  37.855.744
Image.gz        15.720.271   <- kernel saja
Image.gz-dtb    20.083.207   <- termasuk 16 DTB papan lain
```

Kernel 4.19 terkompresi 15,7 MB, dan itu memakai defconfig msm8937 yang jauh
lebih gemuk daripada milik kita (1.547 opsi `=y` vs 597). A37 hanya perlu satu
DTB, bukan 16. Bandingkan dengan TWRP terpasang sekarang yang kernelnya 18,5 MB
mentah dalam partisi 32 MB — kekhawatiran bahwa 4.19 tidak akan muat ternyata
tidak berdasar.

---

## 4. Kerangka clock — Fase 1 jauh lebih murah dari dugaan

Ini temuan paling melegakan.

`clock-gcc-8916.c`, berkas yang sama di dua versi kernel:

```
3.10 kita   : 3.052 baris
3.18 donor  : 2.988 baris
beda        :    58 baris   -> 98% identik
```

Dan kerangka CAF yang menopangnya nyaris tidak bergerak dari 3.18 ke 4.19:

| Berkas | 3.18 | 4.19 | Beda |
|---|---:|---:|---:|
| `clock-local2.c` | 2.924 | 2.907 | 100 |
| `clock-pll.c` | 1.206 | 1.282 | 113 |
| `clock-rpm.c` | 472 | 473 | 13 |
| `clock-voter.c` | 202 | 202 | 8 |
| `clock.c` | 1.411 | 1.407 | 37 |

Total pergeseran **~271 baris dari ~6.300 (4%)**.

Sebabnya: `drivers/clk/msm/` memakai kerangka clock milik CAF sendiri, **bukan**
kerangka `clk` mainline yang berubah drastis antara 3.15 dan 4.0. Rencana
menempatkan clock sebagai risiko utama Fase 1 dengan asumsi harus diadaptasi ke
`clk_hw`. Asumsi itu salah.

**Satu pemetaan jalur yang harus dicatat:** pohon 3.10 kita menaruh driver di
`drivers/clk/qcom/`, sedangkan 3.18, 4.9, dan 4.19 memakai `drivers/clk/msm/`.

---

## 5. MDSS — risiko terbesar Fase 3 hilang

Revisi perangkat keras MDP yang dikenal masing-masing pohon:

```
MDSS 4.19 sasaran : REV_101 103 105 106 107 108 109 110 112 114 115 116 300 320 330
MDSS 3.10 kita    : REV_101 103 105 106 107 108 109
```

**Himpunan kita adalah bagian penuh dari milik 4.19.** msm8916 memakai REV_105
dan REV_106, dan keduanya ditangani MDSS 4.19 dengan cara yang sama.

Artinya inti MDSS **tidak perlu dipindah sama sekali**. Pekerjaan Fase 3
menyusut menjadi device tree, driver panel A37, dan konfigurasi DSI PHY.

Berkas MDSS-nya sendiri memang menyimpang jauh — tetapi itu justru alasan untuk
**memakai versi 4.19 apa adanya**, bukan memindahkan versi kita:

| Berkas | 3.10 | 4.19 | Beda |
|---|---:|---:|---:|
| `mdss_mdp.c` | 3.389 | 5.608 | 38% |
| `mdss_dsi.c` | 2.087 | 4.889 | 50% |
| `mdss_mdp_ctl.c` | 3.761 | 6.451 | 38% |
| `mdss_fb.c` | 3.736 | 5.604 | 31% |
| `mdss_dsi_panel.c` | 1.918 | 3.824 | 47% |
| `mdss_dsi_host.c` | 2.657 | 3.477 | 23% |

Pertumbuhan ~60% itu dukungan untuk SoC yang lebih baru, bukan perombakan.

---

## 6. Donor terverifikasi

```
donor-3.18  clock-gcc-8916.c (2.988)  clock-rpm-8916.c  clock-cpu-8939.c
donor-4.9   clock-gcc-8909.c (2.919)  clock-cpu-8939.c  clock-gcc-8952.c
target 4.19 clock-cpu-8939.c          clock-gcc-8952.c  clock-gcc-8953.c
```

Untuk menulis `clock-gcc-8916.c` di 4.19, urutan rujukannya: pakai versi 3.18
sebagai dasar (98% sama dengan milik kita), lalu bandingkan `clock-gcc-8952.c`
3.18 dengan 4.19 untuk melihat penyesuaian apa yang dituntut kerangka baru.

---

## 7. Dampak pada perkiraan rencana

| Fase | Perkiraan semula | Revisi | Alasan |
|---|---|---|---|
| 1 clock+boot | 40–80 jam | **20–40** | kerangka clock hanya bergeser 4% |
| 3 tampilan | 80–150 jam | **40–90** | inti MDSS tidak perlu dipindah |
| lainnya | tetap | tetap | belum ada data baru |

Perkiraan total turun dari **400–740 jam** menjadi kira-kira **320–610 jam**.

Yang **belum** berkurang risikonya sama sekali: kamera, 66 blob userspace, dan
kompatibilitas ABI. Itu tetap ancaman terbesar dan tidak tersentuh Fase 0.

Dan satu risiko **baru** yang ditemukan fase ini: bug laten di kode CAF yang
hanya muncul dengan toolchain modern. Sepuluh sudah ditemukan hanya untuk
membuat build lewat; berapa lagi yang menunggu belum diketahui.

---

## 8. Ruang kerja

```
/root/kfp/target      LineageOS msm8937 lineage-23.2, 4.19.325, commit 704f6f081
/root/kfp/donor-4.9   CAF msm-4.9  kernel.lnx.4.9.r11-rel
/root/kfp/donor-3.18  CAF msm-3.18 kernel.lnx.3.18.r34-rel
/root/los23/kernel/oppo/msm8939   pohon 3.10 kita (JANGAN DISENTUH)
```

Semua clone `--depth 1`. Total ~3,5 GB.
