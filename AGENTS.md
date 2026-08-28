# AGENTS.md

## Purpose

This project uses **OverPy** to author and maintain **Overwatch Workshop** game logic.

The primary goal of an agent working here is not to write code that merely looks Pythonic. The goal is to produce `.opy` source that:

- compiles successfully with the current OverPy toolchain,
- lowers to valid Overwatch Workshop rules/actions/values,
- preserves Workshop event and reevaluation semantics,
- remains safe under multiplayer concurrency and lifecycle changes,
- stays within practical Workshop server/entity/element limits,
- and is understandable enough to maintain without round-tripping through the in-game editor.

Treat OverPy as a high-level compiler for Workshop, **not as Python**. Python-like syntax does not imply Python runtime semantics, Python libraries, objects, exceptions, generators, arbitrary functions, or unrestricted data structures.

Unless the user explicitly asks to modify the OverPy compiler itself, focus changes on Workshop source such as `.opy`, `.opy.json`, included modules, and compile-time helper scripts. Do not modify OverPy compiler internals just to work around a game-mode bug.

---

## Source of truth

When deciding whether syntax or a Workshop primitive exists, use this order of authority:

1. Existing source in the current game mode, especially its main `.opy` file and included modules.
2. The current OverPy documentation and examples.
3. OverPy's generated/data definitions for actions, values, constants, heroes, maps, and custom-game settings when available.
4. Current Workshop documentation, especially Workshop.codes, when behavior depends on Workshop itself.
5. A real OverPy compilation. If an API name is uncertain, **compile instead of guessing**.
6. In-game verification for runtime semantics that compilation cannot prove.

Useful OverPy source references when working from an OverPy checkout include:

- `README.md` — language syntax, macros, compiler directives, strings, translations, optimization, and usage.
- `examples/` — idiomatic real game modes.
- `src/data/actions.ts` — known Workshop actions and OverPy mappings.
- `src/data/values.ts` — known Workshop values and OverPy mappings.
- `src/data/constants.ts` — enum-like Workshop constants.
- `src/data/heroes.ts`, `src/data/maps.ts`, `src/data/gamemodes.ts` — current identifiers.
- `src/data/customGameSettings.ts` and `customGameSettingsSchema.json` — valid custom-game settings.

Do not invent an OverPy function because a Workshop action with a similar English name exists. Most names are predictable, but several are intentionally renamed or exposed as member functions.

If web access is available and a Workshop behavior is uncertain or may have changed recently, verify it against current Workshop documentation before implementing around an assumption.

---

## Mandatory workflow before editing

Before making a gameplay change, identify the project's actual compilation root and execution model.

Find the main `.opy` file. Follow `#!mainFile` and every relevant `#!include`. Inspect variable declarations, enums, macros, subroutines, rule naming, compiler directives, settings files, and existing cleanup/reset code before adding new state.

Then answer these questions from the code before editing:

- Which Workshop event should own the behavior?
- Is the state global or per-player?
- Which contextual values are valid in that event?
- Can multiple players or multiple rule instances execute this logic concurrently?
- Does the behavior need continuous reevaluation, an edge-triggered rule, a loop, or a timed sequence?
- What must be cleaned up when the player dies, changes hero/team, leaves, the round resets, or the feature is cancelled?
- Does this feature create persistent texts/effects/entities or start persistent statuses/chases/HoT/DoT-like behavior?
- Is there already a helper macro/subroutine or variable that represents the same concept?

Prefer the smallest change that fits the existing architecture. Do not rebuild a mode around a new abstraction unless the task requires it.

---

## Workshop mental model

### Rules are event-driven

A Workshop rule consists conceptually of an event, zero or more conditions, and actions. The event determines when a rule instance can start. Conditions gate the actions. Actions execute in order and may wait, loop, call subroutines, create persistent entities, or mutate state.

In OverPy, a typical player rule looks like:

```opy
rule "example":
    @Event eachPlayer
    @Team 1
    @Condition eventPlayer.hasSpawned()
    @Condition eventPlayer.enabled

    eventPlayer.teleport(vect(0, 0, 0))
```

Do not treat an ongoing rule as a normal `while` loop. For ongoing rules with conditions, Workshop watches the condition state. A rule generally fires when its conditions become satisfied; it does not mean that the action list is automatically executed every server tick while the conditions remain true.

If repeated behavior is required, express that repetition intentionally with a documented loop/chase/reevaluation pattern and include appropriate waits when work can repeat indefinitely or many times.

### Contextual values belong to events

Values such as `eventPlayer`, `attacker`, `victim`, `healer`, `healee`, event damage/healing, and similar event values only make sense where Workshop defines them.

