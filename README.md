# Ubuntu-Mainsail-kiosk
Just a little how-to to make the mainsail WebUI as a kiosk-desktop

0 · What you start with
Ubuntu Server 22.04 (fresh or headless)

Klipper + Moonraker + Mainsail already installed (e.g. via KIAUH)

Your Linux user → <username>

Folder that holds printer.cfg (example uses /home/<username>/printer_Ender3v1_data/config/)

Monitor’s native resolution (example: 1440 × 900, 16 : 10)

1 · Install the lightweight graphics stack

sudo apt update
sudo apt install --no-install-recommends \
  xserver-xorg xinit openbox lightdm chromium-browser \
  x11-xserver-utils unclutter fonts-ubuntu -y
Give your user the needed groups:

sudo usermod -aG dialout,tty,video,plugdev <username>

2 · Make LightDM auto‑login and launch a kiosk session

sudo tee /etc/lightdm/lightdm.conf.d/50-kiosk.conf >/dev/null <<EOF
[Seat:*]
autologin-user=<username>
autologin-user-timeout=0
user-session=mainsail-kiosk
EOF
Desktop entry:

sudo tee /usr/share/xsessions/mainsail-kiosk.desktop >/dev/null <<'EOF'
[Desktop Entry]
Type=Application
Name=Mainsail Kiosk
Exec=/usr/local/bin/mainsail-kiosk-session
EOF
3 · Maths: choose a “virtual desktop” and one scale factor 
𝑆
S
𝑆
=
native width
virtual width
  
=
  
native height
virtual height
S= 
virtual width
native width
​
 = 
virtual height
native height
​
 
Keep the same aspect ratio (otherwise fonts distort).
Make sure virtual width ≥ 1536 px so Mainsail enters widescreen.

Native 1440 × 900	Virtual desktop	Scale factor 
𝑆
S	Comments
(example)	1920 × 1200	0.75	25 % shrink, guaranteed widescreen
1680 × 1050	0.86	14 % shrink
1600 × 1000	0.90	10 % shrink (minimum to hit widescreen)

Pick what feels comfortable. We’ll use 1920 × 1200 @ 0.75 in the script below.

4 · Write the kiosk session script
bash
Copy
Edit
sudo nano /usr/local/bin/mainsail-kiosk-session
bash
Copy
Edit
#!/usr/bin/env bash
xset s off -dpms
unclutter -idle 1 &

/snap/bin/chromium \
  --kiosk http://mainsail.local/ \
  --user-agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123 Safari/537.36" \
  --window-size=1920,1200 \          # ← your chosen virtual desktop
  --force-device-scale-factor=0.75 \ # ← your S value
  --high-dpi-support=0 \
  --touch-events=disabled \
  --noerrdialogs --disable-translate --disable-infobars \
  --disable-session-crashed-bubble
bash
Copy
Edit
sudo chmod +x /usr/local/bin/mainsail-kiosk-session
What happens?

Chromium renders at 1920 × 1200.

scale 0.75 collapses that to exactly 1440 × 900 device pixels.

Mainsail sees ≥ 1536 px ⇒ switches to widescreen tier (no black bars, right drawer docks beside main panel).

5 · (One‑liner) Remove Mainsail’s width cap forever
bash
Copy
Edit
CFG=/home/<username>/printer_Ender3v1_data/config
sudo -u <username> mkdir -p $CFG/.theme
echo '#page-container{max-width:none!important;}' | \
  sudo -u <username> tee $CFG/.theme/custom.css
6 · Enable graphics target & reboot
bash
Copy
Edit
sudo systemctl set-default graphical.target
sudo systemctl restart lightdm   # or just sudo reboot
7 · Adapting to any monitor
Measure native resolution (e.g. 1280 × 720, 1600 × 900, 1920 × 1080).

Pick a virtual desktop ≥ 1536 px wide, same aspect‑ratio.

Compute 
𝑆
S once:
S = native_width ÷ virtual_width

Edit both numbers in the --window-size= and the --force-device-scale-factor= line.

Example for a 1280 × 720 panel you want to upscale to 1600 × 900:

ini
Copy
Edit
--window-size=1600,900
--force-device-scale-factor=0.80
(1600 × 0.80 = 1280, 900 × 0.80 = 720, CSS width = 1600 ≥ 1536 ⇒ widescreen.)

Done!
Power → login splash → black flash → full‑screen Mainsail in widescreen layout every time, even on panels smaller than 1536 px wide. Share, fork, remix at will.
