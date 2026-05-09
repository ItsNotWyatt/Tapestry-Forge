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

These three pieces are NOT yet implemented in this fork. They're the
runtime command-zone behavior — the deck builds correctly today but
playing the deck would surface gaps. Sequencing matters: each item
builds on the prior.

### 1. Mantra return-to-command-zone after resolution

When a Mantra resolves, it currently goes to graveyard (default for
instants/sorceries). It needs to return to the command zone so it can
be cast again.

**Likely files:**
- `forge-game/src/main/java/forge/game/zone/MagicStack.java` —
  resolution → zone-of-rest decision
- `forge-game/src/main/java/forge/game/spellability/SpellAbility.java`
  — post-resolution destination

**Pattern to mirror:** Forge already does this for commanders (cast
from command zone, return to command zone instead of graveyard). The
`Card.isCommander()` check is the gate. We need an equivalent
`Card.isMantra()` (true when card has Mantra subtype AND is in command
zone) plumbed through the same gate.

**Test:** Cast a Mantra from command zone, confirm it returns to
command zone after resolving (or being countered).

### 2. {2}-per-cast Mantra tax

Forge tracks commander tax via `Card.commanderTax` (or similar — find
the actual field). For Mantras, we need a parallel counter that's
incremented per command-zone Mantra cast. Not a shared counter — each
Mantra in a deck gets its own tax tally.

**Likely files:**
- `forge-game/src/main/java/forge/game/card/Card.java` — add
  `mantraCastCount` field, getter, increment-on-cast hook
- `forge-game/src/main/java/forge/game/cost/CostAdjustment.java` —
  apply the +{2} cost when calculating cast cost
- `forge-game/src/main/java/forge/game/spellability/SpellAbility.java`
  — invoke the increment on cast resolution (NOT on bypass — see #3)

**Test:** Cast a Mantra from command zone twice. First cast pays X.
Second cast pays X+2.

### 3. `Linked.Mantra.<property>` SVar reference

The shared bypass ability `{X}, {T}: Cast your Mantra without paying
its mana cost` needs `X` to resolve to the linked Mantra's mana value.
Currently the ability would treat X as a user-paid value (defaulting
to 0). The fix is a new SVar count expression.

**Likely file:**
- `forge-game/src/main/java/forge/game/CardFactoryUtil.java` (or
  wherever `xCount` lives) — extend to recognize
  `Count$Linked.Mantra.ManaValue` and resolve to the activating
  player's designated Mantra in command zone

**Test:** Activate the bypass with X=0. Cost should auto-resolve to
the Mantra's mana value. Activating again same turn should fail
(`ActivationLimit$ 1` is already enforced).

### 4. Bypass ability skips Mantra tax

The bypass `{X}, {T}: cast Mantra free` should NOT increment the
Mantra cast counter (the design doc explicitly says no tax accrues
from the bypass). Whatever increments the counter in #2 needs to be
gated on "is this a command-zone cast paying its actual cost" not
"any cast of the Mantra."

**Test:** Cast Mantra from command zone twice (tax goes 0 → 2 → 4).
Then activate bypass — Mantra is cast (free). Cast Mantra from
command zone again — tax should be 4, not 6. (Bypass didn't
increment.)

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
A:AB$ Cast | Cost$ X T | ValidCard$ Card.YouOwn+IsMantra+inZoneCommand | NoManaCost$ True | ActivationLimit$ 1 | SVar:X:Count$Linked.Mantra.ManaValue | SpellDescription$ Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
Oracle:Vigilance\nChoose a Mantra\nYour Mantra costs {2} less to cast from your command zone.\n{X}, {T}: Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
```

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

1. ✅ Deck-build validation (done in this session)
2. **Next: command-zone return-after-resolution (#1 above)** — smallest
   discrete piece, biggest user-facing payoff (lets you actually keep
   playing your Mantra deck)
3. Mantra cast tax (#2) — blocked on #1
4. SVar Linked.Mantra (#3) — independent of #1/#2, can land in parallel
5. Bypass-skips-tax (#4) — depends on #2
6. Run all 5 commanders + 8 Mantras as a smoke test
7. Tag a `tapestry-mantra-v1` build and point Composer T's
   forgeInstallPath at it

Estimated effort for #1–#4: 1–2 days for someone familiar with Forge's
internals; 2–3 days starting cold.
