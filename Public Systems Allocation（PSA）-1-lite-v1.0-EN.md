# PSA-1-lite — Minimal Data Structures & Minimal Verification Flow 🪶

> **PSA-1-lite does one thing only:** close the engineering loop on *verifiability of service outcomes*.
> All value calculation, weighting, token minting, cycle accounting, and audit accountability belong to PSA-2 / PSA-3 — this layer only holds reference interfaces for those concerns.

---

## Global Conventions

- **`*_ref` fields** — PSA-1 only verifies that a reference exists and that its signature/version is verifiable. It does not parse the content. Changes require a new version; forward-only.
- **Rejection / `risk_flag` wording** — governed by the referenced policy. If no policy is declared, the default is *reject*.
- **No value calculation** — PSA-1 does not compute any value, weight, minted amount, or reward. It outputs verifiable facts for PSA-2 / PSA-3 to reference.
- **`evidence_bundle_hash`** — `H(canonical serialization of EvidenceBundle, excluding `device_sig` and `group_signatures`)`. All signatures **MUST** sign this hash directly. Signing a field subset is strictly prohibited.

---

## A. Minimal Data Structures

---

### A1. ProjectLiteSpec — Minimal Project Specification

| Field | Notes |
|---|---|
| `project_id` | Unique identifier (human-readable or hash) |
| `project_tag_id` *(required)* | Unique identifier for the project's physical tag; used for public identification and project attribution in the physical world. The physical tag is **not** required to appear in evidence photos. |
| `project_notice_commit_hash` *(optional, strongly recommended)* | `H(project_id, version, location_scope, gateway_registry_ref, issuer_region_node_ref, issuer_signature, ts, …)` — locks down the official announcement. The commitment hash can be recomputed for audit purposes. |
| `name` | Project name |
| `version` | Version number — every change requires a new version |
| `owner` | Initiating entity signature / multi-sig rule reference |
| `status` | `Draft` / `Active` / `Frozen` / `Closed` — PSA-1 reads `status` for verification only; transitioning to `Frozen` or `Closed` is the responsibility of `owner` or PSA-3 |
| `time_window` | Start / end (open-ended is permitted) |
| `location_scope` | Geographic scope (GPS polygon / raster grid / parcel code set) |

#### mobility_class *(required)*

One of `{FIXED_SITE, MOBILE_OFFLINE, ONLINE}`:

- **`FIXED_SITE`** — work occurs within gateway coverage. Leaving triggers a release / abandon judgment.
- **`MOBILE_OFFLINE`** — task acquisition must occur within gateway coverage; the worker may leave during execution without triggering a release.
- **`ONLINE`** — no physical location dependency. Proof schema and presence semantics are defined by `proof_policy_ref`. Physical site / geographic sampling requirements do not apply.

#### Policy References

| Field | Required when | Purpose |
|---|---|---|
| `mobile_binding_policy_ref` | `MOBILE_OFFLINE` | Binding / unbinding / transfer / field-status policy; state machine fields, signature semantics, atomic transitions, and risk codes are hardcoded and versioned here |
| `mobile_binding_data_system_ref` | `MOBILE_OFFLINE` | Project-specific binding data system; records and auditably recomputes `target` / `target_slot` strong-binding relationships (bind / unbind / transfer) as a prerequisite for service verification |
| `rest_time_policy_ref` | Always | Rest-time rules: 2 h/day quota, quota refresh on unavailability, prohibition during sampling response windows, FIXED_SITE rest-leave release exemption, etc. |
| `gateway_registry_ref` | Always | Official gateway registry: `gateway_id → pubkey → zone_id_set → issuer_region_node_id → status (Active/Revoked)`. Gateways must be registered and Active; revocation or key rotation requires a new version |
| `issuer_region_node_ref` | Always | Regional node registry: `region_node_id → pubkey_set → zone_id_set → auth_policy`. Used for gateway attribution and authorization audit |

#### gateway_map_policy_ref *(required)*

Defines the ruleset for mapping `gateway observation → location_key → TargetRef`:

- **Mode A** — observation values are normalised to `grid_id`, then mapped to a valid `anchor_id` (e.g. `street_segment_id`) via the official mapping table.
- **Mode B** — the gateway emits a signed `location_attestation`; PSA-1 only verifies the signature and expiry. Can be replayed to Mode A for audit purposes.
- Mapping tables may be generated per project scope and versioned monthly / quarterly. PSA-1 references the version hash only; underlying map data disclosure is not required.
- Conflict resolution for a single `grid_id` mapping to multiple `anchor_id`s must be hardcoded here.

**SPACE → ENTITY Dynamic Upgrade & Canonical Rules** *(must be written into this reference and versioned)*

1. **Canonical Anchor Rule** — within the same `project_id + version`, for the same `location_key`, the system must select exactly one `canonical_target_ref` as the unique key for `anti_replay` / `ConfirmService`. The selection rule must be recomputable; free-text is not permitted. `ServiceTag` may reference `canonical_target_ref` as a normalisation field but **must not** act as the `anti_replay_key` or confirmation-lock primary key.

2. **Default priority** *(overridable, but must be explicit)* — if both a strong-semantic anchor (`ENTITY / RECORD / SYSTEM / PERSON`, `status=Active`) and a `SPACE` anchor are resolved simultaneously, `canonical_target_ref` must take the strong-semantic anchor; `SPACE` is used only as an equivalence mapping and audit replay entry.

3. **SPACE equivalence mapping** — must provide `eq_map(grid_id) → {anchor_id_list, canonical_anchor_id, mapping_proof_ref}`. Once a strong-semantic anchor takes effect, old `SPACE` submissions that resolve to `canonical_anchor_id` are treated as equivalent. `anti_replay` and confirmation locks must use `canonical_target_ref` as the sole primary key.

4. **Conflict handling** — if `eq_map` returns multiple candidates and canonical cannot be determined, `risk_flag` must be raised and a `conflict_reason_code` output.

5. **Scope constraint** — canonical rules affect only the uniqueness key / anti-replay / confirmation lock. `EvidenceBundle` may still carry both `grid_id` and a strong-semantic `anchor_id` for audit replay. Revocation of a strong-semantic anchor falls back to the next priority level; this must be recomputable and auditable.

#### anchor_registry_ref *(required)*

`anchor_id → zone_id / region_node → geometry / boundary digest → allowed_target_slots / schema_ref → status (Active / Revoked)`

#### multi_target_mode *(required)*

`SINGLE` or `MULTI` — affects only `TargetRef` binding; does not automatically change anti-replay, settlement, or audit rules.

- **`SINGLE`** — single object; submissions must match the unique `target_ref`.
- **`MULTI`** — multiple objects; submissions must hit one entry in `target_refs[]`.

`target_refs[]` elements must be unique (dedup key: `anchor_type + anchor_id + target_slot`). `target_refs[]` must not appear in `SINGLE` mode.

---

#### TargetRef Structure

