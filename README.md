# Docker container logging stack

*English | [简体中文](README.zh-CN.md)*

All Docker containers on this host, auto-discovered and searchable in Grafana.

```
  every container on the host
            |  (Docker API: discovery + log tailing)
            v
      Grafana Alloy  --push-->  Loki  <--query--  Grafana
     (logging-alloy)         (logging-loki)    (logging-grafana)
                                                       |
                                              http://localhost:24680
```

| Component | Image | Role | Published |
| --- | --- | --- | --- |
| Loki | `grafana/loki:3.7.6` | log store, 30-day retention | no - network only |
| Alloy | `grafana/alloy:v1.18.1` | discovers + tails every container | UI on `127.0.0.1:12345` |
| Grafana | `grafana/grafana:13.1.3` | query UI, provisioned from files | `127.0.0.1:24680` |

Targets Docker Engine 29.x with Compose v5.x. The obsolete top-level `version:` key is
deliberately absent from `docker-compose.yml`.

## Layout

```
.
|-- docker-compose.yml
|-- .env                      (you create this; git-ignored)
|-- .env.example
|-- .gitignore
`-- config/
    |-- loki/loki.yaml                            # store + retention
    |-- alloy/config.alloy                        # discovery + tailing
    `-- grafana/
        |-- provisioning/
        |   |-- datasources/loki.yml              # the Loki datasource, uid: loki
        |   `-- dashboards/dashboards.yml         # dashboard provider
        `-- dashboards/
            |-- container-logs.json               # overview: is anything wrong?
            `-- log-search.json                   # search: find and read lines
```

The two provisioning subdirectories are mounted individually rather than mounting the
`provisioning/` parent. Mounting the parent would hide the `plugins/` and `alerting/`
directories that ship inside the Grafana image, and Grafana would log an ERROR for each
on every boot - noise that Alloy then collects and the dashboard's error panel counts.

## First run

### Step 0 - tear down the previous stack (DESTROYS all stored logs)

**Do not skip this.** `docker compose up -d` recreates containers but *never* recreates an
existing named volume. The volumes `logging_loki-data`, `logging_alloy-data` and
`logging_grafana-data` from the previous install are still on this host, and inheriting them
breaks the rebuild in three silent ways: the old `grafana.db` already has an `admin` user so
your new `GRAFANA_ADMIN_PASSWORD` is ignored, the old Loki data was written under a different
schema, and the retained streams carry a different `job` label than the one every panel here
selects on.

```bash
cd /home/wren/codes/logging
docker rm -f logging-loki logging-alloy logging-grafana 2>/dev/null || true
docker volume rm -f logging_loki-data logging_alloy-data logging_grafana-data 2>/dev/null || true
docker network rm logging 2>/dev/null || true
```

### Step 1 - bound Docker's own log files (do this before the first start)

See [Host-wide log rotation](#host-wide-log-rotation). Without it, first boot replays a large
backlog and `/var/lib/docker/containers/` grows without limit forever. Loki's 30-day retention
does **not** cover those files.

### Step 2 - configure and start

```bash
cp .env.example .env
$EDITOR .env                     # set GRAFANA_ADMIN_PASSWORD - the stack will not start without it
docker compose up -d
docker compose ps
```

If you reach this machine over the network rather than sitting at it, also set
`GRAFANA_BIND_IP=0.0.0.0` and `GRAFANA_ROOT_URL=http://<host>:24680` in `.env` - and put a
firewall or reverse proxy in front of port 24680, because Grafana is then the only thing
between the network and your logs. The previous stack on this host published `0.0.0.0:3000`;
the default here is loopback-only, which will look like a total outage to a remote user with
no error message anywhere.

Open <http://localhost:24680> and log in. You land directly on **Container Logs**. The Loki
datasource and both dashboards are already there - nothing to click.

There are two dashboards and they answer different questions:

