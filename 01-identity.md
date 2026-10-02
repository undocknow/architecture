# Identity

## Persons and characters

A **person** is an account. It is created by registering with an email address and a password. The email is the login name, so it must be unique and verified, and changing it means verifying again. Apps don't key on the email, though. They key on a stable person ID, so an email change never renames anyone inside Jira or anywhere else.

A **character** is an EVE character the system knows about. A character may belong to no person, for example a killboard visitor. Public sites log characters in directly through EVE SSO. Each site has its own EVE client and keeps its own sessions, and no shared IdP sits in front of them.

A person **links** a character by proving control of it through EVE SSO while logged in. A character can be linked to at most one person. The same link also lets the person log in with that character. Discord and future providers can be linked in the same way. Linked logins never create accounts: a login with an unlinked character is refused, and the user is asked to log in with email and link it. How these logins work is described in [05](05-directory-idp.md#external-logins).

Every EVE login asks for all scopes, so a login also refreshes or creates that character's token in the [token broker](02-token-broker.md).

## The directory

There is one directory for every in-house app. Apps read persons and groups from it and never compute group membership themselves. The groups are written there by the Role Authority (below).

Persons and their groups are in the directory before their first login anywhere. Nobody is provisioned on the fly.

Each person's directory entry lists their linked character IDs and their main character, because in-house apps such as the ESI viewer, fleet tools and SRP need them. Intelligence alts are left out (see [08](08-esi-viewer.md#intelligence-alts)).

When someone leaves, they keep their account and lose their roles. Third-party apps pick up the change at their next group sync. Sessions that are already open last until they expire, so privileged apps should use short session lifetimes.

## How apps connect

- **SAML** is the default for in-house apps: no state, nothing to refresh.
- **OIDC** is for external apps, apps that need refresh tokens, and calls between our own services (client tokens).
- **LDAP** is for lookups and group sync. In-house apps bind with a service account. Jira DC and Confluence DC connect directly.
- **Push integrations** cover apps without any of the above: a Discord bot for invites and role sync, an HTTP authenticator for Mumble, webhooks.
- **SCIM** is outbound only, added as an adapter if some app ever needs it. Nothing provisions accounts into our IdP. HR feeds facts; it doesn't provision accounts.

**Mumble** has neither SAML nor OIDC. The user logs in on the website (so MFA applies), clicks "Connect to Mumble", and gets a short-lived one-time credential. Clients that stay connected for a long time can use an app password instead: generated on the account site, valid only for Mumble, and revocable. The server checks groups on connect and re-authenticates periodically. Kicking users the moment their roles are revoked is postponed for now.

## The Role Authority

Who belongs to a role depends on many things: a character's corp, its in-game roles, features derived from assets or skills, HR positions, grants made by hand, and membership in other roles. SeAT recomputed all of this on a timer, so people waited minutes or hours for access. The Role Authority recomputes only what changed, as soon as it changes.

### How it works

All inputs are **facts**, held in one store that emits an event on every change. They come from:

- the warehouse: corp and alliance affiliation, in-game roles, and asset or skill features (current values only);
- the coalition registry and [Standings](09-standings.md): which entities are members, blue, hostile and so on;
- HR: positions and employment state;
- [Counter Intelligence](04-counter-intelligence.md): `ci_flagged`;
- manual and self-service grants;
- other roles' memberships.

**Rules** are versioned data, written in CEL. A rule reads facts and decides who is in a role. Rules work on precomputed features, never on raw ESI tables. The engine knows which facts each rule reads. When a fact changes, it re-evaluates only the affected rules for only the affected people, and the result lands in seconds.

A rule may depend on other roles, but those dependencies must not form a cycle. Cycles are rejected when the rule is saved.

**Role families.** A single rule can produce one group per entity. For example, `corp[<id>].director` creates a director group for every registered corp. Groups appear and disappear as entities join and leave.

**Results flow one way.** They go from the Role Authority to the directory, to push integrations, and into SAML/OIDC claims at login. Grants made by hand are inputs to rules, so a sync never wipes them out.

Every membership records the rule version and the facts that produced it. "Why did I lose access?" always has an answer.

**Intelligence alts** never feed ordinary rules. If they did, a sudden role change could reveal the alt. Only rules owned by CI or the intelligence master may read their facts.

### Interest groups

People can join interest groups (SIGs) themselves, but whether they may join can depend on other roles. When someone asks to join, eligibility is checked immediately. If they qualify, they're in. If they later stop qualifying, their membership is suspended rather than deleted, and it comes back automatically if they qualify again.

### Member status

`member` is the most important role and the one that changes most, because alliances come and go. It depends on two things that change at different speeds:

- **The coalition registry**, meaning which alliances and corps are members. Coalition leads edit it, and edits take effect within seconds.
- **ESI affiliation**, meaning which corp each character is in. ESI's cache sets the limit here. The warehouse refreshes affiliation for every character through the public bulk endpoint, at about 1000 requests for 1M characters.

A character counts as a member when its corp or alliance is in the registry and its owner is in `good_standing`. `good_standing` is a reusable condition that excludes anyone flagged by Counter Intelligence.

### HR

HR manages job positions. It feeds facts (positions, employment state) into rules, it uses the directory like any in-house app, and it emits joiner/mover/leaver events.

## Role designations

Some powers don't fit the "which data can this role see" model: using the CI system, seeing intelligence alts, reading audit logs. Instead of building special roles into the code, the root owner can attach a **designation** to any ordinary role:

- `ci`: may use the CI system, may be configured to see intelligence alts, and may designate them.
- `intelligence-master`: may be configured to see intelligence alts, and may designate them.
- `intel-self-designation`: may mark one's own alt as an intelligence alt, even when it's in a member or blue entity.
- `audit`: may read audit logs for the role's part of the org tree.

Who holds a designated role is still decided by that role's rules. What the role can see is still set in the visibility configuration. A designation only unlocks things no other role can be configured to do.

## Org tree and delegation

The organization is a tree rooted at the coalition, and its shape is free. Alliances, corps and SIGs sit wherever they make sense; a SIG can hang directly off the coalition. Delegated admins work inside their own subtree and nowhere else. Coalition leads maintain the member registry. SIG managers create SIG roles and write their eligibility rules.

Any node in the tree can carry **conditions**, and every role below that node inherits them. Delegates can add conditions in their subtree but can never remove an inherited one. The root owner puts `good_standing` at the top of the non-public part of the tree. Every non-public role, including those delegates create later, therefore excludes flagged people automatically. Public roles live in a branch without that condition.

Only the root owner can change global policy, such as what `member` means. Every change to rules or the registry is versioned and audited, together with how many memberships it changed.

## Deployables

These deploy separately, so that maintaining one doesn't take down the others:

- **Workforce IdP**: authentication and sessions for persons, and SAML/OIDC to apps. Everything depends on it.
- **Directory** and **directory sync**: the sync service is the only thing that writes to LDAP.
- **Account site**: registration, password and MFA management, linking characters and other logins, consent to EVE tokens, app passwords, joining SIGs.
- **Login bridge**: EVE and Discord logins for the workforce IdP.
- **Token broker**: all external tokens. It must keep running while the account site is down.
- **Discord integration**: server invites and role sync.
- **Admin console**: rules, org tree, registry, delegation.
- **Role Authority**: facts, evaluation, and the claims endpoint the IdP reads at login.