| Field | Notes |
|---|---|
| `anchor_type` | `SPACE` / `ENTITY` / `PERSON` / `RECORD` / `SYSTEM`. `SPACE` is the fallback anchor, expressed via recomputable `grid_id`; it may be upgraded via `gateway_map_policy_ref` but must not be deleted. |
| `anchor_id` | Namespaced and hardcoded (e.g. `grid_id:…` / `asset_id:…` / `person_id:…` / `record_id:…` / `system_id:…`) |
| `anchor_scope` *(optional)* | Scope in which the anchor resides (e.g. `hospital_id`, `substation_id`) |
| `target_slot` *(required)* | Sub-slot of the object; must be a struct (not free text), expressing "which state / component / need / record key of the object is being acted upon" |

**target_slot fields:**

| Field | Notes |
|---|---|
| `slot_ns` *(required)* | `part_key` / `state_key` / `need_key` / `record_key` / `care_slot` |
| `slot_id` *(required)* | Slot identifier — format / namespace constrained by `target_slot_schema_ref` |
| `params` *(optional)* | Key-value pairs — allowed keys and types constrained by `target_slot_param_schema_ref` |

> **Prohibition** — `target_slot` must not carry procedural action narratives. "Steps taken" do not enter PSA-1; "outcome achieved" is proven by `acceptance_rule`.

**TargetSlot three-layer validity check** (priority: `anchor_registry_ref` > `target_slot_registry_ref` > `target_slot_schema_ref`):

- **`target_slot_schema_ref`** *(required)* — format and namespace validation for `slot_ns` / `slot_id`; action semantics prohibited.
- **`target_slot_param_schema_ref`** *(required)* — allowed key set, value types, required/optional, and length limits for `params`.
- **`target_slot_registry_ref`** *(required)* — slot validity registry: `slot_key → status → governance_proof_ref → scope_binding → updated_at`.
  - `slot_key = H(slot_ns, slot_id, param_schema_id)`. If `target_slot_param_schema_ref` exists, `param_schema_id` must be its reference ID — it must not be omitted.
  - `status ∈ {PROPOSED, PUBLISHED, ACTIVE, REVOKED}`. `governance_proof_ref` must pass review / public notice / endorsement and be verifiable. PSA-1 only verifies status and signature; it does not parse the argument content.
  - `scope_binding` must support at least one of: `region_node_id` / `zone_id_set` / `project_tag_id` / `anchor_type` / `anchor_namespace`.
  - If multiple `param_schema` versions are permitted in parallel, an explicit `canonical_param_schema_id` selection rule must be declared; otherwise the registry is invalid.

**TargetRef Examples:**

| Industry | anchor_type | anchor_id | target_slot |
|---|---|---|---|
| Healthcare | `PERSON` | `person_id:PID-xxxx` | `{care_slot, visit_episode, {episode_id:EPI-xxxx}}` |
| Energy | `ENTITY` | `device_id:ASSET-xxxx` | `{state_key, power_outage_alarm, {code:ALM-xxxx}}` |
| Food | `RECORD` | `record_id:BATCH-xxxx` | `{record_key, qc_test_case, {case_id:QC-xxxx}}` |
| Transport | `ENTITY` | `vehicle_id:VEH-xxxx` | `{part_key, subsystem, {name:brake_module}}` |
| Ecology | `SPACE` | `grid_id:H3-xxxx` | `{need_key, river_cleanup, {need_level:L2}}` |

---

#### Additional Required Fields

| Field | Notes |
|---|---|
| `participant_mode` *(required)* | `INDIVIDUAL` — each submission corresponds to a single `root_id`. `GROUP` — submission corresponds to a `participant_group_id` and requires all-member signatures (n-of-n). |
| `participant_group_policy_ref` | Required when `GROUP`. Must define `member_set_ref` (member ID list + version; changes = new version, no retroactive effect). Any member may submit on behalf of the group without lowering the signature requirement. Member governance details belong to PSA-3. |
| `service_templates[]` *(required)* | Whitelist of `service_type_id` values; each must exist in the `ServiceUnitType` set |
| `acceptance_policy_ref` *(required)* | Acceptance policy reference. Mode selection must follow a risk-tiered policy (matching cost and accountability to risk level). |
| `proof_policy_ref` *(required)* | Evidence policy reference (evidence types, required fields, hash structure, timestamp requirements) |
| `anti_replay_policy_ref` *(required)* | Anti-replay policy reference. The unique-key field set must use canonicalization (default: JCS + SHA-256); rules are hardcoded and versioned. |
| `dispute_policy_ref` *(required)* | Dispute entry policy reference; adjudication details belong to PSA-3 |
| `identity_binding_policy_ref` *(required)* | Identity / device binding policy (`ROOT_ID ↔ device_pubkey_set`; challenge-response; registration / revocation via external registry) |
| `presence_policy_ref` *(required)* | Presence policy (in-range challenge-response; session invalidation on departure; `session_ttl`; renewal / replay window) |
| `work_lease_policy_ref` *(required)* | Work lease policy (`lease_ttl`; Pause / Resume / Transfer rules; cross-project freeze semantics) |
| `presence_attendance_policy_ref` *(optional)* | Presence-proof implementation notes. Default: composed from `presence_policy_ref` + `work_lease_policy_ref`. If replaced by an external implementation, equivalent verifiable credentials must be output. |
| `external_refs` *(optional)* | PSA-2 / PSA-3 reference placeholders (e.g. `oracle_set_hash`, `reward_model_id`) — referenced only; content not validated |

---

### A1.1 Project Mobility Branching Rules

Branching switch: `mobility_class`.

**A) `FIXED_SITE`**

Leaving the gateway range invalidates the `PresenceSession`. If the lease is not Paused or Transferred, it is released with `reason=LEFT_ZONE`.

**B) `MOBILE_OFFLINE`**

- **B1. Pre-acquisition prerequisite** — `PresenceSession`, `WorkLease`, and `TaskToken` must be established within gateway range. Leaving after acquisition does not trigger a release; the worker is marked `in-field`.
- **B2. Strong binding** — after acquisition, the worker must establish a strong binding to `target_ref / target_slot` (`enhanced_bind=TRUE`). The binding relationship is managed by `mobile_binding_data_system_ref`, which issues a verifiable receipt.
- **B3. Abandonment requires returning to gateway** — the worker must return to gateway range and complete `unbind` before abandonment is permitted. The `mobile_binding_data_system_ref` receipt is the authoritative record.
- **B4. Transfer may occur outside the gateway** — transfer is only permitted to another worker with a valid in-progress lease on the same project (eligibility hardcoded in `mobile_binding_policy_ref`). The strong-binding relationship must migrate atomically to prevent double-binding.
- **B5. Verification prerequisite** — at submission time, `target_ref / target_slot` must be recomputably consistent with the current binding system record; inconsistency results in rejection or `risk_flag`.

**C) `ONLINE`**

No physical gateway required. Presence / lease may use network-gateway equivalent proofs. Physical site / geographic sampling requirements do not apply (unless the project explicitly declares otherwise).

---

### A1.2 TargetRef (Service Binding Object)

The field structure and validity rules for `TargetRef` / `TargetSlot` are governed solely by **A1.target_ref** as the canonical source.

PSA-1-lite binds service only at the *object layer* — value, weighting, and token minting are out of scope.

---

### A2. ServiceUnitType — Service Unit Type Definition

