# PostgreSQL 18: Convert Existing Streaming Replication to Patroni HA

**Status: Migration completed; planned switchover not yet tested.**  
**Lab date:** 20 November 2026 | **Platform:** Rocky Linux 10.1 / VMware Workstation

This hands-on DBA project converts an **existing**, healthy PostgreSQL 18 primary/standby streaming-replication setup into a Patroni-managed cluster **without reinitializing either database or rebuilding the standby**.

## Architecture

```text
                  Patroni cluster: pg18-ha-lab

  as-pg1 (192.168.136.10)          as-pg2 (192.168.136.11)
  ------------------------          ------------------------
  PostgreSQL 18  LEADER    --WAL--> PostgreSQL 18  REPLICA
  Patroni 4.1.5                    Patroni 4.1.5
  etcd 3.6.17 (Raft leader*) <----> etcd 3.6.17 (follower*)
   :5432 / :8008 / :2379 / :2380    :5432 / :8008 / :2379 / :2380

  *etcd leadership is independent of PostgreSQL leadership.
```

| Property | Value |
|---|---|
| Patroni scope | `pg18-ha-lab` |
| PostgreSQL system ID | `7709811654404845603` (both nodes) |
| PGDATA | `/u01/18/data` (both nodes) |
| PostgreSQL | 18; physical asynchronous replication |
| Patroni | 4.1.5, with `systemd-python` integration |
| etcd | 3.6.17; **2 voting members** |
| Current topology verified | `as-pg1`: Leader/running; `as-pg2`: Replica/streaming |
| Last observed timeline | `7` on both members |
| Last observed standby lag | `0` in `patronictl list` |
| Patroni physical slot | `as_pg2` (active); original `as_pg2_slot` removed |
| Backup | `pg_basebackup` plus `pg_verifybackup` successful |

> **Lab-only HA limitation:** Two-member etcd requires **2/2 quorum**. Losing either whole VM destroys etcd quorum, so this layout **cannot provide automatic failover after one VM failure**. Use three voting etcd members (or an appropriate production DCS architecture) for fault tolerance. Do not mistake a two-VM Patroni installation for production-grade fault-tolerant HA.

## Results verified

1. Confirmed native PostgreSQL streaming replication and asymmetric primary/standby roles.
2. Validated matching database identifiers, checksums enabled, `full_page_writes=on`, and connection authentication.
3. Built a functioning two-member etcd cluster and verified quorum, leadership and both endpoint health.
4. Installed Patroni and generated candidate configurations **from existing running PostgreSQL instances**.
5. Preserved the existing physical replication slot `as_pg2_slot` during adoption and securely reused replication credentials.
6. Migrated existing primary and standby into Patroni, without `initdb` or rebuilding PGDATA.
7. Diagnosed systemd `Type=notify` start timeouts caused by missing `systemd-python`; corrected the dependency and recovered the primary.
8. Verified standby streaming on timeline 7, then safely retired the old slot, leaving Patroni-managed `as_pg2`.
9. Cleared a `cluster_name` pending restart via `patronictl restart` on the replica.
10. Created a non-superuser `rewind_user` with the documented PostgreSQL file-reading grants; **not yet integrated into Patroni**.

## Current cluster status (last observed)

```text
Member  Host             Role     State       TL  Replay/Receive lag
as-pg1  192.168.136.10   Leader   running      7  —
as-pg2  192.168.136.11   Replica  streaming    7  0
```

These are **observations from the lab**, not a live dashboard or guarantee of current health.

## Check the cluster

**RUN AS Linux user `postgres` on `as-pg1`:**

```bash
/opt/patroni/venv/bin/patronictl \
  -c /var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml \
  list pg18-ha-lab
```

**RUN AS Linux user `postgres` on the current leader:**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c '
SELECT application_name, client_addr, state, sync_state,
       sent_lsn, replay_lsn FROM pg_stat_replication;'

/usr/pgsql-18/bin/psql -X -d postgres -c '
SELECT slot_name, slot_type, active, restart_lsn, wal_status
FROM pg_replication_slots ORDER BY slot_name;'
```

**RUN AS Linux user `root` on either node:**

```bash
etcdctl --endpoints='http://192.168.136.10:2379,http://192.168.136.11:2379' endpoint health -w table
systemctl show patroni -p ActiveState -p SubState -p Restart -p UnitFileState
```

## Project files

- [`docs/DETAILED_MIGRATION_RUNBOOK.md`](docs/DETAILED_MIGRATION_RUNBOOK.md) — complete, chronological installation and migration commands, real outputs, incident analysis and explanations.
- [`docs/FOLLOW_UP_SWITCHOEVER.md`](docs/FOLLOW_UP_SWITCHOEVER.md) — safe remaining work, rewind limitations, preflight and planned switchover procedure.
- [`config/`](config/) — lab etcd YAML and Patroni/etcd systemd service templates; audit before reuse.
- [`.gitignore`](.gitignore) — excludes private YAML, credentials, PGDATA and local backups.

## What remains (intentionally deferred)

- Set up cross-node `pg_rewind` authentication in the Patroni-managed PostgreSQL `pg_hba` rules; configure `postgresql.authentication.rewind` privately **on both nodes**, test the connections, and enable `postgresql.use_pg_rewind` in Patroni dynamic configuration.
- Verify **both** etcd endpoints and a healthy zero-lag replica, then perform a supervised planned switchover from `as-pg1` to `as-pg2`.
- Confirm `as-pg1` rejoins as a streaming replica and the slots are managed correctly; optionally switch back.
- For real production HA, replace the two-voter etcd topology with a quorum that tolerates failure, use TLS and secret management, fencing/STONITH safeguards where applicable, backups off-host, monitoring, and client routing (e.g. HAProxy).

**Do not manually promote the standby while DCS quorum is absent. Do not delete PGDATA or etcd keys during troubleshooting.**

## Key lessons

- The Patroni migration procedure can adopt existing databases without a standby rebuild, but **joining may restart PostgreSQL**; treat it as a controlled change, not a guaranteed zero-downtime operation.
- `Type=notify` needs systemd notification support in Patroni's Python environment. Missing `systemd-python` caused a restart loop and multiple timeline changes in this lab.
- A permanent slot can protect an old standby during migration. Once the standby uses a Patroni-managed slot and positions are checked, retire the old slot's DCS definition and let Patroni clean up.
- Patroni's dynamic DCS settings are not updated by simply editing `bootstrap.dcs` after the initial bootstrap.
- **Asynchronous** streaming does not guarantee zero data loss on abrupt primary failure, even if measured lag was zero during a status check.

## References

- [Patroni: migrate from existing PostgreSQL](https://patroni.readthedocs.io/en/latest/existing_data.html)
- [Patroni `patronictl` commands](https://patroni.readthedocs.io/en/latest/patronictl.html)
- [Patroni dynamic configuration](https://patroni.readthedocs.io/en/latest/dynamic_configuration.html)
- [PostgreSQL 18: `pg_rewind`](https://www.postgresql.org/docs/18/app-pgrewind.html)
- [etcd quorum and FAQ](https://etcd.io/docs/v3.6/faq/)

**Security:** Do not commit `patroni-as-pg1-private.yml`, `patroni-as-pg2-private.yml`, passwords, `.pgpass`, backups or PostgreSQL data. The included configuration templates use HTTP and private lab IPs **only for an isolated learning environment**.