Never assume a contextual value is valid because its name exists in OverPy. Confirm that the selected event provides it.

`eventPlayer` refers to the player associated with the current rule instance. Multiple players may be running separate instances of the same rule at the same time.

### `localPlayer` is client-local

`localPlayer` is not ordinary server-side player state. It refers to the player on each viewing client and is intended for supported HUD/visual reevaluation contexts. It cannot be treated like a normal value that can simply be stored in Workshop variables.

Use `localPlayer` deliberately when a single reevaluated visual can replace one visual per player, but account for spectator/replay behavior and verify that the target action supports client-local evaluation.

### Global and player state are different ownership models

Declare shared game state with `globalvar` and player-owned state with `playervar`.

```opy
globalvar gameState
playervar cooldown
```

Use player variables for state that logically follows one player: cooldowns, selections, per-player timers, owned entity IDs, temporary ability state, and player-specific flags.

Use global variables for shared round state, global collections, shared objectives, indexes, global timers, and cross-player coordination.

Do not put player-specific temporary state in a shared global scratch variable unless execution is provably serialized. A wait can allow another rule instance to run before the first resumes, making shared scratch state unsafe.

---

## OverPy syntax and design rules

### Use OverPy-native expressions

Prefer OverPy syntax over literal Workshop-style syntax.

Common patterns include:

```opy
A = 2
A += 1
eventPlayer.score = 10
Hero.ANA
Color.YELLOW
Vector.UP
vect(1, 2, 3)

if eventPlayer.isAlive():
    eventPlayer.teleport(vect(0, 0, 0))

players = getAllPlayers().filter(lambda player: player.isAlive())
```

Workshop functions whose first argument is a player, array, or string are often exposed as member functions. Constants such as heroes, maps, teams, buttons, colors, and reevaluation modes are generally enum values.

Do not guess member/function names when uncertain. Search the current OverPy data/docs or compile a minimal use.

### Indentation is syntax

Use 4 spaces for new code unless the file has a deliberate established convention that must be preserved. Rule bodies, control flow, macros, and subroutines are indentation-sensitive.

Avoid mass reindentation or formatting-only changes around gameplay edits.

### Prefer structured control flow

Use `if`/`elif`/`else`, `for`, `while`/documented loop forms, and `switch` when they express the behavior clearly.

Avoid `goto` unless maintaining existing low-level control flow that truly depends on Workshop `Skip` semantics. OverPy supports labels/gotos, but they are intentionally a low-level escape hatch and are harder to reason about after edits.

### `macro` and `def` solve different problems

Use `macro` for compile-time inlining, reusable expressions, parameterized snippets, and member-like helpers.

```opy
macro Player.isReady = self.hasSpawned() and self.isAlive()
```

Macros can have parameters and defaults because they are expanded by OverPy before Workshop runtime.

Use `def` for a Workshop **subroutine** when runtime code should be shared rather than duplicated in every call site.

```opy
def resetPlayerState():
    eventPlayer.cooldown = 0
    eventPlayer.enabled = true
```

Workshop subroutines do not provide normal function parameters or return values. Do not write or design `def` as if it were a Python function. Pass data through well-owned global/player variables only when needed, and be especially careful with shared globals if calls can overlap.

Choose between them intentionally:

- prefer a macro when the helper is small, expression-like, needs arguments, or relies on call-site context;
- prefer a subroutine when duplicated runtime actions would materially increase Workshop size and the no-parameter/no-return model is acceptable;
- do not convert every repeated line into a subroutine or every helper into a macro without considering compiled size and readability.

### Compile-time preprocessing is not runtime logic

`#!include`, `macro`, `enum`, `#!define`, JavaScript preprocessing, compression helpers, and similar features run as part of source generation/compilation.

They cannot observe live match state.

Prefer AST-aware `macro` over text-based `#!define` for ordinary code reuse. Use `#!define` or JavaScript preprocessing only when the existing project genuinely needs compile-time generation that normal OverPy macros cannot express.

### Preserve compiler directives

Do not casually remove directives such as:

- `#!mainFile`
- `#!include`
- warning controls
- translation setup
- optimization directives
- `#!optimizeStrict`
- output directives
- custom post-compile hooks

`#!optimizeStrict` can exist specifically to preserve odd Workshop type-conversion behavior. Removing it can change runtime semantics even when the source still compiles.

If a mode was decompiled and already has OverPy source, **do not repeatedly decompile the compiled Workshop output back over the source**. Treat `.opy` as the source of truth once the migration to OverPy has happened.

---

## Timing, waits, loops, and concurrency

### Never create an unbounded waitless loop

Workshop can execute many actions in a server tick. A large or infinite loop without a `wait` can spike server load or crash the lobby.

Use a wait in repeating runtime loops unless the loop is intentionally small and bounded and its cost is understood.

OverPy's bare `wait()` is a real Workshop wait, not an async language yield. Its default form corresponds to a minimal Workshop wait with the rule condition ignored. If condition-sensitive abort/restart behavior matters, specify and verify the correct wait behavior explicitly.

### A wait splits execution over time

Any state read before a wait may be stale after the wait. Another player, another rule, a death, a hero swap, a team change, or a round transition may have changed the state.

After waits in long-running sequences, re-check invariants when necessary before mutating gameplay state or destroying resources.

### Avoid shared scratch-state races

Do not use a single global temporary variable as pseudo-local storage across a sequence that can wait or be invoked concurrently.

Prefer player variables for player-owned state. For genuinely global sequences, use explicit ownership/lock/token state if overlapping execution would be harmful.

### Desynchronize expensive startup work when safe

If a costly `eachPlayer` initialization runs for many players on the same tick, consider spreading work across ticks when exact simultaneity is unnecessary. Do not add delays blindly; use them only when they do not alter player-visible correctness.

---

## Conditions and server load

Workshop server load is a correctness concern, not merely a micro-optimization concern.

Put cheap, stable gating conditions before expensive or highly dynamic conditions when possible. Workshop condition evaluation can short-circuit in order, so early false conditions can prevent unnecessary evaluation below them.

Avoid putting large, frequently-mutated arrays directly into many rule conditions. A change to an array can cause conditions involving that variable to be reconsidered even when the changed index is not the one conceptually relevant to a given rule.

Avoid duplicating expensive `Player Dealt/Took Damage`-style event logic across many rules when one event handler plus well-structured branching would be clearer and cheaper.

Use the Workshop inspector while debugging when useful, but release-oriented modes should normally disable inspector recording once initialization/debugging no longer requires it. Existing OverPy modes commonly call `disableInspector()` in setup.

Do not prematurely enable aggressive size optimization to hide a structural problem. First make the mode correct and understandable, then measure/inspect element pressure and apply targeted size optimizations if necessary.

Use `#!debugElementCount` when investigating Workshop element usage or a mode that is close to compilation/size limits.

---

## Persistent resources and cleanup

Treat every persistent Workshop object or ongoing modifier as having an owner and lifecycle.

Examples include HUD text, in-world text, effects, icons, beams, Workshop-created projectiles/entities, damage/healing-over-time effects, health pools, assists, status effects, forced buttons/heroes, camera/facing controls, variable chases, and similar long-lived state.

When adding one of these, determine at creation time:

- who owns it,
- whether its ID/reference must be stored,
- whether it should reevaluate or remain static,
- what event ends it,
- and which reset/leave/death/round paths must clean it up.

Do not create a new HUD/effect every time a value changes when a single reevaluated object or an update to existing state can represent the same UI.

Do not assume the engine will automatically clean up every resource at the exact lifecycle point your feature needs. Explicitly pair create/start behavior with documented destroy/stop/reset behavior where appropriate.

Text/effect/entity slots are scarce Workshop resources. Reuse or reevaluate persistent UI/effects when it is simpler than repeatedly creating replacements.

---

## Reevaluation and UI

Workshop HUD/effect/text actions can continuously reevaluate selected inputs. This is different from repeatedly executing a rule.

Before writing a loop that updates UI every frame, check whether the relevant action's reevaluation mode can express the same behavior.

Likewise, before using reevaluation everywhere, understand which expression is reevaluated, whether it is evaluated server-side or client-side, and whether `localPlayer` is legal in that context.

Use the project's existing `HudReeval`, world-text/effect reevaluation enums, sorting conventions, and spectator visibility settings rather than inventing a parallel UI pattern.

OverPy can automatically split long strings and formatter-heavy strings to satisfy lower-level Workshop string restrictions. Write maintainable source strings first, but still consider the compiled element cost of very large formatted UI.

If the project uses OverPy translations, treat translated values specially. A translated string stored in a variable is not always interchangeable with an ordinary Workshop string; preserve the project's `_`, `__`, `___`, or `t`-string conventions and do not apply ordinary string operations to translation-array representations without verifying the documented behavior.

---

## Arrays and data structures

Arrays are powerful but can be expensive when copied, reconstructed, placed in hot conditions, or mutated deeply.

OverPy supports convenient operations such as indexing, `filter`, `map`, `all`, `any`, sorting, and even lowered 2D/3D assignment. Remember that the compiler may need to reconstruct nested arrays because Workshop's native mutation primitives are more limited.

Do not use deeply nested mutable arrays in hot paths simply because the source syntax looks cheap.