| Field | Notes |
|---|---|
| `service_type_id` | Unique identifier |
| `unit_spec` | Minimum verifiable specification (e.g. tree species, pit radius, soil compaction standard; survival verification window and other extensions belong to PSA-2 / PSA-3) |
| `completion_mode` *(required)* | `RESULT_ACHIEVEMENT` or `FIXED_SERVICE_RESULT` (see below) |

**completion_mode values:**

- **`RESULT_ACHIEVEMENT`** — centers on achieving a single outcome. Process proof uses process-round sampling; no sampling-point selection occurs.
- **`FIXED_SERVICE_RESULT`** — centers on continuous service through a specified duration with the outcome achieved. Process proof uses randomly sampled points; a completion video must be submitted at the end.

> **Constraint** — `FIXED_SERVICE_RESULT + RANDOM_SAMPLED_POINTS`: `total_work_duration` must be recomputable from `duration_profile_ref` or `proof_policy_ref`; if not, this sampling type must not be enabled.
>
> **Online constraint** — heartbeat is based on work-progress updates (a minimum 1-byte change counts). Sampling photos and completion videos are not required. No random-sampling bonus applies.

**Optional profile / parameter references** *(all optional; if used, must be versioned alongside `service_type`; changes require `version+1`; forward-only)*

`work_profile_ref` / `intensity_profile_ref` / `difficulty_profile_ref` / `risk_profile_ref` / `skill_profile_ref` / `non_interrupt_profile_ref` / `duration_profile_ref` / `factor_composition_policy_ref`

**Required fields for executors:** `geo / grid_id`, `time_completed`, `unit_serial`, `object_tag_id`, `evidence_hashes`, `device_sig / credential`. Uniqueness of `unit_serial` is governed by `anti_replay_policy_ref`.

**Acceptance:** `AUTO` / `HUMAN` / `HYBRID`. May include `AUTO_SCAN` (drone / camera / sensor scan as one evidence input; not mandatory).

**acceptance_rule:** acceptance rules (roles, deadlines, thresholds, re-submission-on-failure rules).

**proof_schema:** evidence format (photo / video / location / sensor / third-party report) + hash-manifest structure.

---

#### ProofSchemaType: RANDOM_SAMPLED_POINTS

Applicable to physical on-site projects only. Not applicable to purely network / digital projects.

**Purpose** — without requiring a live stream, form a verifiable proof loop via: unreplaceable randomly assigned points + unified-perspective photo receipt + optional auto-judgment.

Points are written into `WorkLease` at task acquisition and cannot be replaced after issuance. Workers cannot specify their own points.

**Core constraints (hardcoded):**

- **Perspective** — unified `view_id = FRONT` (straight-ahead, single perspective). One photo per point only; multi-angle or multi-photo composites are not permitted.
- **Ultra-wide lens** — equivalent focal length 13–15 mm, or EXIF FOV ≥ 110°. Applies to both `FIXED_SITE` and `MOBILE_OFFLINE`.
- **FIXED_SITE stance (hardcoded)** — dispersed layout → shoot northward from the south side of the point; center-surrounded layout → stand at the point and shoot toward the center. Topology type is hardcoded in `WorkLease` or `proof_policy_ref` and versioned.
- **Points** — count is configurable (default = 4); assigned by the system according to `TargetRef + TargetSlot` spatial semantics and written into `WorkLease.point_set[]`. Workers may not specify points.
- **Photo content** — must directly depict "the service worker actively executing the work content of this acquired project."
- **Minimum capture boundary** — submit only the minimum frame required for verification. Additional environmental imagery must be explicitly declared in `proof_policy_ref` and is off by default.

**ClarityCheck** *(optional prompt; not a rejection condition)*

When the app / AI determines that a photo is too blurry, underexposed, or obstructed to reliably identify `object_tag_id`, the worker must be offered two options:
1. Upload anyway — mark `risk_flag`.
2. Retake — new `photo_hash / SampleCaptureReceipt`; `lease_id` is not changed.

Prompt metrics (hardcoded in `proof_policy_ref`): `sharpness_score` / `exposure_score`; trigger threshold: `clarity_warn_threshold`.

Optional audit trail: `clarity_hint_commit_hash = H(photo_hash, sharpness_score_bucket, exposure_score_bucket, user_choice, ts, nonce)`.

---

### A2.1 Sampling Trigger — SamplingTrigger

**Purpose** — prevent "slot-holding without working / process fraud" by issuing a sample-response challenge during `WorkLease=HELD`, according to `completion_mode` / `mobility_class`.

**Routing:**

| mobility_class | completion_mode | Execution rule |
|---|---|---|
| `FIXED_SITE` | `FIXED_SERVICE_RESULT` | This section (trigger at each quarter of total work time) |
| `FIXED_SITE` | `RESULT_ACHIEVEMENT` | A2.1.0 — Result-achievement process sampling |
| `MOBILE_OFFLINE` | Any | A2.1.1 — Mobile project sampling rules |
| `ONLINE` | Any | Defined separately by `proof_policy_ref`; physical point sampling does not apply |

**1) Trigger frequency**

The system must trigger once at each quarter of total work time, up to 4 times (`trigger_seq ∈ {1, 2, 3, 4}`):

```
trigger_issued_at = work_started_at + n × (total_work_duration / 4)
```

`total_work_duration` must be recomputable from `duration_profile_ref` or `proof_policy_ref`. Rest time does not trigger; rest duration is excluded from elapsed time.

**2) Deadline and release (even-seq failure releases; hardcoded)**

- `trigger_seq ∈ {1, 3}` — `deadline = issued_at + 12 min`; timeout records `INCOMPLETE` only; no release.
- `trigger_seq ∈ {2, 4}` — `deadline = issued_at + 12 min` (30 min if `trigger_seq=4` and seq 3 is `COMPLETE`); timeout triggers immediate `WorkLease` release.

**3) Early trigger** *(only when `completion_mode = RESULT_ACHIEVEMENT`; strictly prohibited for `FIXED_SERVICE_RESULT`)*

Workers MAY initiate an early trigger while `lease=HELD`. The system generates `trigger_id / seq / deadline` and signs it. Counts toward the 1–4 cap; subject to even-seq failure release. Points are still selected from `untriggered_point_set`; workers may not specify.

**4) SamplingTrigger minimum fields**

`trigger_id` / `lease_id` / `trigger_seq (1–4)` / `trigger_issued_at` / `trigger_deadline_at` / `selected_point_id` / `trigger_sig` (gateway or region_node signature)

**5) Point selection rule (hardcoded)**

Three state sets: `untriggered_point_set` / `triggered_but_incomplete_set` / `completed_point_set`.

Each trigger selects only from `untriggered_point_set`, choosing the point nearest to the worker's currently recomputable location. Triggering stops when `untriggered_point_set` is empty.

**6) Retakes / resubmissions**

Retakes are permitted for any point without changing `lease_id`, without generating a new trigger, and without extending the deadline. Once a `point_id` reaches `PointComplete`, it is permanently moved to `completed_point_set`. Valid only while `lease=HELD` and not yet `FINALIZED / RELEASED`.

**7) SampleRound definition**

