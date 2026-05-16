# Windows 11 Network Optimization Guide for Gaming

A concise, prioritized reference for reducing network latency on modern gaming PCs. Every recommendation is sourced from Microsoft documentation, the GamingPCSetup project, or established professional consensus. Apply changes one at a time and test between each.

**Requirements:** Windows 11, a wired Ethernet connection, a modern multi-core CPU (4+ cores), 16 GB+ RAM, and Administrator access. All PowerShell and netsh commands must be run in an elevated (Admin) terminal.

---

## 0 — Before You Start

### Find Your Adapter Name

Every command in this guide uses a variable `$adapter`. Set it once:

```powershell
# List all adapters — note the Name of your wired NIC
Get-NetAdapter | Format-Table Name, InterfaceDescription, LinkSpeed

# Set the variable (replace with your adapter's name)
$adapter = "Ethernet"
```

### Snapshot Your Current Configuration

Save this output to a file before changing anything, so you can always revert:

```powershell
Get-NetAdapterAdvancedProperty -Name $adapter | Format-Table -AutoSize | Out-File "$env:USERPROFILE\Desktop\nic_backup.txt"
netsh int tcp show global >> "$env:USERPROFILE\Desktop\nic_backup.txt"
Get-NetOffloadGlobalSetting | Out-File -Append "$env:USERPROFILE\Desktop\nic_backup.txt"
```

### Reset to Defaults (if needed later)

```powershell
Reset-NetAdapterAdvancedProperty -Name $adapter -DisplayName "*"
```

---

## 1 — NIC Adapter Settings (High Impact)

These are the most consistently impactful changes for reducing gaming latency. They modify how the network adapter processes packets.

> **Note on offloads:** Many guides from the early 2010s recommend disabling *all* offloads to save CPU. On any modern multi-core CPU, that reasoning is obsolete — the CPU overhead is trivial. The reason to disable specific offloads today is that they **add buffering latency**, not because they consume CPU. Checksum offloads, for example, add no buffering and should be left enabled.

### 1.1 — Interrupt Moderation → Disabled

Batches incoming packets into groups before generating a CPU interrupt. Reduces CPU load at the cost of added latency on every packet. Microsoft's own tuning guide states: *"Disable the Interrupt Moderation setting for network card drivers that require the lowest possible latency."*

If fully disabling causes micro-stuttering under heavy combined load (GPU + audio + USB + network), set to `Medium` as a compromise — testing by the GamingPCSetup project found `Medium` provides nearly identical DPC latency with less interrupt storm risk.

```powershell
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Interrupt Moderation" -DisplayValue "Disabled"
```

### 1.2 — Energy Efficient Ethernet → Disabled

Puts the NIC into a low-power sleep state during low activity. The wake-up transition introduces latency spikes incompatible with real-time applications. Some adapters also expose a separate "Green Ethernet" property — disable that too if present.

```powershell
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Energy-Efficient Ethernet" -DisplayValue "Disabled" -ErrorAction SilentlyContinue
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Green Ethernet" -DisplayValue "Disabled" -ErrorAction SilentlyContinue
```

### 1.3 — Flow Control → Disabled

Sends pause frames telling remote devices to stop transmitting until the NIC is ready. In gaming, this means the server stops sending game state updates while the NIC catches up — causing dropped data. TCP already implements its own flow control at a higher layer, making this redundant for internet traffic.

```powershell
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Flow Control" -DisplayValue "Disabled"
```

### 1.4 — Large Send Offload (LSO) → Disabled

Buffers outgoing TCP data on the NIC for segmentation. Designed for server throughput, not the small, time-critical packets used in gaming. Also disable USO (the UDP equivalent) and RSC (receive-side coalescing):

```powershell
# Adapter-level: LSO, RSC, USO
Disable-NetAdapterLso -Name $adapter -ErrorAction SilentlyContinue
Disable-NetAdapterRsc -Name $adapter -ErrorAction SilentlyContinue
Disable-NetAdapterUso -Name $adapter -ErrorAction SilentlyContinue

# OS global level (belt-and-suspenders)
Set-NetOffloadGlobalSetting -ReceiveSegmentCoalescing Disabled
```

