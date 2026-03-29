# PSA-3 · Two-Stage Audit with On-chain Trail 🧾⛓️

**Author:** Zhengyang Pan (潘正阳) · BBBigradish (一根大萝卜)

---

## Purpose

Complete a two-stage audit before any payout is disbursed:

1. **PSA-1** — Confirm the authenticity and validity of service behaviour.
2. **PSA-2** — Re-verify the settlement basis and all calculation dependencies.

Every step of the audit process is recorded on-chain (replayable, accountable, and statistically trackable). The output is a `payout_gate` consumed by the disbursement layer and referenced by PSA-2.

---

## Boundary Constraints (MUST)

- PSA-3 **must not** introduce new factual fields. Facts originate from PSA-1; settlement basis and dependency pointers originate from PSA-2.
- PSA-3 **only** performs verification, grading, escalation, and gate output. Narrative conclusions **must not** replace on-chain evidence commitments.

---

## 0 · Input Anchors (MUST)

### 0.1 PSA-1 Inputs (Fact Source)

| Field | Description |
|---|---|
| `anti_replay_key` | Primary key |
| PSA-1 fact event references | `ServiceSubmitted` / `ServiceConfirmed` / `proof_schema` output fields (`points_count`, `captured_count`, `sampling_compliance_status`, etc.) |
| `evidence_bundle_hash` | Evidence bundle commitment |

### 0.2 PSA-2 Inputs (Settlement Source)

| Field | Description |
|---|---|
| `settlement_id` | Closing account ID |
| `settlement_input_pointer_set` + `settlement_input_pointer_set_hash` | Current-epoch pointer set and its hash |
| `lineitem_proof_ref` | Proof that `anti_replay_key` belongs to `settlement_lineitem_root` |
| `refs_used_digest` | Digest of key references used |
| `amount_commit_hash` | Amount commitment hash |

### 0.3 Project Grade & Default Audit Mode

| Field | Allowed Values |
|---|---|
| `project_grade` | `GLOBAL` · `NATIONAL` · `PROVINCIAL` · `CITY` · `DISTRICT` · `GRID` · `POINT` |

> `project_grade` **must** be traceable to a `ProjectSpec` / Registry (by reference).

---

## 1 · Audit State Machine (MUST) 🧩

Each `anti_replay_key` moves through the following audit states:

```
audit_state ∈ {
  OPENED,
  STAGE1_RUNNING,     STAGE1_NEEDS_HUMAN,  STAGE1_PASSED,  STAGE1_FAILED,
  STAGE2_RUNNING,     STAGE2_NEEDS_HUMAN,  STAGE2_PASSED,  STAGE2_FAILED,
  GATE_ISSUED,
  CLOSED
}
```

**Constraints (MUST):**

- Every state transition **must** be recorded as an on-chain event.
- The same `anti_replay_key` **must not** skip states within the same epoch unless the governing policy explicitly permits it and the exception is recorded on-chain.

---

## 2 · Audit Session & Case Opening (MUST) 📌

Each audit is identified by an `audit_session_id` (replayable to the input snapshot):

```
audit_session_id = H(anti_replay_key, epoch_id, settlement_id,
                     settlement_input_pointer_set_hash, nonce)
```

### Required On-chain Events

```
AuditSessionOpened(
  audit_session_id,
  anti_replay_key,
  epoch_id,
  project_grade,
  settlement_id,
  settlement_input_pointer_set_hash,
  evidence_bundle_hash,
  ts
)

AuditStateUpdated(audit_session_id, from_state, to_state, reason_code, ts)
```

> Once opened, the session immediately enters `STAGE1_RUNNING`.

---

## 3 · Stage 1 Audit — PSA-1 Authenticity / Validity Confirmation (MUST) ✅

**Objective:** Confirm whether the service was genuinely completed, whether evidence is usable, and whether the result is valid.

---

### 3.1 Audit Mode Routing (MUST)

