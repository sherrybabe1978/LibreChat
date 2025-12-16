# DigitalOcean + Cloudflare + R2 + Claude — Setup Checklist (chat.9data.org)

Single-user deployment guide to run LibreChat on a DigitalOcean Droplet with:
- **Domain:** `chat.9data.org`
- **HTTPS:** Cloudflare in front (recommended: Full (Strict))
- **Model provider:** Anthropic Claude
- **Cloud file uploads:** Cloudflare R2 (S3-compatible)

This doc is intentionally step-by-step with copy/paste commands.

---

## 0) What you’ll deploy

We will use `deploy-compose.yml` from this repo.

Services:
- `LibreChat-API` (app backend)
- `LibreChat-NGINX` (80/443 termination on the droplet)
- `chat-mongodb`
- `chat-meilisearch`
- `rag_api` + `vectordb` (present even if you don’t use document chat yet)

---

## 1) Provision the DigitalOcean Droplet

Recommended minimum for "not slow":
- Ubuntu 24.04 LTS (or 22.04)
- **4 vCPU / 8GB RAM**
- 50–100GB SSD
- SSH key auth

After creation you’ll have:
- Droplet public IP: `YOUR_DROPLET_IP`

---

## 2) Cloudflare DNS for `chat.9data.org`

In Cloudflare (zone: `9data.org`):

1) Create an **A record**:
- Name: `chat`
- IPv4: `YOUR_DROPLET_IP`
- Proxy status: **Proxied** (orange cloud)

2) SSL/TLS settings:
- Set **SSL/TLS encryption mode: Full (Strict)**

---

## 3) HTTPS certificate strategy (pick ONE)

### Option A (recommended w/ Cloudflare): Cloudflare Origin Certificate
1) Cloudflare → SSL/TLS → Origin Server → **Create Certificate**
2) Add hostnames:
- `chat.9data.org`
3) Download:
- Origin Certificate (PEM)
- Private Key (PEM)
4) On the droplet, you’ll place these for Nginx to use.

Pros: simple, very reliable with Cloudflare.
Cons: cert is trusted by Cloudflare, not by arbitrary direct visitors (fine since you’ll keep proxy on).

### Option B: Let’s Encrypt on the droplet
Pros: standard public cert.
Cons: more moving parts.

---

## 4) Basic firewall rules (DO Cloud Firewall recommended)

Inbound allow:
- TCP 80 from anywhere
- TCP 443 from anywhere

Inbound allow (admin):
- TCP 22 from **your IP only**

Inbound deny:
- Everything else

Do **not** expose MongoDB/Meilisearch ports.

---

## 5) SSH into the droplet

```bash
ssh root@YOUR_DROPLET_IP
```

---

## 6) Install Docker + Compose (Ubuntu)

Run the following on the droplet:

```bash
apt-get update -y
apt-get install -y ca-certificates curl gnupg

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
  > /etc/apt/sources.list.d/docker.list

apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

docker --version
docker compose version
```

---

## 7) Put LibreChat on the droplet

### Option A: Clone this repo
```bash
apt-get install -y git
git clone https://github.com/danny-avila/LibreChat.git
cd LibreChat
```

If you’re deploying your fork, clone that instead.

---

## 8) Create `.env` (Claude first)

On the droplet in the repo root:

```bash
cp .env.example .env
```

Edit `.env` (use `nano .env`) and set at minimum:

### Required
- `HOST=0.0.0.0`
- `PORT=3080`
- `DOMAIN_CLIENT=https://chat.9data.org`
- `DOMAIN_SERVER=https://chat.9data.org`

### Anthropic
- `ANTHROPIC_API_KEY=YOUR_ANTHROPIC_KEY`
- (Recommended) limit models:
  - `ANTHROPIC_MODELS=claude-3-5-haiku-20241022,claude-3-7-sonnet-20250219`

### Secrets (REQUIRED to change)
Replace these with new random values:
- `JWT_SECRET=...`
- `JWT_REFRESH_SECRET=...`

### Meilisearch key
Set a strong value:
- `MEILI_MASTER_KEY=...`

---

## 9) Start LibreChat using deploy compose

From repo root:

```bash
docker compose -f deploy-compose.yml up -d
docker compose -f deploy-compose.yml ps
```

Logs (if needed):
```bash
docker compose -f deploy-compose.yml logs -f --tail=200
```

---

## 10) Connect HTTPS (Nginx)

`deploy-compose.yml` runs an nginx container that listens on 80/443 and mounts:
- `./client/nginx.conf` → `/etc/nginx/conf.d/default.conf`

You will need to ensure `client/nginx.conf` is configured for:
- server_name `chat.9data.org`
- TLS certificate + key paths

Then mount the cert/key into the nginx container (commonly done via a compose override).

> When you’re ready to implement this, share your preferred cert option (Cloudflare Origin Cert vs Let’s Encrypt) and I’ll provide an exact `docker-compose.override.yml` and nginx config changes.

---

## 11) Create your user, then lock it down

1) Visit `https://chat.9data.org`
2) Register your account
3) After confirming it works, disable registration in `.env`:
- `ALLOW_REGISTRATION=false`

Restart:
```bash
docker compose -f deploy-compose.yml up -d
```

---

## 12) Cloud uploads with Cloudflare R2 (later, but planned)

High level:
1) Create an R2 bucket (private)
2) Create R2 access key + secret
3) Configure LibreChat to use S3 storage for uploads
   - Prefer configuring `fileStrategy: s3` in `librechat.yaml`
   - And set S3 endpoint/credentials via `.env`

> When you’re ready, tell me whether you want R2 behind a custom domain or just direct S3 endpoint usage; I’ll provide exact variables for this repo/version.

---

## 13) Backups (minimum viable)

Back up at least:
- `./data-node` (MongoDB data)
- `./meili_data_v1.12` (optional; can rebuild)
- `./uploads` (if you haven’t moved to R2 yet)

---

## 14) Quick validation checklist

- [ ] `docker compose -f deploy-compose.yml ps` shows all services healthy/running
- [ ] `https://chat.9data.org` loads
- [ ] Claude responses work
- [ ] Registration disabled after account created
