# ESI Viewer

The ESI viewer replaces SeAT's character and corporation pages. People with the right roles can look up skills, assets, wallets, contacts, mail and everything else ESI offers about characters and corps. It is the first back-office module, delivered in phase 6. Beyond what SeAT did, it shows a person across all their alts, searches across many characters at once, and logs every view of sensitive data.

## Architecture

The viewer is a Vue frontend with a thin Rust backend. Users log in with SAML through the workforce IdP. The backend calls the [warehouse](03-warehouse.md) with its own client token and says which person is asking. The viewer stores no ESI data.

All access decisions happen in the warehouse, in a policy module for the viewer. The viewer only passes along who is asking. A bug in the viewer's UI therefore can't show data the policy wouldn't allow. Browsers never call the warehouse directly.

Every panel shows how old its data is. A refresh button asks the warehouse to fetch now, limited per user so nobody can drain the ESI budget.

## Classifying characters

Who can see a character depends on where that character is. Each corp and alliance has a relation to the coalition, provided by [Standings](09-standings.md): **member**, **sub-corp** (a designated alt corp or alliance belonging to a member, whose characters get limited access), **blue**, **neutral** (the default) or **hostile**.

Leadership roles apply to *persons*. If a corp's director can see you because one of your characters is in their corp, the director can also see your other characters, classified by where each one is:

- characters in any coalition member entity count simply as **members**, not as alts;
- characters in a sub-corp are **sub-corp alts**;
- characters in blue, neutral or hostile entities are **blue**, **neutral** or **hostile alts**. Hostile alts are their own class but behave like neutral alts unless configured otherwise;
- **intelligence alts** are designated by hand and override all of the above.

## Who sees what

What each role sees is configuration, not code. The configuration is a matrix: for each role and each character class, it lists which data sections are visible. Sections are fine-grained (skills, assets, wallet journal, contracts, mail, contacts, notifications, location, and so on). Mail is a section like any other.

The defaults follow current practice:

- **Corp directors** see everything about their corp's members and all their alts.
- **Other in-game corp roles** see the same people, with sections chosen per role in the configuration.
- **Alliance leadership**, meaning directors of the alliance's executor corp, sees everything about all alliance members and all their alts.
- **Roles designated `ci`** are configured like any other role. They are the only ones, together with `intelligence-master`, that may be given access to intelligence alts.
- **Roles designated `intelligence-master`** see intelligence alts and nothing else, unless the person also holds other roles.
- **Recruiters** see applicants who shared their data with them (below).

The root owner sets the coalition-wide defaults. An alliance can override them for its own subtree, but only to show less. Someone holding several roles sees the combination of all of them. Every change to the configuration is versioned and audited.

Two rules are fixed in code, so a configuration mistake can't break them:

1. Everyone always sees everything about their own characters.
2. Only roles designated `ci` or `intelligence-master` can ever be given access to intelligence alts.

Leadership grants come from **role families** in the Role Authority. The in-game Director role produces `corp[<id>].director`. Being a director in an alliance's executor corp produces `alliance[<id>].leadership`. These groups appear and disappear as corps join and leave.

## Intelligence alts

An intelligence alt must stay hidden from corp and alliance leadership, including the fact that it exists. It doesn't appear on the person page, in searches, in combined figures such as "can fly X" or total assets, or in counts. Hiding it only in the viewer isn't enough, so it is also:

- left out of the person's LDAP attributes and SAML/OIDC claims;
- kept away from ordinary role rules, so no role change can give it away;
- never named by Discord, Mumble or other integrations;
- known to CI monitoring, so its presence in a hostile corp never triggers alerts to leadership.

**Who can designate one:**

- CI and the intelligence master can do it at any time. They can even designate a character ID *before* anyone links it, so the character is hidden from the moment it shows up.
- The person themselves can do it while linking the character. This isn't allowed if the character is currently in a member or blue entity, unless the person holds a role designated `intel-self-designation`. Each self-designation notifies CI and the intelligence master, who can confirm or revoke it. Otherwise anyone could hide an alt from their own leadership.

The designation is checked inside the same operation that creates the link. The link becomes visible to the directory, the viewer and the Role Authority only after the designation is applied, so there is no moment when leadership could see the alt.

A designation stays in place when the alt changes corp. Only CI or the intelligence master can remove it. The owner always sees their own intelligence alts.

## Recruitment

A recruiter creates a share link with an expiry and, by default, access to all sections. The applicant opens the link and logs in with EVE; no account is needed, and the login stores a token for that character. The recruiter then sees all of the applicant's characters except intelligence alts:

- if the character belongs to a person, the share covers all of that person's characters, and stays current as they link or unlink alts;
- if it belongs to an unclaimed SeAT account, the share covers that account's characters;
- otherwise it covers just that character, and the applicant can open the link again with each further character.

The share ends at its expiry, or earlier if the applicant revokes it.

## Pages

- **Person**: all characters of a person that the viewer may see, with combined skills, assets, wallets and corp memberships. Hidden characters contribute nothing to the totals.
- **Character**: the familiar SeAT-style character sheet.
- **Corporation**: every corp endpoint ESI offers, as complete as the character sheet: members, assets, wallets and journals, structures, industry, contracts, killmails, starbases, customs offices, mining, standings.
- **Search**: across everyone the viewer may see, for example "who has skill X at level 5", "who owns a ship of type Y", "who is in corp Z".
- **Recruitment**: the recruiter's active and expired shares.

Heavy statistics belong in Metabase, not here.

## Audit

Every view of sensitive data is logged in the warehouse, together with the role or share that allowed it. Members can't see who viewed them. Logs are available to the root owner and to roles designated `audit`, limited to their part of the org tree.
