# PostgreSQL 18 — Adopt Existing Streaming Replication into Patroni

> **Status (20 November 2026, current):** **MIGRATION SUCCESSFULLY ADOPTED ON BOTH NODES.** Patroni `4.1.5` manages PostgreSQL `18` on existing PGDATA `/u01/18/data` without a new base backup or standby rebuild. `patronictl list pg18-ha-lab` now shows `as-pg1` **Leader / running** and `as-pg2` **Replica / streaming**, both on timeline **7**, standby receive/replay `0/8000BB8` and reported lag **0**. Original two-member etcd `3.6.17` cluster remains a lab-only, non-fault-tolerant DCS deployment (loss of either etcd member loses quorum). **Update:** existing slot `as_pg2_slot` was successfully retired from DCS and removed from PostgreSQL; active Patroni-managed slot `as_pg2` remains healthy. **Next:** pre-switchover pg_rewind and credentials preflight, followed by controlled switchover. Passwords and private YAML must not be committed to GitHub.

## Goal

Convert the *existing* PostgreSQL 18 primary/standby streaming replication deployment into a Patroni-managed cluster **without recreating the databases or deleting PGDATA**. Learn etcd quorum, configuration ownership, adoption of running Postgres instances, switchover, failure modes, and rewind/rejoin.

## Lab topology — two VMs only

| Host | IP address | Existing PG role | Planned services |
|---|---|---|---|
| `as-pg1` | `192.168.136.10` | Primary | PostgreSQL 18, Patroni, etcd member 1 |
| `as-pg2` | `192.168.136.11` | Standby | PostgreSQL 18, Patroni, etcd member 2 |

PostgreSQL port `5432`; PGDATA `/u01/18/data` **on both hosts**. No third VM or third etcd member in this exercise.

### Important: two-member etcd quorum

A two-member etcd cluster needs **2 of 2** votes for quorum. Losing either VM or either etcd member means there is no DCS quorum. Patroni will not be able to elect a new PostgreSQL leader; the current leader can also demote after it can no longer maintain the leader lock. Thus:

- **Supported learning exercises:** Patroni adoption, normal cluster management, planned switchover (both VMs alive), PostgreSQL *process* failures while **both etcd services remain running**, rewind/rejoin when possible.
- **Not demonstrated by this topology:** successful unattended failover after **an entire VM or etcd member fails**. To tolerate one etcd member failure, a three-voter DCS quorum is normally required.
- **Do not** force-promote a standby when quorum is lost; investigate and recover quorum to avoid split-brain risks.