### 1.5 — NIC Power Management → Disabled

Prevents Windows from powering down the adapter to save energy, which can cause momentary connection drops:

```powershell
Disable-NetAdapterPowerManagement -Name $adapter -ErrorAction SilentlyContinue
```

### 1.6 — Receive / Transmit Buffers → Maximum

Larger buffers reduce dropped packets during traffic spikes. Microsoft recommends increasing receive buffers to maximum. The available range depends on the adapter:

```powershell
# Check valid values for your adapter
Get-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Receive Buffers" | Format-List ValidDisplayValues
Get-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Transmit Buffers" | Format-List ValidDisplayValues

# Set to maximum (adjust the number to match your adapter's max)
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Receive Buffers" -DisplayValue "1024"
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Transmit Buffers" -DisplayValue "1024"
```

---

## 2 — NIC Adapter Settings (Low Impact / Cleanup)

These provide marginal gains individually but collectively remove unnecessary overhead.

### 2.1 — Receive Side Scaling (RSS) → Enabled (if supported)

RSS distributes incoming packet processing across multiple CPU cores instead of pinning it all to Core 0. Beneficial on any multi-core system. Enable it if available, with 2–4 queues:

```powershell
# Check if RSS is available
Get-NetAdapterRss -Name $adapter -ErrorAction SilentlyContinue

# Enable at adapter and OS level (errors mean it's unavailable — that's OK)
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Receive Side Scaling" -DisplayValue "Enabled" -ErrorAction SilentlyContinue
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Maximum Number of RSS Queues" -DisplayValue "4 Queues" -ErrorAction SilentlyContinue

# Optionally move RSS processing off Core 0 (where most OS work happens)
Set-NetAdapterRss -Name $adapter -BaseProcessorNumber 2 -ErrorAction SilentlyContinue
```

