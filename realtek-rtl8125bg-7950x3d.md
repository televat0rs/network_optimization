# Example: Realtek RTL8125BG + AMD Ryzen 9 7950X3D

Hardware-specific configuration for applying the [main guide](../GUIDE.md) to this system. Every advanced property exposed by the Realtek RTL8125BG under the NetAdapterCx driver is listed below with a recommended value and rationale.

## System

| Component | Spec |
|---|---|
| OS | Windows 11 Pro (Build 26200) |
| CPU | AMD Ryzen 9 7950X3D (16C/32T) |
| RAM | 32 GB DDR5 |
| GPU | NVIDIA RTX 4090 |
| NIC | Realtek RTL8125BG 2.5 GbE |
| Driver | NetAdapterCx — "Not Support Power Saving" variant |
| Adapter name | `LAN` |

## Hardware-Specific Notes

### No RSS support

Realtek's NetAdapterCx driver does not implement Receive Side Scaling. The framework supports it, but Realtek chose not to. All NDIS DPC processing is pinned to a single core (typically Core 0). This is not a gaming bottleneck on a 16-core CPU — game traffic is low-bandwidth — but means multi-gigabit throughput can't be parallelized.

The legacy NDIS driver (`rtnic64w10.inf`) supports RSS but receives fewer updates. Not recommended unless RSS is specifically needed.

### Driver variant

Realtek publishes two NetAdapterCx builds:

| Variant | INF file | Power Saving | RSC default |
|---|---|---|---|
| Support Power Saving | `rt68cx21x64.inf` | Enabled | Off |
| **Not Support Power Saving** | `rt68cx21x64sta.inf` | **Disabled** | On |

