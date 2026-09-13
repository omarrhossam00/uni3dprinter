# University 3D Printing Service — Cloud & IoT Architecture

A full-stack, cloud-ready application for managing university 3D-print requests:
- **Students** upload STL files, receive automatic volume and price estimates, submit print requests, and track real-time printing progress.
- **Staff / Admin** review orders, override pricing, preview models in 3D (Three.js), manage a printer queue, and monitor a live fleet of physical 3D printers.
- **Physical 3D Printers** connect via dedicated **ESP32 microcontrollers** over university Wi-Fi, reporting telemetry (nozzle/bed temperatures, progress) and claiming print jobs via an outbound HTTPS connection.

---

## 🏗️ Architecture

```
Student / Admin (Anywhere, Any Device)
      |
    HTTPS
      v
Cloud Frontend (React + Vite)
      |
    HTTPS REST API (Rate limiting, CORS, JWT Auth)
      v
Cloud Backend (Node.js / Express)
   ├── Pluggable Storage (Local Disk for Dev ↔ S3 / Cloudflare R2 for Cloud)
   ├── Database Adapter (SQLite for Dev ↔ PostgreSQL for Cloud)
   ├── Device Auth (Unique SHA-256 API Keys per printer)
   └── Printer Fleet & Job Queue Manager
      ^
      | Outbound HTTPS Polling (No open inbound ports on campus!)
University Wi-Fi
      |
[ ESP32 Microcontroller ]
      | Serial / UART (G-code commands & telemetry)
[ 3D Printer (Marlin / Klipper / RepRap) ]
```

---

## 📁 Project Layout

```
uni3dprint/
  backend/
    data/                 Local SQLite database directory (uni3dprint.db)
    db/
      index.js            Dual-database adapter (SQLite / PostgreSQL)
      schema.js           Centralized DDL definitions
    middleware/
      auth.js             JWT verification for Staff / Admin
      deviceAuth.js       x-device-id + x-api-key verification for ESP32 devices
    routes/
      admin.js            Staff review, price overrides, printer fleet & job queue
      devices.js          ESP32 IoT layer (heartbeats, job claim, progress, file streaming)
      orders.js           Student STL upload, pricing estimate, lookup & order creation
    scripts/
      mock_esp32.js       Standalone ESP32 simulator for testing without hardware
    storage/
      index.js            Storage abstraction (Local Disk ↔ S3 / R2 / Supabase)
    uploads/              Local file uploads (orders, temporary meshes)
    utils/
      estimate.js         Volume & print price calculation logic
      stlParser.js        Signed-tetrahedron STL binary & ASCII mesh parser
      upload.js           Multer configuration & size validation
    server.js             Express application entry point with rate limiters & security
  frontend/
    src/
      components/
        STLViewer.jsx     Three.js in-browser STL 3D renderer
      pages/
        AdminDashboard.jsx Order list, 3D preview, price override & queue action
        AdminLogin.jsx     Staff authentication
        AdminPrinters.jsx  Printer fleet live telemetry, status cards, and registration
        AdminSettings.jsx  Material price-per-gram, density, and setup fee configuration
        AdminStats.jsx     Analytics and order volume metrics
        CheckOrder.jsx     Student order search with live print progress bar
        StudentForm.jsx    Student upload & instant estimation form
      api.js              Centralized client API methods
      App.jsx             Navigation routing & admin authentication guard
      styles.css          Unified styling, telemetry meters, and responsive layout
    vite.config.js        Vite configuration with LAN host & API proxying
  .tools/                 Cloudflare Tunnel binary (cloudflared.exe, git-ignored)
```

---

## ⚡ Quick Start (Local Development)

### 1. Backend Setup

```bash
cd backend
npm install
npm start                 # Runs API on http://localhost:4000
```

> **Node.js 24 Note:** `better-sqlite3@^13.0.3` is used to ensure full compatibility with modern Node.js V8 cleanup hooks.

### 2. Frontend Setup

```bash
cd frontend
npm install
npm run dev               # Runs Vite on http://localhost:5173
```

