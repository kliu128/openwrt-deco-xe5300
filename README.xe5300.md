# OpenWrt on the TP-Link Deco XE5300 (hardware 2.0)

Work-in-progress OpenWrt port for the TP-Link Deco XE5300, a tri-band Wi-Fi 6E
mesh node built on Qualcomm IPQ5018 with two QCN6122 radios and a Realtek
RTL8367S-VB switch.

This is not an official OpenWrt target and is not supported by TP-Link or
OpenWrt. Flashing it can leave a unit unbootable. Everything below describes
what was observed on a small number of hardware 2.0 units; other revisions are
untested.

## Branches

| Branch | Base | What it is |
|---|---|---|
| `xe5300` | upstream OpenWrt `main` | Plain port: device tree, image recipe, board files, switch driver patches. |
| `xe5300-nss` | [kuncy7/openwrt-nss-edma](https://github.com/kuncy7/openwrt-nss-edma) `c3po-tag-8021q` | Same device on the community IPQ50xx NSS tree, plus two fixes for NSS mesh offload. |

Both branches only add the device and its fixes on top of their base. All
other history belongs to the respective upstream projects.

## Hardware

| Part | Detail |
|---|---|
| SoC | Qualcomm IPQ5018, two Cortex-A53 cores |
| RAM / flash | 512 MB / 128 MB NAND |
| 2.4 GHz | IPQ5018 internal radio |
| 5 GHz, 6 GHz | two QCN6122, through the IPQ5018 Q6 (ath11k AHB, multi-PD) |
| Ethernet | three ports on an RTL8367S-VB switch (`lan1` to `lan3`, DSA) |
| Device name | `tplink,deco-xe5300` |

## What works

Observed on hardware:

- Boot from NAND through the stock bootloader, ARM64, both CPUs.
- All three Ethernet ports with DSA; one port used as WAN.
- All three radios as access points (2.4 GHz HE20, 5 GHz HE80 on channel 36,
  6 GHz HE80), WPA2/WPA3, 802.11r.
- 802.11s mesh backhaul on 6 GHz across three nodes, two hops.
- `sysupgrade` in place, settings kept, including from an `xe5300-nss`
  image to an `xe5300` image.
- On `xe5300`: software flow offloading (`flow_offloading '1'`) with one
  port as WAN, wired and Wi-Fi clients; this needs the firewall4 version
  on this branch, which lists the bridge member devices in the flowtable.
- Switching between the OpenWrt slot and the stock slot and back.
- On `xe5300-nss`: NSS offload with ECM (about 540 Mbit/s routed
  to a Wi-Fi client at near-idle CPU in one home test).

Not tested, or tested only by reading the source: VLANs, the button and LEDs
(GPIO numbers come from the stock device tree), failsafe mode, TP-Link's
web recovery, hardware revisions other than 2.0.

## Flash layout and the two slots

The stock bootloader keeps two firmware slots and picks one from the U-Boot
environment variable `tp_boot_idx`. If the chosen slot fails to load, it
falls back to the other. OpenWrt lives in one slot as UBI volumes `kernel`
(FIT), `rootfs` (squashfs) and `rootfs_data`; the stock firmware can stay in
the other.

`sysupgrade` from a running OpenWrt rewrites the slot OpenWrt is running
from and never the other one. A first install from an initramfs boot writes
the slot the bootloader is not currently set to boot and then selects it.

The port never writes the bootloader, the calibration partition (`0:ART`),
`factory_data` or `runtime_data`. Radio calibration and MAC addresses are
read from those partitions at boot.

## Installing

A first install needs a way to boot the initramfs image on the unit (serial
console and TFTP from the bootloader). That is not covered here. Once the
initramfs is running:

```sh
sysupgrade -n /tmp/openwrt-qualcommax-ipq50xx-tplink_deco-xe5300-squashfs-sysupgrade.bin
```

Back up every flash partition before the first write, and keep the copies of
`0:ART`, `factory_data` and `runtime_data` somewhere safe: they are unique
to each unit.

## Returning to stock

If the stock firmware is still in the other slot:

```sh
fw_setenv tp_boot_idx 1   # or 0, whichever slot holds stock
reboot
```

While stock is running, do not let it install a firmware update: stock
updates write the slot it is not running from, which is the OpenWrt slot.

## Building

`xe5300`:

```sh
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig   # Qualcomm Atheros 802.11ax WiSoC-s / ipq50xx / TP-Link Deco XE5300
make -j"$(nproc)"
```

`xe5300-nss` additionally needs the NSS package feed used by the base tree
and `CONFIG_ATH11K_NSS_MESH_SUPPORT=y` if 802.11s is wanted with Wi-Fi
offload. See the base tree's `README.ipq50xx.md`.

The Wi-Fi firmware package is `ath11k-firmware-ipq5018-qcn6122-2.5`
(WLAN.HK.2.5.0.1). It is the only version found to accept this board's
board data files.

## Known issues

- **`clk_ignore_unused` is required.** Without it the Q6 does not start
  after a cold boot. It is set in the device tree's `bootargs-append`.
- **5 GHz sometimes fails its first start after boot.** The radio comes up
  when restarted. ath11k waits 1 s for vdev setup, which appears to be too
  short for this radio. `xe5300` carries a patch raising it to 5 s, as in
  ath10k and ath12k; the first boots of three units with it were clean, which
  is too few to call it fixed.
- **5 GHz HE80 on U-NII-3 (channel 149 and up) does not start.** HE20, HE40
  and VHT80 do. Channel 36 HE80 works.
- **Mesh metric warning (`xe5300-nss`).** On the NSS tree the driver
  reports no transmit rate for mesh peers, so `airtime_link_metric_get()`
  logs a WARN with a stack trace several times a second. At the default
  console log level this stalls the CPU on the serial console and causes
  latency spikes of hundreds of milliseconds. Workaround:
  `sysctl -w kernel.printk="1 4 1 7"`. Not seen on `xe5300`, where the
  transmit rate is reported; that branch still turns the WARN into a
  one-time warning.
- **Software flow offloading with the `rtl8365mb-8021q` tagger
  (`xe5300-nss`).** With bridged switch ports in the flowtable, a frame
  from a bridged port keeps `skb->offload_fwd_mark` set; when the flowtable
  transmits it on a standalone port, `rtl8365mb_8021q_xmit()` passes a NULL
  bridge to `br_vlan_enabled()` and the kernel panics. Workaround: keep
  bridged switch ports out of the flowtable. Not fixed in this tree.
- **NSS Wi-Fi offload with 802.11s relaying (`xe5300-nss`).** Two bugs are
  fixed here (`0004` forwarding setting inverted, `0005` Ethernet header
  lost on path-not-found frames). Others remain, for example the offload
  path learns a node's own stations as proxied behind a relay. Wi-Fi offload
  is best left off on relaying mesh nodes.

## Licence

The OpenWrt build system and the changes here are under GPL-2.0, as
upstream. The board data files under `package/firmware/ipq-wifi/files/`
come from the vendor firmware and are redistributed as for other boards in
that directory.
