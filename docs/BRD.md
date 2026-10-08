# PLCyo: Business Requirements Document (BRD)

**Version 1.0.** Supersedes draft v0.1. Section 14 lists everything that changed.

> **How to use this file with Cursor:** put it at `docs/BRD.md` in the repo and add the rules in Section 12 to `.cursor/rules`. **Do Task 0 first** (Section 8): it creates `docs/LANGUAGE.md` and `docs/SCHEMA.md` from Sections 6 and 7 of this file. After that, each person gives Cursor only *their* task from Section 8, one task at a time, and tells it to read this BRD, `docs/LANGUAGE.md`, and `docs/SCHEMA.md` first. Never ask Cursor to "build the whole project."

---

## 1. Project Summary

**PLCyo** is a small domain-specific programming language for writing interactive, text-driven story games. A writer describes scenes, choices, items, and variables in a `.plcyo` file. The **PLCyo compiler** checks the program and turns it into a story data file (JSON). A **browser runtime** loads that file and plays it as a pixel-art game.

**Demo game: "The Reaper's Dungeon."** The player descends into a dungeon. A pixel-art **Reaper** acts as a snarky dungeon master: he narrates, taunts, and guides. The player picks from options at each step, and the choices (plus items and hidden variables) decide the player's **fate**: a death scene, an escape, or a boss-fight outcome. There is no movement, no real-time controls, and no combat mechanics. Everything is narration, art, and choices.

**Course context:** Programming Languages class, 3-person team, 8 weeks, AI-assisted development (Cursor). This is a self-chosen project with no externally required format. Its purpose is to demonstrate PL fundamentals end to end:

- **Lexical analysis:** a hand-written lexer with source positions.
- **Parsing:** a recursive-descent parser for a grammar we designed, producing an AST.
- **Static semantics:** name resolution, type checking, and control-flow analysis (reachability, dead ends).
- **Code generation:** the AST is compiled to a JSON intermediate representation.
- **Interpretation:** the browser runtime is an interpreter for that IR.

---

## 2. Goals and Non-Goals

### Goals (v1)
1. A documented language grammar with a working compiler (lexer, parser, AST, semantic checker with checks C1-C14, JSON code generator).
2. Clear compile-time errors with file, line, and column.
3. A browser runtime that plays any valid compiled story: narration shown line by line with a typewriter effect, pixel-art scenes, a speaker portrait, choice buttons, an inventory panel, endings, restart.
4. A finished demo game (8-12 scenes, 3+ endings, at least one item-gated choice and one variable-gated choice) with hand-drawn pixel art and a snarky Reaper.
5. Documentation: language reference, a "how to write a story" guide, and a short final write-up and presentation (format defined by us in P3-6).

### Non-Goals (v1). Do NOT build these
- Movement, maps, grids, or real-time controls
- Combat systems, health bars, dice rolls, RNG (stretch only)
- Save/load, accounts, multiplayer, a backend server
- Sound/music (stretch only)
- A GUI story editor
- A scrolling text log or history view (each scene shows only its own lines)
- Loops, functions, user-defined types, or imports in the language
- Anything in the "Known limits" list (Section 6.5)

---

## 3. Users and Use Cases

| User | What they do |
|---|---|
| **Story author** (Person 3, classmates, anyone grading) | Writes a `.plcyo` file, runs the compiler, opens the generated game in a browser |
| **Player** | Opens `index.html`, reads the Reaper's narration, clicks choices, reaches an ending |
| **Developer** (us) | Runs `check`/`build`, reads error messages, runs tests |

**Primary flow:**
```
story.plcyo --> [ python -m compiler.plcyo build ] --> build/index.html (+ runtime files + assets) --> double-click --> play
```

---

## 4. Architecture and Decoupling Strategy

The three parts communicate **only through two contracts**. Neither side needs the other's code to make progress.

```
  Person 1                         Person 2                    Person 3
 (Compiler, Python)               (Runtime, JS/HTML/CSS)      (Story, Art, Tests, Docs)
 .plcyo --> story.json  ======>   story.json --> playable      writes .plcyo + draws art
        CONTRACT A: docs/LANGUAGE.md (grammar)  <-- P3 writes against it, P1 implements it
        CONTRACT B: docs/SCHEMA.md   (JSON)     <-- P1 emits it, P2 consumes it
```

