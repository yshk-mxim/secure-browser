# Secure Browser: full guide

A Linux browser (Google Chrome by default, or Chromium) that runs inside its own lightweight virtual machine (Apple Container). It's built for **exploring untrusted websites in security research** without exposing your Mac. A watering-hole page that exploits the browser lands in a throwaway VM, not on your Mac.

> Tested on Linux only: script syntax, the kill switch and the viewer return path (in network namespaces), and the advisory parser. It has **not** run on a Mac yet. Run `~/secure-browser --test` after your first launch.

> **Use at your own risk.** This is provided "as is", with no warranty and no liability of any kind. That covers security, privacy, anonymity, computer safety, data loss and intellectual property. Nothing here guarantees protection against malware, exploits, tracking or identification. You are responsible for how you use it and for complying with the law and the terms of the sites and services you use. See [LICENSE](../LICENSE).

## Two modes

| | `RESEARCH_MODE=1` (default) | `RESEARCH_MODE=0` |
|---|---|---|
| Purpose | Untrusted sites, security research | Everyday private browsing |
| How you see it | **Pixel stream** in Safari (noVNC) or macOS Screen Sharing | Native macOS window via Cocoa-Way |
| Extra software on the Mac | **None** (only Apple Container) | Cocoa-Way, Waypipe (Homebrew) |
| Clipboard Mac ↔ browser | **Off** | On (can't be turned off) |
| Downloads to the Mac | **Off** | Optional (`~/Downloads/secure-browser`) |
| JavaScript JIT | **Off** (removes a major exploit class) | On |
| Profile | **Throwaway**, in RAM | Throwaway or persistent |
| System files in the VM | **Read-only**; the VM is deleted when the session ends | Normal |
| Audio | Off | Optional |

**Research mode overrides** the convenience switches (`PERSIST_PROFILE`, `INSTALL_PROTON_EXTENSION`, `ENABLE_DOWNLOADS`, `ALLOW_SITE_CLIPBOARD_API`, `ENABLE_AUDIO`). They only apply with `RESEARCH_MODE=0`.

## Why pixel streaming

In research mode your Mac only receives pictures of the browser and sends back keyboard and mouse input. Nothing on the Mac parses the hostile page itself.

- **`VIEWER=safari` (default):** the VM serves the noVNC web viewer on `127.0.0.1` only, and the launcher opens it in Safari. Safari is updated by Apple and sandboxed, and treats the viewer like any untrusted web page. It can't read your clipboard without you pasting.
- **`VIEWER=screensharing`:** opens macOS's built-in Screen Sharing app. It's also Apple-maintained, but it's a native app with no browser sandbox, and it has its own clipboard sync. **Turn off Edit > Use Shared Clipboard every session.** (Using the viewer does not turn on your Mac's own Screen Sharing server.)

Inside the VM, the viewer services run as a separate user (`vncview`, uid 1300) from the browser (`browser`, uid 1200). Clipboard sharing is disabled in the VNC server itself. To reach your Mac, an attacker would need a Chromium exploit, *plus* a privilege escalation inside the VM, *plus* a Safari or Screen Sharing exploit, *or* a VM escape.

Each session gets a new random 8-character viewer password. VNC passwords max out at 8 characters, and the VNC server rate-limits failed logins. It's passed to Safari in the URL fragment, which noVNC reads in the page and never sends to the server. It also appears in the terminal and in Safari's history for that session.

## Where everything comes from (official sources first)

| Component | Source |
|---|---|
| Apple Container | **Apple's signed, notarized `.pkg`** from Apple's GitHub, checksum-verified. On first run the launcher installs it if missing, or **replaces a Homebrew copy** with it (one question; images and volumes are kept). |
| Browser (`BROWSER=chrome`, default) | **Google Chrome for ARM64 Linux from Google's signed apt repository.** Google's signing key is checked against its published fingerprint during the build, and Chrome receives Google's same-day security fixes. |
| Browser (`BROWSER=chromium`) | Debian's official repository. Offered automatically as a fallback if Chrome can't be installed. |
| Everything else in the VM (TigerVNC, noVNC, websockify, WireGuard tools, fonts) | Debian's official signed repositories; the base is Debian's official `debian:trixie-slim` image |
| Viewer | Safari or Screen Sharing, built into macOS |
| Helper commands (`curl`, `openssl`, `pkgutil`, `spctl`, `installer`, `osascript`, `lsof`, `nc`) | Built into macOS |
| WireGuard config | Your VPN provider's dashboard (e.g. Proton) |
| Cocoa-Way and Waypipe (`RESEARCH_MODE=0` only) | The author's Homebrew tap. That's the project's only distribution besides source code, and it's used only if Homebrew is already installed. |

