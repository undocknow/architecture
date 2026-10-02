# Overview

## Context

The system serves one coalition. The coalition is made up of many alliances, and each alliance sets some of its own rules. A few public sites, such as a killboard, sit alongside the coalition's back-office. Everything has a single owner, who delegates administration down the org tree.

The long-term scope is the whole back-office: identity, ESI data, an ESI viewer, fleet tools, SRP, HR, logistics. We build it in phases. The first two pieces are the identity platform and the ESI warehouse.

Today the coalition runs on SeAT v3, with about 70k users and 200k EVE tokens. Existing tools like SeAT and Alliance Auth don't fit where we want to go, so this is a new build and they serve only as references.

## Constraints

- **Self-hosted, no cloud services.** No managed IdP and no cloud key management. We hold our own encryption keys.
- **Scale.** Plan for 200k users and 1M EVE tokens, and assume every token carries every scope.
- **Possibly a single host.** Nothing may depend on replicas. Isolation comes from separate processes and databases, not separate machines.
- **Data is kept forever.** Tables that keep growing are partitioned automatically.
- **No just-in-time provisioning.** Accounts and group memberships exist in the directory before anyone logs into an app.
- **No Java** in our stack, because of its memory use.

## Principles

**One directory for all in-house apps.** No app keeps its own copy of an account. That way, removing someone's roles takes effect everywhere.

**Persons and characters are separate.** People log into enterprise apps as persons, and into public sites as characters. Linking the two is optional.

**Roles are computed, not assigned by hand.** A role's members are derived from facts and recomputed the moment a fact changes. Every membership can be explained. Nobody waits for a polling cycle.

**All ESI traffic goes through the warehouse.** Apps read ESI data from it live rather than keeping their own copies. Because every token is full-scope, access control happens when data is read from the warehouse, not through ESI scopes.

**Credentials are isolated.** Refresh tokens and client secrets sit in their own encrypted store, behind one service. Other data isn't encrypted at rest.

**Components fail and get maintained independently.** The IdP, the account site, the token broker and the admin console are separate deployments.

**People who leave keep their accounts.** Their roles are removed; their accounts are not deleted.

## How the pieces fit

```
 Persons ──login──▶ Workforce IdP (ZITADEL) ──SAML/OIDC──▶ apps
                      │        ▲
                      │        └── login bridge (EVE/Discord logins, link lookup)
                      ▼
              Directory (389-ds, read model) ◀── directory sync ◀── Role Authority
                                                                      ▲
                                                facts ────────────────┘
                                                  ▲          ▲            ▲
                                    ESI warehouse ┘   Standings    Counter Intelligence
                                          ▲
                              leases ─────┘
                                Token broker (EVE + Discord tokens, isolated store)
                                          ▲
           Characters ──EVE SSO──▶ public sites, login bridge, account site (codes go to the broker)
```

- The **Role Authority** turns facts into group memberships, and directory sync writes them to LDAP.
- The **warehouse** calls ESI using access tokens leased from the **broker**.
- **BI** tools read a ClickHouse copy of the warehouse rather than the warehouse itself.

## App tiers

**In-house apps** are our own code, plus apps we fully operate such as Matrix. They log users in with SAML and read the directory over LDAP using their own service account.

**Self-hosted third-party apps** use SAML or OIDC, whichever they support, and LDAP where they can. This covers Jira DC, Confluence DC, Discourse, Canvas, Weblate, WordPress, Dokuwiki, Metabase and Shlink. Mumble uses an HTTP authenticator.

**External services**, such as Discord and partner sites, use OIDC or a push integration, for example a Discord bot that syncs roles.
