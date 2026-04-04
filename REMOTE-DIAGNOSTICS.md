# REMOTE-DIAGNOSTICS.md
*Generated: 2026-04-04 | Remote session via Claude Code*

---

## What was broken and why

### Primary failure: BXT Intelligent Assistant offline
**Root cause:** The chatbot backend used Cloudflare **Quick Tunnels** (`*.trycloudflare.com`). Quick Tunnels are ephemeral — they generate a new random subdomain every time `cloudflared` starts, and they die when the process stops. Six distinct tunnel URLs were found across git history, all now dead:

| Tunnel URL | Status |
|---|---|
| `msgid-curve-disposal-ethernet.trycloudflare.com` | Dead (DNS NXDOMAIN) |
| `thrown-longitude-have-farming.trycloudflare.com` | Dead |
| `whatever-handmade-important-tall.trycloudflare.com` | Dead |
| `protected-article-fought-java.trycloudflare.com` | Dead |
| `nuts-jackson-jessica-drum.trycloudflare.com` | Dead |
| `stem-nirvana-corporations-flexibility.trycloudflare.com` | Dead |
| `merge-televisions-genetics-grew.trycloudflare.com` | Started 2026-04-04, now dead |

Every time the workstation restarted `cloudflared`, the tunnel URL changed, and the URL hardcoded in the JS was stale.

### Secondary issue: Original persona was overwritten
During earlier fixes (well-intentioned), the chatbot greeting was changed from:
> *"Die physikalische Wahrheit erkennen. Engineering – begrenzt nur durch Naturgesetze."*

…to a generic:
> *"BXT Assistent — Fragen zu unseren Leistungen?"*

The KB response responses were also generic and led with "KI" (artificial intelligence) — a liability framing for serious R&D engineering audiences.

### What `bayernxtech.de` is (and isn't)
`bayernxtech.de` resolves to Cloudflare anycast IPs (21.0.0.107). It's a separate marketing/lander site with a WAF that blocks POST requests. It is **not** the FastAPI backend. The FastAPI backend only lives behind the cloudflared tunnel.

### OpenClaw status
OpenClaw is the messaging gateway that bridges Telegram/WhatsApp to the local LLM. It runs on the workstation at:
- Socket: `ws://127.0.0.1:18789`
- Config: `C:\Users\Dylan Kane\.openclaw\`
- It cannot be restarted remotely — requires local workstation access or SSH.

---

## What was fixed remotely

### 1. Chatbot is functional again (KB fallback mode)
- `BACKEND_URL` cleared of the dead tunnel URL → no more 502 errors shown to users
- Chatbot now runs in **KB mode** — instant responses, no network dependency, always online
- Chat header shows "Online" (accurate — KB is always available)

### 2. Original persona restored
- Greeting restored: *"Die physikalische Wahrheit erkennen. Engineering – begrenzt nur durch Naturgesetze. Welche Herausforderung können wir für Sie lösen?"*
- EN greeting: *"Discover ground truth. Engineering bounded only by physics."*
- Full KB rewritten with BXT engineering voice:
  - No "KI" as lead word
  - Physics-first identity throughout
  - Direct, opinionated, technical depth
  - Causality vs. statistics framing
  - "Ground Truth erkennen & physikbasierte Intelligenz bauen"

### 3. Cloudflare Worker backend prepared
- `worker/chat-worker.js` — complete Worker with BXT system prompt, Llama 3.3 70B, session history, full CORS
- `wrangler.toml` — deployment config
- `.github/workflows/deploy-worker.yml` — auto-deploys on push when secrets are set
- **To activate**: Add `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` as GitHub repo secrets → trigger workflow manually

### 4. Self-healing health check workflow
- `.github/workflows/health-check.yml` runs every 6 hours
- Checks: site HTTP 200, widget JS present, KB engine present, original greeting present, backend health
- On failure: opens a GitHub issue with full diagnostics and workstation recovery commands

### 5. Hybrid architecture in place
When `BACKEND_URL` is set to a live tunnel:
- Health check runs on load, on chat open, every 20s
- If backend responds → routes to GPU/Qwen, shows green "xPathfinder Labs" status
- If backend unreachable → silently falls through to KB, no error shown to user

---

## What still needs physical workstation access

### Priority 1 — Get xPathfinder Labs backend back online
The FastAPI server (Qwen on local GPU) is almost certainly still running. The problem is the tunnel.

**Permanent fix (do this once, never repeat the tunnel URL dance):**
```bash
# On workstation WSL2 terminal (dgkallday88@DESKTOP-B58J3QQ)
cloudflared tunnel login
cloudflared tunnel create xpathfinder-labs
cloudflared tunnel route dns xpathfinder-labs chat.bayernxtech.de

