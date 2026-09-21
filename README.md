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


# Verifying Instance Type and Nitro System Support

Before configuring ENA, verify whether the EC2 instance is running on the **AWS Nitro System**. The underlying instance architecture affects the available ENA networking capabilities and performance.

Most newer-generation EC2 instance types are Nitro-based, including:

* **General purpose:** M5, M5a, M5n, M6g, T3, and newer generations
* **Compute optimized:** C5, C5a, C5n, C6g, and newer generations
* **Memory optimized:** R5, R5a, R5n, R6g, and newer generations
* **Storage optimized:** D3, I3en, and newer generations

Older instance families such as **M4, C4, R4, and T2** are not based on the Nitro System and have more limited networking capabilities.

## Check the Instance Type

From inside the EC2 instance, you can check the instance type using the EC2 Instance Metadata Service:

```bash
TOKEN=$(curl -X PUT \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" \
  -s http://169.254.169.254/latest/api/token)

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  -s http://169.254.169.254/latest/meta-data/instance-type
```

Example:

```text
m6g.large
```

You can then check the AWS EC2 documentation to determine whether that instance family uses the Nitro System.

## Check ENA Support from the AWS CLI

If you know the instance ID:

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query "Reservations[].Instances[].{InstanceType:InstanceType,EnaSupport:EnaSupport}"
```

Example:

```text
[
    {
        "InstanceType": "m6g.large",
        "EnaSupport": true
    }
]
```

`EnaSupport: true` indicates that enhanced networking with ENA is enabled for the instance.

> **Note:** ENA support and Nitro System support are related but are not exactly the same thing. The instance family documentation is the authoritative source for determining the underlying instance architecture and supported networking capabilities.

Understanding the **instance type**, **Nitro System**, and **ENA support status** helps establish appropriate expectations for network performance and feature availability before applying the OS-level optimizations described in this guide.


# Implementing CloudWatch Monitoring for ENA

CloudWatch can be used to monitor EC2 network performance and identify when an instance is approaching or exceeding ENA networking limits.

A useful ENA monitoring setup should track:

* ENA allowance metrics
* Network throughput
* Network packet rates
* Instance resource utilization
* Application-specific network metrics

## Key ENA-Specific Metrics to Monitor

A CloudWatch dashboard should include widgets for ENA networking limits, network performance, and overall instance health.

### ENA Allowance Metrics

Important ENA-related metrics include:

* `bw_in_allowance_exceeded` — inbound bandwidth allowance exceeded
* `bw_out_allowance_exceeded` — outbound bandwidth allowance exceeded
* `pps_allowance_exceeded` — packet-per-second allowance exceeded
* `conntrack_allowance_exceeded` — connection-tracking allowance exceeded
* `linklocal_allowance_exceeded` — link-local service allowance exceeded

These metrics are useful for identifying situations where the instance is reaching an EC2 networking allowance.

### Network Throughput

Monitor:

* `NetworkIn` — bytes received by the instance
* `NetworkOut` — bytes transmitted by the instance

These metrics help determine whether the instance is approaching its available network bandwidth.

### Network Packets

Monitor:

* `NetworkPacketsIn` — packets received
* `NetworkPacketsOut` — packets transmitted

Packet rates are especially important for workloads that generate a large number of small packets.

### Instance Performance

Monitor:

* `CPUUtilization` — CPU usage
* Network-related ENA allowance metrics
* Instance-level resource utilization

High network traffic combined with high CPU utilization can indicate that the workload is becoming CPU-bound rather than network-bound.

### Custom Application Metrics

For applications with strict network-performance requirements, consider publishing custom CloudWatch metrics such as:

* Application network latency
* Request latency
* Network error rate
* Connection failures
* Retransmission rate
* Application-level packet processing time

---

# Creating a Comprehensive ENA Monitoring Dashboard

A well-designed CloudWatch dashboard provides at-a-glance visibility into network performance and ENA limits.

Create a dashboard using:

```bash
# Create a CloudWatch dashboard for ENA monitoring

aws cloudwatch put-dashboard \
  --dashboard-name "ENA-Performance-Dashboard" \
  --dashboard-body file://ena-dashboard.json
```

The `ena-dashboard.json` file defines the widgets displayed on the dashboard.

A typical dashboard layout could contain:

```text
+-------------------------------------------------------+
|                  ENA Performance                      |
+--------------------------+----------------------------+
| NetworkIn / NetworkOut   | NetworkPacketsIn/Out       |
+--------------------------+----------------------------+
| CPUUtilization           | ENA Bandwidth Allowance    |
+--------------------------+----------------------------+
| PPS Allowance            | Conntrack Allowance        |
+--------------------------+----------------------------+
| Application Latency      | Application Error Rate     |
+--------------------------+----------------------------+
```

This makes it easier to correlate network traffic with instance resource utilization and ENA allowance violations.

---

# Monitoring and Alerting on ENA-Specific Metrics

CloudWatch exposes EC2 networking allowance metrics that can indicate when an instance is reaching its networking limits.

### Bandwidth Allowance Exceeded

`bw_in_allowance_exceeded`

Indicates that inbound traffic exceeded the instance's available inbound bandwidth allowance.

`bw_out_allowance_exceeded`

Indicates that outbound traffic exceeded the instance's available outbound bandwidth allowance.

Increasing values can indicate that the workload is approaching the instance's network bandwidth limit.

### Packet-per-Second Allowance

`pps_allowance_exceeded`

Indicates that the instance exceeded its packet-per-second allowance.

This is particularly important for workloads generating many small packets.

For example:

```text
Large packets
    ↓
