# PSA-1-lite — Minimum Data Structures / Minimum Validation Flows 🪶

PSA-1-lite does one thing: close the engineering loop on **verifiable service outcomes**.
No value calculation, no weighting, no minting, no periodic accounting, no audit accountability — none of that is computed here. This layer only holds reference interfaces for those concerns.

---

## Global Conventions

- All `*_ref` fields: PSA-1 only validates that the reference exists and that its signature/version is verifiable. It does not parse content. All changes must be versioned; changes take effect forward-only.
- The reject/risk_flag behavior on validation failure is defined by the corresponding policy reference. If not declared, the default is rejection.
- PSA-1 does not compute any value, weight, mint, or reward figures. It outputs verifiable facts only, for consumption by PSA-2 and PSA-3.
- `evidence_bundle_hash = H(EvidenceBundle excluding device_sig and group_signatures fields, canonicalized)`. All signatures MUST sign this hash directly. Signing a subset of fields is strictly prohibited.

---

## A. Minimum Data Structures

### A1. ProjectLiteSpec

- **project_id**: Unique identifier (human-readable ID or hash)
- **project_tag_id** *(required)*: Unique identifier of the project's physical tag. Used for public recognition and project attribution in the physical world. The tag is NOT required to appear in evidence photos.
- **project_notice_commit_hash** *(optional but strongly recommended)*: `H(project_id, version, location_scope, gateway_registry_ref, issuer_region_node_ref, publisher_sig, ts, ...)`; locks the official announcement so it can be verified against at audit time.
- **name**: Project name
- **version**: Version number — any change requires a new version
- **owner**: Initiating party signature or multi-sig rule reference
- **status**: `Draft | Active | Frozen | Closed`; PSA-1 only reads status for validation. PSA-1 does NOT set status to Frozen or Closed — that authority belongs to the owner or PSA-3.
- **time_window**: Start / end (open-ended allowed)
- **location_scope**: Geographic boundary (GPS polygon / grid / parcel code set)

- **mobility_class** *(required)*: `∈ {FIXED_SITE, MOBILE_OFFLINE, ONLINE}`
  - `FIXED_SITE`: Work occurs within the gateway coverage area. Leaving the area triggers a release/abandon judgment.
  - `MOBILE_OFFLINE`: Task acquisition must occur within the gateway area, but execution may continue outside it. Leaving does not trigger a release.
  - `ONLINE`: No physical location dependency. The `proof_schema` and presence protocol are defined by `proof_policy_ref`. Physical point sampling and geographic coverage requirements do not apply.

- **mobile_binding_policy_ref** *(required when MOBILE_OFFLINE)*: Binding/unbinding/transfer/field-status policy reference. The binding state machine fields, signature protocol, atomic migration rules, and risk codes are hardcoded and versioned in this policy.
- **mobile_binding_data_system_ref** *(required when MOBILE_OFFLINE)*: Reference to the project's dedicated binding data system. Records and allows auditable re-computation of target/target_slot strong-binding relationships (bind/unbind/transfer). Binding consistency is a prerequisite for service verification.
- **rest_time_policy_ref** *(required)*: Rest time rules reference. Defines the 2h daily quota, prohibition on refreshing quota via task release/re-acquire, prohibition on rest during sampling response windows, and FIXED_SITE rest-period departure exemption.
- **gateway_registry_ref** *(required)*: Official gateway registry reference. Defines `gateway_id → pubkey → zone_id_set → issuer_region_node_id → status(Active/Revoked)`. Gateways must be registered and Active. Revocation or key rotation requires a new version.
- **issuer_region_node_ref** *(required)*: Regional node registry reference. Defines `region_node_id → pubkey_set → zone_id_set → auth_policy`. Used for gateway attribution and authorization auditing.

- **gateway_map_policy_ref** *(required)*: Mapping rule set reference for `gateway observation → location_key → TargetRef`.
  - *Scheme A*: Observation value normalized to `grid_id`, then mapped to a valid `anchor_id` (e.g. `street_segment_id`) via official mapping table.
  - *Scheme B*: Gateway directly outputs a signed `location_attestation`; PSA-1 only verifies the signature and expiry. Can be replayed to Scheme A for audit.
  - Mapping tables may be generated per project scope on demand and versioned monthly/quarterly. PSA-1 only references the version hash and does not require disclosure of underlying map data.
  - When a single `grid_id` maps to multiple `anchor_id` values, conflict resolution rules must be hardcoded here.

  **【SPACE→ENTITY Dynamic Promotion and Canonical Rules】** *(must be written into this reference and versioned)*
  1. **Canonical Anchor Rule**: For the same `project_id+version` and `location_key`, the system must select exactly one `canonical_target_ref` as the uniqueness key for `anti_replay` / `ConfirmService`. The selection rule must be deterministically computable and must not rely on free text. ServiceTag may reference `canonical_target_ref` for audit normalization, but must not itself serve as the `anti_replay_key` or confirmation lock key.
  2. **Priority** (default; overridable but must be explicit): If both a strong semantic anchor (ENTITY/RECORD/SYSTEM/PERSON, status=Active) and a SPACE anchor (grid_id) can be resolved from the same `location_key`, `canonical_target_ref` MUST use the strong semantic anchor. SPACE is only used as an equivalence mapping and audit replay entry point.
  3. **SPACE Equivalence Mapping**: Must provide `eq_map(grid_id) → {anchor_id_list, canonical_anchor_id, mapping_proof_ref}`. Once a strong semantic anchor is active, old SPACE submissions whose `grid_id` maps to `canonical_anchor_id` are treated as equivalent. `anti_replay` and confirmation locks must use `canonical_target_ref` as the sole uniqueness key.
  4. **Conflict Handling**: If `eq_map` returns multiple candidates and `canonical_anchor_id` cannot be determined, the submission MUST be `risk_flag`'d and emit a `conflict_reason_code`.
  5. **Scope Constraint**: Canonical rules only affect the uniqueness key / anti-replay / confirmation lock. The EvidenceBundle may still carry both `grid_id` and a stronger-semantic `anchor_id` for audit replay. If a strong semantic anchor is revoked, fallback to the next priority level must be deterministically computable and auditable.

- **anchor_registry_ref** *(required)*: Anchor object registry reference. Defines `anchor_id → zone_id/region_node → geometry/boundary digest → allowed_target_slots/schema_ref → status(Active/Revoked)`.

- **multi_target_mode** *(required)*: `∈ {SINGLE, MULTI}`. Only changes the TargetRef binding mode. Does not automatically change `anti_replay`, settlement, attribution, or audit rules.
  - `SINGLE`: One service object. Submissions must match the unique `target_ref`.
  - `MULTI`: Multiple service objects. Submissions must hit one entry in `target_refs[]` whitelist.