| Condition | `review_mode` |
|---|---|
| `project_grade < DISTRICT` | `AI_ONLY` (escalates to human only on `UNCLEAR` / `ANOMALY` / conflict) |
| `project_grade ≥ DISTRICT` | `AI_PLUS_HUMAN` |

**Required Event:**

```
Stage1Started(audit_session_id, review_mode, policy_refs_digest, ts)
```

---

### 3.2 Stage 1 Check Items (Structured Interface — Details Are Referenced, Not Hardcoded)

Three check categories are required. Each **must** produce a status and be recorded on-chain.

#### (A) Random-Sample Photo Check

```
sampling_photo_check_policy_ref    [required]
→ sample_photo_status ∈ { PASS, FAIL, UNCLEAR }
```

#### (B) Completion Evidence Check (completion photo / video / panoramic image)

```
finish_evidence_check_policy_ref   [required]
→ finish_evidence_status ∈ { PASS, FAIL, UNCLEAR }
```

#### (C) Outcome Check

```
outcome_check_policy_ref           [required]
→ outcome_status ∈ { PASS, FAIL, UNCLEAR }
```

**Per-item On-chain Event (MUST):**

```
EvidenceCheckPerformed(
  audit_session_id,
  check_type         ∈ { SAMPLE_PHOTO, FINISH_EVIDENCE, OUTCOME },
  status             ∈ { PASS, FAIL, UNCLEAR },
  policy_ref,
  input_ref_digest,       // digest pointing to PSA-1 proof_schema / event fields
  evidence_ref_optional,  // optional: commitment to sub-evidence set
  ts
)
```

---

### 3.3 Stage 1 Human Escalation (MUST — Grade-Constrained)

**Trigger conditions (minimum scope, MUST):**

- `project_grade < DISTRICT` **and** any check returns `UNCLEAR`, or a conflict / anomaly is detected → enter `STAGE1_NEEDS_HUMAN`
- `project_grade ≥ DISTRICT` defaults to `AI_PLUS_HUMAN`; this trigger is not required

**Required Events:**

```
HumanReviewRequested(
  audit_session_id,
  stage = 1,
  reason_code,
  required_inputs_ref_optional,
  ts
)

AuditStateUpdated(audit_session_id, STAGE1_RUNNING, STAGE1_NEEDS_HUMAN, reason_code, ts)
```

---

### 3.4 Stage 1 Grading & Output (MUST)

```
audit_grade_stage1 ∈ { COMPLIANT, SUSPICIOUS, ANOMALY, INVALID }
```

**Minimum grading rules (MUST):**

| Condition | Grade |
|---|---|
| Any `FAIL` — recoverable (per policy) | `ANOMALY` (human review / supplemental evidence required) |
| Any `FAIL` — not recoverable | `INVALID` |
| Any `UNCLEAR` present | `SUSPICIOUS` (may trigger human review) |
| All `PASS`, no conflicts | `COMPLIANT` |

**Required Event:**

```
Stage1GradeIssued(
  audit_session_id,
  audit_grade_stage1,
  sample_photo_status,
  finish_evidence_status,
  outcome_status,
  policy_refs = {
    sampling_photo_check_policy_ref,
    finish_evidence_check_policy_ref,
    outcome_check_policy_ref
  },
  reason_code,
  ts
)
```

**State Transitions (MUST):**

| Grade | Transition |
|---|---|
| `COMPLIANT` | → `STAGE1_PASSED` |
| `SUSPICIOUS` or `ANOMALY` | → `STAGE1_NEEDS_HUMAN` or remain in `STAGE1_RUNNING` (per policy — must be recorded) |
| `INVALID` | → `STAGE1_FAILED` |

```
AuditStateUpdated(
  audit_session_id, *, STAGE1_PASSED | STAGE1_FAILED | STAGE1_NEEDS_HUMAN,
  reason_code, ts
)
```

---

## 4 · Stage 2 Audit — PSA-2 Settlement Basis / Calculation Dependency Verification (MUST) 🧮

**Entry condition (MUST):** Stage 2 **only** begins after `STAGE1_PASSED`.

**Required Events:**

```
Stage2Started(audit_session_id, policy_refs_digest, ts)

AuditStateUpdated(
  audit_session_id, STAGE1_PASSED, STAGE2_RUNNING,
  reason_code = STAGE1_OK, ts
)
```

---

### 4.1 Calculation Dependency Definition (MUST)

> **Calculation dependencies** = all basis documents, models, data, factors, versions, and pointer sets referenced by PSA-2 in its computation.

PSA-3 must verify that these dependencies match reality and are consistent with the settlement pointer set.

**Dependency items (MUST audit if enabled):**

| Category | Fields |
|---|---|
| Reward model | `reward_model_id` / `amount_calc_policy_ref` |
| Oracle set | `oracle_set_hash` (or `OracleSetSpec` pointer) |
| Share / sector / baseline | `share_root_ref` · `sector_weight_ref` · `baseline_ref` |
| Benefit anchors | `benefit_anchor_set_ref` · `benefit_measure_oracle_ref` |
| **Work differentiation factors** (evolve continuously — referenced, not hardcoded) | `work_profile_ref` · `intensity_profile_ref` · `difficulty_profile_ref` · `risk_factor_ref` · `professionalism_profile_ref` · `non_interruptible_profile_ref` · `duration_profile_ref` · `factor_composition_policy_ref` |

---

### 4.2 Pointer Set Consistency Verification (MUST)

**Objective:** Prevent unauthorised basis substitution at settlement time.

**Required Event:**

```
SettlementPointerSetVerified(
  audit_session_id,
  settlement_input_pointer_set_hash,
  refs_used_digest,
  verification_result  ∈ { PASS, FAIL, UNCLEAR },
  reason_code,
  ts
)
```

---

### 4.3 Dependency Conformance Review (MUST — Details Referenced)

```
dependency_audit_policy_ref    [required]
  // defines how to determine whether dependencies match reality / are correct
  // referenced only — thresholds must not be hardcoded here

→ dependency_audit_status ∈ { PASS, FAIL, UNCLEAR }
```

**Per-group On-chain Event (MUST):**

```
DependencyCheckPerformed(
  audit_session_id,
  dependency_group  ∈ { MODEL, ORACLE, SHARE, BASELINE, BENEFIT, WORK_FACTORS, OTHER },
  status            ∈ { PASS, FAIL, UNCLEAR },
  policy_ref        = dependency_audit_policy_ref,
  checked_refs_digest,      // digest of refs covered in this check
  evidence_ref_optional,    // optional: commitment to verification materials
  ts
)
```

---

### 4.4 Stage 2 Grading & Output (MUST)

```
audit_grade_stage2 ∈ { COMPLIANT, SUSPICIOUS, ANOMALY, INVALID }
```

**Minimum grading rules (MUST):**

| Condition | Grade |
|---|---|
| `pointer_set_verification_result == FAIL` | `ANOMALY` (at minimum: delay) |
| `dependency_audit_status == FAIL` — correctable (per policy) | `ANOMALY` |
| `dependency_audit_status == FAIL` — not correctable | `INVALID` |
| Any `UNCLEAR` present | `SUSPICIOUS` |
| All pass and consistent | `COMPLIANT` |

**Required Event:**

```
Stage2GradeIssued(
  audit_session_id,
  audit_grade_stage2,
  pointer_set_verification_result,
  dependency_audit_status,
  policy_refs = { dependency_audit_policy_ref },
  reason_code,
  ts
)
```

**State Transitions (MUST):**

| Grade | Transition |
|---|---|
| `COMPLIANT` | → `STAGE2_PASSED` |
| `INVALID` | → `STAGE2_FAILED` |
| `SUSPICIOUS` or `ANOMALY` | → `STAGE2_NEEDS_HUMAN` or remain waiting (per policy — must be recorded) |

```
AuditStateUpdated(
  audit_session_id, *, STAGE2_PASSED | STAGE2_FAILED | STAGE2_NEEDS_HUMAN,
  reason_code, ts
)
```

---

