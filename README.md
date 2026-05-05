# DevOps stack (Docker Compose)

**Caddy** (TLS + reverse proxy), **GitLab** + **Registry**, **Nextcloud**, and **directory/file sharing** at `files.devops.local` (Caddy `file_server browse`, read-only tree—**no login**; use only on a **trusted network**). Hostnames are **`*.devops.local`**. Data lives under **`/opt/data`** (see [docker-compose.yml](docker-compose.yml)).

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
# mkdir -p /opt/data/{caddy/data,caddy/config,gitlab/{config,logs,data},nextcloud/{mysql,html},files/srv,gitlab-runner}
```

### 3. Start

```bash
set -a && . .env && set +a
docker compose up -d
```

### 4. Local DNS (hosts)

Replace `<SERVER_IP>` with the machine that serves ports **80/443** for this stack.

**Windows** (run `CMD` as Administrator):

```bat
echo <SERVER_IP> git.devops.local registry.devops.local cloud.devops.local files.devops.local >> C:\Windows\System32\drivers\etc\hosts
```

*127.0.0.1 in WSL localhost mode*

**Linux** (append to `/etc/hosts`):

```bash
echo "<SERVER_IP> git.devops.local registry.devops.local cloud.devops.local files.devops.local" | sudo tee -a /etc/hosts
```

### 5. URLs

- `https://git.devops.local`
- `https://registry.devops.local`
- `https://cloud.devops.local`
- `https://files.devops.local`

### 6. Dev machines: `files.devops.local` and `curl`

The file site uses **Caddy `tls internal`**. Any laptop or CI worker that runs **`curl https://files.devops.local/...`** must first resolve **`files.devops.local`** (see **§4**). For a **trusted LAN**, project tooling may use **`curl -k`** to skip certificate verification so you do not need to install the Caddy root CA on every client; **do not use `-k` on untrusted networks**.


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
export GITLAB_RUNNER_TOKEN=

docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "https://git.devops.local" \
  --token "${GITLAB_RUNNER_TOKEN}" \
  --name "docker-runner-public" \
  --executor "docker" \
  --docker-image "docker-ci-job:latest" \
  --docker-volumes "/cache" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/builds:/builds" \
  --docker-volumes "/opt/data/gitlab-runner/caddy-internal-ca.crt:/etc/ssl/certs/caddy-internal-ca.crt:ro" \
  --docker-volumes "/opt/data/files/srv:/srv:rw" \
  --docker-extra-hosts "git.devops.local:host-gateway" \
  --docker-extra-hosts "registry.devops.local:host-gateway" \
  --docker-pull-policy "never" \
  --docker-network-mode "devops" \
  --tls-ca-file "/etc/gitlab-runner/caddy-internal-ca.crt" \
  --env "GIT_SSL_CAINFO=/etc/ssl/certs/caddy-internal-ca.crt" \
  --env "CURL_CA_BUNDLE=/etc/ssl/certs/caddy-internal-ca.crt" \
  --clone-url "https://git.devops.local"
```

### Backup and migration

**State:** this repo (Compose + [Caddyfile](Caddyfile)), **`.env`**, and **`/opt/data`** (GitLab, DB, Nextcloud, Caddy, Runner, etc.).

**Backup:** `docker compose down`, then archive the repo, `.env`, and **`/opt/data`** (e.g. `sudo tar -C /opt -czf devops-data.tgz data`).

**Restore or move server (whole package):** restore **`/opt/data` to `/opt/data`** as one tree, place repo + `.env` beside `docker-compose.yml`, `chown` like step 2, point **DNS or `hosts`** for `*.devops.local` at the new IP if the machine changed, then `docker compose up -d`.

