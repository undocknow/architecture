# ESI Warehouse

The warehouse is the only component that calls ESI. It fetches data with tokens leased from the [broker](02-token-broker.md), keeps it forever with history, and serves it to every other part of the system. Apps don't keep their own copies of ESI data; they ask the warehouse when they need something.

## Fetching

The fetcher doesn't care who owns a token. Which endpoints a token can read depends on its scopes and on the character's in-game roles. A director's token can read corp wallets; a line member's can't. When ESI answers 403, the fetcher records that and stops trying that endpoint with that token.

### Wide, not fast

The goal is to keep everything reasonably fresh, not to keep a few things extremely fresh. Work that is due is spread fairly across all tokens. Popular data is never refetched early at the expense of other data.

We also deliberately stay below ESI's limits. A global request budget is spent smoothly rather than in bursts. It shrinks when ESI's error-limit or rate-limit headers say so, backs off on 5xx errors and timeouts, and pauses during daily downtime. The budget is configuration, and the worker pool sizes itself to the work that is due.

Work runs in three priority classes:

1. **Interactive.** Someone's API request is waiting for this fetch right now. Part of the budget is reserved for this class, so background work can never starve it. If such a request is waiting on a fetch that is still queued at a lower priority, that fetch is promoted. If the fetch can't start within a short deadline, for example during downtime, the request gets the stored copy with headers showing its age. The fetch stays queued, and its result will be there for the next request.
2. **Fact-critical.** Affiliation and roles, because they drive `member` status and the Role Authority.
3. **Background.** The broad sweep over everything else.

Affiliation is cheap to keep fresh. `/characters/affiliation/` is public and accepts up to 1000 IDs per call, so a pass over 1M characters is about 1000 requests.

### Duplicate fetches

Several tokens can often reach the same data. For example, thirty directors' tokens can all read the same corp wallet. Rather than plan ahead which token fetches what, the cache absorbs the duplicates. Responses are cached under the route and parameters, without the token. Before spending budget, a worker checks whether a copy for that key is still within its `Expires` time, and skips it if so. The check reads Memcached first and falls back to PostgreSQL. Losing Memcached costs speed, never correctness. Requests carry `If-None-Match`, so unchanged data comes back as a cheap 304 and nothing is written.

### Concurrency

This is the part most likely to go wrong, so the rules are strict:

- **One fetch per key at a time.** A worker takes a Valkey lock (`SET NX` with a time limit) before fetching, and skips the key if someone else holds it. It can only release its own lock. If a worker crashes, the lock expires on its own.
- **PostgreSQL has the final say.** A Valkey lock can expire while a slow worker is still running, and it disappears if Valkey restarts. So each fetch also takes an increasing fencing number, and a write is only accepted if its number is newer than the one already stored. A slow worker therefore can't overwrite fresher data.
- **No going backwards.** Data whose `Last-Modified` is older than what is stored is ignored.
- **Readers share a fetch.** Several API reads waiting on the same missing key all wait for one fetch; they don't start one each.
- **History changes are atomic.** Closing the old row version and inserting the new one happen in one transaction.
- **Locks never nest across services.** A worker never holds a fetch lock while waiting on the broker's refresh lock.

Lock time limits and retry behavior will be tuned by load testing.

### Passthrough for unmodelled routes

Every ESI route works from day one, even before we have built a proper data model for it. For such routes the warehouse acts as a caching proxy. It fetches under the same locks, budget and priorities, and stores the raw response with its ESI headers. When a route later gets its data model, it switches over, and the raw archive stays as a record. Passthrough is what lets SeAT move behind the warehouse early in the migration.

## Storage

PostgreSQL is the primary store. Readers and fetchers use separate connection pools, and reads take priority. The design works on a single host. Read replicas could be added later, so code must not assume reads and writes share a connection.

Growing tables (wallet journals, transactions, killmails, market history, notifications, history versions) are partitioned automatically with pg_partman. Public and private ESI data share one database.

The **SDE** (static game data) sits in its own schema, so joins with ESI data still work. It is left out of backups because it can always be reloaded from CCP's release. A new release is loaded into a staging schema and swapped in with a rename in one transaction.

**History.** Snapshot-style endpoints, such as assets, skills, clones and contacts, are kept as row versions with `valid_from` and `valid_to`. On each fetch, only items that actually changed get a new version; an unchanged response writes nothing. Every row is complete, so BI tools can query it directly. Current-state views just filter on `valid_to IS NULL`. Storage grows with how much data changes, not with how often we fetch.

**Encryption.** Only credentials are encrypted, and those live in the broker. The warehouse stores opaque token IDs, nothing secret.

## Read APIs

The warehouse offers two APIs:

- **ESI-compatible REST.** It uses the same paths (including versioned ones such as `/v4/…` and `/latest/…`), the same payloads and the same errors, plus `X-Pages`, ETag and freshness headers. Existing ESI client libraries work after changing the base URL. The freshness headers describe how old the warehouse's copy is.
- **GraphQL.** The same data, plus questions that cross entities, which ESI can't answer.

If the warehouse has no data for a request, it answers 404.

## Who may read what

The warehouse enforces access itself, because apps can't be trusted to filter data that every full-scope token could fetch. It accepts two kinds of credentials.

**Client token.** An app's backend calls the warehouse with a token from our IdP. Browsers never talk to the warehouse directly; apps proxy for them. Access is checked in two steps. First, the app's grant decides what this app may read at all. Second, if the app acts on behalf of a person, the person's roles must cover the data being asked for. Each app has its own policy module inside the warehouse. The [ESI viewer](08-esi-viewer.md) is the main example.

**ESI token.** Anyone holding an EVE access token can use the warehouse exactly as they would use ESI. The token is verified against CCP's keys, and access is limited to what that token could read from ESI. If the warehouse has no fresh copy, it calls ESI with the caller's token at interactive priority, stores the result for everyone, and never keeps the caller's token.

## BI

Metabase and ad-hoc queries run against a ClickHouse copy, not against PostgreSQL. PeerDB streams changes from PostgreSQL into ClickHouse, and the SDE is loaded there as well. A few minutes of lag is fine, as long as syncing never stops. Lag is monitored, and a stalled stream raises an alert.
