# Research Data for UAV-Assisted IoV Trust Experiments

## Scope

This repository contains the research data supporting the experimental results in *A Context-Aware Cross-History Trust Framework for Bidirectional Zero-Trust Handover in UAV-Assisted IoV*.

The package includes processed CSV tables, curated raw JSON records, and documentation describing the experimental conditions and numerical derivations. It does not include the complete executable code or a complete reproduction environment. The data were collected from existing experimental outputs without rerunning the experiments.

## Directory structure

- `processed/` contains byte-identical copies of the accepted processed CSV files, renamed for publication-facing use.
- `raw/` contains curated JSON copies of the formal raw research records.
- `DATA_DICTIONARY.md` defines fields, units, categorical values, and missing-value semantics.

## Manuscript mapping

| Manuscript item | Experiment | Processed data | Raw data |
| --- | --- | --- | --- |
| Fig. 4(a) | E1-A deterministic context degradation | `processed/fig4a_context_robustness.csv` | `raw/e1_raw_records.json` |
| Fig. 4(b) | E1-B paired stochastic channel degradation | `processed/fig4b_snr_robustness.csv` | `raw/e1_raw_records.json` |
| Fig. 5 | E2 On-Off attack resistance | `processed/fig5_onoff_trust.csv`; `processed/fig5_paired_differences.csv` | `raw/e2_raw_records.json` |
| Fig. 6 | E3-A handover whitewashing resistance | `processed/fig6_whitewashing_trust.csv` | `raw/e3_raw_records.json` |
| Fig. 7 | E3-B dual-source history resolution | `processed/fig7_dual_source_resolution.csv` | `raw/e3_raw_records.json` |
| Fig. 8 | E4 local V2U monitoring and protection | `processed/fig8_v2u_anomaly_response.csv` | `raw/e4_raw_records.json` |
| Fig. 9 | E5 corroboration and GCS verification | `processed/fig9_corroboration_gcs.csv` | `raw/e5_raw_records.json` |
| Table 3 | E6 base-parameter validation | `processed/table3_base_parameter_validation.csv` | `raw/e6_parameter_selection_raw.json` |
| Fig. 10 | E6-V2 joint parameter selection | `processed/fig10_joint_parameter_selection.csv`; `processed/e6_joint_grid_all_candidates.csv` | `raw/e6_parameter_selection_raw.json` |
| E6 hold-out text results | Fixed post-selection hold-out validation | `processed/e6_holdout_validation.csv` | `raw/e6_holdout_raw.json` |
| Fig. 11 | E7 controlled computation overhead | `processed/fig11_computation_overhead_ms.csv` | `raw/e7_controlled_benchmark_raw.json` |
| Fig. 12 | E7 controlled communication overhead | `processed/fig12_communication_overhead_bits.csv` | `raw/e7_controlled_benchmark_raw.json` |

The source file historically named `e6_v2_fig8_origin_candidate.csv` was created before the final manuscript numbering was assigned. Its publication-facing copy is `processed/fig10_joint_parameter_selection.csv`, and it supports current Fig. 10, not Fig. 8.

## Experimental structure

### E1

E1-A is a deterministic 60-observation trajectory with a one-second interval. It compares Proposed, Wang2026, and Proposed without context correction. E1-B uses 300 paired channel realizations for Proposed and Proposed without context correction. Replicate identifiers, shared seeds, derived substream seeds, and all per-time observations are retained in `raw/e1_raw_records.json`.

### E2

E2 uses 300 paired trials over a 60-second On-Off behavior schedule. The raw file retains message outcomes, counts, seeds, and all method-specific trust states. The main CSV reports method trajectories; the paired-difference CSV reports replicate-level paired comparisons aggregated by time.

### E3

E3-A contains 300 paired handover trials and the complete pre-handover observation histories. E3-B contains seven controlled dual-source cases for three methods. The auxiliary dual-source records are retained because they verify source-state and blockchain-state resolution semantics.

### E4 and E5

E4 and E5 are deterministic mechanism experiments rather than Monte Carlo studies. Their raw files retain the complete time-ordered state trajectories, report events, counters, verification states, and method variants.

### E6

The E6 raw selection file retains the seven base-parameter validation configurations, all 243 joint candidates, all safety-gate outcomes, infeasibility reasons, response and recovery times, the 15 performance-equivalent optima, and the final tie-break information. The fixed selected base parameters are `(alpha, beta, mu_min, mu_max) = (0.7, 0.3, 0.2, 0.8)`. The selected joint configuration is `(Theta_U2V, lambda_rec, delta_th, L_abn, K_cor) = (0.4, 0.5, 0.1, 3, 3)`.

