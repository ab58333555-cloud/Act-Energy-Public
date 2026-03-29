# PSA-2 | Settlement & Accounting Layer
### Reward Computation Inputs · Periodic Ledger Closing · Dynamic Factors · Minting Formula

> **PSA-2 does exactly one thing:** take the work facts confirmed by PSA-1, compute the corresponding demand-currency reward, and close the ledger entry.
>
> PSA-2 does **not** verify facts (that is PSA-1's job), does **not** disburse funds (that is PSA-3's job), and does **not** adjudicate disputes (that is PSA-3's job).

---

## Global Conventions

| Rule | Description |
|------|-------------|
| **No new fact fields** | PSA-2 introduces no new fact fields. All settleable facts come exclusively from on-chain events confirmed by PSA-1. |
| **No hard-coded values** | PSA-2 hard-codes no values, thresholds, or formulas. All computation parameters are referenced via versioned `*_ref` pointers to external models/data. |
| **Versioned references** | Every reference MUST carry `{doc_hash, version, effective_epoch}`. Changes require `version+1` and take effect forward-only. |
| **Immutable finalization** | Once a ledger entry is `Finalized`, no field may be modified. Subsequent corrections may only be appended as `addendum` entries. |
| **SU definition** | A **Survival Unit (SU)** represents the minimum daily subsistence needs of an adult male (staple food + protein + micronutrients + water + hygiene supplies). The SU serves solely as the value-anchor baseline for the demand currency — used in Item 11 (monetary inflation/deflation rate calculation) — and does not imply direct distribution of physical goods. |

---

## Section 1 — Settlement Input Anchors

> **Strict rule: PSA-1 on-chain events are the sole source of truth.**

PSA-2 introduces no new fact fields. It only defines reference bindings, scope locks, and computation interfaces for settlement inputs. All facts usable for settlement **MUST** originate from events already on-chain in PSA-1-lite, including but not limited to:

- `ServiceSubmitted(project_id, version, service_tag, evidence_bundle_hash, participant_ref, root_id, lease_id, time_completed, ...)`
- `ServiceConfirmed(project_id, version, service_tag, verifier_id, result=ACCEPT, ...)`
- PSA-1 output fields: `points_count`, `captured_count`, `missing_count`, `sampling_commit_hash`, `sampling_compliance_status`, etc.

**Constraint (MUST):** Any ledger entry object MUST correspond to an `anti_replay_key` referenced by a PSA-1 `ServiceConfirmed(..., result=ACCEPT)` event. Unconfirmed entries MUST NOT be recorded.

---

### 1.1 Settlement Aggregate Key (MUST align with PSA-1)

Settlement aggregation, deduplication, and ledger object location MUST align with PSA-1's `anti_replay_policy_ref`:

```
anti_replay_key = H(project_id, version, service_type_id, canonical_target_ref_hash, unit_serial)

canonical_target_ref_hash = H(canonical_target_ref canonicalized)
  — must include at minimum: anchor_type, anchor_id, target_slot
  — governed by the Canonical rules defined in gateway_map_policy_ref
```

**Constraints:**
- `service_tag` / `evidence_bundle_hash` are index and audit anchors only — they MUST NOT substitute for `anti_replay_key`.
- `location_key` does not participate in `anti_replay_key` / settlement key by default.

---

## Section 2 — Reward Computation

Settlement may proceed on a service unit only after PSA-1's `ConfirmService` issues `ServiceConfirmed(..., ACCEPT)` for that unit.

---

### 2.1 Reward Computation Items (12 Items — All Interface Placeholders)

PSA-2 defines the following 12 computation items as the complete input structure for reward calculation. All items are interface placeholders; numeric values and computation methods are provided by the versioned computation model for the corresponding service type. **PSA-2 hard-codes nothing.**

---

#### Item 1 — Minimum Reward (`minimum_reward_ref`)

- **Definition:** The floor reward value for this service type under the current version.
- **Rule:** Exists independently; does not participate in the coefficient multiplication chain; does not enter the base reward calculation. Final reward = `max(minimum_reward, multiplication_result)`.

---

#### Item 2 — Service Type (`service_type_factor_ref`)

- **Definition:** The classification identifier that determines which versioned computation dataset applies to this service instance.
- **Note:** Encompasses not only the category of work but also the location where it occurs and the local scarcity of that type of work. The same category of work in different locations or at different scarcity levels uses different versioned datasets.
- **Role:** Determines which items among Items 3–12 apply and which version of the computation model/data each item references.

---

#### Item 3 — Base Reward (`base_reward_ref`)

- **Definition:** The standard reward baseline for this service type under normal conditions; the multiplicative base for all subsequent coefficients.
- **Rule:** Must be greater than 0. Version determined by service type (Item 2).

---

#### Item 4 — Intensity Factor (`intensity_factor_ref`)

- **Definition:** Adjustment coefficient reflecting the physical/mental exertion level of this work.
- **Rule:** Minimum value = 1. Higher intensity → larger coefficient.

---

#### Item 5 — Difficulty Factor (`difficulty_factor_ref`)

- **Definition:** Adjustment coefficient reflecting the operational complexity of this work.
- **Rule:** Minimum value = 1. Greater difficulty → larger coefficient.

---

#### Item 6 — Risk Factor (`risk_factor_ref`)

- **Definition:** Adjustment coefficient reflecting the personal safety risk posed to the worker.
- **Rule:** Minimum value = 1. Greater danger → larger coefficient.

---

#### Item 7 — Skill Factor (`skill_factor_ref`)

- **Definition:** Adjustment coefficient reflecting the professional qualification/skill threshold required for this work.
- **Rule:** Minimum value = 1. Service types with no professional requirements skip this item entirely (coefficient treated as 1). For those with requirements, higher thresholds → larger coefficient.

---

#### Item 8 — Duration Factor (`duration_factor_ref`)

- **Definition:** Adjustment coefficient or multiplier reflecting the duration of work performed.
- **Rule:** Applies only to service types with `completion_mode = FIXED_SERVICE_RESULT`. Service types with `completion_mode = RESULT_ACHIEVEMENT` skip this item entirely.

---

#### Item 9 — Attribution Factor (`attribution_factor_ref`)

- **Definition:** The deviation ratio between projected benefit and actual benefit — measuring how closely this work's real-world impact aligns with expectations.
- **Calculation:** `attribution_factor = actual_benefit / projected_benefit` (as a percentage). Exceeding projections → coefficient > 1; falling short → coefficient < 1.
- **Note:** Projected benefit is version-locked at project initiation. Actual benefit is output by PSA-3 after service confirmation and written to on-chain facts. PSA-2 references PSA-3's actual benefit output as input for this item.
- **Data source:** `attribution_factor_ref` references the versioned projected-benefit data and PSA-3's actual-benefit output interface.

---

#### Item 10 — Demand Factor (`demand_factor_ref`)

- **Definition:** Adjustment coefficient reflecting current societal demand for this service type. Higher demand → larger coefficient; demand saturation → coefficient approaches zero.
- **Note:** Works in conjunction with the Marginal Factor (Item 12). The demand factor reflects system-level aggregate demand state; the marginal factor reflects individual-level marginal contribution changes.
- **Data source:** `demand_factor_ref` references versioned demand data and the saturation suppression function.

---

#### Item 11 — Monetary Factor (`monetary_factor_ref`)

- **Definition:** Adjustment coefficient reflecting changes in the demand currency's purchasing power. Adjusted upward during inflation, downward during deflation, ensuring reward purchasing power is not distorted by currency fluctuations.
- **Note:** The SU serves as the demand currency's value-anchor baseline. The monetary inflation/deflation rate is calculated by observing actual commodity price changes corresponding to the SU. No direct distribution of goods is involved.
- **Data source:** `monetary_factor_ref` references the joint output of `SU_price_oracle` (observed price data for SU-corresponding commodities) and `monetary_calibration_policy_ref` (rules for deriving the inflation/deflation rate from price observations).

---

#### Item 12 — Marginal Factor (`marginal_factor_ref`)

- **Definition:** Adjustment coefficient reflecting how a worker's accumulated experience/contribution in a given service type affects their individual reward. Two modes: **marginal increasing** and **marginal decreasing**.

- **Marginal Increasing:** Applies to service types where "accumulated experience significantly enhances outcomes" (e.g., complex mechanical operations, long-term field management). More experience → greater skill-driven benefit → rising coefficient.

- **Marginal Decreasing:** Applies to service types with "no skill accumulation effect, or where long-term individual occupation blocks new participants" (e.g., cleaning, hauling, patrol). More experience → diminishing marginal contribution → declining coefficient, incentivizing workers to vacate roles.

- **Rule:** The criteria for marginal increase or decrease, the coefficient calculation method, and the calculation basis (by count/time/cumulative volume, etc.) are all defined by the service type's computation model. PSA-2 reserves the interface and hard-codes nothing.

---

### 2.2 Reward Computation Formula Skeleton (MUST)

```
Final Reward = max(
  minimum_reward,                ← Item 1  — independent floor
  base_reward                    ← Item 3
  × intensity_factor             ← Item 4
  × difficulty_factor            ← Item 5
  × risk_factor                  ← Item 6
  × skill_factor     (if applicable)  ← Item 7
  × duration_factor  (if applicable)  ← Item 8
  × attribution_factor           ← Item 9
  × demand_factor                ← Item 10
  × monetary_factor              ← Item 11
  × marginal_factor              ← Item 12
)
```

**Constraints (MUST):**
- Service type (Item 2) determines applicability and versioned-data references for each item. Non-applicable items (7, 8) are treated as coefficient = 1 and skipped.
- Minimum reward (Item 1) is an independent floor and does not enter the multiplication chain.
- The minimum value constraints (≥ 1) for Items 4–7 are enforced by each `*_factor_ref` computation model.
- Item 9 (attribution factor) depends on PSA-3's actual-benefit output. If PSA-3 data has not yet arrived, ledger status is marked `PARTIAL` and awaits completion.

---

## Section 3 — Minimum Settlement Dependency Catalog (MUST)

### D1 — Single Source of Facts

- Any settleable object MUST point to an `anti_replay_key` corresponding to PSA-1's `ServiceConfirmed(..., result=ACCEPT)`.
- PSA-2 MUST NOT introduce any new fact fields.
- Fact filter conditions MUST be commitment-bound via `confirmed_event_set_ref` and written into the closed ledger entry.

### D2 — Key Alignment and Deduplication

- Settlement aggregate keys MUST be fully consistent with PSA-1's `anti_replay_policy_ref`; self-created substitute keys are prohibited.
- The included set MUST provide `anti_replay_key_set_root` (Merkle root or equivalent commitment) for deduplication and auditability.

### D3 — Canonicalization Binding

- The canonicalization rules for `canonical_target_ref_hash` MUST reference `gateway_map_policy_ref`.
- If certain `service_type` values require additional fields to participate in `anti_replay_key`, this MUST be explicitly declared and versioned in `anti_replay_policy_ref`.

### D4 — Parameters Enter via Reference Only

- All 12 computation item references from Section 2.1, as well as `reward_model_id`, `share_root_ref`, `sector_weight_ref`, `baseline_ref`, and other settlement dependencies, MUST enter the closed ledger entry by reference.
- Every `ref` MUST carry `{doc_hash, version, effective_epoch}`.

### D5 — Per-Epoch Pointer Set Completeness

Each epoch's ledger entry MUST write a `settlement_input_pointer_set` and provide its hash:

```
settlement_input_pointer_set_hash = H(canonicalized(settlement_input_pointer_set))
```

`settlement_input_pointer_set` MUST include at minimum:

```
epoch_spec_ref

── Reward Computation Item References (all MUST if used) ──
minimum_reward_ref
service_type_factor_ref
base_reward_ref
intensity_factor_ref
difficulty_factor_ref
risk_factor_ref
skill_factor_ref                   (optional, if applicable)
duration_factor_ref                (optional, if applicable)
attribution_factor_ref
demand_factor_ref
monetary_factor_ref                (includes SU_price_oracle + monetary_calibration_policy_ref)
marginal_factor_ref

── Optional fields ──
oracle_set_hash                    (optional, if used)
sector_weight_ref                  (optional, if used)
share_root_ref                     (optional, if used)
benefit_anchor_set_ref             (optional, if used)
benefit_measure_oracle_ref         (optional, if used)
baseline_ref                       (optional, if used)
aggregation_policy_ref             (optional, if used)
compensation_topup_policy_ref      (optional, if used)
tail_comp_policy_ref               (optional, if used)
reward_model_id
payout_gate_ref                    (REQUIRED — see D9)
amount_calc_policy_ref             (MUST if any factor is used)
```

### D6 — Forward Effectivity and Version Lock

- Any `ref` upgrade MUST increment `version` by 1 and take effect only for subsequent epochs; retroactive effect on already-closed epochs is prohibited.
- `settlement_input_pointer_set_hash` for closed epochs MUST NOT be replaced or overwritten.

### D7 — Staleness Lock and Fallback

- For critical scope data, when freshness requirements are not met, the `last_published` version MUST be used (as defined by `staleness_lock_policy_ref`).
- `staleness_lock_policy_ref` MUST be versioned and written into `settlement_input_pointer_set`.

### D8 — Partial Closing Permitted, but No Fabrication

- When scope data is missing, generating a ledger skeleton and intermediate commitments is permitted, but fabricating numbers or free-form inference is STRICTLY PROHIBITED.
- The ledger summary MUST record `calc_completeness_status ∈ {COMPLETE, PARTIAL, MISSING}` and write `missing_ref_list_hash`.
- Subsequent completions may only be appended via `SettlementAmendment(addendum)`; original ledger content MUST NOT be overwritten.

### D9 — Disbursement Gate Externalized to PSA-3

- PSA-2 MUST NOT disburse funds directly. It records and closes ledger entries only.
- Closed ledger entries MUST reference `payout_gate_ref` (pointing to PSA-3's gate status interface).
- If `payout_gate_ref` determines `PAYOUT_BLOCKED` or `PAYOUT_DELAYED`, any disbursement module MUST be prohibited from releasing funds.

### D10 — Line Item Commitments and Provability

- This epoch's settlement detail MUST be committed on-chain via `settlement_lineitem_root`. Each individual line item MUST be provable via Merkle proof or equivalent.

Each `LineItem` MUST include at minimum:

| Field | Description |
|-------|-------------|
| `anti_replay_key` | Settlement target identifier |
| `participant_commit_hash` | Anonymized commitment (no forced disclosure of `root_id` plaintext) |
| `amount_commit_hash` | Amount commitment |
| `refs_used_digest` | Hash digest of all 12 computation item refs used |
| `amount_calc_policy_ref` | Computation policy reference |

### D11 — Ledger Attestation and Accountability

- Each epoch's closed ledger MUST carry a verifiable `settlement_attestation_sig` covering `settlement_id + settlement_lineitem_root + settlement_input_pointer_set_hash`.
- Signature threshold and signatory rules MUST reference `settlement_attestor_policy_ref` (versioned, forward-only).

### D12 — Dispute / Suspension / Audit Linkability

PSA-2 does not adjudicate disputes but MUST support auditability:

- `exclusion_set_ref` *(optional)*: Commitment to the set of excluded/suspended objects.
- `psa3_audit_ref` *(optional)*: Location reference when entering audit.

### D13 — ServiceCard Boundaries

`ServiceCard` MUST only index `ref` / `hash` / `proof` / `commit` values. It MUST NOT introduce new fact fields, issue rulings, or write interpretive conclusions.

---

## Section 4 — Ledger Closing and Settlement Record (`SettlementRecord`)

### 4.1 Epoch Identity and Naming (MUST)

| Field | Description |
|-------|-------------|
| `epoch_id` | Unique identifier for this settlement epoch (e.g., `YYYY-WW` / `YYYY-MM` / custom `epoch_seq`) |
| `scope` | Settlement scope: `global` / `sector` / `root_domain` / `project` / `region` / `zone`, etc. |
| `settlement_id` | `H(epoch_id, scope, version, settlement_input_pointer_set_hash, confirmed_event_set_ref, ts, nonce)` |

### 4.2 Settlement Objects and Inclusion Set (MUST)

| Field | Description |
|-------|-------------|
| `confirmed_event_set_ref` | Reference to PSA-1 events included in this epoch's settlement |
| `anti_replay_key_set_root` | Commitment root for all `anti_replay_key` values included in this epoch |
| `exclusion_set_ref` *(optional)* | Reference to excluded objects in this epoch |

> **Hard rule:** Only `anti_replay_key` values referenced by PSA-1's `ServiceConfirmed(..., ACCEPT)` may appear in the settlement ledger. Unconfirmed objects MUST NOT be included.

### 4.3 Settlement Basis Pointers (MUST)

| Field | Description |
|-------|-------------|
| `settlement_input_pointer_set` | Full field set as defined in Section 3 D5 |
| `settlement_input_pointer_set_hash` | `H(settlement_input_pointer_set canonicalized)` |

### 4.4 Settlement Line Item Commitments (MUST — anonymized, publicly disclosable)

| Field | Description |
|-------|-------------|
| `settlement_lineitem_root` | Merkle root of all line items in this epoch |
| `lineitem_schema_ref` *(optional, recommended)* | Reference to line item field structure rules |

**Each `LineItem` minimum fields:**

| Field | Description |
|-------|-------------|
| `anti_replay_key` | — |
| `participant_commit_hash` | Must not force disclosure of `root_id` in plaintext |
| `amount_commit_hash` | — |
| `refs_used_digest` | Hash digest of all 12 computation item refs used |
| `amount_calc_policy_ref` | — |

### 4.5 Disbursement Gate (MUST reference — not adjudicated by PSA-2)

| Field | Description |
|-------|-------------|
| `payout_gate_ref` | Points to PSA-3's disbursement gate determination interface |

> **Constraint:** If determined to be `BLOCKED` or `DELAYED`, disbursement modules MUST NOT release funds. PSA-2 does not adjudicate the reason — it only requires an auditable reference.

### 4.6 Signatures and Accountability (MUST — tiered by project level) ✍️

| Field | Description |
|-------|-------------|
| `settlement_attestation_sig` | Signature over `settlement_id + settlement_lineitem_root + settlement_input_pointer_set_hash` |
| `settlement_attestor_ref` | Reference to signatory |
| `settlement_attestor_policy_ref` | Signing and threshold rules (versioned, forward-only) |

**Project Levels (7 tiers):** `GLOBAL` / `NATIONAL` / `PROVINCIAL` / `CITY` / `DISTRICT` / `GRID` / `POINT`

**Tiered signing requirements:**

| Tier | Requirement |
|------|-------------|
| `GRID`, `POINT` | Automated signing by `system_node` / `gateway` / `auto_settlement_attestor` is permitted |
| `DISTRICT` and above | MUST reference PSA-3's tiered audit/acceptance rules and human + AI / multi-sig requirements as a precondition for issuing the closed ledger |

### 4.7 Immutability and Annotation (MUST)

- **`SettlementFinalized(settlement_id, ts, settlement_attestation_sig)`** — Once finalized, no existing field may be modified.
- **`SettlementAmendment(addendum_id, settlement_id, reason_code, new_refs, ts, sig)`** — Appending annotations or corrected references only; original ledger content MUST NOT be overwritten.

---

### 4.8 Service Card (`ServiceCard`) — Full PSA-1/2/3 Cross-Reference 📇

```
service_card_id  = H(anti_replay_key, project_id, version, epoch_id, nonce)
service_card_ref = reference/index entry for this card
```

A `ServiceCard` MUST be able to locate three packages:

#### Package A — PSA-1 Commitment Package (facts and evidence)

| Field | Description |
|-------|-------------|
| `psa1_fact_ref` | Points to `ServiceSubmitted` / `ServiceConfirmed` events and their evidence commitments |
| `psa1_proof_schema_ref` *(optional)* | If `RANDOM_SAMPLED_POINTS` or other `proof_schema` exists, must allow locating its output fields |

#### Package B — PSA-2 Accounting Package (within this ledger)

| Field | Description |
|-------|-------------|
| `settlement_id` / `epoch_id` / `scope` | — |
| `lineitem_proof_ref` | Merkle proof that this `anti_replay_key`'s `LineItem` belongs to `settlement_lineitem_root` |
| `amount_commit_hash` / `refs_used_digest` | — |

#### Package C — PSA-3 Audit Package

| Field | Description |
|-------|-------------|
| `psa3_audit_ref` *(optional)* | If under audit, locates the audit event and ruling commitment |
| `payout_gate_ref` *(required)* | Final disbursement gate status is output by PSA-3 |

> **Constraint:** `ServiceCard` is for cross-reference only. It MUST NOT introduce new fact fields or produce any ruling. Structural upgrades require `version+1` and are forward-only.

---

## Section 5 — Accounting Parameter Bindings

### 5.1 Minting Accounting Parameters

| Field | Description |
|-------|-------------|
| `reward_model_id` | Reward model identifier (replaceable and upgradeable) |
| `root_domain_id` | Owning root project domain identifier, used to trace domain share and saturation |
| `share_root_ref` | Root project domain share reference (versioned) |
| `saturation_policy` | Demand saturation suppression strategy (works with Item 10 to govern minting tapering to zero) |
| `saturation_metric` | Metric used to measure saturation |
| `damping_function_id` | Identifier of the damping/suppression function |
| `params_hash` | Hash of all saturation parameters |

### 5.2 Monetary Factor Parameters (corresponds to Item 11)

`monetary_factor_ref` data sources:

| Source | Description |
|--------|-------------|
| `SU_price_oracle` | Observed price data for SU-corresponding commodities (SU as value-anchor baseline) |
| `monetary_calibration_policy_ref` | Rules for deriving the inflation/deflation rate from price observation data |

---

## Section 6 — Dynamic Factor Interfaces

### 6.1 Attribution Factor Interface (corresponds to Item 9)

| Field | Description |
|-------|-------------|
| `attribution_model_ref` | Attribution model reference (projected vs. actual benefit computation scope) |
| `attribution_oracle_ref` | Attribution data scope/source reference (actual benefit confirmed and output by PSA-3) |
| `attribution_policy_ref` | Anomaly handling and fallback rules reference |

> **Constraint:** Attribution scope MUST be versioned and written into `settlement_input_pointer_set`.

---

### 6.2 Demand Factor Interface (corresponds to Item 10)

#### 6.2.1 Core Formula Skeleton

```
DemandRate  = total_project_tasks / total_project_duration
              ← from ProjectSpec (system's expected completion rate)

SupplyRate  = completed_tasks / elapsed_time
              ← from PSA-1 on-chain facts (actual completion rate)

DemandFactor = clamp(
  (DemandRate / (SupplyRate + 1))^α,
  floor = 0    (when saturation-zero threshold is triggered),
  cap   = demand_factor_cap   (default = 3, referenced via demand_factor_cap_ref)
)

effective_base_reward = base_reward × DemandFactor
```

> The demand factor acts on the base reward. All subsequent coefficients multiply against `effective_base_reward`.

**Zone behavior:**

| Supply Condition | DemandFactor | Effect |
|-----------------|--------------|--------|
| `SupplyRate < DemandRate` (undersupply) | `(1, cap]` | Reward elevated to attract labor |
| `SupplyRate ≈ DemandRate` (balanced) | `≈ 1` | Normal reward |
| `SupplyRate > DemandRate` (demand saturated) | `[0, 1)` | Reward reduced |
| `SupplyRate > tipping_point` (negative-effect zone) | `< 0` | Negative-effect process triggered (see 6.2.3) |

**Versioned reference fields:**

| Field | Description |
|-------|-------------|
| `demand_factor_cap_ref` | Upper cap reference for demand factor (default = 3; configurable per service type) |
| `demand_alpha_ref` | α parameter reference (versioned per service type; α < 1 = dampened response, α > 1 = amplified response) |
| `demand_saturation_policy_ref` | Saturation-zero threshold reference (defines conditions triggering `DemandFactor = 0`) |
| `demand_tipping_point_ref` | Negative-effect boundary reference (defines the `SupplyRate` value at which negative effects begin) |
| `assignment_policy_ref` | Scope reference for how completed tasks are credited as valid `SupplyRate` |

**Events (MUST):**

```
TaskTaken(project_id, version, participant_ref, t)
TaskStartHeartbeat(project_id, version, participant_ref, t)
TaskAutoReleased(project_id, version, participant_ref, t, reason_code)
```

> **Constraint (MUST):** `DemandRate` comes from `ProjectSpec`; `SupplyRate` comes from PSA-1 on-chain facts. Both are independently recomputable and auditable.

---

#### 6.2.2 Negative Effect Mechanism (`DemandFactor < 0`)

When `SupplyRate` exceeds the boundary defined by `demand_tipping_point_ref`, continued work of this type produces reverse effects, and the system enters the negative-effect handling process.

**The tipping point:** Each service type has a different negative-effect inflection point, defined by an external model versioned per service type. Examples:

> - Tree planting density too high → competition for water → survival rate collapses
> - Excessive fertilization → soil acidification → reverse damage to agricultural yield

**When a negative-effect record is triggered, obligations arise:**

1. **Repair obligation** — caused damage requires remediation
2. **Learning obligation** — participant must complete education/training on why the activity is harmful

> Both obligations may themselves become new service actions entering the PSA-1 process.

**PSA-2's role boundary:** PSA-2 only records the negative-effect event and reference pointers. After PSA-2 emits `NegativeEffectRecordCreated`, PSA-3 receives it and initiates the obligation determination process. All obligation recognition, adjudication, and enforcement happens within PSA-3.

**Events (MUST):**

```
NegativeEffectDetected(project_id, version, service_type_id, supply_rate, tipping_point_ref, ts)
NegativeEffectRecordCreated(anti_replay_key, project_id, demand_factor_value, ts, psa3_ref)
```

**Reference fields:**

| Field | Description |
|-------|-------------|
| `demand_tipping_point_ref` | Boundary definition reference (versioned, provided by service type computation model) |
| `negative_effect_policy_ref` | Negative-effect handling rules reference (obligation types, trigger conditions, etc. — defined by PSA-3) |

---

#### 6.2.3 Dual-Track Negative Effect Detection

Negative effects are discovered through two independent channels:

**Track A — Automated System Detection (primary defense)**

- **Source:** Continuous computation of `SupplyRate` from PSA-1 on-chain data; automatic flag when `demand_tipping_point_ref` threshold is exceeded.
- **Trigger:** System automatically generates `NegativeEffectDetected` event and enters the PSA-3 adjudication process.
- **Does not depend** on any party's report or proactive action.

**Track B — Public Feedback Channel (supplementary defense)**

- **Source:** Real-world negative effects not covered by system data (e.g., ecological impact, social impact, cross-boundary spillovers, etc.).
- **Entry point:** Public feedback channel defined by `public_feedback_policy_ref`, received by PSA-3.
- **PSA-2's role:** Only acknowledges this channel exists, referencing the `public_feedback_policy_ref` interface as a placeholder. Channel design, information reception, authenticity assessment, and adjudication are entirely PSA-3's responsibility.

| Field | Description |
|-------|-------------|
| `public_feedback_policy_ref` | Public feedback channel rules reference (defined by PSA-3; PSA-2 provides interface stub only) |

---

### 6.3 Work Characteristic Factor Interfaces (corresponds to Items 4–7 and Item 12)

| Field | Description |
|-------|-------------|
| `intensity_factor_ref` | Intensity factor reference (Item 4) |
| `difficulty_factor_ref` | Difficulty factor reference (Item 5) |
| `risk_factor_ref` | Risk factor reference (Item 6) |
| `skill_factor_ref` | Skill factor reference (Item 7, if applicable) |
| `marginal_factor_ref` | Marginal factor reference (Item 12) |
| `factor_composition_policy_ref` *(optional)* | Factor composition strategy reference |

---

## Section 7 — Value Weight Input Layer

### 7.1 Top-Level Structural References

| Field | Description |
|-------|-------------|
| `key_sectors_ref` | Key sector collection definition (versioned reference) |
| `sector_weight_ref` | Output reference for sector share values (versioned) |
| `root_domain_map_ref` | Root project domain definitions and sector mapping reference |
| `share_root_ref` | Root project domain share reference (versioned) |

### 7.2 Shock State Inputs

```
shock_index_global(t)    ∈ [0, 1]
shock_index_region(r, t) ∈ [0, 1]
```

Field sets, aggregation, anomaly handling, and supply-cut fallback are all version-bound via `OracleSetSpec`. Trigger/exit thresholds and other rules are version-bound via `activation_interlocks_ref`.

**PSA-2 only records:**

```
ShockModeEntered(scope, t, shock_index_hash, rule_version_hash)
ShockModeExited(scope, t, shock_index_hash, rule_version_hash)
```

All shock state changes are forward-only and non-retroactive.

### 7.3 Domain Classification and Weight Output

Every `RootDomain(D)` / `ProjectSpec` MUST be recomputably bound to exactly one `sector_id`.

- Classification results MUST be versioned, auditable, and forward-only.
- `sector_weight_ref` MUST be written into `settlement_input_pointer_set`.

### 7.4 Benefit Anchors (Value propagation: root project → leaf project)

| Field | Description |
|-------|-------------|
| `benefit_anchor_set_ref` *(MUST)* | Reference to the set of physical benefit anchors used this epoch |
| `benefit_measure_oracle_ref` *(MUST)* | Reference to benefit measurement scope/source |
| `anchor_weight_policy_ref` *(optional)* | Multi-anchor weighting strategy reference |
| `benefit_calibration_policy_ref` *(optional)* | Rules for forming unit-benefit value anchors |

> **Constraint:** `benefit_*` fields may only enter settlement by reference. Replacement requires `version+1` and is forward-only.

### 7.5 Baseline Rules

```
baseline(t, scope) = the benefit level stably achievable with no new contributions
net_benefit(t)     = benefit(t) - baseline(t)
```

> Long-tail rewards / gap compensation apply only to `net_benefit`.

`baseline_ref` MUST contain:

| Field | Description |
|-------|-------------|
| `source_scope` | — |
| `event_set_ref` | PSA-1 event set reference |
| `settlement_stat_ref` | PSA-2 statistics set reference |
| `aggregation_policy_ref` | Window / smoothing / denoising rules |

> **Hard rule:** The baseline may only be derived from the system's own accumulated data — PSA-1 on-chain facts, PSA-2 closed-ledger statistics, and system-internal recomputable aggregated outputs.

**Baseline reset:** When benefits have sustainably risen to a new level, the baseline may be raised accordingly. This is forward-only and non-retroactive for historical closed ledger epochs.

---

## Section 8 — Time Granularity and Quota Cadence

### 8.1 Accounting Epoch Definition (MUST)

`epoch_spec_ref` (versioned) must contain:

| Field | Description |
|-------|-------------|
| `epoch_type` | `WEEK` / `MONTH` / `CUSTOM` |
| `epoch_id_format` | — |
| `timezone_ref` | — |
| `epoch_boundary_rule_ref` | — |

### 8.2 Quota Cadence (for execution rhythm only — not a currency issuance ceiling)

| Field | Description |
|-------|-------------|
| `quota_schedule_ref` *(versioned)* | Project execution target/cadence reference |

> **Constraint:** MUST NOT be interpreted as a currency issuance ceiling or reserve pool budget. It is used solely for execution rhythm control and statistical scope alignment.

### 8.3 Staleness Lock

`staleness_lock_policy_ref` (versioned) must contain:

| Field | Description |
|-------|-------------|
| `data_freshness_requirement` | — |
| `fallback_rule` | — |
| `lock_scope` | — |

> **Rule:** If data does not meet freshness requirements, continue computing using the `last_published` version.

### 8.4 Handling Missing Scope Data

| Field | Description |
|-------|-------------|
| `calc_completeness_status` | `COMPLETE` / `PARTIAL` / `MISSING` (written to ledger summary) |
| `missing_ref_list_hash` | Commitment to the list of missing references |

**Events (MUST):**

```
SettlementComputedPartial(settlement_id, epoch_id, missing_ref_list_hash, ts)
SettlementFinalized(settlement_id, ts, sig)
SettlementAmendment(addendum_id, settlement_id, reason=MISSING_DATA_FILLED, new_refs, ts, sig)
```

> **Constraint:** Under `PARTIAL` / `MISSING` status, no disbursable amount conclusion may be output. Disbursement is gated by PSA-3.

---

## Section 9 — Coverage Gaps and Frozen Buckets 🧊

### 9.1 Coverage Status (MUST)

| Field | Description |
|-------|-------------|
| `coverage_map_ref` *(versioned)* | Mapping: `region_id → coverage_state ∈ {CONFIRMED, UNCONFIRMED}` |
| `coverage_policy_ref` | Status determination rules reference |

**Events:** `CoverageStateSet(...)` / `CoverageStateChanged(...)`

### 9.2 Frozen Buckets (MUST) 🧊

Shares from `UNCONFIRMED` regions are **not zeroed out** — they enter a `gap_bucket` for frozen holding.

| Field | Description |
|-------|-------------|
| `gap_bucket_ref` *(versioned)* | Mapping: `region_id → frozen_share_commit` |

**Rules:**

- `UNCONFIRMED`: Share enters `gap_bucket`; does not participate in current-epoch settlement.
- Once changed to `CONFIRMED`: `gap_bucket` is used only for coverage recovery calculations in subsequent epochs. It does **not** trigger retroactive top-ups, offsets, or clawbacks against past closed epochs.

**Events:** `GapBucketFrozen(...)` / `GapBucketReleased(...)` / `GapBucketRolledForward(...)`

---

## Section 10 — Long-Tail Reward Module

### 10.1 Positioning and Lifecycle

Long-tail rewards are a continuous revenue stream independent of immediate rewards, originating from ongoing benefits that a completed service continues to generate after its conclusion.

A service with long-tail rewards has two parallel lifecycle tracks:

**Track A — Immediate Reward (one-time closure)**
```
PSA-1 Confirmed → PSA-2 computes immediate reward (12 items)
  → PSA-3 audit → Disbursement → Closed
```

**Track B — Long-Tail Reward (continuously open)**
```
PSA-3 audit completed → Long-tail calculation starts
  → Computed and closed each epoch → PSA-3 gates each release
    → Continues until project no longer generates measurable benefit
```

**Constraints (MUST):**
- Immediate rewards and long-tail rewards are closed in separate ledger entries and MUST NOT be merged.
- The long-tail track is activated only after PSA-3 audit approval. If audit is not passed, long-tail rewards do not start.
- The `ServiceCard` MUST display long-tail reward status in a prominent, separate position from the immediate reward status.

---

### 10.2 Scope Boundary (MUST)

Long-tail rewards are bounded by project (`project_id`) and MUST NOT be merged across projects.

- Work completed under a project → long-tail benefit belongs to that project's benefit pool only.
- **Example:** Planting trees in the *Duoyi Forest Project* → long-tail benefit belongs to the Duoyi Forest Project benefit pool, not merged with other forest projects in Nimi County.

**Upward mapping path from project benefit pool:**

```
Project overall benefit (project_id boundary)
  → % share of owning sector
    → % share of owning domain
      → determines total allocatable long-tail reward for this project
```

> **Cap constraint:** The total long-tail rewards for all participants in a project MUST NOT exceed the project's overall benefit.

---

### 10.3 Long-Tail Reward Pool Computation Interface

| Field | Description |
|-------|-------------|
| `project_longtail_pool_ref` | Project long-tail reward pool definition reference (versioned). Includes: `project_id` boundary, benefit observation scope, sector/domain mapping rules, pool total cap rules. |
| `longtail_benefit_oracle_ref` | Project overall benefit observation data scope reference. Actual benefit continuously confirmed and output by PSA-3; PSA-2 references that output as computation input. |
| `longtail_sector_mapping_ref` | Project → sector → domain share mapping rules reference (versioned) |
| `longtail_pool_cap_policy_ref` | Pool total cap rules reference. **Constraint:** Total long-tail rewards MUST NOT exceed the overall benefit boundary of the `project_id`. |

---

### 10.4 Individual Long-Tail Reward Allocation Formula (MUST)

```
LongTailReward_i = ProjectPool × (w1 × IndividualScore_i + w2 × PoolShare)
```

| Variable | Description |
|----------|-------------|
| `ProjectPool` | Total allocatable long-tail reward for this `project_id` in this epoch (computed per Section 10.3) |
| `IndividualScore_i` | Worker i's individual weight score (see Section 10.5) |
| `PoolShare` | Equal share = `1 / number_of_participants_this_epoch` (or as defined by `longtail_pool_share_policy_ref`) |
| `w1 + w2 = 1` | Weights defined by `longtail_weight_policy_ref` (versioned, forward-only) |

**Weight evolution direction (written into policy; PSA-2 references):**

| Stage | w1 (individual) | w2 (pool share) |
|-------|----------------|-----------------|
| Early (insufficient technology to precisely track individual benefit) | 0.3 | 0.7 |
| Technology matured (individual benefit precisely trackable) | 0.7 | 0.3 |

Specific thresholds and transition conditions are version-defined by `longtail_weight_policy_ref`.

---

### 10.5 Individual Weight Score Interface (`IndividualScore`)

The individual weight score reflects a worker's personal contribution and quality, provided by an external computation model. PSA-2 only defines the interface.

`longtail_individual_score_ref` (versioned computation model reference) may include (non-exhaustive, model-defined):

| Input | Source |
|-------|--------|
| Service records | `anti_replay_key` set from PSA-1 (worker's contribution history in this project) |
| Sampling verification results | Sampling/inspection data output by PSA-3 (e.g., survival-rate spot checks) |
| Batch and zone information | Geographic batch information for each service instance |
| Reputation / history weight | Accumulated quality history for this worker in this project |

**Constraints (MUST):**
- Individual weight score data sources may only come from PSA-1 on-chain facts and PSA-3 confirmed outputs. PSA-2 introduces no new facts.
- In early stages where technology is insufficient, `IndividualScore` may be coarse, but the interface MUST be reserved to ensure smooth future upgrades to fine-grained individual measurement.

---

### 10.6 Anti-Collapse Mechanisms (Three Layers — All Interface References)

**Mechanism 1 — Minimum Validity Threshold (`minimum_validity_threshold_ref`)**

- **Definition:** If a worker's service quality in this project is clearly below the effective threshold (e.g., extremely low sampling survival rate), they are not eligible for long-tail rewards.
- **Data source:** PSA-3 sampling verification output.
- **Rule:** Version-defined by `minimum_validity_threshold_ref`; PSA-2 references without hard-coding thresholds.

**Mechanism 2 — Sampling Verification (`sampling_verification_ref`)**

- **Definition:** No full 100% verification required. Random sampling verifies individual contribution quality; results affect `IndividualScore`.
- **Data source:** PSA-3 executes sampling and outputs results; PSA-2 references that output.
- **Rule:** Sampling frequency, scope, and verification scope are version-defined by `sampling_verification_ref`.

**Mechanism 3 — Reputation History Weight (`reputation_weight_ref`)**

- **Definition:** A worker's historical quality record in this project accumulates into a reputation weight, which affects the upper and lower bounds of their `IndividualScore`.
- **Rule:** Accumulation method, decay rules, and bound constraints are version-defined by `reputation_weight_ref`; PSA-2 references without hard-coding.

---

### 10.7 Long-Tail Reward Ledger Structure

Long-tail rewards are closed in independent ledger entries, parallel to the immediate reward ledger structure.

| Field | Description |
|-------|-------------|
| `longtail_settlement_id` | `H(project_id, epoch_id, longtail_pool_ref_hash, participant_set_root, ts, nonce)` |
| `longtail_epoch_id` | Long-tail reward epoch identifier (may differ from immediate reward epoch) |
| `longtail_lineitem_root` | Merkle root of all individual long-tail reward line items this epoch |

**`longtail_settlement_input_pointer_set` (MUST include at minimum):**

```
project_longtail_pool_ref
longtail_benefit_oracle_ref
longtail_sector_mapping_ref
longtail_pool_cap_policy_ref
longtail_individual_score_ref
longtail_weight_policy_ref
longtail_pool_share_policy_ref
minimum_validity_threshold_ref
sampling_verification_ref
reputation_weight_ref
payout_gate_ref                   ← REQUIRED (PSA-3 release interface)
```

**Each `LongTailLineItem` minimum fields:**

| Field | Description |
|-------|-------------|
| `anti_replay_key` | Corresponding original service action |
| `participant_commit_hash` | Anonymized commitment |
| `longtail_amount_commit_hash` | Long-tail reward amount commitment for this epoch |
| `individual_score_commit` | Individual weight score commitment |
| `refs_used_digest` | — |

**Signatures and disbursement:**

| Field | Description |
|-------|-------------|
| `longtail_settlement_attestation_sig` | Signature over `longtail_settlement_id + longtail_lineitem_root + longtail_settlement_input_pointer_set_hash` |
| `payout_gate_ref` | PSA-3 continuously outputs release status; each long-tail epoch is independently gated |

> **Constraint (MUST):** Long-tail rewards continue operating until the termination conditions defined in `project_longtail_pool_ref` are triggered.

---

### 10.8 ServiceCard Long-Tail Reward Display (MUST)

For services with long-tail rewards, the `ServiceCard` MUST prominently display the following in a separate section:

| Field | Description |
|-------|-------------|
| `immediate_reward_status` | `FINALIZED` / `PENDING` / `BLOCKED` |
| `longtail_reward_status` | `ACTIVE` / `NOT_STARTED` / `TERMINATED` |
| `longtail_accumulated_ref` | Cumulative long-tail reward reference (auditable history per epoch) |
| `longtail_current_epoch_ref` | Current epoch long-tail reward computation status reference |
| `longtail_pool_info_ref` | Owning project benefit pool info reference (allows viewing project-level benefit status) |

> **Constraint:** `ServiceCard` display fields still only index `ref` / `hash` / `commit` values. No raw amount figures in plaintext. No ruling implications.

---

*Author: Zhengyang Pan (潘正阳) | BBBigradish (一根大萝卜)*
