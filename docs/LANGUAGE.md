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