**Homebrew is never installed or required.** It is used only for mode-0 Cocoa-Way, and only if it's already there.

## Requirements

- Mac with Apple silicon, macOS 26 or newer
- [Apple Container](https://github.com/apple/container/releases) (install the signed `.pkg`)
- Safari, or Screen Sharing (both built in)
- Optional: a WireGuard config (Proton VPN free or paid, or any WireGuard provider)
- **Homebrew is not required.** Research mode never uses it. Only `RESEARCH_MODE=0` needs Cocoa-Way and Waypipe: if Homebrew is already installed, the script offers to install them by full name (trusting only those formulae) and keeps them updated. Without Homebrew, it tells you to install Cocoa-Way yourself or stay in research mode. It never installs Homebrew.

## Install

```zsh
cp secure-browser ~/secure-browser
chmod 700 ~/secure-browser
~/secure-browser --check
```

## Extra isolation: a separate macOS account (optional, recommended)

Running `secure-browser` from its own macOS user account, apart from your main one, adds a layer on top of the VM and the hardened browser:

1. **The browser** is sandboxed and hardened inside
2. **The VM** (Apple Container) holds the browser and is deleted on exit
3. **The macOS account** holds the VM, so a VM escape lands in an account with none of your files, keychain, browser profiles or logins

How to set it up:

1. Create a standard user in System Settings > Users & Groups (for example `research`). Don't sign in to iCloud or your real accounts there.
2. Log in as that user (Fast User Switching keeps your main session running), then follow **Install** and the VPN steps above. Everything the launcher creates (`~/secure-browser`, `~/.secure-browser-wayland/`, the image, the VPN config) stays in that account's home folder.
3. Apple Container's `.pkg` installs system-wide and needs an admin password. A standard user can't approve that: install or update it once from an admin account, or temporarily make the user an admin.
4. Launch from that account. The viewer needs that account's desktop: stay logged in on screen, or connect with Screen Sharing. Over SSH the VM still starts, and the launcher prints a login link and password instead of opening Safari (see Troubleshooting).

Know the limits:

- The viewer is bound to `127.0.0.1` on the Mac, so other accounts on the same Mac can reach the port. They still need the random per-session password, which is stored in that account's `~/.secure-browser-wayland/state/session.pw` (private to it).
- macOS account separation is a strong boundary but not a perfect one: local privilege-escalation bugs exist, so keep macOS updated.
- The account only protects your *other* accounts. Anything you do inside the research account (logins, files) is exposed to the same attacker if the browser and VM are both compromised.
- This setup hasn't been tested on real Macs yet.

## Set up the VPN (recommended)

WireGuard doesn't use your Proton username or password. The config file is the credential:

Sign in at <https://account.protonvpn.com>, go to **Downloads → WireGuard configuration**, and fill in the form:

| Step | Choose | Why |
|---|---|---|
| **1. Device/certificate name** | Something neutral, e.g. `vm-browser-1` | Avoid your name, your Mac's name or your employer. The name is how you find this config later to **extend** it (it's a certificate that expires) or **revoke** it. |
| **2. Platform** | **GNU/Linux** | The browser VM runs Debian Linux. This gives a standard WireGuard file. |
| **3. NAT-PMP (Port Forwarding)** | **Off** | Opens an inbound port on the VPN server to you. Useful for torrents and hosting; only adds exposure here (and the script doesn't run the client it needs). |
| **3. VPN Accelerator** | **On** (default) | Speed feature with no privacy downside. On the Free plan it's the only option. |
| **4. Server** | See below | The config is tied to **one** server, whose address is written into the file. |

**Choosing a server:**

- **"Best server" (e.g. `CA-FREE#18`)** is a fine start.
- **Free plan:** use the **Free server configs** tab. Free servers can be crowded, and some sites block or challenge known VPN addresses.
- **Paid plan, Secure Core configs:** traffic enters through a hardened server in a privacy-friendly country before exiting elsewhere. Someone watching or compromising the exit server can't easily trace it back to you. It's slower, but it's the strongest choice for researching hostile infrastructure.
- **Country matters:** watering-hole attacks often target visitors by country or network, so the exit country changes what a site serves you. To see a site as its targets do, you'd need an exit in their region. To stay off a targeted region's radar, pick another.
- **Several servers:** create one config per server or country, and swap the file at `~/.config/secure-browser/wireguard.conf`. The next launch uses the new one.

Then click **Create**. Proton shows the config as **text on the page**, not as a file. Save it like this:

**Recommended: import from the clipboard**

1. Select the whole config text in Proton, from `[Interface]` to the last line, and copy it (Command-C).
2. Run:
   ```zsh
   ~/secure-browser --import-vpn
   ```
   Press Return when asked. The script:
   - creates `~/.config/secure-browser/wireguard.conf` as private (`600`) from the start;
   - strips Windows line endings and stray spaces;
   - checks it's a complete config: `[Interface]`, `PrivateKey`, `[Peer]`, an IPv4 `Endpoint` and a `DNS` line;
   - asks before replacing an existing config;
   - **clears the clipboard**, since it held your private key.
3. Run `~/secure-browser --test`. It confirms the tunnel works, the exit IP differs from your Mac's, and the kill switch holds.

You can also pipe the text in, e.g. `pbpaste | ~/secure-browser --import-vpn`. That route doesn't clear the clipboard; run `pbcopy < /dev/null` afterwards.

**Manual alternative (nano)**

Create the file private *before* pasting into it, so the key is never readable by other users, not even briefly:

```zsh
mkdir -p ~/.config/secure-browser && chmod 700 ~/.config/secure-browser
( umask 077; touch ~/.config/secure-browser/wireguard.conf )
nano ~/.config/secure-browser/wireguard.conf      # paste, then Ctrl-O, Return, Ctrl-X
pbcopy < /dev/null                                # clear the clipboard
~/secure-browser --test
```

**Clipboard caution:** with Universal Clipboard (Handoff) on, anything you copy can briefly reach your other Apple devices. Clipboard-history apps (Raycast, Alfred, Paste, etc.) keep a copy until you delete it. Turn those off while copying the key, or remove the entry afterwards.

Treat this file like a password: anyone who has it can use your VPN account. Revoke it in Proton's dashboard if it leaks. The dashboard also shows when it expires; extend it there before it does, or the tunnel won't connect, and the kill switch then keeps the browser offline. With no config file, the launcher asks whether to run without the VPN this time.

## Upgrading from an earlier version (or while a browser is running)

Just replace `~/secure-browser` and run it. The first run:

1. **Rebuilds the image** for the new settings. The image name is the same as before, so the old image is replaced. A browser that is already running keeps its old copy until it restarts.
2. **In research mode, retires the old Cocoa-Way browser:**
   - if the old `Secure Chromium` window is still running, it says so and offers to stop it (recommended: it has no research protections and may have no VPN);
   - it removes the old `Secure Chromium` entry from Cocoa-Way's app list, because launching that entry from Cocoa-Way would start the new image *without* the VPN setup. A backup is kept in `~/.secure-browser-wayland/state/container-sessions.toml.bak`, and `RESEARCH_MODE=0` recreates the entry. Other Cocoa-Way sessions are untouched.
3. **Starts the research VM** and opens the viewer.

On later runs, in either mode:

| Situation | What happens |
|---|---|
| A session is running and nothing changed | Research: reopens the viewer for it. Mode 0: says it is running. |
| A session is running but the image was rebuilt (e.g. a Chromium security update) or settings changed | Tells you, and offers to **restart it with the updated version**. Declining keeps the old session (and its original viewer) until it next starts. |
| No session running | Starts one with the current image and settings |

Your old persistent profile volume (`secure-browser-profile`, with any extensions you installed by hand) is never touched by research mode. It's used again only if you go back to `RESEARCH_MODE=0` with `INSTALL_PROTON_EXTENSION=1` or `PERSIST_PROFILE=1`. Old downloads remain in `~/Downloads/selenium_downloads`.

### Should the profile persist?

Usually not. History, cookies, site data and cache are wiped on exit anyway. What a kept profile adds is extension logins, extensions you installed by hand, bookmarks, settings and site permissions. The costs:

- tampering by a malicious page or extension survives restarts;
- leftover state can link one session to the next;
- a Proton login token sits on disk and in backups.

So `auto` keeps the profile only for the Proton browser extension (`VPN=0`, `INSTALL_PROTON_EXTENSION=1`), so you don't retype its password each launch. With WireGuard (`VPN=1`) there's nothing to sign in to, so leave the extension off and the profile throwaway. To keep bookmarks without persistence, export them once to a file on the Mac. To wipe an existing kept profile, run `~/secure-browser --reset-profile`.

## Everyday use

```zsh
~/secure-browser            # check for updates, launch, open the viewer
~/secure-browser --test     # automated checks (see below)
~/secure-browser --open     # reopen the viewer for the running session (already logged in)
~/secure-browser --stop     # end the session (the research VM is deleted)
~/secure-browser --update   # run all update checks now and rebuild
~/secure-browser --check    # dependency check, no changes
~/secure-browser --uninstall   # remove what it created, at a level you choose (see Uninstall)
~/secure-browser --import-vpn      # save Proton's WireGuard text from the clipboard
~/secure-browser --reset-profile   # delete the persistent profile (mode 0 only)
```

In the research browser, use **Ctrl** shortcuts: Ctrl-T for a new tab, Ctrl-L for the address bar, Ctrl-W to close a tab. Closing the last browser window ends the session and deletes the VM. If you close the Safari tab, run `~/secure-browser` again to reopen the viewer for the running session.

## Automatic updates and security checks

Before each launch, the launcher checks for updates and applies them itself. Nothing is left for you to remember.

| What | How it's checked | What happens |
|---|---|---|
| **Chrome (or Chromium) and the VM's Linux packages** | Every `UPDATE_CHECK_HOURS` (default 6), a short-lived container asks Debian's servers whether any package in the image has an update | The image is **rebuilt automatically** with the latest browser. It's also rebuilt after `MAX_IMAGE_AGE_DAYS` (default 7) even with no detected updates. If a rebuild fails, the launch stops rather than running an unpatched browser. |
| **Apple Container** (the VM boundary) | GitHub security advisories and release notes for `apple/container`, compared with your installed version | Security fix available: the launcher **asks once to update and then does it automatically**: it downloads the latest signed installer from Apple's GitHub release, checks GitHub's published SHA-256, Apple's installer signature and notarization, stops Apple Container, installs (macOS asks for your admin password), restarts it and confirms the new version. Declining, or any failed check, means **no launch**. Non-security updates are offered too, but not required. Advisory with no fix yet: a loud warning, and you choose whether to continue. |
| **Cocoa-Way and Waypipe** (mode 0 only) | `brew outdated`, plus GitHub advisories and release notes for `J-x-Z/cocoa-way` | Outdated: **upgraded automatically**. Still affected by a known issue: mode 0 is **refused**; use research mode instead. |

If a check can't run (for example, you're offline), you get a warning and the existing versions are used. These checks run from your Mac's normal connection, not the VPN. They contact Debian's and GitHub's servers and reveal nothing about your browsing.

