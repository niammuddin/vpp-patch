# Build VPP 25.02 + Patch DPDK 24.11.1 (Fix rx-error 82598EB)

## 1. Prasyarat
- Debian 12 (bookworm) **amd64**, login sebagai **root**
- NIC Intel **82598EB** (8086:10c6), VT-d/IOMMU aktif
- Disk luang ±25 GB, RAM ≥8 GB (build berat)

## 2. Clone source VPP 25.02
`gitlab.fd.io` tidak resolve di jaringan kita, jadi pakai **mirror GitHub**:

```bash
git clone --branch v25.02 --depth 1 https://github.com/FDio/vpp.git /opt/vpp-src
cd /opt/vpp-src
```

## 3. Buat patch DPDK (inti dari semua ini)
Buat file **`/opt/vpp-src/build/external/patches/dpdk_24.11.1/0002-net-ixgbe-fix-fcoe-crc-counters-on-82598.patch`** berisi:

```diff
diff --git a/drivers/net/ixgbe/ixgbe_ethdev.c b/drivers/net/ixgbe/ixgbe_ethdev.c
index 8bee97d..aaba546 100644
--- a/drivers/net/ixgbe/ixgbe_ethdev.c
+++ b/drivers/net/ixgbe/ixgbe_ethdev.c
@@ -3295,10 +3295,10 @@ ixgbe_read_stats_registers(struct ixgbe_hw *hw,
 	hw_stats->ptc1522 += IXGBE_READ_REG(hw, IXGBE_PTC1522);
 	hw_stats->bptc += IXGBE_READ_REG(hw, IXGBE_BPTC);
 	hw_stats->xec += IXGBE_READ_REG(hw, IXGBE_XEC);
-	hw_stats->fccrc += IXGBE_READ_REG(hw, IXGBE_FCCRC);
-	hw_stats->fclast += IXGBE_READ_REG(hw, IXGBE_FCLAST);
 	/* Only read FCOE on 82599 */
 	if (hw->mac.type != ixgbe_mac_82598EB) {
+		hw_stats->fccrc += IXGBE_READ_REG(hw, IXGBE_FCCRC);
+		hw_stats->fclast += IXGBE_READ_REG(hw, IXGBE_FCLAST);
 		hw_stats->fcoerpdc += IXGBE_READ_REG(hw, IXGBE_FCOERPDC);
 		hw_stats->fcoeprc += IXGBE_READ_REG(hw, IXGBE_FCOEPRC);
 		hw_stats->fcoeptc += IXGBE_READ_REG(hw, IXGBE_FCOEPTC);
```

> **Catatan penting:** format harus persis `git diff` (prefix `a/` + `b/`). Versi "rapi manual" tadi membuat error `malformed patch`. Kalau mau ubah, edit langsung di `drivers/net/ixgbe/ixgbe_ethdev.c` lalu `git diff > 0002-....patch`.

## 4. Install build dependencies
```bash
make install-dep
```
Ini otomatis `apt-get update` + install semua paket build. Jika ada dpkg nyangkut: `dpkg --configure -a`. Jika error paket `iperf3`: install ulang terpisah, lalu ulangi.

## 5. Build external deps (DPDK 24.11.1, xdp-tools, rdma-core, quicly)
```bash
make install-ext-deps
```
Yang terjadi:
- Download tarball (DPDK 24.11.1, xdp-tools 1.2.9, rdma-core 55.0, quicly, ipsec-mb 2.0) ke `build/external/downloads/`
- Patch `0001` & `0002` diaplikasikan otomatis ke DPDK → terverifikasi di log:
  ```
  Applying patch: 0002-net-ixgbe-fix-fcoe-crc-counters-on-82598.patch
  patching file drivers/net/ixgbe/ixgbe_ethdev.c
  ```
- Build DPDK dengan **meson 0.57.2** (di-venv) — config disable sebagian besar driver, statis
- Hasil: paket **`vpp-ext-deps_25.02-0_amd64.deb`** terinstall ke `/opt/vpp/external/`

### Fix yang kita temui di langkah ini
1. **Butuh clang/llvm** untuk xdp-tools:
   ```bash
   apt-get install -y clang llvm libbpf-dev
   ```
2. **`libbpf.a` tanpa `-fPIC`** → link plugin af_xdp gagal nanti di langkah 6. Fix: rebuild `libbpf.a` dengan `-fPIC` lalu timpa:
   ```bash
   cd /tmp/opencode/xdp-tools-1.2.9/lib/libbpf/src
   make BUILD_STATIC_ONLY=y EXTRA_CFLAGS="-fPIC"
   cp libbpf.a /opt/vpp/external/x86_64/lib64/libbpf.a
   ```

## 6. Build VPP + generate .deb
```bash
make pkg-deb
```
Menghasilkan **10 paket .deb** di `/opt/vpp-src/build-root/`:
`vpp`, `vpp-plugin-core`, `vpp-plugin-dpdk`, `libvppinfra`, `libvppinfra-dev`, `vpp-dev`, `vpp-dbg`, `vpp-crypto-engines`, `vpp-plugin-devtools`, `python3-vpp-api`.

## 7. Install paket (4 esensial)
```bash
dpkg -i /opt/vpp-src/build-root/libvppinfra_25.02-release_amd64.deb \
       /opt/vpp-src/build-root/vpp_25.02-release_amd64.deb \
       /opt/vpp-src/build-root/vpp-plugin-core_25.02-release_amd64.deb \
       /opt/vpp-src/build-root/vpp-plugin-dpdk_25.02-release_amd64.deb
```
Jika butuh dependensi sistem: `apt-get install -y libnuma1 libnl-3-200 libnl-route-3-200 libunwind8 libssl3 libpcap0.8 libelf1`

## 8. Konfigurasi (link-up + persistensi boot)
1. **`/etc/vpp/startup.conf`** — bagian `dpdk`:
   ```
   dpdk {
     dev 0000:01:00.0 { devargs fiber_sdp3_no_tx_disable=1 }
     dev 0000:01:00.1 { devargs fiber_sdp3_no_tx_disable=1 }
     uio-driver vfio-pci
   }
   ```
2. **Hugepages** (persisten di `/etc/sysctl.d/`):
   ```bash
   echo "vm.nr_hugepages=1024" > /etc/sysctl.d/99-vpp-hugepages.conf
   sysctl -p
   ```
3. **Binding vfio-pci saat boot** (3 file):
   - `/etc/modprobe.d/vpp-vfio.conf`: `options vfio-pci ids=8086:10c6`
   - `/etc/modules-load.d/vfio-pci.conf`: `vfio-pci`
   - `/etc/udev/rules.d/99-vpp-vfio.rules`: bind `0000:01:00.0/.1` ke `vfio-pci`
4. **NIC Linux harus down** sebelum start VPP: `ip link set enp1s0f0 down` (dan `.1`)

## 9. Verifikasi
```bash
vppctl show version          # built by root ... 25.02-release
vppctl show interfaces       # rx-error hilang (tidak muncul lagi)
ping ...                      # 0% loss, link up 10 Gbps
```

---

**Ringkasan perintah inti:** `clone → buat patch 0002 → make install-dep → make install-ext-deps → make pkg-deb → dpkg -i → konfigurasi startup.conf + vfio`.

Semua ini sudah terarsip di `/root/backup-vpp-2502-fix-rxerror.tar.gz` (patch + 4 .deb + startup.conf + paket-versi.txt).