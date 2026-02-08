# Linux-UPS-Monitoring-Auto-Shutdown-Configuration-Guide
Protect your Linux system from data loss during power outages with automated UPS monitoring and shutdown. This repository provides complete setup instructions for Network UPS Tools (NUT) with smart battery management.

![unnamed (2)](https://github.com/user-attachments/assets/0dfed288-d5bb-4795-afd7-66cb9767e6cf)


✨ Key Features:
- Automatic shutdown on low battery (customizable 5-20%)
- Real-time battery level monitoring
- System logs and event notifications
- Extended power outage protection
- Easy configuration templates

If your UPS has the same ID when you run lsusb, you can copy the repository files; otherwise, you can follow the guide on how to configure it from scratch.

```
Bus 001 Device 005: ID 0665:5161 Cypress Semiconductor USB to Serial
```

If you don't have the same model, you should follow the guide for other models.

# Quick Start Guide - Cypress Semiconductor UPS (Voltronic-QS Protocol)

## ⚠️ Important: Check Your UPS ID First

**Before using this quick guide**, verify your UPS ID by running:
```bash
lsusb
```

Look for this **exact** line:
```
Bus 001 Device XXX: ID 0665:5161 Cypress Semiconductor USB to Serial
```

### ✅ If you see `ID 0665:5161` → You can use this quick configuration!

The device number (XXX) doesn't matter, only the ID `0665:5161` needs to match.

### ❌ If you have a different ID → Use the full manual setup

Follow the complete guide instead: [Manual Setup Guide for All Models](MANUAL-SETUP.md)

## About This Configuration

This is a ready-to-use configuration specifically for UPS devices using:
- **USB Chip:** Cypress Semiconductor USB to Serial
- **Vendor ID:** 0665
- **Product ID:** 5161
- **Protocol:** Voltronic-QS
- **Driver:** nutdrv_qx


## Compatible Brands/Models

This configuration works with many UPS brands that use Voltronic chipsets:
- Voltronic Power (OEM)
- APC, Armac, Blazer, Energy Sistem
- Fenton Technologies, GE
- Lyonn
- Mustek, Powercool
- Any UPS that came with "Viewpower" software

**How to verify:** Run `lsusb` and look for:
```
Bus XXX Device XXX: ID 0665:5161 Cypress Semiconductor USB to Serial
```

---

## Quick Installation

### Step 1: Install NUT

**Ubuntu/Debian/Linux Mint:**
```bash
sudo apt update
sudo apt install nut inotify-tools
```

**Fedora:**
```bash
sudo dnf install nut inotify-tools
```

**Arch Linux:**
```bash
sudo pacman -S nut inotify-tools
```

---

### Step 2: Copy Configuration Files

Clone this repository:
```bash
git clone https://github.com/YOUR_USERNAME/ups-monitor-linux.git
cd ups-monitor-linux/config-examples/cypress-voltronic-qs
```

Copy all configuration files:
```bash
# Backup original configs (just in case)
sudo cp -r /etc/nut /etc/nut.backup

# Copy pre-configured files
sudo cp nut/ups.conf /etc/nut/
sudo cp nut/nut.conf /etc/nut/
sudo cp nut/upsd.users /etc/nut/
sudo cp nut/upsmon.conf /etc/nut/
sudo cp nut/upssched.conf /etc/nut/
sudo cp nut/upssched-cmd /etc/nut/
sudo chmod +x /etc/nut/upssched-cmd

# Copy wrapper script
sudo cp upssched-wrapper /usr/local/bin/
sudo chmod +x /usr/local/bin/upssched-wrapper
```

---

### Step 3: Start Services
```bash
# Start the driver
sudo upsdrvctl start

# Enable and start services
sudo systemctl enable nut-server nut-monitor
sudo systemctl start nut-server
sudo systemctl start nut-monitor
```

---

### Step 4: Verify It's Working

Check UPS status:
```bash
upsc ups
```

You should see something like:
```
battery.charge: 100
battery.charge.low: 10
battery.voltage: 27.0
ups.status: OL
input.voltage: 220.0
```

Monitor in real-time:
```bash
watch -n 2 'upsc ups | grep -E "battery.charge|ups.status|input.voltage"'
```

---

## Configuration Overview

### Default Settings:
- **Low battery threshold:** 10%
- **Warning threshold:** 20%
- **Shutdown delay:** 30 seconds after reaching low battery
- **Extended outage:** Shutdown after 15 minutes on battery (backup safety)

### Behavior:
1. **Power fails** → Logs event, starts 15-minute timer
2. **Battery reaches 10%** → Waits 30 seconds, then shuts down
3. **Power returns** → Cancels all shutdown timers

---

## Customization

### Change Low Battery Threshold

Edit `/etc/nut/ups.conf`:
```ini
override.battery.charge.low = 5  # Change to 5%, 15%, etc.
```

Restart driver:
```bash
sudo upsdrvctl stop && sudo upsdrvctl start
```

### Change Shutdown Delay

Edit `/etc/nut/upssched.conf`:
```ini
AT LOWBATT * START-TIMER shutdown-lowbatt 60  # 60 seconds instead of 30
```

Restart monitor:
```bash
sudo systemctl restart nut-monitor
```

### Change Extended Outage Timer

Edit `/etc/nut/upssched.conf`:
```ini
AT ONBATT * START-TIMER shutdown-onbatt 600  # 10 minutes instead of 15
```

Restart monitor:
```bash
sudo systemctl restart nut-monitor
```

---

## Monitoring Commands

### View current status:
```bash
upsc ups
```

### View battery info:
```bash
upsc ups | grep battery
```

### View recent events:
```bash
journalctl -u nut-monitor --since "10 minutes ago" | grep upssched
```

### Real-time monitoring:
```bash
journalctl -f -u nut-monitor
```

---

## Testing

**WARNING:** Only test if you're prepared for a simulated power event.

### Monitor logs:
```bash
journalctl -f -u nut-monitor | grep upssched
```

### Unplug UPS from wall power

You should see:
```
upssched: Power failure - Running on battery
```

UPS status will change from `OL` (OnLine) to `OB` (On Battery)

### Plug back in

You should see:
```
upssched: Power restored
```

## Security Notes

**IMPORTANT:** The default configuration uses these passwords:
- Admin password: `admin123`
- Monitor password: `monpass123`

For production systems, change these in `/etc/nut/upsd.users`:
```bash
sudo nano /etc/nut/upsd.users
```

Then restart:
```bash
sudo systemctl restart nut-server nut-monitor
```

## Contact 📧
https://www.linkedin.com/in/peraltaleandro/