Use **"Not Support Power Saving"**, then disable RSC manually (the script below does this). Download from [Realtek](https://www.realtek.com/Download/List?cate_id=584) or [Station-Drivers](https://www.station-drivers.com/). As of early 2026, the latest version is v1126.x / v11.026.x.

### Interrupt Moderation

Fully disabled on this system with no issues. The 7950X3D has enough headroom that interrupt storms from a single 2.5 GbE NIC are not a concern. On lower-core-count CPUs (4–6 cores), consider `Medium` instead — the GamingPCSetup project found `Medium` provides nearly identical DPC latency with better stability under combined load.

### Power plan

Using the **AMD Ryzen High Performance** plan (installed via the AMD chipset driver). The generic Windows High Performance plan also works — the key is avoiding Balanced mode.

---

## Complete NIC Advanced Properties Reference

Every property visible in Device Manager → Advanced tab for the RTL8125BG under the NetAdapterCx driver. Some properties may appear or disappear depending on the exact driver version. The script in the next section applies all of these.

> **Note:** Properties marked with `(extended)` may only appear on certain driver versions or board vendor customizations. Properties marked `N/A` are not available on the Realtek NetAdapterCx driver.

### High Impact — Latency-Critical

| Property | Recommended | Default | Why |
|---|---|---|---|
| Interrupt Moderation | **Disabled** | Enabled | Batches packets before generating CPU interrupts — adds latency to every packet. Biggest single improvement. |
| Flow Control | **Disabled** | Rx & Tx Enabled | Sends pause frames that stop the server from sending game data. TCP has its own flow control. Realtek's implementation has historically been buggy. |
| Large Send Offload v2 (IPv4) | **Disabled** | Enabled | Buffers outgoing TCP data on the NIC. Designed for server throughput, not small gaming packets. |
| Large Send Offload v2 (IPv6) | **Disabled** | Enabled | Same as above for IPv6. |
| Recv Segment Coalescing (IPv4) | **Disabled** | Enabled¹ | Merges incoming TCP segments before delivering to the stack. Receive-side counterpart to LSO. |
| Recv Segment Coalescing (IPv6) | **Disabled** | Enabled¹ | Same as above for IPv6. |
| Energy-Efficient Ethernet | **Disabled** | Enabled | Puts NIC into low-power sleep during low activity. Wake-up transition causes latency spikes. |
| Green Ethernet | **Disabled** | Enabled | Adjusts voltage based on cable length. Can cause instability during transitions. |
| Power Saving Mode | **Disabled** | Disabled¹ | Allows NIC to enter low-power states. Already disabled by the "Not Support Power Saving" driver variant. |

¹ Default depends on driver variant. "Not Support Power Saving" disables Power Saving but enables RSC.

### Medium Impact — Buffers and Offloads

| Property | Recommended | Default | Why |
|---|---|---|---|
| Receive Buffers | **1024** (max) | 512 | Larger buffers prevent dropped packets during traffic spikes. 32 GB RAM means memory is not a concern. |
| Transmit Buffers | **1024** (max) | 128 | Same reasoning. Set to maximum. |
| Auto Disable Gigabit | **Disabled** | Disabled | Throttles the connection below gigabit speeds. Should never be enabled on a gaming rig. |
| Gigabit Lite | **Disabled** | Enabled | Limits link speed to save power. Same reasoning as Auto Disable Gigabit — always disable. |
| Advanced EEE | **Disabled** | Disabled | Extended Energy Efficient Ethernet features. Disable alongside standard EEE. |
| EEE Max Support Speed | **2.5 Gbps Full Duplex** | 1.0 Gbps Full Duplex | If EEE can't be fully disabled, at least don't let it cap your link speed. |

### Low Impact — Checksum Offloads (Leave Enabled)

These add no buffering latency. Modern Realtek implementations handle them correctly. Disabling forces the CPU to calculate every checksum for no benefit. Only disable as a troubleshooting step if experiencing unexplained packet loss.

| Property | Recommended | Default | Why |
|---|---|---|---|
| IPv4 Checksum Offload | Rx & Tx Enabled | Rx & Tx Enabled | No buffering, works correctly on RTL8125B family. |
| TCP Checksum Offload (IPv4) | Rx & Tx Enabled | Rx & Tx Enabled | Same. |
| TCP Checksum Offload (IPv6) | Rx & Tx Enabled | Rx & Tx Enabled | Same. |
| UDP Checksum Offload (IPv4) | Rx & Tx Enabled | Rx & Tx Enabled | Same. |
| UDP Checksum Offload (IPv6) | Rx & Tx Enabled | Rx & Tx Enabled | Same. |

### Negligible Impact — Cleanup

| Property | Recommended | Default | Why |
|---|---|---|---|
| ARP Offload | Disabled | Enabled | Responds to ARP requests while PC is asleep. Irrelevant on a desktop. |
| NS Offload | Disabled | Enabled | Same as ARP Offload for IPv6 Neighbor Solicitation. |
| Wake on Magic Packet | Disabled | Enabled | Not needed unless using Wake-on-LAN. |
| Wake on Pattern Match | Disabled | Enabled | Not needed on a gaming desktop. |
| WOL & Shutdown Link Speed | **Not Speed Down** | 10 Mbps First | Prevents the NIC from dropping to 10 Mbps after shutdown. If WOL is disabled, this is moot. |
| Shutdown Wake-On-Lan | Disabled | Enabled | Controls WOL behavior at shutdown. Disable with other WOL settings. |
| Priority & VLAN | Priority & VLAN Disabled | Priority & VLAN Enabled | Only needed with managed switches using 802.1q VLAN tagging. Adds unnecessary header bytes on home networks. |
| Jumbo Frame | Disabled | Disabled | Only useful for LAN bulk transfers with end-to-end support. No benefit for internet gaming. |
| Speed & Duplex | Auto Negotiation | Auto Negotiation | Correctly detects 2.5 Gbps link. No benefit to manual override. |
| Network Address | (not present) | (not present) | MAC address override. No reason to change. |

### Not Available on Realtek NetAdapterCx

| Property | Status | Why |
|---|---|---|
| Receive Side Scaling | **N/A** | Realtek's NetAdapterCx implementation does not support RSS. |
| Maximum Number of RSS Queues | **N/A** | Requires RSS. |

---

## Full Script

Pre-configured for this hardware. Uses `Set-NetAdapterAdvancedProperty -DisplayName` for every setting (the same labels visible in Device Manager), so the exact values being applied are explicit and auditable.

```powershell
#Requires -RunAsAdministrator
# ================================================================
# Network Optimization — Realtek RTL8125BG / Windows 11
# Adapter name: LAN (change if different on your board)
# ================================================================

$adapter = "LAN"

Write-Host "`n=== Verifying adapter ===" -ForegroundColor Cyan
$nic = Get-NetAdapter -Name $adapter -ErrorAction Stop
Write-Host "  Found: $($nic.InterfaceDescription) @ $($nic.LinkSpeed)"

# ---------------------------------------------------------------
# HIGH IMPACT — Latency-critical
# ---------------------------------------------------------------
Write-Host "`n=== High Impact Settings ===" -ForegroundColor Cyan

Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Interrupt Moderation"              -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Flow Control"                       -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Large Send Offload v2 (IPv4)"      -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Large Send Offload v2 (IPv6)"      -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Energy-Efficient Ethernet"          -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Green Ethernet"                     -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Power Saving Mode"                  -DisplayValue "Disabled"          -EA 0

# RSC — enabled by default on "Not Support Power Saving" variant
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Recv Segment Coalescing (IPv4)"    -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Recv Segment Coalescing (IPv6)"    -DisplayValue "Disabled"          -EA 0

# Also disable via dedicated cmdlets (covers variations in display names across driver versions)
Disable-NetAdapterLso             -Name $adapter -EA 0
Disable-NetAdapterRsc             -Name $adapter -EA 0
Disable-NetAdapterUso             -Name $adapter -EA 0
Disable-NetAdapterPowerManagement -Name $adapter -EA 0

# ---------------------------------------------------------------
# MEDIUM IMPACT — Buffers, power features
# ---------------------------------------------------------------
Write-Host "`n=== Medium Impact Settings ===" -ForegroundColor Cyan

Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Receive Buffers"                   -DisplayValue "1024"              -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Transmit Buffers"                  -DisplayValue "1024"              -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Auto Disable Gigabit"              -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Gigabit Lite"                       -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Advanced EEE"                       -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "EEE Max Support Speed"             -DisplayValue "2.5 Gbps Full Duplex" -EA 0

# ---------------------------------------------------------------
# CHECKSUM OFFLOADS — Leave ON (no buffering, works correctly)
# ---------------------------------------------------------------
Write-Host "`n=== Checksum Offloads (keeping enabled) ===" -ForegroundColor Cyan

Enable-NetAdapterChecksumOffload -Name $adapter -EA 0

# ---------------------------------------------------------------
# LOW IMPACT — Cleanup
# ---------------------------------------------------------------
Write-Host "`n=== Low Impact Cleanup ===" -ForegroundColor Cyan

Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "ARP Offload"                       -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "NS Offload"                        -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Wake on Magic Packet"              -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Wake on pattern match"             -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "WOL & Shutdown Link Speed"         -DisplayValue "Not Speed Down"    -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Shutdown Wake-On-Lan"              -DisplayValue "Disabled"          -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Priority & VLAN"                   -DisplayValue "Priority & VLAN Disabled" -EA 0
Set-NetAdapterAdvancedProperty -Name $adapter -DisplayName "Jumbo Frame"                       -DisplayValue "Disabled"          -EA 0

# Datacenter/VM features (not all may be present — errors are expected)
Disable-NetAdapterIPsecOffload                  -Name $adapter -EA 0
Disable-NetAdapterQos                           -Name $adapter -EA 0
Disable-NetAdapterEncapsulatedPacketTaskOffload -Name $adapter -EA 0
Disable-NetAdapterSriov                         -Name $adapter -EA 0
Disable-NetAdapterVmq                           -Name $adapter -EA 0

# ---------------------------------------------------------------
# GLOBAL OFFLOAD SETTINGS (OS level)
# ---------------------------------------------------------------
Write-Host "`n=== Global Offload Settings ===" -ForegroundColor Cyan

Set-NetOffloadGlobalSetting -ReceiveSegmentCoalescing Disabled
Set-NetOffloadGlobalSetting -PacketCoalescingFilter Disabled

# ---------------------------------------------------------------
# TCP/IP STACK
# ---------------------------------------------------------------
Write-Host "`n=== TCP/IP Stack ===" -ForegroundColor Cyan

netsh int tcp set global autotuninglevel=normal
netsh int tcp set global ecncapability=disabled
netsh int tcp set global timestamps=disabled
netsh int tcp set global initialRto=2000
netsh int tcp set global rsc=disabled
netsh int tcp set global chimney=disabled

# ---------------------------------------------------------------
# POWER PLAN
# ---------------------------------------------------------------
Write-Host "`n=== Power Plan ===" -ForegroundColor Cyan

powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c 2>$null
if ($LASTEXITCODE -ne 0) {
    powercfg -duplicatescheme 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c | Out-Null
    powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
}

# ---------------------------------------------------------------
# INTERFACE METRIC & DNS
# ---------------------------------------------------------------
Write-Host "`n=== Interface Metric & DNS ===" -ForegroundColor Cyan

Set-NetIPInterface -InterfaceAlias $adapter -InterfaceMetric 5
Set-DnsClientServerAddress -InterfaceAlias $adapter -ServerAddresses ("1.1.1.1","1.0.0.1")

# ---------------------------------------------------------------
# DONE
# ---------------------------------------------------------------
Write-Host "`n=== Done ===" -ForegroundColor Green
Write-Host @"
Adapter settings, TCP/IP stack, power plan, metric, and DNS applied.

The following registry changes must be applied manually (see main guide Sections 4.3 and 5):

  Nagle's Algorithm:
    HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{your adapter GUID}
      TcpAckFrequency = 1  (DWORD)
      TCPNoDelay      = 1  (DWORD)

  MMCSS:
    HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile
      NetworkThrottlingIndex = ffffffff  (DWORD, hex)
      SystemResponsiveness  = a         (DWORD, hex = 10 decimal)

    ...\SystemProfile\Tasks\Games
      GPU Priority          = 8         (DWORD)
      Priority              = 6         (DWORD)
      Scheduling Category   = High      (String)
      SFIO Priority         = High      (String)

  NDU Service:
    HKLM\SYSTEM\CurrentControlSet\Services\Ndu
      Start = 4  (DWORD, disabled)

  NetBIOS:
    Adapter Properties > IPv4 > Properties > Advanced > WINS tab
      Disable NetBIOS over TCP/IP

Restart your PC for all changes to take effect.
"@
```

---

## Verification

```powershell
$adapter = "LAN"

# Link speed — should show 2.5 Gbps
Get-NetAdapter -Name $adapter | Select-Object Name, InterfaceDescription, LinkSpeed, Status

# Full settings audit — compare against the reference table above
Get-NetAdapterAdvancedProperty -Name $adapter | Format-Table DisplayName, DisplayValue -AutoSize

# Confirm RSC is off (enabled by default on "Not Support Power Saving" variant)
Get-NetAdapterRsc -Name $adapter

# Confirm LSO is off
Get-NetAdapterLso -Name $adapter

# Confirm RSS is unavailable (expected — will show error or Enabled=False)
Get-NetAdapterRss -Name $adapter

# Confirm USO is off
Get-NetAdapterUso -Name $adapter

# MSI-X support (should show MsiXInterruptSupported: True)
Get-NetAdapterHardwareInfo -Name $adapter | Format-List *Interrupt*

# Global offload state
Get-NetOffloadGlobalSetting

# TCP globals
netsh int tcp show global

# Packet loss and latency consistency (expect 0% loss, stable RTT)
ping -n 100 8.8.8.8

# DPC latency — download LatencyMon (resplendence.com) and run during gameplay
# Target: NDIS.sys DPC values consistently under 100 µs
```

---

## Rollback

Reset all NIC advanced properties to factory defaults:

```powershell
Reset-NetAdapterAdvancedProperty -Name "LAN" -DisplayName "*"
```

For full rollback including TCP/IP globals, restore from the backup file created in Section 0 of the main guide.
