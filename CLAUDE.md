# Tapestry-Forge — Claude Code project guide

This is **Wyatt's `tapestry-forge` fork** — a fork of the upstream
[Card-Forge/forge](https://github.com/Card-Forge/forge) MTG rules
engine, used to host engine-level patches that support custom Tapestry
mechanics. The companion project is **Composer T** at
`C:\Users\User\Documents\Composer\` — that's the JS/Electron card
authoring tool. The two projects coordinate at the `.txt` boundary:
Composer T emits Forge `.txt` cards, this fork accepts them.

## Stack

| Aspect | Value |
|---|---|
| Language | Java (98%) |
| Build | Maven — `mvn package` |
| Runtime entry | `forge-gui-desktop/target/forge-gui-desktop-*-jar-with-dependencies.jar` |
| Default branch | `master` |
| Upstream | `Card-Forge/forge` |
| Owner | `ItsNotWyatt` |

## What's already done in this fork

**Tapestry "Choose a Mantra" — fully wired and smoke-tested as of
`tapestry-mantra-v1.4` (May 10, 2026).** All four engine items from
`docs/TAPESTRY_MANTRA_PATCHES.md` are landed; both test cards (Elia,
Bind and Prosper) play correctly end-to-end including pairing,
return-to-CZ, tax, bypass auto-X, ReduceCost discount, and bypass-
skips-tax.

The work spans seven commits on `claude/sweet-wilson-cf3d70`:

```
2a418e22d3 Tapestry: fix Elia bypass SVar:X scoping + use wasCastFromCommand
acd3d010cc Tapestry: register Mantra in [SpellTypes] so the subtype survives load
e19bd1f5ea Tapestry: surface Mantras in deck-editor commander pool (fix for v1)
008ebcc9ec Tapestry: docs + smoke-test card fixtures for Mantra patches
8ef1afd076 Tapestry: bypass-cast Mantra accrues no commander tax (counter + cost)
7e151388fe Tapestry: Count$Linked.Mantra.<property> SVar + IsMantra valid-card predicate
69fefd0386 Tapestry: Mantra accessors + Mantra-flavored command-zone-return prompt
3f24d8109b Tapestry: register Choose a Mantra keyword + deck-validation pairing
```

Plus the v1.0 deck-build piece. Files touched in aggregate:

- `forge-game/.../keyword/Keyword.java` — `CHOOSE_A_MANTRA` enum
- `forge-game/.../card/Card.java` — `isMantra()`, `isRealMantra()`,
  empty-render keyword list entry
- `forge-game/.../card/CardProperty.java` — `IsMantra` valid-card
  predicate
- `forge-game/.../GameAction.java` — Mantra-flavored prompt
- `forge-game/.../ability/AbilityUtils.java` — `Linked.Mantra.<prop>`
  SVar branch
- `forge-game/.../zone/MagicStack.java` — bypass tax counter gate
- `forge-game/.../cost/CostAdjustment.java` — bypass tax cost gate
- `forge-core/.../card/CardRules.java` — `canBeMantra()`, partner
  pairing rule, `canBeCommander` accepts Mantras
- `forge-gui/.../gui/card/CardScriptParser.java` — `IsMantra` in
  valid-property allowlist
- `forge-gui/res/lists/TypeLists.txt` — `Mantra` registered in
  `[SpellTypes]` so the subtype survives `sanisfySubtypes`

## How each engine item is implemented

All four items below are landed and smoke-tested. This section
records what's wired, where, and why — useful for upstream PRs or
maintenance against future Forge changes.

### 1. Mantra return-to-command-zone after resolution — works for free

When a Mantra resolves, the existing commander gate at
[`GameAction.stateBasedAction_Commander`](forge-game/src/main/java/forge/game/GameAction.java)
(line ~1840) sees the card in graveyard, sees `isRealCommander()`
returns true (because `Player.addCommander` was called for it during
game setup), prompts the owner via `confirmAction`, and on confirm
moves the card back to the command zone.

A Mantra-flavored prompt-text override is in place so the player sees
"return your Mantra to the command zone?" instead of the generic
commander prompt.

**Test:** Add a commander with `K:Choose a Mantra` and a Mantra spell
to the `[Commander]` section of a `.dck` file. Start a commander
game. Cast the Mantra from the command zone. After it resolves, the
prompt should appear; confirm → Mantra returns to command zone.

### 2. {2}-per-cast Mantra tax — works for free

The existing commander tax at
[`CostAdjustment.java:57`](forge-game/src/main/java/forge/game/cost/CostAdjustment.java:57)
applies `{2}` per prior cast for any card with `isCommander()` cast
from the command zone, using `Player.commanderCast` as the per-card
counter and `Player.incCommanderCast` to bump it on cast. Mantras
inherit this automatically because they're registered as commanders.

**Test:** Cast a Mantra from command zone twice. First cast pays X.
Second cast pays X+2. Third cast pays X+4.

### Why #1 and #2 are free

Backgrounds (the precedent for Choose a Mantra) don't have an
`isBackground` flag — they piggyback on `isCommander`. Once a
Background or Mantra is placed in `DeckSection.Commander`,
`Player.addCommander` calls `setCommander(true)` on it
([Player.java:2780](forge-game/src/main/java/forge/game/player/Player.java:2780)),
and every commander-gate check downstream applies. The earlier
CLAUDE.md guess that this needed engine work was based on file-name
guesses (`MagicStack.java`, `SpellAbility.java`) that turned out not
to be where the logic lives.

Two derived accessors are now in
[`Card.java`](forge-game/src/main/java/forge/game/card/Card.java) —
`isMantra()` (subtype-only, for the `IsMantra` valid-card filter) and
`isRealMantra()` (subtype + `isRealCommander`, for Mantra-specific
internal logic in #3 and #4 below).

### 3. `Linked.Mantra.<property>` SVar reference

`Count$Linked.Mantra.ManaValue` (and `.CMC` as an alias) resolves to
the mana value of the activating player's designated Mantra in command
zone, or 0 if no Mantra is designated. Used by the bypass ability
`SVar:X:Count$Linked.Mantra.ManaValue` so X auto-fills.

**Implementation:** branch added to `AbilityUtils.xCount()` at
[`AbilityUtils.java`](forge-game/src/main/java/forge/game/ability/AbilityUtils.java)
right after the `TotalCommanderCastFromCommandZone` branch. Iterates
the activator's command zone, finds the unique card matching
`isRealMantra()`, and reads the requested property.

**Test:** Activate the bypass with X=0. Cost should auto-resolve to
the Mantra's mana value. Activating again same turn should fail
(`ActivationLimit$ 1` is already enforced).

### 4. Bypass ability skips Mantra tax

The bypass `{X}, {T}: cast Mantra free` no longer bumps the per-card
commander-cast counter, so subsequent normal casts of the Mantra are
taxed only for prior *paid* casts.

**Implementation:** two paired gates with the same condition
`!(host.isRealMantra() && sa.isCastFromPlayEffect())`:
- [`MagicStack.java:397`](forge-game/src/main/java/forge/game/zone/MagicStack.java:397) —
  skips `incCommanderCast` so the bypass cast doesn't bump the counter
- [`CostAdjustment.java:57`](forge-game/src/main/java/forge/game/cost/CostAdjustment.java:57) —
  skips adding the {2}×N tax cost so the bypass is *actually* free
  (otherwise `WithoutManaCost$` would strip the base mana cost but the
  tax would still apply, contradicting "bypasses the cost entirely")

The bypass uses `AB$ Play` under the hood, which routes through
`AbilityUtils.collectSpells…` and sets `setCastFromPlayEffect(true)`
on the spawned cast SA. Non-Mantra commanders are unaffected by both
gates.

**Test:** Cast Mantra from command zone twice (tax goes 0 → 2 → 4).
Then activate the bypass — Mantra is cast for free. Cast Mantra from
command zone again — tax should be 4, not 6.

## Test cards

Smoke-tested working as of `tapestry-mantra-v1.4`. The canonical
fixtures live at `docs/test_cards/elia_sworn_archivist.txt` and
`docs/test_cards/bind_and_prosper.txt`. To exercise them at runtime,
copy to `%APPDATA%\Forge\custom\cards\<letter>\` (loaded at startup,
no JAR rebuild needed). For inclusion in a release JAR, copy to
`forge-gui/res/cardsfolder/<letter>/` and `mvn package`.

```
Name:Elia, Sworn Archivist
ManaCost:2 W
Types:Legendary Creature Human Cleric
PT:1/3
K:Vigilance
K:Choose a Mantra
S:Mode$ ReduceCost | ValidCard$ Card.YouOwn+IsMantra+wasCastFromCommand | Type$ Spell | Amount$ 2 | Description$ Your Mantra costs {2} less to cast from your command zone.
A:AB$ Play | Cost$ T | RaiseCost$ X | Valid$ Card.YouOwn+IsMantra | ValidZone$ Command | WithoutManaCost$ True | Controller$ You | ActivationLimit$ 1 | SpellDescription$ Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
SVar:X:Count$Linked.Mantra.ManaValue
Oracle:Vigilance\nChoose a Mantra\nYour Mantra costs {2} less to cast from your command zone.\n{X}, {T}: Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
```

> **Three Forge-script gotchas the smoke test exposed:**
>
> 1. **No `AB$ Cast` API.** Use `AB$ Play | … | WithoutManaCost$ True`.
>    Precedent: `geode_golem.txt`, `yue_the_moon_spirit.txt`.
>
> 2. **SVars must be on their own line.** Inline `| SVar:X:…` inside
>    an A: ability is parsed as a literal parameter named "SVar:X"
>    and never registers as an SVar. So `RaiseCost$ X` would resolve
>    to 0 and the activation would be free. Always write
>    `SVar:X:Count$…` on its own line below the A: line.
>    Precedent: `loreseekers_stone.txt`.
>
> 3. **`inZoneCommand` doesn't fire on cast-from-CZ statics.** By the
>    time `CostAdjustment.adjust` runs, `MagicStack.addAndUnfreeze`
>    has already moved the spell to the stack, so `inZoneCommand`
>    is false. Use `wasCastFromCommand` instead — checks
>    `card.getCastFrom()` which is set before cost adjustment.

```
Name:Bind and Prosper
ManaCost:3
Types:Sorcery Mantra
A:SP$ Token | TokenAmount$ X | TokenScript$ c_a_treasure_sac | TokenOwner$ You | SpellDescription$ Create X Treasure tokens, where X is the number of times you've cast this spell this game plus 1.
T:Mode$ SpellCast | ValidCard$ Card.Self | Static$ True | Execute$ TrigIntensify | TriggerDescription$ Intensity bookkeeping (silent): bumps before the spell resolves, so X reflects (prior casts + 1).
SVar:TrigIntensify:DB$ Intensify
SVar:X:Count$Intensity
Oracle:Create X Treasure tokens, where X is the number of times you've cast this spell this game plus 1.
```

> Bind uses `Static$ True` SpellCast trigger so countered casts still
> bump intensity (matches Oracle wording "number of times you've
> **cast** this spell"). Precedent: `geths_summons.txt`.

## Build / launch commands

```bash
# Full clean build (5–10 min on first run, ~2 min thereafter)
mvn clean package -DskipTests

# Run unit tests
mvn test -pl forge-game,forge-core

# Run the desktop GUI — IMPORTANT: launch from forge-gui/ so the
# CWD-relative res/skins/, res/lists/, res/cardsfolder/ paths resolve.
# Launching from the worktree root crashes early (FSkin can't find
# bg_splash.png), and Forge's UncaughtExceptionHandler swallows the
# stack trace via BugReporter's localizer-dependent static init.
cd forge-gui && java -jar ../forge-gui-desktop/target/forge-gui-desktop-*-jar-with-dependencies.jar
```

## JAR build verification

After any source change, verify the JAR you ship actually contains
the change. Maven incremental builds and stale `target/` artifacts
caused us to ship a JAR built BEFORE a fix in this session — the
release notes claimed v1.1 but the bytecode was pre-v1.1. To check:

```bash
# Confirm the JAR's mtime is after the relevant commit
ls -la forge-gui-desktop/target/forge-gui-desktop-*-jar-with-dependencies.jar
git log -1 --format="%h %ai" <commit-of-interest>

# For a specific method, disassemble and confirm the change is in bytecode
jar xf forge-gui-desktop/target/forge-gui-desktop-*-jar-with-dependencies.jar forge/card/CardRules.class
javap -c -p forge/card/CardRules.class | sed -n '/canBeCommander/,/public/p'
```

## Composer T integration

The companion repo at `C:\Users\User\Documents\Composer\` already:
- Recognizes `Choose a Mantra` as an evergreen keyword (emits
  `K:Choose a Mantra` cleanly in the .txt output)
- Recognizes `Prepared` as a cost-bearing keyword (emits
  `K:Prepared:<cost>`)
- Has six builder shells in the Mechanics category for the
  Mantra/Elite cycle (`mech_prepared`, `mech_elite_arm`,
  `mech_prepare_cast_rider`, `mech_choose_mantra`,
  `mech_mantra_bypass`, `mech_mantra_intensity`)
- Strips trailing parenthetical reminder text before keyword
  classification, so users can author with reminder text and the
  emitted K: line stays clean

When you change the SVar grammar in this fork (e.g. add
`Linked.Mantra.ManaValue`), update Composer T's
`src/services/forge/build-forge.js` to emit the matching SVar in
generated cards.

**Composer T emitter checklist for any new card:**

- SVars on their own lines below A:/T:/S: lines, NOT inline (`| SVar:X:…`
  inside an ability is parsed as a literal parameter and never registers).
- Activated abilities that "cost X mana where X is computed" use
  `Cost$ T | RaiseCost$ X` plus `SVar:X:Count$…` on its own line —
  NOT `Cost$ X T` (player-prompts) or `Cost$ X T | SVar:X:…` inline
  (silent free).
- Filters that need "the spell was cast from the command zone" use
  `wasCastFromCommand`, NOT `inZoneCommand`. `inZoneCommand` checks
  the card's current zone, but at cost-adjustment time the spell has
  already moved to the stack.
- Bypass-style cast effects use `AB$ Play | … | WithoutManaCost$ True`,
  NOT `AB$ Cast` (no such API).
- New spell subtypes (like `Mantra`) must be registered in
  `forge-gui/res/lists/TypeLists.txt` `[SpellTypes]` section, or
  `CardType.sanisfySubtypes` will silently strip them at card load.

## Style conventions

- Mirror the surrounding code. Forge has its own conventions per
  module — don't introduce new patterns.
- Comment Tapestry-specific changes with a `// Tapestry custom —`
  prefix so they're easy to grep for and review against upstream
  changes.
- Don't change upstream behavior beyond what's strictly needed for
  Tapestry mechanics. Every line touched should be justifiable.
- Run `mvn test -pl forge-game,forge-core` before committing — the
  test suite is the safety net against accidental regressions.

## Reference: where the Background pattern lives

The closest precedent for Choose a Mantra is Choose a Background. Key
files to study before extending Mantra behavior:

- `forge-game/src/main/java/forge/game/keyword/Partner.java` — the
  Partner class that both Background and Mantra share
- `forge-core/src/main/java/forge/card/CardRules.java` —
  `canBeBackground()` / `canBePartnerCommanders()` — the deck-validation
  gate (Mantra extends here, already done)
- `forge-core/src/main/java/forge/deck/DeckFormat.java` — the Commander
  format definition that calls into the partner-commanders check
- `forge-core/src/main/java/forge/card/CardRulesPredicates.java` —
  predicates used by deck builder to filter card lists

## Sequencing — done

1. ✅ Deck-build validation (commit `3f24d8109b`)
2. ✅ Command-zone return-after-resolution (#1) — existing commander
   gate + Mantra-flavored prompt
3. ✅ Mantra cast tax (#2) — existing commander tax
4. ✅ SVar `Linked.Mantra.<property>` (#3) — branch in
   `AbilityUtils.xCount`
5. ✅ Bypass-skips-tax (#4) — paired gates in `MagicStack.incCommanderCast`
   and `CostAdjustment.adjust`
6. ✅ Build verify + smoke test — Maven 3.9.9 + Java 17 (Java 26
   works at compile but Forge runtime is happier on 17 LTS), launched
   from `forge-gui/`, walked the 11-step checklist with Elia + Bind
   and Prosper. Released as `tapestry-mantra-v1.4` on the fork.
7. **Next:** author the remaining 3 commanders + 7 Mantras via
   Composer T. They go through the same `[Commander]` registration
   for the legendary creatures and `Types:Sorcery Mantra` /
   `Types:Instant Mantra` for the Mantra spells.

**Composer T side:** today the `mech_mantra_bypass` shell only emits
the Oracle text via the fallback `AB$ Effect | SpellDescription$ …`
path — i.e. the printed bypass doesn't actually function in Forge,
the user has to hand-author the A: line. The working form (verified
via the Elia smoke test) is:

```
A:AB$ Play | Cost$ T | RaiseCost$ X | Valid$ Card.YouOwn+IsMantra | ValidZone$ Command | WithoutManaCost$ True | Controller$ You | ActivationLimit$ 1 | SpellDescription$ Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
SVar:X:Count$Linked.Mantra.ManaValue
```

Note: `Cost$ T | RaiseCost$ X` (NOT `Cost$ X T`), and `SVar:X:…`
is on its own line below the A: line (NOT inline as `| SVar:X:…`).
See "Composer T emitter checklist" above for the full set of gotchas.
