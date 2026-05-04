# DevOps stack (Docker Compose)

**Caddy** (TLS), **GitLab** + **Registry**, **Nextcloud**, **FileBrowser**. Hostnames are fixed to **`*.devops.local`**. All persistent data is under **`/opt/data`** (see [docker-compose.yml](docker-compose.yml)).

---

## Deployment

### 1. Secrets

Copy [`.env.example`](.env.example) to **`.env` in the same directory as `docker-compose.yml`**, then set `MYSQL_*` and `GITLAB_SMTP_PASSWORD`.

```bash
cp .env.example .env
```

### 2. Data directory (permissions)

Create the tree once; own it as the user that runs `docker compose` so bind mounts and the CA copy step work without fighting root-owned files under `/opt/data`:

```bash
sudo mkdir -p /opt/data
sudo chown "$USER:$USER" /opt/data
mkdir -p /opt/data/{caddy/data,caddy/config,gitlab/{config,logs,data},nextcloud/{mysql,html},filebrowser/{srv,database},gitlab-runner}
```

### 3. Start

```bash
set -a && . .env && set +a
docker compose up -d
```

### 4. Local DNS (hosts)

Run `CMD` as `Administrator`:

```bat
echo <SERVER_IP> git.devops.local registry.devops.local cloud.devops.local files.devops.local >> C:\Windows\System32\drivers\etc\hosts
ipconfig /flushdns
```

Replace `<SERVER_IP>` with the machine that serves ports 80/443.

### 5. URLs

- `https://git.devops.local`
- `https://registry.devops.local`
- `https://cloud.devops.local`
- `https://files.devops.local`

---

## Operations

### GitLab Runner and Caddy `tls internal`

Caddy’s internal root (often root-only readable):

`/opt/data/caddy/data/caddy/pki/authorities/local/root.crt`

After it exists (e.g. visit `https://git.devops.local` once), install a **644** copy for the Runner and jobs:

```bash
sudo install -m 644 \
  /opt/data/caddy/data/caddy/pki/authorities/local/root.crt \
  /opt/data/gitlab-runner/caddy-internal-ca.crt
```

**Register** (replace token/name as needed). Job bind uses the same host path:

```bash
docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "https://git.devops.local" \
  --token "YOUR_REGISTRATION_TOKEN" \
  --name "docker-runner-public" \
  --executor "docker" \
  --docker-image "docker-ci-job:latest" \
  --tls-ca-file /etc/gitlab-runner/caddy-internal-ca.crt \
  --docker-volumes "/cache" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/builds:/builds" \
  --docker-volumes "/opt/data/gitlab-runner/caddy-internal-ca.crt:/etc/ssl/certs/caddy-internal-ca.crt:ro" \
  --env "GIT_SSL_CAINFO=/etc/ssl/certs/caddy-internal-ca.crt" \
  --docker-pull-policy "never" \
  --docker-network-mode "devops" \
  --clone-url "https://git.devops.local"
```

### Backup and migration

**State:** this repo (Compose + [Caddyfile](Caddyfile)), **`.env`**, and **`/opt/data`** (full tree: GitLab, DB, Nextcloud, Caddy, Runner, etc.).

**Backup:** `docker compose down`, then archive the repo, `.env`, and **`/opt/data`** (e.g. `sudo tar -C /opt -czf devops-data.tgz data`).

**Restore or move server (whole package):** restore **`/opt/data` to `/opt/data`** as one tree, place repo + `.env` beside `docker-compose.yml`, `chown` like step 2, point **DNS or `hosts`** for `*.devops.local` at the new IP if the machine changed, then `docker compose up -d`.
