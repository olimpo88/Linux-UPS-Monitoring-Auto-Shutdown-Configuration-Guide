
📦 Supported Systems: 
Ubuntu, Debian, Linux Mint, Fedora, Arch Linux

🔌 Compatible UPS Brands:
- Voltronic Power and OEM variants
- Armac, Blazer, Energy Sistem
- Fenton Technologies, General Electric (GE)
- Gtec, Hunnox, Masterguard
- Mustek, Powercool, SKE
- Atlantis Land, Crown, Mecer, Kstar
- Many others using Voltronic/Megatec/Q1 protocol

💡 Easy Detection:
If your UPS came with "Viewpower" software or uses Cypress Semiconductor USB chip (VendorID 0665), it's likely compatible!

Perfect for home servers, workstations, and production systems running Linux.

# UPS Monitoring Setup Guide for Linux

## System Information
- UPS Device: Cypress Semiconductor USB to Serial (ID 0665:5161)
- Driver: nutdrv_qx
- Protocol: Voltronic-QS
- Low Battery Threshold: 10%
- Shutdown Delay: 30 seconds after reaching low battery

---

## Step 1: Install NUT (Network UPS Tools)

### For Ubuntu/Debian/Linux Mint:
```bash
sudo apt update
sudo apt install nut
```

### For Fedora/RHEL:
```bash
sudo dnf install nut
```

### For Arch Linux:
```bash
sudo pacman -S nut
```

---

## Step 2: Detect Your UPS

Run the scanner to identify your UPS:
```bash
sudo nut-scanner -U
```

Expected output should show:
- driver = "nutdrv_qx"
- vendorid = "0665"
- productid = "5161"

---

## Step 3: Configure UPS Driver

Edit the UPS configuration file:
```bash
sudo nano /etc/nut/ups.conf
```

Add this configuration at the end of the file:
```ini
[ups]
    driver = nutdrv_qx
    port = auto
    vendorid = 0665
    productid = 5161
    desc = "Main UPS"
    
    # Low battery at 10% charge
    override.battery.charge.low = 10
    
    # Warning at 20% charge
    override.battery.charge.warning = 20
```

Save and exit (Ctrl+O, Enter, Ctrl+X)

---

## Step 4: Set NUT Operation Mode

Edit the NUT mode configuration:
```bash
sudo nano /etc/nut/nut.conf
```

Find the line `MODE=none` and change it to:
```ini
MODE=standalone
```

Save and exit.

---

## Step 5: Configure UPS Users

Edit the users file:
```bash
sudo nano /etc/nut/upsd.users
```

Add this configuration:
```ini
[admin]
    password = admin123
    actions = SET
    instcmds = ALL

[upsmon]
    password = monpass123
    upsmon master
```

Save and exit.

**Note:** Change the passwords for production use.

---

## Step 6: Configure Monitoring and Auto-Shutdown

Edit the monitor configuration:
```bash
sudo nano /etc/nut/upsmon.conf
```