**Note:** Some drivers (notably Realtek's NetAdapterCx) don't support RSS. If these commands error out, RSS isn't available on the current driver — move on.

### 2.2 — Checksum Offloads → Leave Enabled

Modern NICs handle checksums reliably with no buffering penalty. Disabling forces the CPU to verify every packet's checksum for no latency benefit. Only disable as a troubleshooting step.

```powershell
Enable-NetAdapterChecksumOffload -Name $adapter -ErrorAction SilentlyContinue
```

### 2.3 — Other Cleanup

Features designed for servers, VMs, or sleep states that serve no purpose on a gaming desktop:

```powershell
# Jumbo Frames — only useful for LAN bulk transfers, requires end-to-end support
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Jumbo Packet" -DisplayValue "Disabled" -ErrorAction SilentlyContinue

# Wake-on-LAN features
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Wake on Magic Packet" -DisplayValue "Disabled" -ErrorAction SilentlyContinue
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Wake on Pattern Match" -DisplayValue "Disabled" -ErrorAction SilentlyContinue

# Sleep-mode offloads (respond to ARP/NS while PC is asleep)
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "ARP Offload" -DisplayValue "Disabled" -ErrorAction SilentlyContinue
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "NS Offload" -DisplayValue "Disabled" -ErrorAction SilentlyContinue

# Priority & VLAN — only needed with managed switches using 802.1q
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Priority & VLAN" -DisplayValue "Priority & VLAN Disabled" -ErrorAction SilentlyContinue

# Datacenter/virtualization features
Disable-NetAdapterIPsecOffload                  -Name $adapter -ErrorAction SilentlyContinue
Disable-NetAdapterQos                           -Name $adapter -ErrorAction SilentlyContinue
Disable-NetAdapterEncapsulatedPacketTaskOffload  -Name $adapter -ErrorAction SilentlyContinue
Disable-NetAdapterSriov                         -Name $adapter -ErrorAction SilentlyContinue
Disable-NetAdapterVmq                           -Name $adapter -ErrorAction SilentlyContinue

# Packet coalescing (OS level) — batches packets, the opposite of low latency
Set-NetOffloadGlobalSetting -PacketCoalescingFilter Disabled
```

> **Expect some errors.** Different vendors expose different properties. `-ErrorAction SilentlyContinue` handles missing properties gracefully.

---

## 3 — Windows Power Plan

Set the OS power profile so the CPU and peripherals stay at full performance:

```powershell
# Activate the built-in High Performance plan
powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

# If that fails (plan not present), create it first
powercfg -duplicatescheme 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
```

If the CPU vendor provides a custom power plan (e.g., "AMD Ryzen High Performance" via chipset driver), that works too. The critical thing is avoiding Balanced mode.

---

## 4 — TCP/IP Stack Tuning

These settings operate at the Windows networking stack level, independent of the NIC hardware.

### 4.1 — TCP Global Parameters

```powershell
# Check current state
netsh int tcp show global

# Auto-tuning — dynamically scales the TCP receive window. Leave on Normal
# unless a specific router is known to mishandle window scaling.
netsh int tcp set global autotuninglevel=normal

# ECN — some routers/ISPs mishandle Explicit Congestion Notification
netsh int tcp set global ecncapability=disabled

# TCP timestamps — removing saves 12 bytes per TCP packet
netsh int tcp set global timestamps=disabled

# Initial RTO — default 3000ms is conservative. 2000ms recovers faster from
# initial packet loss. Don't go below 1000ms without a very stable connection.
netsh int tcp set global initialRto=2000

# RSC (global) and chimney (deprecated) — ensure both are off
netsh int tcp set global rsc=disabled
netsh int tcp set global chimney=disabled
```

### 4.2 — Verify MTU Size

If the OS sends packets larger than the line supports (common with PPPoE), every oversized packet gets fragmented — adding overhead and latency. Windows defaults to 1500 bytes; many ISPs support only 1492.

```powershell
# Find the largest unfragmented payload (start at 1472, decrease by 10 until it works)
ping -f -l 1472 8.8.8.8

# MTU = largest working value + 28 (IP + ICMP headers)
# If 1472 works → MTU is already correct at 1500 → no change needed

# Set MTU if needed
netsh interface ipv4 show subinterface
netsh interface ipv4 set subinterface "Ethernet" mtu=1492 store=persistent
```

### 4.3 — Disable Nagle's Algorithm

Buffers small outgoing TCP packets and combines them before sending — adding latency to exactly the kind of traffic games produce.

**Caveat:** Only directly affects TCP. Most modern FPS/BR/MOBA games use UDP, so Nagle's has no effect on them. TCP-based games include many MMORPGs, some older titles, and certain indie games. Many modern games override this at the application level with `TCP_NODELAY`.

**How to apply (Registry):**

1. Run `ipconfig`, note the IPv4 address of the gaming adapter
2. Open `regedit`, navigate to:
   `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces`
3. Find the subkey with the matching IPv4 address (in `DhcpIPAddress` or `IPAddress`)
4. Create two DWORD (32-bit) values:
   - `TcpAckFrequency` = `1`
   - `TCPNoDelay` = `1`
5. Restart. To revert, set both to `0` or delete them.

### 4.4 — Pin Interface Metric

If both Ethernet and Wi-Fi are connected, Windows routes traffic via metric values (lower = higher priority):

```powershell
Set-NetIPInterface -InterfaceAlias $adapter -InterfaceMetric 5
```

### 4.5 — DNS

Faster DNS resolution speeds up matchmaking, server browsers, and launcher requests (does not affect in-game ping once connected):

```powershell
Set-DnsClientServerAddress -InterfaceAlias $adapter -ServerAddresses ("1.1.1.1","1.0.0.1")
```

### 4.6 — Disable NetBIOS over TCP/IP

Eliminates an unnecessary listening service: Adapter Properties → IPv4 → Properties → Advanced → WINS tab → "Disable NetBIOS over TCP/IP".

### 4.7 — Disable Delivery Optimization

Prevents Windows from using bandwidth to distribute updates to other PCs:

Settings → Windows Update → Advanced options → Delivery Optimization → Turn off "Allow downloads from other PCs".

---

## 5 — MMCSS Registry Tweaks

The Multimedia Class Scheduler Service (MMCSS) manages how Windows prioritizes network and CPU resources for multimedia and gaming tasks. All values live at:

`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile`

### 5.1 — NetworkThrottlingIndex

MMCSS limits non-multimedia packet processing to ~10 packets/ms (~100 Mbps) by default to protect audio/video playback. On any modern multi-core CPU this bottleneck is unnecessary.

Set `NetworkThrottlingIndex` (DWORD) to `ffffffff` (hex) to remove the limit.

If audio crackling or stuttering occurs under heavy network load, revert to `a` (hex, the default).

### 5.2 — SystemResponsiveness

Controls what percentage of CPU is reserved for background tasks. Default is `14` (hex) = 20%.

Set to `a` (hex) = 10%. Microsoft confirms values below 10 are treated as 10 — this is the lowest functional setting.

### 5.3 — Games Task Priority

At `...\SystemProfile\Tasks\Games`:

| Value | Type | Recommended | Default |
|---|---|---|---|
| GPU Priority | DWORD | `8` | `8` |
| Priority | DWORD | `6` | `2` |
| Scheduling Category | String | `High` | `Medium` |
| SFIO Priority | String | `High` | `Normal` |

Takes effect when a game registers with MMCSS (many modern titles do via the Game Bar).

### 5.4 — Disable NDU Service

NDU tracks per-app network usage in the background — minor overhead, safe to disable:

At `HKLM\SYSTEM\CurrentControlSet\Services\Ndu`, set `Start` (DWORD) to `4`.

---

## 6 — Consolidated Script

Save as `optimize-network.ps1` and run as Administrator. Adjust the `$adapter` variable first.

```powershell
#Requires -RunAsAdministrator
# ================================================================
# Network Optimization for Gaming — Windows 11
# ================================================================

$adapter = "Ethernet"  # <-- CHANGE THIS to match Get-NetAdapter output

Write-Host "`n=== NIC Adapter Settings ===" -ForegroundColor Cyan

# High Impact
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Interrupt Moderation"      -DisplayValue "Disabled"  -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Energy-Efficient Ethernet"  -DisplayValue "Disabled"  -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Green Ethernet"             -DisplayValue "Disabled"  -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Flow Control"               -DisplayValue "Disabled"  -EA 0
Disable-NetAdapterLso             -Name $adapter -EA 0
Disable-NetAdapterRsc             -Name $adapter -EA 0
Disable-NetAdapterUso             -Name $adapter -EA 0
Disable-NetAdapterPowerManagement -Name $adapter -EA 0

# Buffers (adjust "1024" to your adapter's maximum if different)
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Receive Buffers"  -DisplayValue "1024" -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Transmit Buffers" -DisplayValue "1024" -EA 0

# RSS (enable if supported)
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Receive Side Scaling"          -DisplayValue "Enabled"  -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Maximum Number of RSS Queues"  -DisplayValue "4 Queues" -EA 0

# Checksum offloads stay ON
Enable-NetAdapterChecksumOffload -Name $adapter -EA 0

# Low Impact Cleanup
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Jumbo Packet"         -DisplayValue "Disabled" -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Wake on Magic Packet" -DisplayValue "Disabled" -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Wake on Pattern Match" -DisplayValue "Disabled" -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "ARP Offload"          -DisplayValue "Disabled" -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "NS Offload"           -DisplayValue "Disabled" -EA 0
Disable-NetAdapterIPsecOffload                  -Name $adapter -EA 0
Disable-NetAdapterQos                           -Name $adapter -EA 0
Disable-NetAdapterEncapsulatedPacketTaskOffload -Name $adapter -EA 0
Disable-NetAdapterSriov                         -Name $adapter -EA 0
Disable-NetAdapterVmq                           -Name $adapter -EA 0

Write-Host "`n=== Global Offload Settings ===" -ForegroundColor Cyan

Set-NetOffloadGlobalSetting -ReceiveSegmentCoalescing Disabled
Set-NetOffloadGlobalSetting -PacketCoalescingFilter Disabled

Write-Host "`n=== TCP/IP Stack ===" -ForegroundColor Cyan

netsh int tcp set global autotuninglevel=normal
netsh int tcp set global ecncapability=disabled
netsh int tcp set global timestamps=disabled
netsh int tcp set global initialRto=2000
netsh int tcp set global rsc=disabled
netsh int tcp set global chimney=disabled

Write-Host "`n=== Power Plan ===" -ForegroundColor Cyan

powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c 2>$null
if ($LASTEXITCODE -ne 0) {
    powercfg -duplicatescheme 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c | Out-Null
    powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
}

Write-Host "`n=== Interface Metric ===" -ForegroundColor Cyan

Set-NetIPInterface -InterfaceAlias $adapter -InterfaceMetric 5

Write-Host "`n=== Done ===" -ForegroundColor Green
Write-Host "Registry changes (Nagle, MMCSS, NDU) must be applied manually — see Sections 4.3 and 5."
Write-Host "Restart your PC for all changes to take effect."
```

---

## 7 — Verification

```powershell
# Confirm adapter link speed
Get-NetAdapter -Name $adapter | Select-Object Name, LinkSpeed, Status

# Confirm NIC settings applied
Get-NetAdapterAdvancedProperty -Name $adapter | Format-Table DisplayName, DisplayValue -AutoSize

# Confirm global offload state
Get-NetOffloadGlobalSetting

# Confirm TCP globals
netsh int tcp show global

# Check MSI/MSI-X support (ensures efficient interrupt delivery)
Get-NetAdapterHardwareInfo -Name $adapter | Format-List *Interrupt*

# Packet loss test (expect 0% loss and consistent RTT)
ping -n 100 8.8.8.8

# DPC latency — download LatencyMon (resplendence.com) and run during gameplay
# Target: NDIS.sys DPC values consistently under 100 µs
```

---

## 8 — Common Myths

**"Disable all offloads"** — Outdated advice from the era of buggy NIC firmware. Modern checksum offloads work correctly, add no buffering, and should stay enabled. Only LSO, RSC, and USO introduce latency-relevant buffering.

**"SG TCP Optimizer Optimal preset"** — Designed for download throughput, not gaming latency. Automated presets can conflict with manual tweaks. Use custom mode or skip it.

**"Disable C-States in BIOS"** — Increases idle power with no measurable network latency benefit. On AMD Ryzen, this reduces Precision Boost headroom and can actually lower boost clocks.

**"Game Booster software"** — On any modern system with a multi-core CPU and 16 GB+ RAM, background processes use negligible resources. Benchmarks consistently show zero gain.

**"Set SystemResponsiveness to 0"** — Microsoft documentation states values below 10 are treated as 10. Setting to 0 does nothing more than 10.

**"Disable RSS"** — Was reasonable for dual-core CPUs. On 4+ core systems with a supporting NIC, RSS distributes NDIS DPC processing across cores and improves latency consistency.

---

## 9 — Driver Tips

Download NIC drivers directly from the manufacturer (Intel, Realtek, etc.) rather than relying on Windows Update, which often ships older or generic versions.

**Realtek:** The Windows 11 NetAdapterCx driver comes in two variants — "Power Saving" and "Not Support Power Saving". Use the latter for gaming, then disable RSC manually (Section 1.4). Realtek's NetAdapterCx implementation does not support RSS.

**Intel:** Drivers generally have full RSS support. Consider setting Interrupt Moderation to `Medium` or `Adaptive` rather than fully disabling — Intel's documentation and the GamingPCSetup project both note this provides a good balance.

**After any driver update:** Run `Get-NetAdapterAdvancedProperty -Name $adapter` to verify settings weren't reset to defaults — driver installations commonly overwrite manual configuration.

---

*Sources: Microsoft Learn (Network Adapter Performance Tuning), GamingPCSetup project (djdallmann), SpeedGuide.net, Intel Ethernet Adapter User Guide. May 2026.*