## Settings (top of the script)

| Setting | Default | Meaning |
|---|---|---|
| `RESEARCH_MODE` | `1` | Hardened pixel-streaming mode (see above) |
| `VIEWER` | `safari` | `safari` or `screensharing` |
| `BROWSER` | `chrome` | `chrome` = Google's official Chrome (same-day security fixes); `chromium` = Debian's build |
| `VPN` | `1` | WireGuard with kill switch; `0` = normal network |
| `WIREGUARD_CONF` | `~/.config/secure-browser/wireguard.conf` | WireGuard file location |
| `UPDATE_CHECK_HOURS` | `6` | How often update and advisory checks run |
| `MAX_IMAGE_AGE_DAYS` | `7` | Rebuild at least this often |
| `PERSIST_PROFILE` | `auto` | Mode 0 only. `auto` = kept **only** when `INSTALL_PROTON_EXTENSION=1` (so the extension login survives), otherwise throwaway. `1` = always keep, `0` = always throwaway. |
| `INSTALL_PROTON_EXTENSION` | `0` | Mode 0 only |
| `ENABLE_DOWNLOADS` | `1` | Mode 0 only |
| `ALLOW_SITE_CLIPBOARD_API` | `1` | Mode 0 only |
| `ENABLE_AUDIO` | `1` | Mode 0 only. Needs `cocoa-way-audio-relay` in `~/.secure-browser-wayland/hardened-build/` |