- **Contract A, the language (`docs/LANGUAGE.md`)**: the grammar and semantics (Section 6). P1 implements it and P3 writes stories in it.
- **Contract B, the story JSON (`docs/SCHEMA.md`)**: the output format (Section 7). P1 emits it and P2 consumes it.
- **Golden files** link everything: `tests/golden/tiny.json` and `tests/golden/features.json` are the expected compiler output for `examples/tiny.plcyo` and `examples/features.plcyo`. P3 writes them by hand in Week 1. P1's tests assert that the compiler output equals the golden file (compared as parsed JSON, not as text). P2 develops the runtime against the golden files before the compiler exists.
- The demo game has no hand-written golden file because it changes constantly. At feature freeze (Week 7), its compiled output is snapshotted into `tests/golden/reapers_dungeon.json` as a regression test.
- **Any change to Contract A or B requires a PR approved by all three people.** This is the one rule that keeps us from breaking each other's work.

### Technology choices (decided)
| Part | Tech | Why |
|---|---|---|
| Compiler | **Python 3.11+**, standard library only (hand-written lexer + recursive-descent parser) | Team's strongest language; hand-written parser gives the best error messages and no dependencies |
| Compiler tests | `pytest` (the only dev dependency) | Standard and simple |
| Runtime | **Vanilla HTML + CSS + JavaScript** (no framework, no build tools, no npm packages) | Simple, runs anywhere, easy for AI to generate |
| Runtime tests | **Node 18+**, plain script using the built-in `node:assert` module | Repeatable without installing anything |
| Art | PNG pixel art, rendered with `image-rendering: pixelated` | Matches the aesthetic |

### How to run things
All commands run from the **repo root**.

| Command | What it does |
|---|---|
| `python -m compiler.plcyo check examples/tiny.plcyo` | Parse and check; print errors and warnings |
| `python -m compiler.plcyo build examples/reapers_dungeon.plcyo --out build/` | Check, then produce a playable game in `build/` |
| `python -m compiler.plcyo graph examples/reapers_dungeon.plcyo` | *(stretch)* Print a Graphviz DOT scene graph |
| `pytest` | Run compiler tests (configured by `pytest.ini`) |
| `node runtime/tests/engine.test.js` | Run runtime engine tests |
| `python -m http.server 8000` | Serve the repo for runtime dev mode (see below) |

`check` exits with code 1 if there are errors, 0 otherwise (warnings alone do not fail). `build` refuses to produce output if there are errors. `build` and `check` accept `--assets DIR` (default `assets/`) and `build` accepts `--runtime DIR` (default `runtime/`).

### Output packaging
`build` produces:
- `build/index.html`: a copy of `runtime/index.template.html` with the story JSON injected (see below).
- `build/engine.js`, `build/ui.js`, `build/style.css`: copied from `runtime/`. Plain `<script src>` and `<link>` tags work when the file is opened by double-click.
- `build/assets/art/` and `build/assets/sprites/`: copies of `assets/art/` and `assets/sprites/`.

**Injection.** The template contains exactly one tag:
```html
<script type="application/json" id="plcyo-story">__PLCYO_STORY__</script>
```
`build` replaces `__PLCYO_STORY__` with the story JSON, after replacing every `<` character in the serialized JSON with `\u003c`. The result is still valid JSON (the runtime parses it unchanged), and story text containing `</script>` or `<!--` can no longer break the page.

**Asset paths.** The emitter writes asset paths relative to the game folder: `assets/art/<file>` for `art`, `assets/sprites/<file>` for `sprite`. The runtime prefixes every asset path with a **base path**:
- **Built mode** (story JSON was injected): base is empty, because `index.html` sits next to `assets/`.
- **Dev mode** (story loaded with `?story=`): base defaults to `../`, because the template lives in `runtime/` and `assets/` is at the repo root. Override with `?base=...`.

**Dev mode.** Run `python -m http.server 8000` at the repo root and open `http://localhost:8000/runtime/index.template.html?story=../tests/golden/features.json`. The runtime picks its story like this:
1. If the `plcyo-story` tag contains JSON (not the placeholder), use it.
2. Otherwise, if `?story=` is present, `fetch()` that file.
3. Otherwise, show a visible "No story loaded" message. Never a blank screen.

---

## 5. Repository Structure

