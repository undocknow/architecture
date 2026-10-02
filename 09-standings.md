# Standings

The Standings subsystem decides how the platform treats every corp and alliance: member, sub-corp, blue, neutral or hostile. Those relations drive the ESI viewer's character classes, roles such as "blue pilot", and CI's monitoring.

## Inputs

- **In-game contacts.** Each member alliance's contact list, plus the contact lists of individually configured corps (for example corps without an alliance). The warehouse fetches them with the tokens of characters who hold the needed in-game roles. We only read contacts and never write anything back to the game.
- **The coalition registry**, which says which entities are members.
- **Entries made in the Standings UI**: manual standings, overrides, and marking an entity as a **sub-corp** of a member.

## How standings combine

The coalition and each alliance have separate standing lists. The coalition's list consists only of entries made in our UI. An alliance's list is its in-game contacts plus its own entries.

Each alliance decides whether to **follow the coalition**:

- If it follows, then inside that alliance its own list wins, the coalition's list fills the gaps, and anything left is neutral.
- If it doesn't, only its own list counts, and everything else is neutral.

An alliance's standings affect only that alliance. When an alliance marks someone hostile, they are hostile in that alliance's viewer classes, roles and alerts, not coalition-wide. Because of this, a question about a relation always names a point of view: "what is entity E, as seen from alliance X?"

A standing value becomes a relation through configurable thresholds and labels, for example +5 or higher is blue, −5 or lower is hostile, and a "sub-corp" label means sub-corp. New relation types can be added through configuration. Membership always comes from the coalition registry and overrides any standing.

## Who can change what

Editing standings is a permission on ordinary roles, limited to a part of the org tree like everything else:

- alliance diplomats edit their alliance's list;
- coalition diplomats edit the coalition's list;
- the leadership of a member entity marks its own sub-corps;
- the root owner sets thresholds and labels.

Every edit is versioned and audited.

## Outputs

Standings publish facts, "entity E has relation R as seen from node N", whenever a contact sync or an edit changes something. Roles come from ordinary Role Authority rules that read these facts; Standings never writes groups itself.

Sub-corp access works through such roles. Characters in a sub-corp get roles like `sub-corp.member` instead of `member`, and each app's role mapping decides what that allows. For example, sub-corp characters get no normal Mumble access. Someone who also has a character in a real member corp is a member anyway, so the limits only affect people whose characters are all in sub-corps.
