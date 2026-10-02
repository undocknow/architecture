# EVEployment Architecture

EVEployment is the identity platform and back-office for one EVE Online coalition: its member alliances, their corps, and a few public-facing sites run by the same owner. These documents describe where the system is headed. It is built and migrated in phases, starting from a heavily patched SeAT v3.

## Reading order

1. [Overview](00-overview.md): the context, the constraints, and how the pieces fit together.
2. [Identity](01-identity.md): persons and characters, the directory, how roles are computed, delegation.
3. [Token broker](02-token-broker.md): where EVE and Discord tokens live and who may use them.
4. [ESI warehouse](03-warehouse.md): the single gateway to ESI, plus storage, history and read APIs.
5. [Counter Intelligence](04-counter-intelligence.md): the blacklist system and sneak-in monitoring.
6. [Directory and IdP products](05-directory-idp.md): 389-ds, ZITADEL and the login bridge.
7. [Technology stack](06-stack.md): Rust, PostgreSQL, NATS and the rest, plus the proof of concept.
8. [Roadmap and migration](07-roadmap.md): build order, and how we get off SeAT.
9. [ESI viewer](08-esi-viewer.md): the first back-office module, and who may see whose data.
10. [Standings](09-standings.md): in-game contacts turned into relations and roles.

Supporting documents:

- [SeAT v3 change spec](changes/seat-v3-token-cutover.md): what our SeAT fork must change for the token cutover.
- [Decision log](decisions.md): every decision, with the reason behind it.
- [Open questions](open-questions.md): what isn't settled yet.

## Vocabulary

- **Person**: an account, registered with email and password. A person can have no EVE characters at all.
- **Character**: an EVE character the system knows about. It may belong to no person, for example someone who only uses the killboard.
- **Link**: a verified connection between a person and a character.
- **EVE token**: a refresh/access token pair from EVE SSO, used only to call ESI.
- **Client token**: a token our own IdP issues to one of our services so it can call another. It is never an EVE token.
- **Fact**: anything role rules can depend on: a character's corp, an in-game role, an HR position, membership in another role, and so on.
- **Org tree**: the hierarchy of coalition, alliances, corps and interest groups. It limits what delegated admins can change and what leaders can see.