```
plcyo/
├── README.md
├── pytest.ini                # testpaths = compiler/tests, pythonpath = .
├── .cursor/rules/            # Cursor rules (Section 12)
├── docs/
│   ├── BRD.md                # this file
│   ├── LANGUAGE.md           # Contract A: grammar + semantics (P1 owns, all approve)
│   ├── SCHEMA.md             # Contract B: JSON format (P1 owns, all approve)
│   └── STORY_GUIDE.md        # how to write a PLCyo story (P3)
├── compiler/                 # P1 owns. No __init__.py at this level.
│   ├── plcyo/
│   │   ├── __init__.py
│   │   ├── __main__.py       # CLI: check / build / (graph)
│   │   ├── lexer.py
│   │   ├── ast_nodes.py
│   │   ├── parser.py
│   │   ├── checker.py        # semantic analysis
│   │   ├── emitter.py        # AST -> JSON
│   │   └── errors.py         # diagnostic type with line/col
│   └── tests/
│       ├── test_lexer.py
│       ├── test_parser.py
│       ├── test_checker.py
│       ├── test_emitter.py   # golden comparisons
│       ├── test_cli.py
│       └── invalid/          # P3: one .plcyo + .expected per check
├── runtime/                  # P2 owns
│   ├── index.template.html   # build injects the story JSON here
│   ├── engine.js             # pure game logic, no DOM
│   ├── ui.js                 # rendering, typewriter, buttons, inventory
│   ├── style.css
│   └── tests/
│       ├── engine.test.js    # node runtime/tests/engine.test.js
│       └── fixtures/         # deliberately broken stories for P2-7
├── examples/                 # P3 owns
│   ├── tiny.plcyo
│   ├── features.plcyo
│   └── reapers_dungeon.plcyo # the demo game
├── assets/                   # P3 owns
│   ├── art/                  # scene and ending backgrounds
│   └── sprites/              # Reaper and other portraits
└── tests/golden/             # expected JSON: tiny, features (Week 1), reapers_dungeon (Week 7 snapshot)
```

**Python packaging note.** Code inside `compiler/plcyo/` uses relative imports (`from .lexer import Lexer`). Tests import with `from compiler.plcyo import ...`, which works because `pytest.ini` sets `pythonpath = .`.

**Ownership rule:** you edit only your own folders. If you need a change elsewhere, open an issue or PR for the owner.

---

## 6. The PLCyo Language (Contract A, v1.0)

> Task 0 copies this section into `docs/LANGUAGE.md`. It is frozen at the end of Week 1. After that, changes need a PR approved by all three people.

### 6.1 Example program

```plcyo
# The Reaper's Dungeon (excerpt: vault_door and boss are omitted)
story "The Reaper's Dungeon" start gate

speaker reaper "The Reaper" sprite "reaper.png"

item torch "Rusty Torch"
item key   "Bone Key"
var courage = 0
var met_ghost = false

scene gate {
  art "dungeon_gate.png"
  say reaper "Another soul at my door. Try not to die too quickly."
  say "Iron gates groan open into darkness."
  choice "Take the torch from the wall" {
    give torch
    courage += 1
    goto corridor
  }
  choice "Walk in blind. Bravery!" {
    courage += 2
    goto corridor
  }
}

scene corridor {
  art "corridor.png"
  if has torch {
    say reaper "Light. How quaint. It shows you the pit, at least."
  } else {
    say reaper "Darkness. I do so enjoy this part."
  }
  choice "Search the bones" { give key  goto vault_door }
  choice "Follow the cold whisper" if not met_ghost { goto ghost }
  choice "Jump the pit" if courage >= 2 { goto boss }
  choice "Turn back" { goto death_coward }
}

scene ghost {
  met_ghost = true
  say "A pale shape drifts out of the wall and points deeper in."
  say reaper "Ignore him. He still owes me a soul."
  choice "Go back" { goto corridor }
}

ending death_coward {
  kind death
  art "death.png"
  say reaper "Cowards die slowly. You, however, will die now."
}
```

The `ghost` scene shows a **scene-level action** (`met_ghost = true` runs as soon as the scene is entered) and has no `art`, so the corridor background stays.

### 6.2 Grammar (EBNF)

```ebnf
program      = storyDecl { decl } ;
storyDecl    = "story" STRING "start" IDENT ;
decl         = itemDecl | varDecl | speakerDecl | sceneDecl | endingDecl ;

itemDecl     = "item" IDENT STRING ;
varDecl      = "var" IDENT "=" ( NUMBER | "true" | "false" ) ;
speakerDecl  = "speaker" IDENT STRING [ "sprite" STRING ] ;

sceneDecl    = "scene" IDENT "{" { sceneStmt } "}" ;
endingDecl   = "ending" IDENT "{" { endingStmt } "}" ;
endingStmt   = "kind" ( "death" | "victory" | "neutral" ) | artStmt | sayStmt ;

sceneStmt    = artStmt | sayStmt | ifStmt | choiceStmt | action ;
artStmt      = "art" STRING ;
sayStmt      = "say" [ IDENT ] STRING ;              (* IDENT = speaker; omitted = narrator *)
ifStmt       = "if" expr "{" { sceneStmt } "}" [ "else" "{" { sceneStmt } "}" ] ;
choiceStmt   = "choice" STRING [ "if" expr ] "{" { action } "goto" IDENT "}" ;
action       = "give" IDENT | "take" IDENT
             | IDENT ( "=" | "+=" | "-=" ) expr ;

expr         = orExpr ;
orExpr       = andExpr { "or" andExpr } ;
andExpr      = notExpr { "and" notExpr } ;
notExpr      = [ "not" ] cmpExpr ;
cmpExpr      = primary [ ( "==" | "!=" | "<" | "<=" | ">" | ">=" ) primary ] ;
primary      = NUMBER | "true" | "false" | IDENT | "has" IDENT | "(" expr ")" ;

(* Lexical *)
IDENT   = letter { letter | digit | "_" } ;              (* must not be a reserved word *)
NUMBER  = digit { digit } ;                              (* non-negative integers only *)
STRING  = '"' { any char except '"', '\', newline | escape } '"' ;
escape  = '\"' | '\\' | '\n' ;
COMMENT = "#" { any char except newline } ;              (* ignored *)
```

