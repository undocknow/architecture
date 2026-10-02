# Roadmap and Migration

We are replacing a heavily patched SeAT v3. One team builds everything, so the phases run one after another. From phase 1 onward SeAT reads ESI data through the new warehouse, so it keeps working while it is replaced piece by piece.

## Rules for the migration

- **Run in shadow first.** Each replacement first runs alongside SeAT, and its output is compared with SeAT's before it takes over.
- **Lose nothing.** Tokens, character groupings, the blacklist and SIG memberships are all imported. Nobody has to re-link a character because the system changed.
- **Keep a way back.** Every phase can be rolled back until its exit criteria are met.
- **One refresher per token, even mid-migration.** If SeAT and the broker ever both refresh the same token, it dies.

## Phases

**0. Foundations and proof of concept.** Run the Docker proof of concept ([06](06-stack.md#proof-of-concept)). Set up the repository, shared Rust crates, CI/CD, monitoring and backups. Take an inventory of our SeAT fork: everything that touches `refresh_tokens`, every ESI route it calls, its SSO callback, and where SIG memberships live. *Done when* every proof-of-concept risk is resolved or has an accepted fallback.

**1. Token broker, warehouse core, SeAT cutover.** Build the broker, and the warehouse's scheduler, pacing, locking and passthrough mode. Then patch SeAT to read through the warehouse and move its tokens into the broker in one maintenance window ([change spec](changes/seat-v3-token-cutover.md)). *Done when* all SeAT ESI traffic goes through the warehouse, the broker is the only refresher, and no tokens were lost.

**2. Fact data.** Build proper data models, with history, for what roles depend on: affiliation, corp and alliance history, in-game roles and titles, and the asset and skill features SeAT's roles use today. *Done when* these facts stay current within ESI's cache times and are published on NATS.

**3. Identity platform.** ZITADEL, 389-ds, directory sync, the login bridge, the account site and the admin console. SeAT users can claim their accounts (below). *Done when* a person can register or claim an account, log in by email or EVE, link characters, and appear in LDAP.

**4. Role Authority and CI core.** The rule engine, `member` status, interest groups and the claims endpoint. CI's entries and the `ci_flagged` fact come now rather than later, because SeAT enforces its blacklist today and the new roles must respect it even in shadow mode. SeAT's roles are translated into rules, and a report compares the resulting memberships with SeAT's. *Done when* every remaining difference is understood and intended, and changes show up within seconds.

**5. App cutover.** Move apps one at a time, least critical first: wiki, Discourse, Matrix, Confluence, Jira, Mumble, Discord. Each keeps a short rollback window. *Done when* no app takes identity or roles from SeAT.

**6. Full warehouse and ESI viewer.** Build the remaining data models, GraphQL, ClickHouse and Metabase. Ship the [ESI viewer](08-esi-viewer.md), which replaces SeAT's character and corporation pages. *Done when* people use the viewer instead of SeAT.

**7. SeAT retirement and full CI.** Finish CI: monitoring, notes, Jira-backed requests, alerts. After the claim deadline, SeAT becomes read-only, is archived and is shut down. The SeAT sync adapter and SeAT's temporary key are removed.

**8. More modules.** Fleet tools, then SRP, then the rest (HR, logistics, killboard and other public sites). Each gets its own design document before work starts.

## Moving the tokens

EVE refresh tokens belong to the client that issued them and may change on every refresh. The broker can take over SeAT's tokens only by holding SeAT's client secret and becoming the only refresher. We do this by making SeAT stop talking to ESI altogether:

- SeAT's ESI client points at the warehouse, authenticates with a temporary static key, and never refreshes.
- SeAT's SSO callback hands codes to the broker instead of exchanging them itself.
- SeAT's `refresh_tokens` table stays, because much of SeAT reads it. A small adapter on our side keeps its status columns (valid or dead, scopes, owner hash) in sync with the broker. Only placeholder values replace the secrets.

The switch happens in one maintenance window:

1. stop SeAT's workers;
2. import every token into the broker;
3. have the broker refresh each token once;
4. restart SeAT in its new mode.

Rolling back means exporting the broker's *current* tokens back to SeAT, because SeAT's old copies may already have been replaced. The details are in the [change spec](changes/seat-v3-token-cutover.md).

If routing SeAT through the warehouse turns out to be blocked, the fallback is that SeAT keeps calling ESI but gets its access tokens from the broker. That is equally safe, but SeAT's traffic then stays outside our pacing.

## Moving the users

A SeAT v3 account is a group of characters. Email is optional and was never verified, so we don't trust it, and there are no passwords.

Each SeAT group is imported as an **unclaimed account** that keeps its characters and main character. It lives in the account site, not in ZITADEL, because it has no trusted email. When a user first logs in with EVE, the bridge recognizes the character. The account site then asks for an email (always verified; SeAT's address is at most offered as a suggestion) and a password. The new person starts with all of their old characters already linked, and their tokens become theirs.

Until they claim, users keep using SeAT for whatever hasn't moved yet. Before SeAT is retired, we announce a claim deadline. Characters never claimed keep their tokens without an owner, and their data stays in the warehouse.

## Moving the blacklist

SeAT marks blacklisted characters with `active = 0` on its per-character user rows, and in our SeAT that bit means nothing else. Each such character becomes a CI entry with "imported from SeAT" as the reason, and CI fills in the real reasons over time.

The new model is stricter. A flagged character flags its whole person, and so every alt. A SeAT group that had one inactive alt among active ones therefore becomes fully flagged. The import produces a list of people whose status widens, for CI to review before phase 4's comparison is accepted.

## Moving SIG memberships

SIG memberships in SeAT are imported in phase 4 as if each person had joined the SIG themselves. Eligibility is checked straight away. Those who still qualify are active; those who don't are suspended and listed in a report for SIG managers, rather than silently dropped. Where these memberships live in our SeAT will be found during the phase 0 inventory.
