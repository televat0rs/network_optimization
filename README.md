# win11-network-optimization

Research-backed NIC, TCP/IP stack, and MMCSS tuning for low-latency gaming on Windows 11. PowerShell-first — every setting is scriptable and uses `-ErrorAction SilentlyContinue` to handle vendor differences. Prioritized by measured impact on DPC latency and packet timing.

## Structure

| File | Description |
|---|---|
| [`GUIDE.md`](GUIDE.md) | Hardware-independent guide with consolidated apply script |
| [`examples/`](examples/) | Per-hardware configs with full Advanced tab property tables and ready-to-run scripts |

## Scope

NIC adapter properties (interrupt moderation, LSO/RSC/USO, EEE, flow control, power management, buffers, RSS), TCP/IP globals (auto-tuning, ECN, timestamps, initial RTO, MTU, Nagle), MMCSS registry (NetworkThrottlingIndex, SystemResponsiveness, Games task priority), and OS-level cleanup (NDU, NetBIOS, Delivery Optimization, interface metrics, DNS).

## Quick Start

```powershell
# Backup
$adapter = "Ethernet"
Get-NetAdapterAdvancedProperty -Name $adapter | Format-Table -AutoSize | Out-File "$env:USERPROFILE\Desktop\nic_backup.txt"

# Apply — see GUIDE.md Section 6 for the full script, or examples/ for hardware-specific versions

# Revert
Reset-NetAdapterAdvancedProperty -Name $adapter -DisplayName "*"
```

## Hardware Examples

| Example | NIC | Key Notes |
|---|---|---|
| [Realtek RTL8125BG + 7950X3D](examples/realtek-rtl8125bg-7950x3d.md) | Realtek RTL8125BG 2.5 GbE (NetAdapterCx) | No RSS support, RSC on by default, full property reference |

PRs welcome for other hardware (Intel I225/I226, Killer, Aquantia, etc.). Include `Get-NetAdapterAdvancedProperty` output so display names can be verified.

## Sources

- [Microsoft Learn — Network Adapter Performance Tuning](https://learn.microsoft.com/en-us/windows-server/networking/technologies/network-subsystem/net-sub-performance-tuning-nics)
- [GamingPCSetup — Network Configuration](https://djdallmann.github.io/GamingPCSetup/CONTENT/DOCS/NETWORK/)
- [Microsoft Learn — TCP/IP Performance Known Issues](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/tcpip-performance-known-issues)
- [SpeedGuide.net](https://www.speedguide.net/)
- [Intel Ethernet Adapter User Guide](https://www.intel.com/content/www/us/en/support/articles/000005480/ethernet-products.html)