- `round_key = H(lease_id, trigger_seq, selected_point_id)`
- `SampleRoundStatus = COMPLETE` if and only if: exactly 1 valid receipt exists for that `round_key` satisfying `view_id=FRONT` + valid signature + `proof_policy_ref` hard rules.
- `round_completed_at` — timestamp of that valid receipt.
- **Atomicity** — retakes after exit / interruption require a new `capture_session_id`; cross-session splicing is not accepted.

**8) Start-of-clock**

`trigger_issued_at` is generated and signed by the issuer (gateway / region_node) and is the sole valid start point. Whether the client receives it does not affect the clock. Anomalies are handled exclusively through the PSA-3 dispute entry.

**9) PSA-1 verification**

- `trigger_sig` must pass verification against `gateway_registry_ref` or `issuer_region_node_ref`.
- `selected_point_id ∈ WorkLease.point_set[]` and must belong to `untriggered_point_set`.
- `trigger_seq` must be monotonically increasing and ≤ 4; rollback or repeat results in rejection or `risk_flag`.
- `trigger_deadline_at` must be consistent with the trigger sequence number and previous trigger completion status; it must be recomputable.

---

### A2.1.0 Result-Achievement Process Sampling (`FIXED_SITE + RESULT_ACHIEVEMENT`)

Applicable only when `mobility_class = FIXED_SITE` and `completion_mode = RESULT_ACHIEVEMENT`. No point selection occurs; sampling always revolves around the same single result object / result state.

**1) Triggering and sampling**

- Up to 4 rounds (`process_trigger_seq ∈ {1, 2, 3, 4}`; monotonically increasing); 1 photo per round.
- Default interval: every 45 minutes. Workers MAY trigger early. Rest time excluded. If a round completes early, the next round's clock restarts from `round_completed_at`.
- Deadline, even-round timeout release, 12 m / 30 m windows, and freshness rules are identical to A2.1. Substitute `process_round_id` for `point_id`.

**2) Photo requirements**

1 photo per round; must directly depict "the service worker actively advancing the single outcome corresponding to this acquired task." Signature, timestamp, device binding, and no cross-session splicing are the same as `SampleCaptureReceipt`.

**3) Receipt mapping**

Uses the A2.1.2 `SampleCaptureReceipt` structure. `capture_ref = process_round_id (∈ {1, 2, 3, 4})`; all other fields and signature semantics unchanged.

**4) Completion determination**

- `process_key = H(lease_id, process_round_id)` — once complete, permanently `COMPLETE` (unless PSA-3 adjudication revokes it).
- Completion condition: 1 valid `SampleCaptureReceipt` satisfying: `lease_id` matches; `process_round_id ∈ {1, 2, 3, 4}`; `view_id=FRONT`; `device_sig` verifiable; `proof_policy_ref` hard rules met.
- `round_completed_at` — timestamp of that valid receipt.
- Skipped rounds / rollback / duplicate submission: rejected or `risk_flag`.

**5) Sampling compliance status output**

- `captured_count = |completed_round_set|`; `points_count = 4`; `missing_count = points_count - captured_count`
- `captured_count == 4` → `sampling_compliance_status = COMPLETE`; otherwise `INCOMPLETE`
- PSA-1 must output: `captured_count` / `missing_count` / `completed_round_set_digest` / `missing_round_set_digest` / `sampling_commit_hash` / `sampling_compliance_status`
- `sampling_commit_hash = H(lease_id, sort_by(process_round_id)(photo_hash list), completed_round_set_digest, missing_round_set_digest, nonce)`

**6) Completion video** *(if required by the project)*

After the result is achieved, a completion video of ≥ 480p must be submitted. PSA-1 verifies signature, timestamp, result binding, and consistency with `evidence_bundle_hash`.

---

### A2.1.1 Mobile Project Sampling Trigger Rules (`MOBILE_OFFLINE`)

**Purpose** — permit field workers to leave gateway range while maintaining a verifiable loop of in-progress presence and outcome; trigger lock-freeze and rescue coordination on abnormal communication loss.

**1) Trigger frequency** — once every 60 minutes; `trigger_seq` monotonically increasing; no 4-trigger cap.

**2) Response window** — the worker must respond within 30 minutes of each trigger; otherwise `LOCK_FREEZE` is triggered and an alert is raised.

Valid response (either of the following):
- (a) Upload 1 compliant ultra-wide-angle sampling photo (equivalent 13–15 mm or FOV ≥ 110°).
- (b) *Independently verifiable outcome type* — upload a ≥ 480p completion video and successfully ingest it; this counts as a response, resets the response timer, and the next 60-minute cycle restarts from that timestamp.

**3) Sampling photo content**

- *Work-content type* — task-related scene + worker-associable elements + minimum elements associable with `target_ref / target_slot`.
- *Independently verifiable outcome type* — completed outcome + identifiable completion-location elements + target-association elements.

**4) LOCK_FREEZE** *(triggered after 30 minutes without a response)*

- `WorkLease.state → LOCK_FREEZE`; alert raised.
- An information package must be sent synchronously to police / emergency services, containing at minimum: `project_id`, `version`, `service_type_id`, `lease_id`, `root_id / participant_ref`, current binding record digest (desensitisation semantics hardcoded in `mobile_binding_policy_ref`), most recent sampling photo / video timestamp and GPS (if available), last valid response time, most recent location fix, gateway information, `contact_hint` (if provided by the project).
- Unfreezing requires a confirmed receipt from police / emergency services (receipt format and authority set defined by `dispute_policy_ref` or `proof_policy_ref`).

**5) Post-unfreeze handling**

- **Continue** — transfer to another eligible worker via `TRANSFER` (eligibility governed by `mobile_binding_policy_ref`).
- **End** — enter the PSA-3 dispute / audit entry.

**6) PSA-1 verification**

- Trigger / freeze / unfreeze receipt signatures must pass verification against `gateway_registry_ref` or `issuer_region_node_ref`.
- Submission / confirmation must satisfy the `mobile_binding_data_system_ref` binding-consistency prerequisite (see A1.1-B5).

---

### A2.1.2 Sampling Receipt — SampleCaptureReceipt

Every sampling photo must generate a `SampleCaptureReceipt` signed by the device.

**Minimum fields:**

| Field | Notes |
|---|---|
| `lease_id` *(required)* | Work lease this receipt is bound to |
| `capture_ref` *(required)* | `FIXED_SERVICE_RESULT` → `point_id` (must be `∈ WorkLease.point_set[]`); `RESULT_ACHIEVEMENT` → `process_round_id` |
| `view_id` *(required)* | Hardcoded `FRONT` |
| `photo_hash` *(required)* | Hash of photo content (canonicalization semantics fixed by `proof_policy_ref`) |
| `ts` *(required)* | Capture timestamp |
| `location_proof` *(required)* | Either GPS (`lat + lng + accuracy`) or gateway-mapped location (`gateway_id + zone_id + signed location assertion`). If gateway-mapped, must pass `gateway_registry_ref` signature verification and be within `ttl`. |
| `capture_session_id` *(required)* | Session ID for this capture session; session ends on exit / interruption; retake requires a new session |
| `device_pubkey` *(required)* | — |
| `device_sig` *(required)* | Signature over `(lease_id, capture_ref, view_id, photo_hash, ts, location_proof, capture_session_id, device_pubkey, and binding fields)` |
| `challenge_hash` *(optional)* | If the project enables a capture challenge code, it must be included in the signature |

