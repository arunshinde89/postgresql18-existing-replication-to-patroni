# Follow-up: pg_rewind, planned switchover and verification

**Status:** Not performed as of 20 November 2026. This is a future lab procedure, not a claim of successful switchover.

## Why defer?

The cluster currently uses `postgresql.use_pg_rewind: false` in its dynamic Patroni config; `rewind_user` exists on the primary, but the required cross-node PostgreSQL **normal database** connections and private Patroni rewind credentials have not been configured or validated. The two automatic PGDATA-removal flags are false and should remain false for this exercise.

A *clean planned switchover* may succeed and allow the old primary to rejoin without `pg_rewind`, but if the timelines diverge without rewind configured, rejoin can fail. There is no reason to rush this after the migration objective is met.

## Before performing the optional switchover

1. Confirm the role has replicated to the standby and that its password was actually set. Never print the password or hash.
2. Use Patroni-managed `pg_hba` configuration to allow **narrow** `host postgres rewind_user <peer>/32 scram-sha-256` connections on **both** nodes, with the appropriate Patroni syntax and existing rules preserved. Test that rules are actually loaded. Authentication to the source for `pg_rewind` is **not** a `replication` pseudo-database connection.
3. Add `postgresql.authentication.rewind` with username `rewind_user` and its password to both **private**, `0600` Patroni YAML files, without publishing them. Validate both YAML files. Apply configuration changes safely using the documented Patroni reload procedure; note that `bootstrap.dcs` no longer manages live DCS after initial bootstrap.
4. Test secure remote connections to the `postgres` database in **both directions** as `rewind_user`, without displaying passwords, and verify rewind function privileges.
5. Enable `postgresql.use_pg_rewind=true` via `patronictl edit-config` and verify with `patronictl show-config`. Keep `remove_data_directory_on_rewind_failure=false` and `remove_data_directory_on_diverged_timelines=false`.
6. Confirm `as-pg1` is Leader/running, `as-pg2` Replica/streaming with no pending restart and near-zero lag, both etcd endpoints are healthy, and there are no writes requiring a higher durability guarantee.

## Optional controlled switchover (only after preflight)

**RUN AS Linux user `postgres` on `as-pg1`:**

```bash
CFG=/var/lib/pgsql/patroni-migration/patroni-as-pg1-private.yml
/opt/patroni/venv/bin/patronictl -c "$CFG" list pg18-ha-lab
/opt/patroni/venv/bin/patronictl -c "$CFG" switchover pg18-ha-lab \
  --leader as-pg1 --candidate as-pg2
/opt/patroni/venv/bin/patronictl -c "$CFG" list pg18-ha-lab
```

**Expected (not yet observed):** `as-pg2` becomes Leader/running; `as-pg1` rejoins Replica/streaming; timeline increments; LSNs and slot states are healthy. Investigate if the former primary fails to rejoin; do **not** delete data or force promotion.

## Verification commands

**RUN AS Linux user `postgres` on each database node:**

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c \
 'SELECT pg_is_in_recovery() AS is_standby;'
```

Note: `pg_current_wal_lsn()` is not available in recovery. Check roles separately, and run `pg_current_wal_lsn()` **only on the leader**; on replicas use `pg_last_wal_receive_lsn()` and `pg_last_wal_replay_lsn()`.

**Leader only** (run as `postgres`):

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c \
 'SELECT application_name, client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;'
```

**Replica only** (run as `postgres`):

```bash
/usr/pgsql-18/bin/psql -X -d postgres -c \
 'SELECT status, sender_host, slot_name, received_tli FROM pg_stat_wal_receiver;'
```

**Further reading:** [PostgreSQL 18 pg_rewind](https://www.postgresql.org/docs/18/app-pgrewind.html), [Patroni dynamic configuration](https://patroni.readthedocs.io/en/latest/dynamic_configuration.html), [patronictl switchover](https://patroni.readthedocs.io/en/latest/patronictl.html).
