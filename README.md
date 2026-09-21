# AWS ENA Performance Optimization Guide

This guide covers operating-system-level optimizations that can be used to maximize the performance of **AWS Elastic Network Adapter (ENA)** on Linux and Windows EC2 instances.

> **Important:** These settings are workload- and environment-dependent. Benchmark before and after applying changes, and validate them in a non-production environment first. Not every workload will benefit from every optimization.

---

# Linux

Linux systems offer extensive networking configuration options that can be tuned to maximize ENA performance.

The main optimization categories are:

1. ENA Verification
2. Jumbo Frame Configuration
3. Kernel Parameter Optimization
4. IRQ Balance Optimization

---

## 1. Verify ENA

First, identify the network interface:

```bash
ip link
```

Then check whether the interface is using the ENA driver:

```bash
# Check if ENA is enabled on the network interface
ethtool -i eth0 | grep ena

# Expected output if ENA is enabled:
# driver: ena
# version: 2.6.0
# firmware-version:
# bus-info: 0000:00:05.0
# supports-statistics: yes
# supports-test: no
# supports-eeprom-access: no
# supports-register-dump: no
# supports-priv-flags: no
```

The most important line is:

```text
driver: ena
```

For the complete driver information:

```bash
ethtool -i eth0
```

> Replace `eth0` with the actual network interface name if necessary, such as `ens5`.

---

# 2. Jumbo Frame Configuration

Jumbo frames allow network packets larger than the traditional 1500-byte MTU.

AWS ENA supports jumbo frames with an MTU of up to **9001 bytes** on supported configurations.

Larger frames can reduce packet-processing overhead for large data transfers, potentially improving throughput and reducing CPU utilization.

### Configure Jumbo Frames

```bash
# Configure jumbo frames for better performance
sudo ip link set dev eth0 mtu 9001

# Verify the configuration
ip link show eth0 | grep mtu
```

Expected output should contain:

```text
mtu 9001
```

### Make the Configuration Persistent

For RHEL/CentOS/Amazon Linux configurations using traditional network scripts:

```bash
echo 'MTU=9001' | sudo tee -a /etc/sysconfig/network-scripts/ifcfg-eth0
```

For Ubuntu/Debian systems using `/etc/network/interfaces`:

```bash
echo 'MTU=9001' | sudo tee -a /etc/network/interfaces.d/eth0
```

> **Note:** Modern Ubuntu and other distributions may use **Netplan** or NetworkManager instead of `/etc/network/interfaces`. Verify which networking system your OS uses before making persistent configuration changes.

### Important

Jumbo frames require the entire network path to support the configured MTU.

For AWS VPC networking, jumbo frames are supported for traffic between supported resources, but MTU behavior can vary depending on the destination and network path.

---

# 3. Kernel Parameter Optimization

Linux kernel networking parameters can be adjusted for high-throughput workloads.

The following configuration increases TCP buffer limits, the network device backlog, and connection queues.

Add the following settings to `/etc/sysctl.conf`:

```bash
cat << EOF | sudo tee -a /etc/sysctl.conf

# Increase TCP maximum receive buffer size
net.core.rmem_max = 16777216

# Increase TCP maximum send buffer size
net.core.wmem_max = 16777216

# Increase TCP receive buffer auto-tuning limits
net.ipv4.tcp_rmem = 4096 87380 16777216

# Increase TCP send buffer auto-tuning limits
net.ipv4.tcp_wmem = 4096 65536 16777216

# Increase the length of the processor input queue
net.core.netdev_max_backlog = 30000

# Increase the maximum number of connections
net.core.somaxconn = 1024

# Increase the maximum number of SYN requests allowed
net.ipv4.tcp_max_syn_backlog = 1024

# For high-throughput, low-latency environments, TCP cookies
# may be disabled if appropriate for the security environment.
#
# WARNING: Only disable TCP SYN cookies when you understand
# the security implications for your workload.
net.ipv4.tcp_syncookies = 0

EOF
```

Apply the changes without rebooting:

```bash
sudo sysctl -p
```

### Verify the Parameters

```bash
sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.core.netdev_max_backlog
sysctl net.core.somaxconn
```

> **Warning:** Kernel networking parameters should not be changed blindly. Values that improve throughput for one workload may have little effect—or have negative effects—on another. Test under realistic traffic.

---

# 4. IRQ Balance Optimization

Network packets generate interrupts that need to be processed by CPU cores.

If networking interrupts become concentrated on a small number of CPU cores, those CPUs can become bottlenecks.

**irqbalance** helps distribute hardware interrupts across available CPU cores.

### Install irqbalance

For RHEL/CentOS/Amazon Linux:

```bash
sudo yum install -y irqbalance
```

For Ubuntu/Debian:

```bash
sudo apt-get install -y irqbalance
```

### Enable and Start the Service

```bash
sudo systemctl enable irqbalance
sudo systemctl start irqbalance
```

### Verify

```bash
sudo systemctl status irqbalance
```

You can also inspect interrupt distribution with:

```bash
cat /proc/interrupts
```

---

# Windows

Windows systems use different networking configuration mechanisms to optimize ENA performance.

The main areas covered here are:

1. ENA Verification
2. Jumbo Frame Configuration
3. Receive Side Scaling (RSS)
4. TCP Optimization

---

# 1. Verify ENA

PowerShell can be used to check whether the Amazon ENA network adapter is present.