**Atomicity rule** — cross-session splicing is not accepted. Under the same session, no other image files may be associated (unless `proof_policy_ref` explicitly enables this and it is versioned).

**PSA-1 verification:**
- `device_pubkey ∈ root_id`'s `device_pubkey_set` (`identity_binding_policy_ref`)
- `device_sig` verifiable
- `lease_id` consistent with `EvidenceBundle`
- `location_proof` present and valid
- `capture_ref`: in `FIXED_SERVICE_RESULT` mode, `point_id` must be `∈ WorkLease.point_set[]`; in `RESULT_ACHIEVEMENT` mode, must satisfy the recomputable binding rule for `process_round_id` defined in `proof_policy_ref`
- If `challenge_hash` is enabled: must pass verification under `presence_policy_ref` / `proof_policy_ref`

---

### A2.1.3 Random-Point Sampling EvidenceBundle Field Requirements (`FIXED_SERVICE_RESULT`)

`EvidenceBundle / ResultUnit` must contain: `sample_captures[]` *(optional but strongly recommended)* and `sampling_commit_hash` *(required)*.

**PointComplete definition** (`point_key = H(lease_id, point_id)`):

Across all historical submissions (cumulative across `trigger_seq`), at least 1 valid `SampleCaptureReceipt` must exist satisfying: `lease_id` matches; `capture_ref = point_id ∈ WorkLease.point_set[]`; `view_id=FRONT`; `device_sig` verifiable; `proof_policy_ref` hard rules met. Once achieved, permanently `COMPLETE` (unless PSA-3 adjudication revokes it).

**Sampling compliance status:**

- `captured_count = |completed_point_set ∩ WorkLease.point_set[]|`
- `points_count = |WorkLease.point_set[]|`
- `missing_count = points_count - captured_count`
- `captured_count == points_count` → `COMPLETE`; otherwise `INCOMPLETE`
- `INCOMPLETE` does not invalidate service authenticity; the missing-set digest must be output for the settlement layer to process.
- `sampling_commit_hash = H(lease_id, sort_by(point_id)(photo_hash list), completed_point_set_digest, missing_point_set_digest, nonce)`

---

### A2.1.4 Auto-Judgment Thresholds & Settlement Hooks

**Optional fields:**

- `defect_report_digest` — machine-judgment problem digest (hash commitment of: type / count / confidence / model version).
- `outcome_attestation` — signed assertion of the outcome from an official AI or authorized verifier; must be bound to and signed over `outcome_binding_hash`, which must contain at minimum: `project_id`, `version`, `service_type_id`, `root_id / participant_ref`, `target_ref`, `location_key` (if applicable), `presence_session_id + presence_commit_hash`, `lease_id + lease_event_head_hash`, `task_token_id` (if enabled), `evidence_bundle_hash`, `sampling_commit_hash`. Signature must pass verification against the `ACTIVE` authority set specified in `proof_policy_ref`.

**PSA-1 output facts** *(for PSA-2 / PSA-3 reference; PSA-1 does not compute reward values)*

`points_count` / `captured_count` / `missing_count` / `missing_point_set_digest` / `sampling_commit_hash` / `sampling_compliance_status`

**Settlement layer may implement:**

- **Evidence compliance discount** — `evidence_compliance_factor = f(captured_count / points_count)` (where `f` may be linear or step-wise).
- **Random-sampling bonus** (`FIXED_SERVICE_RESULT + RANDOM_SAMPLED_POINTS`; not applicable for `ONLINE`) — `sampling_bonus_factor = 0.05 × captured_count`, capped at `+0.20`.
- Each completed sampling requirement also counts as 1 valid task heartbeat.

---

### A2.1.5 Failure Handling

- Any signature verification failure or binding inconsistency must result in rejection or `risk_flag`.
- `sampling_compliance_status = INCOMPLETE` must not invalidate service authenticity; a completion-degree flag must be output for the settlement layer to handle.

---

### A3. EvidenceBundle — Evidence Package

**Single vs. multi-result mode:**

- If `result_units[]` is non-empty: all verifications are executed per `result_unit`; top-level fields serve as compatibility fields only.
- Otherwise: single-result semantics apply (top-level fields are authoritative).

**Each ResultUnit minimum fields:**

`service_type_id` / `target_ref` (with `target_slot`) / `unit_serial` / `time_completed` (inherits from top-level if absent) / `evidence_items[]` / *(when `RANDOM_SAMPLED_POINTS` or `RESULT_ACHIEVEMENT`)* `sampling_commit_hash` + `sample_captures[]` + `sampling_compliance_status`

**EvidenceBundle fields:**

| Field | Notes |
|---|---|
| `project_id` / `version` / `service_type_id` / `result_units[]` | — |
| `target_ref` *(required)* | Matched per `multi_target_mode` (`SINGLE` → matches `target_ref`; `MULTI` → hits one entry in `target_refs[]`) |
| `unit_serial` *(required)* | — |
| `root_id` *(required)* | Worker's root identity ID |
| `presence_session_id` *(required)* | Must reference a valid `PresenceSession` |
| `presence_commit_hash` *(required)* | Must equal the `presence_commit_hash` of the corresponding session |
| `lease_id` *(required)* | Must reference a valid `WorkLease` (`state=HELD` or a policy-permitted equivalent) |
| `lease_event_head_hash` *(required)* | Must equal `WorkLease.lease_event_head_hash` |
| `participant_ref` *(required)* | `{INDIVIDUAL, root_id}` or `{GROUP, participant_group_id, submitter_root_id, member_set_ref}` |
| `group_signatures` *(required for GROUP)* | All-member n-of-n signatures over the same `evidence_bundle_hash`; `group_sig_agg_hash = H(signature list sorted by member_id)` |
| `task_token_id` *(conditionally required)* | Required if the project enables two-phase tokens; must satisfy `token.lease_id == bundle.lease_id` and `token.root_id == bundle.root_id` |
| `location_attestation` *(optional, strongly recommended)* | `Sig_gateway(project_id, version, gateway_id, zone_id, root_id, ts, location_key, confidence, ttl, nonce)`; PSA-1 verifies signature source (`gateway_registry_ref`) and that it is within `ttl` |
| `location_key` *(conditionally required)* | `anchor_type=SPACE` → `grid_id`; otherwise recomputable from `location_attestation` or `gateway_map_policy_ref`. Must be consistent with `location_attestation` if present. Used for scope verification / audit replay / sampling index; does not participate in the `anti_replay` unique key by default (unless `anti_replay_policy_ref` explicitly declares otherwise). Required when `proof_policy_ref` or `service_type` declares `requires_location_key = TRUE`. |
| `geo` *(optional)* | — |
| `grid_id` | Required when `anchor_type=SPACE`; otherwise optional. If provided, must satisfy `gateway_map_policy_ref` recomputable consistency with `location_key`. |
| `object_tag_id` *(required)* | Physical tag ID |
| `time_completed` | Required when `result_units[]` is empty; optional (used as default for all `ResultUnit`s) when non-empty |
| `mobility_trace_commit_hash` *(required)* | `H(gateway_id, zone_id, root_id, time_window, path_digest, sample_count, confidence_stats, nonce, …)`; must be signed by `gateway_id` or `issuer_region_node_id` |
| `trace_data_ref` *(optional)* | Reference to encrypted trace package; decrypted only on dispute per `dispute_policy_ref` |
| `evidence_items[]` | `{type, hash, timestamp, source, optional_signature}` |
| `nonce` *(optional)* | — |
| `device_sig` *(required)* | Device signature over `evidence_bundle_hash`; `device_pubkey ∈ root_id`'s `device_pubkey_set` |