- **target_ref** *(required when SINGLE)* / **target_refs[]** *(required when MULTI)*: Service object definition.
  - `target_refs[]` elements must not be duplicated (dedup key: `anchor_type + anchor_id + target_slot`). When SINGLE, `target_refs[]` must not appear (or must be empty).

  **TargetRef structure:**
  - `anchor_type`: `∈ {SPACE, ENTITY, PERSON, RECORD, SYSTEM}`
    - `SPACE`: Fallback anchor. Uses a computable `grid_id` as the minimal verifiable reference when an object is not yet formally registered or the real-world state is temporary/emergent. Can be upgraded via `gateway_map_policy_ref` mapping, but must not be deleted.
  - `anchor_id`: Hardcoded to a namespace (e.g. `grid_id:...` / `asset_id:...` / `person_id:...` / `record_id:...` / `system_id:...`)
  - `anchor_scope` *(optional)*: Scope of the anchor (e.g. `hospital_id`, `substation_id`)
  - `target_slot` *(required)*: Object sub-slot. Must be a struct (not free text). Expresses which state/part/need/record-key of the object is being acted upon.
    - `slot_ns` *(required)*: `∈ {part_key, state_key, need_key, record_key, care_slot}`
    - `slot_id` *(required)*: Slot identifier (format/namespace constrained by `target_slot_schema_ref`)
    - `params` *(optional)*: Key-value parameters (allowed keys/types constrained by `target_slot_param_schema_ref`)
    - **Prohibition**: `target_slot` must not carry procedural action narratives. "Steps taken" do not enter PSA-1. "Result achieved" is proven via `acceptance_rule`.

  **TargetSlot three-layer validation** (priority: `anchor_registry_ref` > `target_slot_registry_ref` > `target_slot_schema_ref`):
  - `target_slot_schema_ref` *(required)*: Validates `slot_ns`/`slot_id` format and namespace; prohibits action-verb semantics.
  - `target_slot_param_schema_ref` *(required)*: Validates `params` — allowed key set, value types, required/optional, length limits.
  - `target_slot_registry_ref` *(required)*: Slot validity registry. Defines `slot_key → status → governance_proof_ref → scope_binding → updated_at`.
    - `slot_key = H(slot_ns, slot_id, param_schema_id)`. If `target_slot_param_schema_ref` exists, `param_schema_id` must use its reference ID — omission is not permitted.
    - `status ∈ {PROPOSED, PUBLISHED, ACTIVE, REVOKED}`. `governance_proof_ref` must pass review/public notice/endorsement and be verifiably signed. PSA-1 only validates status + signature; it does not parse the rationale.
    - `scope_binding` must support at least one of: `region_node_id / zone_id_set / project_tag_id / anchor_type / anchor_namespace`.
    - If multiple `param_schema` versions are allowed in parallel, the rule for selecting `canonical_param_schema_id` must be explicitly declared; otherwise the registry is considered invalid.

  **TargetRef Examples:**

  | Industry | anchor_type | anchor_id | target_slot |
  |----------|-------------|-----------|-------------|
  | Healthcare | PERSON | person_id:PID-xxxx | {care_slot, visit_episode, {episode_id:EPI-xxxx}} |
  | Energy | ENTITY | device_id:ASSET-xxxx | {state_key, power_outage_alarm, {code:ALM-xxxx}} |
  | Food | RECORD | record_id:BATCH-xxxx | {record_key, qc_test_case, {case_id:QC-xxxx}} |
  | Transport | ENTITY | vehicle_id:VEH-xxxx | {part_key, subsystem, {name:brake_module}} |
  | Ecology | SPACE | grid_id:H3-xxxx | {need_key, river_cleanup, {need_level:L2}} |

- **participant_mode** *(required)*: `∈ {INDIVIDUAL, GROUP}`
  - `INDIVIDUAL`: Each submission corresponds to a single `root_id`.
  - `GROUP`: Each submission corresponds to a `participant_group_id`. All members must sign (n-of-n) for the submission to enter the valid confirmation flow.
- **participant_group_policy_ref** *(required when GROUP)*: Must define `member_set_ref` (member ID list + version; changes require a new version; no retroactive recomputation). Any member may submit on behalf of the group, but this does not reduce the signature requirement. Member governance details belong to PSA-3.

- **service_templates[]** *(required)*: Whitelist of `service_type_id` values. Each entry must have a corresponding `ServiceUnitType` definition.
- **acceptance_policy_ref** *(required)*: Acceptance policy reference. Acceptance mode selection must follow a risk-tiered policy (matching cost and accountability to project risk level).
- **proof_policy_ref** *(required)*: Evidence policy reference (evidence types, required fields, hash structure, timestamp requirements).
- **anti_replay_policy_ref** *(required)*: Anti-replay policy reference. The uniqueness key field set must be canonicalized before hashing (default: JCS + SHA-256). Normalization rules are hardcoded and versioned.
- **dispute_policy_ref** *(required)*: Dispute entry policy reference. Adjudication details belong to PSA-3.
- **identity_binding_policy_ref** *(required)*: Identity/device binding policy (`ROOT_ID ↔ device_pubkey_set`; challenge-response protocol; device registration/revocation provided by external registry).
- **presence_policy_ref** *(required)*: Presence policy (in-area challenge-response; session invalidated on departure; `session_ttl`; renewal/replay windows).
- **work_lease_policy_ref** *(required)*: Work lease policy (`lease_ttl`; Pause/Resume/Transfer rules; cross-project freeze protocol).
- **presence_attendance_policy_ref** *(optional)*: Presence proof implementation note. Default: `presence_policy_ref + work_lease_policy_ref` forming challenge-response + lease state machine. If replaced with an external implementation, it must output an equivalent verifiable credential.
- **external_refs** *(optional)*: PSA-2/3 reference placeholders (e.g. `oracle_set_hash`, `reward_model_id`). Referenced only; content not validated.

---

### A1.1 Mobility Mode Rules

Branching switch: `mobility_class`

**A) FIXED_SITE**
Leaving the gateway area invalidates the PresenceSession. If the worker has not Paused or Transferred, the WorkLease is released with `reason=LEFT_ZONE`.

**B) MOBILE_OFFLINE**
- **B1. Acquisition prerequisite**: PresenceSession / WorkLease / TaskToken must all be established within the gateway area. Leaving after acquisition does not trigger a release; the system marks the worker as `in-field`.
- **B2. Strong binding**: After acquisition, the task must enter strong-binding mode (`enhanced_bind=TRUE`), binding the worker to `target_ref/target_slot`. The binding relationship is managed by `mobile_binding_data_system_ref`, which issues a verifiable receipt.
- **B3. Abandonment requires return and unbind**: To abandon a task, the worker must return to the gateway area and complete an `unbind`. The `mobile_binding_data_system_ref` receipt is the authoritative confirmation.
- **B4. Transfer may occur outside the gateway area**: Transfer is only allowed to another worker who is currently active on the same project (eligibility defined by `mobile_binding_policy_ref`). The strong-binding relationship must be atomically migrated to the recipient to prevent double-binding.
- **B5. Verification precondition**: At submission time, `target_ref/target_slot` in the EvidenceBundle must be deterministically consistent with the current binding record in the binding system. Inconsistency results in rejection or `risk_flag`.

