# VPP Patch 25.02 — Installation Guide

Panduan instalasi VPP yang telah di-patch untuk **Debian 12 (Bookworm)** dengan NIC **Intel 82598EB 10GbE (`8086:10c6`)**.

> [!IMPORTANT]
> Patch ini dibuat khusus untuk hardware dan NIC yang disebutkan di bawah. Penggunaan pada hardware atau NIC lain belum tentu kompatibel.

## Persyaratan

Sebelum melakukan instalasi, pastikan sistem memenuhi persyaratan berikut:

* **OS:** Debian 12 (Bookworm) amd64
* **Kernel:** Linux 6.1
* **NIC:** Intel 82598EB 10GbE
* **PCI Device ID:** `8086:10c6`
* **IOMMU:** Intel VT-d / DMAR aktif
* **IOMMU Mode:** Translated

Jika IOMMU belum aktif pada kernel, tambahkan parameter berikut pada kernel command line:

```text
intel_iommu=on
```

IOMMU juga harus diaktifkan melalui BIOS/UEFI.

---

## Hardware yang Digunakan

Build dan patch ini diuji menggunakan hardware berikut:

| Komponen          | Spesifikasi                     |
| ----------------- | ------------------------------- |
| Appliance         | Sophos XG Network Appliance     |
| Motherboard       | Sophos "XG"                     |
| BIOS              | American Megatrends Inc. 5.11   |
| CPU               | Intel Core i7-6700K @ 4.00 GHz  |
| CPU Core / Thread | 4 Core / 8 Thread               |
| RAM               | 7.7 GiB                         |
| Storage           | ADATA ASU800SS-128GT — 119.2 GB |
| OS                | Debian 12 Bookworm              |
| Kernel            | 6.1.0-51-amd64                  |
| IOMMU             | Intel VT-d / DMAR               |
| IOMMU Domain      | Translated                      |
| NIC               | Intel 82598EB 10GbE             |
| PCI ID            | `8086:10c6`                     |

---

## Instalasi

### 1. Clone Repository

Clone repository ini:

```bash
git clone https://github.com/niammuddin/vpp-patch
cd vpp-patch
```

Update package index dan install dependency yang dibutuhkan:

```bash
apt-get update

apt-get install -y \
  libnuma1 \
  libnl-3-200 \
  libnl-route-3-200 \
  libunwind8 \
  libssl3 \
  libpcap0.8 \
  libelf1
```

Kemudian install package VPP:

```bash
dpkg -i \
  libvppinfra_*.deb \
  vpp_*.deb \
  vpp-plugin-core_*.deb \
  vpp-plugin-dpdk_*.deb
```

> [!NOTE]
> Jika `dpkg` mengalami error karena versi package bentrok dengan repository **FD.io**, nonaktifkan repository FD.io terlebih dahulu dengan me-rename file `.list` terkait.
>
> Alternatifnya, gunakan:
>
> ```bash
> dpkg -i --force-downgrade \
>   libvppinfra_*.deb \
>   vpp_*.deb \
>   vpp-plugin-core_*.deb \
>   vpp-plugin-dpdk_*.deb
> ```

---

### 2. Pasang Konfigurasi VPP

Salin `startup.conf` dari repository:

```bash
cp startup.conf /etc/vpp/startup.conf
```

> [!WARNING]
> PCI address NIC pada setiap perangkat bisa berbeda. Jangan langsung menggunakan PCI address dari konfigurasi contoh tanpa melakukan pengecekan.

Cari NIC Intel `82598EB` dengan:

```bash
lspci -nn | grep 8086:10c6
```

Contoh output:

```text
01:00.0 Ethernet controller [0200]: Intel Corporation 82598EB 10-Gigabit ...
01:00.1 Ethernet controller [0200]: Intel Corporation 82598EB 10-Gigabit ...
```

Jika PCI address berbeda, sesuaikan bagian berikut di `/etc/vpp/startup.conf`:

```text
0000:01:00.0
0000:01:00.1
```

dengan PCI address NIC pada perangkat yang digunakan.

---

### 3. Siapkan HugePages

Konfigurasi berikut menyediakan **1024 × 2 MiB = 2 GiB HugePages**:

```bash
echo "vm.nr_hugepages=1024" >> /etc/sysctl.conf
sysctl -p
```

Verifikasi:

```bash
grep Huge /proc/meminfo
```

Pastikan `HugePages_Total` menunjukkan jumlah HugePages yang telah dialokasikan.

---

### 4. Siapkan `vfio-pci` dan Jalankan VPP

Load kernel module `vfio-pci`:

```bash
modprobe vfio-pci
```

Daftarkan Intel `82598EB` (`8086:10c6`) ke driver `vfio-pci`:

```bash
echo "8086 10c6" > /sys/bus/pci/drivers/vfio-pci/new_id
```

Turunkan interface kernel sebelum VPP mengambil alih NIC:

```bash
ip link set enp1s0f0 down
ip link set enp1s0f1 down
```

> [!NOTE]
> Nama interface seperti `enp1s0f0` dan `enp1s0f1` dapat berbeda pada setiap perangkat.
>
> Cek nama interface dengan:
>
> ```bash
> ip link
> ```

Kemudian jalankan VPP:

```bash
systemctl start vpp
```

Periksa status service:

```bash
systemctl status vpp
```

Jika VPP gagal start, periksa log:

```bash
journalctl -u vpp -n 100 --no-pager
```

---

## Verifikasi

### Cek Versi VPP

```bash
vppctl show version
```

Versi yang diharapkan:

```text
v25.02-release built by root on debian
```

### Cek Interface

```bash
vppctl show interface
```

Pastikan interface berhasil terdeteksi dan **`rx-error` tidak muncul**.

Untuk melihat detail error counter:

```bash
vppctl show errors
```

---

## Catatan Penting

> [!IMPORTANT]
> **DPDK tidak perlu di-install secara terpisah.**
>
> Patch DPDK sudah ter-embed secara statik di dalam package:
>
> ```text
> vpp-plugin-dpdk_*.deb
> ```

Pastikan sistem tidak memiliki instalasi VPP lain yang dapat menyebabkan konflik:

```bash
dpkg -l | grep vpp
```

Jika sebelumnya pernah menggunakan VPP dari repository FD.io, pastikan package dan repository lama tidak menyebabkan versi dependency bercampur.

Konfigurasi CLI VPP setelah instalasi tetap harus dilakukan secara manual sesuai kebutuhan jaringan.

---

## Troubleshooting

### VPP Tidak Bisa Mengakses NIC

Pastikan IOMMU aktif:

```bash
dmesg | grep -Ei 'DMAR|IOMMU'
```

Periksa driver yang sedang digunakan NIC:

```bash
lspci -nnk | grep -A3 8086:10c6
```

NIC yang digunakan oleh VPP/DPDK seharusnya menggunakan:

```text
Kernel driver in use: vfio-pci
```

### Cek HugePages

```bash
grep Huge /proc/meminfo
```

### Cek Status VPP

```bash
systemctl status vpp
```

### Cek Log VPP

```bash
journalctl -u vpp -n 100 --no-pager
```

### Cek Interface VPP

```bash
vppctl show interface
```

### Cek Error Counter

```bash
vppctl show errors
```

---

## Environment yang Telah Diuji

```text
OS       : Debian 12 Bookworm
Kernel   : 6.1.0-51-amd64
VPP      : v25.02-release
NIC      : Intel 82598EB 10GbE
PCI ID   : 8086:10c6
IOMMU    : Intel VT-d / DMAR
Mode     : Translated
```

> [!CAUTION]
> Patch ini ditujukan untuk **Intel 82598EB (`8086:10c6`)**. Jangan mengasumsikan patch akan bekerja dengan NIC Intel atau perangkat PCI lainnya tanpa pengujian terlebih dahulu.
