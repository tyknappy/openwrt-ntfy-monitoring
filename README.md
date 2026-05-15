# OpenWrt Multi-Site Network Monitoring with ntfy

Push notifications to your phone for network events across two OpenWrt-based routers, using the free [ntfy.sh](https://ntfy.sh) service.

Designed for a two-site topology: a main location with reliable internet (often with multi-WAN failover), and a remote location whose connection you want to monitor (cabin, RV, secondary home, parents' house, etc.).

## Features

- **WAN up/down alerts** on the remote router (when it can still send)
- **Boot/power-recovery alerts** after the remote router restarts
- **Daily heartbeat** confirming the remote router is alive
- **Cross-site watchdog**: the home router pings the remote router over Tailscale every 2 minutes, so you still get notified when the remote site is fully offline and can't send its own alerts

## Hardware This Was Built On

- Home router: GL.iNet Flint 2 (GL-MT6000) running OpenWrt 21.02
- Remote router: GL.iNet Beryl AX (GL-MT3000) running OpenWrt 21.02

Should work on any OpenWrt-based router with minor path adjustments. Tested only on the above.

## Prerequisites

- Two OpenWrt-based routers (or one router and any always-on Linux box at the remote site)
- A [Tailscale](https://tailscale.com) account with both routers added to the same tailnet
- The [ntfy app](https://ntfy.sh) on your phone (free, no account needed)
- An SSH client (Termius, PuTTY, OpenSSH, anything)
- Willingness to copy and paste commands into a terminal

## Quick Start

1. Clone or download this repo to your computer.
2. Copy `my-values.example.txt` to `my-values.txt` and fill in your specific values.
3. Open `SETUP.md` and follow the step-by-step instructions. Wherever you see a placeholder like `YOUR_NTFY_TOPIC_HERE`, substitute the corresponding value from your `my-values.txt`.

`my-values.txt` is listed in `.gitignore` and will never be committed, so your secrets stay local even if you push changes back to GitHub.

## Security Notes

- **ntfy topic names are secrets.** Anyone who knows your topic can read your alerts and send fake notifications to your phone. Make it long and random. Treat it like a password.
- The default ntfy.sh server is public infrastructure. For higher security, consider self-hosting ntfy.
- Tailscale provides authenticated, encrypted connections between your sites. Don't expose router admin interfaces to the public internet.

## License

MIT. See [LICENSE](LICENSE).