**C) ONLINE**
No physical gateway required. Presence/lease may use network-gateway-equivalent proof. Physical point sampling and geographic coverage requirements do not apply unless the project explicitly declares otherwise.

---

### A1.2 TargetRef (Service Binding Object)

The field structure and validity rules for TargetRef/TargetSlot have `A1.target_ref` as their sole authoritative source. PSA-1-lite binds services at the object layer only. Value, weight, and minting are not discussed here.

---

### A2. ServiceUnitType

- **service_type_id**
- **unit_spec**: Minimum verifiable specification (e.g. tree species, pit radius, soil compaction standard; survival window and similar extensions belong to PSA-2/3)
- **completion_mode** *(required)*: `∈ {RESULT_ACHIEVEMENT, FIXED_SERVICE_RESULT}`
  - `RESULT_ACHIEVEMENT`: Centers on achieving a single result/state. Process proof uses process sampling; no point extraction.
  - `FIXED_SERVICE_RESULT`: Centers on delivering continuous service for a defined duration with a guaranteed outcome. Process proof uses random point sampling; a completion video is required at the end.
  - *Constraint*: When `FIXED_SERVICE_RESULT + RANDOM_SAMPLED_POINTS`, `total_work_duration` must be deterministically computable from `duration_profile_ref` or `proof_policy_ref`. If not computable, this sampling type must not be used.
  - *Constraint (ONLINE)*: Heartbeat is based on work progress updates (minimum 1-byte change counts as valid). No sampling photos, completion photos, or completion videos required. No random sampling bonus applies.

- **Profile/parameter references** *(all optional)*: If used, must be committed with the `service_type` version. Changes require `version+1`, forward-only.
  `work_profile_ref / intensity_profile_ref / difficulty_profile_ref / risk_profile_ref / skill_profile_ref / non_interrupt_profile_ref / duration_profile_ref / factor_composition_policy_ref`

- **required_fields**: Executor required fields (`geo/grid_id`, `time_completed`, `unit_serial`, `object_tag_id`, `evidence_hashes`, `device_sig/credential`). `unit_serial` uniqueness is constrained by `anti_replay_policy_ref`.
- **acceptance_mode**: `AUTO / HUMAN / HYBRID`. May include `AUTO_SCAN` (drone/camera/sensor scan as one input source; not mandatory).
- **acceptance_rule**: Acceptance rules (role, deadline, threshold, re-submission rules on failure).
- **proof_schema**: Evidence format (photos/video/location/sensor/third-party reports) + hash manifest structure.

---

#### ProofSchemaType: RANDOM_SAMPLED_POINTS

Applies to physical on-site projects. Does not apply to pure online/electronic projects.

**Purpose**: Without requiring live streaming or constant supervision, form a verifiable result-proof loop via "non-replaceable random point assignment + uniform-angle photo receipts + optional automated judgment". Points are written into the WorkLease at acquisition time and cannot be replaced after issuance.

**Core constraints (hardcoded):**
- **View angle**: Fixed at `view_id = FRONT` (straight ahead, single angle). Only 1 photo per point. Multi-angle or composite replacements are not permitted.
- **Ultra-wide angle**: Equivalent focal length 13–15mm or EXIF FOV ≥ 110°. Applies to both FIXED_SITE and MOBILE_OFFLINE sampling photos.
- **FIXED_SITE positioning (hardcoded)**: Dispersed layout → worker stands south of point, shoots north. Hub-centered layout → worker stands at point, shoots toward the center. Topology type is hardcoded in WorkLease or `proof_policy_ref` and versioned.
- **Point count**: Configurable (default=4). Points are assigned by the system based on TargetRef+TargetSlot spatial semantics and written into `WorkLease.point_set[]`. Workers may not specify points.
- **Photo content**: Must directly demonstrate "the worker is executing the work they accepted under this project".
- **Minimum capture boundary**: Submit only the minimum image content required for validation. Additional environmental imagery requires explicit declaration in `proof_policy_ref` and is off by default.

**ClarityCheck** *(optional prompt; not a rejection condition)*:
When APP/AI determines the photo is too blurry/dark/obscured to reliably identify `object_tag_id`, the worker must be offered two options:
1. Upload anyway — marks submission as `risk_flag`.
2. Retake — generates new `photo_hash`/`SampleCaptureReceipt`; `lease_id` is not replaced.

Prompt metrics (hardcoded in `proof_policy_ref`): `sharpness_score` / `exposure_score`; threshold: `clarity_warn_threshold`.
Optional log: `clarity_hint_commit_hash = H(photo_hash, sharpness_score_bucket, exposure_score_bucket, user_choice, ts, nonce)`.

---

### A2.1 Random Sampling Trigger (SamplingTrigger)

**Purpose**: Prevent "slot-squatting without working / process fabrication". During `WorkLease=HELD`, issue sampling response challenges according to `completion_mode` / `mobility_class`.

**Branch routing:**

| mobility_class | completion_mode | Rule applied |
|---|---|---|
| FIXED_SITE | FIXED_SERVICE_RESULT | This section — random sampling (triggered at 1/4 work-time intervals) |
| FIXED_SITE | RESULT_ACHIEVEMENT | A2.1.0 — Result-achievement process sampling |
| MOBILE_OFFLINE | Any | A2.1.1 — Mobile project sampling rules |
| ONLINE | Any | Defined by `proof_policy_ref`; physical point sampling does not apply |

**1) Trigger frequency**
System triggers once at each 1/4 interval of total work time, max 4 times (`trigger_seq ∈ {1,2,3,4}`):
`trigger_issued_at = work_started_at + n × (total_work_duration / 4)`
`total_work_duration` must be deterministically computable from `duration_profile_ref` or `proof_policy_ref`.
Rest time does not trigger and is not counted toward elapsed time.

**2) Deadline and release (Even-fail release, hardcoded)**
- `trigger_seq ∈ {1,3}`: `deadline = issued_at + 12min`. Timeout records INCOMPLETE; no release.
- `trigger_seq ∈ {2,4}`: `deadline = issued_at + 12min` (30min if `trigger_seq=4` and seq=3 was COMPLETE). Timeout triggers immediate WorkLease release.

**3) Early trigger** *(only allowed when `completion_mode = RESULT_ACHIEVEMENT`; strictly prohibited for `FIXED_SERVICE_RESULT`)*
Worker MAY request early trigger while `lease=HELD`. System generates `trigger_id/seq/deadline` and signs it. Still counts toward the 1..4 cap; still subject to even-fail release; point still selected from `untriggered_point_set`; worker may not specify the point.

**4) SamplingTrigger minimum fields**
`trigger_id / lease_id / trigger_seq (1..4) / trigger_issued_at / trigger_deadline_at / selected_point_id / trigger_sig` (gateway or region_node signature)

