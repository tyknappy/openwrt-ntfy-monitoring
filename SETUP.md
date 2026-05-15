# Setup Guide

Complete setup for monitoring a remote OpenWrt router from a home OpenWrt router, with ntfy push notifications.

---

## Before You Start: Fill In Your Values

Open `my-values.txt` (which you created by copying `my-values.example.txt`). You'll substitute these values into the commands below wherever you see the placeholder text in `ALL_CAPS_LIKE_THIS`.

The placeholders used throughout this guide:

| Placeholder | What it is | Example |
|---|---|---|
| `YOUR_NTFY_TOPIC_HERE` | Your private ntfy topic name. Long and random. | `MyHome2Cabin8492XqRz` |
| `YOUR_REMOTE_TAILSCALE_IP` | Tailscale IP of the remote router | `100.x.x.x` |
| `YOUR_REMOTE_HOSTNAME` | Tailscale hostname of the remote router | `cabin-router` |
| `YOUR_HOME_HOSTNAME` | Tailscale hostname of the home router | `home-router` |
| `YOUR_REMOTE_LOCATION` | Friendly name for the remote location | `Cabin`, `RV`, `Office` |
| `YOUR_REMOTE_WAN_NAME` | Friendly name for the remote ISP | `Starlink`, `T-Mobile Home`, `Verizon 5G` |

---

## What You'll Get When This Is Done

Notifications to your phone for:
- **WAN Online / Dropped** (from the remote router directly)
- **Remote Router Online** when the remote router boots up after a power loss
- **Remote Heartbeat** every morning at 8 AM, confirming the remote router is alive
- **Remote Unreachable / Reachable** (from the home router), the backup detection for full outages

All alerts go to a single ntfy topic.

---

## How to SSH Into Each Router

You'll connect to these two devices in this guide. Pay attention to which one each section uses.

**Remote router:**
Connect via Tailscale using `YOUR_REMOTE_HOSTNAME` or `YOUR_REMOTE_TAILSCALE_IP`.

**Home router:**
Connect via Tailscale using `YOUR_HOME_HOSTNAME`, or locally at your router's LAN IP.

The SSH prompt will show your router's model (something like `root@GL-MT3000:~#`). Confirm you're on the right one before running commands. The remote router and home router should show different prompts.

---

# PART 1: Remote Router Setup

All commands in this section run on the **remote router**.

## Step 1: Subscribe to the ntfy Topic on Your Phone

Open the ntfy app on your phone:
1. Tap the **+** button.
2. Enter your topic name exactly: `YOUR_NTFY_TOPIC_HERE`
3. Tap **Subscribe**.

Leave the app running in the background.

## Step 2: WAN Up/Down Alerts

SSH into the remote router. Paste this entire block into the terminal at once and press Enter. **Replace `YOUR_NTFY_TOPIC_HERE` with your actual topic before pasting**, and replace `YOUR_REMOTE_LOCATION` and `YOUR_REMOTE_WAN_NAME` with your friendly names:

```sh
cat > /etc/hotplug.d/iface/99-ntfy-wan << 'EOF'
#!/bin/sh

[ "$INTERFACE" = "wan" ] || exit 0

TOPIC="YOUR_NTFY_TOPIC_HERE"

if [ "$ACTION" = "ifup" ]; then
    curl -H "Title: ✅ YOUR_REMOTE_WAN_NAME Online" \
         -H "Tags: white_check_mark,satellite" \
         -H "Priority: default" \
         -d "YOUR_REMOTE_LOCATION YOUR_REMOTE_WAN_NAME connection is back online." \
         https://ntfy.sh/$TOPIC
elif [ "$ACTION" = "ifdown" ]; then
    curl -H "Title: ⚠️ YOUR_REMOTE_WAN_NAME Dropped" \
         -H "Tags: warning,satellite" \
         -H "Priority: high" \
         -d "YOUR_REMOTE_LOCATION YOUR_REMOTE_WAN_NAME connection has gone offline." \
         https://ntfy.sh/$TOPIC
fi
EOF
chmod +x /etc/hotplug.d/iface/99-ntfy-wan
```

OpenWrt's hotplug system will now automatically run this script whenever an interface changes state. The first line filters out everything except the `wan` interface.

### Test It

Fake an event without dropping your real connection:

```sh
ACTION=ifup INTERFACE=wan /etc/hotplug.d/iface/99-ntfy-wan
ACTION=ifdown INTERFACE=wan /etc/hotplug.d/iface/99-ntfy-wan
```

You should get one of each notification within a few seconds.

**Important caveat:** When the WAN actually goes down for real, the "Dropped" notification has no internet to leave the remote site. You'll usually only see the "Online" notification when it recovers. The Home Router Watcher in Part 2 fills this gap for full outages.

## Step 3: Boot / Power Loss Alert

Edit the local startup file:

```sh
vi /etc/rc.local
```

You'll see some text ending in `exit 0`. We need to add a line **above** `exit 0`.

1. Press `i` to enter insert mode.
2. Position your cursor on a blank line directly above `exit 0`.
3. Paste this, with your values substituted:

```sh
(sleep 60 && curl -H "Title: 🔄 YOUR_REMOTE_LOCATION Router Online" -H "Tags: electric_plug" -H "Priority: default" -d "The router at YOUR_REMOTE_LOCATION has just booted up or recovered from a power loss." https://ntfy.sh/YOUR_NTFY_TOPIC_HERE) &
```

4. Press `Esc`.
5. Type `:wq` and press Enter to save and exit.

The 60-second sleep gives the WAN time to come up before sending. The `&` at the end runs it in the background so it doesn't hold up the boot.

### Test It

```sh
reboot
```

Wait about 90 to 120 seconds. You should get the boot notification.

## Step 4: Daily Heartbeat at 8 AM

Open the crontab:

```sh
crontab -e
```

1. Press `i`.
2. Add this line at the bottom (substituting your values):

```
0 8 * * * curl -H "Title: 💓 YOUR_REMOTE_LOCATION Heartbeat" -H "Tags: heartbeat" -H "Priority: low" -d "Router at YOUR_REMOTE_LOCATION is online. Monitoring active." https://ntfy.sh/YOUR_NTFY_TOPIC_HERE
```

3. Press `Esc`, type `:wq`, press Enter.

Reload cron:

```sh
/etc/init.d/cron restart
```

### Test It

Temporarily change the schedule to every minute:

```sh
crontab -e
```

Press `i`, change `0 8 * * *` to `* * * * *`, save with `Esc` then `:wq`, then:

```sh
/etc/init.d/cron restart
```

Wait up to 60 seconds for the notification. **Then change it back to `0 8 * * *`** and restart cron again. Otherwise you'll get a notification every minute forever.

---

# PART 2: Home Router Watcher

All commands in this section run on the **home router**.

This is the safety net. Even if the remote router can't send a "Dropped" alert (because the WAN is completely offline), the home router will detect the silence over Tailscale and notify you.

## Step 1: Create the Watcher Script

SSH into the home router. Paste this entire block, with your values substituted:

```sh
cat > /root/check-remote-router.sh << 'EOF'
#!/bin/sh

# Remote router Tailscale IP
TARGET="YOUR_REMOTE_TAILSCALE_IP"
TOPIC="YOUR_NTFY_TOPIC_HERE"
STATE_FILE="/tmp/remote_router_state"

# Try 3 pings, 3 second timeout each. Success if at least 1 reply.
if ping -c 3 -W 3 -q "$TARGET" >/dev/null 2>&1; then
    NEW_STATE="up"
else
    NEW_STATE="down"
fi

# Read previous state. Default to "up" if file does not exist yet.
OLD_STATE=$(cat "$STATE_FILE" 2>/dev/null || echo "up")

# Only send a notification if the state actually changed.
if [ "$NEW_STATE" != "$OLD_STATE" ]; then
    if [ "$NEW_STATE" = "down" ]; then
        curl -H "Title: ⚠️ YOUR_REMOTE_LOCATION Unreachable" \
             -H "Tags: warning,house" \
             -H "Priority: high" \
             -d "YOUR_REMOTE_LOCATION appears to be offline. The router is no longer responding to pings from home." \
             https://ntfy.sh/$TOPIC
    else
        curl -H "Title: ✅ YOUR_REMOTE_LOCATION Reachable" \
             -H "Tags: white_check_mark,house" \
             -H "Priority: default" \
             -d "YOUR_REMOTE_LOCATION is back. The router is responding to pings from home." \
             https://ntfy.sh/$TOPIC
    fi
    echo "$NEW_STATE" > "$STATE_FILE"
fi
EOF
chmod +x /root/check-remote-router.sh
```

