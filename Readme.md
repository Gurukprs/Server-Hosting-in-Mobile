## Capability

* ✔ Host **multiple frontend apps** (React, *after* `npm run build`)
* ✔ Host **backend services** (like Render-style APIs)
* ✔ Both **publicly accessible 24/7**
* ✔ Phone plugged in, **screen off forever**
* ✔ No concern about battery life or degradation
* ✔ Strong build → **not fragile, not hacky**

## 🧱 PHASE 1 — FOUNDATION (MOST CRITICAL PHASE)

> If Phase 1 is solid, everything above it becomes stable.
> If Phase 1 is weak, the whole system becomes fragile.

## 1️⃣ Install & HARDEN Termux (Non-negotiable)

### ✅ Use **F-Droid Termux only**

⚠️ **Do NOT use Play Store Termux** (deprecated, broken)

* Install **F-Droid**
* From F-Droid install:

  * **Termux**
  * (Later) Termux:Boot

---

### 1.1 Initial setup (run once)

```bash
pkg update && pkg upgrade -y
pkg install -y git curl wget unzip nano vim
```

---

### 1.2 Grant storage access (important for SD card)

```bash
termux-setup-storage
```

This creates:

```
/storage/emulated/0/
```

Your SD card will appear as:

```
/storage/XXXX-XXXX/
```

(We’ll mount projects there later.)

---

## 2️⃣ CRITICAL Android-side settings (do this NOW)

### 🔴 These prevent Android from killing your server

Go to **Settings → Apps → Termux**:

* Battery → **Unrestricted**
* Allow background activity → **ON**
* Data usage → Allow background data
* Remove all power-saving restrictions

Then:

* Enable **Developer options**
* Turn **OFF**:

  * “Suspend execution for cached apps”
  * Any OEM battery killer options

📌 This step alone prevents 80% of failures.

---

## 3️⃣ Base Directory Layout (Anti-fragile design)

We separate **code**, **static files**, and **logs**.

### 📁 Create root layout

```bash
mkdir -p ~/infra/{bin,logs,run}
mkdir -p ~/apps/{backends,frontends}
```

Result:

```
~/infra/
 ├─ bin/        # startup scripts
 ├─ logs/       # rotated logs
 └─ run/        # pid files

~/apps/
 ├─ backends/
 └─ frontends/
```

---

## 4️⃣ SD Card strategy (low internal storage safe)

### 🔹 Rule

* **Backends** → internal storage (safer)
* **Frontend static builds** → SD card (read-heavy)

### Example SD structure

```bash
mkdir -p /storage/XXXX-XXXX/www/{site1,site2}
```

Later we’ll map:

```
/storage/XXXX-XXXX/www/site1 → frontend #1
/storage/XXXX-XXXX/www/site2 → frontend #2
```

No writes at runtime → SD card stays healthy.

---

## 5️⃣ Install runtimes (backend foundation)

### 5.1 Python (for APIs & static serving)

```bash
pkg install -y python
python --version
```

---

### 5.2 Node.js (for Express & APIs)

```bash
pkg install -y nodejs
node -v
npm -v
```

⚠️ **No frontend tooling here**
Node is **backend-only**.

---

## 6️⃣ Process resilience (NO PM2, Android-safe)

PM2 is **NOT recommended** on Android (fragile).
We’ll use **shell-level watchdog scripts** (much safer).

### 6.1 Install `nohup` & `procps`

```bash
pkg install -y procps
```

---

## 7️⃣ Termux auto-start on boot (mandatory for 24/7)

Install:

* **Termux:Boot** (from F-Droid)

Then:

```bash
mkdir -p ~/.termux/boot
```

Create bootstrap file:

```bash
nano ~/.termux/boot/start.sh
```

Paste:

```bash
#!/data/data/com.termux/files/usr/bin/bash
termux-wake-lock
bash ~/infra/bin/start-all.sh >> ~/infra/logs/boot.log 2>&1
```

Make executable:

```bash
chmod +x ~/.termux/boot/start.sh
```

📌 This guarantees:

* Power cut
* Phone reboot
* Wi-Fi reconnect
  → **Services come back automatically**

---

## 8️⃣ Placeholder master startup script

Create:

```bash
nano ~/infra/bin/start-all.sh
```

Paste (for now):

```bash
#!/data/data/com.termux/files/usr/bin/bash

echo "[BOOT] $(date)"

# Backend services will go here
# Static servers will go here
# Cloudflare tunnel will go here

echo "[BOOT COMPLETE]"
```

Make executable:

```bash
chmod +x ~/infra/bin/start-all.sh
```

We’ll fill this **incrementally** (safe approach).

---

## 9️⃣ Screen OFF + headless operation

Once this phase is done:

* Use `scrcpy --turn-screen-off`
* Or simply lock screen OFF (no lock set)

