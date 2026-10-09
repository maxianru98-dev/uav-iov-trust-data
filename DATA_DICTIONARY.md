# Data Dictionary

## General conventions

- Time is measured in seconds unless a field name explicitly uses `_ms` or `_ns`.
- SNR is measured in decibels (`dB`). Delay fields ending in `_ms` are milliseconds.
- Trust, context quality, ratios, probabilities, and confidence limits are dimensionless.
- Communication overhead is measured in bits.
- Computation overhead is measured in milliseconds; primitive batch means are also available in nanoseconds.
- Boolean values in JSON are `true` or `false`. CSV Boolean values use `True`/`False` or uppercase `TRUE`/`FALSE` as preserved from the formal outputs.
- An empty CSV field represents a value that was not reached or is not applicable under the stated branch. It does not mean zero.
- JSON `null` means unavailable or not applicable. It does not mean zero, false, or an imputed value.
- Record order is significant and follows the formal execution output order.

## Core trust and state definitions

These definitions follow the implemented Section IV formulas and branch conditions used to produce the records.

### `eta` and trust-update quantities

`eta` is the context/environment factor computed from caller-supplied SNR:

`eta = 1 / (1 + exp(-kappa * (SNR - SNR_th)))`.

The implementation uses an algebraically equivalent stable branch for negative logistic arguments; if `kappa = 0`, `eta = 0.5`. It does not derive SNR from another field. In E1-B, `eta_realized` is this formula applied to the stored realized SNR. In E4, CSV `eta` is the `environment_factor` returned by the same trust update.

The related fields are:

- `delay_quality = max(0, 1 - delay/tau_max)`;
- `direct_trust = alpha * primary_metric + beta * delay_quality`;
- `contextual_trust = eta * direct_trust + (1-eta) * previous_historical_trust`;
- `selected_mu = mu_min` when `contextual_trust < previous_historical_trust`, otherwise `mu_max`;
- `historical_trust = selected_mu * previous_historical_trust + (1-selected_mu) * contextual_trust`.

### `abnormality`, persistence, and breaker state

`abnormality` is a Boolean indicator, not a magnitude:

`abnormality = (contextual_trust < Theta_V2U)`.

The comparison is strict; equality is not abnormal. If abnormal, the persistence counter becomes `min(previous_counter + 1, L_abn)`; otherwise it becomes `max(previous_counter - 1, 0)`. The full dual-condition breaker is true only when both `historical_trust < Theta_V2U` and `counter >= L_abn`.

### Dual-source resolution fields

For two valid inputs, the signed difference is `D = T_src - T_bc`, not an absolute difference. The fields mean:

- `resolution_quality`: `NORMAL` when at least one valid source supplies the formal result; `PROVISIONAL` when both sources are unavailable; `REVIEW` for the remaining no-valid-source combinations containing invalid evidence.
- `resolution_branch`: `BOTH_VALID_SOURCE` when both are valid and `D <= epsilon`, returning `T_src`; `BOTH_VALID_RECOVERY` when both are valid and `D > epsilon`, returning `T_bc + lambda_rec * D`; `SOURCE_ONLY` when only source history is valid; `BLOCKCHAIN_ONLY` when only blockchain history is valid; `BOTH_UNAVAILABLE` when both are unavailable; `REVIEW_FALLBACK` for the other no-valid-source combinations.
- For `BOTH_UNAVAILABLE` and `REVIEW_FALLBACK`, inherited trust is `min(T_init, Theta_U2V)`.
- `access_state`: `RESTRICTED` whenever quality is not `NORMAL`; with `NORMAL` quality, `NORMAL` when `inherited_trust >= Theta_U2V`, otherwise `DENIED`.

All three access-state values are formally defined. The current Fig. 7 records realize only `NORMAL` and `RESTRICTED`; no `DENIED` row occurs in this controlled case set.

## Categorical identifiers and encoded lists

