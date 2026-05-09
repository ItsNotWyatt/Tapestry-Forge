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

**Tapestry "Choose a Mantra" keyword — minimum viable patches landed by
the Composer T session on May 8, 2026.** The keyword is registered, the
deck-builder validation is wired, and the pairing rules accept a Mantra
spell as the second piece alongside a Choose-a-Mantra commander. Files
touched:

- `forge-game/src/main/java/forge/game/keyword/Keyword.java` —
  added `CHOOSE_A_MANTRA` enum entry mapping to `Partner.class`
- `forge-game/src/main/java/forge/game/card/Card.java` — added
  `Choose a Mantra` to the empty-render keyword list (line ~2594)
- `forge-core/src/main/java/forge/card/CardRules.java` —
  added `canBeMantra()` method, extended `canBePartnerCommander()` to
  treat Mantra subtype as a valid command-zone partner, extended
  `canBePartnerCommanders(b)` to allow commander+Mantra pairings

**Net effect:** a deck containing a commander with `K:Choose a Mantra`
and a single instant/sorcery with subtype `Mantra` should now pass
deck validation when the Commander format check runs. Color identity
union, singleton enforcement, and the `Either-may-be-cast-first` rule
all flow through the existing Partner machinery.

## What still needs Java work

The deck-build piece landed in commit `3f24d8109b`. **Items #1 and #2
below also already work** via Forge's existing commander gate — see
"Why #1 and #2 are free" below. Only items #3 and #4 still need
new logic. Sequencing: #3 and #4 are independent, so either can land
first.

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

Once #1–#4 above land, paste these `.txt` files into
`<forge-install>/res/cardsfolder/custom/e/` and
`<forge-install>/res/cardsfolder/custom/b/`:

```
Name:Elia, Sworn Archivist
ManaCost:2 W
Types:Legendary Creature Human Cleric
PT:1/3
K:Vigilance
K:Choose a Mantra
S:Mode$ ReduceCost | ValidCard$ Card.YouOwn+IsMantra+inZoneCommand | Type$ Spell | Amount$ 2 | Description$ Your Mantra costs {2} less to cast from your command zone.
A:AB$ Play | Cost$ X T | Valid$ Card.YouOwn+IsMantra | ValidZone$ Command | WithoutManaCost$ True | Controller$ You | ActivationLimit$ 1 | SVar:X:Count$Linked.Mantra.ManaValue | SpellDescription$ Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
Oracle:Vigilance\nChoose a Mantra\nYour Mantra costs {2} less to cast from your command zone.\n{X}, {T}: Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
```

> **Note:** the bypass uses `AB$ Play | … | WithoutManaCost$ True`
> — there's no `AB$ Cast` API in Forge. `AB$ Play` with
> `WithoutManaCost$ True` is the precedent (see `geode_golem.txt`,
> `yue_the_moon_spirit.txt`). Composer T's emitter must produce
> `AB$ Play`, not `AB$ Cast`.

```
Name:Bind and Prosper
ManaCost:3
Types:Sorcery Mantra
A:SP$ Token | TokenAmount$ X | TokenScript$ c_a_treasure_sac | TokenOwner$ You | SpellDescription$ Create X Treasure tokens, where X is the number of times you've cast this spell this game plus 1.
SVar:X:Number$1/Plus.PlayerCounters.IntensityCount
Oracle:Create X Treasure tokens, where X is the number of times you've cast this spell this game plus 1.
```

## Build / test commands

```bash
# Full clean build (5–10 min on first run, ~2 min thereafter)
mvn clean package -DskipTests

# Run unit tests
mvn test -pl forge-game,forge-core

# Run the desktop GUI for manual testing
java -jar forge-gui-desktop/target/forge-gui-desktop-*-jar-with-dependencies.jar
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

## Sequencing recommendation

1. ✅ Deck-build validation
2. ✅ Command-zone return-after-resolution (#1) — covered by existing
   commander gate; Mantra-flavored prompt added
3. ✅ Mantra cast tax (#2) — covered by existing commander tax
4. ✅ SVar `Linked.Mantra.<property>` (#3) — branch in
   `AbilityUtils.xCount`
5. ✅ Bypass-skips-tax (#4) — gated `incCommanderCast` for Mantra +
   `isCastFromPlayEffect`
6. **Next: build verify + smoke test.** Run
   `mvn -pl forge-game,forge-core -am compile -DskipTests`. If it
   compiles, drop the Elia + Bind and Prosper test cards into
   `res/cardsfolder/custom/e/` and `res/cardsfolder/custom/b/` and
   work the test checklist in `docs/TAPESTRY_MANTRA_PATCHES.md`.
7. Author the remaining 3 commanders + 7 Mantras via Composer T.
8. Tag a `tapestry-mantra-v1` build and point Composer T's
   `forgeInstallPath` at it.

**Composer T side:** today the `mech_mantra_bypass` shell only emits
the Oracle text via the fallback `AB$ Effect | SpellDescription$ …`
path — i.e. the printed bypass doesn't actually function in Forge,
the user has to hand-author the A: line. Composer T can ship a real
emitter for it later that produces:

```
A:AB$ Play | Cost$ X T | Valid$ Card.YouOwn+IsMantra | ValidZone$ Command | WithoutManaCost$ True | Controller$ You | ActivationLimit$ 1 | SVar:X:Count$Linked.Mantra.ManaValue
```

There is no `AB$ Cast` API in Forge — `AB$ Play` is the precedent
(see `geode_golem.txt`, `yue_the_moon_spirit.txt`).
