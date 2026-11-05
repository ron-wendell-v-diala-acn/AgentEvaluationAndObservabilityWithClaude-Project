# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Used the offline test's recorded response for
  `POL-2025-009` with the required `endorsements` field set to `null`; no source fixture was edited.
- **Command I ran:** From `04-hitl-routing/solution`,
  `.venv\Scripts\pytest.exe tests\test_us01_retry.py -v`.
- **What I predicted:** The missing-source case should escalate immediately because another model
  call cannot add information absent from the source.
- **What actually happened (paste the key output line):**
  `test_ac_01_04_missing_source_halts_immediately PASSED`. The test asserts a
  `RetryFutileEscalation` for `endorsements`, pattern `endorsements_absent`, with exactly one API
  call ([retry-perturbation.txt](./01-policy-pipeline/retry-perturbation.txt)).
- **How this differs from the unperturbed run:** The format/consistency failure tests exercise
  retry; the missing-source test stops after the first call and escalates.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Selected the prebuilt perturbation input
  `income_sum_mismatch.txt` (not modified in place), where the stated total does not match the
  monthly components; compared it with the clean appraisal fixture.
- **Command I ran:** `.venv\Scripts\mortgage-extract.exe fixtures\documents\income_sum_mismatch.txt
  --mode replay`, alongside the clean appraisal replay.
- **What I predicted:** The schema-valid values would still fail the deterministic sum check.
- **What actually happened (paste the key output line):** `consistent: false`; calculated
  **9642.17**, stated **10892.17**, delta **-1250.0**, exit code **1**
  ([discrepancy-run.txt](./02-mortgage-extraction/discrepancy-run.txt)).
- **How this differs from the unperturbed run:** The appraisal replay is `consistent: true` and
  normalizes `gross_living_area_sqft` to `2400`
  ([extract-run.txt](./02-mortgage-extraction/extract-run.txt)).

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Enabled the `--simulate-timeout` option so the
  logistics source fails.
- **Command I ran:** `.venv\Scripts\supply-chain-investigate.exe meridian --offline
  --simulate-timeout`.
- **What I predicted:** The coordinator would finish and mark logistics-only evidence Incomplete,
  rather than treating it as an empty successful source.
- **What actually happened (paste the key output line):** `Sources unavailable: logistics
  unavailable (timeout)` and `late_shipment_count [missing source: timeout reading logistics]`;
  exit code **0** ([timeout-run.txt](./03-supply-chain/timeout-run.txt)).
- **How this differs from the unperturbed run:** The healthy run retains the **95.0%** audit and
  **78.0%** logistics claims in **Contested**; on timeout, the logistics claim moves to **Incomplete**
  and only the audit's **95.0%** remains ([briefing.md](./03-supply-chain/briefing.md),
  [timeout-run.txt](./03-supply-chain/timeout-run.txt)).
