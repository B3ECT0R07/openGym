# openGym — complete Windows 11 setup, from download to first workout

Written for: Vaibhav, Windows 11, PowerShell in VS Code, everything heavy on `D:`.

This is the whole path in one place: install the two prerequisites, land the repo in
`D:\WORK\Website work\openGym`, open it in VS Code, create one config file, start it, prove it's
healthy, make your profile, then keep it running and backed up. Follow the phases in order. Every
phase ends with a **Checkpoint** — if the checkpoint doesn't match, stop there and read the matching
entry in [Phase 13](#phase-13--troubleshooting) instead of continuing.

---

## The cheat sheet (what you'll actually type, all of it)

Once the prerequisites in Phases 1–2 are installed, the entire setup is these commands, run in the
VS Code terminal:

```powershell
Set-Location "D:\WORK\Website work\openGym"
Copy-Item .env.example .env
docker compose pull
docker compose up -d
docker compose ps
Invoke-RestMethod http://localhost:8080/api/health
Start-Process "http://localhost:8080"
```

Everything else in this document explains those seven lines, and what to do when one of them
misbehaves.

---

## Phase 0 — Understand what you're installing (2 minutes, saves an hour)

**What openGym is:** a self-hosted gym and body-weight tracker. It is *not* a Vercel site like your
portfolio, and it is *not* a Python project like AstraQuant or TradingAgents. It ships as two Docker
containers plus a data folder:

```
Your PC
└── Docker Desktop
    ├── container "web"   → nginx serving the React app, on http://localhost:8080
    └── container "api"   → Node server: passkeys, accounts, your workout data
            │
            └── D:\WORK\Website work\openGym\data   ← everything you log lives here
```

**Why Docker at all:** it packages Node and the app so you don't install Node, npm, Python, or any
dependency on Windows. You never run `npm install`. You never run `python`. If a guide ever tells you
to, it's for a different kind of project.

**Three facts that prevent 90% of first-time confusion:**

| Fact | Why it matters |
|---|---|
| The `data` folder **is** the app's state | Profiles, passkeys, workout history, uploaded exercise photos. Back it up and you can rebuild anything. |
| Passkeys only work on `http://localhost` or real HTTPS | On the PC that means `localhost`. Your phone on `192.168.x.x` needs a different path (Phase 12). |
| `localhost` and `127.0.0.1` are different to a passkey | The passkey is bound to the hostname `localhost`. On `127.0.0.1` the prompt silently won't appear. |

**Disk budget:** roughly 2–2.5 GB on `D:` for Docker Desktop plus the two openGym images, a few MB
for your data, plus whatever photos/videos you attach. About 1 GB stays on `C:` for the Windows
Subsystem for Linux base and Docker's service — that part is unavoidable and does not grow with your
usage once the data disk is on `D:`.

**Final folder layout you're aiming for:**

```
D:\WORK\Website work\openGym\
├── docker-compose.yml      ← the stack definition (this is the file the folder must contain)
├── .env                    ← YOU create this in Phase 5 (copy of .env.example)
├── .env.example            ← reference; leave as-is
├── api\                    ← API source (no need to touch)
├── frontend\               ← React source (no need to touch)
├── web\                    ← nginx image definition
├── catalogue\              ← exercise library + media, baked into the image
├── docs\                   ← the project's own documentation
├── data\                   ← CREATED on first run: your profiles, passkeys, history
└── coach-auth\             ← CREATED on first run: only used if you enable the AI coach
```

---

## Phase 1 — Install WSL2 (once per machine)

Docker Desktop on Windows 11 runs Linux containers inside WSL2. Install the plumbing first, with no
Linux distribution — Docker brings its own minimal VM.

1. Start → type `PowerShell` → right-click → **Run as administrator**.
2. Run:

```powershell
wsl --install --no-distribution
```

3. If Windows says a reboot is required, reboot, then continue.
4. Back in a **normal** (non-admin) PowerShell window:

```powershell
wsl --update
wsl --status
```

**Checkpoint:** `wsl --status` reports `Default Version: 2` and `WSL version: 2.x`. It should *not*
say "WSL is not installed".

Optional, and recommended if you have 16 GB of RAM or less — stop Docker's VM from claiming half of
it. This config file has to live on `C:`, it's about 100 bytes:

```powershell
@"
[wsl2]
memory=4GB
processors=4
swap=2GB
"@ | Set-Content -Path "$env:USERPROFILE\.wslconfig"
wsl --shutdown
```

Raise `memory` to `6GB` or `8GB` if you have 32 GB. This only takes effect after `wsl --shutdown`
(and restarting Docker Desktop).

---

## Phase 2 — Install Docker Desktop onto `D:`

### 2.1 Download the installer to `D:`

```powershell
New-Item -ItemType Directory -Force "D:\Apps\Downloads" | Out-Null
Invoke-WebRequest -Uri "https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe" -OutFile "D:\Apps\Downloads\DockerDesktopInstaller.exe"
```

### 2.2 Install with both locations on `D:` (PowerShell as Administrator)

```powershell
Set-Location "D:\Apps\Downloads"
Start-Process -Wait -FilePath ".\DockerDesktopInstaller.exe" -ArgumentList 'install', '--accept-license', '--backend=wsl-2', '--installation-dir=D:\Apps\Docker\Desktop', '--wsl-default-data-root=D:\Apps\Docker\WSL'
```

Why the two flags, in plain terms:

- `--installation-dir` puts the Docker Desktop program on `D:` instead of `C:\Program Files\Docker`.
- `--wsl-default-data-root` puts the **virtual disk that holds every image and container layer** on
  `D:`. This is the flag that actually keeps `C:` clean; the program itself is the small part.

> Keep those two paths as **siblings**. Do not point `--wsl-default-data-root` at something inside
> `--installation-dir` — nesting them has been reported to lose the data disk on a Docker Desktop
> update.

If your Windows administrator account is different from `becto`, add yourself to the Docker group
(also Administrator):

```powershell
net localgroup docker-users becto /add
```

### 2.3 First launch

1. Start → **Docker Desktop**.
2. Accept the service agreement.
3. When asked, choose **Use WSL 2 instead of Hyper-V** (this is the default).
4. Skip the sign-in / account screen — not needed for local use.
5. In Settings → General, un-tick **Start Docker Desktop when you sign in** if you'd rather start it
   manually. It uses 1–2 GB of RAM while running.
6. Windows Firewall may prompt: allow it on private networks.

Wait until the whale icon in the system tray stops animating (30–60 s on first start).

### 2.4 Point the image store at `D:`

Docker Desktop → **Settings → Resources → Advanced → Disk image location**. It must read
`D:\Apps\Docker\WSL`. If it reads something under `C:`, change it there and let Docker restart — that
setting is the runtime equivalent of the installer flag, and it's what to use if the flag didn't take.

### 2.5 Verify (open a **new** PowerShell window)

```powershell
docker version
docker compose version
docker info --format '{{.OSType}}'
```

**Checkpoint:**
- `docker version` prints a `Client` and a `Server` block. If the Server block errors, Docker Desktop
  isn't running yet.
- `docker compose version` prints `Docker Compose version v2.x`. Note the **space** in `docker
  compose`; the old hyphenated `docker-compose` is not what you use here.
- `docker info --format '{{.OSType}}'` prints `linux`.

---

## Phase 3 — Get the repo into `D:\WORK\Website work\openGym`

Pick **one** of the two options. Option A is better if you have Git (you do) because updating later is
one command; Option B avoids Git entirely.

### Option A — Git clone (recommended)

```powershell
Set-Location "D:\WORK\Website work"
git clone https://github.com/B3ECT0R07/openGym.git openGym
Set-Location "D:\WORK\Website work\openGym"
```

That's your fork. If you'd rather track the original project, clone
`https://github.com/DuarteSantos8/openGym` instead — the app is identical.

Expected output ends with something like `Resolving deltas: 100% ... done.` The `catalogue\` folder is
large (~250 MB), so give it a minute on a slow connection.

### Option B — Download the ZIP (no Git)

1. Open <https://github.com/B3ECT0R07/openGym> in a browser.
2. Green **Code** button → **Download ZIP**.
3. Save it to `D:\Apps\Downloads`, then extract:

```powershell
Set-Location "D:\Apps\Downloads"
Expand-Archive -Path ".\openGym-main.zip" -DestinationPath "D:\WORK\Website work" -Force
```

4. GitHub's ZIP extracts into a folder named after the branch (`openGym-main`). Rename it so your
   paths match every command below:

```powershell
Rename-Item -Path "D:\WORK\Website work\openGym-main" -NewName "openGym"
```

If the rename says the name already exists, you already have a folder there — inspect it before
going further rather than merging two copies.

### Checkpoint for both options — the folder must look like this

```powershell
Set-Location "D:\WORK\Website work\openGym"
Get-ChildItem -Force | Select-Object Name, Mode
```

You must see `docker-compose.yml` **directly** in this folder, alongside `api`, `frontend`, `web`,
`catalogue`, `docs`, `.env.example`, `README.md`.

> **The classic ZIP mistake:** extracting produces
> `D:\WORK\Website work\openGym\openGym-main\docker-compose.yml` — a folder inside a folder. Then
> `docker compose` says *"no configuration file provided"*. Fix the nesting with the `Rename-Item` /
> move step above so `docker-compose.yml` sits at the top of `openGym`. Check with:
> `Test-Path "D:\WORK\Website work\openGym\docker-compose.yml"` — it must print `True`.

Also confirm the clone isn't sitting in a OneDrive-synced folder. `D:\WORK\...` is not synced, which
is what you want (syncing a live data folder causes file-lock oddities).

---

## Phase 4 — Open the folder in VS Code and get the right terminal

### 4.1 Open the folder (not the file)

In VS Code: **File → Open Folder…** → navigate to `D:\WORK\Website work\openGym` → **Select Folder**.

Open the **folder**, not `docker-compose.yml` on its own. VS Code's terminal and relative paths all
key off the folder you opened.

If VS Code asks *"Do you trust the authors of the files in this folder?"* → **Yes, I trust the
authors** (it's your own clone).

### 4.2 Confirm you're looking at the right thing

The Explorer pane (left, `Ctrl+Shift+E`) should list `api`, `frontend`, `web`, `catalogue`, `docs`,
`docker-compose.yml`, `README.md`, `.env.example`. Dot-files like `.env.example` are visible because
VS Code shows them by default — if you don't see it, you've opened a subfolder.

### 4.3 Open the integrated terminal

`Ctrl+`` ` `` (backtick), or **Terminal → New Terminal**.

Then make sure of three things:

1. The terminal dropdown (top-right of the terminal panel) says **PowerShell**. If it says
   `Command Prompt` or `Git Bash`, click the dropdown → **Select Default Profile** → **Windows
   PowerShell**, then open a new terminal with `Ctrl+Shift+`` ` ``.
2. The prompt starts in `D:\WORK\Website work\openGym`. If it starts somewhere else (e.g.
   `C:\Users\becto`), run the `Set-Location` command below.
3. No red `(venv)` / `(base)` prefix. If you see one, the terminal inherited an activated Python env
   from a previous project — type `deactivate` first. It doesn't break Docker, but it makes paths
   confusing.

Confirm the working directory:

```powershell
Get-Location
```

**Checkpoint:** it prints `D:\WORK\Website work\openGym`. If not:

```powershell
Set-Location "D:\WORK\Website work\openGym"
```

> Rule for the rest of this guide: **every command runs from that folder.** If you open a second
> terminal tab later, it also starts there. If you ever see an error about a missing
> `docker-compose.yml`, your current directory is wrong — run `Get-Location` and fix it.

### 4.4 Things you do *not* need in VS Code

- No Python extension, no interpreter, no virtualenv. openGym doesn't run Python.
- No Node.js install. The React build happens inside the container.
- Never open `frontend/index.html` with Live Server or double-click it. It is a source template
  compiled at build time; opening it directly gives you a blank page and means nothing is wrong.

The only VS Code features you'll use here: the Explorer (to create `.env`), the integrated
PowerShell terminal, and maybe the text editor to tweak `.env`.

---

## Phase 5 — Create `.env` (required, and the usual reason a first run fails)

`docker-compose.yml` tells Compose to load a file called `.env`. If that file doesn't exist, the stack
refuses to start. You create it once by copying the example.

### 5.1 Copy it

In the VS Code terminal:

```powershell
Set-Location "D:\WORK\Website work\openGym"
Copy-Item .env.example .env
```

Then verify it exists and is a single file named exactly `.env`:

```powershell
Get-ChildItem -Force .env* | Select-Object Name, Length
```

**Checkpoint:** you see both `.env` and `.env.example`, with `.env` roughly 14 KB (it copies all the
explanatory comments, which is fine and useful).

> If Windows Explorer "hides known file extensions", a file you create by right-clicking may end up
> named `.env.txt` and Compose will ignore it. The `Copy-Item` above avoids that entirely — prefer it
> over hand-creating the file.

### 5.2 Look at it (optional but useful)

```powershell
code .env
```

The top of the file is the part that matters:

```
RP_ID=localhost
ORIGIN=http://localhost:8080
WEB_PORT=8080
RP_NAME=openGym
```

Leave those four exactly as they are for a local install. What they mean:

| Setting | Meaning | Leave as |
|---|---|---|
| `RP_ID` | The hostname your passkey is bound to. Must match the browser address bar. | `localhost` |
| `ORIGIN` | Full URL the app is served from. Must match exactly, port included. | `http://localhost:8080` |
| `WEB_PORT` | The port on your PC that serves the UI. | `8080` |
| `RP_NAME` | The name shown inside the Windows Hello passkey prompt. | `openGym` |

Everything below those four lines is commented-out documentation for optional features. The file is
safe to leave as-is.

### 5.3 Optional additions for your setup

Add these as new plain lines (no `#`) near the top of `.env` if you want them now:

```
# The first profile created on this instance becomes its admin
FIRST_USER_ADMIN=1

# Also allow name + password sign-in next to passkeys
# (this is what makes the app usable from your phone over plain home Wi-Fi)
PASSWORD_LOGIN=1
```

- `FIRST_USER_ADMIN=1` gives you an Admin dashboard link in Settings — user list, invite codes,
  disable accounts, activity log. Turn it on *before* you create your profile so you're the one who
  gets it.
- `PASSWORD_LOGIN=1` adds a password option to the login screen. Passkeys stay the primary method;
  this just gives you a fallback and a way in from the LAN (Phase 12).
- `ALLOW_GUEST=0` would remove the *Continue without account* button if you want the server to be the
  only door. Optional.

Save the file (`Ctrl+S`). No restart yet — the values are read when the containers start.

---

## Phase 6 — First start

Make sure Docker Desktop is running (whale icon in the tray, not animating). Then, from
`D:\WORK\Website work\openGym`:

### 6.1 Pull the prebuilt images

```powershell
docker compose pull
```

This downloads two images from `ghcr.io`: the API (small, tens of MB) and the web image (larger, it
carries the ~80 MB exercise animation library). Expect **1–5 minutes** depending on your connection.
Layers print as `Pulling`, `Downloading`, `Extracting`, `Pull complete`.

**Checkpoint:** the last lines say `Pulled` for both `opengym-api` and `opengym-web`. If you get
`unauthorized` or `denied`, you have a corporate proxy or VPN intercepting `ghcr.io` — disconnect it
and retry, or build from source instead (6.1b).

**6.1b — Alternative: build from source** (no images downloaded; needs ~5–10 minutes and more disk):

```powershell
docker compose up -d --build
```

Everything else below is identical.

### 6.2 Start it

```powershell
docker compose up -d
```

Expected output:

```
[+] Running 3/3
 ✔ Network opengym_default  Created
 ✔ Container opengym-api-1  Started
 ✔ Container opengym-web-1  Started
```

What just happened on disk: `.\data` and `.\coach-auth` were created, along with the session secret
and the push-notification keys inside them.

### 6.3 Checkpoint — status

```powershell
docker compose ps
```

You want both services listed. `api` should reach `(healthy)` within ~20 seconds; `web` shows `Up`.
`State` should be `running`, not `restarting` or `exited`.

If something is `restarting`, read the logs before doing anything else:

```powershell
docker compose logs --tail 50 api
docker compose logs --tail 50 web
```

---

## Phase 7 — Prove it's actually working

Three checks, in increasing strength:

**1. The API answers directly**

```powershell
Invoke-RestMethod http://localhost:8080/api/health
```

**Checkpoint:** exactly this shape:

```
ok       users writable
--       ----- --------
True         0     True
```

`users : 0` means a fresh instance with no profiles yet. `writable : True` means the `data` folder on
`D:` is being written correctly — this single check covers the most common misconfiguration
(a permissions or path problem on the data mount).

**2. The API loaded the config you think it did**

```powershell
docker compose logs api | Select-String "gym-api on"
```

**Checkpoint:** one line like

```
gym-api on :3000 (rpID=localhost, origin=http://localhost:8080)
```

Both values must match Phase 5.2. This is the line to quote whenever a passkey question comes up.

**3. Open the app**

```powershell
Start-Process "http://localhost:8080"
```

**Checkpoint:** the openGym sign-in screen: dark background, logo, **Create profile** and
**Continue without account**. The browser tab shows a padlock / "Not secure" label — normal and
harmless for `localhost`, and passkeys still work.

The first page load is fast; there's no media download step because the exercise animations are
inside the image.

---

## Phase 8 — Make your profile and set up your week

### 8.1 Create the profile + passkey

1. Click **Create profile**.
2. Enter a name (this is what shows on the Today screen).
3. Windows shows a **Windows Hello** dialog — "Use your PIN / fingerprint / face for openGym".
   Confirm it.
   - No PIN set up? Windows walks you through creating one. That PIN is also what unlocks your
     passkey later.
   - Prompt never appears? You're almost certainly on `127.0.0.1` instead of `localhost`. Copy the
     exact URL from Phase 7.3.
4. You land on the Today screen, signed in.

Because `FIRST_USER_ADMIN=1` was set before this step, this profile is now the instance admin:
**Settings → Account → Admin dashboard**.

### 8.2 Pick or build a routine

1. **Settings → Routines** (or the setup prompt on Today).
2. Four starter plans are there as normal, editable routines: Push/Pull/Legs, Upper/Lower, Full Body,
   5×5. Pick one to have a working week in ten seconds, then edit it — nothing about them is special
   or locked.
3. Set the week start (Monday or Sunday) in Settings, and per-weekday routine assignments.
4. Prefer not to be tied to weekdays? Switch to **rotation** mode (A, B, C, A, …): the app gives you
   the next session you haven't done, whenever you show up.

### 8.3 First workout

- **Today** starts the planned session; weights pre-fill from the last time you did each exercise.
- Tap the set number to log a set; the rest timer starts automatically.
- Swipe a set left to delete (with Undo), right to copy it.
- Log body weight from Today; the chart and goal line are under Stats.

### 8.4 Know where your data is

```
D:\WORK\Website work\openGym\data
```

That folder contains all profiles, the passkeys, every workout, and `data\uploads\` for any photos or
videos you attach to custom exercises. Confirm it's populated:

```powershell
Get-ChildItem "D:\WORK\Website work\openGym\data" | Select-Object Name, Length
```

You'll see `db.json`, a `secret` file, `vapid.json`, and possibly `audit.log`.

There's also a per-user export at **Settings → Data & backup → Export**: one JSON with everything, plus
an *Export with photos & videos* variant. Worth doing once now so you know where the button is.

---

## Phase 9 — Optional configuration you may want later

All of it is `.env`, followed by `docker compose up -d` to apply (Compose recreates the affected
containers automatically — no `down` needed, and `data` is untouched).

| Goal | `.env` change | Notes |
|---|---|---|
| Different port (8080 busy) | `WEB_PORT=8090` **and** `ORIGIN=http://localhost:8090` | Both must change together, or passkeys break |
| Your own domain + HTTPS | `RP_ID=gym.yourdomain.com`, `ORIGIN=https://gym.yourdomain.com` | Changing `RP_ID` **invalidates existing passkeys** — do it before you have history you care about |
| Invite-only instance | `INVITE_ONLY=1` | Create your own profile first, then generate codes from the admin dashboard |
| No guest mode | `ALLOW_GUEST=0` | Removes *Continue without account* |
| Longer sign-in | `SESSION_DAYS=180` | Default 90 |
| Smaller activity log | `AUDIT_MAX=1000`, `AUDIT_DAYS=30` | Defaults 5000 / 90 |
| Bigger custom-exercise videos | `MEDIA_VIDEO_MAX_MB=80` and raise `MEDIA_UPLOAD_MAX=96m` | nginx limit must stay above the API limit |
| AI coach (optional, off by default) | see `docs/AI_COACH.md` | Needs a provider key; the default image has no AI runtime — build with `API_TARGET=coach` |

Apply and verify:

```powershell
docker compose up -d
docker compose logs api | Select-String "gym-api on"
```

---

## Phase 10 — Day-to-day command card

Everything runs from `D:\WORK\Website work\openGym` (VS Code terminal):

```powershell
Set-Location "D:\WORK\Website work\openGym"

docker compose ps                 # are both containers up?
docker compose up -d              # start (after a reboot, after a .env change)
docker compose logs -f            # live logs; Ctrl+C exits, containers keep running
docker compose restart            # bounce both without touching data
docker compose stop               # stop containers, keep them
docker compose down               # remove containers and network; KEEPS .\data
```

Two habits worth forming:

- **After a reboot:** open Docker Desktop, wait for the whale to settle, then `docker compose up -d`.
  Enable *Start Docker Desktop when you sign in* in Docker settings if you want it automatic.
- **Bookmark `http://localhost:8080`** in the browser you use daily, and use *Add to Home Screen* on
  Edge/Chrome to get it as an app window.

---

## Phase 11 — Backups, restore, updates

### Back up (once a month is plenty)

```powershell
Set-Location "D:\WORK\Website work\openGym"
New-Item -ItemType Directory -Force "D:\Backups\openGym" | Out-Null
tar -czf "D:\Backups\openGym\opengym-data-$(Get-Date -Format yyyy-MM-dd).tar.gz" data
```

`tar` is built into Windows 11, no extra install. The archive holds every profile, passkey and workout,
plus `audit.log` if the activity log is on — treat it as sensitive and don't drop it in a shared cloud
folder.

Skip uploaded media (usually the bulk of the size):

```powershell
tar -czf "D:\Backups\openGym\opengym-data-$(Get-Date -Format yyyy-MM-dd).tar.gz" --exclude=data/uploads data
```

### Restore

```powershell
Set-Location "D:\WORK\Website work\openGym"
docker compose down
Rename-Item ".\data" ".\data-old"     # keep the old one until you've confirmed the restore
tar -xzf "D:\Backups\openGym\opengym-data-2026-10-25.tar.gz"
docker compose up -d
Invoke-RestMethod http://localhost:8080/api/health
```

### Update

```powershell
Set-Location "D:\WORK\Website work\openGym"
git pull
docker compose pull
docker compose up -d
```

If you built from source instead: `git pull` then `docker compose up -d --build`. Your `./data` is
never touched by an update.

### Move it to another PC later

Copy the whole `openGym` folder (or just `data\`) to the same path on the new machine and run
`docker compose up -d`. Passkeys are bound to the hostname, so `localhost` keeps working on any
machine; a LAN IP or domain would need re-registering.

---

## Phase 12 — Using it from your phone

Browsers only allow passkeys over **HTTPS**, with exactly one exception: `http://localhost`. So
`http://192.168.1.50:8080` on your phone will load the app but will **not** offer a passkey. Three real
options:

**Option 1 — Guest mode over the LAN (zero setup).** Phone → `http://<PC-LAN-IP>:8080` → *Continue
without account*. Data lives only in that phone's browser cache. Fine for trying it; don't track real
training in it.

**Option 2 — Password sign-in over the LAN (recommended for home).** With `PASSWORD_LOGIN=1` in
`.env`: on the PC, Settings → Account → set a password. Then on the phone use the same server address
and the password form. Same account, same synced history, no HTTPS needed. Works on home Wi-Fi only.

Find your PC's LAN address:

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
  Where-Object { $_.IPAddress -like "192.168.*" -or $_.IPAddress -like "10.*" } |
  Select-Object IPAddress, InterfaceAlias
```

Then allow the app through the firewall for private networks if the phone can't connect:
Windows Security → Firewall → *Allow an app through firewall* → Docker Desktop → tick **Private**.
Also make sure the network profile is **Private** (Settings → Network → your Wi-Fi → network profile).

**Option 3 — Real HTTPS on a domain (passkeys on the phone, works anywhere).** Put a Cloudflare
Tunnel or Caddy in front of the `web` container, then set `RP_ID=gym.yourdomain.com` and
`ORIGIN=https://gym.yourdomain.com` in `.env` and `docker compose up -d`. Step-by-step for Cloudflare
Tunnel, Caddy, Traefik and nginx is in the project's own `docs/SELF_HOSTING.md` §3; LAN-only HTTPS with
your own certificate is in `docs/SELF_HOSTING_HTTPS.md`.

A note on the mobile apps: the official Android APK and iOS app are a *separate* build that keeps
everything on the phone with no server. They do not read this instance's data. The Android app can
pair with your server from Settings → Account → *Pair the mobile app* (mint a one-time code on the
PC), which gives you a bearer token instead of a passkey.

---

## Phase 13 — Troubleshooting

Work top to bottom; the first two cover most first-run failures.

| Symptom | Cause | Fix |
|---|---|---|
| `docker: The term 'docker' is not recognized...` | Docker Desktop not running, or terminal opened before install | Start Docker Desktop, wait for the tray icon, open a **new** VS Code terminal |
| `error during connect: ... dockerDesktopLinuxEngine ... The system cannot find the file specified` | Docker engine not started | Same as above — the engine starts with Docker Desktop |
| `unable to get image ... failed to solve: ...` / `no configuration file provided: not found` | Wrong working directory, or nested `openGym\openGym-main` from a ZIP | `Get-Location`, then `Set-Location "D:\WORK\Website work\openGym"`; check `Test-Path .\docker-compose.yml` prints `True` |
| `env file ... .env not found` | `.env` missing (or named `.env.txt`) | `Copy-Item .env.example .env` in that folder |
| `port is already allocated` / `Bind for 0.0.0.0:8080 failed` | Something else owns 8080 | `Get-NetTCPConnection -LocalPort 8080 -State Listen \| Select-Object OwningProcess` then `Get-Process -Id <pid>`; free it, or set `WEB_PORT=8090` + `ORIGIN=http://localhost:8090` |
| App loads, **no passkey prompt** | Opened `127.0.0.1` or the LAN IP instead of `localhost` | Use exactly `http://localhost:8080`. Verify with `docker compose logs api \| Select-String "gym-api on"` |
| Windows Hello says "This option isn't currently available" | No PIN configured, or the account is a local account without Hello | Windows Settings → Accounts → Sign-in options → set up a PIN, retry |
| `api` container `restarting` in a loop | Bad `.env` value (stray quote, trailing space, wrong port) | `docker compose logs --tail 50 api`; fix `.env`; `docker compose up -d` |
| `/api/health` returns `writable: false` or 503 | `data` folder not writable | `docker compose down`, confirm `D:\WORK\Website work\openGym\data` exists, `docker compose up -d`; check antivirus isn't locking it |
| Blank page / stuck on loading | You opened `frontend/index.html` directly | Open the container URL instead: `http://localhost:8080` |
| `WSL 2 is not installed` / `wsl --update` fails | WSL missing or outdated | `wsl --install --no-distribution` as admin, reboot, `wsl --update` |
| Docker Desktop eats `C:` space | Data root still on `C:` | Settings → Resources → *Disk image location* → `D:\Apps\Docker\WSL` |
| Everything broke after a Windows update | Docker engine stopped | Start Docker Desktop, `docker compose up -d`. Data is on disk; nothing lost |
| Very slow first page load or slow file writes | Antivirus scanning the bind mount | Exclude `D:\WORK\Website work\openGym\data` and Docker's install dir from real-time scanning |

### Nuclear reset (deletes all profiles and history)

```powershell
Set-Location "D:\WORK\Website work\openGym"
tar -czf "D:\Backups\openGym\pre-reset-$(Get-Date -Format yyyy-MM-dd-HHmm).tar.gz" data
docker compose down
Remove-Item -Recurse -Force .\data
docker compose up -d
```

Only `.\data` removal destroys your history. `docker compose down` on its own is always safe.

---

## Appendix A — Which files matter, and which to ignore

| Path | What it is | Do you touch it? |
|---|---|---|
| `docker-compose.yml` | The two containers, ports, mounts | Only if you know why |
| `.env` | Your configuration | Yes — this is your file |
| `data\` | All your data | Back it up; never hand-edit unless you're sure |
| `api\server.js` | The API: auth, sync, all endpoints | Only if hacking on the code |
| `frontend\src\` | React app | Only if hacking on the code |
| `web\Dockerfile`, `web\nginx.conf.template` | How the web image is built and proxied | No |
| `catalogue\` | 5,600+ exercises + media, baked into the image | No |
| `docs\` | The project's own docs (`SELF_HOSTING.md` is the deep reference) | Read when you go beyond localhost |
| `helm\`, `kubernetes\`, `mcp\`, `website\` | Other deployment targets / optional extras | No — not used by this setup |

## Appendix B — Optional: VS Code tasks for the commands you'll repeat

Not required. If you'd like them in the Command Palette (`Ctrl+Shift+P` → *Tasks: Run Task*), create
`.vscode\tasks.json` in the repo folder:

```json
{
  "version": "2.0.0",
  "tasks": [
    { "label": "openGym: up",     "type": "shell", "command": "docker compose up -d",  "problemMatcher": [] },
    { "label": "openGym: down",   "type": "shell", "command": "docker compose down",   "problemMatcher": [] },
    { "label": "openGym: logs",   "type": "shell", "command": "docker compose logs -f","problemMatcher": [] },
    { "label": "openGym: status", "type": "shell", "command": "docker compose ps",     "problemMatcher": [] },
    { "label": "openGym: health", "type": "shell", "command": "Invoke-RestMethod http://localhost:8080/api/health", "problemMatcher": [] },
    {
      "label": "openGym: backup",
      "type": "shell",
      "command": "New-Item -ItemType Directory -Force \"D:\\Backups\\openGym\" | Out-Null; tar -czf \"D:\\Backups\\openGym\\opengym-data-$(Get-Date -Format yyyy-MM-dd).tar.gz\" data",
      "problemMatcher": []
    }
  ]
}
```

Add `.vscode/` to `.gitignore` in your fork if you don't want to commit it.

## Appendix C — Running from source without Docker (only if you're changing the code)

Needs **Node.js 22** installed on `D:` (e.g. `D:\Apps\nodejs`, with `D:\Apps\nodejs` on `PATH`). Two
VS Code terminals:

Terminal 1 — the API:

```powershell
Set-Location "D:\WORK\Website work\openGym\api"
npm ci --omit=optional
$env:PORT="3000"
$env:DATA_DIR="D:\WORK\Website work\openGym\data"
node server.js
```

Terminal 2 — the frontend dev server (proxies `/api` to the API above):

```powershell
Set-Location "D:\WORK\Website work\openGym\frontend"
npm ci --ignore-scripts
npm run dev
```

Open the Vite URL it prints (usually `http://localhost:5173`). `localhost` on any port is still a valid
WebAuthn context, so passkeys work. `node_modules` lands inside the project on `D:`, never on `C:`.
For actually using the app, Docker is simpler and is what this guide assumes.

## Appendix D — What "done" looks like

- `docker compose ps` lists `opengym-api-1` (healthy) and `opengym-web-1` (up).
- `http://localhost:8080/api/health` returns `ok: True`, `users: 1`, `writable: True`.
- The app opens at `http://localhost:8080`, signed in with Windows Hello.
- `D:\WORK\Website work\openGym\data` holds `db.json` and your history.
- One backup exists in `D:\Backups\openGym`.

## Appendix E — Handover-document addition

Append under *Section 16 — Future Updates* of your handover document:

```
### Update — openGym Self-Hosted Setup (Windows 11)
- Local path: D:\WORK\Website work\openGym
- Repository: https://github.com/B3ECT0R07/openGym.git (fork of DuarteSantos8/openGym)
- Type: self-hosted Docker stack (containers: api + web) — NOT Vercel, NOT Python
- Workflow: download/clone into D:\WORK\Website work\openGym, open THAT FOLDER in VS Code,
  run everything from the integrated PowerShell terminal at that path
- Required file: .env (Copy-Item .env.example .env) — the stack refuses to start without it
- Docker Desktop install: D:\Apps\Docker\Desktop
- Docker WSL data disk: D:\Apps\Docker\WSL (sibling of install dir, never nested inside it)
- Runtime data: D:\WORK\Website work\openGym\data (profiles, passkeys, history, uploads)
- Backups: D:\Backups\openGym (tar -czf of data/)
- URL on the PC: http://localhost:8080 — always "localhost", never 127.0.0.1 (passkey RP_ID)
- Standard commands (from the project folder):
    docker compose up -d
    docker compose ps
    docker compose logs -f
    docker compose down
- Update: git pull ; docker compose pull ; docker compose up -d
- Verify: Invoke-RestMethod http://localhost:8080/api/health  ->  ok users writable
- Phone: passkeys need HTTPS; over LAN use PASSWORD_LOGIN=1 or guest mode
- Not needed on host: Node, npm, Python, venv. Build happens inside Docker.
```