**Reserved words** (lexed as keyword tokens; never valid as names):
`story item var speaker scene ending art say if else choice give take goto has and or not true false`

**Contextual words** (lexed as ordinary `IDENT`; the parser checks the text only where the grammar requires it, so they are valid names everywhere else):
`start sprite kind death victory neutral`

For example, a scene named `start` is legal (`story "Demo" start start`), but a variable named `if` is not. Using a reserved word as a name is a parse error: `expected a name, found keyword 'if'`.

### 6.3 Semantics

**State.** The game state is the inventory (a set of item ids), the variables, the current scene, and the list of visited scene and ending ids.

**Types.** Each variable's type (int or bool) is fixed by its initializer and never changes. Ints are whole numbers. Literals are non-negative, but values can go below zero at runtime (`courage -= 5`).

**Entering a scene.** A scene is entered at the start of the game and every time a `goto` targets it, including re-entering a scene already visited. On each entry:
1. The text box is cleared. Only this scene's lines will be shown.
2. The body runs top to bottom **exactly once**:
   - `art` sets the background.
   - `say` adds a line (with its speaker, or the narrator).
   - An action (`give`, `take`, `=`, `+=`, `-=`) changes state immediately, so it affects any `if` or guard that comes after it.
   - `if` evaluates its condition against the current state and runs one branch.
   - `choice` evaluates its guard at that point; if the guard is true or absent, the choice is added to the list.
3. The result (background, lines, choices) is stored. Redrawing the screen must never run the body again.

Because the body runs on every entry, a scene-level `courage += 1` fires on every visit. Authors who want "only the first time" can guard it with a bool flag inside an `if`.

**Display.** Lines are shown one at a time, in order, each with its speaker's portrait (narrator lines have none). Choices appear as buttons after the last line, regardless of where they sit in the body.

**Choosing.** Picking a choice runs its actions in order, then enters the `goto` target (a scene **or** an ending).

**Items.** `give` on an item already held does nothing. `take` on an item not held does nothing. `has torch` is true if the inventory contains `torch`. A bare identifier in an expression refers to a variable.

**Endings.** Entering an ending clears the text box, sets its art, shows its lines one at a time, then shows the fate screen (P2-6). Endings have no choices and no actions.

**Art.** `art` may appear at most once per scene or ending, and only at the top level (never inside `if`). If a scene has no `art`, the previous background stays. If no background has been set yet, the runtime shows a plain background color.

### 6.4 Compile-time checks (what the checker must enforce)

| # | Check | Severity |
|---|---|---|
| C1 | `start` names a scene that exists (not an ending) | Error |
| C2 | Duplicate ids (scenes and endings share one namespace; items, vars, and speakers each have their own) | Error |
| C3 | `goto` target does not exist | Error |
| C4 | `give`/`take`/`has` refer to an undeclared item | Error |
| C5 | Variable used or assigned but not declared | Error |
| C6 | `say` speaker not declared | Error |
| C7 | Type error (rules below) | Error |
| C8 | Scene has no `choice` anywhere, including inside `if` branches | Error |
| C9 | Scene has no *unguarded top-level* choice (the player might get stuck) | Warning |
| C10 | Scene or ending unreachable from `start` (following every `goto`, ignoring guards) | Warning |
| C11 | Item declared but never given anywhere (choice actions or scene bodies) | Warning |
| C12 | Referenced file missing: `art` in `assets/art/`, `sprite` in `assets/sprites/` | Warning |
| C13 | `art` inside an `if`, or more than one `art` in a scene or ending | Error |
| C14 | Ending has no `kind`, or more than one | Error |

