# SeAT v3 Token Cutover: Change Spec

For whoever patches our SeAT v3 fork. Background is in the [roadmap](../07-roadmap.md#moving-the-tokens).

The descriptions of SeAT's current behavior below come from upstream SeAT v3, from memory. Our fork is heavily patched, so check every point during the inventory (step 0) before writing code.

## Goal

After the cutover, the [token broker](../02-token-broker.md) is the only party that refreshes EVE tokens. EVE SSO can replace a refresh token every time it is used, so a second refresher would slowly kill tokens. SeAT must therefore stop doing three things:

- refreshing tokens;
- calling ESI directly (it calls the warehouse instead, which speaks the same API);
- exchanging SSO codes itself (the broker does that).

## Approach: change who writes `refresh_tokens`, not who reads it

Much of SeAT reads `refresh_tokens`: job dispatch, scope checks, choosing a corp token by in-game role, the token status pages, owner-hash handling. Rewriting all of that is the expensive part, so we leave it alone. The table keeps its rows and meaning: one row per character with a token, plus its scopes, owner hash, expiry and the `deleted_at` soft-delete used for dead tokens. Only the secret columns change. `refresh_token` and `token` hold placeholders that nothing uses.

The code that *writes* the table does change:

- **Writing back refreshed tokens after ESI calls** is switched off.
- **The SSO callback** sends the code to the broker, and fills the row from the broker's answer.
- **Marking tokens dead on `invalid_grant`** disappears from SeAT, since SeAT no longer refreshes. A small adapter on our side, written in Rust rather than in SeAT, takes over. It listens to broker events (token created, scopes changed, owner changed, token dead or revived) and updates the rows in SeAT's database.
- **Our own patches** that touch tokens are adapted, or confirmed to only read.

Putting the adapter on our side keeps the PHP patch small. SeAT is on its way out, so we'd rather add a temporary adapter than more PHP.

## Changes in SeAT

Everything sits behind one switch, `SEAT_TOKEN_MODE=legacy|broker`, so rolling back is a configuration change plus a token export.

**1. ESI calls go to the warehouse.** Point eseye's host settings at the warehouse. Instead of a character's access token, every request carries a static key for SeAT's app grant in the warehouse. The IdP doesn't exist yet in phase 1, so this key is temporary and goes away with SeAT. The simplest hook is probably a custom authentication class that sends the key and reports an expiry far in the future, so eseye never tries to refresh. The warehouse works out from the URL which character or corp is meant, and uses whichever broker token can serve it.

**2. No refresh, no write-back.** Remove or bypass the code that saves refreshed tokens after ESI calls. In upstream SeAT that code lives in the base ESI job. Any check of `token` or `expires_on` that would trigger a refresh becomes a no-op in broker mode.

**3. The SSO callback uses the broker.** SeAT keeps its own EVE client and callback URL, but its client secret moves to the broker. In the callback, post the code to the broker instead of letting Socialite exchange it. The broker returns the character ID, name, owner hash, scopes and expiry. SeAT's user and group logic then runs as before, writing the row with placeholders, and the adapter reconciles it shortly after.

**4. Token status comes from the broker.** In broker mode SeAT must not soft-delete tokens itself. The adapter sets and clears `deleted_at` and keeps scopes and owner hash current. A 403 from the warehouse (no usable token, or missing in-game role) must not be read as "token dead". Check how our SeAT handles 403 today.

**5. Backoff keeps working.** SeAT already slows down based on ESI's error-limit headers. The warehouse sends the same headers, reflecting SeAT's share of the request budget, so SeAT backs off when asked to. SeAT's own response cache can stay, and ETag/304 works as before.

## What the warehouse must support for SeAT

- Versioned paths exactly as eseye calls them (`/v4/…`, `/latest/…`, `/legacy/…`), including the `datasource` parameter.
- ESI-identical errors and status codes, `X-Pages`, ETag/304, `Expires` and `Last-Modified`.
- Passthrough for every route SeAT calls that has no data model yet.
- An app grant that lets SeAT read everything it reads today, through the static key. This is a documented, temporary exception to the rule that apps use client tokens.
- A request budget for SeAT, reflected in the error-limit headers.

## Step 0: inventory

Before writing any patch, search our whole fork, including every package, plugin and patch, for:

- uses of the `RefreshToken` model or the `refresh_tokens` table, separating reads from writes;
- uses of `refresh_token`, `->token`, `expires_on`, `character_owner_hash`, and soft-delete queries on tokens;
- places that construct eseye or its authentication, and any direct calls to `esi.evetech.net` or `login.eveonline.com` (including `/oauth/verify`);
- the SSO callback and the Socialite configuration;
- performance patches that cache or batch token access.

Every write must be covered by changes 2–4 or explicitly justified. Reads need no change.

## Cutover window

1. Register SeAT's client in the broker, create the warehouse grant and key, and deploy the adapter (idle).
2. Stop SeAT's workers and scheduler, and close SSO login.
3. Export `refresh_tokens`, including soft-deleted rows, and import them into the broker.
4. Have the broker refresh every token once. Failures were most likely dead already, and the adapter will mark them.
5. Deploy the patched SeAT in broker mode, and start the adapter so it reconciles every row.
6. Overwrite the secret columns with placeholders. This can wait until the rollback window closes, but the old values must never be used again, because they may already have been replaced.
7. Restart workers and scheduler, and reopen SSO.
8. Watch warehouse request rates, SeAT job failures, broker refresh errors and the number of dead tokens.

**Rollback** within the window: stop SeAT's workers, export the broker's *current* tokens back into `refresh_tokens`, switch to legacy mode, and remove SeAT's client from the broker *before* SeAT's workers start again. Two refreshers must never run at the same time.

## Testing

Run the whole procedure, rollback included, on a staging copy of SeAT with a test EVE client and a few test characters. Check that:

- character and corp jobs succeed through the warehouse;
- adding a character through SSO works;
- revoking a token on CCP's side shows up as invalid in SeAT;
- owner changes are handled.

Then load-test the normal job schedule against the warehouse, and confirm that SeAT backs off when the warehouse's headers tell it to.