- E2 `method`: `Proposed`, `FATE2025`, or `Proposed w/o Asymmetry`.
- E2 `comparison`: exactly `Proposed - Proposed w/o Asymmetry` or `Proposed - FATE2025`. Subtraction is A minus B within the same replicate and time before aggregation; positive means Proposed has the higher numerical trust.
- E3-A `method`: `Proposed`, `CDTE2025`, or `No Cross-History`.
- E3-B `method`: `Proposed Dual-Source`, `Source-only`, or `Blockchain-only`.
- E3-B `case_id`: `C1` both valid and consistent; `C2` both valid with source significantly higher; `C3` source valid/blockchain unavailable; `C4` source unavailable/blockchain valid; `C5` source invalid/blockchain valid; `C6` both unavailable; `C7` source invalid/blockchain unavailable.
- E4 `scenario_id`: `E4_TRANSIENT` has normal `[0,20)`, anomalous `[20,22)`, and normal `[22,60)` intervals; `E4_PERSISTENT` has normal `[0,20)`, anomalous `[20,40)`, and normal `[40,60)` intervals.
- E4 `method`: `Proposed Dual-Condition Breaker`, `History-only Breaker`, or `No Local Breaker`.
- E5 `case_id`: `E5_VERIFIED` uses a verified GCS result after the corroboration threshold; `E5_REJECTED` uses a rejected GCS result after the threshold. Before the threshold, verification is `NOT_EVALUATED`.
- E5 `method`: `Proposed Two-Gate Authorization` or `Corroboration-Only Authorization (No GCS Gate)`.
- E6 Part-A `validation_group`: `ALPHA_BETA`, `MU_MIN`, or `MU_MAX`. Actual `config_id` values are `AB-1`, `AB-2`, `AB-3`, `MIN-1`, `MIN-3`, `MAX-1`, and `MAX-3`; nominal duplicate rows `MIN-2` and `MAX-2` are omitted.
- E6 joint `config_id`: `JPS-001` through `JPS-243` in Cartesian-product order `(Theta_U2V, lambda_rec, delta_th, L_abn, K_cor)`. `JPS-122` is the selected representative.

In `fig9_corroboration_gcs.csv`, `eligible_event_ids` and `distinct_vehicle_ids` use a literal vertical bar (`|`) between elements, without JSON brackets or quoting; an empty list is an empty cell. Read a nonempty value with `cell.split("|")`. Event IDs remain in eligible-event time order. Vehicle IDs remain in deterministic first-seen order after target/admissibility/window filtering and exact-identity deduplication. In raw JSON, the same values are native JSON arrays.

`not_reached_flags` in `table3_base_parameter_validation.csv` is different: it is a JSON-array string such as `[]` or `["DETECTION_NOT_REACHED","HISTORICAL_RESPONSE_NOT_REACHED"]` and must be parsed with a JSON parser, not split on `|`.

## Processed CSV files

### `processed/fig4a_context_robustness.csv`

One deterministic row per second, 60 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `time_s` | s, integer | Observation time, 0 through 59. |
| `q_env` | dimensionless | Caller-provided environmental quality. |
| `proposed_trust` | dimensionless | Proposed historical trust. |
| `wang2026_trust` | dimensionless | Wang2026 final trust. |
| `proposed_wo_context_trust` | dimensionless | Proposed ablation without context correction. |

### `processed/fig4b_snr_robustness.csv`

Two method rows per second, 120 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `time_s` | s | Observation time. |
| `mean_snr_db` | dB | Prespecified mean SNR before Gaussian shadowing. |
| `method` | category | `Proposed` or `Proposed w/o Context`. |
| `n` | count | Paired replicate count, 300. |
| `trust_mean` | dimensionless | Mean trust across replicates. |
| `trust_sample_sd` | dimensionless | Sample SD with `ddof=1`. |
| `trust_standard_error` | dimensionless | Sample SD divided by `sqrt(n)`. |
| `trust_ci95_low`, `trust_ci95_high` | dimensionless | Student-t 95% confidence limits. |

### `processed/fig5_onoff_trust.csv`

Three method rows per second, 180 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `time_s` | s | Observation time. |
| `behavior_state` | category | Prespecified normal or malicious interval state. |
| `method` | category | `Proposed`, `FATE2025`, or `Proposed w/o Asymmetry`. |
| `n` | count | Paired replicate count, 300. |
| `trust_mean` | dimensionless | Mean trust. |
| `trust_sample_sd` | dimensionless | Sample SD. |
| `trust_standard_error` | dimensionless | Standard error. |
| `trust_ci95_low`, `trust_ci95_high` | dimensionless | Student-t 95% confidence limits. |

### `processed/fig5_paired_differences.csv`

