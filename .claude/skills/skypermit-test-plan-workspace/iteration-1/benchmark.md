# Benchmark — skypermit-test-plan, iteration 1

| Eval | with_skill pass | without_skill pass | with_skill time / tokens | without_skill time / tokens |
|---|---|---|---|---|
| eval-0 master | 8/8 (1.00) | 6/8 (0.75) | 461 s / 116k | 281 s / 88k |
| eval-1 gate-pipeline | 9/10 (0.90) | 9/10 (0.90) | 423 s / 111k | 363 s / 96k |
| eval-2 operator-access | 6/7 (0.86) | 7/7 (1.00) | 267 s / 93k | 196 s / 79k |
| **Mean** | **0.92** | **0.88** | **384 s / 107k** | **280 s / 88k** |

Delta: +0.04 pass rate, +37% time, +22% tokens.

## Analyst notes
- Assertions are weakly discriminating: most pass in both configs. The real differences are qualitative —
  per-scenario Ready/Blocked/Needs-clarification statuses and per-row sources (skill), and fewer invented
  expected results (baseline marked undefined boundaries/session behaviour as Ready in eval-0 and eval-2).
- with_skill failures: one row with source "—" (eval-2); advance-booking 60-day boundary only partially tested (eval-1).
- with_skill misses: missing COMPLETED maintenance endpoint in 03 (eval-1); Ready rows contradicting the plan's own findings (eval-0 TS-REV-05 400 vs 401; eval-2 TS-SUSP-05).
- coverage-areas.md contained an impossible scenario (window covering whole curfew with both ends outside — curfew ≥ 6 h, window ≤ 240 min).