# Create ~/.cloudflared/config.yml:
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: YOUR-TUNNEL-ID
credentials-file: /home/dgkallday88/.cloudflared/YOUR-TUNNEL-ID.json
ingress:
  - hostname: chat.bayernxtech.de
    service: http://localhost:8000
  - service: http_status:404
EOF

# Install as system service (survives reboots):
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Then update one line in `index.html`:
```js
const BACKEND_URL = 'https://chat.bayernxtech.de/api/chat/';
```
Push → done. The URL is now permanent. Never changes again.

### Priority 2 — FastAPI server as service
The FastAPI server should also run as a systemd service so it survives reboots:
```bash
# Create /etc/systemd/system/bxt-api.service
sudo systemctl enable bxt-api
sudo systemctl start bxt-api
```

### Priority 3 — OpenClaw gateway
```bash
# In C:\Users\Dylan Kane\.openclaw\ or WSL2:
openclaw gateway start

# Telegram (restart):
openclaw channels restart telegram

# WhatsApp (if 401 auth error — needs phone to scan QR):
openclaw channels link whatsapp
```

### Priority 4 — Spectre XT compute node
Needs physical access if the worker process stopped. Check `C:\Users\Dylan Kane\.openclaw\` for logs.

---

## Recommended architecture changes

### Stop using Quick Tunnels for production
Quick Tunnels (`cloudflared tunnel --url`) are for local development demos. They are explicitly documented by Cloudflare as non-production. The tunnel URL changes every restart. Use **Named Tunnels** with a permanent subdomain (`chat.bayernxtech.de`).

### Run everything as services, not terminal processes
`cloudflared`, the FastAPI server, and OpenClaw should all run as `systemd` services on WSL2. Then a workstation reboot doesn't kill anything — it all comes back automatically within seconds.

### WSL2 service persistence
WSL2 doesn't run systemd by default on older setups. Check `/etc/wsl.conf`:
```ini
[boot]
systemd=true
```
If `systemd=false`, services won't auto-start after reboot.

### Add CORS headers to FastAPI explicitly
```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://bayern-x-tech.github.io"],
    allow_methods=["POST", "GET", "OPTIONS"],
    allow_headers=["Content-Type"],
)
```

### Repo structure — only one repo
The `bayern-x-tech` org has only one repository: `bayern-x-tech.github.io`. All tooling, worker code, and CI lives here. Consider:
- Moving `worker/chat-worker.js` and backend code to a separate private repo
- Adding branch protection on `main`
- Requiring PR reviews for production pushes

---

## Org-wide resilience audit results

| Check | Status | Notes |
|---|---|---|
| Repos in org | 1 | Only `bayern-x-tech.github.io` |
| GitHub Pages | ✅ Healthy | Deploying from `main` |
| GitHub Actions | ✅ | `static.yml` (Pages), `deploy-worker.yml` (CF Worker), `health-check.yml` (new) |
| Branch protection | ❌ None | Direct pushes to `main` allowed |
| Monitoring | ✅ Added | Health check every 6h, opens issue on failure |
| Backend uptime | ❌ Tunnel-dependent | See Priority 1 above |
| OpenClaw | ❌ Down | Needs local restart |
| WhatsApp channel | ❌ Likely 401 | Needs QR re-scan |
| Telegram channel | ❌ Down | Needs `openclaw channels restart telegram` |

---

## Commands to run when back at workstation (in order)

```bash
# 1. Start OpenClaw and messaging channels
openclaw gateway start
openclaw channels restart telegram
openclaw channels link whatsapp   # scan QR on phone

# 2. Verify FastAPI is running
curl http://localhost:8000/health

# 3. Set up permanent named tunnel (one-time)
cloudflared tunnel create xpathfinder-labs
cloudflared tunnel route dns xpathfinder-labs chat.bayernxtech.de
# (create config.yml as above)
sudo cloudflared service install

# 4. Update BACKEND_URL in index.html
# const BACKEND_URL = 'https://chat.bayernxtech.de/api/chat/';
# git add index.html && git commit -m "fix: connect to permanent named tunnel" && git push

# 5. Verify live
curl https://chat.bayernxtech.de/health
# Should return {"status":"ok"} or similar
```
