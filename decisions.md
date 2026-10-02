# Decision Log

Decisions keep their original numbers so they can be referred to. Each entry gives the decision and why it was made. Entries marked *(proposed)* are awaiting confirmation or a proof of concept. Decisions that were later replaced are listed at the end.

## Scope and constraints

- **D1** Self-hosted only, with no cloud services. *Owner requirement.*
- **D2** A new build. SeAT and Alliance Auth serve only as references. *They don't fit the target model.*
- **D3** The documents cover the whole back-office; delivery is incremental.
- **D15** Plan for 1M tokens, all full-scope. *Performance must not depend on narrow scopes.*
- **D24** Design for a single host; replicas are optional. *Budget.*
- **D51** No Java. *Memory footprint.*

## Identity and directory

- **D4** Persons (email and password) and characters (EVE SSO) are separate, and a link between them is optional. *Some people have no characters; some characters have no person.*
- **D5** One directory for all in-house apps, with no on-the-fly provisioning. *Copies of accounts in each app make deactivation unreliable.*
- **D7** SAML for in-house apps, and OIDC for external and stateful ones. In-house apps read LDAP with a service account. *SAML is simpler and stateless.*
- **D8** No inbound SCIM. Outbound SCIM only as an adapter, if ever needed. *Nothing upstream provisions accounts, and none of our apps need SCIM.*
- **D9** People who leave keep their accounts and lose their roles.
- **D13** IdP, account site, token broker, admin console and Role Authority deploy separately. *Maintaining one must not take down the IdP.*
- **D45** The directory is 389 Directory Server, with a flat layout and one writer (directory sync). *Its plugins and tooling are mature; one writer keeps concurrency simple.*
- **D46** Jira and Confluence connect to LDAP without filtering. *License count doesn't matter.*
- **D47** Email is the login name, and apps key on a separate stable person ID. *Fewer mistakes; email changes don't rename anyone.*
- **D48** Directory entries list linked characters and the main character, but never intelligence alts. *In-house apps need them.*
- **D49** Mumble uses an HTTP authenticator with one-time tokens or app passwords. *Mumble has no SSO; tokens issued after an SSO login keep MFA in force.*
- **D50** Linked logins (EVE, Discord) never create accounts.
- **D52** The IdP holds credentials, and the directory holds no user passwords. Apps that can only check passwords get per-app app passwords. LDAP is internal only. *No app needs LDAP password checks, and this removes the only way around MFA.*
- **D53** *(proposed)* ZITADEL is the workforce IdP. Groups are added at login from the Role Authority. *Go, SAML and OIDC, external logins that can't create accounts, a replaceable login UI.*
- **D54** No shared IdP for public sites. Each has its own EVE client and sessions, and the broker exchanges their codes. *Sites don't need shared login state.*
- **D57** External logins go through our login bridge, the only external provider in ZITADEL. Links are looked up in our own store at login time. *One source of truth for links; EVE's quirks stay in our code.*
- **D58** Whether an external login counts as MFA is configurable per provider, and sensitive apps and roles can demand our own MFA. *CCP's 2FA is optional and can't be verified.*

## Roles and delegation

- **D10** Role membership is recomputed from facts whenever they change, and rule dependencies may not form cycles. *Polling made access changes slow.*
- **D11** `member` depends on the registry (seconds) and on ESI affiliation (limited by its cache). *Alliances come and go; that needs a fast manual switch.*
- **D12** A free-form org tree, with delegation limited to subtrees.
- **D44** Conditions on a tree node apply to every role below it. Delegates can add conditions but never remove inherited ones. *Restrictions like `good_standing` can't be forgotten or bypassed.*
- **D77** Leadership roles are role families generated per corp and alliance from in-game roles and the executor corp. *Groups follow entities automatically.*
- **D87** Special powers are designations (`ci`, `intelligence-master`, `intel-self-designation`, `audit`) that the root owner attaches to ordinary roles. *No hard-coded special roles.*

## Token broker

- **D14** A token is identified by character, client and scope set, and there are several EVE clients. *Availability and branding.*
- **D17** Only credentials are encrypted, in an isolated store. *A leak elsewhere exposes no credentials.*
- **D18** The broker is the only refresher, and leases access tokens to others. *Refresh-token rotation allows only one refresher.*
- **D29** Special-case apps may lease raw access tokens, but are registered and get quotas. *Some needs can't go through the warehouse.*
- **D55** The broker holds all external tokens (EVE, Discord), including tokens without an owner. The IdP only checks logins.
- **D56** Every EVE login requests all scopes and stores the token. *Capture tokens at every opportunity.*

## Warehouse

