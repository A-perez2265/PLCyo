## 7. Story JSON Format (Contract B, v1.0)

> Task 0 copies this section into `docs/SCHEMA.md`. Same freeze rule as Section 6.

The compiler emits one JSON object. Expressions are serialized as trees so the runtime can evaluate them. Story text in the example below is abbreviated; real output contains the full strings, with escapes already decoded.

```json
{
  "version": 1,
  "title": "The Reaper's Dungeon",
  "start": "gate",
  "speakers": {
    "reaper": { "name": "The Reaper", "sprite": "assets/sprites/reaper.png" }
  },
  "items": {
    "torch": { "name": "Rusty Torch" },
    "key":   { "name": "Bone Key" }
  },
  "vars": { "courage": 0, "met_ghost": false },
  "scenes": {
    "gate": {
      "kind": "scene",
      "art": "assets/art/dungeon_gate.png",
      "body": [
        { "type": "say", "speaker": "reaper", "text": "Another soul at my door." },
        { "type": "say", "speaker": null, "text": "Iron gates groan open into darkness." },
        {
          "type": "choice",
          "text": "Take the torch from the wall",
          "guard": null,
          "actions": [
            { "op": "give", "item": "torch" },
            { "op": "add", "var": "courage", "value": { "t": "num", "v": 1 } }
          ],
          "goto": "corridor"
        }
      ]
    },
    "corridor": {
      "kind": "scene",
      "art": "assets/art/corridor.png",
      "body": [
        {
          "type": "if",
          "cond": { "t": "has", "item": "torch" },
          "then": [ { "type": "say", "speaker": "reaper", "text": "Light. How quaint." } ],
          "else": [ { "type": "say", "speaker": "reaper", "text": "Darkness. Lovely." } ]
        },
        {
          "type": "choice",
          "text": "Follow the cold whisper",
          "guard": { "t": "not", "e": { "t": "var", "name": "met_ghost" } },
          "actions": [],
          "goto": "ghost"
        },
        {
          "type": "choice",
          "text": "Jump the pit",
          "guard": { "t": "cmp", "op": ">=", "l": { "t": "var", "name": "courage" }, "r": { "t": "num", "v": 2 } },
          "actions": [],
          "goto": "boss"
        }
      ]
    },
    "ghost": {
      "kind": "scene",
      "body": [
        { "type": "action", "op": "set", "var": "met_ghost", "value": { "t": "bool", "v": true } },
        { "type": "say", "speaker": null, "text": "A pale shape drifts out of the wall." },
        { "type": "choice", "text": "Go back", "guard": null, "actions": [], "goto": "corridor" }
      ]
    },
    "death_coward": {
      "kind": "ending",
      "ending_kind": "death",
      "art": "assets/art/death.png",
      "body": [ { "type": "say", "speaker": "reaper", "text": "Cowards die slowly." } ]
    }
  }
}
```

### Field reference

| Path | Type | Notes |
|---|---|---|
| `version` | int | Always `1` |
| `title` | string | From `story "..."` |
| `start` | string | Id of a scene |
| `speakers.<id>.name` | string | Display name |
| `speakers.<id>.sprite` | string, optional | `assets/sprites/<file>`; omitted if the speaker has none |
| `items.<id>.name` | string | Display name |
| `vars.<id>` | int or bool | Initial value |
| `scenes.<id>.kind` | `"scene"` or `"ending"` | Scenes and endings share this map |
| `scenes.<id>.ending_kind` | `"death"`, `"victory"`, `"neutral"` | Endings only |
| `scenes.<id>.art` | string, optional | `assets/art/<file>`; omitted if there is no `art` statement |
| `scenes.<id>.body` | array of body nodes | In source order |

**Body nodes (`type`):**
- `say`: `speaker` (id, or `null` for the narrator), `text`.
- `if`: `cond` (expr), `then` (body nodes), `else` (body nodes; `[]` when there is no `else`).
- `choice`: `text`, `guard` (expr, or `null` when unguarded), `actions` (array of actions), `goto` (id).
- `action`: a scene-level action. Same fields as an action object (below), plus `"type": "action"`.

Ending bodies contain only `say` nodes.

**Actions (`op`):** `give` and `take` (`item`); `set`, `add`, and `sub` (`var`, `value` expr) for `=`, `+=`, and `-=`.

**Expressions (`t`):** `num` (`v`), `bool` (`v`), `var` (`name`), `has` (`item`), `cmp` (`op`, `l`, `r`), `and` (`l`, `r`), `or` (`l`, `r`), `not` (`e`).

**Rules:**
- Optional fields are **omitted, never `null`**. The only `null` values allowed are `say.speaker` (narrator) and `choice.guard` (unguarded).
- Key order is not significant. Golden files are compared as parsed JSON.

---