## Step 2: Schedule It With Cron

```sh
crontab -e
```

Press `i`, add this line:

```
*/2 * * * * /root/check-remote-router.sh
```

Press `Esc`, type `:wq`, press Enter. Then:

```sh
/etc/init.d/cron restart
```

## Step 3: Test It

### Test 1: Confirm the script runs cleanly

```sh
/root/check-remote-router.sh
```

No output, no notification. This is correct. State matches, no change.

### Test 2: Force a "Down" alert

```sh
sed 's|YOUR_REMOTE_TAILSCALE_IP|100.99.99.99|' /root/check-remote-router.sh > /tmp/test-remote.sh
chmod +x /tmp/test-remote.sh
/tmp/test-remote.sh
```

Within about 10 seconds, you should get an "Unreachable" notification.

Clean up:

```sh
rm /tmp/test-remote.sh
rm /tmp/remote_router_state
```

### Test 3: Force an "Up" alert

```sh
echo "down" > /tmp/remote_router_state
/root/check-remote-router.sh
```

You should get a "Reachable" notification within a few seconds.

If all three tests worked, the watcher is live.

---

## How the Two Routers Work Together

| Event | What the Remote Router Sends | What the Home Router Sends |
|---|---|---|
| WAN drops briefly (flap) | "Dropped" then "Online" | (nothing if it recovers in under 2 minutes) |
| WAN drops completely | (nothing, no internet) | "Unreachable" |
| WAN restored | "Online" | "Reachable" |
| Remote router loses power then recovers | "Router Online" | "Reachable" |
| Daily 8 AM | "Heartbeat" | (none) |

If you ever get a "Unreachable" alert from the home router but no "Online" follow-up from the remote router later, something more serious is happening at the remote site (router power, hardware failure) rather than just an ISP burp.

---

## Detection Speed

The home watcher runs every 2 minutes with 3 ping attempts. Worst case detection is roughly 2 to 4 minutes from when the outage actually started.

To detect faster, change `*/2` to `*/1` in the home router crontab. More sensitive, more occasional false alarms from brief Tailscale hiccups.

---

## Maintenance

### Disable Alerts Temporarily

**On the home router:**
```sh
crontab -e
```
Put a `#` at the start of the `/root/check-remote-router.sh` line. Save, then `/etc/init.d/cron restart`.

**On the remote router:**
- WAN alerts: `chmod -x /etc/hotplug.d/iface/99-ntfy-wan`
- Heartbeat: comment out the line in `crontab -e`, restart cron.

### Remove Completely

**Home router:**
```sh
crontab -e
```
Delete the watcher line, save, then:
```sh
/etc/init.d/cron restart
rm /root/check-remote-router.sh
rm /tmp/remote_router_state
```

**Remote router:**
```sh
rm /etc/hotplug.d/iface/99-ntfy-wan
crontab -e
```
Delete the heartbeat line, save, restart cron. Then edit `/etc/rc.local` and remove the curl line above `exit 0`.

---

## Troubleshooting

**No notifications at all from one router.**
SSH in and test curl directly:
```sh
curl -d "test" https://ntfy.sh/YOUR_NTFY_TOPIC_HERE
```
You should get a JSON response and a phone notification. If not, check that your topic name on the phone matches exactly.

**Getting false "Unreachable" alerts from the home router.**
Tailscale relays occasionally cause a single failed cycle. Increase `-c 3` to `-c 5` in the script, or change cron from `*/2` to `*/5`.

**Reboot alert never fires.**
The 60-second sleep might not be enough for slow WANs. Increase it to `sleep 90` or `sleep 120` in `/etc/rc.local`.

**Verify the home watcher is running:**
```sh
cat /tmp/remote_router_state
```
Should show `up`. If the file is missing, no state change has happened since the last home router reboot, which is normal.

---

## Quick Reference

### Remote Router

| What | Where |
|---|---|
| WAN up/down script | `/etc/hotplug.d/iface/99-ntfy-wan` |
| Boot alert | `/etc/rc.local`, line above `exit 0` |
| Daily heartbeat | `crontab -e` |

### Home Router

| What | Where |
|---|---|
| Watcher script | `/root/check-remote-router.sh` |
| State file | `/tmp/remote_router_state` |
| Cron schedule | `crontab -e` |
