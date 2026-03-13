# eGPU USB4 Troubleshooting Guide (Windows 11)
## For WIKO Hi GT Cube and similar USB4 eGPU docks

This guide was written based on real troubleshooting experience with a **WIKO Hi GT Cube (AMD Radeon RX 7600M XT)** connected via USB4 to a **Huawei MateBook 16s**. The fixes should work for any USB4 eGPU setup with Intel USB4 host router on Windows 11.

---

## Prerequisites

All commands run in **PowerShell as Administrator**.

---

## Cheat Sheet — Quick Reference

### Step 1. GPU not coming up after connecting dock

```powershell
# Restart Intel USB4 Host Router — triggers PCIe tunnel renegotiation
$usb4 = Get-PnpDevice | Where-Object { $_.InstanceId -like "*VEN_8086*DEV_A73E*USB4_MS_CM*" }
pnputil /restart-device $usb4.InstanceId
```

Wait 10 seconds. Check status:

```powershell
Get-PnpDevice | Where-Object {
    $_.InstanceId -like "*VEN_1002*DEV_7480*" -or
    $_.InstanceId -like "*VEN_1002*DEV_7480*"
} | Format-List FriendlyName, Status, InstanceId
```

---

### Step 2. GPU still Unknown after Step 1

```powershell
# Restart the PCIe upstream switch inside the dock
$upstream = Get-PnpDevice | Where-Object {
    $_.FriendlyName -like "*Upstream Switch*" -and $_.InstanceId -like "*VEN_8086*DEV_15EF*"
}
pnputil /restart-device $upstream.InstanceId
```

---

### Step 3. GPU came up but monitor has no signal

```powershell
# Disable/Enable AMD GPU adapter — forces display re-enumeration
$gpu = Get-PnpDevice | Where-Object { $_.InstanceId -like "*VEN_1002*DEV_7480*" }
Disable-PnpDevice -InstanceId $gpu.InstanceId -Confirm:$false
Start-Sleep -Seconds 3
Enable-PnpDevice -InstanceId $gpu.InstanceId -Confirm:$false
```

---

### Step 4. DisplayPort not working (only HDMI works)

The DP output runs through a separate USB4 DP tunnel. If it fails, you need to remove and recreate the downstream PCIe bridge entries.

**Important:** Remove all three downstream switch ports, in this specific order — the two "empty" ones first, then the main DP tunnel port last. Removing them out of order causes immediate reconnect before the tunnel is cleanly reset.

```powershell
# Find all three Intel downstream switch ports
$ports = Get-PnpDevice | Where-Object {
    $_.FriendlyName -like "*Downstream Switch*" -and
    $_.InstanceId -like "*VEN_8086*DEV_15EF*" -and
    $_.InstanceId -notlike "*UCA*"  # exclude upstream
}
$ports | Format-List FriendlyName, InstanceId

# Remove them one by one (last one triggers DP tunnel recreation)
$sortedPorts = $ports | Sort-Object InstanceId
pnputil /remove-device $sortedPorts[1].InstanceId
pnputil /remove-device $sortedPorts[2].InstanceId
pnputil /remove-device $sortedPorts[0].InstanceId

# Trigger hardware rescan
pnputil /scan-devices
```

Wait a few seconds. DP should come back.

---

### Step 5. Nothing works — full cold boot sequence

If all software fixes fail, the dock firmware has likely hung internally.

1. Run `shutdown /s /t 0` (full shutdown, NOT restart)
2. Wait **20 seconds** after screen goes dark
3. Boot Windows **without dock connected**
4. Wait for full Windows desktop load
5. Connect USB4 dock
6. Run Step 1 command if GPU doesn't appear automatically

> **Why boot without dock first?** PCIe tunneling is negotiated during USB4 link training. If the dock is connected at boot, a firmware hang can prevent the tunnel from being established. Hot-plugging after boot forces a clean negotiation.

---

## Check Full Stack Status

```powershell
# GPU and monitor status in one command
Get-PnpDevice | Where-Object {
    $_.FriendlyName -like "*Radeon*" -or
    $_.FriendlyName -like "*Mi Monitor*"
} | Where-Object { $_.InstanceId -like "PCI\*" -or $_.InstanceId -like "DISPLAY\*" } |
Format-List FriendlyName, Status, InstanceId

# USB4 topology
Get-PnpDevice | Where-Object { $_.InstanceId -like "*USB4*" } |
Format-List FriendlyName, Status, InstanceId
```

---

## Stability Fixes (Do Once)

### Disable USB Selective Suspend

Prevents Windows from sleeping the USB4 port under load:

```powershell
powercfg /setacvalueindex SCHEME_CURRENT 2a737441-1930-4402-8d77-b2bebba308a3 48e6b7a6-50f5-4782-a5d4-53bb8f07e226 0
powercfg /setdcvalueindex SCHEME_CURRENT 2a737441-1930-4402-8d77-b2bebba308a3 48e6b7a6-50f5-4782-a5d4-53bb8f07e226 0
powercfg /setactive SCHEME_CURRENT
```

Verify:
```powershell
powercfg /query SCHEME_CURRENT 2a737441-1930-4402-8d77-b2bebba308a3
# Both indexes should show 0x00000000
```

### Disable Fast Startup

Fast Startup prevents proper USB4 tunnel initialization on boot:

```powershell
powercfg /hibernate off
# Or via Control Panel → Power Options → Choose what the power buttons do → uncheck Fast Startup
```

---

## Decision Tree

```
GPU not showing up?
│
├─ USB4 dock visible in Settings → Bluetooth & devices → USB?
│   ├─ NO  → Check cable, check dock power, try different USB4 port
│   └─ YES → Run Step 1 (restart Intel USB4 Host Router)
│
├─ GPU Status = Unknown after Step 1?
│   └─ YES → Run Step 2 (restart PCIe upstream switch)
│
├─ GPU Status = OK but no display output?
│   └─ YES → Run Step 3 (disable/enable AMD adapter)
│
├─ GPU OK, HDMI works but DP doesn't?
│   └─ YES → Run Step 4 (remove downstream PCIe bridges)
│
└─ Nothing works → Run Step 5 (cold boot sequence)
```

---

## Tested Hardware

| Component | Model |
|-----------|-------|
| eGPU dock | WIKO Hi GT Cube |
| GPU | AMD Radeon RX 7600M XT |
| Laptop | Huawei MateBook 16s |
| USB4 controller | Intel Gen12 (0xa73e) |
| OS | Windows 11 |
| AMD driver | 25.20.21.01 |

Commands use dynamic device discovery where possible and should work on other USB4 eGPU setups with Intel USB4 controllers.

---

## Notes

- `CM_PROB_PHANTOM` in device logs = USB4 tunnel dropped briefly, usually recovers on its own
- Multiple `Kernel-Power 105` events (power source switching) during gaming = USB4 instability, check cable seating
- If AMD driver EeuDumps appear frequently (`C:\Windows\System32\AMD\EeuDumps`) = driver crash, consider reinstalling AMD Adrenalin cleanly
- Two XMI27B1 monitor entries (UID512 and UID516) = normal after GPU reinit, both can coexist
