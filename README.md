# secure-browser

**A disposable, VPN-locked browser for exploring untrusted websites on a Mac.**

Google Chrome runs inside its own throwaway Linux virtual machine (Apple Container). It's shown to you as a pixel stream in Safari, so a hostile page, such as a watering-hole attack, lands in a VM that's deleted when you close it. Your Mac never parses the page.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Platform: macOS 26+ on Apple silicon](https://img.shields.io/badge/macOS-26%2B%20%C2%B7%20Apple%20silicon-lightgrey)
![Dependencies: Apple Container only](https://img.shields.io/badge/needs-Apple%20Container%20only-green)

> [!WARNING]
> **Use at your own risk.** Provided "as is", with no warranty and no liability of any kind, including for security, privacy, anonymity, computer safety, data loss or intellectual property. It reduces risk; it does not guarantee protection. See [LICENSE](LICENSE).
>
> **Please don't just trust this.** Don't take the claims here on faith. Read the code (it's one script), and run your own security scans and audits before relying on it, as you would with any security tool. If you fork or deploy it, turn on GitHub's code scanning, secret scanning and Dependabot for your copy. Found a vulnerability? Please report it privately through this repo's private vulnerability reporting (Security tab), not as a public issue or pull request. See [SECURITY.md](SECURITY.md).

---

## Why

Researching suspicious links, phishing kits or compromised sites from your everyday browser puts your Mac, accounts and identity one exploit away from the attacker. `secure-browser` puts several independent barriers in the way:

- **A separate VM per session.** It's created on launch, has a read-only system, and is deleted on exit.
- **Pixels only.** Safari shows a video of the browser. No clipboard sharing and no downloads to your Mac.
- **VPN with a kill switch.** With the bundled WireGuard setup, all traffic goes through your VPN. If the tunnel drops, there's no internet at all. Your home network and your Mac's own services are unreachable from the VM.
- **Hardened browser.** Official Google Chrome with same-day security fixes, the JavaScript JIT turned off, and enterprise policies that disable sync, sign-in, password saving, autofill, sensors, USB/Bluetooth and more.
- **Updates itself.** It rebuilds when Google or Debian ship fixes, and won't launch on an Apple Container version with known security issues; it offers to update instead.

Nothing to install but Apple's own Apple Container. **No Homebrew, no Docker, no extra viewer app.**

## Quick start

**Requirements:** a Mac with Apple silicon, macOS 26 or newer. Optional: a WireGuard config from any VPN provider (Proton VPN's free plan works).

```zsh
# 1. Get the script (read it first: it's one file)
curl -fsSL https://raw.githubusercontent.com/yshk-mxim/secure-browser/main/secure-browser -o ~/secure-browser
chmod 700 ~/secure-browser

# 2. (Recommended) Add your VPN: copy the WireGuard config text from your provider, then
~/secure-browser --import-vpn

# 3. Launch
~/secure-browser
```

The first run:

- installs or updates **Apple Container** from Apple's signed package, if needed (asks once, then asks for your admin password);
- builds the browser VM image, which takes a few minutes;
- opens Safari with the browser inside.

When macOS asks to allow **"container-runtime-linux" to find devices on local networks**, choose **Allow**. That's Apple Container relaying the viewer from `127.0.0.1` into the VM. Your network stays blocked from inside the VM either way.

Then check everything once:

```zsh
~/secure-browser --test
```

## Daily use

| Command | What it does |
|---|---|
| `~/secure-browser` | Check for updates, start a session, open the viewer. If a session is already running, reopens its viewer; if that session is out of date, offers to restart it. |
| `~/secure-browser --stop` | End the session (the VM is deleted) |
| `~/secure-browser --test` | Automated checks: tunnel, kill switch, exit IP, policy, isolation |
| `~/secure-browser --update` | Run every update check now and rebuild |
| `~/secure-browser --import-vpn` | Save a WireGuard config from the clipboard (private file, clipboard cleared) |
| `~/secure-browser --check` | Show what's installed; changes nothing |

Inside the browser, use **Ctrl** shortcuts (Ctrl-T, Ctrl-L, Ctrl-W). Closing the last browser window ends the session.

## How it works

```mermaid
flowchart LR
  subgraph Mac["Your Mac"]
    S["Safari<br/>(noVNC viewer)"]
  end
  subgraph VM["Throwaway Linux VM (Apple Container)"]
    V["Viewer services<br/>Xvnc + noVNC<br/>user 1300"]
    C["Google Chrome<br/>JIT off, policies<br/>user 1200"]
    W["WireGuard wg0<br/>kill switch"]
  end
  S -- "pixels + keyboard/mouse<br/>127.0.0.1 only" --> V
  V --- C
  C --> W
  W -- "encrypted" --> P["VPN server"] --> I["Internet"]
  C -. "blocked" .-> L["Home network /<br/>Mac services"]
```

- **Kill switch by routing.** Before the browser starts, every route in the VM is replaced with a single route to the VPN server, and everything else is sent into the tunnel. The only exception: TCP replies from the viewer's own port back to the Mac. That port is held by a separate user, so the browser can't use it to reach your Mac. The browser user can't change routes.
- **Viewer password.** A fresh random password each session, passed to Safari in the URL fragment, which isn't sent to the server.
- **Defense in depth.** To reach your Mac, an attacker needs a browser exploit, then a privilege escalation inside the VM, then an exploit of Safari or of Apple's hypervisor.

## Settings

Edit the top of `~/secure-browser`. The main options:

| Setting | Default | |
|---|---|---|
| `RESEARCH_MODE` | `1` | `1` = the hardened mode described here. `0` = convenience mode: a native window via [Cocoa-Way](https://github.com/J-x-Z/cocoa-way), with optional clipboard, downloads and audio. |
| `VIEWER` | `safari` | or `screensharing` (macOS Screen Sharing; turn off its shared clipboard each session) |
| `BROWSER` | `chrome` | Google's official Chrome, or `chromium` (Debian's build) |
| `VPN` | `1` | WireGuard with kill switch, or `0` for a direct connection |
| `UPDATE_CHECK_HOURS` | `6` | How often to check for browser and Apple Container fixes |

The [full guide](docs/GUIDE.md) covers every option, the VPN setup walkthrough, convenience mode and troubleshooting.

## Where everything comes from

| Component | Source |
|---|---|
| Apple Container | Apple's signed and notarized `.pkg`, checksum-verified against the GitHub release |
| Google Chrome | Google's signed apt repository; the signing key is checked against Google's published fingerprint |
| VM base and tools | Debian's official image and signed repositories |
| Viewer | Safari or Screen Sharing (built into macOS) |
| VPN | Your provider's WireGuard config |

## Limits: read before relying on it

- **The viewer is still an attack surface.** Safari processes the VM's video stream and the viewer page. Keep macOS updated.
- **VM escapes are rare but real.** That's why Apple Container security updates block launching.
- **Not anonymity.** A VPN hides your IP from sites, not from the VPN provider. A Linux Chrome fingerprint is uncommon. Never sign in to your real accounts in the research browser. For anonymity, use Tor Browser.
- **For the most hostile targets** (APT infrastructure, links aimed at you personally), use a disposable cloud VM or spare hardware.
- **Early project.** Shell syntax, the kill-switch and viewer routing (in Linux network namespaces), and the update and advisory logic (with mocks) have been tested; broad testing on real Macs hasn't happened yet. Run `--test` on your machine and report what you find.

## Contributing

Issues and pull requests are welcome, especially reports from different Mac models and macOS versions, Apple Container versions and VPN providers. When reporting a problem, include the output of `~/secure-browser --test`, with IP addresses removed.

## License

[MIT](LICENSE) © 2026 yshk-mxim. Free to use, change and share with attribution. **No warranty and no liability of any kind.** Chrome, Apple Container, Debian packages, noVNC, TigerVNC, WireGuard, Cocoa-Way and VPN services are third-party software under their own licenses and terms.