| | **Container Logs** | **Log Search** |
| --- | --- | --- |
| Answers | Is anything wrong, and where? | Where is that specific line? |
| Shows | Error count and share, volume by severity, busiest containers, a short strip of recent problems | Filters, a severity histogram to navigate time, and a large log list |
| Reach it | the home dashboard | the `Log Search` link in its header |

The header link on each page carries your current filters and time range to the other, so
narrowing to one service on the overview and clicking through lands you on those same lines.
Clicking a container name in the busiest-containers table does the same for that container.

Log lines render one per line with no label prefix. Click any line to open it beside the list
with the full text and every label and structured-metadata field (`container_id`, `image`).

**Why 24680 and not Grafana's usual 3000:** 3000 and 3001 are dev-server territory on this
host - 3000 is held by a long-running `rsbuild dev` server (the `tokenpapa` frontend project),
and anything else you start next will reach for 3001. Compose fails to start Grafana with
`address already in use` the moment it loses that race. 24680 is unclaimed, is not a
registered service port, and sits below this kernel's ephemeral range (32768-60999), so it
cannot be stolen by an outbound connection's source port either. Only the host-side port
moved: Grafana still listens on 3000 *inside* the container, which is why the healthcheck and
the right-hand side of the `ports:` mapping still say 3000. Change `GRAFANA_PORT` in `.env` if
you want a different one - `GRAFANA_ROOT_URL` must be changed to match, or share links and
redirects point at the old port.

## What gets collected

Every container on the host, discovered through the Docker API and refreshed every 2 seconds.
Start a new container and its logs appear without touching any config file.

Stream labels - the contract between `config/alloy/config.alloy` and the dashboard's LogQL.
Change one, change the other:

| Label | Source | Notes |
| --- | --- | --- |
| `job` | constant `docker` | every stream carries it; `{job="docker"}` is "everything" |
| `container` | `__meta_docker_container_name` | leading `/` stripped |
| `compose_project` | `com.docker.compose.project` | absent for non-Compose containers |
| `compose_service` | `com.docker.compose.service` | absent for non-Compose containers |
| `service_name` | compose service, else container name | Loki 3.x / Logs Drilldown convention |
| `stream` | `__meta_docker_container_log_stream` | `stdout` / `stderr` |

Because `compose_project` and `compose_service` are missing on non-Compose containers, every
template variable uses `allValue: ".*"` rather than `".+"`. A matcher that matches the empty
string also selects series that lack the label, so "All" really does mean all. Switching those
to `.+` would silently hide every standalone container.

### Structured metadata (per entry, not indexed)

| Field | Notes |
| --- | --- |
| `container_id` | first 12 chars; tells you *which incarnation* of a recreated container logged a line |
| `image` | **best effort - see below** |

Filter on them in LogQL: `{job="docker"} | container_id="a1b2c3d4e5f6"`. They are stored inside
the chunk, not in the stream index, so they cost no cardinality. That matters most for `image`:
as an index label it would mint a brand-new stream for every container on every release.

**`image` is incomplete, and that is a genuine gap against the requirement.** Alloy's Docker
discovery delegates to Prometheus's moby SD, which exports only container id, name and
network-mode plus `container_label_*` / `network_*` / `port_*` - there is no
`__meta_docker_container_image`. The config recovers what it can, worst source first:

1. `com.docker.compose.image` - Compose stamps this on everything it creates, but the value is
   an image **digest** (`sha256:...`), not a readable tag.
2. `org.opencontainers.image.version` / `org.opencontainers.image.ref.name` - OCI labels that
   Docker copies from the image onto the container. Many images set them; many do not.
3. `logging.image` - stamp it yourself on containers you control:
   `labels: ["logging.image=myrepo/myapp:1.2.3"]`. This is the only source that always yields a
   readable tag.

A plain `docker run` container from an image with no OCI labels gets no `image` value at all.

### Not collected

By design and by requirement: journald, host file logs, and any external/remote push. This
stack only reads container stdout/stderr through the Docker API.

### Excluding a container

```yaml
labels:
  logging.exclude: "true"
```

