# Open Questions

## Needs a proof of concept

- Does ZITADEL work as planned: SAML into Jira, Confluence, Discourse and Canvas; logins through the bridge mapped to the right person; acceptable latency for the claims Action? What happens when the claims endpoint is down?
- Where are per-provider and per-app MFA rules enforced: in ZITADEL, or in the bridge and the Action?
- How much work does the Rust CEL implementation need before it can report which facts a rule reads?
- How much memory does the stack actually use on one host?

## Policy still to decide

- MFA for ordinary persons. The proposal: optional for most people, required for delegated admins and the root owner.
- Should EVE logins count as MFA by default?
- Session lifetimes for privileged third-party apps, since open sessions survive role removal.
- The default visibility matrix for in-game corp roles below Director.
- Default standing thresholds and labels.
- How to spot flagged people early. CI's alert framework exists; the policy will be revisited.

## Engineering details

- The procedure for rotating the broker's master key, and how its offline backup is handled.
- Lease quotas, and the registration process, for special-case apps that hold raw tokens.
- Valkey lock time limits and retry behavior, to be set by load testing.
- The warehouse policies for each app, starting with the ESI viewer and the killboard.
- How Mumble's HTTP authenticator works, and whether connected users can be removed the moment their roles are revoked (postponed).

## Migration

- Where SIG memberships live in our SeAT fork. To be answered by the phase 0 inventory.
