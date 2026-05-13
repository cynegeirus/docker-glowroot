# Glowroot Central Collector with Cassandra

Production-ready Docker Compose stack running [Glowroot Central Collector](https://github.com/glowroot/glowroot) backed by Apache Cassandra. Designed for Linux hosts, plug-and-play deployment, fully configurable through environment variables.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Service Details](#service-details)
- [Agent Installation](#agent-installation)
- [Day-to-Day Operations](#day-to-day-operations)
- [Backup and Restore](#backup-and-restore)
- [Upgrading](#upgrading)
- [Performance Tuning](#performance-tuning)
- [Security Hardening](#security-hardening)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

## Overview

Glowroot is an open-source Java APM (Application Performance Monitoring) tool. In central mode, multiple Glowroot agents (one per monitored JVM) stream trace data to a Central Collector, which persists everything in Cassandra and exposes a web UI for analysis.

This repository provides:

- A hardened `docker-compose.yaml` with healthchecks, resource limits, log rotation, and proper startup ordering.
- A single `.env` file for all tunable parameters (heap sizes, ports, versions, memory limits).
- Sane defaults that work on a typical 8 GB Linux server out of the box.

## Architecture

```
+-------------------+       +---------------------+       +------------------+
| Java App + Agent  | ----> | Glowroot Central    | ----> |    Cassandra     |
| (port 8181/HTTP2) |       | (UI: 4000)          |       | (internal 9042)  |
+-------------------+       +---------------------+       +------------------+
                                     |
                                     v
                              Web Browser :4000
```

- **Cassandra** stores all collected trace, aggregate, and gauge data. Its CQL port (9042) is not published to the host; only the Glowroot container can reach it through the internal Docker network `glowroot_net`.
- **Glowroot Central** exposes two ports:
  - `4000` — Web UI (HTTP).
  - `8181` — Agent collector endpoint (HTTP/2 over plaintext by default).
- A bridge network isolates inter-container traffic.
- Two named volumes (`cassandra_data`, `glowroot_data`) persist state across container restarts and image upgrades.

## Prerequisites

| Requirement       | Minimum Version                    |
|-------------------|------------------------------------|
| Linux kernel      | 4.x (any modern distribution)      |
| Docker Engine     | 20.10+                             |
| Docker Compose    | v2 (the `docker compose` CLI)      |
| RAM               | 8 GB (with default heap settings)  |
| Disk              | 20 GB free (grows with retention)  |
| CPU               | 2 cores minimum, 4+ recommended    |

Verify:

```bash
docker --version
docker compose version
free -h
df -h .
```

## Quick Start

```bash
# 1. Clone the repository
git clone <your-repo-url>
cd Glowroot

# 2. (Optional) review and customize defaults
nano .env

# 3. Bring the stack up
docker compose up -d

# 4. Watch Cassandra become healthy (~60-90 seconds on first boot)
docker compose ps
docker compose logs -f cassandra

# 5. Once Cassandra is healthy, Glowroot starts automatically.
#    Open the UI:
xdg-open http://localhost:4000     # or just visit it in your browser
```

On first launch Glowroot creates the Cassandra keyspace `glowroot` and schema automatically. No manual database setup is required.

## Configuration

All tunable values live in the `.env` file at the repository root. Compose reads it automatically.

| Variable                  | Default              | Description                                                    |
|---------------------------|----------------------|----------------------------------------------------------------|
| `GLOWROOT_VERSION`        | `0.14.2`             | Image tag for `glowroot/glowroot-central`.                     |
| `GLOWROOT_UI_PORT`        | `4000`               | Host port published for the web UI.                            |
| `GLOWROOT_AGENT_PORT`     | `8181`               | Host port published for the agent (HTTP/2) endpoint.           |
| `CASSANDRA_CLUSTER_NAME`  | `GlowrootCluster`    | Cassandra cluster name (must match across nodes if scaled).    |
| `CASSANDRA_DC`            | `dc1`                | Cassandra datacenter name.                                     |
| `CASSANDRA_RACK`          | `rack1`              | Cassandra rack name.                                           |
| `CASSANDRA_HEAP_NEW`      | `512M`               | Young generation heap size for Cassandra JVM.                  |
| `CASSANDRA_HEAP_MAX`      | `4G`                 | Maximum Java heap for Cassandra.                               |
| `CASSANDRA_MEM_LIMIT`     | `6G`                 | Container-level memory limit for Cassandra.                    |
| `GLOWROOT_HEAP_MIN`       | `2G`                 | `-Xms` for Glowroot JVM.                                       |
| `GLOWROOT_HEAP_MAX`       | `4G`                 | `-Xmx` for Glowroot JVM.                                       |
| `GLOWROOT_MEM_LIMIT`      | `6G`                 | Container-level memory limit for Glowroot.                     |

Rule of thumb: `HEAP_MAX` should be roughly 60-75% of `MEM_LIMIT` to leave room for off-heap memory (direct buffers, metaspace, native libraries, page cache).

After editing `.env`, apply changes:

```bash
docker compose up -d
```

Only containers whose configuration changed will be recreated.

## Service Details

### Cassandra

- **Image**: `cassandra:4.1` (pinned to the 4.1.x LTS line).
- **Persistence**: `cassandra_data` volume mounted at `/var/lib/cassandra`.
- **Networking**: only reachable inside `glowroot_net`. Port 9042 is declared with `expose` (documentation), not `ports` (publish), so the host firewall does not need to block it.
- **Healthcheck**: `nodetool status | grep '^UN'` — passes once the node reports itself as Up/Normal. Glowroot waits for this before starting.
- **Tuning applied**:
  - `MAX_HEAP_SIZE` and `HEAP_NEWSIZE` exported via env so the official entrypoint sets them in `cassandra-env.sh`.
  - `CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch` (recommended for production over the default `SimpleSnitch`).
  - `ulimits.memlock=-1` and `nofile=100000` to avoid swap and file-descriptor exhaustion.
  - `stop_grace_period: 2m` to allow a clean shutdown (flushing memtables).

### Glowroot Central

- **Image**: `glowroot/glowroot-central:${GLOWROOT_VERSION}`.
- **Persistence**: `glowroot_data` volume mounted at `/glowroot-central` (config, admin credentials, TLS material).
- **Networking**: ports `4000` (UI) and `8181` (agent) published to the host.
- **Startup ordering**: `depends_on.cassandra.condition: service_healthy` — Glowroot will not start until Cassandra reports healthy, eliminating a common race condition.
- **JVM flags** (`JAVA_OPTS`):
  - `-XX:+UseContainerSupport` — respects container memory limits.
  - `-XX:+UseG1GC` — low-pause garbage collector suitable for collectors.
  - `-Xms` and `-Xmx` set from `.env`.
  - `-XX:+ExitOnOutOfMemoryError` — fail fast on OOM so Docker restarts the container instead of running degraded.

### Networking and Volumes

```yaml
networks:
  glowroot_net:
    driver: bridge        # standard isolated bridge network

volumes:
  cassandra_data:         # Cassandra SSTables, commit log, system tables
  glowroot_data:          # Glowroot config and local state
```

Both volumes use the default `local` driver, which stores data under `/var/lib/docker/volumes/` on the host. To inspect them:

```bash
docker volume ls | grep glowroot
docker volume inspect glowroot_cassandra_data
```

### Logging

All services use the `json-file` log driver with rotation:

- `max-size: 10m` per log file
- `max-file: 5` rotated files retained

This caps each container's log usage at roughly 50 MB and prevents log files from filling the disk.

View logs:

```bash
docker compose logs -f --tail=200 glowroot
docker compose logs -f --tail=200 cassandra
```

## Agent Installation

On every JVM you want to monitor:

1. Download the Glowroot agent matching the central collector version from the [Glowroot releases page](https://github.com/glowroot/glowroot/releases).
2. Unzip somewhere (for example `/opt/glowroot`).
3. Edit `/opt/glowroot/glowroot.properties`:

   ```properties
   collector.address=http://<central-host>:8181
   agent.id=my-service::production
   ```

   - `collector.address` must be reachable from the application JVM.
   - `agent.id` is a free-form string; convention is `<service-name>::<environment>`.

4. Add the agent to your JVM launch arguments:

   ```bash
   java -javaagent:/opt/glowroot/glowroot.jar -jar your-app.jar
   ```

5. Restart your application. Within a minute it appears in the Glowroot UI's agent dropdown.

For containerized applications, mount the agent directory and pass `-javaagent` through `JAVA_TOOL_OPTIONS` or your image's entrypoint.

## Day-to-Day Operations

```bash
# Service status and health
docker compose ps

# Follow logs
docker compose logs -f glowroot
docker compose logs -f cassandra

# Restart a single service (Glowroot, no data loss)
docker compose restart glowroot

# Stop everything (containers stopped, volumes preserved)
docker compose stop

# Start again
docker compose start

# Remove containers and network (volumes preserved)
docker compose down

# Remove containers AND volumes (DESTROYS ALL DATA)
docker compose down -v

# Exec into a running container
docker compose exec cassandra cqlsh
docker compose exec glowroot sh

# Check Cassandra cluster health
docker compose exec cassandra nodetool status
docker compose exec cassandra nodetool info
```

## Backup and Restore

### Hot backup (snapshot, no downtime)

Cassandra supports atomic snapshots while the database is running:

```bash
# Trigger snapshot
docker compose exec cassandra nodetool snapshot glowroot -t backup-$(date +%F)

# Copy snapshot files to host
docker compose exec cassandra sh -c 'cd /var/lib/cassandra/data/glowroot && tar czf - */snapshots/backup-*' \
  > glowroot-snapshot-$(date +%F).tar.gz

# Clean up snapshot inside the container
docker compose exec cassandra nodetool clearsnapshot glowroot -t backup-$(date +%F)
```

### Cold backup (full volume copy)

Requires stopping the stack:

```bash
docker compose stop
docker run --rm \
  -v glowroot_cassandra_data:/data:ro \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/cassandra-$(date +%F).tar.gz -C /data .
docker compose start
```

### Restore

```bash
docker compose down
docker volume rm glowroot_cassandra_data
docker volume create glowroot_cassandra_data
docker run --rm \
  -v glowroot_cassandra_data:/data \
  -v "$(pwd)":/backup \
  alpine sh -c 'cd /data && tar xzf /backup/cassandra-YYYY-MM-DD.tar.gz'
docker compose up -d
```

## Upgrading

### Glowroot

```bash
# 1. Edit .env, bump GLOWROOT_VERSION (e.g. to 0.14.7)
# 2. Pull the new image
docker compose pull glowroot

# 3. Recreate the container
docker compose up -d glowroot

# 4. Verify
docker compose logs -f glowroot
```

Glowroot performs any required schema migrations on startup. Roll back by reverting `GLOWROOT_VERSION` and recreating.

### Cassandra

Minor patch upgrades inside the same major version (e.g. `4.1.5` → `4.1.9`) are usually safe via `docker compose pull cassandra && docker compose up -d cassandra`. For major version jumps (e.g. 4.1 → 5.0), consult the [official Cassandra upgrade documentation](https://cassandra.apache.org/doc/latest/cassandra/operating/upgrading.html), as SSTable format changes may require `nodetool upgradesstables` and a documented procedure.

## Performance Tuning

### Sizing Cassandra

- For a single-node Glowroot deployment monitoring up to a few dozen JVMs, `4G` heap is sufficient.
- For larger fleets, increase `CASSANDRA_HEAP_MAX` (up to ~16G — beyond that, G1 pauses become a concern; consider multi-node clusters).
- Keep heap size at most 50% of container memory; the rest is used by off-heap structures (bloom filters, key cache, compaction buffers) and Linux page cache for SSTables.

### Sizing Glowroot

- Glowroot's memory footprint scales with the number of connected agents and trace volume.
- For 10–50 agents: 2–4 GB heap is typical.
- For 100+ agents: increase to 8–16 GB and monitor GC behavior via the JMX port (not exposed by default).

### Disk Throughput

Cassandra is I/O sensitive. For production deployments:

- Use SSDs, ideally NVMe.
- Mount the Docker volume root (`/var/lib/docker`) on a fast disk, or change the volume `driver_opts` to point to a dedicated mount.

### Retention

Glowroot data retention is configured in the UI under **Configuration > Storage**. Reducing retention lowers Cassandra disk usage.

## Security Hardening

The default configuration is suitable for a trusted internal network. For internet-facing or sensitive deployments:

1. **Set an admin password** immediately. In the UI: **Configuration > Users > admin > Change password**. Or pre-seed by exec'ing into the container before first browser visit:
   ```bash
   docker compose exec glowroot java -jar glowroot-central.jar setup-admin-user admin <strong-password>
   ```

2. **Enable HTTPS** for the UI. Place `ui-cert.pem` and `ui-key.pem` inside the `glowroot_data` volume and set `ui.https=true` in `glowroot-central.properties`. Alternatively, terminate TLS at a reverse proxy (nginx, Traefik, Caddy) in front of port 4000.

3. **Restrict the agent port (8181)** with host firewall rules so only your application subnets can reach it:
   ```bash
   sudo ufw allow from 10.0.0.0/8 to any port 8181
   sudo ufw deny 8181
   ```

4. **Enable Cassandra authentication** if you ever expose port 9042. By default it is internal-only in this stack; do not change that unless necessary.

5. **Run regular image updates** to pick up security patches:
   ```bash
   docker compose pull
   docker compose up -d
   ```

## Troubleshooting

### Glowroot keeps restarting

```bash
docker compose logs --tail=200 glowroot
```

Common causes:

- **Cassandra not yet healthy.** Wait 60–90 seconds on first boot. Check with `docker compose exec cassandra nodetool status` — you should see a single line beginning with `UN`.
- **OOM (`-XX:+ExitOnOutOfMemoryError` triggered).** Increase `GLOWROOT_HEAP_MAX` in `.env`, then `docker compose up -d glowroot`.
- **Cassandra schema corruption** after improper shutdown. As a last resort, `docker compose down -v` and re-import from a backup.

### Cassandra healthcheck never passes

```bash
docker compose exec cassandra nodetool status
docker compose logs --tail=200 cassandra
```

Look for `Starting listening for CQL clients`. If the node is stuck on bootstrap, check disk space (`df -h`) and available memory (`free -h`).

### Port already in use

```
Error: bind: address already in use
```

Change `GLOWROOT_UI_PORT` or `GLOWROOT_AGENT_PORT` in `.env` and re-run `docker compose up -d`.

### Agent does not appear in the UI

1. Verify connectivity from the agent host: `curl -v http://<central-host>:8181` (you should see an HTTP/2 upgrade response).
2. Check the application log for `Glowroot started` and any collector errors.
3. Confirm `agent.id` is unique — agents sharing an id collapse into a single entry.

### Cassandra runs out of disk

```bash
docker compose exec cassandra df -h /var/lib/cassandra
docker compose exec cassandra nodetool tablestats glowroot
```

Lower data retention in the Glowroot UI, then run `nodetool cleanup` and `nodetool compact glowroot`.

## FAQ

**Can I run this on Windows or macOS?**
Docker Desktop on Windows/macOS works for evaluation, but production deployments should use Linux. WSL2 path mapping and Docker Desktop's resource limits make it a poor fit for sustained APM workloads.

**Can I scale Cassandra to multiple nodes?**
Yes, but not with this single-host compose file. For multi-node clusters, deploy Cassandra on dedicated infrastructure (or Kubernetes via the Cassandra operator) and point `CASSANDRA_CONTACT_POINTS` in the `glowroot` service to that cluster.

**Can I scale Glowroot Central horizontally?**
Yes — multiple Glowroot Central instances can share the same Cassandra cluster. Place them behind a load balancer and point agents at the load balancer hostname. Sessions are stateless on the agent endpoint.

**Where does Glowroot store its configuration?**
In Cassandra, not in the `glowroot_data` volume. The volume only holds the Glowroot startup config, admin password file, and optional TLS material.

**How do I reset the admin password?**
```bash
docker compose exec glowroot java -jar glowroot-central.jar setup-admin-user admin <new-password>
docker compose restart glowroot
```

**Does this work behind a corporate proxy?**
Yes for outbound traffic during `docker compose pull`. Set `HTTP_PROXY` / `HTTPS_PROXY` in your Docker daemon config. The Glowroot agent itself does not need a proxy because it talks directly to the central collector inside your network.