**Alloy and Loki both carry this label**, and both are additionally dropped by a
project+service rule in `config.alloy` as belt and braces. Both sit on the push path, so their
own error output feeds back into itself:

- Alloy: a failed push logs an error, which would be collected, pushed, and fail again.
- Loki: every ingestion rejection (429 rate limit, 400 too-old, disk full) writes an error line
  to Loki's stdout. Shipping that back produces another rejection and another line - a loop
  that engages precisely when Loki is already degraded and you most need it to recover.

**Grafana carries the label too, but for a different reason: noise, not safety.** It sits off
the push path and cannot amplify. It is excluded because every panel refresh emits several
`logger=tsdb.loki endpoint=queryData` lines, so on a quiet host Grafana narrating its own
queries drowns out the containers you actually want to read. Drop the label from
`docker-compose.yml` and recreate the container to put its logs back.

So all three stack containers are excluded, and their logs are reachable only through
`docker compose logs -f {alloy,loki,grafana}`.

This label is also the escape hatch for a container whose log driver cannot be read back, or
one so chatty it is not worth indexing.

### Known collection gaps

These are real, none of them can be closed by configuration, and all of them fail *silently*:

- **Short-lived containers.** Discovery is a poll, not an event stream. A container whose entire
  lifetime is under ~2s is never discovered and none of its logs are collected -
  `docker compose run --rm` one-offs, init/migration jobs, and containers that crash-loop
  quickly (which are exactly the ones you want). No error is produced anywhere.
- **Containers with no network.** Upstream discovery builds targets by iterating a container's
  networks; `network_mode: none` (or an unresolvable `network_mode: container:<id>`) yields zero
  targets. Use an `internal: true` network for isolation instead.
- **Unreadable log drivers.** `GET /containers/{id}/logs` works natively only for `json-file`,
  `local` and `journald`. Other drivers work only through Docker's dual-logging cache, and
  `logging: {driver: none}` makes a container permanently invisible *and* produces a
  "could not fetch logs" error on every discovery tick. Never set `driver: none` on this host.
- **Container restarts re-read history.** When a container stops, its target disappears, the
  tailer stops and *deletes* its position entry. A `docker restart` therefore re-reads that
  container's retained log from the beginning. Bounded by the 24h age drop in `config.alloy`
  and by daemon-level json-file rotation.
- **Restarts duplicate ~1s.** Positions are stored at second granularity and Docker's `since` is
  inclusive, so an Alloy restart replays up to one second of already-ingested lines per
  container. Bounded, expected, not a bug to chase.

Verify coverage rather than assuming it:

```bash
docker ps --format '{{.Names}}' | sort
# compare against, in Grafana Explore:
#   count by (container) (count_over_time({job="docker"}[1h]))
# The difference should be exactly logging-alloy and logging-loki.
```

## Retention

30 days, written literally as `retention_period: 720h` in `config/loki/loki.yaml` and enforced
by Loki's compactor, which unlinks expired chunks from the `loki-data` volume.
`compactor.retention_enabled: true` is what makes deletion happen at all; without it the
compactor compacts forever and deletes nothing, silently.

Deletion is not instantaneous. Retention runs every 20 minutes, marks expired chunks, and the
sweeper unlinks them after `retention_delete_delay: 2h`. Expect expired data to linger a few
hours past the 30-day mark.

The container does **not** run with `-config.expand-env`, so `${...}` has no meaning inside
`loki.yaml`. That is deliberate: Loki's expander is drone/envsubst, it documents no escape for a
literal `$`, and an unparsable `${...}` is a hard boot failure (`bad substitution`). Retention is
a fixed requirement, not a per-environment value, so the indirection bought nothing and could
only break the stack.

**Retention does not bound disk growth in the first 30 days.** Nothing is deleted until an index
table's whole 24h window ages out, and Loki has no size-based retention. The real backstops are
`ingestion_rate_mb` / `per_stream_rate_limit` in `loki.yaml` and the per-container `stage.limit`
in `config.alloy`. Sustainable steady state here is roughly 738GB / 30d ~= 24GB/day on disk.