Changing a setting rebuilds the image automatically on the next launch. Stop the browser first with `--stop`.

## How the network is locked down

With `VPN=1`, before the browser starts:

1. Root inside the VM (which can't touch the Mac) brings up WireGuard: kernel WireGuard if the VM has it, otherwise `wireguard-go`.
2. **All routes are deleted** and replaced with only:
   - one route to the VPN server;
   - "everything else into the tunnel": internet, home network, router, other devices, and the Mac's own services;
   - in research mode, one narrow exception so the viewer works. TCP replies from the viewer's listening port (6080 for Safari, 5901 for Screen Sharing) go back to the Mac. This is a routing rule matching TCP source port only. The port is held by the viewer service (uid 1300), so the browser (uid 1200) can't borrow it to reach the Mac. UDP from that port and all other traffic still go into the tunnel.
3. DNS goes only to the VPN's resolver, through the tunnel.
4. With no WireGuard handshake within 15 seconds, the browser doesn't start.
5. The browser runs as uid 1200 without network-admin rights, so it can't change any of this. If the tunnel drops later, there's no route out, and nothing leaks.

On the Mac side, the viewer port is published on `127.0.0.1` only, so other machines on your network can't connect. macOS's firewall isn't involved and doesn't need changes. Any `PreUp`/`PostUp` commands in the WireGuard file are ignored.

## What `--test` checks

1. The image has an unprivileged user, Chromium, the viewer components and the sandbox helper.
2. The policy inside the image matches your settings (JIT, clipboard, downloads).
3. With the VPN on:
   - the tunnel comes up and the browser's exit IP differs from your Mac's;
   - the browser user can't read the WireGuard key;
   - home-network addresses and the Mac's address go into the tunnel;
   - in research mode, only the viewer's TCP replies take the direct route to the Mac;
   - with the tunnel down, nothing gets out.
4. In research mode, while running: the viewer port is bound to `127.0.0.1` only.

Then check by hand in the research browser:

1. `chrome://policy`: no errors. `DefaultJavaScriptJitSetting` should be 2.
2. `chrome://sandbox`: all green. If not, don't browse hostile sites.
3. A DNS/WebRTC leak-test page: only VPN addresses should appear.
4. Copy text on the Mac and try to paste it in the browser, then the reverse. Neither should work.

## What's left (residual risks)

- **The viewer is still an attack surface:** Safari (noVNC) or Screen Sharing processes data from the VM. Safari is the stronger choice because of its sandbox. Keep macOS updated; that also updates both viewers.
- **The VM boundary** (Apple's hypervisor and Apple Container) is strong, but escapes do happen. That's why its security updates block launching.
- **Patch gap:** with `BROWSER=chrome`, fixes arrive the same day Google releases them, and the automatic checks pick them up within `UPDATE_CHECK_HOURS`. With `BROWSER=chromium`, Debian can trail by days. JIT-off protects against many exploits in between.
- **Fingerprint:** Linux Chromium on Apple silicon is uncommon. That's useful against watering holes that target your real profile, but it doesn't make you anonymous.
- **You:** don't sign in to real accounts in the research browser, and don't retype hostile content on the Mac.
- **The most hostile targets** (active APT infrastructure, links aimed at you personally): use a disposable cloud VM or a spare machine. If you want to *capture* an attack rather than avoid it, you need a different setup (a honeyclient with traffic capture).

## Troubleshooting

- **macOS asks "Allow 'container-runtime-linux' to find devices on local networks?"** Choose **Allow**. It's Apple Container's helper relaying the viewer from `127.0.0.1` on your Mac into the VM. Your home network stays blocked by the kill switch inside the VM either way. If you chose Don't Allow and the viewer won't load, turn it on in **System Settings > Privacy & Security > Local Network**.
- **"Couldn't install Google Chrome":** Google's repository was unreachable or had no ARM64 package. The launcher offers Debian's Chromium for that build; set `BROWSER=chromium` to make that permanent.
- **"Apple Container is still 1.x (wanted 1.y)" after an update:** you have two copies of the `container` command, usually an old Homebrew one (`/opt/homebrew/bin/container`) ahead of Apple's installed one (`/usr/local/bin/container`) in PATH. The launcher now detects this and offers to remove the Homebrew copy; your images and volumes are kept. To check by hand: `whence -ap container`.
- **"Couldn't open Safari automatically" / "Domain does not support specified action":** macOS only lets a program open apps when it runs inside a logged-in desktop session. Over SSH, or from an account that isn't logged in on screen, it refuses. The browser VM is fine; only the automatic Safari step failed. The launcher prints a ready-to-use link that logs you in, plus the plain address and password. Paste the link into Safari while logged in on that Mac's screen (directly or through Screen Sharing), or run `~/secure-browser --open` from a Terminal in the desktop session. The password is also in `~/.secure-browser-wayland/state/session.pw`. The Screen Sharing viewer gets the same fallback.
- **Viewer page doesn't load:** run `container logs secure-browser-research`. `secure-browser-vpn:` errors mean the kill switch kept the browser offline as designed. Check the WireGuard file, or test with `VPN=0`.
- **"kernel lacks port-based routing rules":** the VM kernel can't run the viewer behind the kill switch. Please report it. As a temporary workaround, `VPN=0` works, but the browser can then reach your network.
- **Wrong window size:** resize the Safari window. The viewer asks the VM to match it.
- **Keyboard shortcuts:** use Ctrl instead of Command inside the browser.

## Uninstall

`~/secure-browser --uninstall` is interactive. It asks how much to remove, prints the exact list, and deletes nothing until you confirm. Levels are cumulative:

1. **Session only:** stop the browser, delete its VM (`secure-browser-research`) and the session state. The image stays, so relaunching is fast.
2. **+ Browser image:** also delete `localhost/secure-browser-hardened`, the build folder (`~/.secure-browser-wayland/hardened-build`) and stop the image builder (`buildkit`). Relaunching rebuilds the image.
3. **+ Data:** also delete the persistent profile volume (`secure-browser-profile`, mode 0 only), the `Secure Chromium` entry in Cocoa-Way's `container-sessions.toml`, and the whole `~/.secure-browser-wayland` folder (update times, the working VPN copy, logs).
4. **+ Everything:** also delete your saved WireGuard config (`~/.config/secure-browser/wireguard.conf`) and the script itself.

Never removed, because they're shared or not ours: **Apple Container** (system-wide; use the uninstall script from [`apple/container`](https://github.com/apple/container)), **Homebrew packages** (Cocoa-Way and Waypipe, if you installed them for mode 0), the **Debian base image** (`container image delete debian:trixie-slim` if nothing else uses it) and your **`~/Downloads/secure-browser`** folder. The end of the run lists these with the commands. Clear any `127.0.0.1` entries from Safari's history yourself.

If you run it from a separate macOS account, run `--uninstall` as that account; everything it creates is in that account's home folder and container store. The WireGuard config is a credential: when you're done with the VPN account, revoke the config in your provider's dashboard too.

## Files

| Path | Purpose |
|---|---|
| `~/secure-browser` | The launcher |
| `~/.config/secure-browser/wireguard.conf` | Your WireGuard config (`chmod 600`) |
| `~/.secure-browser-wayland/hardened-build/` | Generated image files |
| `~/.secure-browser-wayland/vpn/` | Copy of the WireGuard config mounted into the VM (read-only) |
| `~/.secure-browser-wayland/state/` | Update-check times, current session password |
| `~/.config/cocoa-way/container-sessions.toml` | Mode 0 only. Only the `Secure Chromium` entry is changed. |
| `LICENSE` | MIT license (free to use, change and share with attribution; no warranty or liability) |

## References

- [Apple Container](https://github.com/apple/container) and its [command reference](https://github.com/apple/container/blob/main/docs/command-reference.md)
- [noVNC URL parameters](https://github.com/novnc/noVNC/blob/master/docs/EMBEDDING.md)
- [TigerVNC Xvnc options](https://tigervnc.org/doc/Xvnc.html)
- [Chrome policy: DefaultJavaScriptJitSetting](https://chromeenterprise.google/policies/#DefaultJavaScriptJitSetting)
- [GitHub repository security advisories API](https://docs.github.com/en/rest/security-advisories/repository-advisories)
- [Proton: WireGuard configuration files](https://protonvpn.com/support/wireguard-configurations/)
- [Cocoa-Way](https://github.com/J-x-Z/cocoa-way) (mode 0)

## License

[MIT](../LICENSE). You may use, copy, change, merge, publish, distribute, sublicense and sell copies, as long as the copyright notice and license text are kept (attribution). The software is provided **"as is", without warranty of any kind**, and the authors are **not liable** for any claim, damage or other liability arising from it. That includes security failures, compromise of your computer, privacy or identity exposure, data loss, and intellectual-property claims.

Third-party components this script installs or downloads (Apple Container, Debian, Chromium, TigerVNC, noVNC, WireGuard, Cocoa-Way, Proton VPN and others) are under their own licenses and terms.