Two comparison rows per second, 120 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `time_s` | s | Observation time. |
| `behavior_state` | category | Normal or malicious interval state. |
| `comparison` | category | Ordered replicate-level method difference. |
| `n` | count | Number of paired differences, 300. |
| `difference_mean` | dimensionless | Mean paired trust difference. |
| `difference_sample_sd` | dimensionless | Sample SD of paired differences. |
| `difference_standard_error` | dimensionless | Standard error of paired differences. |
| `difference_ci95_low`, `difference_ci95_high` | dimensionless | Student-t 95% confidence limits. |

The actual comparison labels encode the arithmetic direction: `Proposed - Proposed w/o Asymmetry` and `Proposed - FATE2025`.

### `processed/fig6_whitewashing_trust.csv`

One row per method, 3 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `method` | category | `Proposed`, `CDTE2025`, or `No Cross-History`. |
| `n` | count | Paired replicate count, 300. |
| `trust_mean` | dimensionless | Mean target-domain trust at handover. |
| `trust_sample_sd` | dimensionless | Sample SD. |
| `trust_standard_error` | dimensionless | Standard error. |
| `trust_ci95_low`, `trust_ci95_high` | dimensionless | Student-t 95% confidence limits. |

### `processed/fig7_dual_source_resolution.csv`

Seven cases for three methods, 21 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `case_id` | identifier | `C1` through `C7`, defined in “Categorical identifiers and encoded lists.” |
| `method` | category | `Proposed Dual-Source`, `Source-only`, or `Blockchain-only`. |
| `inherited_trust` | dimensionless or empty | Resolved target-domain trust; empty when no numerical trust is produced. |
| `resolution_quality` | category | `NORMAL`, `PROVISIONAL`, or `REVIEW`; formal meanings are defined above. |
| `access_state` | category | Resulting access state. Current rows contain `NORMAL` or `RESTRICTED`; `DENIED` is formally defined but not realized here. |
| `resolution_branch` | category | `BOTH_VALID_SOURCE`, `BOTH_VALID_RECOVERY`, `SOURCE_ONLY`, `BLOCKCHAIN_ONLY`, `BOTH_UNAVAILABLE`, or `REVIEW_FALLBACK`. |

### `processed/fig8_v2u_anomaly_response.csv`

Two scenarios, 60 times, and three methods, 360 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `scenario_id` | identifier | `E4_TRANSIENT` or `E4_PERSISTENT`, with intervals defined above. |
| `time_s` | s | Observation time. |
| `service_state` | category | Prespecified service state. |
| `method` | category | `Proposed Dual-Condition Breaker`, `History-only Breaker`, or `No Local Breaker`. |
| `ack_ratio` | ratio | Observed acknowledgement ratio. |
| `eta`, `direct_trust`, `contextual_trust`, `historical_trust` | dimensionless | Trust-update quantities defined under “Core trust and state definitions.” |
| `abnormality` | Boolean | Strict predicate `contextual_trust < Theta_V2U`; not an anomaly magnitude. |
| `counter` | count | Consecutive-abnormality counter. |
| `full_breaker`, `effective_breaker` | Boolean | Full-formula and method-effective breaker decisions. |
| `T_rep_pre`, `T_rep_post` | dimensionless | Report-reference trust before and after the reporting policy. |
| `trust_drop` | dimensionless | Trust decrease used by the report trigger. |
| `abnormal_drop_trigger`, `report_trigger` | Boolean | Trigger components and final report decision. |
| `admissible` | Boolean or empty | GCS admissibility when a report exists; empty when no report is evaluated. |

### `processed/fig9_corroboration_gcs.csv`

Two cases, 60 times, and two methods, 240 rows.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `case_id` | identifier | `E5_VERIFIED` or `E5_REJECTED`. |
| `time_s` | s | Evaluation time. |
| `method` | category | `Proposed Two-Gate Authorization` or `Corroboration-Only Authorization (No GCS Gate)`. |
| `target_uav_id` | identifier | Report target. |
| `window_start_exclusive`, `window_end_inclusive` | s | Trailing corroboration-window boundaries. |
| `eligible_event_ids` | pipe-encoded list | Events in eligible time order; empty cell means no eligible events. |
| `window_report_event_count` | count | Eligible report-event count. |
| `distinct_vehicle_ids` | pipe-encoded list | Exact-identity-deduplicated vehicles in first-seen order; empty cell means none. |
| `distinct_vehicle_count` | count | Number of distinct reporting vehicles. |
| `k_cor` | count | Corroboration threshold. |
| `threshold_met` | Boolean | Whether distinct-vehicle count reaches `k_cor`. |
| `verification_status` | category | `NOT_EVALUATED`, `VERIFIED`, or `REJECTED`. |
| `core_gcs_verified` | Boolean or empty | Core GCS result when evaluated. |
| `frozen_escalation_enabled` | Boolean | Whether the tested method requires the GCS gate. |
| `effective_authorization` | Boolean | Final authorization decision. |