**C7 type rules:**
- `NUMBER` is int. `true`/`false` and `has X` are bool. A variable has its declared type.
- `if` conditions, choice guards, and the operands of `and`, `or`, and `not` must be bool. (So `if courage` is an error.)
- `<`, `<=`, `>`, `>=` need two ints. `==` and `!=` need two operands of the same type.
- `x = e` needs `e` to have the same type as `x`. `x += e` and `x -= e` need `x` and `e` to be int.

**Error format:**
```
story.plcyo:12:5: error: unknown scene 'vaultt' in goto (did you mean 'vault'?)
story.plcyo:3:1: warning: item 'lantern' is never given
```
Line and column are required. The "did you mean" suggestion is a nice-to-have. The parser reports the first syntax error and stops. The checker reports **all** semantic errors and warnings it finds, in source order. Pretty output (P1-7) adds the source line and a caret under the column.

### 6.5 Known limits (v1)

These are deliberate. They keep the language small and are documented so nobody builds them by accident.

| Not supported | Workaround |
|---|---|
| Arithmetic in expressions (`courage + 1 > 3`) | Compare against a different constant (`courage > 2`) |
| Negative literals (`var x = -1`) | Start at 0 and use `-=` |
| `else if` | Nest an `if` inside `else { }` |
| String variables | Use bool flags or items |
| Loops, functions, imports, randomness | Not needed for branching stories |

---

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

## 8. Functional Requirements and Work Split

### Task 0 (Week 1, P1 drives, all three approve)

| ID | Requirement | Done when |
|---|---|---|
| T0 | Create the repo skeleton from Section 5. Copy Section 6 into `docs/LANGUAGE.md` and Section 7 into `docs/SCHEMA.md` word for word. Add `pytest.ini` (`testpaths = compiler/tests`, `pythonpath = .`) and `.cursor/rules` from Section 12. | All files exist and the PR is approved by all three people. No other task starts before this. |

### Person 1: Compiler (Python) ➜ owns `compiler/`, `docs/LANGUAGE.md`, `docs/SCHEMA.md`

| ID | Requirement | Done when |
|---|---|---|
| P1-1 | **Lexer**: tokens with line/col; comments; string escapes (`\"`, `\\`, `\n`); reserved words as keyword tokens; contextual words as `IDENT` (Section 6.2) | Unit tests for every token type; unterminated string and unknown escape give clear errors with line/col |
| P1-2 | **AST nodes** as Python dataclasses, each carrying its source line/col | Every grammar construct has a node, including scene-level actions |
| P1-3 | **Parser**: recursive descent implementing the EBNF | `examples/*.plcyo` all parse; bad syntax gives `file:line:col` + message; a reserved word used as a name gives `expected a name, found keyword '...'` |
| P1-4 | **Checker**: C1-C14 and the C7 type rules (Section 6.4); collects all diagnostics | One test per check, using P3's cases in `compiler/tests/invalid/` |
| P1-5 | **Emitter**: AST ➜ JSON per Contract B | Output equals `tests/golden/tiny.json` and `features.json` when compared as parsed JSON |
| P1-6 | **CLI**: `check` and `build` (Section 4: exit codes, `--out`, `--assets`, `--runtime`, injection with `<` escaped, copying runtime files and assets) | `build` on the demo produces a `build/index.html` that plays by double-click; a test story whose text contains `</script>` builds and plays correctly |
| P1-7 | Pretty error output (source line + caret under the column) | Matches the style in Section 6.4 |
| P1-S *(stretch)* | `graph` prints a Graphviz DOT of the scene graph, with guarded edges dashed and endings colored by kind | Renders to a readable diagram for the presentation |

### Person 2: Browser Runtime (HTML/CSS/JS) ➜ owns `runtime/`

Develops against `tests/golden/*.json`, **not** against the compiler.