Find and configure these lines (they may be commented with #):
```ini
# Monitor this UPS
MONITOR ups@localhost 1 upsmon monpass123 master

# Minimum power supplies needed
MINSUPPLIES 1

# Shutdown command
SHUTDOWNCMD "/sbin/shutdown -h +0"

# Power down flag
POWERDOWNFLAG /etc/killpower

# Notification command (uses upssched for timed events)
NOTIFYCMD /usr/local/bin/upssched-wrapper

# Notification flags
NOTIFYFLAG ONBATT SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT SYSLOG+WALL+EXEC
NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC
NOTIFYFLAG REPLBATT SYSLOG+WALL
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC
```

Save and exit.

---

## Step 7: Configure Shutdown Schedule

Edit the scheduler configuration:
```bash
sudo nano /etc/nut/upssched.conf
```

Add this configuration at the end:
```ini
CMDSCRIPT /etc/nut/upssched-cmd
PIPEFN /var/run/nut/upssched.pipe
LOCKFN /var/run/nut/upssched.lock

# Immediate message when power fails
AT ONBATT * EXECUTE onbatt-message

# Timer: shutdown if on battery for 15 minutes (optional backup)
AT ONBATT * START-TIMER shutdown-onbatt 900

# When battery reaches 10% (LOWBATT), wait 30 seconds then shutdown
AT LOWBATT * EXECUTE lowbatt-message
AT LOWBATT * START-TIMER shutdown-lowbatt 30

# Immediate message when power returns + cancel timers
AT ONLINE * EXECUTE online-message
AT ONLINE * CANCEL-TIMER shutdown-lowbatt shutdown-onbatt
```

Save and exit.

---

## Step 8: Create the Action Script

Create the script that handles events:
```bash
sudo nano /etc/nut/upssched-cmd
```

Paste this content:
```bash
#!/bin/bash

case $1 in
    shutdown-lowbatt)
        logger -t upssched "CRITICAL BATTERY - Shutting down in 1 minute"
        wall "WARNING: UPS battery critical (10%). System will shutdown in 1 minute."
        sleep 60
        /sbin/shutdown -h now "UPS battery depleted (10%)"
        ;;
    
    shutdown-onbatt)
        logger -t upssched "No power for 15 minutes - Shutting down"
        wall "WARNING: Extended power outage. Shutting down system."
        /sbin/shutdown -h +1 "Extended power outage"
        ;;
    
    onbatt-message)
        logger -t upssched "Power failure - Running on battery"
        wall "NOTICE: Power failure. System running on UPS battery."
        ;;
    
    online-message)
        logger -t upssched "Power restored"
        wall "NOTICE: Power restored. Shutdown cancelled."
        ;;
    
    lowbatt-message)
        logger -t upssched "Low battery detected"
        wall "WARNING: UPS battery is low. Save your work."
        ;;
    
    *)
        logger -t upssched "Unknown command: $1"
        ;;
esac

exit 0
```

Make it executable:
```bash
sudo chmod +x /etc/nut/upssched-cmd
```

---

## Step 9: Create Wrapper Script for upssched

upssched needs environment variables that upsmon doesn't provide by default. Create a wrapper:
```bash
sudo nano /usr/local/bin/upssched-wrapper
```

Paste this content:
```bash
#!/bin/bash
# Wrapper for upssched that converts arguments to environment variables

MESSAGE="$1"

# Extract notification type from message
if [[ "$MESSAGE" =~ "on battery" ]]; then
    export NOTIFYTYPE="ONBATT"
elif [[ "$MESSAGE" =~ "on line power" ]]; then
    export NOTIFYTYPE="ONLINE"
elif [[ "$MESSAGE" =~ "battery is low" ]]; then
    export NOTIFYTYPE="LOWBATT"
elif [[ "$MESSAGE" =~ "shutdown" ]]; then
    export NOTIFYTYPE="SHUTDOWN"
elif [[ "$MESSAGE" =~ "needs to be replaced" ]]; then
    export NOTIFYTYPE="REPLBATT"
else
    export NOTIFYTYPE="UNKNOWN"
fi

# Extract UPS name (assuming format "UPS name@host ...")
if [[ "$MESSAGE" =~ UPS\ ([^\ ]+) ]]; then
    export UPSNAME="${BASH_REMATCH[1]}"
else
    export UPSNAME="ups@localhost"
fi

# Execute upssched
exec /usr/sbin/upssched
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/upssched-wrapper
```

---

## Step 10: Start the Driver

Start the UPS driver:
```bash
sudo upsdrvctl start
```

Expected output:
```
Network UPS Tools - UPS driver controller 2.x.x
Network UPS Tools - Generic Q* USB/Serial driver 0.x (2.x.x)
USB communication driver (libusb 1.0) 0.x
Using protocol: Voltronic-QS 0.09
```

---

## Step 11: Start and Enable Services

Start the NUT server:
```bash
sudo systemctl enable nut-server
sudo systemctl start nut-server
sudo systemctl status nut-server
```

Start the monitor:
```bash
sudo systemctl enable nut-monitor
sudo systemctl start nut-monitor
sudo systemctl status nut-monitor
```

Both should show **active (running)** in green.

---

## Step 12: Verify Configuration

### Check UPS status:
```bash
upsc ups
```

You should see:
- `battery.charge: 100` (or current charge)
- `battery.charge.low: 10`
- `ups.status: OL` (OnLine = on utility power)
- `input.voltage: ~220` (varies by region)

### Check specific battery info:
```bash
upsc ups | grep battery
```

### Monitor in real-time:
```bash
watch -n 2 'upsc ups | grep -E "battery.charge|ups.status|input.voltage"'
```

Press Ctrl+C to exit.

---

## Step 13: Test the System (Optional)

**WARNING:** This will simulate a battery event. Only do this if you're ready to test.

### View logs in real-time:
```bash
journalctl -f -u nut-monitor | grep upssched
```

### Unplug the UPS from wall power

You should see:
- Log message: "Power failure - Running on battery"
- UPS status changes to `OB` (On Battery)
- Battery charge starts decreasing

### Plug the UPS back in

You should see:
- Log message: "Power restored"
- UPS status changes to `OL` (OnLine)
- Shutdown timers cancelled

---

## Monitoring Commands

### View current UPS status:
```bash
upsc ups
```

### Monitor battery level and status:
```bash
watch -n 2 'upsc ups | grep -E "battery.charge|ups.status"'
```

### View UPS events log:
```bash
journalctl -u nut-monitor | grep upssched
```

### View recent events (last 10 minutes):
```bash
journalctl -u nut-monitor --since "10 minutes ago" | grep upssched
```

### Real-time event monitoring:
```bash
journalctl -f -u nut-monitor
```

---

## System Behavior Summary

### On Power Failure (ONBATT):
1. Event logged to system journal
2. 15-minute countdown timer starts (backup safety)
3. System continues running on battery

### On Low Battery (10% or less):
1. Warning logged: "Low battery detected"
2. 30-second countdown begins
3. After 30 seconds: System shuts down automatically

### On Power Restoration (ONLINE):
1. Event logged: "Power restored"
2. All shutdown timers cancelled
3. System continues normal operation

---

## Troubleshooting

### Check if driver is running:
```bash
sudo upsdrvctl stop
sudo upsdrvctl start
```

### Check if server is running:
```bash
sudo systemctl status nut-server
```

### Check if monitor is running:
```bash
sudo systemctl status nut-monitor
```

### View detailed debug logs:
```bash
sudo systemctl stop nut-monitor
sudo /lib/nut/upsmon -DDDDDD -F
```
(Press Ctrl+C to stop, then restart service: `sudo systemctl start nut-monitor`)

### Test upssched manually:
```bash
sudo NOTIFYTYPE=ONBATT UPSNAME=ups /usr/sbin/upssched
```

---

## Configuration Files Location

- UPS driver config: `/etc/nut/ups.conf`
- NUT mode: `/etc/nut/nut.conf`
- Users: `/etc/nut/upsd.users`
- Monitor: `/etc/nut/upsmon.conf`
- Scheduler: `/etc/nut/upssched.conf`
- Action script: `/etc/nut/upssched-cmd`
- Wrapper script: `/usr/local/bin/upssched-wrapper`

---

## Adjusting Settings

### Change low battery threshold (e.g., to 5%):
Edit `/etc/nut/ups.conf`:
```ini
override.battery.charge.low = 5
```
Then restart: `sudo upsdrvctl stop && sudo upsdrvctl start`

### Change shutdown delay (e.g., to 60 seconds):
Edit `/etc/nut/upssched.conf`:
```ini
AT LOWBATT * START-TIMER shutdown-lowbatt 60
```
Then restart: `sudo systemctl restart nut-monitor`

### Change extended outage timer (e.g., to 10 minutes = 600 seconds):
Edit `/etc/nut/upssched.conf`:
```ini
AT ONBATT * START-TIMER shutdown-onbatt 600
```
Then restart: `sudo systemctl restart nut-monitor`

---

## Disable Debug Logging (Optional)

If you enabled debug mode, you can disable it:

Edit `/etc/nut/upsmon.conf` and remove or comment out:
```ini
# DEBUG_MIN 6
```

Then restart:
```bash
sudo systemctl restart nut-monitor
```

---

## Additional Resources

- NUT Documentation: https://networkupstools.org/docs/user-manual.chunked/index.html