Open [http://localhost:5173](http://localhost:5173). Default admin credentials:
- **Username:** `admin`
- **Password:** `admin123`
- **Dashboard:** [http://localhost:5173/admin](http://localhost:5173/admin)
- **Printer Fleet:** [http://localhost:5173/admin/printers](http://localhost:5173/admin/printers)

---

## 🌐 Accessing the App (Phone & Other Networks)

Depending on where you are connecting from, use the corresponding URL:

| Where you are connecting from | URL to use | How it works |
| :--- | :--- | :--- |
| **Different Network / 4G / 5G / Anywhere in the World** | 👉 **`https://subjective-map-dinner-wellness.trycloudflare.com`** | Secure Cloudflare HTTPS Tunnel. Works on any mobile network without being on the same Wi-Fi. |
| **Same Wi-Fi Network (Phone / Tablet on Home Wi-Fi)** | 👉 **`http://192.168.1.2:5173`** | Local LAN address via `server.host: true`. |
| **This Computer** | 👉 **`http://localhost:5173`** | Local loopback development address. |

### How to start the Cloudflare Tunnel anytime:
```powershell
& .tools/cloudflared.exe tunnel --url http://localhost:5173
```
*(This creates a free, secure public HTTPS `.trycloudflare.com` URL pointing directly to your Vite frontend and backend proxy).*

---

## 🖨️ Testing the Printer API Without Physical Hardware (Mock ESP32)

You do not need a physical 3D printer or ESP32 connected to test the system! A simulator script is included:

1. **Register a test printer:**
   - Log into the Admin panel at `http://localhost:5173/admin/printers`.
   - Click **+ Register New Printer**.
   - Enter a name (e.g. `Prusa Station 1`) and Device ID (e.g. `PRINTER-01`).
   - Copy the generated **API Key** (e.g. `esp_c31a1b...`).

2. **Start the Mock ESP32 client:**
   ```bash
   cd backend
   node scripts/mock_esp32.js --device-id PRINTER-01 --api-key <YOUR_API_KEY>
   ```

3. **Observe the flow:**
   - The printer will immediately appear as **ONLINE** on `/admin/printers` with live temperature readings (nozzle ~25°C, bed ~24°C).
   - Go to `/admin`, select an accepted order, and click **🚀 Queue for 3D Print**.
   - Within 5 seconds, the mock ESP32 claims the job, ramps up nozzle temperatures (~205°C), and streams printing progress from 0% to 100%.
   - In `/check`, the student can see their order live: `Printing — 50%`.
   - Once 100% is reached, the order status changes to `Ready` for pickup and the printer resets to `idle`.

4. **Simulate hardware failure (optional):**
   ```bash
   node scripts/mock_esp32.js --device-id PRINTER-01 --api-key <YOUR_API_KEY> --error
   ```
   At 50%, the mock client triggers an error (extruder jam), sets the printer to `error`, and marks the order as `Failed` with diagnostic notes for staff.

---

## 📡 ESP32 Hardware Integration & Outbound Communication

### Why Outbound (`ESP32 -> Cloud Backend`)?
- Most university Wi-Fi networks block incoming ports (NAT / strict firewall).
- The ESP32 initiates outbound HTTPS requests to the cloud server. No port forwarding or public IP on the printer side is needed.

### ESP32 API Endpoints (`/api/devices/*`):
All device requests require headers:
- `x-device-id`: Configured printer ID
- `x-api-key`: Secret printer key (validated via SHA-256 against `printers.api_key_hash`)

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/devices/heartbeat` | Sends current status, nozzle temp, and bed temp every 5–10s. |
| `GET` | `/api/devices/job/next` | Checks for and claims the oldest queued job in FIFO order. |
| `POST` | `/api/devices/job/:id/progress` | Reports print progress (0–100%) and temperatures. |
| `POST` | `/api/devices/job/:id/complete` | Marks print finished; transitions order to `Ready`. |
| `POST` | `/api/devices/job/:id/error` | Reports hardware errors (e.g. thermal runaway, jam). |
| `GET` | `/api/devices/job/:id/file` | Streams raw file data to the assigned ESP32. |

### What is needed on the actual ESP32 firmware:
1. **Wi-Fi Connection:** Connects to campus WPA2/Enterprise or IoT Wi-Fi.
2. **Serial / UART Bridge:** Connects to the 3D printer controller board (Marlin / RepRap / Klipper via UART TX/RX at 115200 or 250000 baud).
3. **Telemetry Polling:** Reads `M105` response (temperatures) and `M27` (print progress) from the printer motherboard.
4. **HTTPS Client:** Standard Arduino `HTTPClient` or ESP-IDF `esp_http_client` sending JSON payloads to the endpoints above.

---

## 🗄️ Database & Schema

The backend uses a dual-database architecture:
- **Local Dev:** SQLite via `better-sqlite3` (`backend/data/uni3dprint.db`).
- **Cloud Prod:** PostgreSQL via `pg` (enabled automatically when `DATABASE_URL` is set).

### Entities:
- **`users`:** Staff and student accounts (`id`, `username`, `email`, `password_hash`, `role`).
- **`printers`:** Physical devices (`id`, `name`, `device_id`, `api_key_hash`, `status`, `current_job_id`, `nozzle_temp`, `bed_temp`, `last_seen`).
- **`orders`:** Print requests (`id`, `student_id`, `file_reference`, `material_id`, `status`, `printer_id`, `estimated_price`, `final_price`).
- **`printer_jobs`:** Job queue entries (`id`, `order_id`, `printer_id`, `status`, `progress`, `error_message`, `started_at`, `completed_at`).
- **`materials`:** Filament types and pricing (`PLA`, `PETG`, `ABS`).
- **`settings`:** Global system settings (e.g. `setup_fee`).

---

## ☁️ Development vs. Production Setup

| Feature | Development (Local) | Production (Cloud) |
| :--- | :--- | :--- |
| **Backend** | `http://localhost:4000` | Cloud hosting (Railway, Render, Fly.io, VPS) |
| **Frontend** | `http://localhost:5173` (Vite dev server) | **GitHub Pages** (auto-deployed via GitHub Actions) |
| **Database** | SQLite (`DB_TYPE=sqlite`) | Managed PostgreSQL (`DATABASE_URL=postgres://...`) |
| **Storage** | Local disk (`uploads/orders/`) | S3-compatible Object Storage (AWS S3, Cloudflare R2) |
| **CORS** | Open to localhost / local network | Strict origin whitelist via `CORS_ORIGIN` |
| **Secrets** | Dev default fallback | High-entropy random `JWT_SECRET` (enforced at startup) |

### Required Backend Environment Variables (`backend/.env`):

```env
PORT=4000
NODE_ENV=production
JWT_SECRET=strong-cryptographically-random-64-char-string
CORS_ORIGIN=https://<your-github-username>.github.io
ADMIN_USERNAME=admin
ADMIN_PASSWORD=change_this_password

# Database (PostgreSQL in Cloud)
DB_TYPE=postgres
DATABASE_URL=postgresql://user:password@cloud-db.internal:5432/uni3dprint
DB_SSL=true

# Storage (S3 / R2 in Cloud)
STORAGE_PROVIDER=s3
S3_BUCKET=uni3dprint-models
S3_REGION=auto
S3_ENDPOINT=https://<account-id>.r2.cloudflarestorage.com
S3_ACCESS_KEY_ID=your_access_key
S3_SECRET_ACCESS_KEY=your_secret_key
```

---

## 🚀 GitHub Pages Deployment Guide

This project is configured for automatic frontend deployment to GitHub Pages.
The backend must be deployed separately on a server (Railway, Render, Fly.io, etc.).

### Architecture in Production

```
Browser
  |
  | HTTPS (e.g. https://yourname.github.io/uni3dprint/)
  v
GitHub Pages (serves static frontend bundle)
  |
  | HTTPS REST API calls (VITE_API_URL)
  v
Your Backend Server (Railway / Render / VPS / etc.)
  |
  +-- SQLite or PostgreSQL
  +-- Local disk or S3 for STL files
```

---

### Step 1 — Push the code to GitHub

```bash
# From the repo root (d:\uni3dprint or wherever it lives)
git init                          # if not already a git repo
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

Replace `<your-username>` and `<repo-name>` with your actual GitHub username and repository name.

---

### Step 2 — Deploy the backend

Choose one of these options:

#### Option A: Railway (recommended, free tier available)
1. Go to [railway.app](https://railway.app) and create a new project
2. Connect your GitHub repo
3. Set the **Root Directory** to `backend`
4. Set these environment variables in Railway:
   ```
   NODE_ENV=production
   JWT_SECRET=<generate with: openssl rand -hex 32>
   CORS_ORIGIN=https://<your-github-username>.github.io
   ADMIN_USERNAME=admin
   ADMIN_PASSWORD=<strong password>
   DB_TYPE=sqlite          # or postgres if you add a Railway PostgreSQL plugin
   ```
5. Railway will give you a URL like `https://uni3dprint-api-production.up.railway.app`
6. **Copy this URL** — you'll need it in Step 3

#### Option B: Render (free tier, spins down after inactivity)
1. Go to [render.com](https://render.com) and create a new **Web Service**
2. Connect your GitHub repo, set Root Directory to `backend`
3. Build command: `npm install` | Start command: `node server.js`
4. Add the same environment variables as in Option A

#### Option C: VPS / Self-hosted
```bash
cd backend
npm install --omit=dev
NODE_ENV=production JWT_SECRET=... CORS_ORIGIN=... node server.js
```
Use PM2 or systemd to keep the process running.

---

### Step 3 — Configure GitHub Actions Variables

In your GitHub repository:
1. Go to **Settings → Secrets and variables → Actions → Variables** tab
2. Click **New repository variable** and add:

   | Variable Name | Value | Example |
   |---|---|---|
   | `VITE_API_URL` | Full URL to your backend `/api` | `https://uni3dprint-api.up.railway.app/api` |
   | `VITE_BASE_PATH` | GitHub Pages sub-path (repo name with slashes) | `/uni3dprint/` |

   > **Note:** Use the **Variables** tab (not Secrets) — these values are baked into the
   > JavaScript bundle at build time and are visible in the browser, which is expected.
   > They are not sensitive credentials.

   > **Custom domain note:** If you configure a custom domain for GitHub Pages
   > (e.g. `https://3dprint.university.edu`), set `VITE_BASE_PATH=/` instead.

---

### Step 4 — Enable GitHub Pages

1. Go to your repository on GitHub
2. Navigate to **Settings → Pages**
3. Under **Source**, select **GitHub Actions**
4. Click **Save**

---

### Step 5 — Trigger the first deployment

```bash
# Make any small change (or just retrigger)
git commit --allow-empty -m "Trigger first GitHub Pages deploy"
git push
```

Or go to **Actions → Deploy Frontend to GitHub Pages → Run workflow** to trigger it manually.

---

### Step 6 — Verify the deployment

After the GitHub Action completes (~2 minutes), your site is live at:

```
https://<your-github-username>.github.io/<repo-name>/
```

**Checklist:**
- [ ] `https://<user>.github.io/<repo>/` loads the student order form
- [ ] `https://<user>.github.io/<repo>/check` opens directly (no 404)
- [ ] `https://<user>.github.io/<repo>/admin/login` opens directly (no 404)
- [ ] Submitting an STL file gets a price estimate (requires live backend)
- [ ] Admin login succeeds and orders list loads (requires live backend)
- [ ] STL 3D preview renders in the admin dashboard

---

### Step 7 — Set backend CORS correctly

Make sure your backend's `CORS_ORIGIN` environment variable includes your full GitHub Pages URL:

```env
# Single origin
CORS_ORIGIN=https://yourname.github.io

# Multiple origins (comma-separated)
CORS_ORIGIN=https://yourname.github.io,https://3dprint.university.edu
```

If CORS is misconfigured, the browser will block all API calls with a CORS error in the console.

---

### Frontend-only local build test

To test the production build locally before pushing:

```bash
cd frontend

# Set env vars (create a .env.local — it's gitignored)
echo "VITE_API_URL=https://your-backend.railway.app/api" > .env.local
echo "VITE_BASE_PATH=/uni3dprint/" >> .env.local

npm run build
npm run preview          # serves dist/ at http://localhost:4173/uni3dprint/
```

---

## 🛡️ Security Measures

- **Role-Based Access Control:** Strict separation between student and staff endpoints; students can only view their own order status.
- **Hardware Isolation:** ESP32 API keys are hashed with SHA-256; a printer can only query and report on its own assigned print jobs.
- **Brute-Force Protection:** Rate limiting configured via `express-rate-limit` for login, estimation, and device endpoints.
- **File Validation:** Enforced 50MB file size limits, extension checking, and mesh header integrity validation.

