# ECOVACS DEEBOT NEO+ — firmware dump (fw 1.0.9)

Firmware images dumped from a retail **ECOVACS DEEBOT NEO+** robot vacuum
(Costco variant, product `zj2241_n10`, SoC Rockchip PX30 + RK1608).

| Field        | Value                       |
| ------------ | --------------------------- |
| Product      | `zj2241_n10`                |
| Firmware     | `1.0.9`                     |
| Build number | `38`                        |
| Release date | `2025-01-13`                |
| Platform     | Rockchip PX30, Android boot |
| Storage      | NAND (`rkflash0`), GPT      |

This is a **sanitised public subset** of a full device dump: the system
partitions are byte-for-byte copies, while partitions holding per-device
identity and user data are withheld (see *Withheld* below).

## How the dump was obtained

The robot exposes no ADB and no login shell. Root was obtained locally over the
UART console by interrupting U-Boot (`bootdelay=0`, CTRL+C burst) and booting
the stock kernel with `init=/bin/sh`. From that root shell the GPT partitions
were read with `dd` from `/dev/rkflash0pN`, staged on the `data` partition,
served with busybox httpd and pulled over the LAN.

## Partition map (GPT, LBA 512 B)

| #  | Name     | Size   | File             | Filesystem / type                                     |
| -- | -------- | ------ | ---------------- | ----------------------------------------------------- |
| 1  | `uboot`  | 4 MiB  | `p1_uboot.img`   | raw U-Boot                                            |
| 2  | `trust`  | 4 MiB  | `p2_trust.img`   | raw ATF / OP-TEE                                      |
| 3  | `mx`     | 4 MiB  | `p3_mx.img`      | raw (identical to p4)                                 |
| 4  | `my`     | 4 MiB  | `p4_my.img`      | raw (identical to p3)                                 |
| 5  | `sys`    | 2 MiB  | — withheld       | per-device info block                                 |
| 6  | `boot1`  | 6 MiB  | `p6_boot1.img`   | Android boot image (kernel + ramdisk), active slot     |
| 7  | `rootfs1`| 70 MiB | `p7_rootfs1.img` | SquashFS 4.0 (zlib), current rootfs, built 2025-01-13 |
| 8  | `boot2`  | 6 MiB  | `p8_boot2.img`   | Android boot image, backup slot (identical to p6)     |
| 9  | `rootfs2`| 70 MiB | `p9_rootfs2.img` | SquashFS 4.0 (zlib), backup rootfs, built 2023-03-23  |
| 10 | `data`   | 102 MiB| — withheld       | ext4 (`datafs`), writable user-data partition         |

`p3_mx` == `p4_my` and `p6_boot1` == `p8_boot2` (identical MD5).

## Withheld

Not published, because they contain personal/device-specific data:

- **`p5_sys`** — serial number at offset `0x100000` (two copies), device cloud
  ID + password, MAC address and calibration blobs.
- **`p10_data`** — ext4 `/data`: Wi-Fi credentials, certificates, account and
  session state, maps, logs, configuration.
- **`data_files.tgz`** — partial copy of `/data` (logs, autostart scripts).
- **`data_inventory.txt`** — directory listing of `/data`.
- **dump scripts** — lab network addresses.

`checksums/md5sums.txt` is the original dump manifest and still lists the MD5 of
every partition, including the withheld ones, so a full dump can be verified.
Only the published images are covered by `checksums/SHA256SUMS`.

## Verify and extract

```sh
sha256sum -c checksums/SHA256SUMS

# rootfs (squashfs)
unsquashfs -d rootfs1 images/p7_rootfs1.img
unsquashfs -d rootfs2 images/p9_rootfs2.img

# boot images
unpackbootimg -i images/p6_boot1.img -o boot1/
# kernel is LZ4-compressed; ramdisk is the second stage
```

Firmware metadata lives in `rootfs1/etc/fw.manifest`:

```json
{
  "release_date": "2025-01-13",
  "build_num": "38",
  "product": "zj2241_n10",
  "hw_ver": "1.0",
  "fw_ver": "1.0.9"
}
```

## Vendor secrets still present in the rootfs

These are shared across the firmware build (not per-user), but are sensitive:

- `/etc/www/self.key` — device HTTPS private key
- `/etc/www/auth.txt` — web/CGI credentials
- `/etc/conf/medusa/*.conf` — internal service password
- `/etc/conf/eco_passwd.conf` — password derivation config
- cloud CA material under `/etc/ca-certificates` and `/etc/iot-ca`

Treat the rootfs images as confidential material; do not reuse these keys.

## Legal

The firmware images are © ECOVACS / their respective vendors and are published
here for interoperability research and repair of hardware the author owns.
No license is granted. If you are a rights holder and want this removed, open an
issue.