For `RESULT_ACHIEVEMENT` process sampling, the bundle must also carry: `sample_captures[]` / `sampling_commit_hash` / `sampling_compliance_status` (per A2.1.0 semantics — process rounds, not physical points).

---

### A3.1 PresenceSession — Presence Session

**Minimum fields:**

| Field | Notes |
|---|---|
| `project_id` / `version` | — |
| `zone_id` *(optional, strongly recommended)* | — |
| `gateway_id` *(required)* | Must be registered and Active in `gateway_registry_ref` |
| `issuer_region_node_id` *(required)* | — |
| `root_id` / `device_pubkey` *(required)* | — |
| `challenge_hash` *(required)* | One-time challenge code issued by the gateway to `root_id` |
| `gateway_challenge_sig` *(required)* | Gateway signature over `(project_id, version, zone_id, gateway_id, root_id, device_pubkey, challenge_hash, issued_at, expires_at)`. Proves the challenge originates from a registered, Active official gateway. |
| `response_sig` *(required)* | Device signature over `challenge_hash` |
| `issued_at` / `expires_at` *(required)* | — |

**Outputs:**

```
presence_session_id    = H(project_id, version, root_id, device_pubkey, gateway_id,
                           zone_id, challenge_hash, issued_at, expires_at)

presence_commit_hash   = H(presence_session_id, project_id, version, root_id,
                           device_pubkey, gateway_id, zone_id, challenge_hash,
                           gateway_challenge_sig, response_sig, issued_at, expires_at)
```

---

### A3.2 WorkLease — Work Lease

**Purpose** — occupy a specific slot instance; prevent concurrent contention and "claim slot first, submit evidence later."

**Minimum fields:**

| Field | Notes |
|---|---|
| `lease_id` / `project_id` / `version` / `service_type_id` | — |
| `lease_key` *(required)* | `H(project_id, version, service_type_id, anchor_type, anchor_id, target_slot, unit_serial)`. `unit_serial` uniquely identifies the slot instance; different slots map to different `lease_key`s and do not block each other. A `FINALIZED` slot is permanently closed; its `lease_key` may not be re-acquired. |
| `point_set[]` *(conditionally required)* | Required when `proof_schema.type = RANDOM_SAMPLED_POINTS`. Assigned by the system at task acquisition per `TargetRef + TargetSlot` spatial semantics; elements are `point_id`s. Cannot be replaced after issuance. Workers may not specify points. |
| `idle_ttl` | If no valid heartbeat (see A3.2.0) is received within this duration while in `HELD` state, the lease is automatically released (`reason=IDLE_TIMEOUT`). |
| `participant_ref` *(required)* | — |
| `bound_presence_session_id` *(required)* | — |
| `state` | `HELD` / `PAUSED` / `LOCK_FREEZE` / `TRANSFERRED` / `RELEASED` / `FINALIZED` |
| `lease_expires_at` *(required)* | — |
| `pause_expires_at` | Required when `PAUSED` |
| `transfer_record_hash` | Required when `TRANSFERRED` |
| `lease_event_head_hash` *(required)* | Latest head hash of the event commitment chain |
| `lease_event_seq` *(required)* | Monotonically increasing |

**LeaseEvent minimum fields:**

`lease_id` / `event_type (PAUSE|RESUME|LOCK_FREEZE|UNFREEZE|TRANSFER|RELEASE|FINALIZE)` / `from_participant_ref` / `to_participant_ref` (for `TRANSFER`) / `reason_code` (for `RELEASE / FINALIZE`) / `ts` / `prev_event_commit_hash` / `event_seq` / `event_sig(s)`

```
event_commit_hash = H(lease_id, event_type, from_participant_ref, to_participant_ref,
                      reason_code, ts, prev_event_commit_hash, event_seq,
                      H(event_sig list sorted by signer))

lease_event_head_hash = event_commit_hash
```

A broken chain or `event_seq` rollback is invalid.

**TRANSFER dual-signature rule** — `event_sig(s)` must include both `from_sig` ("I transfer to you") and `to_sig` ("I accept the handoff"). Signatures must cover at minimum: `lease_id`, `event_type`, `from/to_participant_ref`, `ts`, `prev_event_commit_hash`, `event_seq`. Either party absent → `TRANSFER` invalid.

**LOCK_FREEZE / UNFREEZE rules:**

- `LOCK_FREEZE` — `to_participant_ref` is empty. The `lease_key` must not be re-acquired during the freeze. Trigger conditions are hardcoded in `proof_policy_ref` / A2.1.1.
- `UNFREEZE` — must have a verifiable unfreeze basis (police / emergency services confirmed receipt). Target state after unfreeze (`HELD / RELEASED / TRANSFERRED`) is hardcoded in `proof_policy_ref` / `mobile_binding_policy_ref`.
- Both events' `event_sig(s)` must cover: `lease_id`, `event_type`, `from_participant_ref`, `ts`, `prev_event_commit_hash`, `event_seq`, and a freeze / unfreeze basis digest.

**Additional constraints for `RANDOM_SAMPLED_POINTS`:**

While in `HELD` state, `SamplingTrigger` time limits apply (per A2.1). When even-round timeout release conditions are met, output `LeaseReleased(reason=SAMPLE_TRIGGER_TIMEOUT)`, minimum fields:

`lease_id` / `lease_key` / `project_id` / `version` / `trigger_id` / `trigger_seq` / `selected_point_id` / `trigger_issued_at` / `trigger_deadline_at` / `release_ts` / `point_key = H(lease_id, selected_point_id)` / `completion_evidence_digest = H(lease_id, selected_point_id, latest_valid_photo_hash, receipt_ts_bucket, nonce)` / `release_attestation_sig` (must pass `gateway_registry_ref` or `issuer_region_node_ref` verification)

This release is independent of `idle_ttl`. After release, `lease_key` returns to acquirable status (or enters `RISK_HOLD`, per policy reference).

---

### A3.2.0 Task Heartbeat Rules

Valid heartbeats (at least one of the following; `idle_ttl` is measured against these):

| Type | Applicable scenario | Trigger condition |
|---|---|---|
| Spatial movement heartbeat | `FIXED_SITE` / physical gateway | Movement ≥ 5 m from the last verifiable location |
| Sampling completion heartbeat | `FIXED_SERVICE_RESULT` | Each randomly sampled point completed |
| Sampling completion heartbeat | `RESULT_ACHIEVEMENT` | Each process sampling round completed |
| Progress update heartbeat | `ONLINE` / non-physical site | Any recomputable progress update (minimum 1 byte) |

`work_lease_policy_ref` may additionally recognize `LeaseEvent`s and other signals, but must not conflict with the table above.

---