The hold-out data are kept separately because the selected parameter tuple was fixed before hold-out evaluation. No alternative candidate was tested or selected from the hold-out results.

### E7

E7 contains controlled software-representative overhead measurements. Each primitive was measured in seven batches after warm-up. Every value in `batch_means_ns` is a per-operation batch mean, not a single-call timing sample. The representative primitive time is the median of the seven batch means. Method-level computation totals are derived from symbolic operation counts multiplied by those representative primitive times.

Communication values are application-layer bit counts under the recorded encoding and field-size assumptions. They are not packet captures or measured link-layer traffic.

## Statistical summaries

For E1-B, E2, and E3-A:

- `n = 300` paired replicates.
- Sample standard deviation uses denominator `n - 1` (`ddof = 1`).
- Standard error is `sample_sd / sqrt(n)`.
- The 95% confidence interval is `mean +/- t_critical * standard_error`.
- The Student-t critical value is `1.9679296690656698` with 299 degrees of freedom.
- Statistics were calculated from unrounded raw values.

E4, E5, and the formal E6 selection traces are deterministic and do not report Monte Carlo confidence intervals.

## How to verify the reported results

The mappings below describe how the included raw records produce the included processed tables. They support numerical checking of the reported results; they do not constitute a complete executable reproduction environment.

### E1: context and channel robustness

- Fig. 4(a) is a direct time-ordered projection of `raw/e1_raw_records.json` at `e1a_records[*]`. For each record, use `time_s`, `q_env`, `proposed.historical_trust`, `wang2026.final_trust`, and `proposed_wo_context.historical_trust`. No grouping or stochastic summary is applied.
- Fig. 4(b) uses `e1b_records[*]`, grouped by `(time_s, method)`. The two method values are `proposed.historical_trust` and `proposed_wo_context.historical_trust`. Each group contains the same 300 paired replicate identifiers; the channel realization is shared within a replicate.
- For each Fig. 4(b) group, compute the arithmetic mean, sample SD with denominator `n-1`, standard error `SD/sqrt(n)`, and the 95% interval `mean +/- 1.9679296690656698 * standard_error`.

### E2: On-Off attack

- Use `raw/e2_raw_records.json` at `records[*]`, grouped by `(time_s, method)`. The method values are `proposed.historical_trust`, `fate2025.combined_trust`, and `proposed_wo_asymmetry.historical_trust`.
- Fig. 5 main trajectories use the same mean, sample-SD, standard-error, and Student-t interval formulas as E1-B.
- The single secondary paired-difference file is built within each `(replicate_id, time_s)` first: `Proposed - Proposed w/o Asymmetry` and `Proposed - FATE2025`. Only after subtraction are the 300 differences grouped by `(time_s, comparison)` and summarized. A positive value therefore means that Proposed is numerically higher than the named comparator.

### E3: handover history

- Fig. 6 uses `raw/e3_raw_records.json` at `e3a_handover_records[*]`, grouped by method across 300 replicates. The values are `proposed.inherited_trust`, `cdte2025.global_trust`, and `no_cross_history.target_initial_trust`, with the same statistics as E1-B.
- Fig. 7 is a direct projection of `e3b_main_records[*]`: select `case_id`, `method`, `inherited_trust`, `resolution_quality`, `access_state`, and `resolution_branch`. It is a controlled case table, not a Monte Carlo summary.

### E4 and E5: deterministic mechanism records

- Fig. 8 expands every `raw/e4_raw_records.json` `records[*]` item once for each member of its nested `variant_results`. Common scenario/time/trust fields are repeated, and method-specific breaker/report fields come from the selected variant. No averaging is performed.
- Fig. 9 similarly expands every `raw/e5_raw_records.json` `records[*]` item once for each nested method variant. `eligible_event_ids` and `distinct_vehicle_ids` are serialized to CSV by joining the JSON array elements with `|`; an empty list becomes an empty cell. No averaging is performed.

### E6: two-stage parameter validation and selection

Part A validates four trust-update parameters while holding the other values at the nominal tuple `(0.7, 0.3, 0.2, 0.8)`:

- `(alpha, beta)`: `(0.6, 0.4)`, `(0.7, 0.3)`, `(0.8, 0.2)`;
- `mu_min`: `0.1`, `0.2`, `0.3`;
- `mu_max`: `0.7`, `0.8`, `0.9`.

