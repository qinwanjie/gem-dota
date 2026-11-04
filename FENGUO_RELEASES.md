# Fenguo parser releases

## 0.12.0+fenguo.3 — 2026-11-04

This fork release builds on `60c937a4f4c1642d60655d1932baa6c564e16946`.
It fixes two independent causes of player/hero mapping rejection in Haohao Fenguo.

### Exact eight-bit Arcana fields

Drow Ranger Arcana's replay send table declares
`m_vecDrowRangerArcanaTargetPlayerID` as `int8[24]`, encoder `fixed8`.
The previous generic signed-varint fallback consumed following values when it
encountered the `0xff` unused-player sentinel. This could either overrun the
301-byte class baseline or silently consume subsequent entity headers. The
entity recovery layer then waited for full snapshots, losing most later hero
entity observations. Missing heroes were an effect of the decode failure, not
anonymous/private player profiles.

The decoder now consumes exactly eight bits and sign-extends signed values;
unsigned fixed8 values retain 0–255. Varint fields keep their existing decoder.
Regression tests cover boundary/sentinel values, an unaligned 24-element array,
and the following float to detect stream desynchronization. Arcana is **not**
added to the cosmetic-baseline exception list: its baselines and packet updates
must actually decode.

### Pregame hero swaps

In match `9053799343`, players 6 and 8 exchanged Lion and Earthshaker before
play. Each player's first 25 snapshots retained the previous hero, while the
next 2,639 snapshots observed the final hero. Economy and scoreboard attribution
followed the players, but assembly used the very first hero label.

Assembly now chooses the first valid gameplay identity using the pause-aware
game clock. It avoids choosing a later transformed hero, does not swap entire
player rows, and keeps the latest observed identity only for pregame-only
recordings. Without any identity observation it leaves the hero unknown.

### Verification contract

Use the ten real mapping failures as an offline integration set. Verify all ten
player identities/teams against match records, KDA independently, entity gaps,
and ability to enter the same preprocessing contract. Keep full replays and raw
profile data local/ignored; do not commit them to this repository. The consuming
app increments its extractor cache version so old broken JSON is re-parsed.

Pinned reference implementations were checked before the protocol change:
Manta `096933cf`, Clarity `7fb3f1d`, OpenDota parser `e58a668f`.
Their generic int8 fallback predates this recorded fixed8 metadata; the new
handling is tested from the failing replay's explicit send-table contract.

### Release acceptance

All ten previously rejected real replays passed the consuming app's local
material adapter and ordinary-report preparation without calling an AI service.
All 100 player hero/team mappings and all 300 KDA values matched match records.
Every replay had ten identified heroes and an entity gap ratio of zero.
Eight recordings previously had entity gaps above 92%; the other two exposed
pregame identity changes. Extractor cache version is now 15.

Targeted parser regression tests: 1,949 passed, one skipped. Full fast suite:
4,571 passed, 16 skipped, 135 deselected, four failed. Those same four failures
were reproduced unchanged at parent `60c937a4`: three concern pandas dtype/NaN
behavior and one concerns positional model construction. They are recorded as
existing baseline failures, not counted as passing. Ruff and mypy passed.
The consuming application's full suite passed all 671 tests.
