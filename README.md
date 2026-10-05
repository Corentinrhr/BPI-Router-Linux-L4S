# Banana Pi Custom Linux Kernel (L4S Support)

This repository provides an automated GitHub Actions CI workflow to build custom Linux kernels for Banana Pi router boards, based on the [frank-w/BPI-Router-Linux](https://github.com/frank-w/BPI-Router-Linux) codebase. It compiles the kernel and packages the necessary boot images and modules for multiple boards, including the BPI-R2, BPI-R3, BPI-R4, BPI-R64, and BPI-R2Pro.

This specific build is customized to include L4S (Low Latency, Low Loss, and Scalable Throughput) features for modern Linux kernels.

---

## Installation Guide (BPI-R4 Example)

The following instructions explain how to install the newly compiled kernel on a Banana Pi BPI-R4 running a `frank-w` Debian/Ubuntu environment.

### 1. Download Release Artifacts
From the repository's **Releases** page, download the specific files for your board architecture. For the BPI-R4, you will need:
- `bpi-r4-<version>.itb` (The bootable Flattened Image Tree containing the kernel and device tree)
- `linux-image-<version>_arm64.deb` and `linux-headers-<version>_arm64.deb` (Kernel modules and headers)

*(Alternatively, download the `bpi-r4_<version>.tar.gz` archive to extract modules manually).*

### 2. Update the Boot Partition
U-Boot requires the `.itb` file to be placed in the `BPI-BOOT` partition, typically mounted at `/boot`. Always back up your working kernel before overwriting it.

```bash
sudo cp /boot/bpi-r4.itb /boot/bpi-r4.itb.bak
sudo cp bpi-r4-6.19.0-bpi-r4-l4s-routing-main.itb /boot/bpi-r4.itb
```

### 3. Install Kernel Modules
The safest way to install the accompanying kernel modules is via the Debian packages, ensuring they are correctly registered with `dpkg`.

```bash
sudo dpkg -i linux-image-6.19.0-bpi-r4-l4s-routing-main_*_arm64.deb
sudo dpkg -i linux-headers-6.19.0-bpi-r4-l4s-routing-main_*_arm64.deb
```

**Manual Extraction Alternative:**
If using the `.tar.gz` file instead of Debian packages, extract the modules directly into your system's library folder.
```bash
sudo tar -xzf bpi-r4_6.19.0-l4s-routing-main.tar.gz --strip-components=2 -C /lib/. BPI-ROOT/lib/
```

### 4. Reboot and Verify
Restart your router to boot into the newly installed L4S kernel.

```bash
sudo reboot
```

Once reconnected via SSH, verify the active kernel version and build date.

```bash
uname -a
```

---

## L4S Routing Setup & Testing

This section describes how to enable and test the L4S architecture on your BPI-R4 router. 

*(Note: Since TCP Prague is not natively included in the mainline kernel tree yet, this test uses **BBR** as the scalable TCP congestion control algorithm alongside **DualPI2** as the Active Queue Management (AQM) scheduler).*

### 1. Prerequisites and Module Activation

First, identify the target interface where you want to apply the testing rules (e.g., `wan` or `lan0`). Then, enable ECN and load the necessary kernel modules.

```bash
# 1. Identify the interface (ignore the '@' part, e.g., use 'wan' instead of 'wan@eth0')
ip -br a
INTERFACE=wan

# 2. Enable ECN (Explicit Congestion Notification) - Required for L4S
sudo sysctl -w net.ipv4.tcp_ecn=1

# 3. Load and activate the BBR TCP congestion control algorithm
sudo modprobe tcp_bbr
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# 4. Load the required queuing discipline (Qdisc) modules
sudo modprobe sch_htb
sudo modprobe sch_dualpi2
```

**Install Ookla Speedtest CLI (AArch64)**
You will need a reliable tool to generate network load. Download the official Ookla client for ARM64:
```bash
cd /tmp
wget https://install.speedtest.net/app/cli/ookla-speedtest-1.2.0-linux-aarch64.tgz
tar -xvzf ookla-speedtest-1.2.0-linux-aarch64.tgz
sudo mv speedtest /usr/local/bin/
```

### 2. Creating an Artificial Bottleneck

For DualPI2 to engage, network congestion must occur **locally on the router**. We will artificially limit the interface bandwidth to 40 Mbit/s using `htb`, and then attach `dualpi2` to manage the queuing.

```bash
# Limit the bandwidth to 40 Mbit/s on the chosen interface
sudo tc qdisc add dev $INTERFACE root handle 1: htb default 10
sudo tc class add dev $INTERFACE parent 1: classid 1:10 htb rate 40mbit

# Attach DualPI2 to manage the throttled traffic
sudo tc qdisc add dev $INTERFACE parent 1:10 handle 10: dualpi2
```

### 3. Testing Methodology

To validate L4S, we will run a stress test (Speedtest) while continuously measuring latency (Ping). The goal is to see stable ping times despite the line being completely saturated.

Open three separate SSH sessions/terminals to the router:

**Terminal 1: Latency Monitoring**
```bash
ping 8.8.8.8
```

**Terminal 2: DualPI2 Qdisc Monitoring**
```bash
watch -n 1 tc -s qdisc show dev wan
```

**Terminal 3: Load Generation**
```bash
speedtest --accept-license --accept-gdpr
```

### 4. Interpreting the Results

During the Speedtest, the `tc` command output in Terminal 2 should look similar to this:

```text
qdisc htb 1: root refcnt 2 r2q 10 default 0x10 direct_packets_stat 57 direct_qlen 1000
 Sent 46109310 bytes 74139 pkt (dropped 0, overlimits 29869 requeues 0)
 backlog 0b 11288p requeues 0
qdisc dualpi2 10: parent 1:10 [Unknown qdisc, optlen=112]
 Sent 46086920 bytes 73958 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
```

- **`overlimits 29869` (on htb):** The artificial bottleneck is working. Thousands of packets attempted to exceed the 40 Mbit/s limit and were queued instead of dropped.
- **`[Unknown qdisc, optlen=112]` (on dualpi2):** DualPI2 successfully processed the traffic (46 MB sent). The "Unknown qdisc" message is normal and expected; it simply means the current version of the `iproute2` (`tc`) utility installed on the OS does not yet know how to visually format DualPI2's advanced statistics (Classic vs L4S), even though the kernel is doing its job.
- **Expected Outcome:** In Terminal 1, the ping to 8.8.8.8 should remain stable without massive latency spikes during the entire Speedtest, confirming that the AQM successfully mitigated Bufferbloat using ECN.

### 5. Cleanup (Post-Test)

Once your tests are complete, you must remove the traffic control rules to lift the 40 Mbit/s restriction and restore your interface's full speed.

```bash
sudo tc qdisc del dev wan root
```
