# HANDOFF — quarkmind

## Orientation for Next Session

**Economy accuracy improvement phase complete.** The five queued issues (#751-#755) are all resolved. Key finding: the cadence variance assumption was inverted — EM is MORE variable than SC2 in controlled scenarios.

**Start with:** `work end` or `work` — the queue is empty. File new issues for the next improvement cycle based on the diagnostic findings.

**Key reference:** `docs/benchmarks/cycle-variance-diagnostic.md` — definitive variance data invalidating the prior hypothesis.

---

## This Session (2026-11-23)

**Completed 5 issues:**
- #751 (S/Low) — Fixed: skip spending for starting entities at game loop 0. Both workers and buildings at loop 0 no longer record false economy spending.
- #752 (M/Med) — Diagnostic: per-worker cycle variance EM σ=0.85 vs SC2 σ=0.56 (controlled). Walk distance (σ=4-6) is the dominant variance source in real games.
- #753 (M/High) — Closed not-planned: premise invalidated by #752 findings.
- #754 (M/High) — Closed not-planned: premise invalidated by #752 findings.
- #755 (XS/Low) — Regenerated standalone baseline, no regressions.

**Key finding:** The cadence variance assumption (#618→#632) was inverted. Controlled EM σ=0.85 > SC2 σ=0.56. The σ≈3+ figure came from comparing controlled EM against wild SC2 data. CLAUDE.md diagnostic findings updated.

**Blog:** "The Assumption That Wasn't" — diary entry about the inverted variance discovery.

**Updated:** CLAUDE.md diagnostic findings paragraph, `docs/benchmarks/cycle-variance-diagnostic.md`, `docs/benchmarks/standalone-emulated-game-baseline.md`

---

## Queue

Empty — all items complete.

---

## Notes

- **Pipeline bug (soredium#617):** Still present. Workaround: create branches manually, set queue state via plan_manager.
- **Accuracy state:** minerals 84.9% balance / 38.2% rate, gas 61.1% balance / 18.7% rate. Rate MAPE root cause is NOT cycle cadence (ruled out by #752). Next investigation direction: unknown — needs fresh analysis.
- **DroolsScoutingRulesTest:** Pre-existing classpath issue in full test suite — unrelated to this session's changes.
