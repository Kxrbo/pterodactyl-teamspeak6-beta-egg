# pterodactyl-teamspeak6-beta-egg

A **Pterodactyl Egg** for running the **TeamSpeak 6 Server (Beta)** on Wings.

This egg is designed for people who want to host TS6 inside Pterodactyl while keeping data persistent and management simple.

---

## Why this egg?

Pterodactyl already supports voice servers well, but TeamSpeak 6 (Beta) has a few quirks compared to TS3:

- TS6 expects a writable **data/log directory** (commonly `/var/tsserver`)
- TS6 requires **explicit license acceptance**
- Depending on your Wings configuration, mounts can be blocked unless explicitly allowed

This egg documents a **known-good setup** and common pitfalls.

---

## Features

- ✅ Uses **official TeamSpeak 6 server image** (`teamspeaksystems/teamspeak6-server:latest`)
- ✅ License handling via `TSSERVER_LICENSE_ACCEPTED` (`accept` / `view`)
- ✅ Standard TeamSpeak ports configurable via panel variables
- ✅ Persistent storage via **mount to `/var/tsserver`**
- ✅ Troubleshooting section for the most common TS6 crashes (log directory / permissions / mounts)

---

## Index

1. [Requirements](#requirements)
2. [Networking / Ports](#networking--ports)
3. [Install (Import Egg)](#install-import-egg)
4. [Create the Server](#create-the-server)
5. [Create & Attach the Mount](#create--attach-the-mount)
6. [Enable Mounts in Wings (IMPORTANT)](#enable-mounts-in-wings-important)
7. [First Start (License)](#first-start-license)
8. [Troubleshooting](#troubleshooting)
9. [License](#license)

---

## Requirements

- ✅ Working and up-to-date **Pterodactyl Panel** + **Wings**
- ✅ **Admin access** to the Pterodactyl Panel
- ✅ SSH **root access** to the Wings node (recommended for troubleshooting)
- ✅ Firewall must allow:
  - Pulling Docker images from registries (Docker Hub / GHCR)
  - (Optional) Access to GitHub if you use scripts/repo distribution

---

## Networking / Ports

Create allocations for the ports you want to use.

### Required
- **9987/UDP** — Voice (Primary allocation)

### Recommended
- **30033/TCP** — File transfer

### Optional (ServerQuery)
- **10011/TCP** — Query (raw)
- **10022/TCP** — Query via SSH
- **10080/TCP** — Query via HTTP

> Tip: In Pterodactyl, the **Primary allocation** becomes `{{SERVER_PORT}}`.

---

## Install (Import Egg)

1. Download/copy the egg JSON file.
2. In the Panel go to:
   - **Admin Area → Nests**
3. Create a custom Nest (recommended).
   > Do **not** use default nests if you can avoid it — updates may overwrite them.
4. Go into your Nest → **Eggs → Import Egg**
5. Upload/select your `egg-teamspeak6-server.json`
6. You should see: **“Egg successfully imported”**

---

## Create the Server

1. **Admin Area → Servers → Create New**
2. Select your TS6 Nest + Egg
3. Choose Docker Image:
   - `teamspeaksystems/teamspeak6-server:latest`
4. Set allocations:
   - Primary: **9987/UDP**
   - Additional: **30033/TCP** (+ optional query ports)
5. Create the server

---

## Create & Attach the Mount

TeamSpeak 6 must be able to write logs/data. The official image expects it at:

- **`/var/tsserver`**

### Create the Mount

1. **Admin Area → Mounts → Create New**
2. Set:
   - **Source:** `/var/lib/pterodactyl/volumes/<SERVER_UUID>`
   - **Target:** `/var/tsserver`
   - **Read Only:** `false`
3. Assign the mount to:
   - the correct **Node**
   - the correct **Egg**

### Attach the Mount to the Server

1. **Admin → Servers → (your server) → Mounts**
2. Tick/Attach your TS6 mount
3. Save

✅ If correctly applied, the container will be able to create:
- `/var/tsserver/logs`

---

## Enable Mounts in Wings (IMPORTANT)

If your Wings config has `allowed_mounts: []`, bind mounts may be blocked and Wings may silently fall back to **Docker-managed volumes**.

### Fix `allowed_mounts`

Edit:

```bash
nano /etc/pterodactyl/config.yml
```

Set:

```yaml
allowed_mounts:
  - /var/lib/pterodactyl/volumes
```

> YAML indentation matters: `allowed_mounts:` must start at the left margin, and the list item must be indented with **two spaces**.

Restart Wings:

```bash
systemctl restart wings
systemctl status wings --no-pager -l
```

---

## First Start (License)

Before TS6 will start, you must accept the license.

In the server **Variables**, set:

- `TSSERVER_LICENSE_ACCEPTED = accept`

Other supported value:
- `view` (prints the license text)

Start the server. On first successful startup you should see output including admin/privilege information in the logs.

---

## Troubleshooting

### ❌ `Failed to create log directory "/var/tsserver/logs"`

This means TS6 cannot write to `/var/tsserver`.

**Checklist**
- Is the mount attached to the server?
- Is `/var/tsserver` really bind-mounted, or is it a Docker volume fallback?

**Check which mount is actually used**
On the node:

```bash
CID=$(docker ps -a --format "{{.ID}} {{.Image}} {{.CreatedAt}}" | awk '$2=="teamspeaksystems/teamspeak6-server:latest"{print $1; exit}')
docker inspect "$CID" --format '{{range .Mounts}}{{println .Source "->" .Destination "rw=" .RW}}{{end}}'
```

You want to see:

```
/var/lib/pterodactyl/volumes/<SERVER_UUID> -> /var/tsserver rw= true
```

If you instead see:

```
/var/lib/docker/volumes/<random>/_data -> /var/tsserver
```

then Wings blocked your bind mount (usually `allowed_mounts`), and you must fix the Wings config.

---

### ❌ Wings fails after editing config.yml (YAML error)

Example:
`yaml: block sequence entries are not allowed in this context`

Your indentation is broken.

Print a section around the error line:

```bash
nl -ba /etc/pterodactyl/config.yml | sed -n '95,120p'
```

Fix indentation and restart Wings.

---

### ❌ TS6 instantly crashes (exit 139)

Most commonly:
- cannot write to `/var/tsserver/logs`
- license not accepted (`TSSERVER_LICENSE_ACCEPTED` not set to `accept`)

---

## License

pterodactyl-teamspeak6-beta-egg is GNU General Public License v3.0. Please check the License before performing any changes on the scripts.

---