All services:

* Survive screen OFF
* Survive USB disconnect
* Survive reboot

---

## ✅ PHASE 1 COMPLETION CHECKLIST

Before proceeding, confirm:

* [ ] Termux from **F-Droid**
* [ ] Battery optimization **disabled**
* [ ] Storage access granted
* [ ] Directory structure created
* [ ] Python + Node installed
* [ ] Termux:Boot installed
* [ ] Auto-start script in place

# 🚀 PHASE 2 — BACKEND SERVICES (HARDENED & AUTO-RECOVERING)

## 1️⃣ Backend directory structure (clean separation)

```bash
mkdir -p ~/apps/backends/{node,python}
```

Result:

```
~/apps/backends/
 ├─ node/
 └─ python/
```

---

## 2️⃣ NODE BACKEND (Express-style)

### 2.1 Minimal Express service

```bash
cd ~/apps/backends/node
npm init -y
npm install express
```

Create `server.js`:

```bash
nano server.js
```

Paste:

```js
const express = require("express");
const app = express();

const PORT = 5000;

app.get("/health", (req, res) => {
  res.json({ status: "ok", service: "node-backend" });
});

app.get("/api/hello", (req, res) => {
  res.json({ message: "Hello from Node backend" });
});

app.listen(PORT, "0.0.0.0", () => {
  console.log(`Node backend running on ${PORT}`);
});
```

Test manually:

```bash
node server.js
```

Visit:

```
http://localhost:5000/health
```

✔ If this works → stop it (`Ctrl+C`)

---

## 3️⃣ PYTHON BACKEND (Flask – lightweight)

### 3.1 Install Flask

```bash
pip install flask
```

Create `app.py`:

```bash
cd ~/apps/backends/python
nano app.py
```

Paste:

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/health")
def health():
    return jsonify(status="ok", service="python-backend")

@app.route("/api/hello")
def hello():
    return jsonify(message="Hello from Python backend")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001)
```

Test:

```bash
python app.py
```

Visit:

```
http://localhost:5001/health
```

✔ Works → stop it

---

## 4️⃣ AUTO-RESTART WATCHDOG (THIS IS THE CORE)

We **do not** use PM2.
We use **infinite-loop watchdog scripts** — proven reliable on Android.

---

### 4.1 Node watchdog

```bash
nano ~/infra/bin/node-backend.sh
```

Paste:

```bash
#!/data/data/com.termux/files/usr/bin/bash

NODE=/data/data/com.termux/files/usr/bin/node

while true; do
  echo "[NODE] starting $(date)"
  $NODE /data/data/com.termux/files/home/apps/backends/node/server.js >> /data/data/com.termux/files/home/infra/logs/node.log 2>&1
  echo "[NODE] crashed, restarting in 2s"
  sleep 2
done

```

```bash
chmod +x ~/infra/bin/node-backend.sh
```

---

### 4.2 Python watchdog

```bash
nano ~/infra/bin/python-backend.sh
```

Paste:

```bash
#!/data/data/com.termux/files/usr/bin/bash

PY=/data/data/com.termux/files/usr/bin/python

while true; do
  echo "[PYTHON] starting $(date)"
  $PY /data/data/com.termux/files/home/apps/backends/python/app.py >> /data/data/com.termux/files/home/infra/logs/python.log 2>&1
  echo "[PYTHON] crashed, restarting in 2s"
  sleep 2
done

```

```bash
chmod +x ~/infra/bin/python-backend.sh
```

---

## 5️⃣ Register backends in master startup script

Edit:

```bash
nano ~/infra/bin/start-all.sh
```

Update it to:

```bash
#!/data/data/com.termux/files/usr/bin/bash

echo "[BOOT] $(date)"

nohup bash ~/infra/bin/node-backend.sh &
nohup bash ~/infra/bin/python-backend.sh &

echo "[BOOT COMPLETE]"
```

---

## 6️⃣ TEST AUTO-RECOVERY (MANDATORY)

### 6.1 Start everything manually

```bash
bash ~/infra/bin/start-all.sh
```

Check:

```
http://localhost:5000/health
http://localhost:5001/health
```

---

### 6.2 Kill a backend (test resilience)

```bash
pkill node
```

Wait 2 seconds → refresh `/health`

✔ It should come back automatically
If it does → **your backend layer is SOLID**

---

## ✅ PHASE 2 COMPLETION CHECKLIST

Confirm:

* [ ] Node backend responds on port 5000
* [ ] Python backend responds on port 5001
* [ ] Killing process auto-restarts it
* [ ] Logs are being written
* [ ] start-all.sh launches both

# 🧱 PHASE 3 — STATIC FRONTEND HOSTING (NON-FRAGILE)

## 1️⃣ SD CARD FRONTEND LAYOUT (CRITICAL)

Assume your SD card path is:

```
/storage/XXXX-XXXX/
```

Create this once:

```bash
mkdir -p /storage/XXXX-XXXX/www
```

This will hold **all phone-hosted frontends**.

### Example layout (scales well)

```
/storage/XXXX-XXXX/www/
 ├─ site1/
 │   ├─ index.html
 │   └─ static/
 ├─ site2/
 │   ├─ index.html
 │   └─ static/
 └─ site3/
