## [2026-08-25]

### Fixed

- Fixed PGSync Producer/Consumer role isolation in daemon mode after identifying logical replication slot contention between both workloads.

- Removed the unconditional startup `sync.pull()` from the daemon execution path. Previously, every PGSync instance executed `pull()` before entering its role-specific receive loop:
  - Consumers could access PostgreSQL logical replication slots already owned by the Producer, resulting in `replication slot is active for PID` errors.
  - Producers performed the initial catch-up twice: once from `main()` and again from `receive()`.

- Restricted PostgreSQL CDC and logical replication slot ownership to the Producer. The Producer is now exclusively responsible for:
  - `poll_db()` to receive PostgreSQL change notifications.
  - `pull()` to perform the initial database/WAL catch-up.
  - `truncate_slots()` to advance and clean up logical replication slots.

- Restricted Consumers to Redis event processing and document synchronization. Consumers may still query PostgreSQL when required to rebuild affected documents and relationships, but they no longer read, advance, validate, or clean up logical replication slots.

- Restricted logical replication validation to Producer instances. Consumers continue to validate PostgreSQL connectivity, Redis availability, schemas, tables, and document dependencies without requiring replication-slot privileges or logical replication configuration.

- Applied the same Producer/Consumer ownership model to both synchronous and asynchronous execution paths.

- Prevented Consumer-only execution outside daemon mode, avoiding execution paths that could incorrectly invoke `pull()` and access PostgreSQL logical replication slots.

### Architecture

The lifecycle is now explicitly separated by role.

**Before:**

```text
main()
 ├── pull()                    # Executed by every instance
 │
 └── receive()
      ├── Producer
      │    ├── poll_db()
      │    └── pull()          # Duplicate initial catch-up
      │
      └── Consumer
           └── poll_redis()
```
Result:
Consumers could compete with the Producer for the same logical replication slot.

**After:**

```text
Producer
 ├── poll_db()
 ├── pull()
 └── truncate_slots()

Consumer
 └── poll_redis()

Both
 └── status()
```

This restores the intended single-Producer / multi-Consumer architecture: the Producer is the sole owner of PostgreSQL CDC and logical replication slots, while Redis acts as the event queue used by Consumers to process and index changes independently without replication-slot contention.



## [2026-08-18]

### Fixed

This change improves PGSync multiprocessing stability during large reindex operations by switching the Python multiprocessing start method from the default `fork` behavior to `spawn`.

It also clarifies index lifecycle responsibilities so that only bootstrap/reindex operations are responsible for creating and publishing new physical Elasticsearch index generations.

### Why

During large reindexes, worker processes created with `fork` may inherit runtime state from the parent process, including:

* PostgreSQL connections and SSL sockets
* Elasticsearch connections
* Connection pools
* PGSync internal process state

Sharing or inheriting this state across forked workers can lead to connection corruption and unstable behavior under parallel workloads. During testing, this was observed through errors such as:

* `SSL error: decryption failed or bad record mac`
* `SSL SYSCALL error: EOF detected`
* Unexpected PostgreSQL connection resets
* Worker failures during long-running reindexes

This made reindex failures difficult to reproduce and made performance tuning unreliable because connection stability was not guaranteed.

### Multiprocessing Stability Fix

PGSync now creates an explicit multiprocessing context using:

`multiprocessing.get_context("spawn")`

and passes that context to `ProcessPoolExecutor` through `mp_context`.

With `spawn`, every worker starts as a clean and independent Python process instead of inheriting the parent's runtime state:

```text
Parent process
     |
     +-- spawn()
           |
           +-- Worker
               +-- initializes PGSync independently
               +-- opens its own PostgreSQL connection
               +-- opens its own Elasticsearch connection
```

This isolates worker resources and prevents PostgreSQL, SSL, Elasticsearch, and connection-pool state from being inherited between processes.

The configured `max_workers` behavior remains unchanged.

Worker exceptions are also re-raised after being logged so failures propagate to the main reindex process instead of allowing execution to continue in a potentially inconsistent state.

### Index Lifecycle Improvement

Index-generation responsibilities are now clearly separated:

* **Bootstrap/Reindex**

  * Creates new physical index generations.
  * Performs the full data load.
  * Finalizes the new index.
  * Publishes the completed generation through Elasticsearch aliases.

* **Producer/Consumer**

  * Operate only against already-published aliases.
  * Do not create new physical index generations during normal startup or restart.

This prevents a Producer or Consumer restart from accidentally creating a new timestamped physical index and makes workload restarts safer and more predictable.

### Validation

After switching multiprocessing from `fork` to `spawn`, full reindex executions completed consistently without the PostgreSQL SSL/protocol corruption and worker crashes observed previously.

This moves the reindex workflow from an unstable execution model with difficult-to-reproduce connection failures to a stable baseline that can now be tuned using reliable performance metrics.

### Expected Result

* Stable multiprocessing during large reindex operations.
* Independent PostgreSQL and Elasticsearch connections per worker.
* Correct propagation of multiprocessing failures.
* Safer Producer/Consumer restarts.
* Controlled physical index creation during bootstrap/reindex only.
* More reliable performance testing and tuning of full reindex workloads.

### Reference

Python multiprocessing documentation:

https://docs.python.org/3/library/multiprocessing.html