- **D16** All ESI traffic goes through the warehouse, and apps read from it live. *Caching, fewer upstream calls, gentler on rate limits.*
- **D19** Duplicate fetches are absorbed by the cache, not by a planner. *A planner costs too much.*
- **D20** Fetching is broad, paced, and aware of rate limits and errors. *Coverage over speed; don't overload ESI.*
- **D21** One fetch per key at a time, using a Valkey lock, with PostgreSQL fencing so stale writes are rejected.
- **D22** History is kept as complete row versions, not diffs. *BI can query it; growth follows the rate of change.*
- **D23** Growing tables are partitioned automatically. *Data is kept forever.*
- **D25** ESI-compatible REST plus GraphQL, and 404 when there is no data.
- **D26** The warehouse enforces read access: app grants and per-app policies for client tokens, ESI rules for ESI tokens. *Every token is full-scope, so control must be central.*
- **D27** Only the SDE is separated: its own schema, not backed up, swapped as a whole.
- **D28** BI reads a ClickHouse copy fed by change data capture. *Faster analytics, and it protects the main database.*
- **D30** On a cache miss, an ESI-token request is fetched with the caller's token. The result is stored, the token is not.
- **D31** BI may lag by minutes, but syncing must not stop.
- **D36** Interactive fetches have the highest priority, a reserved budget, promotion of queued fetches, and a deadline after which stale data is served. *Waiting requests must not stall.*
- **D71** *(proposed)* Routes without a data model are served as a caching proxy. *Every route works from day one, which lets SeAT move behind the warehouse early.*

## Counter Intelligence

- **D32** CI is its own system, run by its own team. *The most sensitive data in the platform.*
- **D33** CI acts through a `ci_flagged` fact, used in rules via `good_standing`.
- **D34** Certain signals trigger alerts directly; suspicious ones only open cases. *Heuristics produce false positives.*
- **D35** Recruiters see only flagged yes or no, and every sensitive read is logged.
- **D37** Subjects can be characters, persons, or corps and alliances (current members, or historical with an optional time window).
- **D38** Entries are lifted by hand; nothing expires automatically.
- **D39** Notes can be written about anyone. Who reads them is role-based, and a note can only be made stricter than the default.
- **D40** Login IP and device data is kept indefinitely in the IdP. CI receives only derived signals.
- **D41** Alerts go through a routing framework. *How to detect people early will be revisited.*
- **D42** Jira runs the workflow for reports and appeals, while CI keeps the record and applies decisions. *Avoids building a second ticket system.*
- **D43** Requests are suspect reports and appeals. Reporters only see the state of their own report.

## ESI viewer and standings

- **D75** Module order: ESI viewer, then fleet tools, then SRP, then the rest.
- **D76** The owner sees everything of their own. Corp roles, directors and alliance leadership see members and all their alts, except intelligence alts.
- **D78** Intelligence alts are hidden from leadership everywhere: viewer, LDAP, claims, rules, integrations. *Hiding them only in the viewer would leak.*
- **D79** Recruiters get a time-limited share link covering all of an applicant's characters except intelligence alts.
- **D80** Sensitive views are logged, and only admins can read the log.
- **D82** What each role sees is a configurable matrix (role × character class × section), with alliance overrides that can only narrow it. Fixed in code: owners see their own characters, and only `ci` and `intelligence-master` roles can see intelligence alts.
- **D83** Hostile alts are their own class, behaving like neutral alts by default.
- **D84** Intelligence alts are designated by CI or the intelligence master (also before linking), or by the person while linking, which needs a role if the alt is in a member or blue entity and is reviewed by CI. The check happens inside the link operation. *No gap in which the alt is visible.*
- **D85** A Standings subsystem turns in-game contacts and managed entries into relations and roles, with edits governed by roles.
- **D86** Characters in any member entity count as members, with no separate "coalition alt" class. Sub-corps are a class of their own, with limited access.
- **D88** The corporation pages cover every corp endpoint.
- **D89** Standings come from alliance contacts and selected corp contacts, read only.
- **D90** The coalition and each alliance have separate standings, and each alliance chooses whether to follow the coalition. An alliance's standings affect only that alliance.
- **D91** Sub-corp characters get limited access through roles, which each app maps as it sees fit.

## Stack

- **D60** Events use an outbox table and NATS JetStream, from the start.
- **D61** *(proposed)* Rules are written in CEL. *Safe and fast, and the facts each rule reads can be extracted.*
- **D62** *(proposed)* pg_partman for partitioning, and PeerDB for feeding ClickHouse.
- **D63** *(proposed)* Docker containers on one host, Caddy, VictoriaMetrics/Grafana/Loki/Tempo, pgBackRest.
- **D64** Rust for all services. Missing library features (such as in CEL) are added by forking. Laravel stays an option for later.
- **D65** Vue with Rsbuild for frontends. ZITADEL's login page is only re-themed.
- **D66** Memcached is the cache and Valkey holds locks, but PostgreSQL is always the real guard.
- **D67** The proof of concept runs on Docker Compose.

## Migration

- **D68** *(proposed)* One track: proof of concept, then broker, warehouse core and SeAT cutover, then fact data, identity, Role Authority with CI core, app cutover, full warehouse with the viewer, SeAT retirement, and finally modules.
- **D69** *(proposed)* SeAT's token readers stay untouched. Only its writers change, and an adapter on our side keeps token status in sync. The switch happens in one maintenance window.
- **D70** *(proposed)* SeAT accounts are imported unclaimed and claimed through an EVE login plus a newly verified email.
- **D72** *(proposed)* CI's core ships with the Role Authority. *SeAT enforces its blacklist today.*
- **D73** *(proposed)* SeAT's blacklisted characters become CI entries, with a review list of people whose status widens.
- **D74** *(proposed)* SeAT SIG memberships are imported; those no longer eligible are suspended and reported.

## Replaced decisions

- **D6** Passwords in the directory. *Replaced by D52.*
- **D59** Go for backend services. *Replaced by D64.*
- **D81** A manually maintained relations registry. *Replaced by D85.*
- An earlier version of **D56** proposed a scope-less EVE client for logins.
