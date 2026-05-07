# DevOps stack (Docker Compose)

**Caddy** on port **80** (reverse proxy), **GitLab**, **Nextcloud**, and **read-only browse for published releases** at `release.devops.local` (**no login**—trusted LAN only). Hostnames: **`*.devops.local`**. Persistent data: **`/opt/data`** (see [docker-compose.yml](docker-compose.yml)).

LAN uses **plain HTTP**; **do not** expose port 80 to the public internet without TLS or a VPN in front.

---

## Deployment

### 1. Secrets (`.env` location)

Copy [`.env.example`](.env.example) to **`.env` in the same directory as `docker-compose.yml`** (the Compose project root).

**Why here:** Compose loads this file automatically for variable substitution and for services that reference `${VAR}`. After a **host reboot**, `docker compose up -d` (or a systemd unit with `WorkingDirectory` set to this folder) will again see those variables—**no** manual `source .env` step. If `.env` lives elsewhere, you must export variables yourself each time.

```bash
cp .env.example .env
# edit: MYSQL_*, GITLAB_SMTP_PASSWORD, etc.
```

### 2. Data directory

Create once; own as the user that runs Compose so bind mounts stay writable:

```bash
sudo mkdir -p /opt/data
sudo chown "$USER:$USER" /opt/data
```

### 3. Start

```bash
docker compose up -d
```

### 4. Local DNS

Point **`*.devops.local`** at the machine that serves port **80** (`<SERVER_IP>`).

**Windows** (Administrator `CMD`):

```bat
echo <SERVER_IP> git.devops.local cloud.devops.local release.devops.local >> C:\Windows\System32\drivers\etc\hosts
```

**Linux** (`/etc/hosts`):

```bash
echo "<SERVER_IP> git.devops.local cloud.devops.local release.devops.local" | sudo tee -a /etc/hosts
```

*(WSL hitting Windows browser: use the host IP that reaches this stack, not necessarily `127.0.0.1`.)*

### 5. URLs

- `http://git.devops.local` — GitLab  
- `http://cloud.devops.local` — Nextcloud  
- `http://release.devops.local` — Release

---

## Operations

### GitLab Runner (HTTP)

Register (adjust token, name, image as needed):

```bash
export GITLAB_RUNNER_TOKEN=

docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://git.devops.local" \
  --token "${GITLAB_RUNNER_TOKEN}" \
  --name "docker-runner-public" \
  --executor "docker" \
  --docker-image "docker-ci-job:latest" \
  --docker-volumes "/cache" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/builds:/builds" \
  --docker-volumes "/opt/data/release/srv:/srv:rw" \
  --docker-extra-hosts "git.devops.local:host-gateway" \
  --docker-extra-hosts "release.devops.local:host-gateway" \
  --docker-pull-policy "never" \
  --docker-network-mode "devops" \
  --clone-url "http://git.devops.local"
```

*To raise parallelism, edit `concurrent` in `/opt/data/gitlab-runner/config.toml`.*

### Backup and migration

**State:** this repo (Compose + [Caddyfile](Caddyfile)), **`.env`** beside `docker-compose.yml`, and **`/opt/data`**.

**Backup:** `docker compose down`, then archive repo, `.env`, and `/opt/data` (e.g. `sudo tar -C /opt -czf devops-data.tgz data`).

**Restore / new host:** restore **`/opt/data`** as one tree, clone repo with `.env` next to `docker-compose.yml`, `chown` like step 2, update **DNS or `hosts`** if the IP changed, then `docker compose up -d`.
