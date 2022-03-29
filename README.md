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

If the debug is enabled, we can see all tests are passed:
```
root@mdm9607-perf:/usrdata# dmesg | grep wireguard
[    0.781861] wireguard: chacha20 self-tests: pass
[    0.808922] wireguard: poly1305 self-tests: pass
[    0.810708] wireguard: chacha20poly1305 self-tests: pass
[    0.813388] wireguard: blake2s self-tests: pass
[    1.014296] wireguard: curve25519 self-tests: pass
[    1.014528] wireguard: allowedips self-tests: pass
[    1.022474] wireguard: nonce counter self-tests: pass
[    1.291262] wireguard: ratelimiter self-tests: pass
[    1.291547] wireguard: WireGuard 1.0.20211208 loaded. See www.wireguard.com for information.
[    1.291558] wireguard: Copyright (C) 2015-2019 Jason A. Donenfeld <Jason@zx2c4.com>. All Rights Reserved.
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

### Setup client side

From the server side, let's say generated config as below:
```
[Interface]
Address = 192.168.100.3/32
PrivateKey = cBoch+dZhWWUfiKzolfruUGdj+mxTY/EqVHJQACb/EU=

[Peer]
PublicKey = GPdmUWIOWy+e5KeQ8h6+T7ivGBzDj+A8I/U67dIQ6go=
PresharedKey = p+ssLtghcZziJ/PwxiBWVb2oBFsA/aXslRG0Wd5iQT0=
AllowedIPs = 0.0.0.0/0
Endpoint = vpn.bacnh.com:51820
PersistentKeepalive = 15
```

Due to the current QuecOpen is not using `bash` so we can't use the `wg-quick` directly to load the config, however those configs can be manually created:
- Wireguard interface with IP: 192.168.100.3/32 and private key: `cBoch+dZhWWUfiKzolfruUGdj+mxTY/EqVHJQACb/EU=`
- Add a peer with endpoint: `vpn.bacnh.com:51820` which is the Public IP of the server, and with the publickey `GPdmUWIOWy+e5KeQ8h6+T7ivGBzDj+A8I/U67dIQ6go=` and PreshareKey `p+ssLtghcZziJ/PwxiBWVb2oBFsA/aXslRG0Wd5iQT0=` as higher security level. And also add PersistentKeepalive every 15sec to ensure the connection remains open.

Let's store the private key and share key into files:
```
echo cBoch+dZhWWUfiKzolfruUGdj+mxTY/EqVHJQACb/EU= > privatekey
echo p+ssLtghcZziJ/PwxiBWVb2oBFsA/aXslRG0Wd5iQT0= > sharekey
```

We can setup `wireguard` instance:
```console
## Create interface
ip link add type wireguard
## Add IP
ip addr add 192.168.100.3/32 dev wireguard0

## Setup private key for the client
wg set wireguard0 listen-port 51871 private-key privatekey

## Add a the server as a peer to this module
wg set wireguard0 peer GPdmUWIOWy+e5KeQ8h6+T7ivGBzDj+A8I/U67dIQ6go= preshared-key sharekey endpoint vpn.bacnh.com:51820 allowed-ips 0.0.0.0/0

## Ensure the connection remains open
wg set wireguard0 peer GPdmUWIOWy+e5KeQ8h6+T7ivGBzDj+A8I/U6
7dIQ6go= persistent-keepalive 15

## Add the routing table to access the peer
ip route add 192.168.100.0/24 dev wireguard0

## The bring up the interface
ip link set wireguard0 up
```

There will be a interface `wireguard0` with IP `192.168.100.3`:

```
root@mdm9607-perf:/usrdata# ifconfig wireguard0
wireguard0 Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
          inet addr:192.168.100.3  P-t-P:192.168.100.3  Mask:255.255.255.255
          UP POINTOPOINT RUNNING NOARP  MTU:1420  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Then, we can see the client is connected to the server via `wg` tool:

```
root@mdm9607-perf:/usrdata# wg show
interface: wireguard0
  public key: XfVGbAM2igd/PVYEK9nc/27CLfrfNyMXyPgFahr3UTg=
  private key: (hidden)
  listening port: 51871

peer: GPdmUWIOWy+e5KeQ8h6+T7ivGBzDj+A8I/U67dIQ6go=
  preshared key: (hidden)
  endpoint: 113.190.95.242:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 1 minute, 16 seconds ago
  transfer: 7.18 KiB received, 7.85 KiB sent
  persistent keepalive: every 15 seconds
  ```

As `wireguard` is working in QuecOpen, further tweak needs to be done to fit customer's usage.

PS: This `wireguard` config is used for demoistration purpose only and it may not work at your side.