## Grafana as code

Nothing in Grafana is configured by clicking.

- **Datasource** - `config/grafana/provisioning/datasources/loki.yml`, `apiVersion: 1`,
  `uid: loki`, `editable: false`. `prune: true` makes the file authoritative: delete an entry
  here and Grafana deletes the datasource on next start.
- **Dashboards** - `config/grafana/provisioning/dashboards/dashboards.yml` is only a *provider*;
  it points at a directory. The dashboards are the JSON files in that directory.
  `allowUiUpdates: false` means the UI cannot save over them.
- `disableDeletion: false` is deliberate and is the dashboard equivalent of `prune: true`.
  Setting it to `true` does **not** mean "the repo is authoritative" - it means "unprovision
  instead of delete", so the moment a JSON file is not visible to Grafana (removed from git,
  branch switched, mount pointed elsewhere) the dashboard survives as an ordinary, freely
  editable DB row. That is the click-ops orphan this stack exists to avoid. UI deletion of a
  provisioned dashboard is blocked by Grafana independently of this flag.
- The provider rescans every 30s, so editing a JSON file on the host shows up without a restart.

The `uid: loki` in the datasource file and the `"uid": "loki"` in every dashboard panel and
template variable are the same contract. Change one without the other and every panel shows
"Datasource loki was not found".

`jsonData.maxLines: 2000` in the datasource must stay at or below
`limits_config.max_entries_limit_per_query: 10000` in `loki.yaml`. If it ever exceeds it, every
panel fails at once with a Loki-side error that names a Loki limit, not a Grafana setting.

Container Logs ships `"refresh": "1m"`; Log Search ships no auto-refresh at all, deliberately -
results should hold still while you read them. Both pickers list 1m and up, and
`GF_DASHBOARDS_MIN_REFRESH_INTERVAL=1m` enforces that floor server-side (Grafana drops shorter
entries at construction, so listing 10s would be dead JSON). Several panels are instant queries
anchored to `[$__range]`; at a 30-day range with a 10s refresh they would stack full-scan
aggregations on a single-binary Loki faster than it can answer them. Use Explore -> Live for
true tailing.

### Adding a dashboard

1. Build it in the UI.
2. **Export -> Save to file** (export *for sharing externally* is not wanted - keep the
   hardcoded `"uid": "loki"`, do not let Grafana turn it into a `__inputs` placeholder).
3. Drop the JSON in `config/grafana/dashboards/`, give it a stable top-level `"uid"`, commit.

Grafana 13 writes `"schemaVersion": 42`. Older exports are migrated forward on load.

## Host-wide log rotation

The `logging:` block in `docker-compose.yml` caps *these three* containers at 3 x 20 MB. It
cannot constrain containers defined elsewhere, and **Loki's 30-day retention does not cover
Docker's own log files at all** - Alloy reads them through the API and never truncates them.
Without rotation, `/var/lib/docker/containers/*/*-json.log` grows without limit for the life of
every container; when the filesystem fills it takes the Docker daemon down, and with it Loki,
Grafana and Alloy simultaneously.

`/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
```

```bash
# log-driver / log-opts are NOT in dockerd's SIGHUP-reloadable set, so
# `systemctl reload docker` exits 0 and silently does nothing. Restart is required.
sudo systemctl restart docker
docker info --format '{{.LoggingDriver}}'
```

This applies to containers created afterwards; existing ones keep their original settings until
recreated. 50m x 3 = 150MB per container is also Alloy's catch-up budget if it stalls - data
rotated away before Alloy reads it is lost, so do not shrink this to 10m x 1.

If this host has been running for months and you want to avoid a large first-boot replay, zero
the existing backlog before the first start (safe - Docker holds the fd and keeps appending):