**5) Point selection rule (hardcoded)**
Three categories: `untriggered_point_set / triggered_but_incomplete_set / completed_point_set`.
Each trigger selects only from `untriggered_point_set`, choosing the point nearest to the worker's current computable position. If `untriggered_point_set` is empty, triggering stops.

**6) Retake / resubmit**
Any point may be retaken without replacing `lease_id` or generating a new trigger or extending a deadline. Once a `point_id` reaches PointComplete, it is permanently added to `completed_point_set`. Valid only while `lease=HELD` and before FINALIZED/RELEASED.

**7) SampleRound definition**
- `round_key = H(lease_id, trigger_seq, selected_point_id)`
- `SampleRoundStatus = COMPLETE` if and only if: a valid receipt exists under this `round_key` satisfying `view_id=FRONT` + verifiable signature + `proof_policy_ref` hard rules.
- `round_completed_at`: the `ts` of that valid receipt.
- **Atomicity**: A retake after exit/interruption requires a new `capture_session_id`. Cross-session splicing is not accepted.

**8) Start time**
`trigger_issued_at` is generated and signed by the issuing party (gateway/region_node) and is the sole valid start point for the 12/30-minute window. Whether the client receives the notification does not affect the start. Signal/device/system anomalies are handled only through the PSA-3 dispute entry.

**9) PSA-1 validation**
- `trigger_sig` must verify against `gateway_registry_ref` or `issuer_region_node_ref`.
- `selected_point_id ∈ WorkLease.point_set[]` and belongs to `untriggered_point_set`.
- `trigger_seq` monotonically increasing and ≤ 4. Rollback/duplicate: reject or `risk_flag`.
- `trigger_deadline_at` must be consistent with and deterministically computable from `trigger_seq` and the completion status of the previous trigger.

---

### A2.1.0 Result-Achievement Process Sampling (FIXED_SITE + RESULT_ACHIEVEMENT)

**Applies**: Only when `mobility_class = FIXED_SITE` and `completion_mode = RESULT_ACHIEVEMENT`. No point extraction. All rounds sample the same result object/state.

**1) Trigger and sampling**
- Max 4 rounds (`process_trigger_seq ∈ {1,2,3,4}`, monotonically increasing), 1 photo per round.
- Default: triggered every 45 minutes. Worker MAY trigger early. Rest time not counted. If a round completes early, the next 45-minute window starts from `round_completed_at`.
- Deadline, even-fail release, 12m/30m windows, and freshness rules are identical to A2.1. Replace `point_id` with `process_round_id`.

**2) Photo requirements**
1 photo per round. Must directly demonstrate "the worker is advancing the single result they accepted". Signature, timestamp, device binding, and no-cross-session rules follow SampleCaptureReceipt.

**3) Receipt mapping**
Reuses A2.1.2 SampleCaptureReceipt structure. `capture_ref = process_round_id (∈ {1,2,3,4})`. All other fields and signature protocols unchanged.

**4) Completion judgment**
- `process_key = H(lease_id, process_round_id)`. Once complete, permanently COMPLETE (unless PSA-3 revokes).
- Completion condition: a valid SampleCaptureReceipt exists with `lease_id` matching, `process_round_id ∈ {1,2,3,4}`, `view_id=FRONT`, verifiable `device_sig`, and satisfying `proof_policy_ref` hard rules.
- `round_completed_at`: the `ts` of that receipt.
- Skip/rollback/duplicate submission: reject or `risk_flag`.

**5) Sampling compliance output**
- `captured_count = |completed_round_set|`; `points_count = 4`; `missing_count = points_count - captured_count`.
- `captured_count == 4` → `sampling_compliance_status = COMPLETE`; otherwise `INCOMPLETE`.
- PSA-1 must output: `captured_count / missing_count / completed_round_set_digest / missing_round_set_digest / sampling_commit_hash / sampling_compliance_status`.
- `sampling_commit_hash = H(lease_id, sort_by(process_round_id)(photo_hash list), completed_round_set_digest, missing_round_set_digest, nonce)`

**6) Completion video** *(if required by project)*
Must submit a ≥ 480p completion video after result is achieved. PSA-1 validates signature, timestamp, result binding, and consistency with `evidence_bundle_hash`.

---

### A2.1.1 Mobile Project Sampling Rules (MOBILE_OFFLINE)

**Purpose**: Allow field work outside the gateway area while maintaining a verifiable in-progress + result-verifiable loop. Trigger lock-freeze and emergency rescue coordination on abnormal loss of contact.

**1) Trigger frequency**: Once every 60 minutes. `trigger_seq` increments monotonically; no 4-trigger cap.

**2) Response deadline**: Worker must respond within 30 minutes of each trigger. Failure triggers `LOCK_FREEZE` and an alert.

Valid response (either one):
- a) Upload 1 compliant ultra-wide-angle sampling photo (equivalent 13–15mm or FOV ≥ 110°).
- b) *Deliverable-independent tasks*: Upload a ≥ 480p completion video and confirm successful ingest. This counts as a response AND resets the response timer. The next 60-minute cycle starts from that moment.

**3) Sampling photo content**
- *Work-in-progress type*: Task-relevant scene + worker-identifiable elements + minimum elements linkable to `target_ref/target_slot`.
- *Deliverable-independent type*: Completed deliverable + location-identifiable elements + target-linked elements.

**4) LOCK_FREEZE (no response within 30 minutes)**
- `WorkLease.state → LOCK_FREEZE`; alert triggered.
- System must synchronize an information package to law enforcement and hospital emergency services (or equivalent emergency coordination units). Package must include at minimum: `project_id, version, service_type_id, lease_id, root_id/participant_ref`, current binding record digest (desensitized per `mobile_binding_policy_ref`), recent sampling photo/video timestamps and GPS (if available), last valid response time, most recent location fix, gateway info, `contact_hint` (if provided by project).
- Unfreezing requires a confirmed receipt from law enforcement/emergency services. Format and authority set defined by `dispute_policy_ref` or `proof_policy_ref`.

**5) Post-unfreeze handling**
- Continue: Transfer via WorkLease.TRANSFER to another worker (subject to `mobile_binding_policy_ref` eligibility rules).
- Terminate: Enter PSA-3 dispute/audit entry.

**6) PSA-1 validation**
- Trigger/freeze/unfreeze receipt signatures must verify against `gateway_registry_ref` or `issuer_region_node_ref`.
- Submission/confirmation must satisfy the binding consistency precondition from `mobile_binding_data_system_ref` (see A1.1-B5).

---

### A2.1.2 Sampling Receipt (SampleCaptureReceipt)

Each sampling photo must generate a `SampleCaptureReceipt` signed by the device.