### `processed/table3_base_parameter_validation.csv`

Seven base-parameter configurations.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `validation_group`, `config_id` | category/identifier | Parameter family and configuration. |
| `alpha`, `beta`, `mu_min`, `mu_max` | dimensionless | Tested trust parameters. |
| `detection_delay_s` | s or empty | Detection delay; empty if not reached. |
| `false_positive_count` | count | False positive observations. |
| `benign_observation_count` | count | Number of benign observations, 16. |
| `false_positive_rate` | ratio | False positives divided by benign observations. |
| `historical_response_delay_s` | s or empty | Historical-trust response delay; empty if not reached. |
| `historical_recovery_time_s` | s or empty | Historical-trust recovery time. |
| `not_reached_flags` | JSON-encoded list | Metrics not reached under the trace. |

### `processed/e6_joint_grid_all_candidates.csv`

All 243 joint candidates.

| Field group | Meaning |
| --- | --- |
| `config_id` | Candidate identifier. |
| `theta_u2v`, `lambda_rec`, `delta_th`, `l_abn`, `k_cor` | Candidate parameter tuple. |
| `predefined_security_feasible` | Whether the predefined `K_cor` security constraint is met. |
| `gate_transient_local_safety`, `gate_persistent_detectability`, `gate_verified_network`, `gate_rejected_gcs`, `gate_duplicate_report`, `gate_malicious_history_access`, `gate_legitimate_recovery` | Individual Boolean safety gates. |
| `safety_feasible` | Conjunction of required feasibility conditions. |
| `t_local_action`, `t_network_action`, `t_normal_recovery` | Event times in seconds or empty if not reached. |
| `d_local_s`, `d_network_s`, `t_recovery_s` | Local delay, network delay, and recovery time in seconds. |
| `robust_neighbor_count` | Number of feasible one-step neighbors. |
| `performance_rank` | Lexicographic performance rank among feasible candidates. |
| `selected` | Final selected-candidate flag. |
| `infeasible_reason` | Empty for feasible candidates; otherwise the recorded exclusion reason. |

### `processed/fig10_joint_parameter_selection.csv`

The 108 safety-feasible candidates used for current Fig. 10.

| Field | Meaning |
| --- | --- |
| `config_id` | Candidate identifier. |
| `theta_u2v`, `lambda_rec`, `delta_th`, `l_abn`, `k_cor` | Candidate parameter tuple. |
| `d_local_s`, `d_network_s`, `t_recovery_s` | Response and recovery times in seconds. |
| `robust_neighbor_count` | Feasible one-step-neighbor count. |
| `performance_rank` | Performance rank. |
| `performance_equivalent_optimum` | Membership in the 15-candidate optimal set. |
| `selected` | Selected representative, `JPS-122`. |

### `processed/e6_holdout_validation.csv`

Thirteen fixed-candidate hold-out checks.

| Field | Meaning |
| --- | --- |
| `scenario` | `HV-A`, `HV-R`, or overall result. |
| `config_id` and five parameter fields | Fixed selected candidate and tuple. |
| `metric_or_gate` | Hold-out metric or gate name. |
| `value` | Recorded numerical, Boolean, or categorical value. |
| `status` | `PASS`, `RECORDED`, or `HOLDOUT PASS`. |
| `event_time` | Event time in seconds; empty when not applicable. |
| `notes` | Optional explanatory text; empty when none is required. |

### `processed/fig11_computation_overhead_ms.csv`

Five method-level values.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `method` | category | Compared method. |
| `controlled_computation_overhead_ms` | ms | Symbolic operation counts multiplied by representative primitive times. |

### `processed/fig12_communication_overhead_bits.csv`

Five method-level values.

| Field | Unit/type | Meaning |
| --- | --- | --- |
| `method` | category | Compared method. |
| `controlled_communication_overhead_bits` | bit | Application-layer communication total under the recorded encoding assumptions. |

