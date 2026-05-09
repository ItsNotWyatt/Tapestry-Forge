# Tapestry Forge — Java Engine Patch Spec

**Target fork:** `ItsNotWyatt/tapestry-forge`
**Composer T support shipped:** v0.3.80
**Status:** spec only — Java patches authored against the user's local clone of the fork

This document specifies the Java-level engine changes required to make
the new mechanics from `MANTRA_ELITE_DESIGN_MASTER.md` fully playable
in Forge. Composer T's scripting layer (v0.3.80) handles everything
that fits into existing Forge `.txt` stanzas; the items in this doc
are the residual engine-level work.

## What's NOT in this doc

These are already shipped in mainline Forge or covered by Composer T
v0.3.80 — no patches needed:

- **Prepared mechanic** — exists in mainline Forge from Strixhaven
  Mystical Archives. `K:Prepared:<cost>` is recognized.
- **Mantra subtype on instants/sorceries** — Forge accepts arbitrary
  subtype strings on the type line; no engine work needed.
- **Intensity scaling** (`number of times you've cast this spell this
  game`) — Forge Alchemy already has Intensity tracking via
  `SVar:X:Intensity` and `Count$Intensity`. Cards write
  `SVar:Intensity:Number$0` plus a per-cast intensity bump.
- **Bypass `{X},{T}: cast your Mantra without paying its cost,
  activate only once each turn`** — fully expressible:
  ```
  A:AB$ Cast | Cost$ X T | ValidCard$ Card.YouOwn+IsMantra+inZoneCommand | NoManaCost$ True | ActivationLimit$ 1 | SVar:X:Count$Linked.Mantra.ManaValue | SpellDescription$ Cast your Mantra from your command zone without paying its mana cost. X is the mana value of your Mantra. Activate only once each turn.
  ```
  The `Linked.Mantra.ManaValue` SVar reference depends on the Choose
  a Mantra implementation below — until that lands, the user fills
  X manually based on their chosen Mantra.

## Patches required

### 1. `Choose a Mantra` deckbuild keyword + command-zone designation

This is structurally identical to **Choose a Background** (the
existing partner-with-Background pattern from CLB) with two
differences:

- Background is a creature subtype; Mantra is an instant/sorcery
  subtype.
- Background lives in the command zone permanently; Mantra is cast
  from the command zone, accruing tax, and goes to graveyard/exile
  per its spell type after resolution. The command zone designation
  persists (it can be cast again from there).

The cleanest implementation path is to **fork the `ChooseABackground`
keyword class** and adapt it for Mantra subtype with persistent
re-cast semantics.

#### Files to add

```
forge-game/src/main/java/forge/game/keyword/
└── ChooseAMantra.java          # NEW — extends Keyword, mirrors ChooseABackground
```

#### Files to modify

```
forge-game/src/main/java/forge/game/keyword/Keyword.java
  + Register CHOOSE_A_MANTRA enum constant
  + Add to fromString() / parseKeyword() switch

forge-game/src/main/java/forge/deck/DeckRecognizer.java
  + Recognize Mantra subtype during deck parse
  + Validate "1 Mantra in command zone" when commander has the keyword

forge-gui/src/main/java/forge/screens/deckeditor/controllers/CCommanderDecks.java
  + UI hook: when user picks a commander with Choose a Mantra,
    show a Mantra-card picker filtered to subtype "Mantra"

forge-game/src/main/java/forge/game/zone/CommandZone.java
  + Accept noncreature, nonpermanent cards (Instant/Sorcery with
    Mantra subtype) when paired with a Choose-a-Mantra commander
  + On cast from command zone, route the spell to its post-resolve
    zone (graveyard/exile per spell type) and re-add to command zone
    after resolution finishes
  + Apply {2} tax per command-zone cast (tracked alongside existing
    commander tax counter)

forge-game/src/main/java/forge/game/spellability/SpellAbility.java
  + When source is in command zone and the source has subtype Mantra,
    add the Mantra-specific tax cost
```

#### Validation rules

Color identity rule (mirrors Choose a Background):
- Commander's color identity ∪ chosen Mantra's color identity = deck color identity
- Colorless Mantras (`Echo, Echo`, `Bind and Prosper` per design doc)
  add no color identity restriction — any commander can pair them

Singleton rule:
- Exactly one Mantra card may be designated per deck
- Designated Mantra is locked to the command zone — it cannot also
  appear in the 99

#### Tax model

- `{2}` additional per command-zone cast of the Mantra
- Tax accrues separately from the standard commander {2}-per-cast
  on the legendary itself (so a commander cast from command zone
  also still uses the existing commander tax counter)