Fewer packets/sec
    ↓
Lower packet-processing overhead


Small packets
    ↓
More packets/sec
    ↓
Higher PPS requirement
```

An application can therefore experience networking limitations because of PPS even when total bandwidth is not saturated.

### Connection Tracking Allowance

`conntrack_allowance_exceeded`

Indicates that the instance exceeded its connection-tracking allowance.

This can be relevant for workloads with:

* Large numbers of concurrent connections
* High connection churn
* NAT-heavy traffic patterns
* Many short-lived TCP connections

### Link-Local Allowance

`linklocal_allowance_exceeded`

Indicates that the instance exceeded its allowance for link-local traffic.

This can be relevant when an application makes a large number of requests to AWS link-local services, such as the EC2 Instance Metadata Service.

---

# Create CloudWatch Alarms for ENA Metrics

CloudWatch alarms can proactively detect networking allowance problems.

## Alarm for High ENA Bandwidth Allowance

Example:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "ENA-Bandwidth-In-Allowance-Exceeded" \
  --metric-name "bw_in_allowance_exceeded" \
  --namespace "AWS/EC2" \
  --statistic "Sum" \
  --period 60 \
  --threshold 0 \
  --comparison-operator "GreaterThanThreshold" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:region:account-id:topic-name
```

This alarm triggers when the metric records values greater than zero for the configured evaluation periods.

## Alarm for Outbound Bandwidth Allowance

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "ENA-Bandwidth-Out-Allowance-Exceeded" \
  --metric-name "bw_out_allowance_exceeded" \
  --namespace "AWS/EC2" \
  --statistic "Sum" \
  --period 60 \
  --threshold 0 \
  --comparison-operator "GreaterThanThreshold" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:region:account-id:topic-name
```

## Alarm for PPS Allowance

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "ENA-PPS-Allowance-Exceeded" \
  --metric-name "pps_allowance_exceeded" \
  --namespace "AWS/EC2" \
  --statistic "Sum" \
  --period 60 \
  --threshold 0 \
  --comparison-operator "GreaterThanThreshold" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:region:account-id:topic-name
```

## Alarm for Connection Tracking

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "ENA-Conntrack-Allowance-Exceeded" \
  --metric-name "conntrack_allowance_exceeded" \
  --namespace "AWS/EC2" \
  --statistic "Sum" \
  --period 60 \
  --threshold 0 \
  --comparison-operator "GreaterThanThreshold" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:region:account-id:topic-name
```

> **Important:** The thresholds above are examples. A value greater than zero may be appropriate for detecting any allowance violation, but production alert thresholds should be based on the application's normal traffic pattern and the severity of the event.

---

# Recommended ENA Monitoring Strategy

A practical monitoring strategy should correlate multiple metrics rather than looking at a single metric in isolation.

For example:

```text
                Network Traffic
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
     NetworkIn/Out        NetworkPacketsIn/Out
          │                       │
          └───────────┬───────────┘
                      ↓
              ENA Allowances
                      │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
   Bandwidth         PPS          Conntrack
   exceeded        exceeded        exceeded
       │              │              │
       └──────────────┼──────────────┘
                      ↓
                Application
                  Metrics
                      │
                      ↓
             Latency / Errors
```

The goal is to determine **what resource is actually becoming the bottleneck**.

For example:

```text
High NetworkOut
+
bw_out_allowance_exceeded > 0
=
Possible bandwidth limitation
```

Whereas:

```text
Moderate NetworkOut
+
Very high NetworkPacketsOut
+
pps_allowance_exceeded > 0
=
Possible PPS limitation
```

And:

```text
High connection count
+
conntrack_allowance_exceeded > 0
=
Possible connection-tracking limitation
```

This correlation is more useful than monitoring `NetworkIn` or `NetworkOut` alone.

---

# Recommended Validation

After implementing CloudWatch monitoring, validate the configuration by generating representative network traffic and observing the metrics.

Check the EC2 instance locally:

```bash
# Network interface statistics
ip -s link show eth0

# ENA driver information
ethtool -i eth0

# ENA driver statistics
ethtool -S eth0
```

Check CloudWatch metrics:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkOut \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --statistics Average \
  --period 60 \
  --start-time 2026-01-01T00:00:00Z \
  --end-time 2026-01-01T01:00:00Z
```

For production monitoring, combine:

1. **Network throughput**
2. **Packet rate**
3. **ENA allowance metrics**
4. **CPU utilization**
5. **Application latency**
6. **Application error rates**

This allows you to distinguish between a bandwidth bottleneck, PPS limitation, connection-tracking limitation, CPU bottleneck, and application-level performance problem.