### A3.2.1 Task Acquisition Concurrency Limits

**Concurrency key** — `INDIVIDUAL` → `root_id`; `GROUP` → `participant_group_id`.

At any given time, the same concurrency key must not hold more than 1 `WorkLease` with `state ∈ {HELD, PAUSED}`.

*(This does not restrict the number of objects or scope of work allowed within a single task per `service_type / target_ref / result_units[]`.)*

---

### A3.2.2 Project Cooldown Freeze Rules

1. **Consecutive-abandonment cooldown** — if the same concurrency key abandons 3 consecutive tasks under the same `project_id` (acquired but ended before `FINALIZED`), the key is frozen until the next Monday 00:00 (project local timezone). Any successful `FINALIZED` resets the counter.
2. **PSA-3 remedy timeout cooldown** — if a PSA-3 remedy procedure has been ongoing for 72 hours without completion, the same freeze applies.
3. **Freeze effect** — no new tasks may be acquired under that `project_id`. The freeze expires automatically (or per policy) at the designated time; upon expiry the counter resets.
4. **Audit trail events** — `ParticipantProjectAcceptanceFrozen(project_id, participant_ref, reason_code, freeze_until_ts, ts)` / `ParticipantProjectAcceptanceUnfrozen(project_id, participant_ref, ts)`

---

### A3.2.3 RestTime — Rest Time

- **Quota** — 2 hours per day; refreshes at 00:00 the next day. Quota may not be refreshed by releasing and re-acquiring a lease.
- **Usage** — may be taken in segments; the worker decides freely.
- **Hard prohibition during sampling response windows:**
  - `FIXED_SITE` — prohibited from `trigger_issued_at` to `trigger_deadline_at`.
  - `MOBILE_OFFLINE` — prohibited from `trigger_issued_at` to `+30 min`.
  - Violation must be rejected or `risk_flag`.
- **FIXED_SITE rest-leave exemption** — leaving the site during rest does not trigger a release. However, if the worker is still off-site when rest ends, a `LEFT_ZONE` release must be triggered immediately.
- **`MOBILE_OFFLINE` / `ONLINE`** — only quota and response-window constraints apply. Online-equivalent implementation is defined by `rest_time_policy_ref` / `proof_policy_ref`.

---

### A3.3 TaskToken — Task Token

**Purpose** — two-phase design: no token is issued during the announcement / browsing phase; `TaskToken` is issued only after `WorkLease` is successfully established, entering the verifiable credential flow.

**Minimum fields:**

`token_id` / `project_id` / `version` / `root_id` / `lease_id` / `gateway_id` / `zone_id` *(strongly recommended)* / `issued_at` / `expires_at` / `token_sig` (gateway signature over the above fields)

**PSA-1 verification:** `token_sig` verifiable via `gateway_registry_ref` + not expired + `token.lease_id == bundle.lease_id` and `token.root_id == bundle.root_id`.

---

### A4. ServiceTag — Service Tag

**Purpose** — submission event identifier and audit-index commitment only. Must not be implemented as an `anti_replay_key` or confirmation-lock primary key.

**Single-result:**

```
service_tag = H(
  project_id, version, service_type_id,
  participant_ref.type,
  INDIVIDUAL → root_id       | GROUP → participant_group_id,
  INDIVIDUAL → device_sig    | GROUP → group_sig_agg_hash,
  target_ref / location_key / object_tag_id / unit_serial,
  time_completed,
  evidence_bundle_hash,
  nonce
)
```

**Multi-result** (`result_units[]` non-empty; a `service_tag_i` is generated per `result_unit`):

```
service_tag_i = H(
  project_id, version, result_unit.service_type_id,
  participant_ref.type,
  INDIVIDUAL → root_id       | GROUP → participant_group_id,
  INDIVIDUAL → device_sig    | GROUP → group_sig_agg_hash,
  result_unit.target_ref / location_key / object_tag_id / result_unit.unit_serial,
  result_unit.time_completed OR EvidenceBundle.time_completed,
  evidence_bundle_hash,
  unit_index_i,
  nonce
)
```

---

## B. Minimal Verification Flow

---

### Flow 1 — PublishProjectLite

**Input:** `ProjectLiteSpec` + owner signature

**Verification:**
- `project_id`, `version`, `owner`, `status`, `time_window`, `location_scope` are all present.
- `service_templates[]` is non-empty; each `service_type_id` corresponds to an existing `ServiceUnitType`.
- Each `ServiceUnitType` contains `acceptance_rule` + `proof_schema`; `required_fields` must include `unit_serial`.
- `acceptance_policy_ref`, `proof_policy_ref`, `anti_replay_policy_ref`, and `dispute_policy_ref` are all present.
- Serialization and hashing semantics are fixed (default: JCS + SHA-256) and must not change within a version.

**Output:** `ProjectLitePublished(project_id, version, hash(spec))`

---

### Flow 2 — ActivateProjectLite

**Input:** `project_id` + `version` + owner signature

**Verification:**
- Project status must be `Draft`. (Publish registers the project as `Draft`; Activate transitions it to `Active`.)
- `location_scope` is non-empty.
- `anti_replay_policy_ref` is bound.
- `dispute_policy_ref` is bound.

**Output:** `ProjectLiteActivated(project_id, version, hash(spec))`

---

### Flow 2.5 — AcquireWorkLease (Task Acquisition)

**Input:** `project_id` + `version` + `service_type_id` + `unit_serial` + `root_id` + `device_sig` + `presence_session_id`

**Prerequisites:**
- The worker has arrived at gateway range and a `PresenceSession` has been established and is valid (see A3.1).
- For `ONLINE` projects: equivalent presence proof via network gateway; physical attendance is not required.

**Verification:**
- Project must be `Active`.
- `presence_session_id` is valid; `session.root_id == root_id`; `session.device_pubkey ∈ root_id`'s `device_pubkey_set`.
- `service_type_id` is in `service_templates[]`.
- **Concurrency check** — the concurrency key does not currently hold any `HELD` or `PAUSED` `WorkLease` (see A3.2.1).
- **Freeze check** — the concurrency key is not within an active `ParticipantProjectAcceptanceFrozen` period for this `project_id` (see A3.2.2).
- `lease_key = H(project_id, version, service_type_id, anchor_type, anchor_id, target_slot, unit_serial)` — not currently occupied by any `HELD / PAUSED` `WorkLease`; and not in `FINALIZED` state.

**System actions:**
- Generate `WorkLease` (`state=HELD`); write `lease_id`, `lease_key`, `participant_ref`, `bound_presence_session_id`, `lease_expires_at`, `lease_event_head_hash (GENESIS)`.
- If `proof_schema.type = RANDOM_SAMPLED_POINTS`: assign points per `TargetRef + TargetSlot` spatial semantics; write to `WorkLease.point_set[]`.
- Issue `TaskToken` (if the project enables two-phase tokens), bound to `lease_id`.

**Output:** `WorkLeaseAcquired(project_id, version, lease_id, lease_key, root_id, participant_ref, point_set[] (if applicable), task_token_id (if applicable), ts)`

---

### Flow 3 — SubmitService

**Input:** `EvidenceBundle` + `device_sig` (INDIVIDUAL) or `group_signatures` (GROUP)