## 5 · Aggregate Verdict — Payout Gate Output (MUST) 🚦

```
payout_gate_status ∈ { PAYOUT_ALLOWED, PAYOUT_DELAYED, PAYOUT_BLOCKED }
```

**Minimum rules (MUST):**

| Condition | Gate Status |
|---|---|
| `STAGE1_FAILED` or `STAGE2_FAILED` | `PAYOUT_BLOCKED` |
| Any unresolved `NEEDS_HUMAN` / `UNCLEAR` / `ANOMALY` | `PAYOUT_DELAYED` |
| `STAGE1_PASSED` **and** `STAGE2_PASSED` **and** no open Dispute / Hold / AuditOpen | `PAYOUT_ALLOWED` |

**Required On-chain Events (replayable):**

```
PayoutGateIssued(
  audit_session_id,
  anti_replay_key,
  epoch_id,
  settlement_id,
  status           = payout_gate_status,
  reason_code,
  stage1_grade     = audit_grade_stage1,
  stage2_grade     = audit_grade_stage2,
  policy_refs_digest,
  ts
)

AuditStateUpdated(audit_session_id, *, GATE_ISSUED, reason_code, ts)
```

---

## 6 · Session Close & Reference Output (MUST) 🔚

```
AuditSessionClosed(audit_session_id, final_status = payout_gate_status, ts)

AuditStateUpdated(audit_session_id, GATE_ISSUED, CLOSED, reason_code, ts)
```

**Interface constraints (MUST):**

- The `payout_gate_ref` output from PSA-3 (pointing to the `PayoutGateIssued` event / object) is a **necessary precondition** for any disbursement.
- If `status` is `DELAYED` or `BLOCKED`, any disbursement module **MUST** refuse to pay. **When in doubt, delay — never overpay.**

---

## 7 · Boundary Alignment with PSA-2 / ServiceCard (MUST) 📇

- PSA-2 **must** reference `payout_gate_ref` (from PSA-3) at settlement closing, for replay of why a payout was allowed, delayed, or blocked.
- ServiceCard indexes only: `psa1_fact_ref` / `lineitem_proof_ref` / `psa3_audit_ref` (`payout_gate_ref`).
- Human-review narrative conclusions from PSA-3 **must not** be written back into PSA-2 or ServiceCard — only `ref` / `hash` / `proof` are permitted.

---

## 8 · Human Review — Maintainer Jury (MUST · Public On-chain) 🗳️🧑‍⚖️

**Purpose:** When AI cannot reach a clear verdict, or when anomalies / disputes arise, a randomly selected panel drawn from the maintainers with an obligation to the project performs human review. The panel votes on (a) the authenticity and validity of service behaviour (PSA-1 side) and (b) the correctness of settlement dependencies (PSA-2 side), and issues a final verdict and payout gate decision.

This section defines: selection rules, voting rules, weights, thresholds, on-chain events, and post-verdict routing. Specific criteria for assessing photos, panoramas, and results are implementation details, versioned and referenced via `evidence_review_policy_ref`.

---

### 8.0 References & Versioned Bindings (MUST — ref-only)