**Minimum fields:**
- `lease_id` *(required)*: The work lease this receipt belongs to.
- `capture_ref` *(required)*: When `FIXED_SERVICE_RESULT` = `point_id` (must be ∈ `WorkLease.point_set[]`). When `RESULT_ACHIEVEMENT` = `process_round_id`.
- `view_id` *(required)*: Hardcoded `FRONT`.
- `photo_hash` *(required)*: Hash of photo content (canonicalization defined by `proof_policy_ref`).
- `ts` *(required)*: Capture timestamp.
- `location_proof` *(required)*: One of — GPS (`lat + lng + accuracy`) OR gateway-mapped location (`gateway_id + zone_id + signed location attestation`). If gateway-mapped, must verify against `gateway_registry_ref` and be within TTL.
- `capture_session_id` *(required)*: Session ID for this capture attempt. Ends on exit or interruption; retake requires a new session.
- `device_pubkey` *(required)*
- `device_sig` *(required)*: Signature over `(lease_id, capture_ref, view_id, photo_hash, ts, location_proof, capture_session_id, device_pubkey, and binding fields)`.
- `challenge_hash` *(optional)*: If the project enables capture challenge codes, must be included in the signature.

**Atomicity rules**: Cross-session splicing is not accepted. No other image files may be associated within the same session as implicit evidence input (unless `proof_policy_ref` explicitly enables and versions this).

**PSA-1 validation:**
- `device_pubkey ∈ root_id`'s `device_pubkey_set` (`identity_binding_policy_ref`).
- `device_sig` verifies.
- `lease_id` matches the EvidenceBundle.
- `location_proof` exists and is valid.
- `capture_ref`: When `FIXED_SERVICE_RESULT`, `point_id` must be ∈ `WorkLease.point_set[]`. When `RESULT_ACHIEVEMENT`, must satisfy the deterministically computable `process_round_id` binding rule in `proof_policy_ref`.
- If `challenge_hash` is present: must verify via `presence_policy_ref` / `proof_policy_ref`.

---

### A2.1.3 Random Point Sampling EvidenceBundle Requirements (FIXED_SERVICE_RESULT)

EvidenceBundle/ResultUnit must include: `sample_captures[]` *(optional but strongly recommended)* / `sampling_commit_hash` *(required)*.

**PointComplete definition** (`point_key = H(lease_id, point_id)`):
Across any historical submission (cumulative across `trigger_seq`), a valid SampleCaptureReceipt must exist satisfying: `lease_id` matches; `capture_ref = point_id ∈ WorkLease.point_set[]`; `view_id = FRONT`; `device_sig` verifiable; all `proof_policy_ref` hard rules satisfied. Once reached, permanently COMPLETE (unless revoked by PSA-3).

**Sampling compliance status:**
- `captured_count = |completed_point_set ∩ WorkLease.point_set[]|`; `points_count = |WorkLease.point_set[]|`; `missing_count = points_count - captured_count`.
- `captured_count == points_count` → `COMPLETE`; otherwise `INCOMPLETE`.
- `INCOMPLETE` does not invalidate service authenticity; the deficit summary must be output for the settlement layer.
- `sampling_commit_hash = H(lease_id, sort_by(point_id)(photo_hash list), completed_point_set_digest, missing_point_set_digest, nonce)`

---

### A2.1.4 Automated Judgment Threshold and Settlement Hooks

**Optional fields:**
- `defect_report_digest`: Machine-determined defect summary (type/count/confidence/model version as a hash commitment).
- `outcome_attestation`: Signed assertion from an official AI or authorized verifier. Must bind and sign `outcome_binding_hash` containing at minimum: `project_id, version, service_type_id, root_id/participant_ref, target_ref, location_key (if present), presence_session_id+presence_commit_hash, lease_id+lease_event_head_hash, task_token_id (if enabled), evidence_bundle_hash, sampling_commit_hash`. Signature must verify against the ACTIVE authority set specified in `proof_policy_ref`.

**PSA-1 output facts** *(for PSA-2/PSA-3; PSA-1 does NOT compute reward values)*:
`points_count / captured_count / missing_count / missing_point_set_digest / sampling_commit_hash / sampling_compliance_status`

**Settlement layer may implement:**
- Evidence compliance discount: `evidence_compliance_factor = f(captured_count / points_count)` (f may be linear or stepped).
- Random sampling bonus (`FIXED_SERVICE_RESULT + RANDOM_SAMPLED_POINTS`; ONLINE excluded): `sampling_bonus_factor = 0.05 × captured_count`, capped at +0.20.
- Each completed sampling requirement also counts as 1 valid task heartbeat.

---

### A2.1.5 Failure Handling

- Any signature verification failure or binding inconsistency: reject or `risk_flag`.
- `sampling_compliance_status = INCOMPLETE`: Must not invalidate service authenticity. Must output completion degree marker for settlement layer processing.

---

### A3. EvidenceBundle

**Single/multi-result mode:**
- `result_units[]` non-empty: All validations execute per `result_unit`. Top-level fields serve as compatibility fields only.
- Otherwise: Single-result mode; top-level fields are authoritative.

**Each ResultUnit minimum fields:**
`service_type_id / target_ref (including target_slot) / unit_serial / time_completed (inherits from top level if absent) / evidence_items[] /` (when `RANDOM_SAMPLED_POINTS` or `RESULT_ACHIEVEMENT`) `sampling_commit_hash + sample_captures[] + sampling_compliance_status`

**EvidenceBundle fields:**
- `project_id / version / service_type_id / result_units[]`
- `target_ref` *(required)*: Matched per `multi_target_mode` (SINGLE→match `target_ref`; MULTI→hit `target_refs[]`).
- `unit_serial` *(required)*
- `root_id` *(required)*: Worker root identity ID.
- `presence_session_id` *(required)*: Must reference a valid PresenceSession.
- `presence_commit_hash` *(required)*: Must equal the corresponding session's `presence_commit_hash`.
- `lease_id` *(required)*: Must reference a valid WorkLease (`state=HELD` or policy-equivalent).
- `lease_event_head_hash` *(required)*: Must equal `WorkLease.lease_event_head_hash`.
- `participant_ref` *(required)*: `{INDIVIDUAL, root_id}` or `{GROUP, participant_group_id, submitter_root_id, member_set_ref}`.
- `group_signatures` *(required when GROUP)*: n-of-n all-member signatures over the same `evidence_bundle_hash`. `group_sig_agg_hash = H(signature list sorted by member_id)`.
- `task_token_id` *(conditionally required when project enables two-phase token)*: Must satisfy `token.lease_id == bundle.lease_id` and `token.root_id == bundle.root_id`.
- `location_attestation` *(optional but strongly recommended)*: `Sig_gateway(project_id, version, gateway_id, zone_id, root_id, ts, location_key, confidence, ttl, nonce)`. PSA-1 validates that signature is from `gateway_registry_ref` and is within TTL.
- `location_key` *(conditionally required)*: When `anchor_type=SPACE` = `grid_id`; otherwise derivable from `location_attestation` or `gateway_map_policy_ref`. If `location_attestation` is present, must be consistent with it. Used only for scope validation / audit replay / sampling index. Does not participate in `anti_replay` uniqueness key by default (unless `anti_replay_policy_ref` explicitly declares it). Required trigger: `proof_policy_ref` or `service_type` declares `requires_location_key = TRUE`.
- `geo` *(optional)*
- `grid_id` *(required when `anchor_type=SPACE`; optional otherwise)*: If provided, must satisfy the deterministically computable consistency relationship with `location_key` per `gateway_map_policy_ref`.
- `object_tag_id` *(required)*: Physical tag ID.
- `time_completed` *(required when `result_units[]` is empty; optional otherwise — if present, serves as the default value for all ResultUnits)*.
- `mobility_trace_commit_hash` *(required)*: `H(gateway_id, zone_id, root_id, time_window, path_digest, sample_count, confidence_stats, nonce, ...)`. Must be signed by `gateway_id` or `issuer_region_node_id`.
- `trace_data_ref` *(optional)*: Reference to the encrypted trajectory package on the server. Decrypted only during dispute per `dispute_policy_ref`.
- `evidence_items[]`: `{type, hash, timestamp, source, optional_signature}`
- `nonce` *(optional)*
- `device_sig` *(required)*: Device signature over `evidence_bundle_hash`; `device_pubkey ∈ root_id`'s `device_pubkey_set`.

