# eGPU USB4 Troubleshooting Guide (Windows 11)
## For WIKO Hi GT Cube and similar USB4 eGPU docks

This guide was written based on real troubleshooting experience with a **WIKO Hi GT Cube (AMD Radeon RX 7600M XT)** connected via USB4 to a **Huawei MateBook 16s**. The fixes should work for any USB4 eGPU setup with Intel USB4 host router on Windows 11.
> **Note:** As of 13 March 2026, there are no other known troubleshooting guides for the WIKO Hi GT Cube. News coverage exists ([Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/this-external-amd-gpu-is-also-a-laptop-charger), [NotebookCheck](https://www.notebookcheck.net/WIKO-Hi-GT-Cube-external-GPU-launches-alongside-Huawei-MateBook-GT-14-gaming-laptop.870855.0.html)) but no community troubleshooting. This guide documents real-world fixes discovered through hands-on debugging. It's possible these issues were triggered by a bad Windows update and most users never encountered them — but if there's even one person in the world with the same problem who finds this guide, it was worth writing.
> 
All commands run in **PowerShell as Administrator**.

---

## ⚡ Do These Once — Prevents Most Problems

### 1. Disable USB Selective Suspend

Windows can "sleep" the USB4 port to save power, which kills the eGPU tunnel mid-session.

```powershell
powercfg /setacvalueindex SCHEME_CURRENT 2a737441-1930-4402-8d77-b2bebba308a3 48e6b7a6-50f5-4782-a5d4-53bb8f07e226 0
powercfg /setdcvalueindex SCHEME_CURRENT 2a737441-1930-4402-8d77-b2bebba308a3 48e6b7a6-50f5-4782-a5d4-53bb8f07e226 0
powercfg /setactive SCHEME_CURRENT
```

Verify (both indexes must show `0x00000000`):
```powershell
powercfg /query SCHEME_CURRENT 2a737441-1930-4402-8d77-b2bebba308a3
```

### 2. Disable Fast Startup

Fast Startup (hibernate-based) prevents proper USB4 PCIe tunnel negotiation on next boot.

```powershell
powercfg /hibernate off
```

Or via: Control Panel → Power Options → Choose what the power buttons do → uncheck **Turn on fast startup**.

---

## 🔍 Device Discovery — Find Your IDs First

Before running any fix, understand your hardware. Every device has a unique InstanceId — some parts are hardware-specific and will differ from this guide. Run these commands once to map your setup.

### Full USB4 stack:
```powershell
Get-PnpDevice | Where-Object { $_.InstanceId -like "*USB4*" } |
    Format-List FriendlyName, Status, InstanceId
```

### Your Intel USB4 host router (needed for Fix 1):
```powershell
Get-PnpDevice | Where-Object { $_.InstanceId -like "*USB4_MS_CM*" } |
    Format-List FriendlyName, Status, InstanceId
```

### PCIe switches inside the dock (needed for Fix 2 and Fix 4):
```powershell
Get-PnpDevice | Where-Object {
    $_.FriendlyName -like "*Switch*" -and $_.InstanceId -like "PCI\*"
} | Format-List FriendlyName, Status, InstanceId
```

### Your GPU:
```powershell
Get-PnpDevice | Where-Object {
    ($_.FriendlyName -like "*Radeon*" -or $_.FriendlyName -like "*GeForce*") -and
    $_.InstanceId -like "PCI\*"
} | Format-List FriendlyName, Status, InstanceId
```

### All connected monitors:
```powershell
Get-PnpDevice | Where-Object { $_.InstanceId -like "DISPLAY\*" } |
    Format-List FriendlyName, Status, InstanceId
```

### GPU problem code (what's actually wrong):
```powershell
Get-PnpDeviceProperty -InstanceId (
    Get-PnpDevice | Where-Object { $_.InstanceId -like "*VEN_1002*DEV_7480*" }
).InstanceId -KeyName "DEVPKEY_Device_ProblemCode"
```

| Code | Meaning |
|------|---------|
| `Empty` | Device not on PCIe bus — tunnel not established |
| `0` | No problem |
| `43` | Driver error — reinstall AMD driver |
| `28` | No drivers installed |

> **Tip:** Run the discovery commands and save the output somewhere. When something breaks, compare current state to the known-good state.

---

## 🛠️ Fixes

### Fix 1. GPU not coming up after connecting dock

Restarts the Intel USB4 Host Router — forces PCIe tunnel renegotiation.

```powershell
$usb4 = Get-PnpDevice | Where-Object { $_.InstanceId -like "*USB4_MS_CM*" }
pnputil /restart-device $usb4.InstanceId
```

Wait 10 seconds, then check status.

> **Other hardware:** `USB4_MS_CM` pattern works across Intel 12th/13th/14th gen. If multiple results appear, use the one with `VEN_8086` in the InstanceId.

---

### Fix 2. GPU still Unknown after Fix 1

Restarts the PCIe upstream switch inside the dock.

```powershell
$upstream = Get-PnpDevice | Where-Object {
    $_.FriendlyName -like "*Upstream Switch*" -and $_.InstanceId -like "PCI\*"
}
pnputil /restart-device $upstream.InstanceId
```

---

### Fix 3. GPU OK but monitor has no signal

Forces AMD driver to re-enumerate all display outputs.

```powershell
$gpu = Get-PnpDevice | Where-Object { $_.InstanceId -like "*VEN_1002*DEV_7480*" }
Disable-PnpDevice -InstanceId $gpu.InstanceId -Confirm:$false
Start-Sleep -Seconds 3
Enable-PnpDevice -InstanceId $gpu.InstanceId -Confirm:$false
```

> **Other hardware:** `DEV_7480` = AMD RX 7600M XT. Replace with your GPU's DEV ID (find it with the discovery commands above).

---

### Fix 4. DisplayPort not working (only HDMI works)

HDMI goes through PCIe directly. DP uses a separate USB4 DisplayPort tunnel — if that tunnel fails, it needs to be recreated.

**Step 4a — List your downstream ports:**
```powershell
Get-PnpDevice | Where-Object {
    $_.FriendlyName -like "*Downstream Switch*" -and
    $_.InstanceId -like "PCI\*" -and
    $_.InstanceId -notlike "*UCA*"
} | Format-List FriendlyName, InstanceId
```

You should see 3 ports. **Identify which one is the DP tunnel:** it's the port whose removal causes your monitor signal to drop. On WIKO Hi GT Cube it's the port with the lowest address suffix (sorted first alphabetically).

**Step 4b — Remove all three, DP tunnel port last:**
```powershell
$ports = Get-PnpDevice | Where-Object {
    $_.FriendlyName -like "*Downstream Switch*" -and
    $_.InstanceId -like "PCI\*" -and
    $_.InstanceId -notlike "*UCA*"
}
$sortedPorts = $ports | Sort-Object InstanceId

# Remove non-DP ports first
pnputil /remove-device $sortedPorts[1].InstanceId
pnputil /remove-device $sortedPorts[2].InstanceId
# Remove DP tunnel port last — triggers recreation
pnputil /remove-device $sortedPorts[0].InstanceId

pnputil /scan-devices
```

Wait a few seconds. DP should come back.

> **Note:** If DP doesn't recover, the DP port on your dock may sort differently. Try a different removal order.

---

### Fix 5. Nothing works — full cold boot sequence

The dock firmware has hung internally. Software fixes won't help.

1. `shutdown /s /t 0` — full shutdown, **not** restart
2. Wait **20 seconds**
3. Unplug dock from wall, wait 10 seconds, plug back in
4. Boot laptop **without** USB4 cable connected
5. Wait for full Windows desktop
6. Connect USB4 cable
7. If GPU doesn't appear, run Fix 1

> **Why boot without dock?** PCIe tunnel negotiation happens at USB4 connect time. Hot-plugging after a clean Windows boot is more reliable than booting with the dock already connected, especially after a firmware hang.

---

## 🔁 Auto-Enable Script (Use With Caution)

This script monitors for the GPU appearing in Unknown state and enables it automatically. Useful if your GPU consistently requires a manual enable after connecting.

> ⚠️ **Important warning:** If the dock or DP connection is unstable (rapidly connecting/disconnecting), this script will spam enable calls and can make the GPU driver hang or cause a dropout loop. **Do not run this script while actively troubleshooting connection issues.** Only use it after the connection is stable.

Save as `AMD_eGPU_Enable.ps1`:

```powershell
while ($true) {
    $gpu = Get-PnpDevice | Where-Object {
        $_.InstanceId -like "*VEN_1002*DEV_7480*" -and $_.Status -eq "Unknown"
    }
    if ($gpu) {
        Enable-PnpDevice -InstanceId $gpu.InstanceId -Confirm:$false
        Start-Sleep -Seconds 5  # debounce — do not remove this
    }
    Start-Sleep -Seconds 7
}
```

Add to Task Scheduler (runs at login, hidden window):
```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File C:\Scripts\AMD_eGPU_Enable.ps1"
$trigger = New-ScheduledTaskTrigger -AtLogOn
$settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit 0
Register-ScheduledTask -TaskName "AMD_eGPU_Enable" -Action $action `
    -Trigger $trigger -Settings $settings -RunLevel Highest -Force
```

---

## 📊 Decision Tree

```
eGPU problem?
│
├─ Dock visible in Settings → Bluetooth & devices → USB4?
│   ├─ NO  → Check cable / dock power / try different USB4 port
│   └─ YES → continue
│
├─ GPU Status = Unknown?
│   ├─ YES → Run Fix 1 (restart USB4 host router)
│   │         Still Unknown? → Run Fix 2 (restart upstream switch)
│   │         Still Unknown? → Run Fix 5 (cold boot)
│   └─ NO  → continue
│
├─ GPU Status = OK but no display?
│   └─ YES → Run Fix 3 (disable/enable GPU adapter)
│
├─ GPU OK, HDMI works but DP black screen?
│   └─ YES → Run Fix 4 (recreate DP tunnel)
│
└─ Random GPU dropout during gaming?
    └─ USB Selective Suspend disabled? Fast Startup disabled?
       Check logs for Kernel-Power 105 events — USB4 cable may be degrading
```

---

## 🔬 Diagnostics

### Check for GPU dropouts in the last 30 minutes:
```powershell
Get-WinEvent -LogName System | Where-Object {
    $_.TimeCreated -gt (Get-Date).AddMinutes(-30) -and
    ($_.ProviderName -eq "Microsoft-Windows-Kernel-Power" -or $_.Id -in @(41, 6008, 105))
} | Format-List TimeCreated, Id, ProviderName, Message
```

**What to look for:**
- `Id 105` = power source switched (AC ↔ battery) — many 105 events during gaming = USB4 instability under load, check cable seating
- `Id 41` = unexpected crash/reboot
- `Id 6008` = previous shutdown was unexpected

### Check for AMD driver crashes:
```powershell
Get-ChildItem "C:\Windows\System32\AMD\EeuDumps" | Sort-Object CreationTime -Descending | Select-Object -First 5
```

Recent files = AMD driver has been crashing. Fix: clean reinstall AMD Adrenalin using DDU (Display Driver Uninstaller) in Safe Mode.

### Check if Tailscale/WireGuard restarts correlate with GPU drops:
```powershell
Get-WinEvent -LogName "Microsoft-Windows-Kernel-PnP/Configuration" -ErrorAction SilentlyContinue |
    Where-Object { $_.Message -like "*Wintun*" } |
    Select-Object -Last 20 |
    Format-List TimeCreated, Message
```

Compare timestamps with GPU dropout events. If they correlate — pause Tailscale during gaming as a test.

---

## 📋 Tested Hardware

| Component | Model |
|-----------|-------|
| eGPU dock | WIKO Hi GT Cube |
| GPU | AMD Radeon RX 7600M XT |
| Laptop | Huawei MateBook 16s |
| USB4 controller | Intel Gen12 (0xa73e) |
| OS | Windows 11 |
| AMD driver | 25.20.21.01 |

---

## 📝 Notes

- `CM_PROB_PHANTOM` in device logs = USB4 tunnel dropped briefly, usually self-recovers
- Two monitor entries with different UIDs (e.g. UID512 and UID516) = normal after GPU reinit, both can coexist
- AMD EeuDump files grow to ~1MB then roll over — frequent new files = driver instability
- `pnputil /restart-device` does not cause data loss — equivalent to Device Manager disable/enable
- USB4 cable quality matters under thermal load — use a certified USB4 40Gbps cable, preferably under 1m
- If the GPU fails to initialize without a display connected — plug in an **HDMI dummy plug** to the dock's HDMI port. This tricks the GPU into thinking a monitor is present and forces full driver initialization. No permanent fix found yet; dummy plug is a reliable workaround.

---

## 🔗 Resources & Further Reading

- **[egpu.io](https://egpu.io)** — the largest eGPU community. Huge database of working configurations, per-device build guides, and an active forum. If your specific hardware combination isn't covered here, search there first.
- **[[GUIDE] Win10/11 solutions for eGPU detection, BSOD, crashing](https://egpu.io/forums/pc-setup/guide-generic-windows-10-solutions-for-egpu-bsod-crashing-and-system-freezing-link-state-power-management-tdrdelay-tdrddidelay-nvidia-power-management-settings/)** — comprehensive egpu.io guide covering TDR delays, Link State Power Management, iGPU disabling and more. Good next step if this guide doesn't solve your issue.
- **[Surface Laptop eGPU not recognizing after Windows 11 update — Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5640804/surface-laptop-5-not-recognizing-egpu-after-window)** — detailed thread on PCIe tunnel negotiation failures after Windows updates, including the PCI Bridge uninstall technique that inspired Fix 4 in this guide.
- **[AOOSTAR AG02 eGPU not detected — USB4 link OK, no PCIe tunnel — egpu.io](https://egpu.io/forums/thunderbolt-enclosures/aoostar-ag02-egpu-not-detected-on-rog-ally-x-usb4-link-ok-no-pcie-tunnel/)** — same symptom as this guide (USB4 connected, dock visible, GPU not on bus). Useful for cross-referencing with other dock models.
