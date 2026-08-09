# Rangkuman Perubahan File

## A. Fix LINK DOWN (carrier down padahal fisik hidup)

### 1. `/etc/vpp/startup.conf` (disedit)
Devarg `fiber_sdp3_no_tx_disable=1` untuk kedua port — menonaktifkan jalur false-down DPDK karena kartu Silicom tidak memasang pin SDP3:
```c
dpdk {
  dev 0000:01:00.0 { devargs fiber_sdp3_no_tx_disable=1 }
  dev 0000:01:00.1 { devargs fiber_sdp3_no_tx_disable=1 }
  uio-driver vfio-pci
}
```

### 2. `/etc/modprobe.d/vpp-vfio.conf` (baru)
```bash
# Claim 82598 10G NICs (8086:10c6) untuk vfio-pci saat boot
options vfio-pci ids=8086:10c6
```

### 3. `/etc/modules-load.d/vfio-pci.conf` (baru)
```bash
vfio-pci   # memuat driver vfio-pci saat boot
```

### 4. `/etc/udev/rules.d/99-vpp-vfio.rules` (baru)
```bash
# Bind 82598 (8086:10c6) ke vfio-pci di boot agar VPP bisa mengklaim
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x8086", ATTR{device}=="0x10c6", ATTR{driver_override}="vfio-pci"
```

---

## B. Fix RX-ERROR MILIARAN (bug DPDK)

### 5. `/opt/vpp-src/build/external/patches/dpdk_24.11.1/0002-net-ixgbe-fix-fcoe-crc-counters-on-82598.patch` (baru)
Patch 2 baris — memindahkan pembacaan register `FCCRC`/`FCLAST` ke dalam guard `if (hw->mac.type != ixgbe_mac_82598EB)`. Register ini tidak ada di chip 82598 → baca garbage → rx-error triliunan.

### 6. Hasil build: 10 file `.deb` di `/opt/vpp-src/build-root/` (baru)
Terutama 4 yang dipakai: `vpp`, `vpp-plugin-core`, `vpp-plugin-dpdk` (berisi patch), `libvppinfra`.

---

## C. Backup & Portabilitas

### 7. `/root/backup-vpp-2502/` (baru) — folder backup
`startup.conf`, 4 `.deb` patched, `paket-versi.txt`, patch file.

### 8. `/root/backup-vpp-2502-fix-rxerror.tar.gz` (baru) — arsip 15MB
Gabungan semua hal di atas untuk install di perangkat lain.

---

**Perangkat lunak (bukan file):** VPP **25.02-release** + DPDK **24.11.1** (downgrade dari 26.06/26.03 — sebab hang). Perlu diingat: file 1–4 sudah persisten di boot; file 5–6 hanya untuk build ulang (hasilnya sudah ter-embed di `.deb`).