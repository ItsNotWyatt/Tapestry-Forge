# Tapestry Mantra smoke-test fixtures

Two `.txt` cards staged for end-to-end testing of the Choose a Mantra
engine patches (CLAUDE.md items #1–#4):

- `elia_sworn_archivist.txt` — Choose-a-Mantra commander, with a
  ReduceCost static and the {X},{T}: cast Mantra free bypass that
  exercises items #1, #2, #3, #4
- `bind_and_prosper.txt` — colorless Mantra spell that uses Forge's
  built-in Intensity tracking to produce Treasure tokens scaling
  with cast count

## How to use

After `mvn package` succeeds, copy these into the source-tree
cardsfolder so they're loaded by the desktop GUI on next start:

```
cp docs/test_cards/elia_sworn_archivist.txt forge-gui/res/cardsfolder/e/
cp docs/test_cards/bind_and_prosper.txt    forge-gui/res/cardsfolder/b/
```

Then build a Commander deck pairing them (both in the
`[Commander]` section of the `.dck` file or the deck builder UI),
plus 98 other White-or-colorless cards.

## What each step in the smoke test verifies

See `mantra_verification_log.md` in this session's memory for the
ordered checklist with expected values.

## Notes on the bypass syntax

The bypass uses `AB$ Play` (Forge's general "play another spell" API)
plus `WithoutManaCost$ True`. There is no `AB$ Cast` API. Precedent:
`forge-gui/res/cardsfolder/g/geode_golem.txt` casts the player's
commander from the command zone via the same shape.

## Notes on Bind and Prosper's intensity trigger

The intensity bookkeeping uses `T:Mode$ SpellCast | ValidCard$ Card.Self
| Static$ True`. The `Static$ True` flag makes the trigger fire silently
during the cast itself (instead of going on the stack as a triggered
ability), so:
- Intensity is bumped *before* the spell resolves, which means
  `Count$Intensity` already reflects "this cast" by the time X is
  computed in the SP$ Token effect.
- A countered cast still bumps intensity (the trigger fires regardless
  of whether the spell resolves), matching the Oracle wording "number
  of times you've **cast** this spell" — not "resolved".

Precedent: `forge-gui/res/cardsfolder/g/geths_summons.txt` uses the
same `Static$ True` SpellCast trigger to capture cast-time state.