| ID | Requirement | Done when |
|---|---|---|
| P2-1 | **Engine** (`engine.js`, no DOM): load story, hold state, evaluate expressions, run actions, implement "enter a scene" exactly as in Section 6.3 (run once per entry, store the result). Works as a browser `<script>` (defines a global `PLCyoEngine`) and in Node (`module.exports` when `module` exists). | `node runtime/tests/engine.test.js` passes; tests cover every expression and action type, scene-level actions, re-entry, and `give`/`take` no-ops |
| P2-2 | **Scene rendering**: background art (`image-rendering: pixelated`, fixed 16:9 aspect ratio), narration box, typewriter effect. Lines advance one at a time: a click while typing finishes the line, a click after it advances to the next. Choices appear after the last line. | Looks right at desktop and laptop sizes; clicking through a scene works smoothly |
| P2-3 | **Speaker portrait**: show the speaker's sprite next to their line; narrator lines have no portrait | Portrait switches line by line |
| P2-4 | **Choices as buttons**; only choices whose guard was true at entry appear | Gated choices appear and disappear correctly |
| P2-5 | **Inventory panel** listing item names | Updates live as items are gained and lost |
| P2-6 | **Ending / fate screen**: ending art + lines, a "Your fate: DEATH / VICTORY / ..." banner from `ending_kind`, a short recap (scenes visited, items held), and a **Play again** button | Works for all 3 ending kinds |
| P2-7 | **Graceful failure**: missing image ➜ colored placeholder with the file name; scene with zero visible choices ➜ visible "dead end" message + restart; no story or unparseable story ➜ visible message. Never a blank screen. | Tested with the deliberately broken fixtures in `runtime/tests/fixtures/` |
| P2-8 | **Template and loading**: `index.template.html` with the exact injection tag from Section 4; story selection order and base-path rule from Section 4 | `build` (P1) can inject without touching runtime code; dev mode plays `features.json` with its art |
| P2-S *(stretch)* | Sound effects/music with mute button; CRT-style visual filter; text speed setting | |

**Suggested engine API** (internal to `runtime/`, so P2 may change it):
```js
const game = PLCyoEngine.create(storyJson);
game.start();        // enter story.start
game.view();         // { kind, art, lines: [{ speaker, text }], choices: [{ index, text }], endingKind }
game.choose(index);  // run the choice's actions, enter its target
game.state();        // { scene, inventory: [...], vars: {...}, visited: [...] }
game.restart();      // reset state and start again
```

### Person 3: Story, Art, Tests, Docs ➜ owns `examples/`, `assets/`, `docs/STORY_GUIDE.md`, `tests/golden/` (with P1)

| ID | Requirement | Done when |
|---|---|---|
| P3-1 | **Week 1:** write `tiny.plcyo` (2 scenes + 1 ending) and `features.plcyo`, then hand-write their golden JSON from Contract B. `features.plcyo` must use every language feature at least once, including: a scene-level action, an action inside an `if`, an `if` with and without `else`, every expression and action type, a speaker with and without a sprite, a narrator line, string escapes, a comment, and a scene named `start` (to exercise contextual words). | P1 and P2 can start immediately |
| P3-2 | **Demo game** `reapers_dungeon.plcyo`: 8-12 scenes, **3+ endings** (at least 1 death, 1 victory), at least one item-gated choice, one variable-gated choice, and one boss scene with a choice-based "fight". The Reaper's voice is snarky. | Playable start to finish with different outcomes; `check` reports zero errors |
| P3-3 | **Pixel art**: backgrounds at **480×270** (dungeon gate, corridor, boss room, plus a death scene and a victory scene); one Reaper portrait at **64×64**. All art uses one shared **16-color palette**, chosen in Week 1. Placeholders are fine early; real art by Week 6. | Files in `assets/art/` and `assets/sprites/`, referenced correctly |
| P3-4 | **Compiler test cases**: for each check C1-C14, a minimal invalid file `compiler/tests/invalid/cNN_<name>.plcyo` and a matching `.expected` file containing the expected first diagnostic as `line:col: severity: message` (no filename), agreed with P1 | P1's test suite runs them all |
| P3-5 | **Docs**: `STORY_GUIDE.md` (how to write a story, with examples, including the known limits), README with run instructions and screenshots | A classmate can write a story from the guide alone |
| P3-6 | **Final write-up + presentation** (no external requirements; this is our format): a 3-5 page write-up covering the language design, grammar, compiler pipeline diagram, checker design, and what we would add next; a ~10-minute presentation with a live demo (write a tiny story, show a compile error, build it, play it) | Ready by Week 8 |

> **Balance note:** P1 has the hardest PL work, P2 has the most UI work, and P3 has the creative work *and* is the main tester. Expect P3 to also pair with whoever is falling behind, usually P1 in Weeks 3-5.

---

## 9. Timeline (8 Weeks)