```powershell
# Check Network Interface
Get-NetAdapter | Where-Object {$_.InterfaceDescription -like "*Elastic Network Adapter*"}
```

Expected output:

```text
Name      InterfaceDescription             ifIndex Status MacAddress           LinkSpeed
----      --------------------             ------- ------ ----------           ---------
Ethernet  Amazon Elastic Network Adapter   12      Up     0A-1B-2C-3D-4E-5F   10 Gbps
```

The important field is:

```text
InterfaceDescription
    Amazon Elastic Network Adapter
```

You can also inspect the adapter with:

```powershell
Get-NetAdapter
```

---

# 2. Jumbo Frame Configuration

Jumbo frames can reduce packet-processing overhead for high-bandwidth workloads.

The following command configures an MTU of 9001:

```powershell
Set-NetAdapterAdvancedProperty `
    -Name "Ethernet" `
    -RegistryKeyword "MTU" `
    -RegistryValue 9001
```

Verify the configuration:

```powershell
Get-NetAdapterAdvancedProperty `
    -Name "Ethernet" `
    -RegistryKeyword "MTU"
```

> **Note:** The exact adapter property and supported values can vary by ENA driver version and Windows configuration. If the `MTU` registry keyword is not available, inspect the adapter's advanced properties with `Get-NetAdapterAdvancedProperty -Name "Ethernet"` and use the property exposed by the installed driver.

---

# 3. Receive Side Scaling (RSS)

**Receive Side Scaling (RSS)** distributes network-processing work across multiple CPU cores.

Without effective RSS, network processing can become concentrated on a limited number of CPUs.

Check the current RSS configuration:

```powershell
Get-NetAdapterRss
```

Enable RSS for an adapter:

```powershell
Enable-NetAdapterRss -Name "Ethernet"
```

Verify:

```powershell
Get-NetAdapterRss -Name "Ethernet"
```

RSS can be particularly useful for high-throughput workloads running on instances with multiple CPU cores.

---

# 4. TCP Optimization

Windows provides `netsh` commands for inspecting and configuring TCP behavior.

Configure TCP settings:

```powershell
# Configure TCP auto-tuning
netsh int tcp set global autotuninglevel=normal

# Enable Receive Side Scaling
netsh int tcp set global rss=enabled

# Enable TCP Chimney Offload
netsh int tcp set global chimney=enabled

# Enable Direct Cache Access
netsh int tcp set global dca=enabled
```

Verify the configuration:

```powershell
netsh int tcp show global
```

> **Important:** TCP Chimney Offload and Direct Cache Access are legacy Windows networking features and may not be available or applicable on modern Windows Server versions and ENA drivers. Do not force these settings on current systems without first confirming that the feature is supported and beneficial for your workload.

For modern Windows systems, focus primarily on **RSS, receive/send buffers, adapter capabilities, and ENA driver versions**, and validate performance with benchmarks.

---

# Performance Benefits

When correctly applied to suitable workloads, ENA and OS-level network optimizations can provide benefits such as:

* Higher throughput for network-intensive applications
* Reduced packet drops under heavy load
* Better handling of connection bursts
* Lower CPU utilization for network processing
* More consistent performance under varying workloads
* Improved performance for large data transfers

The actual improvement depends heavily on:

* EC2 instance type
* ENA driver version
* CPU architecture
* Operating system
* Application workload
* Packet size
* Network throughput requirements
* Number of concurrent connections
* Availability Zone and network path
* Whether the workload is CPU- or network-bound

Claims such as **15–20% throughput improvement should be treated as workload-specific benchmark results, not guaranteed improvements**.

---

# Recommended Validation Process

Before applying these settings to production, establish a baseline.

### 1. Check the current configuration

```bash
ethtool -i eth0
ip link show eth0
```

### 2. Measure network performance

Use an appropriate benchmark such as `iperf3` between instances.

```bash
iperf3 -s
```

On the client:

```bash
iperf3 -c <server-ip>
```

### 3. Monitor CPU utilization

```bash
top
```

or:

```bash
mpstat -P ALL
```

### 4. Monitor network statistics

```bash
ip -s link show eth0
```

and:

```bash
ethtool -S eth0
```

### 5. Apply one optimization at a time

For example:

```text
Baseline
   ↓
Enable jumbo frames
   ↓
Benchmark
   ↓
Adjust kernel parameters
   ↓
Benchmark
   ↓
Configure IRQ balancing
   ↓
Benchmark
```

This makes it possible to determine which optimization actually benefits the workload.

---

# Summary

The relationship between the AWS components and OS configuration can be viewed as:

```text
                    AWS EC2
                       │
                ┌──────┴──────┐
                │             │
             Nitro           CPU
                │
             ENA
                │
        ┌───────┴────────┐
        │                │
      Linux            Windows
        │                │
   ENA driver        ENA driver
        │                │
   ┌────┴─────┐     ┌────┴─────┐
   │          │     │          │
 Jumbo      IRQ   Jumbo       RSS
 Frames    Balance Frames
   │                │
   └────────┬───────┘
            │
       TCP/IP Stack
            │
       Application
```

**ENA** provides the high-performance network interface.

**Nitro** provides the underlying AWS hardware infrastructure and virtualization architecture.

**Jumbo frames** reduce packet overhead.

**Kernel/TCP tuning** adjusts how the operating system handles network traffic.

**IRQ balancing/RSS** distributes network-processing work across CPU cores.

Together, these mechanisms can help an EC2 instance make better use of its available network performance.