**RESULT_ACHIEVEMENT process sampling** must also carry: `sample_captures[] / sampling_commit_hash / sampling_compliance_status` (per A2.1.0 protocol; field names retain `sampling_*` prefix but semantically represent process rounds, not physical points).

---

### A3.1 PresenceSession

**Minimum fields:**
- `project_id / version`
- `zone_id` *(optional but strongly recommended)*
- `gateway_id` *(required)*: Must be registered in `gateway_registry_ref` and Active.
- `issuer_region_node_id` *(required)*
- `root_id / device_pubkey` *(required)*
- `challenge_hash` *(required)*: One-time challenge issued by the gateway to `root_id`.
- `gateway_challenge_sig` *(required)*: Gateway signature over `(project_id, version, zone_id, gateway_id, root_id, device_pubkey, challenge_hash, issued_at, expires_at)`. Proves the challenge came from a registered, Active official gateway.
- `response_sig` *(required)*: Device signature over `challenge_hash`.
- `issued_at / expires_at` *(required)*

**Outputs:**
- `presence_session_id = H(project_id, version, root_id, device_pubkey, gateway_id, zone_id, challenge_hash, issued_at, expires_at)`
- `presence_commit_hash = H(presence_session_id, project_id, version, root_id, device_pubkey, gateway_id, zone_id, challenge_hash, gateway_challenge_sig, response_sig, issued_at, expires_at)`

---

### A3.2 WorkLease

**Purpose**: Occupy a specific slot instance, preventing concurrent preemption and "slot-squatting before submitting evidence".

**Minimum fields:**
- `lease_id / project_id / version / service_type_id`
- `lease_key` *(required)*: `H(project_id, version, service_type_id, anchor_type, anchor_id, target_slot, unit_serial)`. `unit_serial` uniquely identifies the slot instance. Different slots have different `lease_key` values and do not block each other. Once FINALIZED, the slot is permanently closed; that `lease_key` can no longer be acquired.
- `point_set[]` *(required when `proof_schema.type = RANDOM_SAMPLED_POINTS`)*: Written by the system at acquisition time based on TargetRef+TargetSlot spatial semantics. Elements are `point_id` values. Cannot be replaced after issuance. Workers may not specify points.
- `idle_ttl`: WorkLease is auto-released (`reason=IDLE_TIMEOUT`) if no valid heartbeat (see A3.2.0) occurs within this duration while in the HELD state.
- `participant_ref` *(required)*
- `bound_presence_session_id` *(required)*
- `state`: `∈ {HELD, PAUSED, LOCK_FREEZE, TRANSFERRED, RELEASED, FINALIZED}`
- `lease_expires_at` *(required)*
- `pause_expires_at` *(required when PAUSED)*
- `transfer_record_hash` *(required when TRANSFERRED)*
- `lease_event_head_hash` *(required)*: Most recent head hash of the event commitment chain.
- `lease_event_seq` *(required)*: Monotonically increasing.

**LeaseEvent minimum fields:**
`lease_id / event_type (PAUSE|RESUME|LOCK_FREEZE|UNFREEZE|TRANSFER|RELEASE|FINALIZE) / from_participant_ref / to_participant_ref (TRANSFER only) / reason_code (RELEASE/FINALIZE only) / ts / prev_event_commit_hash / event_seq / event_sig(s)`

`event_commit_hash = H(lease_id, event_type, from_participant_ref, to_participant_ref, reason_code, ts, prev_event_commit_hash, event_seq, H(event_sig list sorted by signer))`
`lease_event_head_hash = event_commit_hash`. Missing event, broken chain, or `event_seq` rollback is treated as invalid.

**TRANSFER dual-signature rule**: `event_sig(s)` must contain both `from_sig` ("I transfer to you") and `to_sig` ("I accept the handoff"). Signatures must cover at minimum: `lease_id, event_type, from/to_participant_ref, ts, prev_event_commit_hash, event_seq`. Missing either party's signature invalidates the TRANSFER.

**LOCK_FREEZE / UNFREEZE rules:**
- `LOCK_FREEZE`: `to_participant_ref` must be empty. `lease_key` may not be re-acquired during the freeze. Trigger conditions hardcoded in `proof_policy_ref` / A2.1.1.
- `UNFREEZE`: Requires verifiable unfreeze evidence (law enforcement/emergency services confirmation receipt). Rollback target (`HELD/RELEASED/TRANSFERRED`) hardcoded in `proof_policy_ref` / `mobile_binding_policy_ref`.
- Both events' `event_sig(s)` must cover: `lease_id, event_type, from_participant_ref, ts, prev_event_commit_hash, event_seq, freeze/unfreeze evidence digest`.

**RANDOM_SAMPLED_POINTS additional constraint:**
While HELD, WorkLease is also subject to the SamplingTrigger time constraints (A2.1 is the authoritative source). When even-fail release conditions are met, emit `LeaseReleased(reason=SAMPLE_TRIGGER_TIMEOUT)` with minimum fields:
`lease_id / lease_key / project_id / version / trigger_id / trigger_seq / selected_point_id / trigger_issued_at / trigger_deadline_at / release_ts / point_key = H(lease_id, selected_point_id) / completion_evidence_digest = H(lease_id, selected_point_id, latest_valid_photo_hash, receipt_ts_bucket, nonce) / release_attestation_sig` (must verify against `gateway_registry_ref` or `issuer_region_node_ref`).
This release does not depend on `idle_ttl`. After release, `lease_key` returns to acquirable state (or enters `RISK_HOLD`, per policy references).

---

### A3.2.0 Task Heartbeat Rules

**Valid heartbeat minimum set** (at least one required; `idle_ttl` is enforced against this set):