| Week | Everyone | P1 | P2 | P3 |
|---|---|---|---|---|
| **1** | **Freeze Contracts A + B** (one meeting). Task 0. Pick the art palette. Agree on golden files. | Task 0, then lexer + AST | Static HTML shell + load fixture JSON | `tiny` + `features` examples and golden JSON; first Reaper sprite sketch |
| **2** | | Parser (parse `tiny`, then `features`) | Engine: state, expressions, actions, scene entry + Node tests | Demo story outline on paper |
| **3** | | Emitter (matches goldens) | Scene rendering, line-by-line typewriter, choices | Backgrounds v1 (placeholders OK); invalid test cases |
| **4** | | Checker errors: C1-C8, C13, C14 | Inventory, portrait, ending screen | Demo story draft in `.plcyo` |
| **5** | **Integration checkpoint:** `build features.plcyo` ➜ playable game | CLI `build`, pretty errors, warnings C9-C12 | Failure handling, base paths, fate screen recap | Demo game complete in text; final art in progress |
| **6** | | Fix bugs found by P3; stretch: `graph` | Polish, UI details | Final art, playtest the demo; write STORY_GUIDE |
| **7** | **Feature freeze.** Full playthrough of every ending. Snapshot the demo's golden file. | Bug fixes only | Bug fixes only | Docs, README, screenshots, write-up draft |
| **8** | Presentation prep, dry run, buffer | | | |

**Rule:** if a week's goals slip, cut stretch items and polish first, never the contracts or the demo.

---

## 10. Acceptance Criteria (project is done when…)

1. `python -m compiler.plcyo build examples/reapers_dungeon.plcyo --out build/` succeeds, and `build/index.html` opens by double-click and is playable start to finish.
2. At least 3 distinct endings are reachable, and at least one choice is only available if the player holds an item or meets a variable threshold.
3. `check` on each invalid test case prints the expected diagnostic with the correct line and column.
4. `pytest` passes, and every golden file matches the compiler's output.
5. `node runtime/tests/engine.test.js` passes.
6. The runtime never shows a blank screen: missing art, dead ends, and missing or broken story data all produce a visible message.
7. Story text containing `</script>` builds and plays correctly.
8. `docs/LANGUAGE.md`, `docs/SCHEMA.md`, `docs/STORY_GUIDE.md`, and the README are complete and accurate.

---

## 11. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Grammar changes mid-project and breaks everyone | Freeze in Week 1; change only through a PR approved by all three |
| P2 and P3 blocked waiting on the compiler | Golden JSON files written in Week 1; the runtime never needs the compiler to develop |
| Cursor produces inconsistent code across modules | Contract-first; each person works in their own folder; one small task per prompt; tests define "done" |
| Cursor starts coding before the contract docs exist | Task 0 comes first; the Cursor rules tell it to stop if `LANGUAGE.md` or `SCHEMA.md` is missing |
| Scene-level actions behave unexpectedly on re-entry | Section 6.3 defines "once per entry"; engine tests cover re-entry; STORY_GUIDE shows the "first time only" flag pattern |
| Asset paths work in `build/` but not in dev mode (or the reverse) | Base-path rule in Section 4; P2-8 is done only when both modes show art |
| Art takes too long | Colored placeholders work from Week 1; one Reaper portrait + 4-5 backgrounds is the whole target; reuse backgrounds with different text |
| Story is too big to finish | Fixed budget: 8-12 scenes. Write the full outline before writing `.plcyo` |
| `fetch()` fails on `file://` | Build injects the JSON into the HTML (Section 4) |
| Integration surprises in Week 5 | Integration checkpoint at Week 5; `features.plcyo` exercises everything |
| One person falls behind | Weekly 15-minute sync; P3 pairs to help; cut stretch goals first |

---

## 12. Cursor Rules and Prompting Guide

### `.cursor/rules` (paste this)

```
Project: PLCyo, a DSL compiler (Python) + browser runtime (vanilla JS) that generates a pixel-art, choice-based story game.
Always read docs/BRD.md, docs/LANGUAGE.md, docs/SCHEMA.md before coding.
If docs/LANGUAGE.md or docs/SCHEMA.md does not exist, stop and tell me Task 0 must be done first.

Rules:
- Work ONLY in the folder I name. Do not edit other folders.
- NEVER change docs/LANGUAGE.md, docs/SCHEMA.md, or tests/golden/* unless I explicitly ask.
- Compiler: Python 3.11+, standard library only (pytest is allowed for tests). Hand-written lexer + recursive-descent parser, no parser generators.
- Compiler code uses relative imports inside compiler/plcyo/. Commands run from the repo root: python -m compiler.plcyo ...
- Runtime: vanilla HTML/CSS/JS. No frameworks, no npm packages, no build step.
- engine.js must not touch the DOM and must work both as a browser <script> (global PLCyoEngine) and via require() in Node.
- Runtime tests: node runtime/tests/engine.test.js, using only node:assert.
- Do one task at a time. Stop after finishing it and list what you changed. Do not start the next task.
- Every compiler feature needs a pytest test. Emitter output must equal tests/golden/*.json when both are parsed as JSON.
- Errors must include file:line:col.
- In story JSON, optional fields are omitted, never null (except say.speaker and choice.guard).
- Keep code simple and commented; this is a class project. No extra features beyond the task.
- If the spec is ambiguous, ask me instead of guessing.
```

