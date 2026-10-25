# Testing octeon_pip with the octeon-health test images

These images run the new OCTEON Ethernet driver (octeon_pip, OpenWrt PR #26060)
from RAM. Nothing is written to flash: a reboot or power cycle returns the
router to its installed system. After booting, one command (`octeon-health`)
collects everything needed to debug the driver remotely, runs a few benchmarks
with your PC, and produces one report to attach to the PR.

About 20–30 minutes in total, most of it waiting for benchmarks.

## What you need

- One of the supported routers: EdgeRouter Lite, EdgeRouter 4, EdgeRouter 6P,
  EdgeRouter 8 / Pro, UniFi Security Gateway, Itus Shield, Cisco vEdge 1000.
- A serial console (USB-serial adapter or Ubiquiti console cable, 115200 8N1).
  You need it to stop U-Boot and start the image.
- A PC with one Ethernet port: Linux is easiest; macOS and Windows work for
  everything except the forwarding test. A second Ethernet port on the PC is
  optional (forwarding test).
- To get the image into U-Boot, either a TFTP server on the PC or a
  FAT-formatted USB stick.

## 1. Get the image

From https://github.com/spolack/openwrt/releases/tag/octeon-health-latest
download `openwrt-octeon-generic-<device>-initramfs-kernel.bin` and `sha256sums`.

| Router | `<device>` |
|---|---|
| EdgeRouter Lite | `ubnt_edgerouter-lite` |
| EdgeRouter 4 | `ubnt_edgerouter-4` |
| EdgeRouter 6P | `ubnt_edgerouter-6p` |
| EdgeRouter 8, EdgeRouter Pro | `ubnt_edgerouter` |
| UniFi Security Gateway | `ubnt_unifi-usg` |
| Itus Shield | `itus_shield-router` |
| Cisco vEdge 1000 | `cisco_vedge1000` |

Check it: `sha256sum -c sha256sums --ignore-missing` (Linux), or compare
`shasum -a 256 <file>` / `certutil -hashfile <file> SHA256` with the
line in `sha256sums`.

The `*-sysupgrade.tar` files are for flashing and are not needed here.

## 2. Host setup

### Network

Connect the PC to a **LAN** port of the router:

| Router | LAN ports (connect the PC here) | WAN port |
|---|---|---|
| EdgeRouter Lite, USG, Itus Shield | eth1, eth2 | eth0 |
| EdgeRouter 4 | eth1–eth3 | eth0 |
| EdgeRouter 6P | eth1–eth5 | eth0 |
| EdgeRouter 8 / Pro | eth0 | eth1 |
| Cisco vEdge 1000 | see below | |

The router is `192.168.1.1` and serves DHCP on the LAN. Let the PC take an
address by DHCP, or set `192.168.1.10/24` by hand.

**Cisco vEdge 1000:** its default LAN is the management port `mgmt0`, which
uses a different driver (octeon_mgmt). Benchmarks over it don't test
octeon_pip. Connect the PC there to log in, then move the PC to a front port
and add it to the LAN bridge:
`uci add_list network.@device[0].ports='lan1'; uci commit network; /etc/init.d/network reload`
(see `ip link` for the port names).

**VPN:** disconnect it, or make sure `192.168.1.0/24` goes over the cable:
`ip route get 192.168.1.1` (Linux) must show your Ethernet interface.

### TFTP server (only if you boot over the network)

Put the image into the TFTP directory and start a server:

- **Linux:** `sudo dnsmasq --no-daemon --port=0 --enable-tftp --tftp-root=$PWD --listen-address=192.168.1.10`
  (or `tftpd-hpa` with the file in `/srv/tftp`). The PC needs the static address
  192.168.1.10, because U-Boot doesn't do DHCP here, and the firewall must allow
  UDP 69 (`sudo firewall-cmd --add-service=tftp`, or `sudo ufw allow 69/udp`).
- **macOS:** `sudo launchctl load -F /System/Library/LaunchDaemons/tftp.plist`;
  the directory is `/private/tftpboot`.
- **Windows:** Tftpd64 (https://pjo2.github.io/tftpd64/), set its directory and
  the 192.168.1.10 interface.

### iperf3

Install iperf3 on the PC: `sudo apt install iperf3` / `sudo dnf install iperf3`
/ `brew install iperf3` / Windows build from https://iperf.fr/iperf-download.php.

If the PC's firewall blocks incoming TCP 5201 (the default on many Windows and
Fedora setups), that's fine: `octeon-health` notices and turns the test around
(the router is the server, you paste one loop on the PC).

## 3. Boot the image from U-Boot

1. Open the serial console (115200 8N1), e.g. `picocom -b 115200 /dev/ttyUSB0`
   or PuTTY.
2. Power the router on and press a key (or Ctrl-C) as soon as U-Boot prints
   `Hit any key to stop autoboot`. You get a prompt such as `Octeon ubnt_e300#`.
3. Load the image, **either** over TFTP:
   ```
   setenv ipaddr 192.168.1.1
   setenv serverip 192.168.1.10
   tftpboot $(loadaddr) openwrt-octeon-generic-<device>-initramfs-kernel.bin
   ```
   **or** from a FAT USB stick:
   ```
   usb start
   fatload usb 0 $(loadaddr) openwrt-octeon-generic-<device>-initramfs-kernel.bin
   ```
4. Start it:
   ```
   bootoctlinux $(loadaddr) numcores=$(numcores) endbootargs mem=0
   ```
5. After about 30–60 s, press Enter on the console. The banner says
   `OCTEON test image: run octeon-health`.

Notes:
- U-Boot uses the port it calls `octeth0`/`ethprime` for TFTP, often a
  different one than the OpenWrt LAN. If `tftpboot` times out, try the PC on
  another port, or use the USB stick.
- Boards with a Qualcomm switch (QCA833x): if U-Boot's network doesn't work
  after a warm reboot from OpenWrt, power-cycle the router before using TFTP.
- On the EdgeRouter 8/Pro the kernel gets its device tree from the bootloader.
  That's expected.

## 4. Run octeon-health

Log in from the PC: `ssh root@192.168.1.1` (no password on the test image), or
use the serial console. Then:

```
octeon-health
```

It asks a few questions and tells you what to do on the PC:

1. **Snapshot:** a few seconds, no input.
2. **The IP address of your PC:** it suggests the one you are logged in from.
3. **Start `iperf3 -s` on your PC**, then press Enter. If the PC refuses the
   connection, it offers the other direction: answer `y`, press Enter, then
   paste the printed one-line loop (sh or PowerShell) on the PC. In that mode
   the router waits up to 5 minutes for the loop's first connection, and the
   loop must run to its end in the order printed: don't start it twice and
   don't stop it halfway. If the loop never connects, the remaining tests are
   skipped.
   The benchmarks take about 5 minutes on a 4-core router: tx and rx with 1
   and 4 streams, three runs each, and then the router's iperf3 on each CPU in
   turn. Don't use the network meanwhile.
   **Routers with a DSA switch** (the PC behind a switch port) get an A/B test
   on top, about 3 minutes more: it rewrites the parser's skip setting of the
   switch's conduit port (`PIP_PRT_CFG`) live, compares rx throughput, and
   restores the register afterwards, also when you press Ctrl-C. If the new
   setting breaks the link, it can drop for a few seconds; the test notices
   that and restores the old value. `NOAB=1` turns this test off.
4. **Forwarding test (optional, Linux PC with a second Ethernet port):** answer
   `y`, plug the second port into another LAN port, and run the printed
   commands (a network namespace with that port, about 7 commands, needs sudo).
   Then start the printed `iperf3 -c` command four times as it asks and press
   Enter right after (within about 10 s). The measurement starts when the
   router sees the traffic (it waits up to 30 s after Enter); if the client
   ended before the measurement did, the row says so. Each run takes about
   50 s. Answer `n` to skip.
5. **The report:** it prints `summary.md` and the `scp` line for the archive.

Get the archive from the PC (the `-O` matters, the router has no SFTP):

```
scp -O root@192.168.1.1:/tmp/octeon-health/octeon-health-*.tar.gz .
```

(On Windows: WinSCP with protocol SCP.)

Then post a comment in the PR with the archive attached and `summary.md` pasted
in. Say which router it is and, if you know it, which U-Boot version.

**What the archive contains** (it lists this in `CONTENTS.txt` too): the
summary, the run's output, the benchmark and forwarding results, and two
snapshots (after boot and at the start of the run) with system and kernel
info, `dmesg`, `logread`, network configuration and counters, `ethtool` output
per port, interrupts and pinning, the firewall's flowtables, the driver's
debugfs, the OCTEON registers and the device tree. Before it is packed, MAC
addresses are cut to their vendor part (`aa:bb:cc:xx:xx:xx`), public IPv4 and
global IPv6 addresses are masked, DHCP log lines are left out, and uci
passwords, keys and usernames are redacted; the neighbour and bridge FDB tables
are not captured. Private addresses (192.168.x.x and so on) stay. Have a look
before you post it if your setup is unusual.

The test image keeps everything in RAM: **download the archive before you
reboot.**

## 5. Optional: warm reboot

Problems after a warm reboot (as opposed to a power cycle) are the most useful
thing to report:

1. After the report above, type `reboot` on the console and boot the image
   again from U-Boot (step 3), this time **without** a power cycle.
2. Run `octeon-health` again and attach the second report too, noting "after
   warm reboot".
3. If U-Boot's TFTP or ping fails after the warm reboot, write that down: with
   the U-Boot output, it is a result too.

## Options

Set them in front of the command, e.g. `T=20 N=5 octeon-health`:

| Variable | Default | Meaning |
|---|---|---|
| `PEER` | asked | IP address of the PC |
| `T` | 10 | seconds per benchmark run (at least 5); the forwarding runs take 3×`T` |
| `N` | 3 | runs per benchmark |
| `NOPIN=1` | | skip the per-CPU sweep |
| `NOFWD=1` | | skip the forwarding test question |
| `NOBENCH=1` | | only the forwarding test |
| `NOAB=1` | | skip the DSA conduit parser test (the live register A/B test) |
| `PORT` | 5201 | iperf3 port |
| `FWD_PEER` | | forward to an iperf3 server you already routed yourself, instead of the namespace setup; works from the serial console too |
| `MODE=pcclient` | | the reverse mode (router as server) without asking; needed when running without a terminal, e.g. `ssh root@192.168.1.1 MODE=pcclient octeon-health` |

Without a terminal (a command over ssh, a script), `octeon-health` doesn't
wait at its prompts: it prints the instructions on stderr, gives the PC's
`iperf3 -s` 20 s to answer, and skips the reverse mode unless `MODE=pcclient`
is set.

`octeon-health snapshot` only takes the snapshot, without questions or
benchmarks.

## Troubleshooting

- **No ssh:** check the PC is on a LAN port (table above) and has a
  192.168.1.x address; use the serial console and `ip addr`. Run
  `octeon-health snapshot` there and copy the archive later.
- **`cannot reach iperf3`:** the PC firewall; answer `y` to the reverse mode.
- **`the PC's loop does not run: the remaining tests are skipped`:** in the
  reverse mode, no client came within the wait; run `octeon-health` again and
  paste the loop right after pressing Enter.
- **A forwarding run says `no traffic within 30 s`:** the `iperf3 -c` on the PC
  didn't start or doesn't go through the router; check
  `ip route get 10.99.0.2` on the PC.
- **The forwarding test says `10.99.0.2 does not answer`:** the second port
  is not in the namespace or not up; check `sudo ip -n ohc addr` on the PC.
  Clean up afterwards with `sudo ip netns del ohc; sudo ip route del 10.99.0.0/24`.
- **The router hangs or reboots:** the serial console output is the most
  valuable thing you can send: copy everything from `Linux version` on.