| Type | Applicable scenario | Trigger condition |
|---|---|---|
| Spatial movement heartbeat | FIXED_SITE / physical gateway | Moves ≥ 5m from last verifiable position |
| Sampling completion heartbeat | FIXED_SERVICE_RESULT | Each random sampling point completed |
| Sampling completion heartbeat | RESULT_ACHIEVEMENT | Each process sampling round completed |
| Progress update heartbeat | ONLINE / non-physical work | Any computable progress update (minimum 1 byte) |

`work_lease_policy_ref` may additionally recognize `LeaseEvent` and other signals, but must not conflict with the above.

---

### A3.2.1 Task Acquisition Concurrency Limit

Concurrency key: `INDIVIDUAL → root_id`; `GROUP → participant_group_id`.
At any given time, the same concurrency key must not hold more than 1 WorkLease with `state ∈ {HELD, PAUSED}`.
(This does not limit the number of service objects, results, or work volume within a single task.)

---

### A3.2.2 Project Cooldown Freeze Rules

1. **Consecutive abandonment cooldown**: If the same concurrency key has 3 consecutive "acquired but ended before FINALIZED" events under the same `project_id` → freeze until next Monday 00:00 (project local time). Any successful FINALIZED resets the counter.
2. **PSA-3 rescue timeout cooldown**: PSA-3 rescue procedure reaches 72-hour limit without completion → same freeze.
3. **Freeze effect**: May not acquire any new task under that `project_id`. Auto-unfreezes on expiry or by policy-specified authority. Counter resets to zero after unfreeze.
4. **Audit trail**: `ParticipantProjectAcceptanceFrozen(project_id, participant_ref, reason_code, freeze_until_ts, ts)` / `ParticipantProjectAcceptanceUnfrozen(project_id, participant_ref, ts)`

---

### A3.2.3 RestTime

- **Quota**: 2h per day, resets at 00:00 next day. Must not be refreshed via task release/re-acquire.
- **Usage model**: Segmented use and pausing allowed; worker decides freely.
- **No rest during sampling response windows (hard rule)**:
  - FIXED_SITE: within `trigger_issued_at` to `trigger_deadline_at`.
  - MOBILE_OFFLINE: within `trigger_issued_at` to `trigger_issued_at + 30min`.
  - Violation: reject or `risk_flag`.
- **FIXED_SITE rest departure exemption**: Departure during rest does not trigger release. However, if the worker is still outside the gateway area when rest ends, `LEFT_ZONE` release triggers immediately.
- **MOBILE_OFFLINE/ONLINE**: Only quota and response window constraints apply. Online-equivalent implementation defined by `rest_time_policy_ref` / `proof_policy_ref`.

---

### A3.3 TaskToken

**Purpose**: Two-phase design — no token issued during browsing/announcement phase. Token issued only after WorkLease is successfully established, entering the verifiable credential flow.

**Minimum fields**: `token_id / project_id / version / root_id / lease_id / gateway_id / zone_id (strongly recommended) / issued_at / expires_at / token_sig` (gateway signature over the above fields).

**PSA-1 validation**: `token_sig` verifies against `gateway_registry_ref` + not expired + `token.lease_id == bundle.lease_id` and `token.root_id == bundle.root_id`.

---

### A4. ServiceTag

Used only for submission event identification and audit index commitment. Must NOT be implemented as `anti_replay_key` or confirmation lock key.

**Single-result:**
`service_tag = H(project_id, version, service_type_id, participant_ref.type, (INDIVIDUAL?root_id:participant_group_id), (INDIVIDUAL?device_sig:group_sig_agg_hash), target_ref / location_key / object_tag_id / unit_serial, time_completed, evidence_bundle_hash, nonce)`

**Multi-result** (generates `service_tag_i` per result unit):
`service_tag_i = H(project_id, version, result_unit.service_type_id, participant_ref.type, (INDIVIDUAL?root_id:participant_group_id), (INDIVIDUAL?device_sig:group_sig_agg_hash), result_unit.target_ref / location_key / object_tag_id / result_unit.unit_serial, (result_unit.time_completed or EvidenceBundle.time_completed), evidence_bundle_hash, unit_index_i, nonce)`

---

## B. Minimum Validation Flows

### Flow 1: PublishProjectLite

**Input**: ProjectLiteSpec + owner signature

**Validation:**
- `project_id / version / owner / status / time_window / location_scope` all present.
- `service_templates[]` non-empty; each `service_type_id` has a corresponding `ServiceUnitType`.
- Each `ServiceUnitType` contains `acceptance_rule + proof_schema`; `required_fields` must include `unit_serial`.
- `acceptance_policy_ref / proof_policy_ref / anti_replay_policy_ref / dispute_policy_ref` all present.
- Serialization and hash algorithm fixed (default: JCS + SHA-256); must not change within the same version.

**Output**: `ProjectLitePublished(project_id, version, hash(spec))`

---

### Flow 2: ActivateProjectLite

**Input**: `project_id + version + owner signature`

**Validation:**
- Project status must be `Draft` (Flow 1 registers the project as Draft; Flow 2 transitions it to Active).
- `location_scope` non-empty.
- `anti_replay_policy_ref` bound.
- `dispute_policy_ref` bound.

**Output**: `ProjectLiteActivated(project_id, version, hash(spec))`

---

### Flow 2.5: AcquireWorkLease

**Input**: `project_id + version + service_type_id + unit_serial + root_id + device_sig + presence_session_id`

**Prerequisites:**
- Worker has arrived at the gateway area; PresenceSession is established and valid (see A3.1).
- ONLINE equivalent: network-gateway equivalent presence proof completed; physical presence not required.

**Validation:**
- Project must be Active.
- `presence_session_id` valid; `session.root_id == root_id`; `session.device_pubkey ∈ root_id`'s `device_pubkey_set`.
- `service_type_id` present in `service_templates[]`.
- **Concurrency check**: The concurrency key does not currently hold any HELD or PAUSED WorkLease (see A3.2.1).
- **Freeze check**: The concurrency key is not within an active `ParticipantProjectAcceptanceFrozen` period for this `project_id` (see A3.2.2).
- `lease_key = H(project_id, version, service_type_id, anchor_type, anchor_id, target_slot, unit_serial)`; this `lease_key` is not currently held by any HELD/PAUSED WorkLease and is not in FINALIZED state.

**System actions:**
- Create WorkLease (`state=HELD`); write `lease_id / lease_key / participant_ref / bound_presence_session_id / lease_expires_at / lease_event_head_hash (GENESIS)`.
- If `proof_schema.type = RANDOM_SAMPLED_POINTS`: assign points based on TargetRef+TargetSlot spatial semantics; write into `WorkLease.point_set[]`.
- Issue TaskToken (if the project enables the two-phase token), bound to `lease_id`.

**Output**: `WorkLeaseAcquired(project_id, version, lease_id, lease_key, root_id, participant_ref, point_set[] (if applicable), task_token_id (if applicable), ts)`

---

### Flow 3: SubmitService

