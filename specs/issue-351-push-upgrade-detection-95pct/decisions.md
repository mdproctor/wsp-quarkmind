# Decisions — Issue #351: Push upgrade detection accuracy to ≥95%

## D1: Gap identification methodology

**Choice:** Diagnostic-first per-upgrade-type accuracy report (A) combined with CmdEvent→Upgrade correlation discovery (C)
**Alternatives:**
- Iterative fix-one-rerun loop (B) — no visibility into full gap landscape; slower, risks spending time on rare upgrades while common ones cluster behind a single missing abilLink
**Rationale:** Per-type breakdown reveals clustering patterns (e.g., all upgrades from one building missing due to a single unmapped generic abilLink). Discovery test auto-finds abilLink→UpgradeType mappings from data. Together they cover both "what's missing" and "what abilLink fixes it."
**Trade-offs:** More upfront diagnostic work before any override is added — but the diagnostic itself is reusable for ongoing calibration
**Sources:** AbilityDiscoveryCalibrationTest.discoverMorphAbilLinks (pattern to replicate), StrippedReplayValidationTest (existing divergence report)
**Exploration:** quick
**Status:** captured

## D2: Accuracy target and scope

**Choice:** Aspire to 100% globally across all 179 replays; document inherent limitations (no CmdEvent emitted) as known gaps rather than accepting a lower threshold
**Alternatives:**
- Per-dataset thresholds (higher for oracle, lower for tournament) — masks real gaps behind dataset-specific excuses
**Rationale:** A global target forces us to understand every miss. Upgrades with no CmdEvent are a data limitation, not a detection failure — documenting them separately keeps the target clean.
**Trade-offs:** Some upgrades may genuinely have no signal in stripped replay data — the assertion threshold will reflect achievable accuracy, with documented reasons for each gap
**Sources:** Issue #351 acceptance criteria, StrippedReplayValidationTest upgrade divergence
**Exploration:** quick
**Status:** captured