```bash
sudo truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

Alloy additionally drops any entry older than 24h (`stage.drop { older_than = "24h" }`), which
bounds the backfill even if you skip the truncate.

## Security decisions

### Docker socket exposure - read the whole section

Alloy mounts `/var/run/docker.sock:ro`. **The `:ro` flag is not a security boundary.** It makes
the socket *file* read-only; it does not filter the Docker API. Anything that can talk to that
socket can `POST /containers/create` with `Privileged: true` and `Binds: ["/:/host"]` and own
the host. Docker socket access is root access, full stop, and Alloy runs as root.

**Shipped default: direct mount.** Rationale - this is a single-node, single-tenant host; Alloy
is a first-party Grafana binary running a locally authored config, with remote config disabled
and its unauthenticated UI bound to loopback. The requirement is to reliably collect *all*
containers, and putting an HTTP proxy in front of long-lived `follow` log streams is the exact
path that produces truncated streams and gaps.

**Use the socket proxy if this host runs anything untrusted or internet-facing.** It reduces a
hypothetical Alloy compromise from host-root to a read-only Docker API. Save as
`docker-compose.socket-proxy.yml`:

```yaml
services:
  docker-socket-proxy:
    image: tecnativa/docker-socket-proxy:v0.5.0
    container_name: logging-docker-socket-proxy
    restart: unless-stopped
    environment:
      CONTAINERS: 1   # /containers/json and /containers/{id}/logs
      NETWORKS: 1     # discovery.docker calls NetworkList on EVERY refresh - required
      VERSION: 1      # API-version negotiation on client startup
      POST: 0         # deny every write verb - this is the actual boundary
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - logging
    labels:
      logging.exclude: "true"
    security_opt:
      - no-new-privileges:true
    mem_limit: 64m
    memswap_limit: 64m
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  alloy:
    # !override replaces the list instead of merging, which drops the docker.sock bind.
    volumes: !override
      - ./config/alloy/config.alloy:/etc/alloy/config.alloy:ro
      - alloy-data:/var/lib/alloy/data
    depends_on:
      docker-socket-proxy:
        condition: service_started
      loki:
        condition: service_started
```

The daemon address is hardcoded in `config.alloy` on purpose (an unset environment variable
would otherwise produce an empty host and a confusing failure), so switching also means editing
**both** `host =` lines in `config/alloy/config.alloy`:

```
discovery.docker "containers"   -> host = "tcp://docker-socket-proxy:2375"
loki.source.docker "containers" -> host = "tcp://docker-socket-proxy:2375"
```

```bash
docker compose -f docker-compose.yml -f docker-compose.socket-proxy.yml up -d
```

The proxy is HAProxy, and its `timeout server` will cut the long-lived `logs?follow=1` stream
when it expires. Alloy reconnects from its stored position, so this is churn rather than loss -
but confirm log flow survives past the timeout window before calling it done. Watch for gaps; if
you see them, the direct mount is the better trade.

Running Alloy as a non-root user in the host's `docker` group is *not* a substitute. The
privilege lives in the API, not the uid.

### Anonymous access: off, and worth keeping off

`GF_AUTH_ANONYMOUS_ENABLED=false`. Container logs routinely contain bearer tokens, connection
strings, customer identifiers and stack traces. Anonymous viewer access would hand all of that
to anything that can reach port 24680.

### Published ports

Loki is never published - it is reachable only as `http://loki:3100` inside the `logging`
network. Loki has no authentication of its own, so publishing it would expose an unauthenticated
read/write log API. Grafana and the Alloy UI bind to `127.0.0.1` by default; the Alloy UI in
particular has no login at all and renders every argument value in the config graph.

### Other hardening

`no-new-privileges:true` on every service; `GF_USERS_ALLOW_SIGN_UP=false`; Grafana's phone-home,
update check, plugin-update check, news feed and boot-time plugin preinstall are all disabled,
which also makes startup work on an air-gapped host. Loki's `analytics.reporting_enabled` is
false and Alloy runs with `--disable-reporting`.

## Why Loki has no healthcheck

`grafana/loki:3.7.6` is a Bazel-built distroless Debian 13 image (`docker history` shows
`cacerts_debian13`, `os_release_debian13` and trixie `base-files`/`netbase`/`tzdata`). It ships
no shell, no `wget` and no `curl`; `/usr/bin/loki` is the entrypoint, and `USER 10001` plus
`/loki` ownership come from Loki's own layers (`COPY --chown=10001:10001 /loki /loki`), which is
why `user: "10001:10001"` in `docker-compose.yml` matches the image exactly.

Docker healthchecks run *inside* the container, so there is nothing to execute. The service
declares `healthcheck: disable: true` to record that this is deliberate, and dependents wait on
`service_started`. Ask a container that does have curl when you want a real answer:

```bash
docker compose exec -T grafana curl -fsS http://loki:3100/ready
```

Alloy's healthcheck probes **`/-/healthy`**, not `/-/ready`. `/-/ready` only reports that the
initial config load finished, so it returns 200 even when `loki.write` is failing every push and
zero logs are shipping. `/-/healthy` enumerates every component and returns 500 naming any that
are unhealthy - which is the failure this stack most needs to surface. The Alloy image ships
neither curl nor wget, so the probe uses bash's `/dev/tcp`.

**Consequence to expect:** if any single container on the host uses an unreadable log driver,
the Alloy container can report `unhealthy` while collecting everything else perfectly. Read the
component graph at <http://127.0.0.1:12345> before assuming the collector is down.

## Durability and what a Loki outage costs

`loki.write` buffers in memory and retries with exponential backoff - 20 retries, 500ms to 5m,
roughly one hour of cover. **After that the in-flight batch is discarded permanently**, and
because the tailer has already advanced its read position those lines are never re-read. Alert
on `loki_write_dropped_entries_total`.

Alloy's on-disk WAL would close that window, but the `wal` block is an experimental feature that
requires running the whole process with `--stability.level=experimental`. That is deliberately
not enabled here. To opt in: add `--stability.level=experimental` to the alloy `command:`, add
`wal { enabled = true, max_segment_age = "1h" }` to the `loki.write` endpoint block, and raise
Alloy's `stop_grace_period` above the 30s default drain timeout.

On the Loki side, `ingester.wal.flush_on_shutdown: true` means SIGTERM really does flush head
chunks (the default is false, which would make the 60s `stop_grace_period` pointless).

## Resource footprint

This host has 30 GiB RAM with ~17 GiB in use, 16 cores, 8 GiB of swap and 738 GiB free disk.

| Service | Typical RSS | `mem_limit` = `memswap_limit` | `cpus` | `GOMEMLIMIT` |
| --- | --- | --- | --- | --- |
| Loki | 300-900 MiB | 4 GiB | 4.0 | 3600MiB |
| Alloy | 100-250 MiB | 1 GiB | 2.0 | 900MiB |
| Grafana | 100-200 MiB | 1 GiB | 2.0 | 900MiB |

Two settings make those limits mean something:

- **`memswap_limit` equal to `mem_limit`.** This host runs cgroup v2 with 8 GiB of swap. With
  `mem_limit` alone, `memory.swap.max` stays at `max` and a container that reaches its limit is
  not killed - it spills into swap on the same NVMe device that holds `/var/lib/docker` and the
  whole host degrades into IO thrash while the container stays alive and unresponsive. Equal
  values give a hard cap and a fast OOM-kill that `restart: unless-stopped` recovers from.
- **`GOMEMLIMIT`.** Go reads cgroup *CPU* limits since 1.25 but never cgroup *memory* limits, so
  without this the GC has no idea `mem_limit` exists and grows the heap straight through it.
  Set to ~90% of each limit so the GC works harder before the kernel intervenes.

Loki's cache ceiling is ~640 MiB, not the 128 MiB its `results_cache` block appears to say:
`cache_index_stats_results`, `cache_volume_results`, `cache_series_results` and
`cache_label_results` all default to true and each **clones** `results_cache` into its own
independent embedded cache. There is deliberately no `chunk_store_config` chunk cache - the
object store is a local directory, so those bytes are already served free from the kernel page
cache without GC pressure.