| Reference | Description |
|---|---|
| `human_review_process_ref` | Human audit process reference (this section's `{doc_hash, section_path, version, effective_epoch}`) |
| `vote_rule_ref` | Voting rules reference (threshold / weights / tally basis / invalid-vote handling; versioned) |
| `maintainer_registry_ref` | Maintainer registry and obligation mapping (who holds obligations for which scope / domain; versioned) |
| `selection_policy_ref` | Selection rules reference (sampling ratio / minimum panel size / backfill rules / randomness proof; versioned) |
| `higher_level_policy_ref` | Higher-level maintainer participation rules (applicable scope / headcount cap / eligibility; versioned) |
| `evidence_review_policy_ref` *(optional)* | Evidence review methodology (random-sample photos, completion photos, outcome assessment, etc.; versioned) |
| `remediation_policy_ref` *(optional)* | Post-rejection remediation strategy (suspend / freeze / PSA-1 rework / PSA-2 dependency correction; versioned) |

**Constraint (MUST):** All references above are **locked** at project version activation. Any upgrade requires `version + 1` and is **forward-only** (effective for subsequent epochs only).

---

### 8.1 Trigger Conditions (MUST)

Human audit is entered when **any one** of the following is true:

- AI grading outputs `ANOMALY`, or is inconclusive, or crosses a human-escalation threshold (defined in the PSA-3 main flow).
- A governance state — `Dispute`, `Hold`, `AuditOpen`, etc. — requires human intervention (governed by `governance_ruleset_hash` / `freeze_policy`).

**Required Event:**

```
HumanAuditOpened(
  audit_id,
  scope,
  anti_replay_key_or_project_id,
  epoch_id,
  trigger_reason_code,
  refs_digest,
  ts
)
```

---

### 8.2 Audit Object & Scope (MUST)

Human review **must** cover both stages (sub-audits may be opened separately):

| Stage | Scope |
|---|---|
| **Stage 1** | PSA-1 — service authenticity / validity (was the service genuinely completed? is the evidence credible? does the project information correspond?) |
| **Stage 2** | PSA-2 — settlement dependency review (model version, data source, share allocation, work-type / intensity / difficulty / risk / non-interruptibility / professionalism / duration — are dependencies consistent with reality?) |

**Required Fields:**

```
audit_stage           ∈ { STAGE1_PSA1_FACT, STAGE2_PSA2_DEPENDENCY, BOTH }
psa1_ref_bundle       // PSA-1 fact / evidence reference set
                      //   (ServiceSubmitted/Confirmed + evidence_bundle_hash + proof_schema outputs)
psa2_calc_ref_bundle  // PSA-2 computation result and basis reference set
                      //   (settlement_lineitem proof + refs_used_digest
                      //    + settlement_input_pointer_set_hash)
```

**Constraint (MUST):** Human review only verifies whether references, facts, and basis documents match and are reasonable. PSA-3 **must not** introduce new factual fields.

---

### 8.3 Maintainer Set & Selection (MUST)

#### 8.3.1 Local Maintainer Set (MUST)

```
MaintainerSet_local = Maintainers(
  scope  = project.location,
  domain = project.domain,
  grade  = project_grade
)
// resolved via maintainer_registry_ref
```

#### 8.3.2 Panel Size Rules (MUST)

```
n_total    = |MaintainerSet_local|
n_pick_raw = round(0.4 × n_total)   // round half-up
n_pick     = max(3, n_pick_raw)     // minimum 3
```

**Required Event:**

```
MaintainerPanelSelectionRequested(
  audit_id, n_total, n_pick, selection_policy_ref, ts
)
```

#### 8.3.3 Backfill When Local Set Is Insufficient (MUST)

If `MaintainerSet_local` cannot satisfy `n_pick`:

```
SameTypeSameGradePool = Maintainers(domain = project.domain, grade = project_grade)
```

Supplement from this pool until `n_pick` is satisfied, or raise a "minimum panel size not met" exception (defined by `selection_policy_ref`).

**Required Event:**

```
MaintainerPanelSupplemented(
  audit_id, shortage_count, supplement_pool_ref, ts
)
```

#### 8.3.4 Higher-Level Maintainer Inclusion for DISTRICT and Above (MUST)

When `project_grade ∈ { DISTRICT, CITY, PROVINCIAL, NATIONAL, GLOBAL }`:

```
n_higher_min = 1
n_higher_max = floor(n_pick / 10)   // must not exceed 1/10 of local panel size
n_higher     = clamp(rule_defined, n_higher_min, n_higher_max)
               // specific value governed by higher_level_policy_ref
```

If no higher-level maintainers exist, apply the "no higher-level participation" weighting rule (see §8.5).

**Required Event:**

```
HigherLevelMaintainerInvited(
  audit_id, n_higher, higher_level_policy_ref, ts
)
```

#### 8.3.5 Verifiable Selection (MUST)

Selection must be auditable — includes a randomness source and proof (plaintext random seed need not be public, but a verifiable commitment is required).

**Required Fields:**

```
selection_proof_ref    // verifiable random selection proof reference
panel_members_hash     // commitment hash of the panel member list
```

**Required Event:**

```
MaintainerPanelSelected(
  audit_id, panel_members_hash, selection_proof_ref, ts
)
```

---

### 8.4 Voting Mechanism (MUST)

#### 8.4.1 Vote Options & Reasoning Requirement (MUST)

Every panel member **must** cast a vote:

```
vote ∈ { ACCEPT, REJECT }
```

- `vote_reason_ref` **[required]** — reasoning must be stated explicitly and published on-chain (structured `reason_code` + hash or full text of a written summary).
- Unsubstantiated approvals or rejections are **not permitted**.

**Required Event:**

```
VoteCast(audit_id, voter_ref, vote, vote_reason_ref, ts)
```

#### 8.4.2 Arguments & Submissions (MUST)

Any maintainer may submit an audit argument or analysis, individually or jointly, to influence the panel's direction.

**Required Event (MUST — recommended even when optional in content):**

```
AuditArgumentSubmitted(audit_id, proposer_ref, argument_ref, ts)
```

---

### 8.5 Vote Weights & Pass Threshold (MUST)

#### 8.5.1 Base Threshold (MUST)

```
pass_threshold = strictly greater than 2/3
```

Tally rules are defined and version-locked in `vote_rule_ref` (must not be changed retroactively).

#### 8.5.2 No Higher-Level Participation (MUST)

When `n_higher == 0`:

```
w_i              = 1 / n_pick            // equal weight for all participants
weighted_accept  = Σ(w_i for vote == ACCEPT)

if weighted_accept > 2/3  →  ACCEPT
else                       →  REJECT
```

#### 8.5.3 With Higher-Level Participation (MUST)

When `n_higher ≥ 1`:

```
W_higher = 0.26                          // higher-level group weight
W_local  = 0.74                          // local group weight

w_higher_i = W_higher / n_higher         // per higher-level member
w_local_i  = W_local  / n_local          // per local member

weighted_accept = Σ(w_higher_i for higher votes == ACCEPT)
                + Σ(w_local_i  for local  votes == ACCEPT)

if weighted_accept > 2/3  →  ACCEPT
else                       →  REJECT
```

> **Note:** The `0.26 / 0.74` split is a hard rule in this section. To make it a configurable parameter in the future, move it into `vote_rule_ref` — but it must still be versioned and forward-only.

#### 8.5.4 Verdict Output (MUST — Public On-chain)

```
VoteClosed(audit_id, vote_stats_hash, ts)

HumanAuditVerdictCommitted(
  audit_id,
  outcome          ∈ { ACCEPT, REJECT },
  weighted_accept,
  vote_stats_hash,
  evidence_bundle_hash,
  refs_digest,
  ts
)
```

---

### 8.6 Full On-chain Public Record (MUST)

**Hard rule:** The entire human audit process and its outcome **must** be published on-chain — including selection, member commitment, votes, reason references, statistical digest, and verdict.

**Minimum required on-chain event set:**

```
HumanAuditOpened(...)
MaintainerPanelSelected(...)
VoteCast(...) + vote_reason_ref
VoteClosed(...)
HumanAuditVerdictCommitted(...)
```

> Participant identity may be real-name or pseudonymous-commitment per system policy. This section assumes public identity by default; privacy extensions may be added at the registry layer in the future.

---

### 8.7 Post-Verdict Routing (MUST)

#### 8.7.1 Verdict: ACCEPT ✅

**Actions (MUST):**

- Issue payout gate: `payout_gate_status = PAYOUT_ALLOWED`
- Bundle PSA-1 + PSA-2 + PSA-3 into a **ServiceCard**
- Follow PSA-2 ServiceCard rules: index only, do not adjudicate, do not add new factual fields

**Required Events:**

```
PayoutGateIssued(
  anti_replay_key, epoch_id,
  PAYOUT_ALLOWED,
  reason_code = HUMAN_AUDIT_ACCEPT,
  audit_ref   = audit_id
)

ServiceCardFinalized(
  service_card_id,
  anti_replay_key,
  psa1_ref,
  psa2_ref,
  psa3_audit_ref = audit_id,
  ts
)
```

**Constraint (MUST):** Once `ServiceCardFinalized` is emitted, the minting / disbursement process may begin. Minting and payment are executed at the execution layer, but the gate **must** be open first.

---

#### 8.7.2 Verdict: REJECT ❌

**Actions (MUST):** Reason must be stated; routing is determined by root-cause classification.

```
reject_reason_ref    [required]
// structured reason_code + explanation text
```

---

##### 8.7.2.1 Issue Originates in PSA-1 (Service Authenticity / Validity Problem)

**Handling rules (MUST):**

- **If the work window has already closed** (`work_window_ref` / `work_open_policy_ref`):
  - Freeze the project (`FreezeProject`). The service provider retains the task and may resume after their chosen `work_start_time`.
  - Hard upper limit on total task-hold duration: **MAX_HOLD = 72 hours**.
  - The service provider may declare a `work_start_time` to avoid being forced into an absent work window.

- **If the work window has not yet closed:**
  - Place on hold (`Hold`). Require the service provider to resolve the issue. If the window subsequently closes, convert to a freeze (see above).

**Required Events:**

```
ServiceRejected_PSA1(audit_id, anti_replay_key, reject_reason_ref, ts)

TaskHeld(anti_replay_key, reason_code, until_ts_optional, ts)

ProjectFrozen(
  project_id, scope,
  reason_code    = PSA1_REWORK_REQUIRED,
  evidence_ref   = reject_reason_ref,
  ts
)

WorkerWorkStartDeclared(anti_replay_key, chosen_start_time, ts)

HoldExceededToFreeze(anti_replay_key, ts)

TaskHoldReleasedOrExpired(anti_replay_key, reason_code, ts)
```

**Hard constraint (MUST):** Total cumulative hold / freeze duration for any single `anti_replay_key` **must not exceed 72 hours**. Upon expiry, the task must be released or the process reset (governed by `remediation_policy_ref`).

---

##### 8.7.2.2 Issue Originates in PSA-2 (Incorrect Calculation Dependency / Version / Data Source)

**Handling rules (MUST):**

- Place on hold (`Hold`). Re-compute the reward immediately once the correct dependencies are obtained.
- Submit a system request to correct the project's calculation dependencies to the right version (`version + 1`, forward-only).
- **Note:** Historical settlement records are immutable — corrections are appended as addenda (`addendum`), not overwrites (PSA-2 rules apply).

**Required Events:**

```
ServiceRejected_PSA2(audit_id, anti_replay_key, reject_reason_ref, ts)

PayoutGateIssued(
  anti_replay_key, epoch_id,
  PAYOUT_DELAYED,
  reason_code = PSA2_DEPENDENCY_MISMATCH,
  audit_ref   = audit_id
)

DependencyCorrectionRequested(
  project_id, from_refs_digest, to_refs_digest, reason_ref, ts
)

RecomputeRequested(anti_replay_key, correct_refs_digest, ts)

SettlementAmendment(
  addendum_id, settlement_id,
  reason_code = DEPENDENCY_CORRECTED,
  new_refs, ts, sig
)
// executed at the PSA-2 layer
```

---

### 8.8 Payout Gate Linkage (MUST)

| Condition | Gate Status |
|---|---|
| `HumanAuditVerdictCommitted.outcome == ACCEPT` | `PAYOUT_ALLOWED` |
| `outcome == REJECT` | At minimum `PAYOUT_DELAYED`; if deemed unrecoverable or void → `PAYOUT_BLOCKED` (per `remediation_policy_ref`) |
| Any open `Dispute` / `Hold` / `AuditOpen` state | Gate **must** be `BLOCKED` or `DELAYED` |

> **When in doubt, delay — never overpay.**