Reference: [etcd FAQ — quorum](https://etcd.io/docs/v3.6/faq/).

## Command execution convention — read this first

**Every SQL or `psql` example in this project is run from the Linux `postgres` OS account**, on the indicated server, unless explicitly stated otherwise.

```bash
# Run as root ONLY to switch OS user if you are not already postgres:
su - postgres

# The commands below are then executed at the postgres shell prompt:
/usr/pgsql-18/bin/psql -X -d postgres -c 'SELECT pg_is_in_recovery();'
```

If you are already logged in as `postgres`, **do not** use `runuser` or `sudo`. Commands that modify systemd units, install packages, change firewall settings, or configure etcd at the OS level will be explicitly labeled **RUN AS root**. Patroni and etcd service configuration will be supplied only after preflight is verified. Do not confuse the PostgreSQL SQL superuser role with the Linux `root` user.

## Stage 1 — read-only assessment (completed)

### 1A. Verified configuration, 20 Nov 2026

The following query was **executed as Linux user `postgres`**, on **both servers**:

```sql
SELECT
    pg_is_in_recovery() AS is_standby,
    current_setting('data_directory') AS pgdata,
    current_setting('port') AS port,
    current_setting('wal_log_hints') AS wal_log_hints,
    current_setting('data_checksums') AS data_checksums;
```

| Result | as-pg1 | as-pg2 |
|---|---|---|
| `is_standby` | `f` (primary) | `t` (standby) |
| `pgdata` | `/u01/18/data` | `/u01/18/data` |
| `port` | `5432` | `5432` |
| `wal_log_hints` | `off` | `off` |
| `data_checksums` | `on` | `on` |

**Interpretation:** Checksums are enabled, so turning on `wal_log_hints` is **not** necessary solely to enable `pg_rewind`. Confirm `full_page_writes=on`, WAL availability, permissions, and other prerequisites before planning rewind; checksum status alone does not guarantee successful rewind.

### 1B. Verified streaming replication — primary `as-pg1`, 20 Nov 2026

**RUN AS Linux user `postgres` on `as-pg1`:**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT application_name,
       client_addr,
       state,
       sync_state,
       sent_lsn,
       replay_lsn
FROM pg_stat_replication;"
```

**Observed output:**

| application_name | client_addr | state | sync_state | sent_lsn | replay_lsn |
|---|---|---|---|---|---|
| `walreceiver` | `192.168.136.11` | `streaming` | `async` | `0/50064F8` | `0/50064F8` |

**Interpretation:** The primary sees standby `as-pg2` actively streaming WAL. At this snapshot, sent and replay positions match. However `async` does **not** guarantee zero data loss during a primary crash, and LSN equality at one instant does not guarantee ongoing zero lag.

### 1B.1. Verified WAL receiver — standby `as-pg2`, 20 Nov 2026

**RUN AS Linux user `postgres` on `as-pg2`:**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
    pg_is_in_recovery() AS is_standby,
    status,
    sender_host,
    slot_name,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn
FROM pg_stat_wal_receiver;"
```

**Observed output:**

| is_standby | status | sender_host | slot_name | received_lsn | replayed_lsn |
|---|---|---|---|---|---|
| `t` | `streaming` | `as-pg1` | `as_pg2_slot` | `0/5006600` | `0/5006600` |

**Interpretation:** The standby is connected to `as-pg1` and uses the **existing physical slot `as_pg2_slot`**. At this snapshot, received and replayed LSNs match. Patroni migration must **preserve this slot temporarily**, because Patroni may drop slots it does not recognize. The upstream Patroni migration guide recommends marking pre-existing slots as permanent during adoption, then removing the temporary permanent-slot configuration only after Patroni-managed replacement slots have caught up. Do not drop this slot manually.

Reference: [Patroni — converting an existing cluster](https://patroni.readthedocs.io/en/latest/existing_data.html).

### 1B.2. Verified physical replication slot — primary `as-pg1`, 20 Nov 2026

**RUN AS Linux user `postgres` on `as-pg1`:**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
    slot_name,
    slot_type,
    active,
    active_pid,
    restart_lsn,
    wal_status
FROM pg_replication_slots;"
```

**Observed output:**

| slot_name | slot_type | active | active_pid | restart_lsn | wal_status |
|---|---|---|---|---|---|
| `as_pg2_slot` | `physical` | `t` | `5560` | `0/5006600` | `reserved` |

**Interpretation:** The existing physical slot is active and retaining required WAL, with slot restart LSN `0/5006600`. `active_pid=5560` identifies the process currently using the slot **at the time of the check**; PIDs can change. `reserved` indicates the required WAL is still reserved (it does not guarantee unlimited WAL retention or zero replication lag). We must keep `as_pg2_slot` during Patroni adoption by configuring it temporarily as a permanent physical slot in the cluster's dynamic configuration; do not drop it manually. Patroni will manage its own member slot later. Before removing the original permanent slot definition, compare the replacement Patroni slot's `restart_lsn` and confirm it has caught up, per the [official migration procedure](https://patroni.readthedocs.io/en/latest/existing_data.html).

### 1C. WAL prerequisites — verified on both nodes, 20 Nov 2026

**RUN AS Linux user `postgres` on `as-pg1` AND `as-pg2`:**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT name, setting
FROM pg_settings
WHERE name IN (
    'full_page_writes', 'wal_level',
    'max_wal_senders', 'max_replication_slots'
)
ORDER BY name;"
```

**Actual result (identical on both nodes):**

| name | as-pg1 | as-pg2 |
|---|---|---|
| `full_page_writes` | `on` | `on` |
| `max_replication_slots` | `10` | `10` |
| `max_wal_senders` | `10` | `10` |
| `wal_level` | `replica` | `replica` |

**Interpretation:** PostgreSQL has WAL logging suitable for physical replication and the prerequisites for `pg_rewind` relating to `full_page_writes` and data checksums are met. These results alone do **not** prove that a rewind will succeed: a common ancestor, requisite WAL history, and compatible source/target state are still required. Existing physical replication is asynchronous; do not assume zero data loss in unplanned failover.

### 2A. etcd pre-install discovery — VERIFIED on both VMs, 20 Nov 2026

**RUN AS Linux user `root` on `as-pg1` AND `as-pg2`:**

```bash
hostname
cat /etc/os-release | grep '^PRETTY_NAME='
rpm -q etcd || true
command -v etcd || true
command -v etcdctl || true
ss -lnt | grep -E ':(2379|2380|8008)([[:space:]]|$)' || true
```

**Actual results:** Both servers report **Rocky Linux 10.1 (Red Quartz)**; `rpm -q etcd` says **not installed**; neither `etcd` nor `etcdctl` is on the executable path; ports `2379`, `2380`, and `8008` have no listening sockets in the observed output. This is a snapshot, not a guarantee the ports will remain free.

### 2B. Install official etcd binaries — VERIFIED on both VMs (service not yet started)

**RUN AS Linux user `root` on `as-pg1` AND `as-pg2`.** This sequence assumes the VM reports `x86_64` from `uname -m`; stop and use an appropriate release asset if the architecture differs. Pin the same version on both nodes.

We use **etcd v3.6.17**, a supported, patched v3.6 release published in November 2026. The etcd project recommends pre-built binaries rather than potentially outdated OS package versions.

```bash
# Confirm on each host before continuing:
uname -m  # Expected: x86_64

# Download official amd64 release (only after confirming x86_64):
ETCD_VER=v3.6.17
curl -fL --retry 3 \
  "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz" \
  -o "/tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz"

# Extract into versioned staging folder; do not touch the database files:
mkdir -p "/tmp/etcd-${ETCD_VER}-extract"
tar -xzf "/tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz" \
  -C "/tmp/etcd-${ETCD_VER}-extract"

# Install the server, API CLI, and maintenance CLI:
install -m 0755 \
  "/tmp/etcd-${ETCD_VER}-extract/etcd-${ETCD_VER}-linux-amd64/etcd" \
  "/tmp/etcd-${ETCD_VER}-extract/etcd-${ETCD_VER}-linux-amd64/etcdctl" \
  "/tmp/etcd-${ETCD_VER}-extract/etcd-${ETCD_VER}-linux-amd64/etcdutl" \
  /usr/local/bin/

# Verify installation on both servers:
/usr/local/bin/etcd --version
/usr/local/bin/etcdctl version
/usr/local/bin/etcdutl version
```

**Why these steps:** `curl -fL` stops on HTTP failure and follows redirects; `tar` extracts the official archive without creating a systemd service; `install` places executable binaries in `/usr/local/bin`; version checks prove the binaries run on the host. This stage does **not** start etcd or affect PostgreSQL replication. For hardened deployments verify release provenance/checksums according to your organization's software supply-chain policy before installing.

**Observed results, 20 Nov 2026 (both VMs):**

```text
etcd Version: 3.6.17
Git SHA: 0689586
Go Version: go1.26.11
Go OS/Arch: linux/amd64
etcdctl version: 3.6.17
API version: 3.6
etcdutl version: 3.6.17
API version: 3.6
```

**Success criterion MET:** Matching server/client/utility versions on both nodes. No etcd process or cluster is configured yet. `2379/2380` will be restricted to the private lab network; HTTP without TLS is for this isolated lab only, not production.

References: [etcd v3.6 installation](https://etcd.io/docs/v3.6/install/), [official v3.6.17 release](https://github.com/etcd-io/etcd/releases/tag/v3.6.17), [Nov 2026 patch announcement](https://etcd.io/blog/2026/nov-17-patch-release/).

## Planned stages (commands appended as each stage is verified)

- [x] Record existing PostgreSQL roles, PGDATA, port, checksum and wal_log_hints values.
- [x] Verify primary and standby WAL streaming state and standby slot name.
- [x] Verify primary-side physical slot `as_pg2_slot` active and WAL reserved.
- [x] Verify WAL prerequisites on both nodes (`full_page_writes=on`, `wal_level=replica`, senders=10, slots=10).
- [ ] Confirm a **verified backup** before Patroni adoption (service management and credentials assessed; backup pending).
- [x] Check OS, existing etcd installations and required ports on both VMs — Rocky Linux 10.1; etcd absent; target ports free at check time.
- [x] Install **etcd v3.6.17 binaries** on both hosts and verify versions.
- [x] Prepare **etcd OS service account and directories** on both VMs.
- [x] Configure **two voting etcd members**, one per existing VM; confirm endpoint health/quorum.
- [ ] Install Patroni and create per-node YAML adopting existing PGDATA (not `initdb`).
- [ ] Transfer PostgreSQL service ownership to Patroni in the correct order.
- [ ] Validate roles, WAL replication, configuration/slot management.
- [ ] Test **planned** switchover while both etcd members are healthy.
- [ ] Test PostgreSQL process failure with both etcd members still healthy; record observations.
- [ ] Test `pg_rewind` / safe rejoin of former leader when prerequisites are satisfied.
- [ ] Add observed output, troubleshooting, and rollback checklist.

## Safety rules

- Back up PostgreSQL and verify recovery options before applying changes.
- **Do not** run `initdb`, delete/replace `/u01/18/data`, manually drop a replication slot, remove `standby.signal`, or change PostgreSQL systemd ownership until the migration plan is validated.
- Patroni must be the **only** component controlling the PostgreSQL service after migration; systemd's native PostgreSQL startup service must not fight with Patroni.
- Keep existing streaming replication operational throughout read-only assessment.
- Treat two-member etcd as an explicit lab limitation, not as a safe production HA implementation.

## Authoritative references

- [Patroni — Convert an existing PostgreSQL deployment](https://patroni.readthedocs.io/en/latest/existing_data.html)
- [EDB — Migrating to Patroni](https://www.enterprisedb.com/docs/supported-open-source/patroni/migrating/)
- [etcd — FAQ on cluster quorum](https://etcd.io/docs/v3.6/faq/)
- [PostgreSQL 18 — pg_rewind](https://www.postgresql.org/docs/18/app-pgrewind.html)

### 2C. Create least-privilege etcd user and directories — VERIFIED on both VMs

**RUN AS Linux user `root` on `as-pg1` AND `as-pg2`:**

```bash
# Create a dedicated service account only if it does not already exist.
getent passwd etcd >/dev/null ||   useradd --system --no-create-home --home-dir /var/lib/etcd   --shell /sbin/nologin etcd

# Private directory where this etcd member will persist its Raft state.
install -d -o etcd -g etcd -m 0700 /var/lib/etcd

# Root-managed configuration directory (not writable by etcd).
install -d -o root -g root -m 0755 /etc/etcd

# Verify the account and ownership on each node.
id etcd
ls -ld /var/lib/etcd /etc/etcd
```

**Why:** Running etcd as an unprivileged Linux service account limits access if etcd is compromised. The private `/var/lib/etcd` directory is *etcd's* Raft/WAL database storage and is **not** PostgreSQL's `/u01/18/data` or `pg_wal`. `/etc/etcd` will store the per-node service configuration; this stage creates no config files or active network listeners.

**Actual result (both nodes):** OS account `etcd` exists; on `as-pg2`, UID 995 / GID 994; `/var/lib/etcd` is owned by `etcd:etcd` with `drwx------` permissions, and `/etc/etcd` is owned by `root:root` with `drwxr-xr-x` permissions. User confirmed identical successful setup on `as-pg1`. **Do not start etcd yet.**

**Next:** Create host-specific etcd YAML configurations using names `as-pg1` and `as-pg2` and the respective private IPs; then configure systemd and start members in coordinated fashion. Bootstrap `initial-cluster-state: new` only on initial creation with empty etcd data directories—do not use it to rebuild a live cluster. Two-member etcd needs quorum **2 of 2**; VM failure testing is limited.

Official reference: [etcd 3.6 configuration options](https://etcd.io/docs/v3.6/op-guide/configuration/).


### 2D. Configure two-member etcd cluster — CONFIGURED AND VERIFIED (USER-CONFIRMED)

**RUN AS Linux user `root`.** We are writing `/etc/etcd/etcd.yml` on each node; do not start etcd or change the running PostgreSQL service yet. Reference: [etcd v3.6 official configuration options](https://etcd.io/docs/v3.6/op-guide/configuration/) and [official static-bootstrap instructions](https://etcd.io/docs/v3.6/tasks/operator/how-to-setup-cluster/).

**On `as-pg1` (`192.168.136.10`):**

```bash
cat > /etc/etcd/etcd.yml <<'EOF'
name: as-pg1
data-dir: /var/lib/etcd

listen-peer-urls: http://192.168.136.10:2380
initial-advertise-peer-urls: http://192.168.136.10:2380

listen-client-urls: http://127.0.0.1:2379,http://192.168.136.10:2379
advertise-client-urls: http://192.168.136.10:2379

initial-cluster: as-pg1=http://192.168.136.10:2380,as-pg2=http://192.168.136.11:2380
initial-cluster-state: new
initial-cluster-token: pg18-patroni-lab-20261120
EOF
chmod 0644 /etc/etcd/etcd.yml
```

**On `as-pg2` (`192.168.136.11`):**

```bash
cat > /etc/etcd/etcd.yml <<'EOF'
name: as-pg2
data-dir: /var/lib/etcd

listen-peer-urls: http://192.168.136.11:2380
initial-advertise-peer-urls: http://192.168.136.11:2380

listen-client-urls: http://127.0.0.1:2379,http://192.168.136.11:2379
advertise-client-urls: http://192.168.136.11:2379

initial-cluster: as-pg1=http://192.168.136.10:2380,as-pg2=http://192.168.136.11:2380
initial-cluster-state: new
initial-cluster-token: pg18-patroni-lab-20261120
EOF
chmod 0644 /etc/etcd/etcd.yml
```

Both hosts use **identical** `initial-cluster` and `initial-cluster-token` values. The member-specific `name`, client URLs, and peer URLs **differ**. Ports 2379/2380 are reserved for etcd; port 5432 and PostgreSQL replication remain unchanged. Binding 2379 to loopback and the node's private address enables local admin calls as well as Patroni's cluster member access. These HTTP URLs are acceptable **only in the isolated lab**; production should use peer/client TLS, firewall controls, access controls, and a resilient odd-numbered voting cluster.

**Verify as `root` on each node:**

```bash
hostname
cat /etc/etcd/etcd.yml
ls -l /etc/etcd/etcd.yml
```

**Success criterion:** Host-specific name/IP values are correct, the cluster member list is identical, and the file is readable, with root ownership. No etcd service has been started or cluster initialized.

**Next stage (configuration files confirmed on both nodes):** Define a systemd unit (`User=etcd`, `ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.yml`), check peer network reachability/firewall, and start members. Expect a two-voter cluster to require **both** members running before endpoint health succeeds. Do not restart an initialized member as if it were a fresh cluster or erase `/var/lib/etcd` to fix a quorum problem.


### 2D.1. Configuration verification — VERIFIED, 20 Nov 2026

User confirmed that both `as-pg1` and `as-pg2` have `/etc/etcd/etcd.yml`. The actual output from `as-pg2` shows member name `as-pg2`, address `192.168.136.11`, peer port 2380, client port 2379, and the expected identical `initial-cluster` and cluster token. The configuration is owned by `root:root` with `-rw-r--r--` (0644) permissions. No etcd processes have been started yet.

### 2E. Define systemd unit and verify local networking — READY TO EXECUTE

**RUN AS Linux user `root` on both `as-pg1` and `as-pg2`.** The **same** service file is used on both nodes; member identity and addresses are sourced from their separate `/etc/etcd/etcd.yml` files. Do **not** start or enable etcd in this step.

```bash
cat > /etc/systemd/system/etcd.service <<'EOF'
[Unit]
Description=etcd v3.6 distributed key-value store (PostgreSQL Patroni lab)
Documentation=https://etcd.io/docs/v3.6/
Wants=network-online.target
After=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.yml
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
TimeoutStartSec=120

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemd-analyze verify /etc/systemd/system/etcd.service
systemctl cat etcd
systemctl is-active firewalld || true
getenforce
```

**Why:** `Type=notify` waits for etcd's readiness notification. `User`/`Group` enforce least privilege. `ExecStart` explicitly loads the node-specific YAML. `Restart=on-failure` restarts after an unexpected crash, not on deliberate stop. `network-online.target` requests network availability before startup. `systemd-analyze verify` checks the unit file without starting the service. Check firewalld and SELinux state to understand requirements for opening peer port 2380 and client port 2379 *only on the private lab network*.

**Important:** Two voting members require both running for quorum. Once network access is confirmed, start the two systemd services one after the other; do not treat a temporary no-leader state while only one member is up as corruption. Confirm both with `etcdctl` before configuring Patroni. Keep existing PostgreSQL primary/standby streaming unchanged.

References: [etcd project's official systemd unit](https://github.com/etcd-io/etcd/blob/main/contrib/systemd/etcd.service), [etcd 3.6 configuration](https://etcd.io/docs/v3.6/op-guide/configuration/).

### 2E.1. systemd and host security verification — VERIFIED, 20 Nov 2026

User confirmed the etcd systemd unit is present on **both nodes**, with `Type=notify`, `User=etcd`, `Group=etcd`, and `ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.yml`. `firewalld` is **inactive** and SELinux is **Enforcing** on both hosts. Retain enforcing SELinux; if etcd cannot start, review `journalctl -u etcd` and SELinux AVC audit events before changing any policy. This configuration is a lab only, and neither PostgreSQL service should be stopped while bootstrapping etcd.

### 2F. Start the initial two-member etcd cluster — NEXT STEP

**RUN AS Linux user `root` on both `as-pg1` and `as-pg2`.** Use two terminal sessions. Start the services close together because two voting members require both to form a majority. `--no-block` avoids having the first `systemctl start` command wait for etcd readiness while you still need to start the second member.

```bash
# Run on as-pg1 AND as-pg2 (as root), in separate terminals
systemctl enable etcd
systemctl start etcd --no-block
```

Then, on **each node as root**, inspect status:

```bash
hostname
systemctl is-active etcd
systemctl status etcd --no-pager -l
```

**Expected:** `active (running)` on both after a leader is elected. If either service is `failed`, `activating`, or in a restart loop, collect logs on that node as root:

```bash
journalctl -u etcd -n 50 --no-pager
```

**Do not** reset or delete `/var/lib/etcd` as a generic troubleshooting step; after bootstrap it contains member identities and Raft state. Do not modify PostgreSQL until etcd health and membership are confirmed. Next, check both endpoints and the leader with `etcdctl`.

Official background: https://etcd.io/docs/v3.6/op-guide/configuration/ ; https://etcd.io/docs/v3.6/op-guide/runtime-reconf-design/ ; https://etcd.io/docs/v3.5/tutorials/how-to-check-cluster-status/

### 2F.1. Both etcd services started — VERIFIED, 20 Nov 2026

**Reported by user:** On **both** `as-pg1` and `as-pg2`, the etcd systemd service has been started and is `active`. The captured output from **as-pg2** confirms:

- Unit: `/etc/systemd/system/etcd.service`, enabled.
- Main process: `/usr/local/bin/etcd --config-file=/etc/etcd/etcd.yml` running as the `etcd` service account.
- Cluster version: `3.6` and member ID `d4b8b74ce14b9fc1`.
- Client service listening at `127.0.0.1:2379` and `192.168.136.11:2379`.
- Log warning `serving client traffic insecurely` is expected because this **isolated lab** explicitly uses HTTP rather than TLS; it is not suitable for production.

**Important:** `systemctl is-active etcd` proves the daemon is running, not that the two-member Raft cluster has an elected leader or can commit proposals. Verify both endpoints before starting Patroni.

### 2G. Verify quorum, endpoints, and leader — NEXT STEP

**RUN AS Linux user `root` on `as-pg1`** (one node is enough because the command queries both endpoints):

```bash
ETCD_ENDPOINTS='http://192.168.136.10:2379,http://192.168.136.11:2379'

etcdctl --endpoints="$ETCD_ENDPOINTS" endpoint health --write-out=table
etcdctl --endpoints="$ETCD_ENDPOINTS" endpoint status --write-out=table
etcdctl --endpoints="$ETCD_ENDPOINTS" member list --write-out=table
```

**How to interpret:**

1. `endpoint health`: both endpoints should be **healthy**, meaning they can participate in committing a Raft proposal.
2. `endpoint status`: both endpoints should return without errors, with **exactly one** member showing `IS LEADER = true`, and both at etcd version 3.6.x.
3. `member list`: both expected members (`as-pg1`/`as-pg2`) should be started, with the expected peer/client URLs.

If a check fails, investigate `journalctl -u etcd -n 100 --no-pager` **as `root` on the affected member**. Keep PostgreSQL replication running. Do not remove the etcd data directory or add Patroni before validating quorum.

**Architecture limitation:** With only two etcd voters, losing either member loses quorum. This lab is useful for migration and controlled switchover; it cannot provide resilient automatic failover on a complete VM failure.

Official reference: https://etcd.io/docs/v3.5/tutorials/how-to-check-cluster-status/


### 2G.1. etcd quorum verification — VERIFIED, 20 Nov 2026

**RUN AS Linux user `root` on `as-pg1`.** The two-node cluster was verified with:

```bash
ETCD_ENDPOINTS='http://192.168.136.10:2379,http://192.168.136.11:2379'
etcdctl --endpoints="$ETCD_ENDPOINTS" endpoint health -w table
etcdctl --endpoints="$ETCD_ENDPOINTS" endpoint status -w table
etcdctl --endpoints="$ETCD_ENDPOINTS" member list -w table
```

**Actual results recorded from user output:**

| Node | Endpoint healthy | Member ID | Raft leader? | Raft term | Raft index | Applied index | Member status |
|---|---|---|---|---|---|---|---|
| `as-pg1` | true | `3f21d56d22e727f1` | yes | 2 | 8 | 8 | started |
| `as-pg2` | true | `d4b8b74ce14b9fc1` | no | 2 | 8 | 8 | started |

Both endpoints are healthy; both member records are voting members (`IS LEARNER=false`). Raft leader was `as-pg1` at the time checked. PostgreSQL's write-primary identity is independent of the etcd Raft leader identity. No errors were reported. This confirms **quorum at this instant**, not fault tolerance against VM loss. A two-voter etcd cluster requires both members for further consensus.

**Important migration warning:** Do not delete/replace the PGDATA directory (`/u01/18/data`) on either node. Existing physical replication slot `as_pg2_slot` must be preserved while Patroni takes over; a later configuration step will declare it as permanent to prevent cleanup during the migration. Adopt existing instances rather than initializing replacement clusters. Never operate both the original PostgreSQL systemd service and Patroni as independent process supervisors once the handover begins. Official docs: https://patroni.readthedocs.io/en/latest/existing_data.html and https://www.enterprisedb.com/docs/supported-open-source/patroni/migrating/.

## Stage 3 — Install Patroni, then adopt the existing cluster

### 3A. Install Patroni in an isolated Python virtual environment — NEXT STEP

**RUN AS Linux user `root` on BOTH `as-pg1` and `as-pg2`.** Installing packages or a Python virtualenv does **not** stop or alter the running PostgreSQL instances.

```bash
# Ensure Python, pip, and virtual environment support are available.
dnf install -y python3 python3-pip

# Keep Patroni dependencies isolated from OS Python packages.
mkdir -p /opt/patroni
python3 -m venv /opt/patroni/venv

# Pin a verified Patroni release and include the PostgreSQL driver + etcd v3 client.
/opt/patroni/venv/bin/python -m pip install --upgrade pip
/opt/patroni/venv/bin/python -m pip install 'patroni[psycopg3,etcd3]==4.1.5'

# Verify without starting Patroni.
/opt/patroni/venv/bin/patroni --version
/opt/patroni/venv/bin/patronictl version
```

`psycopg3` provides PostgreSQL connectivity; `etcd3` enables the etcd v3 DCS API matching our etcd 3.6 cluster. Use `etcd3:` (not legacy `etcd:`) when authoring the future Patroni YAML. Both VMs must use the same pinned Patroni release. If `python3 -m venv` is missing, report the error rather than changing live database services.

**DO NOT START PATRONI YET.** Before adoption, we will inventory PostgreSQL service names, existing users/authentication and `primary_conninfo`/`primary_slot_name`, prepare Patroni config, and preserve `as_pg2_slot`. The official migration guide recommends taking over the running primary and standby without reinitializing PGDATA, followed by controlled Patroni restarts in a maintenance window. Reference: https://patroni.readthedocs.io/en/latest/existing_data.html.

### 3A.1. Patroni installation verification — VERIFIED, 20 Nov 2026

**Reported on both nodes** (`as-pg1`, `as-pg2`) after running as Linux user `root`:

```text
patroni 4.1.5
patronictl version 4.1.5
```

Patroni is **installed but not started**. PostgreSQL 18 continues to run in its preexisting streaming replication topology (`as-pg1` primary; `as-pg2` standby). No changes to `/u01/18/data` were made during this step.

### 3B. Inventory native PostgreSQL service management — NEXT STEP (READ-ONLY)

**RUN AS Linux user `root` on BOTH `as-pg1` and `as-pg2`.** Run only these discovery commands; do **not** disable, stop, restart, or enable any database service yet.

```bash
hostname
systemctl list-units --all --type=service | grep -Ei 'postgres|patroni' || true
systemctl list-unit-files --type=service | grep -Ei 'postgres|patroni' || true
ps -eo user,pid,ppid,args | grep -E '[p]ostgres.*-D|[p]ostmaster.*-D' || true
```

**Purpose:** Learn whether native PostgreSQL is managed by `postgresql-18.service`, another systemd unit, or manually; check its systemd enablement; and identify the live postmaster process. Do not assume a service name without evidence. Note that `postgresql-18.service` being *enabled* is a risk during future adoption, because Patroni should be the only process supervisor for PostgreSQL. Per the official Patroni migration procedure, the native unit will be **disabled without stopping the running PostgreSQL process**, once configuration, authentication, and migration readiness are verified. Migrate the primary first, then the standby. Preserve physical slot `as_pg2_slot` by configuring it as a permanent slot during handover; changeover to Patroni-managed slot must be verified before removing the old slot.

Official migration procedure: https://patroni.readthedocs.io/en/latest/existing_data.html


### 3B.1. Native PostgreSQL process/service inventory — VERIFIED, 20 Nov 2026

**RUN AS Linux user `root` on BOTH nodes (read-only).** The following results were reported:

| Node | PostgreSQL process | PID / PPID | PGDATA | Native systemd unit state |
|---|---|---|---|---|
| `as-pg1` | `/usr/pgsql-18/bin/postgres -D /u01/18/data` | `1075 / 1` | `/u01/18/data` | `postgresql-18.service` disabled and not active |
| `as-pg2` | `/usr/pgsql-18/bin/postgres -D /u01/18/data` | `2917 / 1` | `/u01/18/data` | `postgresql-18.service` disabled and not active |

Patroni 4.1.5 is installed on both hosts, but has not been started. The process parent PID of 1 and no active native PostgreSQL systemd unit indicate PostgreSQL is presently operating outside the native systemd service; this alone does not prove the exact command used to start it. **Do not manually start/stop/restart PostgreSQL or enable `postgresql-18.service` during migration.** Since the service is already disabled, there is no need to disable it again. No change to PostgreSQL was made during this inspection.

Official procedure: https://patroni.readthedocs.io/en/latest/existing_data.html

### 3C. Inspect existing PostgreSQL authentication roles — NEXT STEP (READ-ONLY)

**SERVER: `as-pg1` (current primary). RUN AS Linux user: `postgres`.** Run directly in the Linux shell as `postgres`, not as `root`:

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT rolname, rolsuper, rolreplication, rolcanlogin
FROM pg_roles
WHERE rolname IN ('postgres', 'replication', 'replicator', 'rewind', 'patroni')
ORDER BY rolname;"
```

**Why:** The migration documentation requires a PostgreSQL superuser and a PostgreSQL replication role for Patroni. We first inventory existing accounts rather than create redundant users. `pg_roles` does not reveal role passwords. Since database roles are physically replicated, this one query on the primary is enough to check which of these roles exist; node-specific `pg_hba.conf` must be reviewed separately on both nodes.

**Interpretation:** `rolsuper=t` identifies a superuser; `rolreplication=t` identifies a role allowed to initiate physical replication; `rolcanlogin=t` indicates it can authenticate. A separate least-privilege rewind role may be prepared later for `pg_rewind`; the already enabled data checksums and `full_page_writes=on` were confirmed previously. If roles are named differently, inspect the full list in a follow-up rather than guessing credentials.

**Do not paste passwords or any `primary_conninfo` containing passwords into GitHub.** Do not edit `pg_hba.conf`, enable Patroni, or alter the replication slot `as_pg2_slot` until the Patroni configuration is ready.


### 3C.1. Existing PostgreSQL roles — VERIFIED, 20 Nov 2026

**SERVER: `as-pg1` (primary). RUN AS Linux user: `postgres`.** The previous role-discovery SQL returned:

```text
 rolname  | rolsuper | rolreplication | rolcanlogin
----------+----------+----------------+------------
 postgres | t        | t              | t
(1 row)
```

Among the **five explicitly filtered names** (`postgres`, `replication`, `replicator`, `rewind`, `patroni`), only `postgres` exists. It has SUPERUSER, REPLICATION, and LOGIN privileges. **This does not prove there are no other replication-capable database roles with different names.** Nor does it show which role the existing standby connection uses. We will identify that role before choosing Patroni's replication authentication. Using a single powerful role is acceptable for this isolated exercise if authentication matches, but a distinct least-privilege replication role and secure authentication are preferred for production. Do not create roles, rotate passwords, or change a working `pg_hba.conf` at this inspection stage.

### 3D. Verify the active WAL sender's database user — NEXT STEP (READ-ONLY)

**SERVER: `as-pg1` (primary). RUN AS Linux user: `postgres`.**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
    usename,
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;"
```

**Why:** `usename` is the actual PostgreSQL database role used by the WAL sender connection from `as-pg2`, which may not be one of the five names previously checked. `application_name`, `client_addr`, and `state` confirm we are looking at our existing standby connection. This output contains no password. A later read-only step will inspect active host-based authentication rules on **both** nodes before creating Patroni YAML.

Official adoption procedure: https://patroni.readthedocs.io/en/latest/existing_data.html

### 3D.1. Existing standby replication identity — VERIFIED, 20 Nov 2026

**SERVER: `as-pg1` (primary). RUN AS Linux user: `postgres`.** `pg_stat_replication` showed:

```text
 usename   | application_name | client_addr    | state     | sync_state
-----------+------------------+----------------+-----------+------------
 repl_user | walreceiver      | 192.168.136.11 | streaming | async
```

The running standby uses **`repl_user`** to stream from `as-pg1`. The earlier role query returned only `postgres` because it selected five specific names that did *not* include `repl_user`. **Reuse `repl_user` where its privileges and authentication permit** rather than creating a duplicate role during migration. This does not establish that Patroni already has the relevant credentials: we still need to inspect the role and per-node `pg_hba.conf` rules. Keep the existing physical slot `as_pg2_slot` protected during handover. Do not expose passwords, `primary_conninfo` connection strings, or private keys in this public GitHub project.

### 3E. Verify the existing replication role — NEXT STEP (READ-ONLY)

**SERVER: `as-pg1` (primary). RUN AS Linux user: `postgres`** (run in the `postgres` Linux account shell, not as `root`):

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT rolname, rolsuper, rolreplication, rolcanlogin
FROM pg_roles
WHERE rolname IN ('postgres', 'repl_user')
ORDER BY rolname;"
```

**What to check:** The `repl_user` row should exist, have `rolcanlogin = t`, and normally `rolreplication = t` while `rolsuper = f`. `postgres` is the existing superuser. No SQL changes at this stage. **Next**, inspect active `pg_hba_file_rules` on *both* nodes, then decide how Patroni will authenticate to the running instances; do not blindly copy existing trust authentication into a production deployment.

References:
- https://patroni.readthedocs.io/en/latest/existing_data.html
- https://patroni.readthedocs.io/en/latest/yaml_configuration.html

### 3E.1. Existing replication role — VERIFIED, 20 Nov 2026

**SERVER: `as-pg1` (primary). RUN AS Linux user: `postgres`.** The role privilege check returned:

```text
 rolname   | rolsuper | rolreplication | rolcanlogin
-----------+----------+----------------+------------
 postgres  | t        | t              | t
 repl_user | f        | t              | t
```

`repl_user` is an existing, non-superuser PostgreSQL role with REPLICATION and LOGIN privileges. It is also the account shown in `pg_stat_replication` as streaming from `as-pg2`; reusing it is appropriate **provided Patroni authentication and both host authentication configurations are validated first**. `postgres` is the existing superuser. No PostgreSQL roles or passwords were changed. Do **not** paste passwords or full `primary_conninfo` strings into this public project.

### 3F. Inspect PostgreSQL host-based authentication — NEXT STEP (READ-ONLY)

**SERVERS: BOTH `as-pg1` and `as-pg2`. RUN AS Linux user: `postgres`** (execute in the `postgres` Linux account shell).

```bash
hostname
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
    rule_number,
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
ORDER BY line_number;"
```

**Purpose:** Read the current `pg_hba.conf` rules on each local server; PostgreSQL physically replicates database roles, but `pg_hba.conf` is per-node. We need to verify what connections are allowed for `repl_user`, `postgres`, and upcoming Patroni/rewind operations before writing any Patroni YAML. A non-NULL `error` signals an invalid rule; inspect `auth_method` (for example `trust`, `scram-sha-256`, `peer`) and `address`. This view reflects **current file contents**, which may differ from the rules last loaded by PostgreSQL; it is not proof that un-reloaded edits are active. The query does not output passwords. **Do not edit or reload `pg_hba.conf` yet.**

Documentation:
- https://www.postgresql.org/docs/18/view-pg-hba-file-rules.html
- https://patroni.readthedocs.io/en/latest/existing_data.html

### 3F.1. Actual `pg_hba.conf` findings — VERIFIED, 20 Nov 2026

**SERVERS: both `as-pg1` and `as-pg2`. RUN AS Linux user: `postgres`.** The `pg_hba_file_rules` query returned the same eight rules on each node, with no reported parse errors:

| Line | Type | Database | User | Address | Auth method |
|---|---|---|---|---|---|
| 113 | local | all | all | local socket | peer |
| 115 | host | all | all | 127.0.0.1 | scram-sha-256 |
| 117 | host | all | all | ::1 | scram-sha-256 |
| 120 | local | replication | all | local socket | peer |
| 121 | host | replication | all | 127.0.0.1 | scram-sha-256 |
| 122 | host | replication | all | ::1 | scram-sha-256 |
| 125 | host | replication | repl_user | 192.168.136.11 | scram-sha-256 |
| 126 | host | replication | repl_user | 192.168.136.10 | scram-sha-256 |

**Interpretation:** Both nodes have replication connection rules for `repl_user` from both database node IPs, supporting a future role reversal. In the current primary's `pg_stat_replication`, `repl_user` is connected and streaming, so the existing primary-side authentication works in practice. Verify credentials and a live connection in the reverse direction later; rule presence alone doesn't prove a future connection will authenticate. These rules are for PostgreSQL replication connections, **not** general remote SQL access for the `postgres` admin role. The `pg_hba_file_rules` view describes rules on disk, not necessarily the latest rules loaded by an already running backend.

No change was made to PostgreSQL or the authentication file. Preserve the original rules during takeover, and never publish passwords, contents of `.pgpass`, or secret connection strings in GitHub.

### 3G. Locate PostgreSQL configuration and socket paths — NEXT STEP (READ-ONLY)

**SERVERS: both `as-pg1` and `as-pg2`. RUN AS Linux user: `postgres`.**

```bash
hostname
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT name, setting
FROM pg_settings
WHERE name IN (
  'config_file',
  'hba_file',
  'data_directory',
  'unix_socket_directories',
  'listen_addresses'
)
ORDER BY name;"
```

**Purpose:** Before writing `/etc/patroni/patroni.yml`, identify the effective configuration paths and socket directory on *both* instances. Patroni needs the correct `data_dir`, `bin_dir`, PostgreSQL connection addresses, and authentication method. Local UNIX-socket administration can use `peer` when Patroni runs as the matching Linux `postgres` user. Do not start Patroni or rewrite `pg_hba.conf` at this stage.

**Migration caveat:** We must protect the original physical slot `as_pg2_slot` by configuring the existing slot as permanent during adoption, as recommended by the official Patroni 4.1.5 migration guide; then transition to Patroni-managed slots only once their WAL retention positions are safe. The two-voter etcd configuration requires both VMs for quorum and is not a production high-availability topology.

References:
- https://patroni.readthedocs.io/en/latest/existing_data.html
- https://patroni.readthedocs.io/en/latest/yaml_configuration.html
- https://www.postgresql.org/docs/18/view-pg-hba-file-rules.html

### 3G.1. Configuration locations — VERIFIED, 20 Nov 2026

**SERVERS: both `as-pg1` (primary) and `as-pg2` (standby). RUN AS Linux user: `postgres`.** Both `pg_settings` queries returned identical path settings:

| Parameter | Both nodes |
|---|---|
| `config_file` | `/u01/18/data/postgresql.conf` |
| `data_directory` | `/u01/18/data` |
| `hba_file` | `/u01/18/data/pg_hba.conf` |
| `listen_addresses` | `*` |
| `unix_socket_directories` | `/run/postgresql, /tmp` |

**Implications:** The existing PostgreSQL configuration lives inside PGDATA on both nodes. Patroni should be configured to adopt that existing path rather than run `initdb` or create a new standby. Existing `local all all peer` can allow a Patroni service running as Linux `postgres` to perform local PostgreSQL administration via a UNIX socket; this must be verified before relying on it. TCP connections to `127.0.0.1` instead use `scram-sha-256`, not peer authentication. PostgreSQL is already running; do not restart or start Patroni before verifying the full configuration and taking a backup.

Reference: https://patroni.readthedocs.io/en/latest/existing_data.html and https://patroni.readthedocs.io/en/latest/yaml_configuration.html

### 3H. Test local superuser peer authentication through PostgreSQL socket — NEXT STEP (READ-ONLY)

**SERVERS: both `as-pg1` and `as-pg2`. RUN AS Linux user: `postgres`.**

```bash
hostname
/usr/pgsql-18/bin/psql -X -h /run/postgresql -U postgres -d postgres -c "
SELECT current_user, pg_is_in_recovery() AS is_standby;"
```

**Why:** This explicitly uses the UNIX-domain socket under `/run/postgresql` and verifies local access as PostgreSQL superuser `postgres` using the current `peer` rule. Patroni's service will run as the Linux `postgres` account; its configuration will later use `postgresql.use_unix_socket: true`. This tests the local admin access without asking for or printing any passwords. Expected: `current_user=postgres`, `is_standby=f` on `as-pg1` and `t` on `as-pg2`.

**Next checks, not yet executed:** Confirm both nodes' PostgreSQL system identifiers match; check a recoverable backup exists; design Patroni YAML and safe secret injection for `repl_user`; preserve the existing `as_pg2_slot` until Patroni-managed slots catch up; validate Patroni configuration before starting **primary first**. The DCS is intentionally two voting members only, requiring both for quorum. Do not publish credentials in this repository.


### 3H.1. Unix socket / peer-authentication check — VERIFIED, 20 Nov 2026

**SERVERS: both `as-pg1` and `as-pg2`. RUN AS Linux user: `postgres`.** The user ran `psql -h /run/postgresql -U postgres -d postgres` on both nodes. Results:

| Node | `current_user` | `pg_is_in_recovery()` | Meaning |
|---|---|---|---|
| `as-pg1` | `postgres` | `f` | Existing primary, local admin access works |
| `as-pg2` | `postgres` | `t` | Existing standby, local admin access works |

The existing `local all all peer` host authentication permits local administration over the tested UNIX socket when the process runs under Linux `postgres`. This is one prerequisite for configuring Patroni (which will run under Linux `postgres`) without an admin password in the public YAML. This **does not** establish remote TCP authentication. PostgreSQL remains running under existing management; Patroni has not been started.

### 3I. Compare existing PostgreSQL system identifiers — NEXT STEP (READ-ONLY)

**SERVERS: both `as-pg1` and `as-pg2`. RUN AS Linux user: `postgres`.**

```bash
hostname
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
    system_identifier,
    pg_control_version,
    catalog_version_no
FROM pg_control_system();"
```

**Why:** `system_identifier` comes from PostgreSQL cluster initialization and must be identical across a primary and its physical standby. `pg_control_system()` reads this from the cluster control file. Control and catalog versions should also match for this PG18 replication pair. A mismatch means do **not** proceed to Patroni adoption: first determine why the nodes don't belong to the same database system. This is a read-only check; it does not alter `pg_control` or interrupt replication.

**After this check:** confirm a recoverable backup, verify the standby's replication credentials and recovery configuration without exposing secrets, and prepare Patroni YAML plus preservation of the original physical replication slot `as_pg2_slot` following Patroni's existing-data adoption procedure. Only then plan to start Patroni on the existing primary, followed by standby. **Never publish passwords, `.pgpass`, `primary_conninfo` with credentials, or private keys in GitHub.**

Official docs:
- https://www.postgresql.org/docs/18/functions-info.html (`pg_control_system()`)
- https://patroni.readthedocs.io/en/latest/existing_data.html (adopting existing PostgreSQL)

### 3I.1. PostgreSQL system identity — VERIFIED, 20 Nov 2026

**SERVERS:** `as-pg1`, `as-pg2`. **RUN AS Linux user:** `postgres`.

Both existing PostgreSQL instances returned the following identical results:

| Parameter | as-pg1 (primary) | as-pg2 (standby) |
|---|---|---|
| `system_identifier` | `7709811654404845603` | `7709811654404845603` |
| `pg_control_version` | `1800` | `1800` |
| `catalog_version_no` | `202506291` | `202506291` |

**Interpretation:** Both instances are part of the same PostgreSQL physical-replication lineage; their control and catalog versions match. This is a necessary preflight check for adoption by Patroni. It does not replace a recoverable backup or a final replication-health check immediately before changing process management.

### 3J. Check standby replication credential source without printing secrets — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg2` (standby). **RUN AS Linux user:** `postgres`.

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
    current_setting('primary_slot_name') AS primary_slot_name,
    CASE
      WHEN current_setting('primary_conninfo') = ''
        THEN 'primary_conninfo not set'
      WHEN current_setting('primary_conninfo') ~ '(^|[[:space:]])password[[:space:]]*='
        THEN 'password specified in primary_conninfo'
      WHEN current_setting('primary_conninfo') ~ '(^|[[:space:]])passfile[[:space:]]*='
        THEN 'passfile option specified in primary_conninfo'
      ELSE 'no explicit password/passfile option in primary_conninfo'
    END AS credential_hint;"
```

**Purpose:** Determine how to plan credential handling when Patroni begins managing physical replication, while avoiding printing `primary_conninfo` itself, which may contain a plaintext password. This query is a *hint*, not a complete authentication audit: a default `.pgpass`, URI-style DSN, environment variables, or other approaches may also supply a password. The existing slot `as_pg2_slot` must be preserved temporarily as a permanent physical slot in Patroni DCS configuration, per the official conversion guide. Do not publish connection strings, `.pgpass`, authentication secrets, or actual Patroni secret environment files to GitHub. Do not start Patroni yet.

**Before handover:** Ensure a recent restorable PostgreSQL backup exists and that etcd has quorum, as already observed. The existing two-voter etcd cluster requires both etcd members; it is for lab learning, not node-failure HA.

Reference: https://patroni.readthedocs.io/en/latest/existing_data.html


### 3J.1. Existing standby uses a password file — VERIFIED, 20 Nov 2026

**SERVER:** `as-pg2` (standby). **RUN AS Linux user:** `postgres`.

The read-only query reported:

| Setting | Verified value |
|---|---|
| `primary_slot_name` | `as_pg2_slot` |
| `credential_hint` | `passfile specified` |

**Interpretation:** Existing physical streaming replication authenticates as `repl_user` using an explicit PostgreSQL libpq `passfile` option in `primary_conninfo`. This is a client-side credential file, separate from the replication slot, and its contents **must never be printed, uploaded, or committed to GitHub**. `passfile` can supply credentials for a WAL receiver but does not by itself configure Patroni's future replication credentials on both nodes. Patroni will need access to a suitable replication secret when creating or reconfiguring standby connections after migration.

### 3K. Inspect only the current password file path — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg2` (standby). **RUN AS Linux user:** `postgres`.

Use this query to show **only the file path**, never `SHOW primary_conninfo` in terminal output or documentation, because connection strings can contain secrets:

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
    (regexp_match(
        current_setting('primary_conninfo'),
        'passfile[[:space:]]*=[[:space:]]*(''[^'']+''|[^[:space:]]+)'
    ))[1] AS passfile_path;"
```

If a path is returned, verify **only metadata** using `stat -c '%U:%G %a %n' /actual/path` as Linux `postgres`. A libpq password file on Unix must not grant read or write access to group/others (e.g. `0600` is valid); it should belong to a user authorized to read it. Do **not** run `cat`, `grep`, `head`, or paste the file's contents. Do not modify it yet. We will then ensure safe credentials are available to Patroni on both nodes, check for a recoverable backup, and prepare `patroni.yml` following the existing-data migration guide.

**Official references:**
- [PostgreSQL 18 — libpq password file](https://www.postgresql.org/docs/18/libpq-pgpass.html)
- [Patroni — existing data migration](https://patroni.readthedocs.io/en/latest/existing_data.html)
- [Patroni — YAML configuration](https://patroni.readthedocs.io/en/latest/yaml_configuration.html)


### 3K.1. Existing standby password file — VERIFIED, 20 Nov 2026

**SERVER: `as-pg2` only. RUN AS Linux user: `postgres`.**

Observed from a *path-only* extraction of `primary_conninfo` and a file metadata check:

| Check | Observed value |
|---|---|
| `passfile` option | `/var/lib/pgsql/.pgpass` |
| Owner/group | `postgres:postgres` |
| Mode | `600` (`rw-------`) |

An initial command targeting the literal documentation placeholder `/actual/path/to/passfile` failed as expected; replacing the placeholder with the real path succeeded. The file's **contents and password were not displayed**. Current streaming replication is working, but this alone does not prove that Patroni can use those credentials after adopting the cluster.

**Important Patroni nuance:** `postgresql.pgpass` is a *Patroni-managed output path*. Patroni can generate/write this file for libpq connections. Do **not** simply configure `postgresql.pgpass: /var/lib/pgsql/.pgpass` and assume the existing file will remain untouched. For this migration, preserve the original `.pgpass`, and when designing configuration use a *separate* Patroni-managed path (for example `/var/lib/pgsql/patroni.pgpass`) together with a secret-supplied `postgresql.authentication.replication` password. Keep all real secrets off GitHub. See Patroni's [YAML settings](https://patroni.readthedocs.io/en/latest/yaml_configuration.html) and [existing-data migration guide](https://patroni.readthedocs.io/en/latest/existing_data.html).

### 3L. Check password-file presence on the original primary — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg1` (currently primary). **RUN AS Linux user:** `postgres`.

```bash
hostname
if [ -f /var/lib/pgsql/.pgpass ]; then
  stat -c '%U:%G %a %n' /var/lib/pgsql/.pgpass
else
  echo 'No .pgpass file on as-pg1'
fi
```

**Why:** The original primary may need to become a standby after planned switchover; it must then authenticate using the replication role `repl_user`. This check only discovers whether a pre-existing password file exists on the original primary. A missing file is not necessarily a problem because we will configure Patroni-specific replication credentials securely on both nodes. Do not print, copy into GitHub, or overwrite any `.pgpass` file.

**Before changing service management:** Confirm a recoverable PostgreSQL backup and restore plan, define secret handling, prepare and validate both Patroni YAML files, and preserve `as_pg2_slot` as a permanent physical slot during migration until Patroni-managed replacement slots are safe. Both etcd members must remain healthy throughout the migration.


### 3L.1. Primary password file — VERIFIED, 20 Nov 2026

**SERVER:** `as-pg1` (original primary). **RUN AS Linux user:** `postgres`.

The read-only test returned:

```text
as-pg1
No .pgpass file on as-pg1
```

**Interpretation:** This is not an error and does not interrupt current primary-to-standby replication. The primary does not currently act as the streaming client. After a controlled switchover, it would become a standby and need credentials to reach `as-pg2`. We will supply Patroni's replication authentication on **both** nodes without adding any real password to the GitHub repository. Patroni's `postgresql.pgpass` is a generated file path; preserve the existing `as-pg2` `.pgpass` by choosing a distinct Patroni-managed path.

### 3M. Inspect password-file *match fields only* — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg2` (existing standby). **RUN AS Linux user:** `postgres`.

```bash
awk -F: '
  NF >= 5 && $0 !~ /^[[:space:]]*(#|$)/ {
    printf "host=%s port=%s database=%s user=%s password=[REDACTED]\n", $1, $2, $3, $4
  }
' /var/lib/pgsql/.pgpass
```

This displays the host, port, database pattern, and role of each PostgreSQL libpq password-file entry while **not reading out its password field**. It helps determine whether the existing entry is restricted to `as-pg1` or uses a wildcard. It does not verify the password or cover every special escaped-delimiter case. **Do not paste the real `.pgpass` contents or use `cat` on the file.** Do not copy or edit the secret yet.

**Next migration gate:** Confirm a recent recoverable backup of **this** `/u01/18/data` cluster before changing ownership of the PostgreSQL processes; set up secure `repl_user` credentials for Patroni on both nodes; define Patroni scope/DCS settings and temporary permanent preservation of `as_pg2_slot`; validate both configs; adopt primary first, standby second. Two-member etcd remains non-fault-tolerant to a whole-node failure.

References: [PostgreSQL password file](https://www.postgresql.org/docs/18/libpq-pgpass.html), [Patroni existing cluster migration](https://patroni.readthedocs.io/en/latest/existing_data.html), [Patroni YAML configuration](https://patroni.readthedocs.io/en/latest/yaml_configuration.html).

### 3M.1. Existing `.pgpass` matching fields — VERIFIED, 20 Nov 2026

**SERVER: `as-pg2`. RUN AS Linux user: `postgres`.**

The masked inspection of `/var/lib/pgsql/.pgpass` showed exactly the following host/user pattern (password not displayed):

```text
host=as-pg1 port=5432 database=replication user=repl_user password=[REDACTED]
```

**Interpretation:** The existing `as-pg2` WAL receiver can authenticate to `as-pg1`, but this entry alone is not sufficient for the reverse direction after a switchover. There is no pre-existing `.pgpass` on `as-pg1`. During migration we will reuse `repl_user` (`LOGIN REPLICATION`, not superuser), arrange its credentials securely on **both nodes**, and put Patroni's generated password file at a separate path (for example `/var/lib/pgsql/patroni.pgpass`). Do not overwrite the existing `/var/lib/pgsql/.pgpass`, or commit actual credentials into GitHub. Official Patroni docs: [existing-data adoption](https://patroni.readthedocs.io/en/latest/existing_data.html), [configuration](https://patroni.readthedocs.io/en/latest/yaml_configuration.html).

### 3N. Check backup readiness and disk capacity — NEXT STEP (READ-ONLY)

**SERVER: `as-pg1` (current primary). RUN AS Linux user: `root`.**

```bash
hostname
du -sh /u01/18/data
df -h /u01
command -v pgbackrest || echo 'pgBackRest not found in PATH'
ls -l /usr/pgsql-18/bin/pg_basebackup
```

**Why:** Before Patroni starts controlling an existing data directory, confirm that this specific PostgreSQL 18 cluster has a **recent, tested, recoverable backup** (not merely an older backup of another lab). The commands above check database footprint, available storage, and installed backup tools; **they do not themselves prove a backup exists**. If no backup has been verified, plan and take one to separate storage before the handover. Do not rely on an in-VM backup alone to recover from VM/disk loss. No PostgreSQL changes or Patroni start should occur during this step.

**After 3N:** Select/verify a usable backup and recovery location, then prepare protected Patroni replication credentials and YAML files; preserve `as_pg2_slot` as a temporary permanent physical slot when adopting the existing primary, as required by the [Patroni conversion guidance](https://patroni.readthedocs.io/en/latest/existing_data.html).

### 3N.1. Primary backup readiness — VERIFIED, 20 Nov 2026

**SERVER:** `as-pg1` (primary). **RUN AS Linux user:** `postgres` (as shown in the actual terminal output; the preceding discovery commands also work as root).

Observed results:

| Check | Result |
|---|---|
| PGDATA size | `132M` at `/u01/18/data` |
| Filesystem | `/dev/sda4`, 49G total, 4.3G used, 45G available |
| pgBackRest | `/usr/bin/pgbackrest` installed; stanza/repository **not yet verified** |
| PostgreSQL backup client | `/usr/pgsql-18/bin/pg_basebackup` installed |
| Recoverable backup | **Not yet created or verified** for this migration |

**Decision:** Use the PostgreSQL 18 `pg_basebackup` and `pg_verifybackup` utilities for a fresh migration safety copy. Never run `pg_basebackup` with `-D` inside the live PGDATA. A backup on the same VM only protects against migration mistakes; maintain an off-VM backup for machine/storage failure scenarios. We will verify the backup and ideally perform a test restore before Patroni adoption.

### 3O. Check for user-defined tablespaces — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg1` (current primary). **RUN AS Linux user:** `postgres`.

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT spcname, pg_tablespace_location(oid) AS location
FROM pg_tablespace
ORDER BY spcname;"
```

**Why:** Before backing up a cluster onto the **same host**, check for externally located tablespaces. With `pg_basebackup -F plain`, PostgreSQL will otherwise put tablespace data in the **original absolute paths** unless `--tablespace-mapping` is provided; that can conflict with and endanger the running primary. Do **not** perform a plain-format same-host backup without reviewing tablespaces and safe path mapping. Tar-format backups avoid this path collision and can be verified with `pg_verifybackup -F tar -n` (WAL parsing is unavailable for tar-format verification). If only the built-in `pg_default` and `pg_global` appear with empty paths, a plain-format full backup to a *separate non-PGDATA directory* is straightforward and permits WAL verification with `pg_verifybackup`.

Official PostgreSQL 18 references: [pg_basebackup](https://www.postgresql.org/docs/18/app-pgbasebackup.html), [pg_verifybackup](https://www.postgresql.org/docs/18/app-pgverifybackup.html).

**Do not stop PostgreSQL, start Patroni, modify replication slots, or alter authentication while completing backup safeguards.**


### 3O.1. Tablespaces — VERIFIED, 20 Nov 2026

**SERVER:** `as-pg1`. **RUN AS Linux user:** `postgres`.

Observed result:

```text
spcname    | location
-----------+---------
pg_default |
pg_global  |
(2 rows)
```

**Interpretation:** No custom/external tablespace locations were found. A plain-format same-server physical base backup may be stored under a separate directory outside `/u01/18/data` without tablespace mappings. The destination must not overlap the live PGDATA, and it must not already contain files. Official reference: [PostgreSQL 18 pg_basebackup](https://www.postgresql.org/docs/18/app-pgbasebackup.html).

### 3P. Create pre-migration physical base backup — NEXT STEP

**SERVER:** `as-pg1` (existing primary). **RUN AS Linux user:** `postgres`.

This command uses the verified local UNIX socket and `peer` auth, avoiding any credential prompts. The backup is **not** written inside PGDATA. No PostgreSQL shutdown is necessary.

```bash
mkdir -p /var/lib/pgsql/backups
chmod 700 /var/lib/pgsql/backups
BACKUP_DIR=/var/lib/pgsql/backups/patroni-pre-migration-20261120

if [ -e "$BACKUP_DIR" ]; then
  echo "Backup destination already exists; choose a new name."
else
  /usr/pgsql-18/bin/pg_basebackup \
    --host=/run/postgresql \
    --username=postgres \
    --pgdata="$BACKUP_DIR" \
    --format=plain \
    --wal-method=stream \
    --checkpoint=fast \
    --manifest-checksums=SHA256 \
    --progress \
    --verbose
fi
```

- `--wal-method=stream`: includes WAL needed for consistent recovery, using an additional replication connection; PostgreSQL 18 can use a temporary slot for the backup stream.
- `--manifest-checksums=SHA256`: records file checksums for `pg_verifybackup`.
- `--checkpoint=fast`: asks the primary to complete the checkpoint sooner; it can increase short-term I/O, so choose carefully outside a lab.
- `--host=/run/postgresql --username=postgres`: use the local socket with existing `peer` auth and replication privileges.
- `--pgdata`: **backup output directory**, not the running data directory `/u01/18/data`.

**Next, after the backup reports success:** Verify it with the *PostgreSQL 18* binary, for example:

```bash
# RUN AS Linux postgres, on as-pg1 (after successful backup)
PATH=/usr/pgsql-18/bin:$PATH /usr/pgsql-18/bin/pg_verifybackup \
  /var/lib/pgsql/backups/patroni-pre-migration-20261120
```

Do not infer backup health merely from the existence of a directory. Review `pg_basebackup` exit status and `pg_verifybackup` output, then ideally perform a test restore before allowing Patroni to control the instances. A same-VM backup does not protect from loss of that VM or volume; keep an independent copy for real disaster recovery. Official reference: [PostgreSQL 18 pg_verifybackup](https://www.postgresql.org/docs/18/app-pgverifybackup.html).

**Safety gate:** No Patroni start, replication-slot changes, or PostgreSQL service restart until the backup is verified and Patroni configuration/adoption plan is reviewed.

### 3P.1. Pre-migration base backup — COMPLETED, 20 Nov 2026

**SERVER:** `as-pg1` (primary). **RUN AS Linux user:** `postgres`.

The PostgreSQL 18 `pg_basebackup` command above completed successfully:

```text
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/7000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_7694"
36049/36049 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/7000120
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
```

**Interpretation:** A plain-format full base backup was taken using local socket authentication. The required WAL was streamed and the `backup_manifest` was created. The backup's temporary slot `pg_basebackup_7694` is separate from the standby's permanent-in-use slot `as_pg2_slot`; a successful backup normally removes its temporary slot. **The backup has not yet been integrity-verified, nor test-restored.**

### 3Q. Verify the base backup — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg1`. **RUN AS Linux user:** `postgres`.

```bash
/usr/pgsql-18/bin/pg_verifybackup --progress \
  /var/lib/pgsql/backups/patroni-pre-migration-20261120
```

The successful final line should be `backup successfully verified`. The verifier uses `backup_manifest` to check expected files and their checksums and, for this **plain-format** backup, parses the WAL required to recover it. Use the version 18 binary, matching the backup. Verification detects many integrity issues but is **not a substitute for a test restore**. If the verification fails, stop the migration and investigate; do not modify the backup files or live PGDATA to bypass the error.

**Do not start Patroni, change slots, or restart PostgreSQL until the backup passes and the adoption plan is ready.**

Reference: [PostgreSQL 18 pg_verifybackup](https://www.postgresql.org/docs/18/app-pgverifybackup.html).

### 3Q.1. Physical backup verification — PASSED, 20 Nov 2026

**SERVER:** `as-pg1`. **RUN AS Linux user:** `postgres`.

Command executed:

```bash
/usr/pgsql-18/bin/pg_verifybackup --progress \
  /var/lib/pgsql/backups/patroni-pre-migration-20261120
```

Observed output:

```text
35536/35536 kB (100%) verified
backup successfully verified
```

**Result:** The PostgreSQL 18 plain-format backup passed manifest, data-file checksum, and required WAL verification. It has **not** been restored in an isolated test. It resides on the same VM, so it is a migration safeguard, *not* an independent disaster-recovery copy. Continue protecting live PGDATA and the active `as_pg2_slot`.

### 3R. Preserve original configuration on both nodes — COMPLETED, 20 Nov 2026

**SERVERS:** `as-pg1` AND `as-pg2`. **RUN AS Linux user:** `postgres`.

Patroni can adopt a running instance, but its configuration management may rename the original `postgresql.conf` to `postgresql.base.conf` and regenerate `postgresql.conf`. Preserve both nodes' *original* local configuration and any standby-specific state before the handover. This is a file-copy operation; it does not restart or reload PostgreSQL.

```bash
hostname
DEST=/var/lib/pgsql/backups/patroni-config-pre-migration-20261120
mkdir -p "$DEST" && chmod 700 "$DEST"

for f in postgresql.conf postgresql.auto.conf pg_hba.conf pg_ident.conf standby.signal; do
  if [ -f "/u01/18/data/$f" ]; then
    cp -p "/u01/18/data/$f" "$DEST/"
  fi
done

ls -ld "$DEST"
ls -l "$DEST"
```

Expected: `postgresql.conf`, `postgresql.auto.conf` (when present), and `pg_hba.conf` on both nodes; `standby.signal` on `as-pg2` only. There is no need to create a missing file. Configs are sensitive: do **not** commit these backup copies to GitHub or print their contents. We will create sanitized templates in the GitHub project later.

**After 3R:** Prepare node-specific Patroni YAML files using existing PGDATA `/u01/18/data`, the local Unix socket for administrator connections, etcd3 at the two known endpoints, and dedicated Patroni-managed replication credentials. Configure the pre-existing `as_pg2_slot` as a temporary **permanent** physical slot through Patroni `bootstrap.dcs.slots` before starting Patroni on the primary. Validate with `patroni --validate-config` before handing over PostgreSQL management. Keep `postgresql-18.service` disabled and the existing databases running until the adoption sequence is ready. See [official Patroni migration guide](https://patroni.readthedocs.io/en/latest/existing_data.html) and [Patroni configuration management](https://patroni.readthedocs.io/en/latest/patroni_configuration.html).


### 3R.1. Configuration backup results — COMPLETED, 20 Nov 2026

**SERVERS:** `as-pg1` and `as-pg2`. **RUN AS Linux user:** `postgres`.

The user completed the configuration backup command on both nodes. The primary (`as-pg1`) listing was:

```text
-rw-------. 1 postgres postgres  5677 Nov 19 10:01 pg_hba.conf
-rw-------. 1 postgres postgres  2681 Nov 19 09:26 pg_ident.conf
-rw-------. 1 postgres postgres    88 Nov 19 09:26 postgresql.auto.conf
-rw-------. 1 postgres postgres 32690 Nov 19 09:43 postgresql.conf
```

The preserved files reside in `/var/lib/pgsql/backups/patroni-config-pre-migration-20261120` on **each respective VM**; the standby's backup should also include `standby.signal` if present (the primary does not have it). These are sensitive local backups, **not** GitHub artifacts. Neither PostgreSQL instance was stopped or restarted in this step.

### 3S. Generate a Patroni draft from the existing primary — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg1` **only**. **RUN AS Linux user:** `postgres`.

Patroni 4.1.5 provides `--generate-config` to read the settings of a *running* PostgreSQL instance and produce a starting YAML file. This is a **draft**, not a configuration to launch yet: it must be reviewed for the existing replication slot, `etcd3` endpoints, local authentication, security, and PostgreSQL parameters.

```bash
umask 077
mkdir -p /var/lib/pgsql/patroni-migration
/opt/patroni/venv/bin/patroni \
  --generate-config \
  --dsn "host=/run/postgresql user=postgres dbname=postgres" \
  /var/lib/pgsql/patroni-migration/generated.yml
ls -l /var/lib/pgsql/patroni-migration/generated.yml
```

The connection uses the already-verified peer-authenticated PostgreSQL Unix socket. The generated YAML may contain sensitive configuration: **do not paste the entire file into chat or commit it to GitHub**. To view only known nonsecret fields, use:

```bash
grep -E '^(scope:|name:|  (listen:|connect_address:|data_dir:|bin_dir:))' \
  /var/lib/pgsql/patroni-migration/generated.yml
```

**Stop after generating and reviewing this draft.** Do not start Patroni, modify `pg_hba.conf`, rotate credentials, or restart native PostgreSQL yet. The migration plan is primary first, then standby; before starting Patroni, configure `bootstrap.dcs.slots.as_pg2_slot` as a temporary permanent physical slot, provide secret replication credentials on both VMs, and validate the node-specific configurations. See:
- [Patroni 4.1.5 - Patroni configuration generation and validation](https://patroni.readthedocs.io/en/latest/patroni_configuration.html)
- [Patroni - Converting an existing Postgres cluster](https://patroni.readthedocs.io/en/latest/existing_data.html)


### 3S.1. Patroni-generated primary draft — VERIFIED, 20 Nov 2026

**SERVER:** `as-pg1` **RUN AS Linux user:** `postgres`.

The running PostgreSQL 18 primary was inspected using Patroni 4.1.5 `--generate-config`; the output was written to `/var/lib/pgsql/patroni-migration/generated.yml` with permission `600` (owned by `postgres`). Selected observed fields:

```text
scope: '#FIXME'
name: as-pg1
restapi.listen: 127.0.1.1:8008
restapi.connect_address: 127.0.1.1:8008
postgresql.listen: '*:5432'
postgresql.connect_address: 127.0.1.1:5432
postgresql.bin_dir: /usr/pgsql-18/bin
postgresql.data_dir: /u01/18/data
```

The generated loopback URLs are **not reachable by the peer VM**, and `scope` is a placeholder. **Do not start Patroni using `generated.yml`.** Preserve it as an original local baseline, not a GitHub artifact.

### 3T. Create a separate primary Patroni candidate — NEXT STEP (OFFLINE CONFIG ONLY)

**SERVER:** `as-pg1` **RUN AS Linux user:** `postgres`.

The following script changes only the *new candidate* YAML; it does not touch PostgreSQL, etcd, or the original generated configuration. It uses PyYAML already provided by the Patroni virtualenv. If the candidate already exists, it refuses to overwrite it. It also protects the existing physical standby slot through Patroni's DCS bootstrap config.

```bash
/opt/patroni/venv/bin/python - <<'PYCODE'
import os
from pathlib import Path
import yaml

source = Path('/var/lib/pgsql/patroni-migration/generated.yml')
target = Path('/var/lib/pgsql/patroni-migration/patroni-as-pg1.yml')
cfg = yaml.safe_load(source.read_text())

cfg['scope'] = 'pg18-ha-lab'
cfg['name'] = 'as-pg1'
cfg['restapi'].update({
    'listen': '192.168.136.10:8008',
    'connect_address': '192.168.136.10:8008',
})
cfg['postgresql'].update({
    'connect_address': '192.168.136.10:5432',
    'use_unix_socket': True,
})
cfg.pop('etcd', None)
cfg['etcd3'] = {'hosts': ['192.168.136.10:2379', '192.168.136.11:2379']}
dcs = cfg.setdefault('bootstrap', {}).setdefault('dcs', {})
dcs.setdefault('postgresql', {})['use_slots'] = True
dcs.setdefault('slots', {})['as_pg2_slot'] = {'type': 'physical'}

fd = os.open(str(target), os.O_CREAT | os.O_EXCL | os.O_WRONLY, 0o600)
with os.fdopen(fd, 'w') as file:
    yaml.safe_dump(cfg, file, sort_keys=False)

print('Candidate created:', target)
print('Scope:', cfg['scope'])
print('REST API:', cfg['restapi']['connect_address'])
print('PostgreSQL:', cfg['postgresql']['connect_address'])
print('DCS:', ', '.join(cfg['etcd3']['hosts']))
print('Protected physical slot:', ', '.join(dcs['slots']))
PYCODE
```

**Stop here; do not start Patroni.** The next tasks are to review the candidate for any remaining `#FIXME` markers and sensitive/generated authentication sections, provide `repl_user` credentials securely on both servers, and validate the finished node-specific configurations. Do not commit the real YAML to GitHub if it contains credentials. `bootstrap.dcs.slots` is the correct path for *permanent slots*; `bootstrap.dcs.postgresql.use_slots` enables Patroni-managed slots.

Official references: [Patroni 4.1.5 config generation and validation](https://patroni.readthedocs.io/en/latest/patroni_configuration.html), [migrate an existing PostgreSQL cluster](https://patroni.readthedocs.io/en/latest/existing_data.html), and [dynamic slots configuration](https://patroni.readthedocs.io/en/latest/dynamic_configuration.html).


### 3T.1. Primary Patroni candidate created — USER CONFIRMED, 20 Nov 2026

**SERVER:** `as-pg1`. **RUN AS Linux user:** `postgres`.

The user confirmed completion of the `3T` Python command creating the private candidate `/var/lib/pgsql/patroni-migration/patroni-as-pg1.yml`. The specific resulting values have **not** yet been independently inspected; they are the intended settings from the previous script. Original `generated.yml` remains unchanged. PostgreSQL remains running outside Patroni management.

### 3U. Audit Patroni candidate without disclosing passwords — NEXT STEP (READ-ONLY)

**SERVER:** `as-pg1` **only**. **RUN AS Linux user:** `postgres`.

Inspect the actual candidate without printing raw YAML or any password values. The audit prints address/paths, authentication *usernames and whether a password field is present*, number of `pg_hba` entries, special parameter override names, the slot plan, and the YAML paths (not contents) of any remaining `FIXME` placeholders.

```bash
/opt/patroni/venv/bin/python - <<'PYCODE'
from pathlib import Path
import yaml

path = Path('/var/lib/pgsql/patroni-migration/patroni-as-pg1.yml')
cfg = yaml.safe_load(path.read_text())
pg = cfg.get('postgresql') or {}
rest = cfg.get('restapi') or {}
auth = pg.get('authentication') or {}
dcs = (cfg.get('bootstrap') or {}).get('dcs') or {}

print('scope:', cfg.get('scope'))
print('name:', cfg.get('name'))
print('restapi.listen:', rest.get('listen'))
print('restapi.connect_address:', rest.get('connect_address'))
print('postgresql.listen:', pg.get('listen'))
print('postgresql.connect_address:', pg.get('connect_address'))
print('postgresql.data_dir:', pg.get('data_dir'))
print('postgresql.bin_dir:', pg.get('bin_dir'))
print('postgresql.use_unix_socket:', pg.get('use_unix_socket'))
print('postgresql.pgpass:', pg.get('pgpass', 'not set'))
print('etcd3.hosts:', ', '.join((cfg.get('etcd3') or {}).get('hosts') or []))
print('bootstrap.dcs.postgresql.use_slots:', (dcs.get('postgresql') or {}).get('use_slots'))
print('bootstrap.dcs.slots:', ', '.join(sorted((dcs.get('slots') or {}).keys())))
print('postgresql.pg_hba rule count:', len(pg.get('pg_hba') or []))
for role in ('superuser', 'replication', 'rewind'):
    value = auth.get(role) or {}
    print(f'authentication.{role}: username={value.get("username", "MISSING")}, '
          f'password_field={"present" if value.get("password") else "missing"}')
print('postgresql.parameters special keys:', ', '.join(
    k for k in ('config_file', 'hba_file', 'ident_file', 'primary_conninfo',
                'primary_slot_name', 'unix_socket_directories')
    if k in (pg.get('parameters') or {})) or 'none')

def find_fixme(value, loc='config'):
    if isinstance(value, dict):
        for key, item in value.items():
            find_fixme(item, loc + '.' + str(key))
    elif isinstance(value, list):
        for i, item in enumerate(value):
            find_fixme(item, f'{loc}[{i}]')
    elif isinstance(value, str) and 'FIXME' in value.upper():
        print('Unresolved placeholder at:', loc)
find_fixme(cfg)
PYCODE
```

**Do not paste the full private YAML**, a password, or the contents of `.pgpass` into chat or GitHub. If authentication information is missing or a placeholder exists, resolve it in a later controlled step. No Patroni service start, PostgreSQL restart, slot deletion, or DCS mutation should occur during this audit.

**Additional migration safeguards:** The existing slot `as_pg2_slot` must remain protected as a temporary permanent slot while the standby is adopted. Since the existing `.pgpass` entry only matches `as-pg1`, Patroni will need a replication password for `repl_user` on both nodes before switchover. Keep native `postgresql-18.service` disabled. The two-voter etcd cluster requires both members and cannot withstand a complete VM failure.

References: [Patroni config generation/validation](https://patroni.readthedocs.io/en/latest/patroni_configuration.html), [existing cluster adoption](https://patroni.readthedocs.io/en/latest/existing_data.html), [YAML settings](https://patroni.readthedocs.io/en/latest/yaml_configuration.html).


### 3U.1. Candidate audit — VERIFIED, 20 Nov 2026

**SERVER:** `as-pg1`. **RUN AS Linux user:** `postgres`.

User-supplied result of safe YAML audit:

```text
Cluster: pg18-ha-lab
Node: as-pg1
REST API: 192.168.136.10:8008
PostgreSQL: 192.168.136.10:5432
PGDATA: /u01/18/data
Unix socket: True
etcd3: ['192.168.136.10:2379', '192.168.136.11:2379']
Replication slots enabled: True
Preserved slots: ['as_pg2_slot']
Patroni pgpass: not set
superuser username: postgres password: configured
replication username: #FIXME password: configured
rewind username: MISSING password: not configured
pg_hba rules: 8
PostgreSQL special parameter keys: ['config_file', 'hba_file', 'ident_file']
Unresolved placeholder: config.postgresql.authentication.replication.password
Unresolved placeholder: config.postgresql.authentication.replication.username
```

**Important:** The audit's `password: configured` means a nonempty YAML value, **not** a valid secret: `replication.password` is still the placeholder `#FIXME`. **Do not run `patroni --validate-config` or start Patroni using this unfinished file.** We must set `replication.username` to `repl_user`, load its real SCRAM password securely, add a dedicated Patroni-managed `postgresql.pgpass` location, validate settings, and prepare the `as-pg2` candidate.

### 3V. Securely transfer the existing replication credential source — NEXT STEP

**SOURCE SERVER:** `as-pg2`. **RUN AS Linux user:** `postgres`.

The working standby already has `/var/lib/pgsql/.pgpass` with a record matching `as-pg1:5432:replication:repl_user`, owned by `postgres` with permissions `0600`. To avoid typing, printing, or pasting the password, copy **only over SSH** into the private working directory on `as-pg1`:

```bash
scp -p /var/lib/pgsql/.pgpass \
  postgres@as-pg1:/var/lib/pgsql/patroni-migration/source-repl.pgpass
```

**DESTINATION SERVER:** `as-pg1`. **RUN AS Linux user:** `postgres`.

```bash
stat -c '%U:%G %a %n' \
  /var/lib/pgsql/patroni-migration/source-repl.pgpass
```

Expected owner: `postgres:postgres`; mode: `600`. If the SSH copy fails, report just the connection/authentication error and stop—don't paste the credential contents, use insecure transfer, change PostgreSQL passwords, or change existing `.pgpass` files. The source file will later be parsed locally to populate the **private** Patroni configuration, then this temporary copy should be securely deleted. **Never commit a private Patroni YAML or `.pgpass` file to GitHub.** The original `as-pg2` `.pgpass` must remain untouched during migration. After the two-node setup is verified, consider credentials and auth hardening separately for production.

The copy alone does **not** make reverse replication work; Patroni must receive the credential on both nodes and generate its own password file for connections to whichever primary is active. Continue only after reviewing `stat` output. PostgreSQL and etcd are not modified by this step.

Official sources: [Patroni existing cluster migration](https://patroni.readthedocs.io/en/latest/existing_data.html), [YAML configuration and `postgresql.pgpass`](https://patroni.readthedocs.io/en/latest/yaml_configuration.html).

### 3V.1 — Transfer confirmed (20 Nov 2026)

**On `as-pg1`, RUN AS Linux user `postgres`.** The user verified:

```text
postgres:postgres 600 /var/lib/pgsql/patroni-migration/source-repl.pgpass
```

The original `.pgpass` on `as-pg2` remains untouched. This copy is sensitive; never commit it to GitHub.

### 3W — Populate Patroni replication authentication from the existing password file (NEXT STEP)

**SERVER:** `as-pg1` only. **RUN AS Linux user:** `postgres`.

The current private draft `/var/lib/pgsql/patroni-migration/patroni-as-pg1.yml` still has `#FIXME` replication username and password. Read the matching `as-pg1:5432:replication:repl_user` record from the safely copied `.pgpass` file and write a **new**, owner-only Patroni config without printing a secret. Do not touch the original YAML draft.

```bash
/opt/patroni/venv/bin/python - <<'PY'
import os
import stat
from pathlib import Path
import yaml

base = Path('/var/lib/pgsql/patroni-migration')
source = base / 'source-repl.pgpass'
draft = base / 'patroni-as-pg1.yml'
target = base / 'patroni-as-pg1-private.yml'

if stat.S_IMODE(base.stat().st_mode) & 0o077:
    raise SystemExit('STOP: migration directory must not be accessible by group/others')
if stat.S_IMODE(source.stat().st_mode) != 0o600:
    raise SystemExit('STOP: source-repl.pgpass must be mode 600')

# Read only the record already confirmed to work for streaming replication.
matching = []
for line in source.read_text().splitlines():
    if not line or line.lstrip().startswith('#'):
        continue
    fields = line.split(':', 4)
    if len(fields) == 5 and fields[:4] == ['as-pg1', '5432', 'replication', 'repl_user']:
        matching.append(fields[4])
if len(matching) != 1 or not matching[0]:
    raise SystemExit('STOP: expected exactly one matching replication password entry')

# PostgreSQL .pgpass escapes backslashes and colons with a backslash.
encoded = matching[0]
chars = []
i = 0
while i < len(encoded):
    if encoded[i] == '\\' and i + 1 < len(encoded):
        i += 1
    chars.append(encoded[i])
    i += 1
secret = ''.join(chars)

cfg = yaml.safe_load(draft.read_text())
rep = cfg.setdefault('postgresql', {}).setdefault('authentication', {}).setdefault('replication', {})
rep['username'] = 'repl_user'
rep['password'] = secret
cfg['postgresql']['pgpass'] = str(base / 'patroni.pgpass')

# O_EXCL ensures we do not overwrite a previously prepared config.
fd = os.open(target, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
with os.fdopen(fd, 'w') as f:
    yaml.safe_dump(cfg, f, sort_keys=False)
print('Created private candidate:', target)
print('Replication username:', rep['username'])
print('Replication password: set (not displayed)')
print('Patroni-managed pgpass location:', cfg['postgresql']['pgpass'])
PY

stat -c '%U:%G %a %n' /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml
```

**Expected:** file owner `postgres:postgres`, mode `600`; username `repl_user`; password set without being printed. This creates only a *candidate*. The separate `patroni.pgpass` path is for Patroni to manage when needed; do not create or modify it manually at this stage.

**Security and operations:** Do not share private YAML or `.pgpass` contents. Do not run `patroni --validate-config --print` because `--print` may expose secrets. Later validate the candidate with `patroni --validate-config -i <file>` (without `--print`); `-i` ignores the PostgreSQL listen port currently occupied by the running instance. The other node still needs its own config; do not start Patroni, restart PostgreSQL, or change DCS yet. This two-member etcd cluster still cannot tolerate a complete VM failure.

Patroni documentation: [PostgreSQL authentication and pgpass settings](https://patroni.readthedocs.io/en/latest/yaml_configuration.html), [configuration validation](https://patroni.readthedocs.io/en/latest/patroni_configuration.html), [existing-cluster adoption](https://patroni.readthedocs.io/en/latest/existing_data.html).

---

### Step 3X — Validate private Patroni primary configuration (2026-11-20)

**Observed result (as-pg1):** The private configuration `/var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml` exists and is owned by `postgres:postgres` with permissions `600`. The password was intentionally not displayed or committed. This confirms file creation and permissions only; it does **not** establish that the Patroni configuration is valid.

**Server:** `as-pg1`  
**Linux user:** `postgres`  
**Impact:** validation only. Do not start Patroni or restart PostgreSQL.

```bash
/opt/patroni/venv/bin/patroni --validate-config \
  /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  --ignore-listen-port

echo "Patroni validator exit code: $?"
```

Use `--ignore-listen-port` because PostgreSQL is already running on port `5432` during the adoption migration. Do **not** use `--print` / `-p` because the private configuration contains credentials; any diagnostics shared externally should be reviewed and redacted first. A successful validation is a prerequisite, not proof of safe adoption. The standby needs a separately prepared configuration. Preserve the original PGDATA, `repl_user`, and `as_pg2_slot`; do not change etcd membership or PostgreSQL services yet.

Official references: [Patroni configuration validation](https://patroni.readthedocs.io/en/latest/patroni_configuration.html#validate-patroni-configuration) and [adopting an existing PostgreSQL cluster](https://patroni.readthedocs.io/en/latest/existing_data.html).

---

### Step 3X.1 — Resolve Patroni `nosync` / `sync_priority` validation conflict (2026-11-20)

**Observed on `as-pg1`:**

```text
tags  Multiple of ('nosync', 'sync_priority') provided
Validator exit code: 1
```

The candidate private YAML contains two mutually exclusive ways to configure a node's synchronous-standby eligibility. The existing PostgreSQL setup is **asynchronous**. Remove `sync_priority` from the private candidate, keeping `nosync` unchanged; re-run validation. This only edits an offline candidate YAML, not PostgreSQL, etcd or the original generated YAML.

**Server:** `as-pg1`  
**Linux user:** `postgres`  
**Security:** Do not print, paste, or commit the private YAML: it contains credentials.

```bash
/opt/patroni/venv/bin/python - <<'PY'
import os
import shutil
from pathlib import Path
import yaml

path = Path('/var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml')
backup = path.with_name(path.name + '.before-tags-fix')

cfg = yaml.safe_load(path.read_text())
tags = cfg.get('tags', {})
if not isinstance(tags, dict) or not {'nosync', 'sync_priority'}.issubset(tags):
    raise SystemExit('STOP: expected both nosync and sync_priority in tags; inspect keys only')
if backup.exists():
    raise SystemExit('STOP: private backup already exists; do not overwrite')

shutil.copy2(path, backup)
os.chmod(backup, 0o600)
tags.pop('sync_priority')

new_file = path.with_name(path.name + '.tmp')
fd = os.open(new_file, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
with os.fdopen(fd, 'w') as f:
    yaml.safe_dump(cfg, f, sort_keys=False)
os.replace(new_file, path)
print('Removed tags.sync_priority; retained tags.nosync')
print('Private config mode:', oct(path.stat().st_mode & 0o777))
PY

/opt/patroni/venv/bin/patroni --validate-config \
  /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  --ignore-listen-port

echo "Validator exit code: $?"
```

**Expected:** validator exit code `0`; if it reports another issue, stop and diagnose it before proceeding. Do not start Patroni yet. **No passwords are displayed.**

Reference: [Patroni YAML configuration, node tags](https://patroni.readthedocs.io/en/latest/yaml_configuration.html).

---

### Step 3X.2 — Validated as-pg1 Patroni candidate (2026-11-20)

**Observed on `as-pg1` as Linux user `postgres`:**

```text
Validator exit code: 0
```

The private configuration `/var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml` passed Patroni 4.1.5 configuration validation using `--ignore-listen-port` after removing the conflicting `tags.sync_priority` key. **This verifies only the configuration validator, not successful database adoption.** Do not start Patroni or restart PostgreSQL yet. Keep the original cluster and verified physical backup intact.

### Step 3Y — Generate Patroni configuration on the existing as-pg2 standby

**Server:** `as-pg2` (`192.168.136.11`)  
**Linux user:** `postgres`  
**Impact:** read-only inspection of PostgreSQL and creation of a private YAML draft. No restart, failover, change to etcd or change to PostgreSQL.

```bash
umask 077
mkdir -p /var/lib/pgsql/patroni-migration
chmod 700 /var/lib/pgsql/patroni-migration

/opt/patroni/venv/bin/patroni --generate-config \
  --dsn "host=/run/postgresql user=postgres dbname=postgres" \
  /var/lib/pgsql/patroni-migration/generated-as-pg2.yml

stat -c '%U:%G %a %n' \
  /var/lib/pgsql/patroni-migration/generated-as-pg2.yml

grep -E '^(scope:|name:|  (listen:|connect_address:|data_dir:|bin_dir:))' \
  /var/lib/pgsql/patroni-migration/generated-as-pg2.yml
```

**Expected:** `postgres:postgres` mode `600`, `name: as-pg2`, and `data_dir: /u01/18/data`. The generator will likely propose loopback addresses and `#FIXME` placeholders: these are **not** ready to run. Do not share the full draft as it can contain authentication details. Review the output and then prepare the actual standby candidate with its unique identity and IP address, with replication authentication and safe slot preservation. Avoid copying the primary's full generated PostgreSQL configuration blindly because each node can have node-specific settings.

**Operational note:** The official Patroni adoption procedure starts Patroni on the primary first, then each standby, once the complete configurations are validated. Patroni manages the existing running PostgreSQL instances; it is not yet permitted to start any new instance. The existing two-member etcd cluster requires both voting members for quorum and cannot survive loss of one VM.

References: [Patroni 4.1.5 configuration generation](https://patroni.readthedocs.io/en/latest/patroni_configuration.html#patroni-configuration-for-a-running-instance), [migration of existing PostgreSQL](https://patroni.readthedocs.io/en/latest/existing_data.html).


---

### Step 3Y.1 — Verify generated standby file on `as-pg2` (2026-11-20)

**Observed** — `postgres` user on `as-pg2`:

```text
postgres:postgres 600 /var/lib/pgsql/patroni-migration/generated-as-pg2.yml
scope: '#FIXME'
name: as-pg2
  connect_address: 127.0.1.1:8008
  listen: 127.0.1.1:8008
  bin_dir: /usr/pgsql-18/bin
  connect_address: 127.0.1.1:5432
  data_dir: /u01/18/data
  listen: '*:5432'
```

The file is private and the PGDATA / node name are correct. The loopback connect addresses and cluster placeholder **must be replaced** before using the draft. Keep PostgreSQL running. Patroni 4.1.5 can adopt existing instances, but a valid YAML does not guarantee a successful adoption.

### Step 3Z — Prepare private standby candidate on `as-pg2` (pending execution)

**Server:** `as-pg2` (`192.168.136.11`)  
**RUN AS:** Linux `postgres`  
**Impact:** creates a new private YAML file; does not alter PostgreSQL, etcd, or the generated source YAML.

Use the existing `as-pg2` `.pgpass` entry (`as-pg1:5432:replication:repl_user`) to supply the same working replication password. **Never print or publish the password.** The script avoids overwriting an existing file and requires a private directory and password file. Keep `bootstrap.dcs.slots` preservation configured in the primary YAML for the initial DCS bootstrap.

```bash
/opt/patroni/venv/bin/python - <<'PYCODE'
import os
import stat
from pathlib import Path
import yaml

base = Path('/var/lib/pgsql/patroni-migration')
source = base / 'generated-as-pg2.yml'
secret_file = Path('/var/lib/pgsql/.pgpass')
target = base / 'patroni-as-pg2-private.yml'

if stat.S_IMODE(base.stat().st_mode) & 0o077:
    raise SystemExit('STOP: migration directory must be private')
if stat.S_IMODE(secret_file.stat().st_mode) != 0o600:
    raise SystemExit('STOP: .pgpass must have mode 600')

matching = []
for line in secret_file.read_text().splitlines():
    if not line.strip() or line.lstrip().startswith('#'):
        continue
    parts = line.split(':', 4)
    if len(parts) == 5 and parts[:4] == ['as-pg1', '5432', 'replication', 'repl_user']:
        matching.append(parts[4])
if len(matching) != 1 or not matching[0]:
    raise SystemExit('STOP: expected exactly one repl_user .pgpass entry')

raw = matching[0]
password_chars = []
i = 0
while i < len(raw):
    if raw[i] == '\\' and i + 1 < len(raw):
        i += 1
    password_chars.append(raw[i])
    i += 1

cfg = yaml.safe_load(source.read_text())
cfg['scope'] = 'pg18-ha-lab'
cfg['name'] = 'as-pg2'
cfg['restapi'].update({
    'listen': '192.168.136.11:8008',
    'connect_address': '192.168.136.11:8008'
})
pg = cfg['postgresql']
pg['connect_address'] = '192.168.136.11:5432'
pg['use_unix_socket'] = True
pg['authentication']['replication'] = {
    'username': 'repl_user',
    'password': ''.join(password_chars)
}
pg['pgpass'] = str(base / 'patroni.pgpass')
cfg.pop('etcd', None)
cfg['etcd3'] = {'hosts': ['192.168.136.10:2379', '192.168.136.11:2379']}

# The generator may populate both mutually exclusive Patroni node tags.
tags = cfg.get('tags') or {}
if 'nosync' in tags and 'sync_priority' in tags:
    tags.pop('sync_priority')

fd = os.open(target, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
with os.fdopen(fd, 'w') as f:
    yaml.safe_dump(cfg, f, sort_keys=False)

print('Created:', target)
print('Cluster:', cfg['scope'])
print('Node:', cfg['name'])
print('REST API:', cfg['restapi']['connect_address'])
print('PostgreSQL:', pg['connect_address'])
print('Replication user:', pg['authentication']['replication']['username'])
print('Replication password: set (hidden)')
print('etcd3 endpoints:', len(cfg['etcd3']['hosts']))
PYCODE

stat -c '%U:%G %a %n' \
  /var/lib/pgsql/patroni-migration/patroni-as-pg2-private.yml
```

**Expected:** `postgres:postgres 600`; safe summary shows `pg18-ha-lab`, `as-pg2`, the `.11` IP and `repl_user`. This remains a candidate until it passes `patroni --validate-config --ignore-listen-port` and the cross-node configuration is reviewed. Do **not** run Patroni or restart Postgres yet. Do not upload the generated YAML, private YAML, or `.pgpass` to GitHub.

Reference: [Patroni 4.1.5 generated configuration](https://patroni.readthedocs.io/en/latest/patroni_configuration.html#patroni-configuration-for-a-running-instance), [official migration guidance](https://patroni.readthedocs.io/en/latest/existing_data.html).

### Step 3AA — Standby Patroni candidate validation (COMPLETE, 2026-11-20)

**Server:** `as-pg2`  
**RUN AS:** Linux `postgres`

```bash
/opt/patroni/venv/bin/patroni --validate-config \
  /var/lib/pgsql/patroni-migration/patroni-as-pg2-private.yml \
  --ignore-listen-port
printf 'Validator exit code: %s\n' "$?"
```

**Observed:** `Validator exit code: 0`. Both `as-pg1` and `as-pg2` configuration candidates now pass Patroni 4.1.5 validation. This checks configuration validity only, **not** safe adoption, runtime credentials, DCS membership, or PostgreSQL health. Neither Patroni daemon has been started. Existing PostgreSQL primary/standby remain running. The backups and original configs are preserved.

### Step 3AB — Cross-check both Patroni candidates without exposing secrets (COMPLETE, 2026-11-20)

**Servers:** `as-pg1` and `as-pg2`  
**RUN AS:** Linux `postgres`  
**Impact:** Read-only. Do not print private YAML, passwords or `.pgpass` contents.

```bash
/opt/patroni/venv/bin/python - <<'PY'
import socket
from pathlib import Path
import yaml

node = socket.gethostname().split('.')[0]
ips = {'as-pg1': '192.168.136.10', 'as-pg2': '192.168.136.11'}
if node not in ips:
    raise SystemExit('STOP: unexpected hostname')

path = Path(f'/var/lib/pgsql/patroni-migration/patroni-{node}-private.yml')
cfg = yaml.safe_load(path.read_text())
pg = cfg.get('postgresql') or {}
repl = (pg.get('authentication') or {}).get('replication') or {}
dcs = (cfg.get('bootstrap') or {}).get('dcs') or {}
checks = {
    'cluster scope': cfg.get('scope') == 'pg18-ha-lab',
    'member name': cfg.get('name') == node,
    'REST listen': (cfg.get('restapi') or {}).get('listen') == f'{ips[node]}:8008',
    'REST connect': (cfg.get('restapi') or {}).get('connect_address') == f'{ips[node]}:8008',
    'Postgres connect': pg.get('connect_address') == f'{ips[node]}:5432',
    'existing PGDATA': pg.get('data_dir') == '/u01/18/data',
    'Postgres bin_dir': pg.get('bin_dir') == '/usr/pgsql-18/bin',
    'etcd3 endpoints': set((cfg.get('etcd3') or {}).get('hosts') or []) == {
        '192.168.136.10:2379', '192.168.136.11:2379'},
    'replication user': repl.get('username') == 'repl_user',
    'replication secret set': isinstance(repl.get('password'), str) and
        bool(repl.get('password')) and 'FIXME' not in repl.get('password', '').upper(),
    'Patroni-managed pgpass': pg.get('pgpass') == '/var/lib/pgsql/patroni-migration/patroni.pgpass',
}
if node == 'as-pg1':
    checks['preserve original slot'] = (
        (dcs.get('postgresql') or {}).get('use_slots') is True
        and (dcs.get('slots') or {}).get('as_pg2_slot', {}).get('type') == 'physical'
    )
for label, passed in checks.items():
    print(f'{label}: {"PASS" if passed else "FAIL"}')
print('Summary:', 'ALL PASS' if all(checks.values()) else 'INVESTIGATE FAILURES')
PY
```

Verify that every check passes on **both** machines. If the primary's slot preservation check fails, **do not start Patroni**: it could remove the original slot used by the standby. The permanent slot is a temporary migration safeguard; later inspect Patroni's replacement slot and retire the original only after WAL retention/LSN checks.

**After this audit**, verify that the DCS namespace for `pg18-ha-lab` does not already contain cluster state, review authentication and `pg_hba`, and then hand over process control in the order prescribed by the [Patroni 4.1.5 migration guide](https://patroni.readthedocs.io/en/latest/existing_data.html): primary first, standby second. A two-voter etcd cluster loses quorum on failure of either VM; it is not a production-ready automatic failover topology.


### Step 3AB — Observed results (2026-11-20)

**Servers:** `as-pg1` and `as-pg2` · **RUN AS:** `postgres` · **Impact:** Read-only.

The user reported `RESULT: ALL PASS` for both nodes. Output confirmed the `pg18-ha-lab` scope, node identity, each correct `.10` / `.11` address, `/u01/18/data` PGDATA, `repl_user` with a non-placeholder secret, and both etcd3 endpoints. The script specifically checks permanent preservation of `as_pg2_slot` on **as-pg1 only**; confirm `Original slot protected: PASS` appeared on that primary before starting Patroni.

### Step 3AC — Verify etcd has no existing Patroni DCS state (NEXT)

**Server:** `as-pg1` · **RUN AS:** Linux `root` · **Impact:** Read-only. Do not start Patroni or modify etcd keys yet.

Patroni 4.1.5 defaults to DCS namespace `/service`, combined with `scope: pg18-ha-lab`, giving `/service/pg18-ha-lab/`. If a custom `namespace` has been set in the private YAML, use that namespace rather than the default in this check.

```bash
ETCD_ENDPOINTS="http://192.168.136.10:2379,http://192.168.136.11:2379"

etcdctl --endpoints="$ETCD_ENDPOINTS" \
  get /service/pg18-ha-lab/ \
  --prefix --keys-only
```

**Expected:** no keys returned. If keys appear, STOP and inspect them; do not delete DCS data automatically. A successful empty result supports a clean adoption of the existing primary first, then its running standby, under the official [Patroni existing-data migration guide](https://patroni.readthedocs.io/en/latest/existing_data.html). Configuration validation does not prove runtime readiness. This remains a lab-only two-voter etcd topology: loss of either VM removes quorum, preventing safe automatic failover.


### Step 3AC — Existing Patroni DCS prefix empty (COMPLETE, 2026-11-20)

**Server:** `as-pg1` · **RUN AS:** Linux `root` · **Impact:** Read-only.

```bash
ETCD_ENDPOINTS="http://192.168.136.10:2379,http://192.168.136.11:2379"
etcdctl --endpoints="$ETCD_ENDPOINTS" get /service/pg18-ha-lab/ --prefix --keys-only
```

**Observed:** no output; no keys found at the default Patroni namespace/scope prefix. This is the desired pre-migration state **provided that** `namespace` is unset or `/service` in both Patroni YAML files. Do not create, edit, or delete etcd keys manually.

### Step 3AD — Confirm namespace and preserved original slot (NEXT)

**Servers:** `as-pg1` and `as-pg2` · **RUN AS:** Linux `postgres` · **Impact:** Read-only; does not print credentials.

```bash
/opt/patroni/venv/bin/python - <<'PYTHON'
from pathlib import Path
import socket
import yaml
node = socket.gethostname().split('.')[0]
if node not in {'as-pg1', 'as-pg2'}:
    raise SystemExit('STOP: unexpected node')
p = Path(f'/var/lib/pgsql/patroni-migration/patroni-{node}-private.yml')
cfg = yaml.safe_load(p.read_text())
dcs = ((cfg.get('bootstrap') or {}).get('dcs') or {})
namespace = cfg.get('namespace', '/service')
slots = dcs.get('slots') or {}
print('Node:', node)
print('Namespace:', namespace)
print('Scope:', cfg.get('scope'))
print('Existing slot preserved:', slots.get('as_pg2_slot') == {'type': 'physical'})
print('DCS use_slots:', (dcs.get('postgresql') or {}).get('use_slots'))
print('Local postgresql.use_slots:', (cfg.get('postgresql') or {}).get('use_slots', 'not set'))
PYTHON
```

**Required on primary:** `Namespace` = `/service` (or `/service/`), `Existing slot preserved` = `True`, `DCS use_slots` = `True`, and `Scope` = `pg18-ha-lab`. On standby the `Existing slot preserved` output may be `False` because the slot is declared in the *initial DCS configuration on the primary*. This is expected. If the namespace differs, verify the actual prefix in etcd before starting Patroni.

**Next:** verify service ownership/startup safety, then start Patroni primary-first under a dedicated systemd unit. Official source: [Patroni migration for existing PostgreSQL data](https://patroni.readthedocs.io/en/latest/existing_data.html). A two-member etcd cluster has no quorum if one VM fails; do not claim VM-failure automatic failover.


### Step 3AD — Final DCS/slot check completed (2026-11-20)

**Server:** `as-pg1` · **RUN AS:** `postgres` · **Impact:** Read-only.

Observed:

```text
Cluster: pg18-ha-lab
Namespace: /service
Replication slots enabled: True
Original slot preserved: True
```

Together with prior successful Patroni config validators (`exit code: 0` on both nodes), these results establish that the initial Patroni DCS configuration is prepared to preserve `as_pg2_slot`. Startup safety and runtime adoption still require validation.

### Step 3AE — Prepare Patroni systemd service on the existing primary (NEXT)

**Server:** `as-pg1` only · **RUN AS:** Linux `root` · **Impact:** Writes a new systemd unit, **does not start or enable** Patroni and does not restart PostgreSQL.

Create `/etc/systemd/system/patroni.service` using the repository's [`config/patroni-as-pg1.service`](config/patroni-as-pg1.service) as the exact file contents, or use:

```bash
cat > /etc/systemd/system/patroni.service <<'EOF'
[Unit]
Description=Patroni PostgreSQL 18 HA Manager
Wants=network-online.target etcd.service
After=network-online.target etcd.service

[Service]
Type=notify
User=postgres
Group=postgres
ExecStart=/opt/patroni/venv/bin/patroni /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=5
TimeoutStartSec=120
TimeoutStopSec=120
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
```

Verify (still **RUN AS root**, `as-pg1`):

```bash
systemd-analyze verify /etc/systemd/system/patroni.service
systemctl cat patroni
systemctl is-enabled patroni || true
systemctl is-active patroni || true
```

**Expected:** No errors from `systemd-analyze verify`, unit displays correct private YAML path, and the Patroni service is not yet active. **STOP before `systemctl start patroni`;** confirm the service first. Original PostgreSQL 18 processes must remain running. `KillMode=process` prevents systemd from directly killing Postgres children, but Patroni itself controls PostgreSQL shutdown/startup after it adopts the instance.

**Safety:** This private YAML contains credentials, so do not commit it to GitHub. The repository's `.service` template includes only a path, not any secret. Prior to service start also verify etcd quorum and existing PostgreSQL replication health. The two-member etcd quorum cannot tolerate a lost node, so this is a lab-only HA learning exercise.

**References:** [Patroni 4.1.5 existing-data migration](https://patroni.readthedocs.io/en/latest/existing_data.html), [official Patroni sample systemd service](https://github.com/patroni/patroni/blob/master/extras/startup-scripts/patroni.service).


### Step 3AE — Primary Patroni unit verified (COMPLETE, 2026-11-20)

**Server:** `as-pg1` · **RUN AS:** Linux `root`.

Observed `systemd-analyze verify /etc/systemd/system/patroni.service` returned no errors. `systemctl cat patroni` showed `User=postgres`, `Group=postgres`, `Type=notify`, `KillMode=process`, and:

```ini
ExecStart=/opt/patroni/venv/bin/patroni /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml
```

`systemctl is-enabled patroni` returned `disabled`, and `systemctl is-active patroni` returned `inactive`. This is the expected pre-adoption state. Original PGDATA is unchanged; `postgresql-18.service` was already disabled and inactive.

### Step 3AF — Start Patroni on the existing primary (NEXT: live adoption)

Patroni's official [existing-data migration guide](https://patroni.readthedocs.io/en/latest/existing_data.html) says to start on the primary first. It detects running PostgreSQL and begins monitoring it. This does **not** imply a new initdb or standby rebuild. **Do this only in the lab after confirming both etcd endpoints are healthy and as-pg1 remains primary.** No production-style automatic VM failover exists with two-member etcd.

**1. RUN AS Linux `root` — `as-pg1` only (etcd preflight):**

```bash
ETCD_ENDPOINTS='http://192.168.136.10:2379,http://192.168.136.11:2379'
etcdctl --endpoints="$ETCD_ENDPOINTS" endpoint health -w table
```

Stop if either endpoint reports unhealthy.

**2. RUN AS Linux `postgres` — `as-pg1` only (PostgreSQL preflight):**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "SELECT pg_is_in_recovery() AS is_standby; SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```

Stop if `pg_is_in_recovery()` is `t` or the standby connection is not `streaming`.

**3. RUN AS Linux `root` — `as-pg1` only (start primary Patroni):**

```bash
systemctl enable patroni
systemctl start patroni
systemctl is-active patroni
systemctl status patroni --no-pager -l
```

**4. RUN AS Linux `postgres` — `as-pg1` only (read cluster membership):**

```bash
/opt/patroni/venv/bin/patronictl -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml list pg18-ha-lab
/usr/pgsql-18/bin/psql -X -d postgres -c 'SELECT pg_is_in_recovery();'
```

Expected: `as-pg1` is the Patroni **Leader**, PostgreSQL remains writable (`pg_is_in_recovery() = f`). `as-pg2` may not yet appear in `patronictl list` because its Patroni daemon has **not** been started, even though it is still streaming natively. Check the existing slot `as_pg2_slot` remains present after startup, and troubleshoot failures with `journalctl -u patroni -n 80 --no-pager`; **never delete PGDATA, stop the healthy primary blindly, or wipe etcd keys as a quick fix.**

**Afterwards:** create the standby Patroni unit pointing to its own private YAML; start on `as-pg2` only after primary health and slot preservation are confirmed. Later restart the standby under Patroni and schedule primary restart during a planned lab window. Keep private YAML and credential files out of GitHub.

### Step 3AF — Incident: Patroni first start timed out (UNDER INVESTIGATION, 2026-11-20)

**Server:** `as-pg1` · **RUN AS:** Linux `root` · **Status:** `systemctl start patroni` reported `Job for patroni.service failed because a timeout was exceeded.` This is a service-start timeout, not proof that PostgreSQL stopped or that the existing PGDATA was damaged. **Do not repeatedly start Patroni, reset etcd, delete PGDATA, or manually restart PostgreSQL before checking its state.** Patroni may already have modified the DCS/managed the primary during the attempted handover.

**Leading hypothesis to verify:** The systemd unit uses `Type=notify`, but Patroni was installed with the `psycopg3,etcd3` extras, **without** the optional `systemd` extra (`systemd-python`) needed for `sd_notify()` integration. Patroni 4.1.x documents this requirement, and a missing module has previously caused startup timeouts with `Type=notify`. Other causes remain possible: inspect logs and live database before applying any fix.

Read-only diagnostic commands (**RUN AS root on `as-pg1`**):

```bash
systemctl status patroni --no-pager -l
journalctl -u patroni -n 80 --no-pager -o short-iso
/opt/patroni/venv/bin/python - <<'PY'
try:
    import systemd.daemon
    print('systemd-python: INSTALLED')
except ImportError:
    print('systemd-python: MISSING')
PY
runuser -u postgres -- /usr/pgsql-18/bin/pg_isready -h /run/postgresql -p 5432
```

If PostgreSQL is still reachable, also inspect role with:

```bash
runuser -u postgres -- /usr/pgsql-18/bin/psql -X -d postgres -Atc 'SELECT pg_is_in_recovery()'
```

**Interpretation:** if logs show `systemd integration is not supported` / missing `python-systemd` and module import reports MISSING, install the Patroni `systemd` extra **in the same Python virtual environment**, with Rocky Linux build prerequisites as needed, then plan a controlled service start only after validating PostgreSQL's state and Patroni cluster state. Never change `Type=notify` blindly as the first action.

References: [Patroni installation dependencies](https://patroni.readthedocs.io/en/latest/installation.html); [Patroni 4.1.5 release notes](https://patroni.readthedocs.io/en/latest/releases.html); [PostgreSQL mailing list: missing python3-systemd caused Patroni Type=notify timeouts](https://www.postgresql.org/message-id/emb8322483-c237-4c49-ba54-c2767ade2054@fd3c26d9.com).

### Step 3AF — Diagnosis confirmed: Type=notify without systemd-python (2026-11-20)

**RUN AS root on `as-pg1` unless otherwise marked.** Reported evidence:

- `systemctl status patroni` showed `activating (start)` with Patroni PID `9708`.
- Its logs repeatedly reported `no action. I am (as-pg1), the leader with the lock`.
- PostgreSQL PID changed from `1075` to `9732`; this first adoption **did restart PostgreSQL**. Do not describe this handover as zero-downtime.
- A WAL sender was streaming to `as-pg2` and `pg_isready` reported accepting connections at the time of inspection.
- `/opt/patroni/venv/bin/python` could not import `systemd.daemon` (`systemd-python: MISSING`).
- Our unit specifies `Type=notify`; Patroni 4.1.x only implements that notification integration when the optional `systemd` dependency is installed. **This is the diagnosed systemd timeout mechanism, not evidence that PostgreSQL failed.**

**First: read-only live-state check, no immediate restart:**

```bash
systemctl show patroni -p ActiveState -p SubState -p MainPID -p NRestarts
runuser -u postgres -- /opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml list pg18-ha-lab
runuser -u postgres -- /usr/pgsql-18/bin/pg_isready -h /run/postgresql -p 5432
journalctl -u patroni -n 15 --no-pager
```

**Repair plan after verifying live state:** either (a) install Patroni's `systemd` extra *into the same virtual environment* and arrange a **controlled** Patroni restart to activate it, or (b) use `Type=simple` (documented by Percona's Patroni setup) and a controlled restart. Changing the unit alone while it is already activating is not a reliable immediate repair. `systemctl stop/restart patroni` can stop PostgreSQL; coordinate a brief lab outage, confirm the backup and standby state, then verify leader and replication slot after restart. With two etcd voting members, an entire VM failure loses etcd quorum. Do **not** attempt to repair by removing etcd keys, deleting PGDATA, or repeatedly starting services without assessing the status.

Sources: [Patroni installation extras](https://github.com/patroni/patroni/blob/master/docs/installation.rst); [Patroni 4.1.5 release notes](https://patroni.readthedocs.io/en/latest/releases.html); [Percona PostgreSQL 18 Patroni unit example](https://docs.percona.com/postgresql/18/solutions/ha-patroni.html).

### Step 3AG — Patroni restart loop stopped (COMPLETE, 2026-11-20)

**Server:** `as-pg1` · **RUN AS:** Linux `root`.

After a `Type=notify` readiness timeout, systemd had restarted Patroni four times. A subsequent diagnostic found `MainPID=0`, `ActiveState=activating`, `SubState=auto-restart`; `patronictl list` showed the stale member status `Replica/stopped`, and `pg_isready` reported no response. Do not mistake the stale Patroni member role for proof that the PostgreSQL data directory was converted into a standby.

To stop the restart loop, the following were successfully executed:

```bash
systemctl stop patroni
systemctl disable patroni
systemctl show patroni -p ActiveState -p SubState -p MainPID -p NRestarts
```

Observed result:

```text
MainPID=0
NRestarts=0
ActiveState=inactive
SubState=dead
```

**Safety:** Do **not** clear etcd keys, delete PGDATA, manually promote `as-pg2`, or start Patroni on `as-pg2` while the primary is being recovered. Confirm actual PostgreSQL state before restarting any manager.

### Step 3AH — Repair Patroni's systemd readiness dependency (NEXT)

The observed unit has `Type=notify`, but the Patroni virtual environment is missing `systemd-python` (`import systemd.daemon` raised `ImportError`). Patroni documented the `systemd` extra for `sd_notify()` integration; keep `Type=notify` for this production-style lab and install the missing dependency while Patroni is inactive. Refer to [Patroni installation](https://patroni.readthedocs.io/en/latest/installation.html) and [python-systemd README](https://github.com/systemd/python-systemd/blob/main/README.md).

**RUN AS root — `as-pg1` only:**

```bash
dnf install -y gcc python3-devel systemd-devel pkgconf-pkg-config
/opt/patroni/venv/bin/python -m pip install systemd-python
```

**Verification — RUN AS root — `as-pg1`:**

```bash
/opt/patroni/venv/bin/python - <<'PY'
import systemd.daemon
print('systemd-python: INSTALLED')
print('sd_notify integration: AVAILABLE')
PY
/opt/patroni/venv/bin/patroni --version
```

Expected Patroni version remains `4.1.5`. **Do not run `systemctl start patroni` yet.** After dependency verification, read-check both etcd endpoints and actual PG primary/standby availability before controlled startup. `Type=notify` should remain unchanged until these checks pass. Repeat the dependency installation on `as-pg2` before starting Patroni there.

### Step 3AH — systemd notification dependency repaired (COMPLETE, 2026-11-20)

**Server:** `as-pg1` · **RUN AS:** `root`.

After stopping the service restart loop, `systemd-python` was installed into `/opt/patroni/venv`. Verification succeeded:

```text
systemd-python: INSTALLED
sd_notify integration: AVAILABLE
patroni 4.1.5
```

The `Type=notify` systemd unit was not changed. This fixes the missing Python dependency, **but the Patroni service has not yet been shown healthy after a restart**. Keep PostgreSQL under a single manager; do not promote the standby or erase etcd metadata.

### Step 3AI — Read-only state verification before another Patroni start (NEXT)

**Server:** `as-pg1` · **RUN AS:** `root`.

```bash
runuser -u postgres -- /usr/pgsql-18/bin/pg_isready -h /run/postgresql -p 5432

runuser -u postgres -- /usr/pgsql-18/bin/pg_controldata /u01/18/data \
  | grep -E 'Database cluster state|Database system identifier'

etcdctl \
  --endpoints='http://192.168.136.10:2379,http://192.168.136.11:2379' \
  endpoint health -w table

etcdctl \
  --endpoints='http://192.168.136.10:2379' \
  get /service/pg18-ha-lab/leader
```

Interpretation: PG may not be serving while Patroni is stopped. The data directory's system identifier should remain `7709811654404845603`; both etcd members should be healthy. Leader lock may be absent after Patroni stops; inspect rather than modifying any keys. **Do not start services until results are reviewed.** On `as-pg2`, install `systemd-python` before enabling a `Type=notify` Patroni unit.

Reference: https://patroni.readthedocs.io/en/latest/installation.html

### Step 3AI — Pre-restart state checks confirmed (COMPLETE, 2026-11-20)

**Server:** `as-pg1` · **RUN AS:** root.

Observed after installing `systemd-python` and stopping Patroni's restart loop:

```text
pg_isready: /run/postgresql:5432 - no response
Database system identifier: 7709811654404845603
Database cluster state: shut down
etcd 192.168.136.10:2379: healthy
etcd 192.168.136.11:2379: healthy
/service/pg18-ha-lab/leader: no value returned
```

This shows a **cleanly shut down** PGDATA on the former primary, etcd quorum, and no active Patroni leader lock observed. Existing etcd metadata must be preserved; an absent leader lock alone is **not** a reason to clear cluster state.

### Step 3AJ — Controlled restart after systemd integration repair (NEXT)

First confirm that `as-pg2` is **still a standby** (do not promote it):

**RUN AS:** `postgres` on `as-pg2`:

```bash
/usr/pgsql-18/bin/psql -X -d postgres \
  -c 'SELECT pg_is_in_recovery() AS is_standby;'
```

Continue only if the result is `t`. If the standby is unexpectedly primary or inaccessible, reassess the role before bringing up the former primary.

Then **RUN AS:** `root` on `as-pg1`:

```bash
systemctl start patroni
systemctl show patroni -p ActiveState -p SubState -p MainPID -p NRestarts
runuser -u postgres -- /usr/pgsql-18/bin/pg_isready -h /run/postgresql -p 5432
runuser -u postgres -- /opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  list pg18-ha-lab
```

**Expected (not yet observed):** systemd active/running; PostgreSQL accepting connections; `as-pg1` holding the Patroni leader lock. **Stop and inspect logs** if the service fails; do not repeat restarts or promote `as-pg2` without checking DCS and both database roles. Leave Patroni disabled at boot until its restart is proven healthy. Verify physical slot `as_pg2_slot` and standby streaming after this recovery, before starting Patroni on `as-pg2`. The standby will also need the `systemd-python` dependency before using its planned `Type=notify` unit.

Official guidance: https://patroni.readthedocs.io/en/latest/existing_data.html

### Step 3AJ — Patroni primary recovered (OBSERVED, 2026-11-20)

**Server:** `as-pg1` · **Patroni version:** 4.1.5 · **PostgreSQL:** 18

After installation of `systemd-python` and controlled restart, `patronictl list pg18-ha-lab` returned:

```text
Member  Host            Role    State    TL
as-pg1 192.168.136.10  Leader  running  7
```

Patroni now reports the primary as Leader/running with system identifier `7709811654404845603`. The current timeline is **7**, higher than the earlier timeline 1: this is consistent with prior promotion attempts during the `Type=notify` restart loop. Timeline 7 does **not** establish whether `as-pg2` has followed the timeline changes. Do not conclude the replication side is healthy solely from `patronictl list`; only one Patroni member is registered so far. The Linux systemd `ActiveState` has not yet been explicitly shown after recovery, so verify it separately.

### Step 3AK — Verify original replication after Patroni primary recovery (NEXT)

**RUN AS:** `postgres` on **`as-pg1`**:

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT application_name, client_addr, state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
SELECT slot_name, slot_type, active, restart_lsn, wal_status
FROM pg_replication_slots
ORDER BY slot_name;"
```

**RUN AS:** `root` on **`as-pg1`** to verify the systemd readiness fix:

```bash
systemctl show patroni -p ActiveState -p SubState -p MainPID -p NRestarts
```

Interpretation: `as-pg2` should be connected as `streaming` through `repl_user`, and the original `as_pg2_slot` should still exist. If either condition is missing, investigate before starting Patroni on `as-pg2`. Then check `pg_is_in_recovery()`, `pg_stat_wal_receiver`, WAL replay position and errors on `as-pg2`; timeline compatibility is especially important because the primary has reached timeline 7. Do not delete any slot, promote the standby, use `pg_rewind`, or rebuild its PGDATA without confirming the actual replication state first.

Source references: https://patroni.readthedocs.io/en/latest/existing_data.html ; https://www.postgresql.org/docs/18/warm-standby.html

### Step 3AK — Primary WAL sender and original slot verified (OBSERVED, 2026-11-20)

**Run as:** `postgres` on `as-pg1` (PostgreSQL 18, Patroni 4.1.5).

After recovery of the Patroni primary (`Leader`, timeline 7), the primary reported:

```text
application_name | client_addr    | state     | sync_state | sent_lsn  | replay_lsn
walreceiver      | 192.168.136.11 | streaming | async      | 0/8000BB8 | 0/8000BB8

slot_name   | slot_type | active | restart_lsn | wal_status
as_pg2_slot | physical  | t      | 0/8000BB8   | reserved
```

**Interpretation:** The standby is connected and replayed the WAL reported as sent at that moment. The pre-migration physical slot `as_pg2_slot` remains active and its required WAL is reserved. This is evidence of current replication transport, but does **not** by itself establish that the standby has fully followed primary timeline 7. The standby remains outside Patroni management. Do not start its Patroni agent until the following read-only timeline check succeeds.

### Step 3AL — Verify standby timeline and WAL replay (NEXT)

**Run as:** `postgres` on `as-pg2`:

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT
  pg_is_in_recovery() AS is_standby,
  w.status,
  w.sender_host,
  w.slot_name,
  w.received_tli,
  w.flushed_lsn,
  pg_last_wal_receive_lsn() AS received_lsn,
  pg_last_wal_replay_lsn() AS replayed_lsn
FROM pg_stat_wal_receiver AS w;"
```

**Expected:** exactly one row, `is_standby = t`, `status = streaming`, `sender_host = as-pg1`, `slot_name = as_pg2_slot`, `received_tli = 7`; receive and replay LSNs should be close (or match for an idle system). If this returns no rows, a different timeline, or inconsistent positions, investigate before handing the standby to Patroni. Never manually promote, rebuild, or rewind based solely on a suspected mismatch.

Next after this check: confirm `systemd-python` is installed on `as-pg2` before creating/starting its `Type=notify` Patroni unit; the earlier primary startup timeout was caused by missing readiness notification support.

References: https://www.postgresql.org/docs/18/monitoring-stats.html ; https://patroni.readthedocs.io/en/latest/existing_data.html

### Step 3AL — Standby has followed primary timeline 7 (OBSERVED, 2026-11-20)

**Run as:** `postgres` on `as-pg2`.

The existing, *not-yet-Patroni-managed* PostgreSQL standby reports:

```text
is_standby | status    | sender_host | slot_name   | received_tli | flushed_lsn | received_lsn | replayed_lsn
t          | streaming | as-pg1      | as_pg2_slot | 7            | 0/8000BB8   | 0/8000BB8    | 0/8000BB8
```

This shows the standby has received WAL on timeline **7**, matching the current Patroni-managed primary timeline, and had replayed through its reported received position when checked. The pre-migration physical replication slot `as_pg2_slot` remains active on `as-pg1`. No `pg_basebackup`, `pg_rewind`, slot deletion, or manual promotion is required by these observations. Note that asynchronous replication does not guarantee zero data loss after a future failure.

### Step 3AM — Add systemd notification support to as-pg2 (NEXT)

**Server:** `as-pg2` **Linux user:** `root`.

Patroni 4.1.5 is already installed in `/opt/patroni/venv`; its previous installation omitted optional `systemd-python`. The primary's `Type=notify` service timed out and repeatedly restarted PostgreSQL until the integration was installed. Install the *same prerequisite on the standby before creating or starting its unit*:

```bash
dnf install -y gcc python3-devel systemd-devel pkgconf-pkg-config
/opt/patroni/venv/bin/python -m pip install systemd-python

/opt/patroni/venv/bin/python - <<'PY'
import systemd.daemon
print('systemd-python: INSTALLED')
print('sd_notify integration: AVAILABLE')
PY
/opt/patroni/venv/bin/patroni --version
```

Expected: `systemd-python: INSTALLED`, `sd_notify integration: AVAILABLE`, `patroni 4.1.5`. These commands **do not start Patroni and do not change the running PostgreSQL standby**. Next, inspect and prepare the standby's systemd unit pointing to `/var/lib/pgsql/patroni-migration/patroni-as-pg2-private.yml` and verify replication and DCS health before starting it. Ensure the native `postgresql-18.service` remains disabled. Do not overwrite `/u01/18/data` or delete `as_pg2_slot`.

Patroni references: https://patroni.readthedocs.io/en/latest/installation.html ; https://patroni.readthedocs.io/en/latest/existing_data.html


### Step 3AM — systemd integration verified on as-pg2 (OBSERVED, 2026-11-20)

**Server:** `as-pg2`  **RUN AS:** `root`.

The user installed the `systemd-python` integration in `/opt/patroni/venv` and verified:

```text
systemd-python: INSTALLED
sd_notify integration: AVAILABLE
patroni 4.1.5
```

This prevents the specific missing-notification problem that caused an earlier `Type=notify` timeout on `as-pg1`. This does **not** mean the standby has been adopted; as-pg2 is still streaming under the original PostgreSQL process.

### Step 3AN — Prepare the standby Patroni systemd unit (NEXT; do not start yet)

**Server:** `as-pg2`  **RUN AS:** `root`.

Before changing systemd, confirm the dedicated configuration file exists and is readable by Linux `postgres`:

```bash
stat -c '%U:%G %a %n' /var/lib/pgsql/patroni-migration/patroni-as-pg2-private.yml
systemctl is-enabled postgresql-18.service || true
```

Expected private YAML permissions: `postgres:postgres 600`; native PostgreSQL unit is disabled. Create the *initial migration* unit:

```bash
cat > /etc/systemd/system/patroni.service <<'EOF'
[Unit]
Description=Patroni PostgreSQL 18 HA Manager (as-pg2)
Wants=network-online.target etcd.service
After=network-online.target etcd.service

[Service]
Type=notify
User=postgres
Group=postgres
ExecStart=/opt/patroni/venv/bin/patroni /var/lib/pgsql/patroni-migration/patroni-as-pg2-private.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
Restart=no
TimeoutStartSec=120
TimeoutStopSec=120
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemd-analyze verify /etc/systemd/system/patroni.service
systemctl cat patroni
systemctl is-enabled patroni || true
systemctl is-active patroni || true
```

`Restart=no` is **intentional during the first adoption attempt**, due to the earlier repeated primary restarts when the missing systemd readiness dependency caused timeouts. After both nodes are healthy under Patroni, review an appropriate service restart policy and enable at boot. The official packaged service uses `Type=notify` and `KillMode=process`; this lab service keeps those settings while suppressing automatic retries during migration.

**Do not start/enable this standby Patroni unit yet.** Before the handover, recheck `as-pg1` Patroni leadership, etcd quorum, standby `pg_is_in_recovery() = true`, WAL receiver streaming on timeline 7, and the private YAML validation (do not dump the private YAML to GitHub). Then adopt standby via systemd and confirm both Patroni members in `patronictl list`. Do not delete/rebuild PGDATA or drop `as_pg2_slot`.

References: https://patroni.readthedocs.io/en/latest/existing_data.html ; https://sources.debian.org/src/patroni/4.1.5-2/extras/startup-scripts/patroni.service


## Stage 4 — Both existing PostgreSQL nodes adopted by Patroni (OBSERVED, 2026-11-20)

**Observed on:** `as-pg1` as Linux user `postgres` (run `patronictl` as `postgres`).

```bash
/opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  list pg18-ha-lab
```

Observed cluster state:

| Member | Host | Role | State | Timeline | Receive LSN | Replay LSN | Reported lag |
|---|---|---|---|---:|---|---|---:|
| as-pg1 | 192.168.136.10 | Leader | running | 7 | — | — | — |
| as-pg2 | 192.168.136.11 | Replica | streaming | 7 | `0/8000BB8` | `0/8000BB8` | 0 |

**Outcome:** Existing primary and replica were adopted by Patroni, using existing PGDATA, and both appear together in one Patroni cluster. No standby rebuild was required. Timeline advanced to 7 during the earlier `Type=notify` timeout/retry incident; the standby subsequently caught up to the same timeline. A zero lag value is an observed snapshot, not a promise of zero loss for asynchronous replication.

### Step 4A — Verify slots after both members are under Patroni (NEXT; read-only)

**Server:** `as-pg1`  **RUN AS:** Linux `postgres`.

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT r.application_name, r.client_addr, r.state, r.sync_state,
       s.slot_name, r.sent_lsn, r.replay_lsn
FROM pg_stat_replication AS r
LEFT JOIN pg_replication_slots AS s ON s.active_pid = r.pid;

SELECT slot_name, slot_type, active, restart_lsn, wal_status
FROM pg_replication_slots
ORDER BY slot_name;"
```

**Purpose:** Confirm streaming remains healthy and identify which physical replication slot Patroni is currently using after standby adoption. The original `as_pg2_slot` was explicitly protected in the initial Patroni bootstrap DCS settings. **Do not drop any slots yet.** Check DCS-managed/permanent slot configuration and required WAL retention before any cleanup.

### Stage 4B — Future tasks (NOT YET VERIFIED)

1. Confirm standby-side WAL receiver and replication slot after adoption, including `received_tli = 7`.
2. Check `systemctl is-active patroni` on both VMs and set tested restart/autostart policies; `as-pg2` initial unit used `Restart=no`.
3. Validate `patronictl` cluster state, synchronous/asynchronous policy, timeline and data consistency; plan a *controlled* switchover with both etcd members alive.
4. Verify old primary re-joins as Replica, `pg_rewind` behavior (if applicable), and replication slot management.
5. Document that a two-voter etcd cluster **does not** offer full automatic VM-failure failover.

**Safety:** Never commit `/var/lib/pgsql/patroni-migration/*private.yml`, `.pgpass`, or passwords to a public GitHub repository.

### Step 4A — Observed slots after Patroni adoption (CONFIRMED, 2026-11-20)

**Server:** `as-pg1` (Leader). **RUN AS:** Linux `postgres`.

Actual query results:

| Application | Standby address | Replication | Mode | Active slot | Sent LSN | Replay LSN |
|---|---|---|---|---|---|---|
| `as-pg2` | `192.168.136.11` | streaming | async | `as_pg2` | `0/8000BB8` | `0/8000BB8` |

| Slot | Type | Active | restart_lsn | wal_status |
|---|---|---|---|---|
| `as_pg2` | physical | true | `0/8000BB8` | reserved |
| `as_pg2_slot` | physical | false | `0/8000BB8` | reserved |

**Interpretation:** Patroni has switched the replica to the member-named slot `as_pg2`. The previous manual slot `as_pg2_slot` is unused, and the new slot has caught up to the old slot's `restart_lsn` at this check. The old permanent slot may retain WAL unnecessarily if left configured. Do not drop the slot manually while Patroni is configured to maintain it.

**Next step, READ-ONLY — inspect dynamic DCS configuration.** On `as-pg1`, RUN AS Linux `postgres`:

```bash
set -o pipefail
/opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  show-config pg18-ha-lab | \
/opt/patroni/venv/bin/python -c '
import sys, yaml
cfg = yaml.safe_load(sys.stdin) or {}
slots = cfg.get("slots") or {}
print("Permanent slots in DCS:", sorted(slots))
print("Original slot configured:", "as_pg2_slot" in slots)
print("Slot management enabled:", (cfg.get("postgresql") or {}).get("use_slots", "not explicitly set"))
'
```

This prints no credentials. Once live DCS configuration is confirmed, remove **only** the old permanent slot's DCS configuration via `patronictl edit-config`; don't remove other slots or the whole cluster configuration. The official Patroni migration guide requires the Patroni-created slot's `restart_lsn` to be greater than or equal to the original's before removing original permanent-slot protection: https://patroni.readthedocs.io/en/latest/existing_data.html . After that, verify old slot cleanup, check service lifecycle/autostart, and schedule a controlled switchover with both etcd voters alive. This two-voter etcd lab still cannot survive the loss of one etcd voter.

### Step 4B — Remove the migrated original permanent slot (READY FOR EXECUTION; NOT YET VERIFIED)

**Observed on `as-pg1` as Linux `postgres` (2026-11-20):** Patroni dynamic configuration reported:

```text
Permanent slots: ['as_pg2_slot']
Original slot configured: True
Slot management: True
```

**Prerequisites already observed:** `as-pg2` is streaming on Patroni-managed slot `as_pg2` (`active = true`); the old `as_pg2_slot` is inactive; both had `restart_lsn = 0/8000BB8` and `wal_status = reserved` when last inspected. Patroni is managing both nodes and cluster status shows Leader / streaming Replica on timeline 7. If replication state has changed or any check fails, recheck slot positions and do not remove protection.

**Goal:** Remove **only** `slots.as_pg2_slot` from Patroni's *dynamic DCS configuration*, preserving `postgresql.use_slots = true` and the active auto-managed `as_pg2` slot. Do not edit `bootstrap.dcs` on disk (it only applies during initial bootstrap), and do not manually run `pg_drop_replication_slot` at this stage.

**Server:** `as-pg1` — **RUN AS:** Linux `postgres`.

```bash
PAGER=cat /opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  edit-config pg18-ha-lab \
  --set 'slots.as_pg2_slot=null'
```

`null` tells `patronictl edit-config` to remove that nested dynamic key. **Review the diff carefully**: it should remove only the old `as_pg2_slot` mapping. Confirm the edit only if that is all it changes. Do not paste private configuration or passwords into GitHub. No PostgreSQL restart is required for this DCS edit.

**After completing the edit, verify (READ-ONLY):**

```bash
set -o pipefail
/opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  show-config pg18-ha-lab | \
/opt/patroni/venv/bin/python -c '
import sys, yaml
cfg = yaml.safe_load(sys.stdin) or {}
print("Permanent slots:", sorted((cfg.get("slots") or {}).keys()))
print("Slot management:", (cfg.get("postgresql") or {}).get("use_slots"))
'
```

Expected permanent slots list: `[]`. `use_slots` should remain `True`. After Patroni's HA loop removes unneeded slots, run on `as-pg1` as Linux `postgres`:

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c \
 "SELECT slot_name, slot_type, active, restart_lsn, wal_status FROM pg_replication_slots ORDER BY slot_name;"
```

Expected eventually: `as_pg2` exists and is active; the old `as_pg2_slot` is absent. **If it persists**, inspect Patroni logs and DCS state rather than dropping it blindly. Then recheck `patronictl list` before changing systemd autostart or testing controlled switchover.

Official references: https://patroni.readthedocs.io/en/latest/existing_data.html and https://patroni.readthedocs.io/en/latest/patronictl.html . The two-voting-member etcd lab has no quorum if either VM fails.

### Step 4C — Permanent-slot protection removal applied (2026-11-20)

**Observed** on `as-pg1` by Linux user `postgres`:

```text
Apply these changes? [y/N]: y
Configuration changed
```

This confirms that `patronictl edit-config pg18-ha-lab --set 'slots.as_pg2_slot=null'` was accepted. **It does not alone prove that the physical slot is deleted.** Earlier, the active Patroni-managed physical slot was `as_pg2` (`active=t`) and the retired slot was `as_pg2_slot` (`active=f`); both had `restart_lsn=0/8000BB8` and WAL status `reserved`.

**Next check — Server `as-pg1`, Linux user `postgres` (read-only):**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c "
SELECT slot_name, slot_type, active, restart_lsn, wal_status
FROM pg_replication_slots ORDER BY slot_name;"
```

Expected after Patroni's slot-management loop: `as_pg2` remains active and `as_pg2_slot` is absent. If the old slot remains, inspect dynamic DCS configuration (`patronictl show-config`) and Patroni logs before manual cleanup. **Do not manually drop the active `as_pg2` slot.** After verifying slots, reconfirm both nodes with `patronictl list` and review `systemd` service lifecycle settings before attempting a controlled switchover. A two-voter etcd cluster has no quorum if either VM goes down.

Reference: https://patroni.readthedocs.io/en/latest/existing_data.html


### Step 4D — Old physical slot successfully removed (VERIFIED, 2026-11-20)

**Server:** `as-pg1` — **RUN AS:** Linux `postgres`. Following the approved dynamic configuration edit that removed `slots.as_pg2_slot` from etcd, PostgreSQL returned:

```text
 slot_name | slot_type | active | restart_lsn | wal_status
-----------+-----------+--------+-------------+-----------
 as_pg2    | physical  | t      | 0/8000BB8   | reserved
```

**Result:** The temporary permanent slot `as_pg2_slot` has been removed, and the Patroni-managed `as_pg2` slot is active. The inactive old slot no longer retains WAL. Do not drop `as_pg2`. The local `bootstrap.dcs` content in the original candidate YAML is bootstrap-only; remove obsolete slot references from a sanitized **example** configuration before reusing it to bootstrap a fresh cluster, but never change live DCS configuration by editing that section.

### Step 4E — Verify Patroni service lifecycle on both nodes (NEXT, READ-ONLY)

**Servers:** `as-pg1` AND `as-pg2` — **RUN AS:** Linux `root`.

```bash
hostname
systemctl show patroni \
  -p ActiveState -p SubState -p MainPID -p NRestarts -p UnitFileState -p Type -p Restart
systemctl is-enabled patroni || true
```

Expected **right now**: Patroni active on both. Earlier `as-pg1` was enabled with `Restart=on-failure` and `as-pg2` was disabled with `Restart=no`; confirm these current settings before making changes. Neither service should be stopped or restarted as part of this inspection. Once verified, make a deliberate plan for startup and restart policy, then reconfirm cluster state and DCS health before a controlled `patronictl switchover` with both etcd members alive. A two-voter etcd cluster is **not** fault tolerant to a full VM failure.

Reference: [Patroni patronictl documentation](https://patroni.readthedocs.io/en/latest/patronictl.html).

### Step 4F — Make Patroni services persistent (2026-11-20, pending verification)

**Observed:** Both nodes were reported healthy and running. Explicit `as-pg2` output:

```text
Type=notify
Restart=no
MainPID=11753
NRestarts=0
ActiveState=active
SubState=running
UnitFileState=disabled
```

The replica was deliberately configured with `Restart=no` during adoption. Before a controlled switchover, switch it to the upstream Patroni systemd example's `Restart=on-failure` and enable Patroni autostart on both nodes. These actions must **not** restart the running PostgreSQL or Patroni processes.

**Server: `as-pg2`; RUN AS: root**

```bash
install -d -m 0755 /etc/systemd/system/patroni.service.d
cat > /etc/systemd/system/patroni.service.d/restart-policy.conf <<'EOT'
[Service]
Restart=on-failure
RestartSec=5
EOT
systemctl daemon-reload
systemd-analyze verify /etc/systemd/system/patroni.service
```

**Both servers: `as-pg1`, `as-pg2`; RUN AS: root**

```bash
systemctl enable patroni   # do not use --now and do not restart
hostname
systemctl show patroni -p ActiveState -p SubState -p UnitFileState -p Type -p Restart -p MainPID
```

Expected after changes: `ActiveState=active`, `SubState=running`, `UnitFileState=enabled`, `Type=notify`, `Restart=on-failure`, nonzero `MainPID`. If a node differs, inspect before proceeding. Keep `postgresql-18.service` disabled on both nodes to prevent conflicting process managers. **Do not run `systemctl restart patroni`** to apply a unit-file change; `daemon-reload` only reloads systemd's configuration.

Next: verify healthy `patronictl list` and etcd quorum, then perform a supervised `patronictl switchover` from `as-pg1` to `as-pg2`. Two-member etcd still requires both members for quorum and does not provide single-VM-failure tolerance.

Sources: [Patroni 4.1.5 upstream systemd service](https://sources.debian.org/src/patroni/4.1.5-2/extras/startup-scripts/patroni.service) and [systemctl reference](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html).


### Step 4G — Pending restart cleared; rewind preflight (2026-11-20)

**Observed:** After a controlled Patroni PostgreSQL restart on `as-pg2`, `patronictl list pg18-ha-lab` reports `as-pg1` Leader/running (timeline 7), `as-pg2` Replica/streaming (timeline 7), reported lag 0, and no pending-restart flag. The existing Patroni slot `as_pg2` is active; the retired `as_pg2_slot` is absent. `as-pg2` service confirmed active/running, `Type=notify`, `Restart=on-failure`, and enabled. Its config restart was handled with `patronictl restart`, not by restarting the Patroni systemd process.

**Pre-switchover safeguard:** Patroni's DCS setting `postgresql.use_pg_rewind` defaults to `false`. Because this exercise must not rebuild the old primary after leadership changes, confirm the setting and whether automatic PGDATA deletion/reinit on rewind failure or divergent timelines is disabled. The cluster has `data_checksums=on` and `full_page_writes=on`, meeting the critical `pg_rewind` data requirements. But remote authentication for a rewind connection to the *new* primary is not yet verified; the observed `pg_hba.conf` rules only permit replication connections from the other VM, not ordinary remote superuser / rewind connections. Do not change the cluster or initiate switchover until this is considered.

**Server:** `as-pg1` — **RUN AS:** Linux `postgres` (read-only, does not print credentials):

```bash
set -o pipefail
/opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  show-config pg18-ha-lab | \
/opt/patroni/venv/bin/python -c '
import sys, yaml
cfg = yaml.safe_load(sys.stdin) or {}
pg = cfg.get("postgresql") or {}
print("use_pg_rewind:", pg.get("use_pg_rewind", False))
print("remove_data_directory_on_rewind_failure:", pg.get("remove_data_directory_on_rewind_failure", False))
print("remove_data_directory_on_diverged_timelines:", pg.get("remove_data_directory_on_diverged_timelines", False))
'
```

If `use_pg_rewind` is false, decide on enabling it via `patronictl edit-config` **and** verify a workable rewind authentication path on the new primary before switching roles. A clean planned switchover may rejoin without rewind in some situations, but a diverged former primary cannot be assumed to rejoin without a rewind or reclone. Do **not** enable options that delete or reinitialize PGDATA automatically in this no-rebuild lab.

Once all preflight checks are satisfied and both etcd endpoints are healthy, Patroni's supported planned-switchover syntax is:

```bash
# Example; DO NOT RUN before completing rewind/authentication preflight
/opt/patroni/venv/bin/patronictl -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  switchover pg18-ha-lab --leader as-pg1 --candidate as-pg2
```

References: https://patroni.readthedocs.io/en/latest/dynamic_configuration.html and https://patroni.readthedocs.io/en/latest/patronictl.html .

### Step 4H — `pg_rewind` preflight result and dedicated role (2026-11-20)

**Observed from `patronictl show-config pg18-ha-lab`:**

```text
use_pg_rewind: False
remove_data_directory_on_rewind_failure: False
remove_data_directory_on_diverged_timelines: False
```

**Decision:** Do not switch roles yet. The pre-existing PG18 cluster has `data_checksums = on` and `full_page_writes = on`, but a dedicated PostgreSQL rewind connection has not been configured. The original `pg_hba.conf` rules permitted cross-node physical replication using `repl_user`, but did not allow ordinary remote connections for the `pg_rewind` source connection. Because Patroni adopted an already initialized database, it has not automatically created a new rewind role.

**Next action: create `rewind_user` on the current primary** (it will stream to the standby). This user is privileged for file-reading functions; restrict it to the two database node addresses, use a strong password, and do not publish credentials.

**Server:** `as-pg1` / **RUN AS:** Linux `postgres`

```bash
/usr/pgsql-18/bin/psql -X -v ON_ERROR_STOP=1 -d postgres <<'SQL'
CREATE ROLE rewind_user LOGIN;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_ls_dir(text, boolean, boolean) TO rewind_user;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_stat_file(text, boolean) TO rewind_user;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_read_binary_file(text) TO rewind_user;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_read_binary_file(text, bigint, bigint, boolean) TO rewind_user;
SQL

# Prompts for the password without exposing it in shell history:
/usr/pgsql-18/bin/psql -X -d postgres -c '\password rewind_user'

# Confirm role exists; do not display password hashes:
/usr/pgsql-18/bin/psql -X -d postgres -c \
  "SELECT rolname, rolcanlogin, rolsuper FROM pg_roles WHERE rolname = 'rewind_user';"
```

**Do not run a switchover yet.** Remaining tasks: (1) add narrow `host postgres rewind_user <peer-IP>/32 scram-sha-256` rules on both nodes using Patroni-managed `pg_hba` settings, and verify reload/loaded entries; (2) set `postgresql.authentication.rewind` with the same credential in the private Patroni YAML on both nodes, preserving permissions and not committing secrets to GitHub; (3) verify normal TCP connections for `rewind_user` in both directions (without printing secrets); (4) set `postgresql.use_pg_rewind=true` in Patroni dynamic config with `patronictl edit-config`; (5) retain both `remove_data_directory_*` settings false; (6) verify both etcd members and both PostgreSQL nodes, then perform a controlled switchover.

Sources: [PostgreSQL 18 `pg_rewind` — non-superuser permissions](https://www.postgresql.org/docs/18/app-pgrewind.html); [Patroni authentication and `pg_rewind` settings](https://patroni.readthedocs.io/en/latest/yaml_configuration.html); [Patroni dynamic settings](https://patroni.readthedocs.io/en/latest/dynamic_configuration.html).
