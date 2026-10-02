# Technology Stack

## Languages

Every service is written in **Rust**. That means the warehouse, token broker, Role Authority, directory sync, login bridge, CI, and the Discord and Mumble integrations, as well as the backends of modules like the ESI viewer. Rust gives the smallest memory footprint, and its strictness pays off exactly where mistakes are expensive: token rotation, locking and rule evaluation. Laravel is still an option for business modules heavy on forms and workflows later on, but not for now.

Frontends use **Vue 3 with Rsbuild**, in TypeScript. We don't fork ZITADEL's login page (a React app); we only change its theme, so we never write React.

## Key libraries

Most needs are covered by standard crates: `tokio`, `axum` and `reqwest` for async HTTP, `sqlx` for PostgreSQL, `async-nats`, `clickhouse`, `ldap3`, `async-graphql`, plus `jsonwebtoken`, `aes-gcm` and `rustls` for tokens and encryption.

Three areas need more care:

- **The ESI-compatible API** is generated from ESI's published OpenAPI spec, so routes and payloads keep matching ESI as it changes.
- **The rule language is CEL**, using the Rust implementation `cel-interpreter`. It is less mature than Google's Go version. Where it falls short, for example in reporting which variables a rule reads, we fork it and improve it.
- **The login bridge needs an OIDC provider**, and no mature Rust crate exists for that. We write a minimal one: authorization code flow, discovery, signing keys and ID tokens. ZITADEL is its only client. It's security-sensitive, so it stays small and gets reviewed.

## Infrastructure

- **PostgreSQL** is the main database. pg_partman partitions growing tables automatically.
- **ClickHouse** holds the BI copy. **PeerDB** streams changes into it.
- **NATS JetStream** carries events from day one. Each service writes its data and its outgoing events in the same transaction (an outbox table), and a relay publishes those events to NATS. A change and its event therefore never get out of step.
- **Memcached** is the key/value cache: ESI freshness metadata, hot API responses, rendered reference data. It never holds credentials.
- **Valkey** holds locks. A Valkey lock only prevents duplicate work; PostgreSQL is always the real guard (see [03](03-warehouse.md#concurrency) and [02](02-token-broker.md#exactly-one-refresher)).
- **Caddy** is the reverse proxy and handles TLS.
- **VictoriaMetrics, Grafana, Loki and Tempo** cover metrics, logs and traces.
- **pgBackRest** backs up PostgreSQL. The broker's database gets its own encrypted backup. The directory and ClickHouse can be rebuilt from their sources, and the SDE can be reloaded, so none of them need their own backup.
- **Deployment** uses Docker containers. Compose is used for the proof of concept, and production runs the same images. Kubernetes isn't needed on a single host.

## Proof of concept

A Docker Compose setup with PostgreSQL, ZITADEL, 389-ds, the login bridge, a thin directory sync, NATS, Valkey, Memcached, Caddy, a Jira DC trial, and one of Discourse or Confluence. It is done when:

1. SAML login to Jira works, with groups delivered by the claims Action;
2. "Log in with EVE" through the bridge maps to the right person, is refused when the character isn't linked, and stores the token;
3. MFA after a bridge login follows the configuration;
4. Jira syncs users and groups from 389-ds;
5. CEL in Rust can report which facts a rule reads (in our fork if necessary);
6. we have measured how much memory the identity stack uses.

## Memory on one host (rough, to be measured)

PostgreSQL gets as much memory as we can give it. ClickHouse needs a few GB. 389-ds needs about 1 GB for 200k entries. ZITADEL needs a few hundred MB. Each Rust service uses tens of MB. NATS, Valkey, Memcached, Caddy and monitoring together need one to three GB.
