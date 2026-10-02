# Counter Intelligence

Counter Intelligence (CI) is what other platforms call the blacklist. It is run by a separate team: the people holding roles designated `ci`. It has its own deployment, database and UI, because its contents (reasons, evidence, notes) are the most sensitive data in the platform after credentials.

## Effect on everything else

CI publishes one fact, `ci_flagged`, to the [Role Authority](01-identity.md#the-role-authority). Rules use it like any other fact. In practice they use it through the reusable `good_standing` condition, which the root owner places at the top of the non-public part of the org tree. Every non-public role therefore drops flagged people, and public roles such as killboard login don't. A new flag takes effect within seconds.

## Entries

An entry flags a subject. The subject can be:

- **a character**;
- **a person**, which covers all of that person's characters;
- **a corp or an alliance**, in one of two modes:
  - **current members**: anyone in it right now, who stops matching on leaving;
  - **historical**: anyone who has ever been in it. A historical entry can be limited to a time window, such as "in corp X between March and June". A character matches if any of their stints overlaps the window. For an alliance, the stint must be in a corp *while that corp was in the alliance*.

Flags spread through links. If any of a person's characters matches, the person matches, and so do all of their other characters.

Historical matching needs corp history for characters and alliance history for corps. Both are public and swept by the warehouse, which turns them into membership periods. A new entry is matched against all stored periods. A newly seen character is checked against all entries as soon as its history arrives.

Entries never expire on their own; CI lifts them by hand. Every change is versioned.

## Reports and appeals

Two kinds of request reach CI:

- **Suspect reports.** Anyone with the right role (by default, any member) can suggest someone for review. The reporter only sees whether their report was received, is under review, or is closed. They never see the outcome or the reasons.
- **Appeals.** Recruiters can appeal an existing entry.

Building a full ticket system into CI would duplicate Jira, so the work is split. Discussion, assignment and queues live in a restricted Jira project. CI keeps a small record of each request (subject, state, decision, who decided), and CI is the one that carries out decisions.

A request is filed in CI or through a Jira request type, and either way both sides get a linked record. When someone moves the Jira issue to *Approved* or *Rejected*, a webhook tells CI. CI checks that this person is allowed to decide (both systems use the same directory), applies the decision, and logs it. If they aren't allowed, CI moves the issue back.

Jira only knows what was copied into it, so two things keep it current. Each issue links to the live CI page for the subject. While a request is open, CI also posts new signals, affiliation changes and notes to the issue as they happen. The Jira connection sits behind an adapter, so Jira could be replaced later.

## Notes

Anyone with the right role can write a note about anyone, flagged or not. That makes notes a watchlist as well. Notes can't be edited, only appended to. Who can read them is role-based, and it can be scoped through the org tree, for example "leadership of the subject's alliance". By default CI and alliance leadership can read them. A note can be written with stricter visibility than the default, never looser.

## Monitoring

CI watches for flagged people getting back in, whether through an alt or because a corp accepted them without checking.

Some signals are certain, and they raise alerts directly:

- a flagged character is linked to a person, or a person links a flagged character (the flag spreads at once);
- a flagged character, or one matched by a corp or alliance entry, joins a member corp;
- a character's owner hash changes, meaning it was sold or transferred.

Other signals are only suspicious, and they open a case for CI to review rather than flagging anyone:

- persons sharing login IPs or devices (the IdP keeps login events indefinitely; CI only receives derived signals such as "A and B shared an IP");
- ISK, contracts or assets flowing to flagged characters;
- overlapping corp histories, shared contacts, mail patterns.

CI confirms a case (creating an entry), dismisses it, or keeps watching. Intelligence alts are known to CI. Their presence in hostile corps is expected, so signals about them go only to CI and the intelligence master, never to corp or alliance leadership.

The signal set will grow, so detection is built as a framework rather than a fixed list. It runs on the same event model as the Role Authority.

## Alerts

Each alert type has its own routing. By default an alert goes to CI and to the leadership of the corp or alliance involved, with recipients found through roles and the org tree. Delivery channels are pluggable: in-app, Discord, mail. How to spot people *before* they get in is still an open problem. For now we're building the routing framework and will revisit the policy.

## Who sees what

- **Recruiters** can look up whether a character or person is flagged. They see only yes or no, plus "contact CI", never the reason, and they never see open cases about suspected alts. They can file appeals.
- **Note readers** see notes according to the visibility rules above.
- **CI** sees everything: entries, reasons, evidence, notes, cases and requests.
- **The root owner** decides which roles carry the `ci` designation, and configures note visibility and alert routing.

Every read of a reason, piece of evidence, note or case is logged.
