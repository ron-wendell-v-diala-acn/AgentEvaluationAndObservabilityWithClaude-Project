# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Temporarily changed
  `policy_extractor/routing.py`'s `DEFAULT_CONFIDENCE_THRESHOLD` from `0.90` to `0.96`.
- **Command I ran:** From `04-hitl-routing/solution`,
  `.venv\Scripts\pytest.exe tests\test_us04_routing.py -v --tb=short`.
- **What I predicted:** Fields at 0.95 confidence that pass the default 0.90 threshold would now
  fall below 0.96 and route to `human_review`, breaking tests that expect auto-approval.
- **What actually happened (paste the key output line):** With 0.96, **4 failed, 5 passed**;
  for example, `test_ac_04_02_all_clear_routes_to_auto_approve` got `human_review` instead of
  `auto_approve`. After restoring 0.90, the same test file returned **9 passed**
  ([routing-threshold-perturbation.txt](./01-policy-pipeline/routing-threshold-perturbation.txt)).
- **How this differs from the unperturbed run:** At the original 0.90, the all-clear 0.95-confidence
  extraction auto-approves and all routing tests pass. Raising the threshold changes the routing
  boundary; it is not evidence that the baseline implementation is broken.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Temporarily changed
  `mortgage_extractor/config.py`'s `DEFAULT_TOLERANCE_USD` from **1.00** to **1300.00**, then
  replayed the bundled mismatch document.
- **Command I ran:** `.venv\Scripts\mortgage-extract.exe
  fixtures\documents\income_sum_mismatch.txt --mode replay`.
- **What I predicted:** The **1250.0** difference would fall within the raised tolerance, hiding the
  mismatch; restoring **1.00** would flag it again.
- **What actually happened (paste the key output line):** At **1300.00**, `consistent: true`,
  `discrepancies: []`, exit code **0**. At the restored **1.00**, `consistent: false` with
  calculated **9642.17**, stated **10892.17**, delta **-1250.0**, exit code **1**
  ([tolerance-perturbation.txt](./02-mortgage-extraction/tolerance-perturbation.txt)).
- **How this differs from the unperturbed run:** The original tolerance correctly rejects the
  mismatch. Increasing it masks a substantial error, showing why tolerance is a policy choice that
  needs calibration rather than an arbitrary way to make the check pass.

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