### Prompt template (one per task)

```
Read docs/BRD.md, docs/LANGUAGE.md, docs/SCHEMA.md.
I am Person <N>. My task is <ID from Section 8>: <requirement text>.
Work only in <my folder>. Done when: <"Done when" column>.
Write the code and tests, run the tests, and summarize what you changed.
Do not start any other task.
```

### Suggested first prompts

- **P1 (first):** "Task T0: create the repo skeleton from BRD Section 5, copy Section 6 into `docs/LANGUAGE.md` and Section 7 into `docs/SCHEMA.md` word for word, and add `pytest.ini` and `.cursor/rules` from Section 12. Do not write any compiler code."
- **P1 (next):** "Task P1-1: implement the lexer in `compiler/plcyo/lexer.py` per the Lexical section and reserved/contextual word lists in `docs/LANGUAGE.md`, with pytest tests for each token type and error cases."
- **P2:** "Task P2-1: implement `runtime/engine.js` against `docs/SCHEMA.md` and the semantics in `docs/LANGUAGE.md` Section 6.3. Write `runtime/tests/engine.test.js` that loads `tests/golden/features.json` and checks expressions, actions, scene entry, and re-entry."
- **P3:** "Task P3-1: write `examples/tiny.plcyo` using only the grammar in `docs/LANGUAGE.md`, then hand-write the matching `tests/golden/tiny.json` per `docs/SCHEMA.md`."

---

## 13. Decisions Log

**Resolved in v0.1**
- Compiler in Python (team strength); runtime in vanilla JS; connected only by the JSON contract.
- Output is a single `index.html` with inlined story JSON (plus runtime files and assets).
- Narration-and-choices game: no movement, no combat controls. The Reaper is the guide/dungeon master.
- Language stays small (no loops or functions).

**Resolved in v1.0**
- Self-chosen project with no external requirements; the write-up and presentation format is ours (P3-6).
- The Reaper's tone is snarky.
- Art: 480×270 backgrounds, 64×64 portrait, one shared 16-color palette (P3 picks the palette in Week 1).
- `graph` command stays as a stretch goal for the presentation.
- Scene-level actions are added, with "run once per entry" semantics.
- Only statement and expression keywords are reserved; `start`, `sprite`, `kind`, and the ending kinds are contextual.
- An ending without `kind` is an error (no default).
- Text is shown one line at a time, with the portrait switching per line.
- The emitter writes full relative asset paths; the runtime applies a base path.
- The CLI runs as `python -m compiler.plcyo` with no install step.
- Engine tests run in Node with `node:assert`.

**Open**
- None blocking. P3 picks the exact 16-color palette in Week 1.

---

## 14. Changes from Draft v0.1

| Area | Change |
|---|---|
| Process | Added Task 0 to create `LANGUAGE.md` and `SCHEMA.md` before any coding; Cursor rules now stop if they are missing |
| Language | Added scene-level actions (`action` is now a `sceneStmt`), with "run once per entry" semantics |
| Language | Defined reserved words and contextual words; reserved words as names are a parse error |
| Language | Added the `\\` string escape (needed to write a literal backslash) |
| Language | Defined runtime semantics: text clears per scene, choices render after lines, `give`/`take` are no-ops when redundant, re-entry re-runs the body |
| Language | Added a Known Limits section (no arithmetic, negative literals, `else if`, string variables) |
| Checker | C1 now requires `start` to be a scene; C7 now spells out the type rules, including bool-only conditions; C8 counts choices inside `if` branches; C11 counts scene-level `give`; C12 names the two asset folders |
| Checker | Added C13 (`art` placement) and C14 (ending `kind`) |
| JSON | Added the `action` body node; `art` and `sprite` are omitted when absent and use full relative paths (`assets/art/...`, `assets/sprites/...`); `if.else` is `[]` when there is no `else`; added a field reference table |
| Build | Defined the exact injection tag and placeholder, `<` escaping, copied runtime files, asset folder layout, base-path rule for dev mode, exit codes, and CLI flags |
| Build | CLI runs as `python -m compiler.plcyo` from the repo root; added `pytest.ini` |
| Runtime | Line-by-line display with per-line portraits; engine works in browser and Node; Node-based engine tests; suggested engine API; visible message when no story loads |
| Tests | Golden files are compared as parsed JSON; the demo's golden file is a Week 7 snapshot instead of hand-written; invalid cases use `.expected` files without filenames |
| Scope | Project framed as self-chosen with no external requirements; defined our own write-up and presentation format; set art sizes and tone |
