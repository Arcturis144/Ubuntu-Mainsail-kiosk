# Ubuntu-Mainsail-kiosk
Just a little how-to to make the mainsail WebUI as a kiosk-desktop

# Full‑Screen **Mainsail** Kiosk on Ubuntu Server 22.04

> **Goal**  Turn a plain Ubuntu Server box into a one‑purpose desktop that boots straight into a **full‑screen Mainsail WebUI**.  
> Even on a 1440 × 900 (16∶10) monitor the dashboard appears in **“widescreen”** layout by rendering at 1920 × 1200 and down‑scaling *live* (DSR‑style).

---

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Install a lightweight X stack](#2-install-a-lightweight-x-stack)
3. [Auto‑login with LightDM](#3-auto-login-with-lightdm)
4. [Math § ─ pick virtual resolution & scale factor](#4-math-§-─-pick-virtual-resolution--scale-factor)
5. [Write the `mainsail‑kiosk‑session` script](#5-write-the-mainsail‑kiosk‑session-script)
6. [Remove Mainsail’s 1024 px cap](#6-remove-mainsails-1024px-cap)
7. [Enable graphics target & reboot](#7-enable-graphics-target--reboot)
8. [Adapting to any monitor](#8-adapting-to-any-monitor)
9. [Troubleshooting](#9-troubleshooting)

---

## 1 · Prerequisites

| Item | Example |
|------|---------|
| Ubuntu Server 22.04 | fresh/minimal install |
| Klipper + Moonraker + Mainsail | installed (e.g. via **KIAUH**) |
| Linux user account | `<username>` *(replace in commands)* |
| Native monitor resolution | **1440 × 900** *(16∶10)* |

> You **do not** need a full desktop; Openbox + LightDM is plenty.

---

## 2 · Install a lightweight X stack

```bash
sudo apt update && sudo apt install --no-install-recommends \
  xserver-xorg xinit openbox lightdm chromium-browser \
  x11-xserver-utils unclutter fonts-ubuntu -y
```

Add user to the required groups:

```bash
sudo usermod -aG dialout,tty,video,plugdev <username>
```

Log out & back in once so the groups take effect.

---

## 3 · Auto‑login with LightDM

```bash
sudo tee /etc/lightdm/lightdm.conf.d/50-kiosk.conf >/dev/null <<EOF
[Seat:*]
autologin-user=<username>
autologin-user-timeout=0
user-session=mainsail-kiosk
EOF
```

Add a desktop entry so LightDM can start the custom session:

```bash
sudo tee /usr/share/xsessions/mainsail-kiosk.desktop >/dev/null <<'EOF'
[Desktop Entry]
Type=Application
Name=Mainsail Kiosk
Exec=/usr/local/bin/mainsail-kiosk-session
EOF
```

---

## 4 · Math § ─ pick virtual resolution & scale factor

We want **CSS width ≥ 1536 px** so Quasar/Mainsail flips to the *widescreen* tier.

Formula (keep same aspect‑ratio):

```text
S = native_width  /  virtual_width
  = native_height /  virtual_height
```

| Native 1440 × 900 | Virtual desktop | Scale factor `S` | Notes |
|-------------------|-----------------|------------------|-------|
| (example) | **1920 × 1200** | **0.75** | 25 % smaller UI; always widescreen |
|  | 1680 × 1050 | 0.86 | 14 % smaller |
|  | 1600 × 1000 | 0.90 | 10 % smaller (minimum to hit widescreen) |

> **We’ll use 1920 × 1200 @ 0.75** below; swap your own pair if you prefer.

---

## 5 · Write the `mainsail‑kiosk‑session` script

```bash
sudo nano /usr/local/bin/mainsail-kiosk-session
```

```bash
#!/usr/bin/env bash
xset s off -dpms                 # no blanking
unclutter -idle 1 &              # hide mouse cursor

/snap/bin/chromium \
  --kiosk http://mainsail.local/ \
  --user-agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123 Safari/537.36" \
  --window-size=1920,1200 \
  --force-device-scale-factor=0.75 \
  --high-dpi-support=0 \
  --touch-events=disabled \
  --noerrdialogs --disable-translate --disable-infobars \
  --disable-session-crashed-bubble
```

```bash
sudo chmod +x /usr/local/bin/mainsail-kiosk-session
```

### 🔍 Why these flags?

| Flag | Purpose |
|------|---------|
| `--window-size=1920,1200` | Chromium renders to a 1920 × 1200 viewport. |
| `--force-device-scale-factor=0.75` | Down‑scales 0.75 × → fits 1440 × 900 hardware. |
| `--high-dpi-support=0` | Disables auto Hi‑DPI. |
| `--touch-events=disabled` | Ensures Quasar uses desktop, not tablet, heuristics. |

---

## 6 · Remove Mainsail’s 1024 px cap

> Do **once** per printer‑config folder (replace the path below).

```bash
CFG=/home/<username>/printer_Ender3v1_data/config
sudo -u <username> mkdir -p $CFG/.theme
echo '#page-container{max-width:none!important;}' | \
  sudo -u <username> tee $CFG/.theme/custom.css
```

---

## 7 · Enable graphics target & reboot

```bash
sudo systemctl set-default graphical.target
sudo systemctl restart lightdm   # or just sudo reboot
```

### Boot flow

```
BIOS → LightDM auto‑login → Openbox → mainsail‑kiosk‑session → full‑screen Mainsail (widescreen tier)
```

---

## 8 · Adapting to **any** monitor

1. Measure native resolution (e.g. 1280 × 720).  
2. Pick a virtual desktop with **same aspect‑ratio** and `width ≥ 1536`.  
3. Compute scale `S = native_width / virtual_width`.  
4. Edit **both** numbers in the script:

```bash
--window-size=<virtual_w>,<virtual_h>
--force-device-scale-factor=<S>
```

Example for 1280 × 720 → 1600 × 900:

```bash
--window-size=1600,900
--force-device-scale-factor=0.80   # 1280 / 1600
```

Mainsail will still see ≥ 1536 CSS‑px → widescreen.

---

## 9 · Troubleshooting

| Symptom | Fix |
|---------|-----|
| LightDM loops back to login | Ensure the script is executable and every back‑slash line‑continuation is present. |
| Browser opens but dashboard is < 1536 px | Double‑check scale math; make sure Hi‑DPI is off (`--high-dpi-support=0`). |
| Fonts too tiny | Move virtual res closer to native and recalc `S` (e.g. 1680 × 1050 @ 0.86). |

---
# Embedding a Web Terminal into a Mainsail Kiosk

Integrate a lightweight web‑terminal (ttyd) directly into the Mainsail UI so you can run shell commands from the same dashboard that manages your printer.

---

## Prerequisites

| Requirement              | Notes                                                 |
| ------------------------ | ----------------------------------------------------- |
| **Mainsail + Moonraker** | Already running on the host SBC/PC.                   |
| **Nginx**                | Bundled with MainsailOS/KIAUH installs.               |
| **sudo access**          | Needed for package installs and editing system files. |

> Replace bracketed tokens (e.g. `[INSTANCE_ROOT]`) with paths, user names or addresses that match **your** setup.

---

## 1  Install `ttyd`

```bash
sudo apt update
sudo apt install build-essential cmake git libjson-c-dev libwebsockets-dev

git clone https://github.com/tsl0922/ttyd.git
cd ttyd && mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

`ttyd` is now in `/usr/local/bin`.

---

## 2  Create a systemd service

```bash
sudo nano /etc/systemd/system/ttyd.service
```

```ini
[Unit]
Description=Web terminal (ttyd for Mainsail)
After=network.target

[Service]
User=[PRINTER_USER]                 # account that owns the printer instance
Group=[PRINTER_USER]
WorkingDirectory=[INSTANCE_ROOT]    # e.g. /home/pi/printer_DATA

ExecStart=/usr/local/bin/ttyd \
          --port 7681 \
          --writable \
          -c [USERNAME]:[PASSWORD] \
          --cwd [INSTANCE_ROOT] \
          bash

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ttyd.service
```

---

## 3  Reverse‑proxy `/terminal/` through Nginx

Edit the server block that already hosts Mainsail (usually `/etc/nginx/conf.d/mainsail.conf`).  Add **inside** the `server { … }` braces:

```nginx
location ^~ /terminal/ {
    proxy_pass         http://127.0.0.1:7681/;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection "upgrade";
    proxy_set_header   Host $host;
}
```

Reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

*Test*

```bash
curl -I http://<HOST-IP>/terminal/
```

Expect `HTTP/1.1 401 Unauthorized` (Basic‑Auth challenge).

---

## 4  Add a **Terminal** button to Mainsail’s sidebar

Create the theme folder if it doesn’t exist:

```bash
mkdir -p [INSTANCE_ROOT]/config/.theme
nano [INSTANCE_ROOT]/config/.theme/navi.json
```

```json
[
  {
    "title": "Terminal",
    "href": "/terminal/",
    "icon": "mdi-console",
    "position": 25
  }
]
```

Refresh Mainsail → a **Terminal** entry appears; click it and log in with `[USERNAME] / [PASSWORD]`.

---

## 5  Optional hardening

| Option                 | Snippet / Action                                                                                                                 |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **HTTPS**              | Add `listen 443 ssl; …` and point to your TLS cert.                                                                              |
| **LAN‑only**           | Inside the `location /terminal/` block:<br>`allow 192.168.0.0/24;`<br>`deny  all;`                                               |
| **Move auth to Nginx** | Remove `-c user:pass` from the service and add:<br>`auth_basic "Printer shell";`<br>`auth_basic_user_file /etc/nginx/.htpasswd;` |

---

## 6  Troubleshooting

| Check                   | Command                              | Expected               |
| ----------------------- | ------------------------------------ | ---------------------- |
| Service status          | `systemctl status ttyd`              | `active (running)`     |
| ttyd local reachability | `curl http://127.0.0.1:7681/`        | HTML with "ttyd" title |
| Proxy path              | `curl -I http://<HOST-IP>/terminal/` | `401 Unauthorized`     |

Once those tests pass, the integrated terminal is ready for daily use. Enjoy entirely browser‑based tinkering with your printer!


---

## Acknowledgements
* Drafted and refined with the help of ChatGPT.

---

### License

This guide is released under the **[Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/)**.

© 2025 [Arcturis144](https://github.com/Arcturis144) — you’re free to copy, remix, and share, provided you give appropriate credit and link to the license.

---

Enjoy your **fullscreen Mainsail** kiosk—no black borders, no tablet UI, even on smaller monitors!
