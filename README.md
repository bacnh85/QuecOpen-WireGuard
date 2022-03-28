## What is `Wireguard` VPN

[WireGuard®](https://www.wireguard.com/)  is an extremely simple yet fast and modern VPN that utilizes state-of-the-art cryptography. It aims to be faster, simpler, leaner, and more useful than IPsec, while avoiding the massive headache. It intends to be considerably more performant than OpenVPN. WireGuard is designed as a general purpose VPN for running on embedded interfaces and super computers alike, fit for many different circumstances. Initially released for the Linux kernel, it is now cross-platform (Windows, macOS, BSD, iOS, Android) and widely deployable. It is currently under heavy development, but already it might be regarded as the most secure, easiest to use, and simplest VPN solution in the industry.

## Prepare

### Compiling and install `wireguard` kernel module

`Wireguard` was merged into the Linux kernel from 5.5, thus it is needed to integrate `Wireguard` as kernel module or build-in into kernels 3.10 to 5.5.

1) Source for SDK build environment

```console
cd ql-ol-sdk
source ql-ol-crosstool/ql-ol-crosstool-env-init
```

2) Integrate `WireGuard` kernel module into Linux kernel tree

In fact, `WireGuard` module can be built as module, however building WireGuard as build-in, directly within the SDK kernel tree is better option, for integration later.

Before doing that, a small patch is needed to ensure we can compile it:
```console
git clone https://git.zx2c4.com/wireguard-linux-compat
cd wireguard-linux-compat
git apply 001-fix-compiling-on-quecopen-sdk.patch
cd ..
./wireguard-linux-compat/kernel-tree-scripts/jury-rig.sh ql-ol-kernel
```

There will be a wireguard folder in the `ql-ol-kernel/build/wireguard`:

```console
$ tree ql-ol-kernel/build/wireguard/
ql-ol-kernel/build/wireguard/
`-- lib
    `-- modules
        `-- 3.18.20-g46733c7-dirty
            |-- build -> /opt/EC25-E/ql-ol-sdk/ql-ol-kernel/build
            |-- kernel
            |-- modules.alias
            |-- modules.alias.bin
            |-- modules.builtin.bin
            |-- modules.dep
            |-- modules.dep.bin
            |-- modules.devname
            |-- modules.softdep
            |-- modules.symbols
            |-- modules.symbols.bin
            `-- source -> /opt/EC25-E/ql-ol-sdk/ql-ol-kernel

6 directories, 9 files
```

3) Build kernel modules

Let's run `make kernel_menuconfig` and select `IP: WireGuard secure network tunnel` option, then all its dependencies are selected:
- CONFIG_NET for basic networking support
- CONFIG_INET for basic IP support
- CONFIG_NET_UDP_TUNNEL for sending and receiving UDP packets
- CONFIG_CRYPTO_ALGAPI for crypto_xor

```
[*] Networking support -->
    Networking options -->
        [*] TCP/IP networking
        [*]   IP: WireGuard secure network tunnel
        [ ]     Debugging checks and verbose messages
```

Then, build the kernel modules and rootfs:

```console
make kernel
make kernel_module
make rootfs
```

In the target folder, simple script can be used to flash both rootfs and kernel:

```bash
#!/bin/sh

adb reboot bootloader
sleep 10

fastboot flash system mdm9607-perf-sysfs.ubi
fastboot flash boot mdm9607-perf-boot.img

fastboot reboot
```

Then at EC25-E QuecOpen module, we can validate wireguard module is loaded together with the kernel:

```console
root@mdm9607-perf:/usrdata# cat /sys/module/wireguard/version
1.0.20211208

root@mdm9607-perf:~# dmesg | grep wireguard
[    0.754805] wireguard: WireGuard 1.0.20211208 loaded. See www.wireguard.com for information.
[    0.754816] wireguard: Copyright (C) 2015-2019 Jason A. Donenfeld <Jason@zx2c4.com>. All Rights Reserved.
```

### Compile and install the wg(8) tool

In the same terminal environment with previous session, doing below steps to compile `wireguard-tools`

1) Clone the source and build/install

```console
git clone https://git.zx2c4.com/wireguard-tools
make -C wireguard-tools/src
make -C wireguard-tools/src install DESTDIR=$PWD/wg-tools
```

The result is in the folder `wg-tools`:

```console
$ tree wg-tools
wg-tools
`-- usr
    |-- bin
    |   `-- wg
    `-- share
        |-- bash-completion
        |   `-- completions
        |       `-- wg
        `-- man
            `-- man8
                `-- wg.8
7 directories, 3 files
```

2) Integrate into SDK or target

The `wg` tool can copy over the taget:

```console
adb push wg-tools/usr/bin/wg /usr/bin
```

Or integrate into the `ql-ol-rootfs`:

```console
cp wg-tools/usr/bin/wg ql-ol-rootfs/sbin/
```

## Test `Wireguard` at target

### Setup `Wireguard` server

This can be done easily using [`wg-easy`](https://github.com/WeeJeWel/wg-easy) docker image - where you can put into your Linux host and open port `51820` for `Wireguard` clients connect to and `51821` for the web UI.

![Wireguard UI](https://raw.githubusercontent.com/WeeJeWel/wg-easy/master/assets/screenshot.png)

### Setup Wireguard config at QuecOpen target

Following steps can be done to setup Wireguard follows the generated Wireguard config from above step, with example:

