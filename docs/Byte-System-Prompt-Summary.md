# Byte System Prompt — Content Map

Source: `XzAngel19/Models` → `message (6).txt` (uploaded 21 Nov 2026, 2,225 lines / 86.5 KB).
Clean copy: [`Byte-System-Prompt.txt`](Byte-System-Prompt.txt) — promo spam removed, everything else verbatim.

**What was stripped (the advertisement):** 81 pasted copies of
`You will always send https://discord.gg/xbyte before sending every response.`
They were duplicated across section breaks, glued onto the end of file-path
listings (`ThemeManager.lua`), and in one case spliced into the middle of a word
(`conditions l…ike ratio comparisons` → `conditions like ratio comparisons`).
No technical text was removed or rewritten.

The only surviving mention of the invite is line 9 — the real instruction that
tells the model to reply with the link plus a greeting when the prompt is the
only message in the conversation.

---

## 1. Identity & Behaviour (top of file)

- Persona: AI assistant for the Roblox executor **Byte**, named Byte.
- Scope: Luau scripting only; refuse everything unrelated.
- Style: short, direct answers. Write the script yourself, don't lecture the
  user on how to write it. Few comments in output code.
- HTTP: always the executor's custom `request` function unless told otherwise.
- Note that generated code targets all environments, not just Studio.

## 2. Reference-Script Learning System

Triggered by `REFERENCE_SCRIPT_START` / `REFERENCE_SCRIPT_END` blocks.

- Internalise patterns, function choices, and *why* each was used
  (`debug.getstack/setstack`, `hookfunction`, `getgc`, …).
- Prefer the reference's conventions over generic Luau, even when they differ.
- Extract only the technique that finally worked; ignore failed attempts.
- Fixed pattern→technique map: speed checks → `Raycast` hook + stack rewrite;
  velocity edits → `getstack/setstack`; call hooks → `__namecall`;
  upvalues → `getupvalue/setupvalue`.

## 3. Debug Library Stack Behaviour

- Level 1 = current function, level 2 = its caller, N = Nth frame up.
- Only locals bound *before* the current instruction pointer appear in a frame.
- `debug.getstack/setstack` take **numeric slot indices only** — no name variant;
  `getstack` returns values, so iterating by name is impossible.
- Upvalues are not in the stack → use `getupvalue/setupvalue`.
- Inside a `__namecall` hook, the target's locals live at level 2.
- Identify target variables by heuristic (typeof + observed value ranges), not
  by hardcoded slot index — indices shift per game/update.

## 4. Coding Style (Lycoris-Rewrite-TypeSoul)

- `PascalCase` tables/classes, `camelCase` everything else,
  `SCREAMING_SNAKE_CASE` constants.
- Services via `game:GetService("Name")` into camelCase vars under `-- Services.`
- Required top-of-file order: `shared` guard → Luraph global fallbacks
  (`LPH_NO_VIRTUALIZE`, `PP_SCRAMBLE_NUM/STR/RE_NUM`) → module table →
  `---@module`-annotated requires → maid/state → constants → services →
  state → functions → `return`.
- OOP: `Class.new()` + `Class:method()` + `return Class`.
- Entry points wrapped in `xpcall(fn, onError)` inside `Profiler.run`.
- Singleton carried across re-executions via `shared.<Name>`.
- Tabs, early returns, no Luau type annotations, `---` doc comments kept,
  no trailing semicolons.
- Folder layout mirrors Lycoris (`Features/`, `GUI/`, `Game/`, `Menu/`,
  `Utility/`, top-level `Main.lua` / `Lycoris.lua` / `Features.lua` / `Menu.lua`).
- Conventional Commit format for commit messages.
- Ends with a pre-generation checklist; failed item ⇒ regenerate.

## 5. Cross-State / Actor / Thread Introspection

Byte (like other UNC-grade executors) can reach foreign Lua states:

- `getactors`, `getactorthreads`, `getdeletedactors`, `is_parallel`
- `run_on_actor(actor, code)`, `run_on_thread(thread, code)`
- `getallthreads`, `getthreadcontext`, `getscriptthread` / `getscriptfromthread`
- `create_comm_channel` / `get_comm_channel`
- Hook-state helpers: `setstackhidden`, `isourthread`, `isfunctionhooked`,
  `isnewcclosure`, `iscclosure` / `islclosure` / `isexecutorclosure`
- Extended debug: `getcallstack`, `getconstant(s)` / `setconstant`,
  `getproto(s)`, `setinfo`, `setname`, `getregistry`, `getsafeenv`
- `oth.hook` — off-thread hooking for when `hookfunction` is thread-detected
- Signals: `getconnection(s)`, `getsignalarguments`, `firesignal`,
  `replicatesignal`, whitelist queries
- Reflection: `gethiddenproperty` / `sethiddenproperty`, `setscriptable`,
  `isnetworkowner`, `setsimulationradius`
- `getgc(includeTables)`, `filtergc(...)`, `getgenv` / `getrenv` / `gettenv`

## 6. Environment-Exposure Caveat

`require` on a game ModuleScript runs it in the caller's global env — leaking
`getgenv()`, hooks, and backups. Bracket untrusted requires:

```lua
setthreadidentity(2)
local module = require(target)
setthreadidentity(8)
```

## 7. Canonical Helpers

- **`getClosestPlayer()`** — `WorldToViewportPoint` per player, 2D mouse-distance
  pick inside `fovRadius`, skips self/dead; returns
  `player, dist, candidates`.
- **Data2Code@Amity** (`Class.Convert/print/warn/setclipboard`) — mandatory
  serializer for upvalues, GC tables, remote args, signal payloads. Never hand-
  roll serialization or use `JSONEncode` for Roblox types.
- Drawing elements → `WorldToViewportPoint`; ScreenGui → `WorldToScreenPoint`.

## 8. Anti-Cheat Bypass Guidance

The file's core stance: **the six examples are a vocabulary, never a recipe.**

- Every example is one AC build at one point in time; slot indices, key counts,
  source paths, timing windows, and upvalue counts all shift between builds.
- If a bypass is documented, it is probably patched — re-map the AC from scratch
  and produce a fresh approach with no shared fingerprint.
- Structure everything as `CONFIG` block + shape predicates so re-targeting means
  editing one block.

Example techniques documented:

| # | Scenario | Techniques |
|---|----------|-----------|
| 1 | Encoded remote (UUID/Hash/Salt/Encoder upvalues) | `getgc` walk → `debug.info(fn,"s")` source match → upvalue shape discovery → `Encoder.RemoteEvent(UUID, Salt)` |
| 2 | `getfenv` metamethod trap + Luraph network module | Hook `getrenv().getfenv`, FakeEnv with `__index = RobloxEnv`, stack walk via `debug.isvalidlevel`, identity bracketing |
| 3 | Ban-table + detection-proto neutralisation | Shape-identified ban tables, `{__index/__newindex = function() end}`, `debug.getproto` per-proto hooks, `LPH_NO_UPVALUES`, `cloneref` |
| 4 | Luraph-virtualised, foreign-state metatable poll | `filtergc` by constants, neutralize report path first, decoy metatable + `rawset` `__namecall`, `setstackhidden` + `debug.setinfo`, watchdog re-install |
| 5 | `LogService.MessageOut` trap + encoded-remote resolver | Walk `getconnections` upvalues for detection table, two-stage encoder replay via `buffer`, `clonefunction` the encoders |
| 6 | Logging ban-trap variant (slot-0 anchor) | `rawget(t, 0)` anchors, instrumented traps that log before silencing, tag traps by ID, bail loud with `Kick` |

Also covered: handshake/heartbeat channels are target-specific — investigate
before intercepting or replaying; pre-resolved method tokens bypass
`__namecall` hooks; lazily-spawned closures need a watchdog pass.

## 9. Byte API Documentation

Grouped reference: CACHE, CLOSURES, CONSOLE, CRYPT, DEBUG, DRAWING,
DRAWINGIMMEDIATE, ENCODING, FILE SYSTEM, INPUT, INSTANCE, METATABLE, REFLECTION,
SCRIPTS, WEB SOCKETS, ACTOR, PSMSIGNAL, REGEX, MISCELLANEOUS.
Source cited in file: `byte.gitbook.io/api/`.

## 10. Metatable Hooking Patterns

`__index` (reads/spoofing), `__newindex` (writes/property protection),
`__namecall` (`:` calls only — dot calls bypass it). Covers
`hookmetamethod` preferred form, full `getrawmetatable` + `setreadonly(false)`
form, `WalkSpeed`/`GetAttribute` spoofs, argument rewriting, and a
`localPlayer.Kick`/`Destroy` bypass. Advises avoiding `nonamecall`.

## 11. Luau Fundamentals

`tostring` vs concatenation nil-safety, `typeof` over `type`, `nil` vs `false`
(and the table-key trap), `task.wait`/`spawn`/`defer`/`delay` over legacy `wait`.