The nominal tuple occurs in all three one-factor sequences and is retained only once, giving seven unique configurations in `part_a_complete_details[*]` and `processed/table3_base_parameter_validation.csv`.

Part B uses the complete Cartesian grid in `candidate_details[*]`:

- `Theta_U2V`: `0.3`, `0.4`, `0.5`;
- `lambda_rec`: `0.25`, `0.5`, `0.75`;
- `delta_th`: `0.05`, `0.10`, `0.15`;
- `L_abn`: `2`, `3`, `4`;
- `K_cor`: `2`, `3`, `4`.

This is `3^5 = 243` candidates. The predefined security rule is `K_cor >= 3`; rows with `K_cor = 2` are rejected before dynamic evaluation. A dynamically evaluated row is safety-feasible only when all seven recorded gates are true: no transient false local action; persistent local action reached; verified network action reached; no authorization under rejected GCS verification; duplicate-report deduplication safe; malicious inherited history does not receive `NORMAL` access; and legitimate recovery is reached.

Only safety-feasible candidates are ranked. Performance rank is the ascending lexicographic rank of `(d_local_s, d_network_s, t_recovery_s)`. All 15 rows sharing the best tuple are performance-equivalent; no member is claimed superior on those three metrics. The representative is selected by: maximum safety-feasible one-step-neighbor count, then minimum discrete Manhattan distance from the five-dimensional center, then ascending parameter tuple, then `config_id`. This yields `JPS-122`, `(0.4, 0.5, 0.1, 3, 3)`. The hold-out JSON and CSV evaluate only this already selected tuple; they do not feed back into selection.

### E7: controlled overhead derivation

- The timer is `time.perf_counter_ns`. Each primitive is warmed up, then measured in seven batches. LIGHT operations use 1,000 warm-up calls and 10,000 calls per batch; MEDIUM uses 200 and 2,000; HEAVY uses 50 and 200.
- Each `batch_means_ns` element equals one batch's elapsed nanoseconds divided by its iteration count. The representative primitive time is the median of those seven per-operation batch means. `median_ms = median_ns / 1,000,000`. No outlier removal or fastest-batch selection is applied.
- A method computation total is `sum(operation_count[primitive] * primitive_timings[primitive].median_ns) / 1,000,000`. The counts are stored under `operation_counts`, and each term is retained under `method_computation_derivations`.
- The operation-count vectors are: Proposed = 12 SHA-256, 3 ECDSA signs, 4 ECDSA verifies, one IEnc encrypt/decrypt pair, and one AES-GCM encrypt/decrypt pair; SELAP2026 = 18 SHA-256, 1 PUF, 1 fuzzy-extractor reconstruction, and eight ASCON calls (four encryptions and four decryptions across the recorded payload sizes); SCZ-HA2026 = 1 Lagrange recovery and 2 ECDSA verifies; BAZAM2025 = 6 pairings, 4 SHA-256, 1 G1 scalar multiplication, and 4 GT exponentiations; BCADS2026 = 3 G1 scalar multiplications, 8 SHA-256, 1 MGSD proof generation, 1 MGSD verification, and one AES-GCM encrypt/decrypt pair for the recorded `Ed` payload.
- Communication is derived per method by summing the `bit_length` of each top-level field within a message, then summing the message totals. Nested component descriptions validate their parent field and are not added a second time. Proposed therefore uses `M1=1600`, `M2=5288`, and `M3=1752` bits, totaling `8640` bits.
- The seven reported primitive timing observations per primitive are batch means, not seven individual call samples. The five Fig. 11 values are method-level derived totals, not direct single-call protocol timings.

## Raw-data curation note

The curated JSON files retain observations, pairing identifiers, seeds, scenario definitions, statistical rules, parameter candidates, safety-gate outcomes, operation counts, encoding assumptions, benchmark-input digests, conformance records, and numerical derivations. Machine-local paths and internal process-control metadata are excluded.

The E7 fixtures contain deterministic benchmark inputs and test values. They are intended only for benchmarking and must not be used as operational credentials. The conformance records document checks performed on these representative inputs; they do not establish the security of a complete protocol implementation.

## Repository

https://github.com/maxianru98-dev/uav-iov-trust-data

## Limitations

- The package does not include complete executable code or dependency installation instructions.
- Experimental conditions and data interpretation are described in this README and DATA_DICTIONARY.md.
- The E7 results are controlled software-representative measurements, rather than measurements from a deployed UAV network.