If `result_units[]` is non-empty, all verifications below are executed per `result_unit`; otherwise single-result semantics apply.

**Verification checklist:**

① Project must be `Active`.

② **PresenceSession** — session exists and is valid (not expired, not departed); `session.root_id == bundle.root_id`; `session.device_pubkey ∈ root_id`'s `device_pubkey_set`.

③ **Departure release branching:**
  - `FIXED_SITE` — session invalidated and lease not Paused / Transferred → release (`reason=LEFT_ZONE`); rest-leave exemption applies; if still off-site when rest ends, release immediately.
  - `MOBILE_OFFLINE` — departure does not trigger release; on-duty constraints are governed by A2.1.1 + `mobile_binding_policy_ref`; release is only permitted under explicit abandonment or valid transfer conditions.
  - `ONLINE` — defined by `proof_policy_ref` / `work_lease_policy_ref` online-equivalent implementation.

④ `service_type_id` must be in `service_templates[]`.

⑤ `proof_schema` verification passes (evidence types complete, hash fields complete, timestamp format, source fields present).

⑥ **RANDOM_SAMPLED_POINTS** *(if applicable)*:
  - `WorkLease.point_set[]` exists and is non-empty; `WorkLease` corresponding to `bundle.lease_id` is valid.
  - Each receipt in `sample_captures[]`: `device_sig` verifiable; `lease_id` matches `bundle.lease_id`; `capture_ref (point_id) ∈ WorkLease.point_set[]`.
  - `sampling_commit_hash` is recomputable and consistent.
  - If `requires_location_key`: `bundle.location_key` must exist and be valid.
  - **Submission freshness** — `submit_success_ts ≤ most_recent SampleRoundStatus=COMPLETE round_completed_at + 30 min`.

⑦ **TaskToken** *(if enabled)*: token exists and is not expired; `token_sig` verifiable; `token.lease_id == bundle.lease_id`; `token.root_id == bundle.root_id`.

⑧ **TargetSlot verification** — must pass all three layers of schema validation for `slot_ns / slot_id / params` (format, length, allowed key set, action semantics prohibited). Registry semantics take priority over `schema_ref`. If `target_slot` is required, absence results in rejection.

⑨-a **Service object matching:**
  - `SINGLE` — `bundle.target_ref` must match `ProjectLiteSpec.target_ref` (at minimum `anchor_type + anchor_id`; `target_slot` per project requirements).
  - `MULTI` — `bundle.target_ref` must hit one entry in `ProjectLiteSpec.target_refs[]`.

⑨-b **Geographic scope verification:**
  - `anchor_type=SPACE` — `grid_id` must fall within `location_scope`; if `geo` is submitted, map to `grid_id` first then verify.
  - `anchor_type≠SPACE` — if `geo / grid_id` is provided, it must fall within `location_scope` (optional verification).

⑨-c **Anchor registry verification:**
  - `anchor_id` must be `Active` in `anchor_registry_ref`.
  - If `zone_id / gateway_id` is provided: `anchor_id` attribution must be consistent with them (semantics defined by `anchor_registry_ref`).
  - `target_slot` must satisfy the `allowed_target_slots` specified by the registry.

⑩ **Anti-replay:**
  ```
  anti_replay_key           = H(project_id, version, service_type_id,
                                canonical_target_ref_hash, unit_serial)
  canonical_target_ref_hash = H(anchor_type, anchor_id, target_slot, … canonicalized)
  ```
  `location_key` does not participate by default (unless `anti_replay_policy_ref` explicitly declares it). When `result_units[]` is non-empty, compute separately per `result_unit`.

⑪ **Participant verification:**
  - `INDIVIDUAL` — `root_id` matches; `device_sig` verifiable by `device_pubkey_set`.
  - `GROUP` — `member_set_ref` resolvable; `group_signatures` covers all members (n-of-n); each signature is valid over the same `evidence_bundle_hash`. Otherwise, reject immediately.

⑫ **Device binding** — if `device_pubkey` is not in `device_pubkey_set`, prompt "device not bound / inconsistent" and raise `risk_flag`. Whether to reject is determined by `acceptance_policy_ref` or PSA-3.

⑬ **WorkLease:**
  - `lease_id` exists and `state=HELD` (or a policy-permitted equivalent).
  - `lease_key` locks the specific slot instance (including `unit_serial`); only one person may occupy the same `lease_key` at a time. `FINALIZED` slots may not be re-acquired; `RELEASED` slots may be re-acquired.
  - `WorkLease.bound_presence_session_id`'s `session.root_id == bundle.root_id`.
  - `bundle.lease_event_head_hash == WorkLease.lease_event_head_hash`.

⑭ **Concurrency / freeze** — concurrency key must not hold multiple `HELD / PAUSED` `WorkLease`s simultaneously; must not acquire new tasks during an active `ParticipantProjectAcceptanceFrozen` period.

**Output:** `ServiceSubmitted(project_id, version, service_tag, evidence_bundle_hash, participant_ref, root_id, lease_id, time_completed)` → enters the pending-acceptance queue.

---

### Flow 4 — ConfirmService

**Input:** `service_tag` + verifier signature (or automated verification output)

**Verification:**
- `acceptance_rule` is satisfied; acceptance mode complies with the risk-tiered policy of `acceptance_policy_ref`.
- Verifier role / permissions comply with `acceptance_policy_ref`.
- **Confirmation lock** (anti-replay re-verification) — the same `(project_id, version, service_type_id, canonical_target_ref_hash, unit_serial)` must not be confirmed more than once. `location_key` / `object_tag_id` do not participate in the confirmation lock by default.
- `AUTO / HYBRID` may accept `scan_proof_hash` as a verification input. Scan failure or contradiction triggers `ServiceDisputed` (adjudication belongs to PSA-3).

**Outputs:**

**`ServiceConfirmed`:**
- `WorkLease.state → FINALIZED`; the slot is permanently closed and accepts no further acquisitions or submissions. Other slots under the same task are unaffected.
- The service record (`service_tag` / `evidence_bundle_hash`) enters the PSA-2 settlement queue. After PSA-2 completes, the record enters PSA-3 audit together with PSA-1 output. After PSA-3 passes, it is sealed.
- **Demand saturation flag output** *(for PSA-2 reward calculation; PSA-1 does not block new acquisitions)*:
  - `fulfilled_count` — total `FINALIZED` slots under this `target_ref + target_slot`
  - `demand_unit` — original demand quantity, defined by `unit_spec` or `proof_policy_ref`
  - `saturation_flag` — `TRUE` when `fulfilled_count ≥ demand_unit`; when `TRUE`, PSA-2 may calculate reward approaching zero

**`ServiceRejected` / `ServiceDisputed` / timeout:**
- `WorkLease.state → RELEASED`; slot returns to acquirable status (or enters `RISK_HOLD`, per `dispute_policy_ref` / `work_lease_policy_ref`).

```
ServiceConfirmed(project_id, version, service_tag, verifier_id, result=ACCEPT)
ServiceRejected(…, reason_code)
ServiceDisputed(…, dispute_id)   ← entry point only; adjudication in PSA-3
```

---

*Author: Zhengyang Pan (潘正阳) | BBBigradish (一根大萝卜)*