## Raw JSON files

### `raw/e1_raw_records.json`

- Top-level `method_orders` records the output ordering separately for E1-A and E1-B; `environment` records the software platform used for the accepted run.
- `e1a_records[*]`: 60 deterministic time-ordered records. Common keys include `time_s` and `q_env`; nested `proposed`, `wang2026`, and `proposed_wo_context` objects contain the method states. The plotting fields are `proposed.historical_trust`, `wang2026.final_trust`, and `proposed_wo_context.historical_trust`.
- `e1b_records[*]`: 18,000 records ordered by replicate and time. Pairing keys are `replicate_id`, `shared_replicate_seed`, and `substream_seed`; channel keys include mean/realized SNR and `eta_realized`; nested `proposed` and `proposed_wo_context` objects contain the two historical-trust outputs.
- `rng_fixture` defines the Gaussian shadowing generator, seed derivation, draw order, shared-channel rule, and no-clipping/no-resampling rules.
- `statistics_contract` gives `n=300`, sample SD with `ddof=1`, `SE=SD/sqrt(n)`, the Student-t critical value, and the confidence-limit formulas.

### `raw/e2_raw_records.json`

- `records[*]`: 18,000 replicate-time records ordered by replicate and time. Each record contains `replicate_id`, time/behavior state, seed fields, `message_outcomes`, `true_count`, and `false_count` as unaggregated evidence.
- `records[*].proposed`, `.fate2025`, and `.proposed_wo_asymmetry` contain the method-specific update states. The processed values are respectively `historical_trust`, `combined_trust`, and `historical_trust`.
- `rng_fixture` records the paired draw construction; `statistics_contract` records the 300-replicate summary formulas.
- `paired_difference_contract.comparison_order` gives the two exact A-minus-B labels; `replicate_level_difference_first=true` requires subtraction before group statistics. This secondary file does not replace the primary trajectories.

### `raw/e3_raw_records.json`

- `e3a_scenario`, `e3a_context`, `e3a_rng_fixture`, and `e3a_statistics_contract` define the fixed handover scenario, context, paired generator, and 300-replicate statistics.
- `e3a_observation_records[*]`: 12,000 pre-handover observation records containing the replicate/time evidence and Proposed state trajectory.
- `e3a_handover_records[*]`: 300 replicate-level handover results. The Fig. 6 inputs are `proposed.inherited_trust`, `cdte2025.global_trust`, and `no_cross_history.target_initial_trust`.
- `e3b_main_records[*]`: 21 case/method results. Key fields are `case_id`, `method`, source/blockchain states and trusts, `inherited_trust`, `resolution_quality`, `access_state`, `resolution_branch`, and `signed_difference` where both sources are valid.
- `e3b_auxiliary_audit_records[*]`: 3 auxiliary source-state checks retained because they verify Algorithm-2 branch and signed-difference semantics.
- Numerical trust fields may be `null` when the formal branch yields no numerical trust.

### `raw/e4_raw_records.json`

- `scenario_order` and `method_order` give deterministic traversal order. `execution_contract` states that the experiment has no seed, Monte Carlo mean, sample SD, or confidence interval.
- `records[*]`: 120 common scenario-time records (2 scenarios x 60 times). Each contains `scenario_id`, `time_s`, `service_state`, `ack_ratio`, the trust-update fields (`eta`, `delay_quality`, `direct_trust`, `contextual_trust`, `historical_trust`), Boolean `abnormality`, and persistence `counter`.
- `records[*].variant_results[*]`: one nested result for each of the three methods, including full/effective breaker decisions, `T_rep_pre`, `T_rep_post`, trust drop, trigger fields, report metadata, and `admissible` when GCS admissibility was actually evaluated.
- In processed CSV, empty `admissible` means no triggered report reached admissibility evaluation; it is not equivalent to `False`.

### `raw/e5_raw_records.json`

- `case_order`, `method_order`, and `experiment_scope` define deterministic traversal and the focal corroboration-gate scope.
- `report_schedule[*]`: five fixed report events, including event/vehicle/target identity, event time, and admissibility metadata.
- `records[*]`: 120 common case-time records (2 cases x 60 times). They contain case/time, target, trailing-window bounds, eligible JSON arrays, distinct-vehicle arrays/counts, `k_cor`, threshold state, and GCS verification state.
- `records[*].variant_results[*]`: one result per method, including whether the GCS gate is enabled and the final `effective_authorization`.
- Window boundaries are `(window_start_exclusive, window_end_inclusive]`.
- Distinct-vehicle counts, not raw event counts, are compared with `k_cor`.

