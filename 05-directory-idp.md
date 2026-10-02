# Directory and IdP Products

Our own services already handle the EVE-specific parts: computing roles, delegation, character links and external tokens. The products below only need to provide login protocols, credential storage, MFA and an LDAP directory.

## Where credentials live

Passwords, MFA and sessions live in the workforce IdP. The directory is a read model with no user passwords in it. No app needs them there:

- in-house apps log users in with SAML and use LDAP only with their own service account;
- Jira DC and Confluence DC log users in with SAML and use LDAP only to sync users and groups;
- other apps use SAML or OIDC;
- Mumble uses one-time tokens or app passwords.

Keeping passwords out of LDAP closes the only way MFA could be skipped. If an app that can only check passwords ever turns up, it gets its own app password, and only that app password's hash goes into LDAP. LDAP is reachable only from the internal network.

## Directory: 389 Directory Server

Exactly one service writes to the directory: **directory sync**. It applies changes as they arrive:

- persons from the IdP (created, email changed, disabled);
- group memberships from the Role Authority;
- character links from the account site.

The layout is flat:

- `ou=people`: one entry per person. `uid` is the stable person ID. The entry also has `mail`, a display name, and the linked character IDs and main character (intelligence alts excluded).
- `ou=groups`: one group per role, with namespaced names such as `coalition.member`, `corp-98000001.director` or `sig.logistics`. The org tree itself lives in the admin console; apps only need group names.
- `ou=services`: one bind account per app, with access rules limiting what each app can read.

Characters don't get their own entries; they appear only as attributes on persons. Large groups such as `coalition.member`, with roughly 150k members, are kept fast by sending only additions and removals, never whole-group rewrites. The memberOf and referential-integrity plugins handle the reverse lookups and clean-up.

We chose 389-ds because those plugins, password policy and replication come built in and mature, and its admin tooling is good. OpenLDAP would also have worked.

## Workforce IdP: ZITADEL (proposed)

ZITADEL is written in Go and runs on PostgreSQL. It does SAML, certified OIDC, MFA and passkeys. It can use an external login provider without letting that provider create accounts, and its login UI is a separate app we can re-theme. Keycloak would also fit, but it is Java. The lighter alternatives each miss something we need: Authelia and Kanidm have no SAML, and Authentik uses as much memory as Keycloak.

How we use it:

- **No on-the-fly accounts.** The account site creates each person in ZITADEL at registration, and directory sync puts them in LDAP before they log into anything.
- **Groups come from the Role Authority.** At login, a ZITADEL Action fetches the person's groups and characters from the Role Authority's claims endpoint and adds them to the SAML assertion or OIDC token. ZITADEL stores no roles itself.
- **Fast revocation.** When a critical role is removed, the Role Authority asks ZITADEL to end that person's sessions.

## External logins

ZITADEL's built-in external logins keep their own table of which external account belongs to whom. Using them directly would mean copying every EVE and Discord link into ZITADEL, giving two sources of truth that drift apart. So ZITADEL gets exactly one external provider: our **login bridge**, a small OIDC provider written in Rust. The bridge runs the real EVE or Discord login and looks up the link in our own store at the moment of login:

```
ZITADEL login page ── "Log in with EVE" ──▶ login bridge ──▶ EVE SSO (CCP's page, the only page the user sees)
                                                  │ code ─▶ token broker (stores the token)
                                                  │ character → person, looked up in the account site's link store
ZITADEL ◀── "this is person P"  or  "not linked" ─┘
```

Inside ZITADEL, each person has a single, fixed link to the bridge, created together with the person. Adding or removing EVE or Discord links never touches ZITADEL. New providers are added to the bridge only.

Every EVE login asks for all scopes and passes its code to the broker, so a login also refreshes the character's token. If the character isn't linked to anyone, the token is still kept (without an owner) and the login is refused with "log in with email to link this character". The account site can then link it using the token it just received, so the user doesn't see CCP's page a second time.

A side benefit: EVE SSO isn't quite standard OIDC, and the bridge absorbs that quirk. ZITADEL only ever talks to a standard OIDC provider.

**MFA after an external login is configurable.** Each provider has a setting for whether its login counts as MFA. CCP offers two-factor login, but it's optional for players and EVE's tokens don't tell us whether it was used. Trusting EVE logins as MFA is therefore a policy choice, not something we can verify. Sensitive apps and roles, such as the admin console, CI and directors, can demand our own second factor regardless of that setting.

## Public sites

Public sites don't use ZITADEL. Each one has its own EVE client, runs EVE SSO itself, and keeps its own sessions. They share only the warehouse and the broker's token storage (see [02](02-token-broker.md#how-sites-log-in)). Characters without a person never touch ZITADEL at all.

## Proof of concept

To be checked before we commit:

- SAML login to Jira DC, Confluence DC, Discourse and Canvas, with groups from the claims Action;
- ZITADEL with the bridge as its only external provider: account creation off, and logins mapped through the pre-created link;
- where per-provider and per-app MFA rules are enforced (ZITADEL itself, or the bridge and the Action);
- how fast the claims Action is, and what happens when the claims endpoint is down;
- memory and CPU use at 200k users.
