# Token Broker

The token broker holds every credential we get from an external provider: EVE SSO tokens, Discord OAuth tokens, the Discord bot token, and the client secrets of all our OAuth applications. Nothing else in the system ever sees a refresh token. The IdP doesn't hold these tokens either; it only uses external providers to check who is logging in.

The broker belongs with the account site, but it deploys separately. Maintenance on the account site must not stop ESI fetching or the Discord integration.

## What it does

- **Stores secrets.** Everything lives in its own encrypted database, which only the broker can reach. An SQL injection or a leak anywhere else in the system yields token IDs, never usable credentials.
- **Exchanges codes.** Sites send the authorization code to the broker, and the broker turns it into tokens with the right client secret.
- **Leases access tokens.** A consumer asks "give me an access token for token #N" and gets one valid for about 20 minutes. The broker caches each access token for its lifetime, so all consumers share it and EVE SSO is called as rarely as possible.
- **Audits.** Every lease is logged: who, which token, when.

## Registered clients

Every OAuth client we operate is registered with its secret:

- the EVE clients used by the account site and the login bridge (several, for availability and branding);
- each public site's own EVE client;
- SeAT's EVE client, during migration;
- the Discord application.

## EVE tokens

A token is identified by its character, its client and its scope set. One character can therefore have several tokens, for example from logging in through different sites. Capacity planning assumes 1M tokens, all with full scopes.

A token doesn't need an owner. A token minted on the killboard belongs to a character that no person has linked. It is stored, refreshed and used by the warehouse like any other. If the character is linked later, its tokens simply show up on that person's account.

## How sites log in

Public sites run their own EVE SSO login with their own client and keep their own sessions. They share no state with each other or with the rest of the platform. The only thing the broker adds is that the site never handles the refresh token:

```
Site ──redirect──▶ EVE SSO (the user sees CCP's page)
Site ◀──code────── EVE SSO
Site ──code──▶ Broker ──exchange──▶ EVE SSO
Site ◀── character ID, owner hash, access token ── Broker   (the refresh token stays with the broker)
```

The site creates its own session for the character, and reads any ESI data it needs from the warehouse. The broker records the token's owner hash. When the owner hash changes, the character has been sold or transferred, so the change is passed to [Counter Intelligence](04-counter-intelligence.md#monitoring).

## Exactly one refresher

EVE SSO may issue a new refresh token every time one is used. If two parties refresh the same token, one of them is left holding a dead token. That is why the broker is the only party that ever refreshes:

- **One refresh at a time per token.** A Valkey lock makes concurrent lease requests queue up behind the refresh already in progress. The guarantee itself comes from a row lock on the token in the broker's own database, held for the whole refresh. A lost Valkey lock here would destroy a token, not just waste a request, so the database has to be the real guard.
- **The new refresh token is saved in the same transaction as the refresh.** If that write fails, the token is lost, which is worth an alert.
- **Refresh is lazy.** It happens only when someone asks for a lease. No background loop walks through 1M tokens.
- **Revocation is detected.** When EVE SSO answers `invalid_grant` (the user revoked access, or the character changed hands), the token is marked dead and a fact is emitted. The warehouse stops using the token.
- **Access tokens stay inside the broker.** They are cached in its own memory and store, never in the shared Memcached.

## Who uses it

- **The warehouse** is the main consumer. Its fetches go through its own scheduler, cache and pacing.
- **The Discord integration** uses a user's `guilds.join` token plus the bot token to invite people and sync their Discord roles when their memberships change.
- **The login bridge** sends every login code to the broker, so each EVE login refreshes or creates that character's token.
- **The account site** lists linked accounts and revokes them on request.
- **Special-case apps** that truly can't go through the warehouse can lease raw EVE access tokens and call ESI directly. Each one is registered and documented, and has its own lease quota, because its traffic bypasses the warehouse's caching and pacing.

## Keys

With no cloud key management, the master key lives only on the broker host, with an offline backup. Each stored value is encrypted with a data key, and the data key is encrypted with the master key. Rotating the master key therefore doesn't mean re-encrypting every token. The rotation procedure still has to be written.