### `raw/e6_parameter_selection_raw.json`

- `frozen_design_metadata`: the tested Part-A values, five parameter ranges, fixed attack/recovery traces, gate definitions, and ranking contract used for this accepted selection.
- `part_a_complete_details[*]`: seven unique base-parameter configurations. Each item contains its parameter tuple, derived summary metrics, and the underlying detection/response/recovery trace details used to produce Table 3.
- `candidate_details[*]`: all 243 candidates. Besides the tuple and gate/result fields represented in CSV, each item retains `attack_audit` (vehicle trajectories, reports, network trajectory, delays, rejected-GCS count, duplicate-report evidence), `malicious_history_audit` (dual-source result and access decision), and `recovery_audit` (recovery trace and time).
- `candidate_details[*].infeasible_reasons` is a JSON array of specific failed rules. The simpler CSV `infeasible_reason` is empty, `INFEASIBLE_PREDEFINED_SECURITY_CONSTRAINT`, or `DYNAMIC_SAFETY_GATE_FAILURE`.
- `performance_equivalent_optima[*]`: the 15 candidates sharing the best `(d_local_s, d_network_s, t_recovery_s)` tuple.
- `selection_audit`: best tuple, tie-set size, robustness-neighbor maximum, center-distance filter, and final tie stage. It is research-method evidence, not a repository-integrity audit.
- `selection_result`: the selected `JPS-122` tuple and its metrics.
- `holdout_status` remains `HOLDOUT_NOT_RUN` in this selection-stage file because selection was completed before the separate hold-out run.

### `raw/e6_holdout_raw.json`

- The three uppercase preselection flags document that the fixed candidate was not reselected using hold-out results.
- `fixed_P_star_tuple` and `config_id` identify the already selected candidate.
- `hv_a` contains `trace_definition`, per-vehicle trajectories, reports, the network trajectory, transient false-action count, local/network event times and delays, rejected-GCS authorization count, duplicate-report evidence, and `holdout_gate_results`.
- `hv_r` contains `recovery_trace_definition`, stepwise dual-source recovery results, the first normal-access time, recovery duration, malicious-initial-history check, and `holdout_gate_results`.
- `overall_holdout_status` is the combined fixed-candidate result.

### `raw/e7_controlled_benchmark_raw.json`

- `environment` and `dependencies` retain platform, software identity, and version information but omit local executable/module paths.
- `fixture.seed`, derivation rules, labeled input lengths, and SHA-256 digests identify the deterministic byte strings supplied to the representative primitives. The digests correspond to benchmark inputs for SHA-256, ECDSA, IEnc, AES-GCM, ASCON, PUF, fuzzy-extractor, SCZ, BAZAM, and BCADS/MGSD representatives; they are not repository-file hashes or operational secrets.
- `fixture.derived_values` retains deterministic public/test values needed to reconstruct those inputs, including payload/AAD sizes and representative-specific fixtures.
- `conformance` retains representative-specific validity evidence: signature verification/tamper rejection, encryption round-trip/tamper rejection, PUF expected output, fuzzy-extractor syndrome/correction/key recovery, SCZ threshold reconstruction, pairing/bilinearity checks, and MGSD proof verification/tamper rejection where applicable. Retention supports interpretation of the benchmark; public-release suitability remains subject to human review.
- `operation_counts[method][primitive]` is the symbolic count multiplier. `operation_count_reconciliation` records how those counts map to the compared schemes.
- `method_computation_derivations[method]` retains every `count * median_ns` term and the summed method total.
- `communication_derivations[method].top_level_message_fields` contains field names and bit lengths; `.message_totals_bits` contains per-message sums; `.total_bits` is the method total. Nested components validate parent sizes and are not summed again.
- `primitive_timings` contains `warmup`, `iterations_per_batch`, seven `batch_means_ns`, `median_ns`, and `median_ms` for every primitive.
- Every `batch_means_ns` value is a per-operation batch mean. It is not an individual function-call observation.
- `primary_results` contains the five method-level values used in Figs. 11 and 12.
