# Maltrail Kit

Docker Compose deployment of [Maltrail](https://github.com/stamparm/maltrail), built from an unmodified upstream checkout so it can be upgraded with a plain `git pull`. One compose file serves both roles:

| Host role | `COMPOSE_PROFILES` | Runs |
| --- | --- | --- |
| Server + sensor | `server` | `maltrail-config`, `maltrail-sensor`, `maltrail-server`, `nginx` |
| Sensor only | *(empty)* | `maltrail-config`, `maltrail-sensor` → reports to the server over UDP |

```text
                 Server 1 (COMPOSE_PROFILES=server)                    Server 2 (sensor only)
  browser ─HTTPS:NGINX_PORT─▶ nginx ──▶ maltrail-server ◀──UDP:8337── maltrail-sensor
                                              ▲                         (host network)
                                       maltrail_logs volume
                                              │
                                       maltrail-sensor (host network)
```

## Features

- Maltrail server and sensor built from `../source` (`docker/Dockerfile`), with no file in the checkout edited
- Layered configuration: upstream defaults stay current after a `git pull`, and only your changes live in this repo
- Server, and the nginx in front of it, are optional through the `server` profile
- Nginx reverse proxy with HTTPS (self-signed certificates, same setup as backup-kit). The UI port 8338 is never published.
- Per-client login throttling in nginx, plus the `Secure` flag added to Maltrail's session cookie
- Signed sensor → server events (`LOG_SERVER_SECRET`)
- Security hardened: `cap_drop: ALL`, no-new-privileges, AppArmor `docker-default`, read-only root filesystems. The sensor gets only `NET_RAW`/`NET_ADMIN` and is not privileged.
- JSON logging with rotation, CPU/memory limits and healthchecks on every service

## Directory Structure

```text
maltrail/
├── source/                          # git clone of stamparm/maltrail — never edited
└── maltrail-kit/
    ├── docker-compose.yml
    ├── .env.example                 # role, ports, upstream path
    ├── config/
    │   ├── maltrail.kit.conf        # container invariants (tracked)
    │   └── maltrail.local.conf.example
    ├── nginx/
    │   ├── nginx.conf
    │   └── certs/                   # server.crt / server.key (gitignored)
    └── README.md
```

### How the configuration is built

Maltrail reads a single `maltrail.conf`. Editing the tracked upstream file would make every `git pull` conflict, and copying it would freeze its defaults (feeds, capture filter, severity regexes) at the day you copied it. So `maltrail-config`, a one-shot container, concatenates three layers into the `maltrail_config` volume on every `docker compose up`:

1. `../source/maltrail.conf`: upstream defaults, refreshed by `git pull`
2. `config/maltrail.kit.conf`: settings this stack depends on (ports, `USE_SSL false`, `LOG_DIR`)
3. `config/maltrail.local.conf`: your per-host settings and secrets (gitignored)

When an option appears more than once, the last occurrence wins. That holds in both the Python server and the Rust sensor. An array such as `USERS` is replaced as a whole. Server, sensor and the sensor's trail updater all read the merged file through `-c /etc/maltrail/maltrail.conf`.

## Setup

Do this on **each** host.

### 1. Clone both repositories side by side

```bash
mkdir maltrail && cd maltrail
git clone https://github.com/stamparm/maltrail.git source
git clone https://github.com/barrax63/maltrail-kit.git
cd maltrail-kit
```

If the upstream checkout lives elsewhere, set `MALTRAIL_SRC` in `.env`.

### 2. Environment

```bash
cp .env.example .env
```

- Server 1: `COMPOSE_PROFILES=server`
- Server 2: `COMPOSE_PROFILES=` (empty)

Also set `TZ`. On the server host, review `NGINX_PORT`, `EVENTS_UDP_BIND` and `EVENTS_UDP_PORT`.

### 3. Shared secret (once, same value on both hosts)

```bash
openssl rand -hex 32
```

### 4. Per-host Maltrail settings

```bash
cp config/maltrail.local.conf.example config/maltrail.local.conf
```

**Server 1:**

```text
LOG_SERVER_SECRET <the value from step 3>
SENSOR_NAME server1
USERS
    admin:<hash>:0:
```

Build the image first, then generate the hash. The command prompts for the password:

```bash
docker compose build
docker compose run --rm --no-deps maltrail-server python3 server.py --hash-password
```

**Server 2:**

```text
LOG_SERVER_SECRET <the value from step 3>
SENSOR_NAME server2
LOG_SERVER <server 1 address>:8337
```

### 5. TLS certificate (server host only)

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key \
  -out nginx/certs/server.crt \
  -subj "/CN=maltrail.example.lan" \
  -addext "subjectAltName=DNS:maltrail.example.lan,DNS:localhost,IP:127.0.0.1"

chmod 644 nginx/certs/server.key nginx/certs/server.crt
```

### 6. Start

```bash
docker compose up -d --build
docker compose ps
```

The first start downloads and builds the trail set. The sensor turns healthy within about 3 minutes. Then open `https://<server 1>:<NGINX_PORT>`, using the port from `.env`. Server 2 appears under **Sensors** in the UI once its first heartbeat arrives, after at most `HEARTBEAT_PERIOD` (300 s).

## Maintenance

### Update

```bash
git pull                                 # this kit
git -C ../source pull                    # upstream Maltrail
docker compose pull --ignore-buildable   # nginx, busybox
docker compose build --pull              # Maltrail image on fresh base images
docker compose up -d
```

`up` re-runs `maltrail-config`, so new upstream defaults take effect. Before you upgrade, look at what changed in upstream's container setup. The compose file here mirrors `docker/docker-compose.yml` and relies on `docker/entrypoint.sh`:

```bash
git -C ../source fetch && git -C ../source log --oneline HEAD..@{u} -- docker/ maltrail.conf
```

### Change the configuration

Edit `config/maltrail.local.conf`, then:

```bash
docker compose up -d --force-recreate
```

`restart` alone does not re-merge the configuration.

### Check

```bash
docker compose logs -f maltrail-sensor
docker compose logs -f maltrail-server
docker compose exec maltrail-sensor /usr/local/bin/maltrail-entrypoint maltrail-sensor -c /etc/maltrail/maltrail.conf -T
docker compose exec maltrail-server python3 server.py -c /etc/maltrail/maltrail.conf --doctor
docker compose exec maltrail-sensor cat /etc/maltrail/maltrail.conf   # the merged config
```

## Differences from upstream `docker/docker-compose.yml`

| Upstream | Here | Why |
| --- | --- | --- |
| Bind-mounts the repo's `maltrail.conf` | Merges three layers into a volume | Upgrades without editing tracked files |
| Server always runs | `server` profile | Sensor-only hosts |
| UI published on 8338 over plain HTTP | Unpublished, behind nginx with TLS on `NGINX_PORT` | HTTPS and per-client login throttling |
| 8337/udp on all interfaces | `EVENTS_UDP_BIND:EVENTS_UDP_PORT` | Docker ports bypass the host firewall |
| Default capability set | `cap_drop: ALL` + only what the entrypoint/capture need | House baseline |
| Writable root filesystem | `read_only: true` + `/tmp` tmpfs | All writes go to the volumes |
| No limits, default logging | `deploy.resources`, json-file rotation | House baseline |
| `container_name`, `depends_on: [server]` | Neither | The sensor writes to the shared volume and does not need the server running |

The image, entrypoint, uid handling, host networking for the sensor and both healthchecks are upstream's, unchanged.

## Security Considerations

1. **Sensor privileges**: `network_mode: host` with `NET_RAW` + `NET_ADMIN`, not privileged. The upstream entrypoint starts as root (hence `CHOWN`/`SETUID`/`SETGID`), picks the uid for the volumes and execs the sensor as uid 10001, passing only those two capabilities on as ambient capabilities. The server keeps none.
2. **Event intake (8337/udp)**: always set `LOG_SERVER_SECRET`. Without it, anyone who can reach the port can inject detections. The secret authenticates but does not encrypt, so run the traffic over WireGuard/IPsec if it crosses an untrusted network, and bind `EVENTS_UDP_BIND` to that interface.
3. **Clocks**: signed events older or newer than `LOG_SERVER_SKEW` (900 s) are dropped silently. Keep NTP running on both hosts.
4. **Login lockout behind a proxy**: Maltrail does not read `X-Forwarded-For`, so every client looks like nginx. Its own lockout (5 consecutive failed logins) therefore locks the login for everyone, not just the attacker. nginx throttles `/login` per real client IP, but keep the UI reachable only from a LAN or VPN.
5. **Default credentials**: if `USERS` is not set, upstream's `admin`/`changeme!` stays in place. Upstream accepts it only from the server's own loopback, so it cannot log in through nginx.
6. **TLS**: TLS 1.2/1.3 only, HSTS and the usual security headers. Maltrail adds its own CSP. nginx adds `Secure` to the session cookie.
7. **Read-only root filesystems** on all services. If a future upstream version writes outside `/var/log/maltrail`, `/var/lib/maltrail` or `/tmp`, the container log names the path. Add a tmpfs or volume for it rather than dropping `read_only`.