```

Each folder = one React app (`npm run build` output).

---

## 2️⃣ BUILD FRONTENDS (ON YOUR PC ONLY)

On your PC (never on phone):

```bash
npm run build
```

Then copy **contents of `build/`**, not the folder itself:

```
build/*
   ↓
/storage/XXXX-XXXX/www/site1/
```

Repeat for each site.

⚠️ Important:

* Use **HashRouter** in React if needed
* No server-side routing
* No `.env` secrets in frontend

---

## 3️⃣ STATIC SERVER (SINGLE INSTANCE)

We’ll serve **everything** from `/www`.

### 3.1 Create frontend server script

```bash
nano ~/infra/bin/frontend-server.sh
```

Paste:

```bash
#!/data/data/com.termux/files/usr/bin/bash

FRONTEND_ROOT="/storage/sdcard1/www"
LOG="$HOME/infra/logs/frontend.log"

echo "[FRONTEND] boot $(date)" >> "$LOG"

if [ ! -d "$FRONTEND_ROOT" ]; then
  echo "[FRONTEND] ERROR: $FRONTEND_ROOT not found" >> "$LOG"
  exit 1
fi

cd "$FRONTEND_ROOT" || exit 1

while true; do
  echo "[FRONTEND] starting http.server $(date)" >> "$LOG"
  python -m http.server 8000 >> "$LOG" 2>&1
  echo "[FRONTEND] crashed, restarting in 2s $(date)" >> "$LOG"
  sleep 2
done

```

Make executable:

```bash
chmod +x ~/infra/bin/frontend-server.sh
```

---

## 4️⃣ URL STRUCTURE (IMPORTANT)

With this setup:

```
http://PHONE_IP:8000/site1/
http://PHONE_IP:8000/site2/
```

After Cloudflare Tunnel (next phase):

```
https://your-domain/site1/
https://your-domain/site2/
```

✔ Scales to unlimited sites
✔ No extra ports
✔ No extra servers

---

## 5️⃣ REGISTER FRONTEND SERVER IN BOOT SEQUENCE

Edit master startup:

```bash
nano ~/infra/bin/start-all.sh
```

Final version now:

```bash
#!/data/data/com.termux/files/usr/bin/bash

echo "[BOOT] $(date)"

nohup bash ~/infra/bin/node-backend.sh &
nohup bash ~/infra/bin/python-backend.sh &
nohup bash ~/infra/bin/frontend-server.sh &

echo "[BOOT COMPLETE]"
```

---

## 6️⃣ TEST LOCALLY (MANDATORY)

Run:

```bash
bash ~/infra/bin/start-all.sh
```

Check from phone browser or PC (same Wi-Fi):

* `http://PHONE_IP:8000/site1/`
* `http://PHONE_IP:8000/site2/`
* Backend still works:

  * `:5000/health`
  * `:5001/health`

✔ All must work together

---

## ✅ PHASE 3 COMPLETION CHECKLIST

Confirm:

* [ ] React builds copied to SD card
* [ ] Frontends accessible at `/siteX/`
* [ ] Frontend server auto-restarts
* [ ] start-all.sh launches everything
* [ ] Screen can be OFF

---

Great — thanks for pasting the full Phase 3 context.
Since you’ve **actually completed Phase 3 correctly**, we can now move to the **most sensitive but also most powerful step**.

---

# 🌍 PHASE 4 — INTERNET EXPOSURE (CLOUDFLARE TUNNEL, HARDENED)

## 🔑 CORE DECISION (LOCKED)

* Tunnel provider: **Cloudflare**
* Tunnel type: **Named tunnel**
* Tunnel runs **on the phone**
* Phone exposes:

  * Frontends (`:8000`)
  * Backends (`:5000`, `:5001`)
* Cloudflare handles:

  * HTTPS
  * DNS
  * Reconnects
  * Public routing

---

## 1️⃣ Install `cloudflared` (Termux)

```bash
pkg install -y cloudflared
cloudflared --version
```

✔ If version prints → continue
❌ If not → stop and fix before proceeding

---

## 2️⃣ Authenticate Cloudflare (ONE TIME ONLY)

Run:

```bash
cloudflared tunnel login
```

What happens:

* Browser opens
* Log in to Cloudflare
* Select your account
* Approve access

Cloudflare will store credentials at:

```
~/.cloudflared/cert.pem
```

✔ This step is **never repeated**

---

## 3️⃣ Create a NAMED tunnel (IMPORTANT)

Choose a **stable name**, e.g.:

```bash
cloudflared tunnel create phone-server
```

Output will show:

* Tunnel UUID
* Tunnel credentials JSON

Example:

```
Tunnel ID: a1b2c3d4-xxxx
```

A file will be created:

```
~/.cloudflared/a1b2c3d4-xxxx.json
```

⚠️ Do NOT delete this file

---

## 4️⃣ Cloudflare Tunnel config (CRITICAL FILE)

Create config directory (if not exists):

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

### ✅ Paste THIS (adjust domains later)

```yaml
tunnel: phone-server
credentials-file: /data/data/com.termux/files/home/.cloudflared/phone-server.json

ingress:
  # Frontend sites (static)
  - hostname: phone.yourdomain.com
    service: http://localhost:8000

  # Node backend
  - hostname: api.yourdomain.com
    service: http://localhost:5000

  # Python backend
  - hostname: pyapi.yourdomain.com
    service: http://localhost:5001

  # Catch-all (required)
  - service: http_status:404
```

⚠️ Replace:

* `yourdomain.com` → your real domain
  (or keep `*.trycloudflare.com` later if you want)

---

## 5️⃣ DNS ROUTING (Cloudflare Dashboard)

For each hostname, run:

```bash
cloudflared tunnel route dns phone-server phone.yourdomain.com
cloudflared tunnel route dns phone-server api.yourdomain.com
cloudflared tunnel route dns phone-server pyapi.yourdomain.com
```

Cloudflare will:

* Create DNS records automatically
* Bind them to your tunnel
* No manual DNS edits needed

✔ This is what makes it **robust**

---

## 6️⃣ Test tunnel manually (MANDATORY)

Before automating, test:

```bash
cloudflared tunnel run phone-server
```

Now test from **any internet connection**:

* `https://phone.yourdomain.com/site1/`
* `https://api.yourdomain.com/health`
* `https://pyapi.yourdomain.com/health`

✔ If all work → proceed
❌ If not → STOP and debug (do not continue)

---

## 7️⃣ Auto-start tunnel on boot (24/7 guarantee)

Create tunnel watchdog:

```bash
nano ~/infra/bin/cloudflare-tunnel.sh
```

Paste:

```bash
#!/data/data/com.termux/files/usr/bin/bash

LOG="$HOME/infra/logs/cloudflared.log"

while true; do
  echo "[CLOUDFLARE] starting $(date)" >> "$LOG"
  cloudflared tunnel run phone-server >> "$LOG" 2>&1
  echo "[CLOUDFLARE] crashed, restarting in 5s $(date)" >> "$LOG"
  sleep 5
done
```

Make executable:

```bash
chmod +x ~/infra/bin/cloudflare-tunnel.sh
```

---

## 8️⃣ Register tunnel in master boot script

Edit:

```bash
nano ~/infra/bin/start-all.sh
```

Final **Phase-4 version**:

```bash
#!/data/data/com.termux/files/usr/bin/bash

echo "[BOOT] $(date)"

nohup bash ~/infra/bin/node-backend.sh &
nohup bash ~/infra/bin/python-backend.sh &
nohup bash ~/infra/bin/frontend-server.sh &
nohup bash ~/infra/bin/cloudflare-tunnel.sh &

echo "[BOOT COMPLETE]"
```

---

## 9️⃣ FAILURE TESTS (NON-OPTIONAL)

Do **all** of these:

### 🔹 Test 1 — Reboot phone

* Power OFF → ON
* Do nothing
* Wait 2–3 minutes
* Check URLs from another network

### 🔹 Test 2 — Kill tunnel

```bash
pkill cloudflared
```

Wait → URL should recover

### 🔹 Test 3 — Wi-Fi OFF → ON

Tunnel must reconnect automatically

✔ If all pass → system is **strong**

---

## ✅ PHASE 4 COMPLETION CHECKLIST

Confirm:

* [ ] Frontend accessible via HTTPS
* [ ] Node API accessible via HTTPS
* [ ] Python API accessible via HTTPS
* [ ] Tunnel restarts automatically
* [ ] Works after reboot
* [ ] Screen stays OFF

---

## 🏁 WHAT YOU HAVE BUILT (REALITY CHECK)

You now have:

* A **self-healing server**
* Running on a **Android phone**
* Hosting:

  * Multiple frontends
  * Multiple backends
* Publicly reachable
* HTTPS secured
* No cloud VM costs
* No sleep / cold starts

This is **not a toy setup** — it’s a legitimate edge server.

---

Congratulation, You’ve built something genuinely impressive here 👏🎉