**Input**: EvidenceBundle + `device_sig` (INDIVIDUAL) or `group_signatures` (GROUP)

If `result_units[]` is non-empty, all validations execute per result unit. Otherwise, single-result mode applies.

**Validation checklist:**

① Project must be Active.

② **PresenceSession**: Session exists and is valid (not expired, not departed); `session.root_id == bundle.root_id`; `session.device_pubkey ∈ root_id`'s `device_pubkey_set`.

③ **Departure/release routing:**
- FIXED_SITE: Session invalid and no Pause/Transfer → release (`reason=LEFT_ZONE`). Rest-period exemption applies; if still outside area when rest ends, release immediately.
- MOBILE_OFFLINE: Departure does not trigger release. In-progress constraint enforced by A2.1.1 + `mobile_binding_policy_ref`. Release only when explicit abandonment or lawful transfer conditions are met.
- ONLINE: Defined by `proof_policy_ref` / `work_lease_policy_ref` online-equivalent implementation.

④ `service_type_id` must be in `service_templates[]`.

⑤ `proof_schema` validation passes (evidence types complete, hash fields present, timestamp format, source fields).

⑥ **RANDOM_SAMPLED_POINTS validation** (if applicable):
- `WorkLease.point_set[]` exists and is non-empty; `bundle.lease_id`'s WorkLease is valid.
- Each receipt in `sample_captures[]`: `device_sig` verifies; `lease_id` matches `bundle.lease_id`; `capture_ref (point_id) ∈ WorkLease.point_set[]`.
- `sampling_commit_hash` can be deterministically recomputed.
- If `requires_location_key`: `bundle.location_key` must exist and be valid.
- **Submission freshness**: `submit_success_ts ≤ most recent SampleRoundStatus=COMPLETE round_completed_at + 30min`.

⑦ **TaskToken** (if project enables): token exists and not expired; `token_sig` verifies; `token.lease_id == bundle.lease_id`; `token.root_id == bundle.root_id`.

⑧ **TargetSlot validation**: Must pass three-layer schema validation of `slot_ns / slot_id / params` (format, length, allowed key set, action-verb prohibition). Registry protocol takes priority over `schema_ref`. If `target_slot` is required and absent, reject.

⑨-a **Service object matching:**
- SINGLE: `bundle.target_ref` must match `ProjectLiteSpec.target_ref` (at minimum `anchor_type + anchor_id`; `target_slot` per project requirements).
- MULTI: `bundle.target_ref` must hit one entry in `ProjectLiteSpec.target_refs[]`.

⑨-b **Geographic scope validation:**
- `anchor_type=SPACE`: `grid_id` must fall within `location_scope`. If `geo` is submitted, map to `grid_id` first.
- `anchor_type≠SPACE`: If `geo/grid_id` is provided, it must fall within `location_scope` (optional validation).

⑨-c **Anchor registry validation:**
- `anchor_id` must be Active in `anchor_registry_ref`.
- If `zone_id/gateway_id` is provided: `anchor_id` attribution must be consistent (per `anchor_registry_ref`).
- `target_slot` must satisfy the registry's `allowed_target_slots`.

⑩ **Anti-replay:**
`anti_replay_key = H(project_id, version, service_type_id, canonical_target_ref_hash, unit_serial)`
`canonical_target_ref_hash = H(anchor_type, anchor_id, target_slot, ... canonicalized)`
`location_key` does not participate by default (unless `anti_replay_policy_ref` explicitly declares it). When `result_units[]` is non-empty, compute per result unit.

⑪ **Participant validation:**
- INDIVIDUAL: `root_id` consistent; `device_sig` verifies against `device_pubkey_set`.
- GROUP: `member_set_ref` referenceable; `group_signatures` covers all members (n-of-n); each signature valid over the same `evidence_bundle_hash`. Failure: direct rejection.

⑫ **Device binding**: If `device_pubkey` is not in `device_pubkey_set`, prompt "device not bound / mismatch" and mark `risk_flag`. Whether to reject is decided by `acceptance_policy_ref` or PSA-3.

⑬ **WorkLease:**
- `lease_id` exists and `state=HELD` (or policy-equivalent).
- `lease_key` locks a specific slot instance (including `unit_serial`); only one person may hold it at a time. FINALIZED slots may not be re-acquired; RELEASED slots may.
- `WorkLease.bound_presence_session_id`'s `session.root_id == bundle.root_id`.
- `bundle.lease_event_head_hash == WorkLease.lease_event_head_hash`.

⑭ **Concurrency/freeze**: Concurrency key must not simultaneously hold multiple HELD/PAUSED WorkLeases. Must not be within an active `ParticipantProjectAcceptanceFrozen` period.

**Output**: `ServiceSubmitted(project_id, version, service_tag, evidence_bundle_hash, participant_ref, root_id, lease_id, time_completed)` → enters pending acceptance queue.

---

### Flow 4: ConfirmService

**Input**: `service_tag` + verifier signature (or automated validation output)

**Validation:**
- `acceptance_rule` satisfied; acceptance mode compliant with `acceptance_policy_ref` risk-tiered policy.
- Verifier role/permission compliant with `acceptance_policy_ref`.
- **Confirmation lock** (second anti-replay check): The same `(project_id, version, service_type_id, canonical_target_ref_hash, unit_serial)` must not be confirmed twice. `location_key` / `object_tag_id` do not participate in the confirmation lock by default.
- AUTO/HYBRID may accept `scan_proof_hash` as validation input. Scan failure or contradiction triggers `ServiceDisputed` (adjudication belongs to PSA-3).

**Output:**

`ServiceConfirmed`:
- `WorkLease.state → FINALIZED`. The slot is permanently closed; no new acquisition or submission is accepted. Other slots under the same task are unaffected.
- The service record (`service_tag` / `evidence_bundle_hash`) enters the PSA-2 settlement queue. After PSA-2 completes, both PSA-1 and PSA-2 outputs enter PSA-3 audit. After PSA-3 approval, the record is sealed.
- **Demand saturation output** (for PSA-2 reward computation): `fulfilled_count` (total FINALIZED slots under this `target_ref + target_slot`) / `demand_unit` (original demand quantity, defined by `unit_spec` or `proof_policy_ref`) / `saturation_flag` (`TRUE` when `fulfilled_count ≥ demand_unit`). When `saturation_flag=TRUE`, PSA-2 may compute reward approaching zero. PSA-1 does not block new acquisitions.

`ServiceRejected / ServiceDisputed / timeout`:
- `WorkLease.state → RELEASED`. Slot returns to acquirable state (or enters `RISK_HOLD`, per `dispute_policy_ref` / `work_lease_policy_ref`).

```
ServiceConfirmed(project_id, version, service_tag, verifier_id, result=ACCEPT)
ServiceRejected(..., reason_code)
ServiceDisputed(..., dispute_id)   // entry point only; adjudication in PSA-3
```

---

*Author: Zhengyang Pan (潘正阳) | BBBigradish (一根大萝卜)*