- The Mantra commander's `{X},{T}: cast Mantra free` bypass ability
  does NOT apply Mantra tax (it bypasses the cost entirely)
- After resolution, the Mantra returns to command zone (does not go
  to graveyard/exile permanently — this is the key difference from
  a normal cast)

#### Either-cast-first rule

> Either this permanent or your Mantra may be cast first.

Mirrors Background's "either may be cast first" — no special engine
handling beyond removing the "must cast commander first" assumption
(which Forge already doesn't have in the general case).

### 2. Mantra subtype recognition in command zone

Currently Forge's command zone holds permanents. Adding instant/
sorcery support requires:

- Allow `Card.isInZone(ZoneType.Command)` to return true for cards
  with the Mantra subtype that are designated alongside a
  Choose-a-Mantra commander
- After the Mantra spell resolves, return it to command zone instead
  of the default destination (graveyard/exile)
- Treat the Mantra as "in your command zone" for triggers like
  "Whenever you cast your Mantra" — `IsCastFromCommand$ True`

### 3. Linked-Mantra SVar reference

For the bypass ability's `X = mana value of your Mantra`, the SVar
needs a way to reference the linked Mantra card on the same player's
command zone. Suggested SVar key:

```
SVar:X:Count$Linked.Mantra.ManaValue
```

Implementation: extend `CardFactoryUtil.xCount()` to recognize
`Linked.Mantra.<property>` and resolve to the activating player's
designated Mantra in command zone.

Without this, the bypass ability's `X` defaults to 0 — the user must
manually pay X each activation matching the Mantra's mana value. Not
catastrophic but defeats the convenience.

### 4. Test cards for engine validation

Once the patches are in, paste these `.txt` files into
`res/cardsfolder/custom/e/` to smoke-test:

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
A:SP$ Token | TokenAmount$ X | TokenScript$ c_a_treasure_sac | TokenOwner$ You | SubAbility$ DBNoOp | SpellDescription$ Create X Treasure tokens, where X is the number of times you've cast this spell this game plus 1.
SVar:X:Number$1/Plus.PlayerCounters.IntensityCount
SVar:DBNoOp:DB$ Pump | Defined$ Player.You
Oracle:Create X Treasure tokens, where X is the number of times you've cast this spell this game plus 1.
```

The test checklist:
1. Build a deck with Elia + Bind and Prosper. Deck builder should
   permit the pairing (W identity from Elia, colorless from Bind).
2. Both cards appear in command zone at game start.
3. Cast Bind and Prosper from command zone. Pays {3} (no tax on
   first cast). Creates 1 Treasure (`Plus.PlayerCounters.IntensityCount`
   should report 0 → +1 = 1, OR check the Forge Intensity SVar
   semantics if my count expression is wrong).
4. Bind and Prosper returns to command zone.
5. Cast Bind and Prosper again. Pays {5} (3 + 2 tax). Creates 2
   Treasures.
6. Activate Elia's `{X},{T}` with X=3 (Bind's cost). Bind casts
   without paying its mana cost. No tax. Creates 3 Treasures
   (now intensity is 2 → +1 = 3). Returns to command zone.
7. Try to activate Elia's `{X},{T}` again same turn — should fail
   (`ActivationLimit$ 1`).

### 5. Deferred / nice-to-have

- **UI affordance for Choose a Mantra in deck editor** — add a small
  badge next to commanders with the keyword indicating they need a
  Mantra picked. Mirrors how Companion is displayed.
- **Stats display** — show Mantra cast count this game in the player
  HUD (analogous to the experience counter display for Lurrus etc.)

## Estimated effort

Java work: ~2-3 days for an experienced Forge contributor. The
ChooseABackground implementation is the working precedent — the
adaptation to Mantra subtype + persistent command zone re-cast is
mostly mechanical, with the noncreature-in-command-zone change being
the riskiest part (touches game state validation in several places).

## Sequencing

1. Land patches in a `tapestry-mantra` branch on the `tapestry-forge`
   fork
2. Smoke-test the two cards above — Elia + Bind and Prosper
3. Author the remaining 3 commanders (Nemiah, Vorn, Sorvak, Selva)
   and 7 Mantras as `.txt` files via Composer T's editor (v0.3.80
   has the builder shells for the bypass + intensity)
4. Full deck-builder flow test with one of each pairing
5. Merge to fork main, tag a Tapestry-flavored Forge build
6. Rev Composer T's `forgeInstallPath` setting picker to recommend
   the Tapestry build