For large static numeric/vector datasets, prefer the project's existing compile-time data/compression approach. OverPy provides compression helpers specifically to reduce Workshop element cost for large constant arrays.

Use enums/macros for symbolic constants rather than unexplained numeric indexes wherever that improves correctness without bloating runtime logic.

---

## Custom game settings

When editing `.opy.json` or generated custom-game settings, do not guess setting names, hero keys, map keys, mode keys, or value shapes.

Validate against the current OverPy custom-game settings schema/data. Game patches can add, remove, or rename Workshop-facing settings.

Keep settings changes separate from rule-logic changes when practical so behavioral regressions are easier to isolate.

---

## Hero, map, and patch-sensitive behavior

Overwatch changes over time. Hero abilities, available Workshop actions/values, custom-game settings, maps, and constants can change independently of this source tree.

When a task depends on a specific hero mechanic or a recently added Workshop feature, verify that the current OverPy version knows the corresponding action/value/enum and that current Workshop documentation still describes the expected behavior.

Do not simulate a missing feature with a complicated workaround until verifying that a newer native Workshop action/value does not already exist.

Conversely, do not assume a newly documented Workshop primitive is available in the project's pinned OverPy version. Check both sides.

---

## Compilation and validation

A gameplay edit is not complete until the edited main `.opy` file compiles.

When working from the OverPy repository/toolchain itself, a reliable local build path is:

```sh
pnpm install
pnpm run compile-standalone
pnpm run compile-cli
node out/overpy_cli.js compile -i path/to/main.opy -o /tmp/main.ws.txt
```

If an installed OverPy CLI is already available, the equivalent interface is:

```sh
overpy compile -i path/to/main.opy -o /tmp/main.ws.txt
```

Use the actual project main file, not an included module in isolation, unless the module explicitly declares `#!mainFile` so OverPy can resolve the compilation root.

Treat compile warnings as actionable. Do not add `@SuppressWarnings` or `#!suppressWarnings` simply to make output quiet. Suppress a warning only when the specific warning has been investigated and the Workshop behavior is intentionally safe.

For risky changes, inspect the compiled Workshop output around the affected rules. Compilation success proves that OverPy accepted the source; it does **not** prove multiplayer runtime correctness.

When relevant, provide a short manual in-game verification matrix covering the affected lifecycle, for example:

- initial lobby/startup,
- one player and multiple players,
- join-in-progress and leave,
- death/respawn,
- hero/team changes,
- round/reset transitions,
- host vs non-host behavior,
- spectator/replay visibility for client-local UI,
- cancellation while a timed sequence is waiting,
- and repeated use to detect leaked texts/effects/entities.

Do not claim runtime behavior is fully tested if only compilation was performed.

---

## Editing policy

Preserve existing behavior outside the requested scope.

Do not rename or reorder variable declarations, enum values, subroutine declarations, or indexed state casually. Numeric identity can matter in compiled/decompiled Workshop code and in data stored by index.

Do not replace readable working OverPy with hand-written raw Workshop text.

Do not edit generated `.ws.txt` output as the primary source when `.opy` exists.

Do not remove apparently redundant cleanup, waits, reevaluation flags, or optimization directives until their Workshop purpose has been understood.

Do not introduce a generic polling loop when an event, condition transition, chase, or reevaluation primitive can model the behavior directly.

Do not use a global variable where per-player ownership is required merely to save a declaration.

Do not add persistent visual/gameplay entities without a cleanup plan.

Do not guess API names, event context, or reevaluation support. Search/compile/verify.

Prefer comments that explain Workshop-specific intent or quirks, such as why a wait exists, why an unusual reevaluation mode is required, or why state must be per-player. Avoid comments that merely restate obvious syntax.

---

## Definition of done

For an OverPy Workshop change, consider the task complete only when all applicable points are true:

- The correct main file and include graph were identified.
- The chosen event and contextual values match Workshop semantics.
- Global vs per-player state ownership is correct.
- Timed logic is safe across waits and concurrent rule instances.
- Repeating logic cannot create an accidental waitless infinite loop.
- Persistent resources have an explicit lifecycle/cleanup path.
- Hot conditions and loops were reviewed for server-load risk.
- Existing compiler/optimization/translation directives were preserved unless intentionally changed.
- The main `.opy` compilation succeeds.
- Compiler warnings introduced by the change are resolved or specifically justified.
- Patch-sensitive APIs were verified rather than guessed.
- Generated Workshop output was inspected when the lowering itself matters.
- Any behavior that requires Overwatch runtime testing is clearly identified instead of being presented as compiler-verified.

The default priority is: **correct Workshop semantics → multiplayer/lifecycle safety → compile correctness → server/resource safety → maintainability → size optimization**.