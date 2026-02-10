# 🔧 Summon Doctor — Quick Reference

Emergency cheat sheet. Print this and tape it to your SSD.

---

## Quick Start (Manual)

```bash
cd "/Volumes/YOUR_DRIVE_NAME/Doctor"

OPENCLAW_CONFIG_PATH="./state/openclaw.json" \
OPENCLAW_STATE_DIR="./state" \
openclaw gateway start
```

---

## If You Have a Local Agent Running

Just ask it:
> "Start Doctor on the external drive"

---

## To Stop Doctor

```bash
# Find the process
ps aux | grep openclaw | grep 18790

# Kill it
kill [PID]

# Or if using LaunchAgent:
launchctl unload ~/Library/LaunchAgents/com.doctor.airgap.plist
```

---

## Disable Auto-Start

```bash
# Create DISABLED flag
touch "/Volumes/YOUR_DRIVE_NAME/Doctor/DISABLED"

# Re-enable later
rm "/Volumes/YOUR_DRIVE_NAME/Doctor/DISABLED"
```

---

## SSH to Nodes

```bash
# Use Doctor's SSH config
ssh -F /Volumes/YOUR_DRIVE_NAME/Doctor/workspace/.ssh/config node1
ssh -F /Volumes/YOUR_DRIVE_NAME/Doctor/workspace/.ssh/config node2
```

---

## Something's Wrong?

1. **Is the drive mounted?** `ls /Volumes/`
2. **Is Doctor running?** `ps aux | grep openclaw`
3. **Check logs:** `cat /tmp/doctor-airgap.log`
4. **Port in use?** `lsof -i :18790`

---

## Network Quick Reference

Update this with your actual network:

| Node | Hostname | IP | User |
|------|----------|-----|------|
| Agent 1 | ?.local | 10.0.0.? | ? |
| Agent 2 | ?.local | 10.0.0.? | ? |
| Agent 3 | ?.local | 10.0.0.? | ? |

---

*Full guide: [README.md](README.md)*