`ingester.wal.replay_memory_ceiling` is pinned to 2GB (default 4GB) so WAL replay after an
unclean restart cannot exceed `GOMEMLIMIT`.

Disk: Loki stores compressed chunks at roughly 10-20% of raw volume. A host emitting 1 GiB/day
of raw logs lands around 3-6 GiB for the full 30-day window.

## Operations

```bash
# status / health
docker compose ps
docker compose logs -f alloy
docker compose logs -f loki        # Loki is excluded from collection; this is the only way

# apply a config change (Alloy and Loki reload only on restart)
docker compose restart alloy
docker compose up -d               # after editing docker-compose.yml or .env

# dashboards rescan within 30s; datasource changes need a restart
docker compose restart grafana

# what labels does Loki actually have?
docker compose exec -T grafana curl -fsS 'http://loki:3100/loki/api/v1/labels'
docker compose exec -T grafana curl -fsS 'http://loki:3100/loki/api/v1/label/container/values'

# is retention running?
docker compose exec -T grafana curl -fsS http://loki:3100/metrics \
  | grep loki_compactor_apply_retention_last_successful_run_timestamp_seconds

# is Alloy dropping anything?
curl -fsS http://127.0.0.1:12345/metrics | grep -E 'loki_process_dropped_lines_total|loki_write_dropped_entries_total'

# Alloy component graph and live debugging
xdg-open http://127.0.0.1:12345

# upgrade: edit the tags in docker-compose.yml, then
docker compose pull && docker compose up -d

# back up Grafana state (dashboards are in git; this is users, prefs, annotations)
docker run --rm -v logging_grafana-data:/data -v "$PWD:/backup" alpine \
  tar czf /backup/grafana-data.tgz -C /data .

# full reset - DESTROYS ALL STORED LOGS
docker compose down -v
```

## Troubleshooting

**Admin password from `.env` does not work** - `grafana-data` survived a previous install and
Grafana only applies the password when it *creates* the admin user. Either run First-run step 0,
or reset in place:
`docker compose exec -T grafana grafana-cli admin reset-admin-password '<new password>'`.

**"Datasource loki was not found"** - the dashboard's `"uid": "loki"` no longer matches `uid:`
in `datasources/loki.yml`. Check `docker compose logs grafana | grep -i provisioning`.

**Dashboard is empty but Alloy is healthy** - confirm labels exist (`/loki/api/v1/labels`). If
`job` is missing, Alloy is discovering targets but not shipping; check the component graph at
<http://127.0.0.1:12345>.

**Alloy is `unhealthy` but logs are flowing** - expected if some container on the host uses an
unreadable log driver. The probe is `/-/healthy`, which reports *any* unhealthy component. Open
the component graph to see which one.

**A container is missing** - (a) it carries `logging.exclude=true`, (b) it is `logging-alloy` or
`logging-loki` (both excluded by design), (c) it uses a log driver other than
`json-file`/`local`/`journald`, (d) it has no attached network, or (e) it lived for less than one
2s discovery poll. See "Known collection gaps".

**No logs older than 24h after a fresh start** - by design. `stage.drop { older_than = "24h" }`
in `config.alloy` bounds the first-boot backfill. Widen it if you want more history, up to Loki's
`reject_old_samples_max_age` of 720h.

**Compose refuses to start with a message about `GRAFANA_ADMIN_PASSWORD`** - working as intended.
`.env` is missing or the password is empty.

**Loki shows no health status** - expected. See "Why Loki has no healthcheck".

**Panels fail with "max entries limit per query exceeded"** - `jsonData.maxLines` in the
datasource exceeded `limits_config.max_entries_limit_per_query` in `loki.yaml`.

**429s / missing logs from one container** - it hit `per_stream_rate_limit` (4MB) or Alloy's
per-container `stage.limit` (1000 lines/s). Check
`loki_process_dropped_lines_total{reason="rate_limit"}`.
