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