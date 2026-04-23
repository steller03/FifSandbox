# Thoracic Spine Metastasis Automation Strategy

**Status:** Draft v0.4.2 — V1 implementation north star, FROZEN
**Owner:** Stephen Eller
**Purpose:** Defines the strategy, decision points, primitives, and unresolved questions for automated palliative thoracic spine metastasis planning in ProtocolPlanner. This is the document that drives V1 implementation of 3D automated planning.
**Audience:** Self, future maintainers, adversarial LLM reviewers, dosimetry team reviewers, Claude Code planning passes.

**Revision note:** This is v0.4.2, the final freeze version. v0.4.1 went through a freeze confirmation review (fifth adversarial round) with three independent reviewers: one returned a clean GO, two returned GO WITH MINOR PATCHES with a combined 5-item edit list of stale cross-reference cleanup. v0.4.2 applies those 5 edits. The document is now frozen. A patch changelog for v0.4.1 → v0.4.2 is appended to §29.

**Freeze posture:** v0.4 is expected to be the freeze version. After v0.4, the next artifact is the architectural planning document that specifies interface contracts for the primitives named here. Any v0.5 would be a narrow patch addressing issues surfaced by a final pre-freeze review.

---

## 1. Purpose and Scope of This Document

This document drives V1 implementation. It is the source of truth for what gets built, in what order, and against what design constraints. Every architectural and primitive decision for 3D automation traces back to this document.

Three things this document is for:

1. **Define V1.** What ships, what does not ship, what the recipe contract looks like, what primitives are required.
2. **Force the technique-family abstraction to be load-bearing on day one.** T-spine has multiple technique variants that share most of their behavior. If the abstraction is wrong here, it will be wrong at every subsequent site.
3. **Build the primitives that WBRT deferred.** Real energy selection from depth. Real beam weighting from depth. The FIF generator as a first-class subsystem with real clinical workflow semantics. Cross-cutting helpers like `IsocenterHelper` with half-beam-block support.

---

## 2. Clinical Context

### 2.1 What palliative T-spine RT is

Thoracic spine metastasis radiotherapy is one of the highest-volume palliative treatments in any radiation oncology department. Indications include pain from osseous metastatic involvement (most common), prevention or treatment of pathologic fracture, treatment or prevention of cord compression (urgent), and palliation of paraspinal soft-tissue extension.

Standard fractionations:

- **30 Gy in 10 fractions** — most common multi-fraction regimen
- **20 Gy in 5 fractions** — common, faster turnaround
- **8 Gy in 1 fraction** — single-fraction for uncomplicated bone pain, increasingly common
- Other palliative regimens exist but are less common

The intent is **palliative**, but this does not mean "loose dosimetry." The cord is intentionally in the target, and our physicians apply a specific constraint: maximum dose to the spinal cord must remain below 105% of the prescription dose. This drives the entire dosimetric posture of T-spine planning — plans must be uniform, and the FIF workload is driven by satisfying the cord rule while keeping the global hot spot under control.

### 2.2 Why T-spine first

- **Highest-volume palliative site** — daily clinical impact, fast ROI on automation.
- **Multiple technique variants force the framework abstraction to be load-bearing on day one.**
- **Forces real implementation of energy selection and beam weighting** — both reusable across every future site.
- **FIF is more tractable to validate than WBRT FIF.** Hot spots are predictable in location.
- **Geometry is straightforward relative to other 3D sites.** The recipe still has to handle multi-level field extent, oblique beam paths through lung, patient midline determination, arc clearance for DCA, HBB placement, and the interaction between cord position and hot spot management. But there is no head pose estimation, no couch kick, no head-tilt math.
- **Cord constraint is the dosimetric driver of plan evaluation and escalation.** Tighter homogeneity requirements drive plans toward more beams, higher energies, and more FIF segments, but this is realized post-dose-calc (via the cord Dmax acceptance test and the escalation ladder), not predicted upstream in Stage 1. The Stage 1 proposal itself is depth-driven, not cord-driven; see §7.4.

### 2.3 The engineering pattern T-spine establishes

Every primitive built for T-spine must be designed to be reused for pelvis, WBRT, larynx, and future 3D sites. T-spine is not a "spine recipe" — it is the **first composition of the 3D automation framework**, and the framework gets built by building T-spine. This includes cross-cutting primitives like `IsocenterHelper` (with HBB support, §11) that T-spine exercises and every subsequent site inherits.

---

## 3. Clinical Eligibility Envelope and Design Boundaries

This section defines the non-negotiable boundaries of what the recipe does and does not do. These are not implementation details. They are clinical and architectural positions that the recipe is designed around.

### 3.1 The PTV is authoritative

The physician contours the PTV, CTV, and GTV. The recipe plans to those contours without attempting to validate or second-guess the physician's anatomic decisions. In particular:

- **The recipe does not identify vertebral levels as a safety function.** If the physician contours the wrong vertebra, that is an upstream contouring error that the recipe is not designed to detect.
- **Level identification, if performed at all, is a convenience for technique defaulting only.**
- **The recipe trusts the PTV as the ground truth for what gets treated.** Field length, beam directions, MLC fitting, HBB placement, and dose evaluation all derive from the physician-contoured structures.

### 3.2 The cord rule, operationally

The clinical rule is:

**`SpinalCord.Dmax < 105% of Rx`** (strict inequality) — measured on the *true cord* contour (not a PRV or expanded volume), obtained via the standard ESAPI DVH query on the cord structure. This is a routine operation that we perform all the time in clinical practice; the ESAPI API exposes it directly and behaves well at the dose grid resolutions we use (2.5 mm or finer, see §22.8).

The cord rule is a single floating-point comparison against a single threshold. There is no voxel-level boundary handling, no isodose surface intersection logic, no partial-volume reasoning.

**The cord rule is the acceptance test for cord dose.** It is not a separate optimization objective competing with global hot spot reduction. The two objectives are physically coupled in T-spine geometry — the hottest part of the target is typically near or at the cord-PTV interface, so lowering global Dmax lowers cord Dmax automatically. The FIF kernel pursues a single objective (minimize global Dmax subject to the cold spot constraint), and the cord rule is checked at convergence as a pass/fail acceptance test.

When the cord rule is not satisfied at convergence, the recipe surfaces the failure with an explicit recommendation to escalate technique or beam energy (§15.11). The recipe does not autonomously escalate; the dosimetrist decides whether to re-run. The pacemaker override (§8.5) is the one documented exception where cord rule deviation is expected and flagged as a known limitation rather than a failure.

### 3.3 Re-treats are out of scope (and always will be)

Re-irradiation planning is not performed by this recipe. Retreats are planned manually. The recipe does not query prior RT, does not compute cumulative cord dose, does not check for field overlap with prior treatments, and does not surface prior-RT flags. This is a deliberate design boundary, not a deferral.

### 3.4 Only 3D conformal single-isocenter targets

The recipe is exclusively for 3D conformal techniques on single-isocenter targets. The following are categorically out of scope:

- VMAT and IMRT
- Stereotactic spine SBRT
- Multi-isocenter workflows
- Craniospinal irradiation
- Proton or other particle therapies

### 3.5 No wedges, no compensators

**Dynamic wedges (EDW) and physical wedges are explicitly excluded.** Not deferred, not optional — excluded. Wedge commissioning is not universal across clinics.

**Electronic compensators are a theoretical alternative but are also out of scope.** FIF is operationally simpler and is the committed V1 approach. Compensators are not in scope now or in any future version of this recipe.

The implication: **field-in-field is the only dose uniformity tool available to this recipe.** All cord rule satisfaction, all global hot spot reduction, all depth compensation residual cleanup goes through FIF subfield generation.

### 3.6 No CC:Gen dependency

The recipe does not depend on CC:Gen as a structure generation pipeline. Structures arrive at the recipe from two sources: **physicians contour targets (PTV, CTV, GTV)** and **Limbus produces OAR contours automatically**. The recipe inspects existing structures via the structure contract validator (§6.3); it does not invoke a structure generation pipeline.

The recipe performs its own **scoped structure operations** inside FIF: converting isodose regions to temporary structures for BEV projection during subfield synthesis, and cleaning up those temporary structures when the subfield is accepted, undone, or on FIF exit. These operations are direct ESAPI calls inside the FIF kernel and are not a CC:Gen integration.

**No per-vertebra structures are used.** The `LevelEstimator` works from the PTV centroid and OAR anatomical landmarks (§7.6), not from vertebral body contours. A generic bulk "VB" structure may be present on some CTs but is not consumed by the recipe.

### 3.7 Eligibility refusal conditions

The canonical list of refusal conditions lives in §19.1 (hard gates). The recipe will refuse to run on any case where:

- Required structures are missing or fail the structure contract
- The target geometry is outside V1 scope (see §10 and §19.1)
- Required machine capability is unavailable
- The case is flagged upstream as a re-treatment (out of scope)
- The case is flagged upstream as cord compression with spinal hardware
- Patient anatomy is outside expected ranges (suggests setup or scan error)
- Image quality is insufficient for reliable dose calculation
- The dose grid resolution is coarser than the recipe minimum (see §22.8)

### 3.8 What eligibility does NOT gate on

- Level identification confidence
- Prior RT status
- Physician preference for specific beam arrangements (handled in Stage 1 confirmation; if the physician wants a technique the recipe does not support, the recipe declines to plan and the case goes to manual)
- Exact fractionation
- Pre-dose-calc cord feasibility (no reliable estimator exists before dose calculation)

---

## 4. In-Scope / Out-of-Scope for V1

### In scope

- Single-level and contiguous multi-level (≤4 levels) thoracic spine palliative RT
- Four technique variants: 2-field (PA + anterior DCA), 3-field (PA + posterior obliques), 4-field (AP + PA + posterior obliques), **hybrid 3-field posterior + anterior DCA** (terminal escalation rung, §15.11)
- Automated technique *recommendation* from patient depth and clinic configuration (dosimetrist confirms in Stage 1 before planning commences); cord rule compliance is determined post-FIF, not predicted upstream
- **Half beam block (HBB) as a dosimetrist-selectable option on zero, one, or two axes** (§11)
- Depth-driven energy selection, using ESAPI radiological path length if available and geometric depth with lung-path bias as fallback
- Per-arrangement default field weight tables, with analytic AP/PA depth-compensated adjustment for the 4-field case
- Energy override for pacemaker patients (§8.5), capped at 6X
- Field-in-field hot spot reduction with cord Dmax acceptance check and plateau-based exit
- Standard fractionations (30/10, 20/5, 8/1) plus arbitrary dose/fraction inputs
- Single-isocenter plans only
- Selectable PTV coverage normalization (default V100 = 95%)
- Three-state acceptance result (complete / review needed / aborted with reason)
- Structured decision log with recipe/config version provenance
- Setup field generation (kV pair, CBCT)
- DRR generation
- Plan/course/field naming per clinic convention
- Couch model insertion
- Two-stage user workflow in a single-process executable (Stage 1 synchronous proposal, Stage 2 asynchronous-feeling execution)
- Plan ownership and cleanup on partial failure

### Out of scope (V1 and beyond)

- Re-irradiation (permanently out of scope, planned manually)
- Prior RT checking
- VMAT, IMRT, SBRT, CSI, multi-isocenter workflows
- **Dynamic wedges (EDW), physical wedges, electronic compensators**
- Junction matching with prior or adjacent fields
- Cumulative cord dose calculation
- Hippocampal-sparing or any IMRT-requiring technique
- Lumbar spine (V1.1), cervical spine (future)
- Patient contouring validation (PTV is authoritative)
- CC:Gen-driven structure generation (recipe performs scoped structure ops internally, not via CC:Gen)
- Level identification as a safety gate
- Pre-dose-calc cord feasibility prediction
- Autonomous technique escalation on cord rule failure
- Recipe-driven HBB recommendation (HBB is always dosimetrist-selected)

---

## 5. Why Four Techniques, and How They Compose

### 5.1 The four techniques

| Technique | Beams | Typical use case |
|---|---|---|
| **2-field PA + anterior DCA** | PA static + anterior dynamic conformal arc (~50°-310°, ~100° arc span) | Single levels, thinner patients; growing clinical usage |
| **3-field** | PA + LPO + RPO (approximately 145° / 215°) | Moderate patients needing spread entrance dose across posterior |
| **4-field** | AP + PA + LPO + RPO | Thicker patients needing anterior contribution for depth and cord homogeneity |
| **Hybrid** | PA + LPO + RPO + anterior DCA | Terminal escalation rung — used when 4-field at highest available energy cannot meet the cord rule; overkill for routine cases, deliberately chosen when escalation demands it |

These are not four different recipes. They are **four composition modes of a single T-spine recipe.** The recipe recommends one of the base three (2-field, 3-field, 4-field) in Stage 1; the hybrid is only reached via the §15.11 escalation path when the base techniques have exhausted their envelope.

### 5.2 What the techniques share

- Field length determination (anatomy-driven from target SI extent + margin)
- Isocenter placement rule (target geometric center, with HBB offset if selected)
- Energy selection rule (depth-based with effective path length, plus pacemaker override)
- MLC fitting (BEV projection of target plus PTV block margin; HBB is handled via isocenter placement, not at the fitter level)
- **FIF generator with cord Dmax acceptance check** (the same generator runs for all four techniques via policy selection)
- Normalization (selectable mode)
- Acceptance gates
- Setup field generation
- Decision log format
- Couch model insertion
- Structure contract validation
- HBB logic (applies uniformly across all four techniques)

### 5.3 What the techniques differ in

Only four things differ across techniques:

1. **Number, angles, and delivery mode of beams** (output of `BeamArrangementBuilder`, controlled by a `techniqueVariant` parameter)
2. **Default starting field weights** (each technique has its own default weight table; the 4-field default is then adjusted analytically for the AP/PA pair based on depth)
3. **DCA control point generation** (2-field and hybrid use DCA; 3-field and 4-field do not)
4. **FIF subfield synthesis policy assignments** — the kernel and policies are shared, but which beams receive subfield synthesis depends on the arrangement (static beams do; DCA beams do not)

Every other pipeline step is shared. This is the technique-family pattern made concrete.

### 5.4 Single-process two-stage execution

The recipe runs as a **single standalone executable** built on `Pscripts.Planning`. It is not a plugin launched from the Eclipse UI and it does not hand off work to a separate background process. The patient context, the ESAPI session, and the recipe state all live in one process from start to finish.

The "two-stage" framing refers to the **user experience**, not the process model:

- **Stage 1** is synchronous and fast. The user interacts with the UI to confirm or override the recipe's proposal. The ESAPI session is open, the patient is loaded, and all state is in memory.
- **Stage 2** is the execution phase. The user is not blocked — they can minimize the window and do other work while dose calculation and FIF run — but the process is the same process as Stage 1, using the same ESAPI session against the same patient.

This resolves the execution model cleanly: there is no serialization boundary between stages, no session re-acquisition, and no state reconstruction. The Stage 1 result is just in-memory data that Stage 2 consumes directly.

```
Executable starts, opens patient, acquires ESAPI session.

Stage 1 — Proposal (synchronous, seconds)
  1. validate_inputs()                       # structure contract, geometric sanity, eligibility
  2. estimate_level_from_anatomy()           # PTV centroid + OAR anatomical landmarks
  3. propose_technique_and_energy()          # advisory; uses depth + level hint + clinic defaults
  4. emit_proposal_to_ui()                   # presents to dosimetrist
  5. await_confirmation()                    # dosimetrist confirms technique, energies, normalization, HBB, pacemaker flag

Stage 2 — Execution (asynchronous-feeling, 5-10 minutes)
  6. apply_confirmed_inputs()                
  7. place_isocenter(hbb_selection)          # target center + HBB offset if selected
  8. build_beam_arrangement(technique, hbb)  # beams honor HBB jaw / collimator constraints
  9. fit_mlc_to_each_beam()                  # HBB is transparent to the fitter; refit from HBB-offset isocenter
  10. apply_default_weights_for_arrangement()
  11. adjust_ap_pa_analytically()              # 4-field only
  12. calculate_dose()                       
  13. run_fif_until_plateau()                # plateau-based exit, not threshold
  14. normalize()                            
  15. recheck_acceptance_post_normalization()
  16. emit_decision_log_and_report()
  17. return three-state result

Executable exits, session closes.
```

---

## 6. Inputs and Structure Contract

### 6.1 Required structures

The recipe refuses to run without these:

| Structure | Purpose | Source | Notes |
|---|---|---|---|
| `PTV_Spine` (or per-level equivalent) | Target volume for coverage, MLC fitting, normalization, HBB extent calculation | Physician | Authoritative per §3.1 |
| `CTV_Spine` (or `GTV_Spine`) | Underlying clinical target, BEV projection source | Physician | |
| `SpinalCord` | **Subject of the cord Dmax acceptance check** — must be the *true cord* contour, not a PRV | Limbus | Closed, contiguous volume |
| `External` / `Body` | Patient separation, skin position, dose calculation boundary | Standard autocontour | |
| Lung structures | Effective path length calculation for oblique beams, level estimation anchors | Limbus | |

The structure contract validator detects if `SpinalCord` is a PRV or expanded volume (by structure code, naming, or geometry) and refuses the case rather than silently applying the cord rule to the wrong volume.

### 6.2 Optional structures

| Structure | Purpose |
|---|---|
| Heart, carina, great vessels, esophagus, stomach, liver, kidneys | **Level estimation anatomical anchors** (§7.6); level-appropriate OAR sanity bounds (soft, report-only) |
| Pacemaker / CIED contour | Future enhancement: may inform isocenter placement for divergence avoidance (§8.5). Not used in V1 beyond acknowledging its presence. |

Per-vertebra structures are **not used** by the recipe. A generic bulk "VB" structure containing all vertebrae may be present on some CTs but is not consumed.

### 6.3 Structure contract validation

The `StructureContractValidator` enforces:

**Metadata-level checks:**
- Structure IDs match expected codes via `99VMS_STRUCTCODE` scheme
- DICOM type matches expected (ORGAN, PTV, EXTERNAL)
- The `SpinalCord` structure is identified as the true cord, not a PRV or expanded volume

**Geometric-level checks:**
- Each required structure is non-empty
- Each required structure is a single connected component (or a known multi-component exception, e.g., paired lungs)
- No zero-volume slices within the structure's SI extent
- Structure centroid is inside the body contour
- **Containment hierarchy: GTV ⊆ CTV ⊆ PTV** with small tolerance (where all three are present; GTV may be absent in palliative cases)

**Contract-level checks:**
- All required structures present
- Structure code-to-name mapping is self-consistent

Failure of any check triggers refuse-and-explain before planning begins. The validator's output is part of the decision log.

**Calibration posture:** The geometric checks should start *loose* and tighten with validation experience. Real physician and Limbus output can be messier than the theoretical contract anticipates — physician contours can have small disconnected regions from editing artifacts, Limbus output can have slice gaps in low-contrast regions, and false rejections will erode dosimetrist trust faster than missed catches. The validator's tolerances are clinic-configurable, and the V1 release configuration starts conservative. Tightening happens as validation data accumulates.

### 6.4 Other inputs

- **Prescription:** Total dose, fractionation, normalization mode (V100 = 95% default, V100 = 97%, D95%, PTV mean, or isocenter)
- **Machine:** Linac selection; recipe queries available energies, MLC type, max field size, max MU per field, jaw limits
- **Clinic preferences:** Default technique preferences, energy thresholds, technique selection thresholds, FIF parameters, default weight tables per arrangement — all in a versioned configuration file
- **Pacemaker flag:** Mandatory Stage 1 user input (yes/no, no default; see §8.5)
- **HBB selection:** Optional Stage 1 user input specifying zero, one, or two HBB axes (§11)
- **PTV block margin:** Clinic-configurable, typically 8 mm for 3D plans. This is the distance from the PTV edge to the MLC block edge, used by the `MLCFitter` for BEV projection margin and by the `IsocenterHelper` for HBB offset calculation (§11.2). Surfaced as a user-configurable value in the Stage 1 UI so the dosimetrist can override the clinic default on a per-case basis.
- **Couch model:** Inserted if not already present

---

## 7. Technique Selection (Advisory, Stage 1)

### 7.1 The workflow

Technique selection in Stage 1 is **advisory**. The algorithm runs fast (target < 30 seconds) and produces a recommendation that the dosimetrist reviews before Stage 2 begins.

```
Stage 1 — Proposal:
1. Algorithm measures posterior-to-target depth, anterior-to-target depth, and patient separation at target level.
2. Algorithm proposes a base technique (2-field, 3-field, or 4-field) based on depth thresholds and clinic-configured per-level defaults.
3. Algorithm proposes per-beam energies based on depth (and pacemaker override if flagged).
4. Algorithm proposes the normalization mode (V100 = 95% by default).
5. The proposal is presented to the dosimetrist, including the reasoning trace.
6. Dosimetrist confirms, modifies (technique, energies, normalization, pacemaker flag, HBB selection), or cancels.

Stage 2 begins after confirmation.
```

### 7.2 What the recipe does NOT propose

- **HBB is never recipe-proposed.** HBB is a dosimetrist-specified input (§11). The recipe does not suggest HBB usage; the dosimetrist selects it independently of the technique proposal.
- **The hybrid technique is never recipe-proposed in Stage 1.** It is only reachable via the §15.11 escalation path when base-technique FIF has failed to meet the cord rule.

### 7.3 Inputs to the proposal algorithm

The relevant measurement is not total separation. It is the depth from each candidate beam direction to the deepest target point. Specifically:

- **Posterior-to-target depth:** distance from posterior skin to the anterior aspect of the target, along the PA beam axis
- **Anterior-to-target depth:** distance from anterior skin to the posterior aspect of the target, along the AP beam axis
- **Oblique depths:** computed at LPO and RPO angles, each to the target center

### 7.4 No pre-dose-calc cord feasibility check

The proposal algorithm uses depth and clinic-configured defaults; the cord rule is checked post-dose-calc in §15. If the cord rule fails, the recipe surfaces the escalation recommendation (§15.11) and the dosimetrist decides whether to re-run with a different technique or energy.

### 7.5 Proposal algorithm (V1 strawman)

```
IF posterior-to-target depth is small (clinic threshold A):
    PROPOSE 2-field PA + DCA  (subject to DCA feature gate)
ELSE IF posterior-to-target depth is moderate (clinic threshold B):
    PROPOSE 3-field
ELSE:
    PROPOSE 4-field

OVERRIDE by level estimator:
    If level estimator suggests a different default for this PTV centroid, use the level default instead.

OVERRIDE by clinic configuration:
    If clinic configuration disables a technique, propose the next-up technique instead.
```

Thresholds are clinic-configurable and must be validated against historical data. The validation methodology is to pull historical cases, classify each by what technique was actually used, and tune the depth thresholds so the algorithm's proposal matches the historical choice on at least 80% of cases.

### 7.6 The level estimator (anatomy-driven, not vertebra-driven)

The `LevelEstimator` determines the approximate thoracic level of the PTV using **PTV centroid position relative to OAR anatomical landmarks**. No per-vertebra structures are consulted. The anchors the estimator uses are inferred from auto-contoured OARs:

- **Carina** — approximately T4-T5
- **Top of heart** — approximately T5-T6
- **Bottom of esophagus** — approximately T11
- **Top of stomach** — approximately T11-T12
- **Diaphragm dome** — approximately T10-T12
- **Kidney superior poles** — approximately T12-L1

By comparing the PTV centroid's SI position to these anchors, the estimator classifies the PTV as upper T-spine (T1-T4), mid T-spine (T5-T9), or lower T-spine (T10-T12). This classification drives technique defaulting and OAR sanity bound selection.

The estimator works even when not all anchors are present — it uses whichever subset of OAR landmarks are available. If no anchors are available, the estimator falls back to PTV centroid SI coordinate only and flags low confidence, but this does not block the recipe.

Per §3.1, this estimator is not a safety function.

### 7.7 Open questions

- Threshold values for depth-based technique proposal — validate against historical data
- Per-level default table — clinic input required
- OAR anchor SI coordinate tables (what Z value corresponds to each anchor in typical anatomy)
- How HBB selection is presented in the Stage 1 UI — UI workstream concern

---

## 8. Energy Selection

### 8.1 The decision

For each beam in the arrangement, choose an energy from the linac's available list such that the target receives adequate dose without excessive depth-dose falloff or skin dose concerns. The dominant input is **effective depth from entrance to the deepest target point along the beam axis**.

### 8.2 Effective depth via ESAPI radiological path length

**V1 approach:** Use ESAPI's radiological path length calculation if available.

**Fallback:** Geometric depth with a lung-path bias heuristic — for beams whose path traverses substantial lung, the recipe biases the energy selector upward to the next available energy.

**Out of scope:** Custom HU-to-density ray casting.

### 8.3 Algorithm

```
FOR each beam in the arrangement:
    1. Compute entrance point on patient surface for this beam direction.
    2. Get effective depth (ESAPI path length if available, geometric otherwise).
    3. If geometric fallback is in use AND beam path traverses lung structure:
         bias depth upward to next energy bin (lung correction heuristic).
    4. Look up energy from the (effective depth → energy) table for this machine.
    5. If the selected energy is not available on this machine, fall back upward to the next available energy.
    6. Apply pacemaker override if active (§8.5) — cap at 6X.
    7. Log the decision with depth, effective-or-geometric flag, lung-correction flag, override flag, and selected energy.
```

Strawman depth-to-energy table (validate against clinic practice):

| Effective depth to deepest target point | Preferred energy |
|---|---|
| ≤ 8 cm | 6 MV |
| 8 – 13 cm | 6 MV or 10 MV |
| 13 – 18 cm | 10 MV or 15 MV |
| > 18 cm | 15 MV |

### 8.4 Why this primitive matters beyond T-spine

Every site uses energy selection. The depth table changes per site; the interface does not.

### 8.5 Pacemaker override

For patients with a cardiac implantable electronic device (CIED — pacemaker, ICD, CRT-D, treated the same for V1 purposes), high-energy beams must be avoided due to neutron production.

- **Mandatory Stage 1 user prompt:** "Does this patient have a pacemaker?" — Yes/No, no default. The dosimetrist must answer. This is intentional CYA behavior; forcing explicit acknowledgment reduces the risk of missed CIED cases.
- **Energy cap when Yes:** 6 MV only. All beams in the arrangement are capped at 6X regardless of what the depth-to-energy table would have selected.
- **Acceptance posture:** When the pacemaker flag is set, the recipe acknowledges that the cord rule may not be achievable at the constrained energy. FIF runs to plateau as normal, and if the cord rule fails at exit, the failure is logged as a **known-limitation soft flag** rather than a recipe failure. The dosimetrist reviews the plan understanding that the cord rule deviation is a documented consequence of the pacemaker constraint.
- **Decision log:** The pacemaker flag, the 6X energy cap, and the resulting cord Dmax are all recorded with explicit "known limitation: pacemaker energy override active" notation.

**Pacemaker contour as future enhancement:** If the CT has a contoured pacemaker structure, the recipe may in a future version use its geometry to inform isocenter placement or beam angle avoidance for divergence-favorable positioning. This is acknowledged as a future enhancement and is not implemented in V1 — V1 only enforces the energy cap. The recipe ignores the pacemaker contour for geometric purposes but may log its presence in the decision log.

---

## 9. Beam Weighting

### 9.1 Per-arrangement default weight tables

Each technique has a **default starting weight table**, applied to all beams simultaneously at the start of dose calculation. The 4-field arrangement also receives an analytic AP/PA depth-compensated adjustment on top of the default. Defaults are clinic-configurable.

Strawman defaults (validate against clinic practice):

| Technique | Default starting weights |
|---|---|
| **2-field PA + DCA** | PA 70%, DCA 30% |
| **3-field PA + LPO + RPO** | PA 50%, LPO 25%, RPO 25% |
| **4-field AP + PA + LPO + RPO** | AP 25%, PA 35%, LPO 20%, RPO 20% (then AP/PA pair adjusted analytically) |
| **Hybrid PA + LPO + RPO + anterior DCA** | **TBD — derive from retrospective cases during validation.** The architecture commits to per-arrangement defaults; the specific numbers for the hybrid case will be tuned from clinical data before Phase 7 rollout. |

All beams in 4-field come in **simultaneously** at the default weights before dose calculation, then the analytic AP/PA adjustment shifts the balance. FIF then refines all weights as it generates subfields.

### 9.2 Analytic AP/PA depth-compensated adjustment

For the AP/PA pair in 4-field, the textbook depth-compensated weight ratio is:

```
weight_PA / weight_AP = exp(μ_eff × (d_AP - d_PA))
```

**Composition with default weights:**

1. Start with the default arrangement weights (strawman: AP 25%, PA 35%, LPO 20%, RPO 20%).
2. Compute the analytic ratio `r = weight_PA / weight_AP` from depth.
3. Hold the *sum* of AP and PA weights constant at 60%. Redistribute: `new_AP = 60% / (1 + r)`, `new_PA = 60% × r / (1 + r)`.
4. Leave oblique weights unchanged at defaults.
5. Apply all four weights simultaneously before initial dose calculation.

This is one equation, applied once before dose calculation, not an optimization.

**FIF does not preserve the analytic ratio.** When FIF generates subfields and adjusts field weights, the analytic ratio is a starting condition, not an invariant. Both pre- and post-FIF weights are logged in the decision log.

### 9.3 The BeamRoleContext interface

The `BeamWeightSolver` takes a `BeamRoleContext` parameter carrying semantic information about each beam's role — dominant coverage, depth-compensation partner, oblique fill, DCA fill — rather than a flat technique enum. The context carries (a) the beam's role in the arrangement, (b) its delivery mode, (c) its FIF policy reference, and (d) a reference to its sibling beams for relational logic. Full field enumeration is deferred to the architectural planning phase.

### 9.4 Open questions

- Default weight tables — validate against historical practice
- Hybrid technique default weights — TBD, derive from retrospective data
- μ_eff values per energy — validate against dosimetry measurements or published tables

---

## 10. Field Length and Multi-Level Handling

Field length in the SI direction is determined by the target's SI extent plus a margin for setup uncertainty and microscopic disease (strawman 5-10 mm, clinic configurable).

Per §3.1, the PTV is authoritative. The recipe uses the PTV's SI extent directly.

**Constraints:**
- Single-level: the simple case
- Multi-level (2-4 contiguous levels): supported, treated as a single concurrent volume with one field length
- More than 4 contiguous levels: out of scope for V1 (refuse-and-explain)
- Non-contiguous levels with vertebral skipping: out of scope — clinical practice is to treat skipped-level cases as **separate plans on separate isocenters**. The recipe refuses to plan a non-contiguous target; the dosimetrist re-submits each contiguous segment as its own recipe run.
- Junction matching with prior fields: out of scope permanently

The non-contiguous refusal is detectable from the PTV structure itself — if the PTV is multi-component along the SI axis with gaps, that is a non-contiguous case. The structure contract validator catches this in the connectivity check.

**HBB interaction:** When HBB is selected on the S/I axis (superior or inferior), the field length calculation is adjusted — the field extends from the HBB edge outward, not symmetrically from the target center. The `IsocenterHelper` handles this when computing the HBB-offset isocenter (§11).

---

## 11. Isocenter Placement and Half Beam Block

### 11.1 Default isocenter placement

For routine cases without HBB, isocenter is placed at the target geometric center. The algorithm may apply minor adjustments:
- **Rounding** to clean-number coordinates per clinic convention
- **Small anterior offset** for gantry/couch clearance reasons
- **4-field AP offset** to balance AP/PA contributions (configurable)

The algorithm verifies the isocenter falls inside the body contour; if not, refuse-and-explain.

### 11.2 Half Beam Block (HBB)

HBB is a dosimetrist-selectable option that modifies isocenter placement to produce beams whose divergence is clipped at the isocenter plane on one or two axes. It is a cross-cutting primitive from `Pscripts.Planning.IsocenterHelper` and will be reused by every subsequent 3D site, not just T-spine.

**What HBB does:** For each selected HBB axis, the isocenter is offset from the target center by half the PTV extent along that axis **plus the PTV block margin** (§6.4 — the same margin used by the `MLCFitter` for BEV projection, typically 8 mm for 3D plans), so the isocenter lands exactly at the block edge. The jaw on the blocked side is then set to zero at isocenter, which clips divergence at the isocenter plane — no beam divergence beyond the HBB.

Using the PTV block margin for HBB (rather than a separate HBB-specific margin concept) keeps the geometry consistent: the block margin defines where the block edge falls relative to the PTV for every beam in the plan, and HBB places the isocenter at that block edge. There is no "PRV margin" concept in the recipe; the cord is evaluated on the true cord (§3.2), not a PRV-expanded volume.

**Axes and directions:** HBB can be applied on zero, one, or two axes from the following:

- **S/I axis:** Superior (S) or Inferior (I)
- **L/M axis:** Lateral (L) or Medial (M)
- **A/P axis:** Anterior (A) or Posterior (P)

Six single-axis options, plus twelve two-axis combinations, plus the no-HBB default.

**Complementarity constraint:** Two-axis HBB must use *different* axes. Valid: Superior + Medial, Inferior + Anterior, Lateral + Posterior. Invalid: Superior + Inferior (same axis), Lateral + Medial (same axis), Anterior + Posterior (same axis). The `IsocenterHelper.PlaceForHBB()` API validates this and rejects same-axis combinations.

The two-axis case is uncommon but the recipe supports it. Geometrically, two-axis HBB treats only one quadrant of the BEV per beam.

### 11.3 Collimator constraint

HBB requires the collimator to be aligned with the HBB axes so that jaw settings can clip divergence in the correct directions.

- **Default: collimator 0°.** For single-axis HBB on S/I, the Y jaw clips; for single-axis HBB on L/M or A/P, the X jaw clips. For two-axis HBB (always two perpendicular axes due to complementarity), both X and Y jaws clip simultaneously at collimator 0°.
- **Alternative: collimator 90°.** Used for long narrow targets where 90° is favorable. Uncommon. When selected, the jaw roles swap (X clips what Y would have clipped and vice versa).

The `BeamArrangementBuilder` forces the collimator to 0° by default when HBB is active, overriding any clinic-convention collimator rotation. The 90° option is available via clinic configuration or dosimetrist override.

### 11.4 DCA interaction

HBB applies to all beams in the arrangement, including the DCA if one is present. For the 2-field PA + DCA case with superior HBB (the most common HBB + DCA scenario for T-spine), the DCA's control points all inherit the jaw-at-zero-at-isocenter setting from the beam-level jaw definition. Eclipse propagates jaw settings to control points automatically.

The recipe verifies the jaw does not exceed the jaw limit (typically 0.0 cm at isocenter — fully clipped). Small geometric deviations are accepted and logged; the specific deviation magnitude is recorded in the decision log. The 95% solution is to set the HBB jaws at 0 cm for no divergence, and deviations from that target are captured for review.

### 11.5 MLC fitting interaction

HBB changes where the isocenter is placed (§11.2), and the `MLCFitter` is called with the HBB-offset isocenter. From the fitter's perspective, HBB is transparent — the fitter receives an isocenter position and a target, fits MLC leaves to the BEV projection of the target plus the PTV block margin (§6.4), and returns the leaf positions. The geometry naturally produces correct leaf positions relative to the new isocenter because the isocenter is already at the block edge and the jaw on the blocked side clips everything beyond the isocenter plane.

**No post-fit leaf override is required.** The `MLCFitter` does not need HBB awareness as a parameter — HBB awareness lives entirely in the isocenter placement via `IsocenterHelper.PlaceForHBB()`. When the recipe shifts the isocenter for HBB, it simply re-invokes the standard MLC fitting method from the new isocenter position, and the leaves are refit naturally. This is the same pattern used any time the recipe changes isocenter for any reason (anterior offset for clearance, 4-field AP offset, etc.) — MLC refit follows isocenter change.

**Post-fit jaw sanity check.** After the MLC fit completes for an HBB beam, the recipe verifies that the jaw on the blocked side is set to 0 cm at isocenter as expected. If the jaw is not at 0 cm (e.g., due to a geometric edge case), the deviation is logged per §11.4. This check is cheap and is the only HBB-specific post-processing step on the beam.

### 11.6 HBB is never recipe-proposed

HBB is always a dosimetrist-specified input. The recipe does not propose HBB in Stage 1 and does not recommend HBB during escalation. If the dosimetrist re-runs after escalation, the HBB selection persists from the original run unless explicitly changed.

### 11.7 The IsocenterHelper API

The `IsocenterHelper` service in `Pscripts.Planning` exposes:

```
IsocenterHelper.PlaceForTarget(ptv, arrangementConfig) → IsocenterPosition
IsocenterHelper.PlaceForHBB(ptv, hbbSelection, arrangementConfig) → IsocenterPosition
```

Where `hbbSelection` is a value object containing zero, one, or two axis-direction values (with same-axis validation). The helper returns the offset isocenter plus any metadata needed by downstream primitives (jaw constraints, collimator constraints).

This helper already exists in `Pscripts.Planning` in a simpler form; V1 extends it with the HBB support described here. Every subsequent 3D site will use the same helper.

---

## 12. Beam Arrangement by Technique

### 12.1 Beam intents carry clinical intent, not just mechanics

The `BeamArrangementBuilder` returns **beam intents** that carry both machine-agnostic geometry and clinical intent fields:

- Role (dominant coverage, depth-compensation partner, oblique fill, DCA fill)
- Whether the beam supports hotspot subtraction in FIF
- Delivery mode (static, dynamic conformal arc)
- Which subfield synthesis policy FIF should apply to this beam
- **HBB awareness:** the beam intent carries the HBB selection so that jaw limits and collimator constraints are correctly set when the intent is realized as an ESAPI beam

### 12.2 2-field PA + anterior DCA

```
Beam 1: PA static
  - Gantry 180°, Couch 0°, Collimator 0° (HBB default) or per clinic convention
  - Role: dominant coverage
  - Delivery: static
  - Subfield policy: static-beam policy

Beam 2: Anterior Dynamic Conformal Arc
  - Gantry: ~50° to ~310° (approximately 100° total arc span)
  - Couch 0°, Collimator 0°
  - Role: DCA fill
  - Delivery: dynamic conformal arc
  - Subfield policy: DCA policy (no subfield synthesis in V1)
```

### 12.3 3-field PA + posterior obliques

```
Beam 1: PA static
Beam 2: LPO — Gantry ~145°, oblique fill
Beam 3: RPO — Gantry ~215°, oblique fill
```

### 12.4 4-field AP + PA + posterior obliques

```
Beam 1: AP — Gantry 0°, depth-compensation partner
Beam 2: PA — Gantry 180°, dominant coverage, depth-compensation partner
Beam 3: LPO — oblique fill
Beam 4: RPO — oblique fill
```

### 12.5 Hybrid PA + LPO + RPO + anterior DCA (terminal escalation)

```
Beam 1: PA static — dominant coverage
Beam 2: LPO — oblique fill
Beam 3: RPO — oblique fill
Beam 4: Anterior DCA — DCA fill
```

This is the configuration reached when 4-field at the highest available energy has failed to meet the cord rule. It exercises three static beams with subfield synthesis plus one DCA without, which is the most complex FIF configuration in V1 and is the specific case where the DCA route-and-undo logic (§15.4.1) gets stress-tested.

### 12.6 The BeamArrangementBuilder contract

```
BeamArrangementBuilder.build(
    technique: TechniqueVariant,
    target: Structure,
    isocenter: IsocenterPosition,   // includes HBB metadata
    machine: Linac,
    config: TechniqueConfig
) → List[BeamIntent]
```

Beam intents encode the HBB jaw and collimator constraints derived from the isocenter position.

---

## 13. MLC Fitting

Standard operation: BEV project the target, add the PTV block margin (§6.4, strawman 8 mm for 3D plans, clinic configurable), generate MLC leaf positions, enforce minimum leaf gap, optionally smooth, trim to jaw boundaries.

**HBB handling:** The `MLCFitter` does not need explicit HBB awareness. HBB lives in the isocenter placement (§11.2), and the MLC fitter is simply re-invoked with the HBB-offset isocenter. From the fitter's perspective, HBB is transparent — it receives an isocenter and a target, fits leaves to the BEV projection of the target plus the block margin, and returns. The jaw-at-zero clipping on the blocked side is handled at the beam level by the `BeamArrangementBuilder`, not by the MLC fitter. The recipe performs a post-fit jaw sanity check on HBB beams to confirm the blocked-side jaw is at 0 cm (§11.5).

Whenever the recipe changes the isocenter for any reason (HBB offset, 4-field AP offset, clearance adjustment), the MLC is refit from the new isocenter. This is the same pattern regardless of why the isocenter moved.

For T-spine palliative, generous isotropic margin is acceptable because the cord is in the target and conformality is less important than coverage at palliative dose levels. OAR-aware aperture modifications are not performed in V1.

For the DCA, MLC fitting is per-control-point — Eclipse handles control point MLC motion natively from beam-level jaw and MLC settings. When HBB is active on a DCA beam, the jaw setting propagates to all control points automatically.

**Open question (Phase 0):** Eclipse MLC fitting API — `Beam.FitMLCToStructure` vs direct leaf position manipulation. See Appendix A.3.

---

## 14. Dynamic Conformal Arc (DCA) Specifics

### 14.1 Arc parameters for T-spine

| Parameter | Strawman default |
|---|---|
| Start angle | 50° |
| Stop angle | 310° |
| Arc span | ~100° through anterior (going the short way) |
| Arc direction | CW or CCW per clinic convention |
| Control point spacing | Default Eclipse auto-generated (no custom control point manipulation) |

The recipe does not create custom control points. It creates a DCA beam with start/stop angles and lets Eclipse generate the control points natively. The `DynamicConformalArcBuilder` is a thin wrapper that sets arc parameters and delegates control point generation to Eclipse.

### 14.2 DCA contribution model for BeamContributionEstimator

The `BeamContributionEstimator` treats the DCA as a single beam with a total field weight contribution, not as an integration over control points. This is the simplified contribution model for V1 and is sufficient for the binary "is the DCA the dominant contributor to this hot spot?" question that the kernel needs.

Per-control-point integration is explicitly out of scope for V1 — we are not doing arc dosimetry modeling, just standard beam weighting.

### 14.3 Collision and clearance

Because the arc stays in the anterior quadrant, collision risk is minimal. A standard collision check is performed as sanity verification, not as a load-bearing safety primitive.

### 14.4 HBB interaction

See §11.4. HBB applies to DCA beams; the jaw-at-zero setting propagates to all control points via Eclipse's native behavior.

### 14.5 DCA feature gate

DCA is behind a clinic configuration feature gate. When disabled, the Stage 1 proposer skips 2-field entirely and proposes 3-field for cases that would otherwise be 2-field candidates. The hybrid technique is also disabled when the DCA feature gate is closed.

---

## 15. Field-in-Field Generation

### 15.0 FIF is the only dose uniformity tool

Per §3.5, dynamic wedges, physical wedges, and electronic compensators are categorically excluded. **Field-in-field is the only dose uniformity mechanism available.**

**FIF runs to plateau, not to a target Dmax.** The exit condition is not a Dmax threshold but a diminishing-returns criterion: when consecutive iterations stop producing meaningful improvement, FIF has done its job. See §15.5 for the primary plateau exit and §15.6 for the full set of exit conditions.

**Oscillation and convergence difficulty are engineering problems to handle**, not problems to avoid. The kernel must track the best plan seen across iterations (the `BestPlanTracker`), undo subfields that don't meaningfully improve the plan, and return the best plan on any exit condition.

### 15.1 The actual workflow (clinical and ESAPI)

The FIF workflow the recipe automates:

```
1. Identify the highest hot region: locate the voxel with current global Dmax, define a hot region at (current global max - 2-3%).
1.5. Convert the hot region to a temporary ESAPI structure by creating an isodose-based contour. This is a standard ESAPI operation. Tag the structure with a recipe-specific prefix so cleanup can find it on FIF exit or partial failure.
2. Generate a subfield on the dominant contributing beam by BEV-projecting the temporary hot-region structure onto the beam and closing MLC leaves over the projection.
3. Add the subfield to the parent beam. In Eclipse, subfields are additional fields that share the parent beam's structure but have their own MLC shape.
4. Adjust the parent beam's field weight via `Beam.GetEditableParameters()` to reduce its contribution, effectively transferring MU to the subfield.
5. Recalculate dose.
6. Check whether the subfield produced ≥0.3% ΔDmax improvement. If yes, accept the subfield and advance. If no, undo the subfield (revert field weight, remove the subfield) and try the same procedure on the next-ranked beam in the contribution list.
7. Clean up the temporary hot-region structure after the subfield is accepted or undone.
8. Check convergence per §15.5. Repeat until plateau, budget exhaustion, oscillation, or cold-spot exhaustion.
```

The temporary structure lifecycle (create in step 1.5, clean up in step 7) is a recipe-internal operation performed directly via ESAPI structure modification calls. There is no CC:Gen involvement. If the recipe exits partway through an iteration (e.g., crash, cancellation), the `PlanLifecycleManager` cleanup pass removes any tagged temporary structures along with the partial plan.

**Open implementation question (Phase 0, Appendix A.7):** The end-to-end FIF workflow (create subfield, adjust field weight, recalculate dose, read back composite) needs to be empirically demonstrated against ESAPI before Phase 4 begins. This is the gating Phase 0 question and the single largest schedule risk in the project.

### 15.2 Cord rule as acceptance test, not optimization objective

The cord rule (§3.2) is `SpinalCord.Dmax < 105% of Rx`, measured via standard ESAPI DVH query at FIF exit. It is an acceptance test, not an objective.

The FIF kernel pursues a single objective: **minimize global Dmax** subject to the cold spot soft constraint. The cord rule is checked after FIF exits. Pass/fail is a single floating-point comparison.

This works because the cord and global hot spot are physically coupled in T-spine geometry. There is no priority logic between "cord regions" and "global regions" inside FIF — they are not separate things.

When the cord rule fails at convergence, see §15.11 for escalation.

### 15.3 The kernel + policies pattern

| Component | Role |
|---|---|
| `HotspotLocator` | Finds the highest hot region above threshold; creates and manages the temporary isodose structure for BEV projection |
| `BeamContributionEstimator` | Given a hot region, estimates each beam's total contribution (static beams: standard ray-cast; DCA: total arc field weight) |
| `SubfieldSynthesisPolicy` | Given a beam and a hot region, produces a subfield (MLC segment + weight adjustment); different policies for static beams and DCA beams |
| `ColdSpotGuard` | Checks whether a proposed subfield would create an unacceptable cold spot; rejects subfields that would |
| `ConvergencePolicy` | Decides when to stop iterating; tracks iteration history including the N-beam-in-a-row failure counter and oscillation detection |
| `BestPlanTracker` | Maintains the best plan state seen across iterations; returns the best plan on any exit condition |

The kernel orchestrates these components. Full interface specification is deferred to the architectural planning phase.

### 15.4 Subfield synthesis policies

Two policies are required for V1:

- **Static-beam policy:** BEV-project the hot region onto the beam, close MLC leaves over the projection, reduce parent field weight by the subfield's MU fraction. Applies to all static beams in all four techniques.
- **DCA policy:** The DCA beam **does not receive subfield synthesis in V1**. When the `BeamContributionEstimator` identifies the DCA as the dominant contributor to a hot spot, the kernel routes subfield generation to the next-ranked static beam. See §15.4.1 for the DCA-dominated failure case.

#### 15.4.1 DCA-dominated hot spot failure case

When a hot spot exists in a region that only the DCA can reach effectively (e.g., anterior-lateral entrance dose), routing to a static beam may not improve global Dmax. The kernel handles this:

1. The routed subfield is applied and dose is recalculated.
2. The kernel checks improvement. If <0.3%, the subfield is undone (per §15.5).
3. The undone subfield counts as a failed iteration. The failure counter increments, and the kernel advances to the next iteration targeting the same hot region on the next-ranked beam.
4. If the kernel cycles through all eligible beams without finding a ≥0.3% improvement, FIF exits as converged (plateau) at the best plan seen via the N-beam-in-a-row rule.
5. At exit, if the cord rule failed, §15.11 escalation recommends moving away from DCA (e.g., 2-field → 3-field, or from 3-field without DCA where that's the starting point).

This is the specific configuration that stress-tests the DCA route-and-undo logic: the hybrid technique has three static beams and one DCA, meaning the kernel has three candidates to route to. The 2-field case has only PA as a candidate (N=1) and fails fast if PA can't fix a DCA-dominated hot spot.

### 15.5 Iteration discipline, plateau exit, and the N-beam-in-a-row rule

**One subfield per iteration.** Each iteration generates exactly one subfield, applies it, recalculates, and checks improvement.

**The 0.3% rule.** A subfield is *accepted* only if it produces ≥0.3% ΔDmax improvement. Subfields producing <0.3% improvement are *undone*: the field weight adjustment is reverted, the subfield is removed, the temporary structure is cleaned up, and the kernel advances to the next-ranked beam in the contribution list.

A 0.1% improvement means dose was pushed around without making the plan more uniform. Accepting such subfields accumulates noise, consumes the subfield budget, and obscures actual convergence. The 0.3% floor is a clinical heuristic — if a subfield can only buy you less than that, you have reached the practical limit for that beam.

**The N-beam-in-a-row rule.** FIF exits as converged when the kernel has tried **N consecutive subfield attempts that each failed the 0.3% rule**, where N equals the number of beams in the arrangement eligible for subfield synthesis (static beams only, not DCA beams):

| Arrangement | Eligible beams | N |
|---|---|---|
| 2-field (PA + DCA) | PA only | 1 |
| 3-field (PA + LPO + RPO) | PA, LPO, RPO | 3 |
| 4-field (AP + PA + LPO + RPO) | AP, PA, LPO, RPO | 4 |
| Hybrid (PA + LPO + RPO + DCA) | PA, LPO, RPO | 3 |

The failure counter **resets on any successful subfield** (≥0.3% improvement). The kernel tracks consecutive failures, not cumulative failures.

Interpretation: if the kernel has tried to improve the current hottest region by generating a subfield on every eligible beam and none of them produced meaningful improvement, the plan has reached the practical global minimum for that patient geometry and FIF is done.

**For 2-field, N=1 is the floor case.** The kernel has only PA as a candidate; one failed attempt exits FIF as converged. This is the correct behavior — there is nowhere else to send subfield work in a 2-field arrangement.

**Best-plan tracking.** The `BestPlanTracker` stores the lowest global Dmax (subject to cold spot constraint) seen at any iteration. On any exit condition, the kernel restores the best plan state. The returned plan may be from an earlier iteration than the exit iteration.

**Oscillation detection.** The kernel also maintains a sliding window of the last 4 hot regions targeted. If the same region appears more than once in the window without being resolved, that is detected as oscillation and the kernel exits with the best plan seen and a "FIF oscillation detected" soft flag. Oscillation detection is a safety net under the N-beam rule — the N-beam rule is the primary exit, oscillation detection catches pathological cases that the N-beam rule might not resolve cleanly.

### 15.6 Convergence parameters

| Parameter | Default | Notes |
|---|---|---|
| ΔDmax acceptance threshold | 0.3% per subfield | Primary rule — subfields producing less than this are undone |
| N-beam-in-a-row failure count | = number of eligible static beams | Primary exit — FIF exits as converged after N consecutive undo operations |
| Calibration goal (global Dmax) | 106% of Rx | Not an exit criterion. Used for validation and calibration — if the recipe consistently exits with Dmax well above or below 106%, the iteration budget or 0.3% threshold may need adjustment. |
| Clinical acceptability ceiling (global Dmax) | 108% of Rx | Classification applied at FIF exit; above this, soft flag raised. Not an exit criterion. |
| Cord rule threshold | 105% of Rx (on `SpinalCord.Dmax`) | Hard acceptance test at exit |
| Max iterations | 12 | Safety ceiling; not expected to be reached under normal plateau behavior |
| Max total subfields per plan | 10 | Safety ceiling |
| Min MU per control point on a new subfield | 3-4 MU | Configurable; subfields below this are rejected |
| Oscillation detection window | 4 most recent hot regions | Secondary exit condition — safety net |

**FIF does not stop because Dmax reaches a target. FIF stops because Dmax is no longer improving.**

Classification at exit:
- **Dmax ≤ 106%** — below calibration goal, excellent result
- **Dmax 106% - 108%** — within clinical acceptability ceiling, normal result
- **Dmax 108% - 115%** — above ceiling, soft flag for review
- **Dmax > 115%** — hard gate failure (§19.1)

All of these are **classifications applied after FIF has exited via plateau, N-beam rule, oscillation detection, or max-iteration safety ceiling**. None of them are exit criteria.

### 15.7 Cold spot prevention as a soft constraint

If a proposed subfield would drop PTV V100 below the configured threshold (default 95%), the `ColdSpotGuard` rejects the subfield and the kernel tries up to 3 alternatives within the same iteration (smaller subfield, different region boundary, different beam). If all 3 are rejected, the iteration exits with a "FIF limited by cold spot constraint" soft flag and the best plan seen is returned.

### 15.8 Post-normalization re-check and recovery

After normalization, the recipe re-runs acceptance checks:

- **If post-normalization global Dmax exceeds 108%:** up to 2 additional FIF iterations targeting the new Dmax, then exit (soft flag if still above).
- **If post-normalization cord Dmax ≥ 105%:** escalate per §15.11 (re-entering FIF would chase the same coupled problem).
- **If post-normalization V100 drops below threshold:** soft flag, plan returned for review.

### 15.9 Open implementation questions — deferred to Phase 4

**The strategy document deliberately does not specify FIF algorithmic details beyond the architectural posture established here.** FIF tuning and edge case handling are implementation decisions that will be made with empirical data in Phase 4, against real cases, supported by a dedicated sandbox application (§15.12).

Items explicitly deferred to Phase 4:

- Tuning the N-beam-in-a-row rule for edge cases (what if the dominant contributor is tied between two beams; what if the ordering changes mid-iteration)
- Exact behavior when a subfield would be smaller than the MU-per-control-point floor and no alternative exists
- Interaction between post-normalization re-FIF and the N-beam rule
- What happens when the recipe encounters a plan that has been manually pre-FIF'd by the dosimetrist (probably: refuse with a clean error, detect existing subfields during validation)
- Cold spot retry variant selection order
- `ConvergencePolicy` threshold calibration against diverse anatomies
- `BeamContributionEstimator` attribution edge cases

These are implementation questions, not strategy questions. The strategy document commits to the posture (plateau-driven, best-plan-tracked, undo-and-retry with 0.3% threshold, N-beam-in-a-row termination) and the kernel structure (kernel + six named policies). Specifics are deferred to Phase 4 implementation and sandbox iteration.

**This deferral is deliberate.** Attempting to specify every FIF edge case in the strategy document would either lock in decisions that should be made with data, or bloat the document into unusability. The Phase 4 sandbox is the correct venue for this work.

### 15.10 Why this primitive matters more than anything else

Every site uses FIF. The exact policies differ, but the kernel is invariant. Invest in this primitive disproportionately. T-spine is the right place to build it because the cord rule forces FIF to be non-trivial from day one, and the failure modes are visible and testable.

### 15.11 Cord rule failure handling and escalation recommendations

When FIF exits with `SpinalCord.Dmax ≥ 105%` and the pacemaker override is **not** active, the recipe surfaces a failure with an explicit escalation recommendation. The recipe does not autonomously escalate; the dosimetrist re-runs Stage 1 with the recommended changes.

The recipe **will try all techniques at its disposal** via dosimetrist-driven re-runs before recommending manual planning. There is no hard cap on escalation attempts at the recipe level — the ladder has a natural termination at the hybrid rung, and if the hybrid fails, manual planning is the recommendation.

Recommendation ladder:

```
IF current technique is 2-field:
    IF DCA-dominated hot spot was detected during FIF (§15.4.1):
        RECOMMEND escalate to 3-field (removes DCA entirely)
    ELSE IF higher energy is available AND not at the pacemaker cap:
        RECOMMEND escalate PA energy to next level
    ELSE:
        RECOMMEND escalate to 3-field

ELSE IF current technique is 3-field:
    IF higher energy is available:
        RECOMMEND escalate dominant-contributor beam(s) to next level (typically 15X)
    ELSE:
        RECOMMEND escalate to 4-field

ELSE IF current technique is 4-field:
    IF energy is not yet at 15X:
        RECOMMEND escalate AP and PA to 15X (obliques as clinically appropriate)
    ELSE IF hybrid is enabled in clinic config:
        RECOMMEND escalate to hybrid 3-field posterior + anterior DCA at 15X
    ELSE:
        RECOMMEND escalate to manual planning

ELSE IF current technique is hybrid:
    RECOMMEND escalate to manual planning (final automated rung exhausted)
```

**Rationale for the ladder:**
- 2-field with DCA-dominated failure → 3-field (not energy escalation) because escalating energy on the DCA doesn't help; the DCA is disabled for subfield synthesis.
- Energy escalation is attempted before technique escalation at each level because it's less disruptive to the plan structure.
- 15X is the typical "highest energy" for T-spine; the recipe escalates straight to 15X from 10X when escalating energy.
- The hybrid technique is the final automated rung because it combines the benefits of 3-field posterior coverage with anterior DCA fill, at the cost of being overkill for most cases. It is clinic-configurable (may be disabled entirely, in which case 4-field at 15X is the final rung).

The recommendation is presented to the dosimetrist along with the failed plan. The dosimetrist can:

- Accept the failed plan as-is (marginal case, clinical judgment) — manual override, recorded in decision log
- Re-run Stage 1 with the recommended escalation
- Discard the plan and plan manually

Every escalation attempt is a fresh Stage 1 → Stage 2 run. The recipe tracks escalation history in the decision log so the dosimetrist can see the full sequence of attempts.

When pacemaker override is active and the cord rule fails, §8.5 applies — the failure is logged as a known-limitation soft flag, not a failure, and no escalation is recommended.

### 15.12 FIF sandbox as a supporting artifact

A dedicated sandbox application will be built against `Pscripts.Planning` to support Phase 4 FIF development. Rationale:

- FIF behavior is the most iteration-heavy primitive in the framework
- Running experimental FIF code inside ProtocolPlanner risks destabilizing production code
- The sandbox allows rapid iteration against representative test cases without the full recipe overhead
- The sandbox shares the kernel and policies with the production recipe — what differs is the test harness

The sandbox is not an alternative recipe. It is a development tool. The FIF kernel, policies, and `BestPlanTracker` are production code in `Pscripts.Planning`; the sandbox is a thin wrapper that exercises them against a minimal test plan. Tuned behavior is integrated into the main recipe after sandbox validation.

See §26 for when the sandbox is built in the implementation sequence.

---

## 16. Normalization

Normalize the plan so that the prescription dose covers a defined fraction of the target volume. The normalization mode is **selectable** per case via the prescription input:

- **V100 = 95% of PTV** (default) — 95% of the PTV receives 100% of the prescription
- **V100 = 97% of PTV** — tighter coverage
- **D95% = Rx** — alternative phrasing of approximately the same coverage rule
- **PTV mean = Rx** — used for some palliative cases
- **Isocenter dose = Rx** — legacy mode

V100 = 95% is the default and matches clinic practice.

Normalization happens **after** FIF and **before** the post-normalization acceptance re-check (§15.8).

---

## 17. (Section removed in v0.2)

Held the v0.1 re-irradiation logic. Removed in v0.2 per §3.3.

---

## 18. OAR Considerations by Level

OAR handling for T-spine is intentionally "soft sanity bound only." The recipe reports OAR doses and flags when they exceed configurable thresholds, but does not modify the plan to reduce OAR dose. The cord is the exception — the cord rule is a first-class acceptance test.

### 18.1 OARs by T-spine region

- **Upper (T1-T4):** brachial plexus, lung apices, esophagus, cervical cord (above target)
- **Mid (T5-T9):** lungs, heart, esophagus
- **Lower (T10-T12):** lungs (lower lobes), kidneys, liver, stomach

### 18.2 How the recipe uses level information

The level estimator (§7.6) identifies which category the target falls into using OAR anatomical landmarks, and the recipe applies the appropriate OAR threshold set. Multi-level targets spanning categories use the union of relevant OARs.

### 18.3 What "soft sanity bound" means

The recipe reports OAR doses, compares them to thresholds, and flags exceedances. The recipe does not modify beam weights, apertures, or optimization to spare OARs. The dosimetrist reviews flagged plans.

### 18.4 Exception: the cord rule

The cord rule (§3.2, §15.2) is **not** a soft sanity bound. It is a first-class acceptance test on `SpinalCord.Dmax`. The pacemaker override (§8.5) is the only documented case where the cord rule may be deliberately deviated from.

---

## 19. Acceptance Gates and Three-State Result

Three states: **Complete**, **Complete with review needed**, **Aborted with reason**.

### 19.1 Hard gates (canonical refusal list)

§19.1 is the canonical list. Any other section that mentions refusal references this list.

- Required structures missing, geometrically invalid, or fail contract
- `SpinalCord` structure is a PRV or expanded volume (the cord rule requires the true cord)
- Target non-contiguous (refers patient to manual multi-plan workflow per §10)
- Target spans more than 4 contiguous levels
- Target crosses the cervicothoracic or thoracolumbar junction (detected via structure contract, not level identification)
- Required machine capability unavailable
- Spinal hardware in or near target, flagged upstream
- Image quality insufficient for reliable dose calculation
- Dose grid resolution coarser than the recipe minimum (§22.8)
- Patient separation out of expected range
- Dose calculation fails or returns nonphysical result
- Post-normalization coverage below hard floor (V100 < 90% of PTV)
- Post-normalization global max above absolute ceiling (> 115% of Rx)
- **Cord rule failure when pacemaker override is not active AND the full escalation ladder is exhausted** (4-field 15X failed AND hybrid failed or is disabled)
- Physician requests a technique the recipe does not support
- HBB selection is invalid (same-axis combination attempted)

### 19.2 Soft gates (review classifications applied at FIF exit)

**FIF outcome classifications** (none of these are exit criteria — exit was triggered by plateau, N-beam rule, oscillation, budget, or cold-spot exhaustion):

- **FIF exited with global Dmax ≤ 106%** — below calibration goal, no flag (or informational only)
- **FIF exited with global Dmax 106% - 108%** — within clinical ceiling, awareness flag
- **FIF exited with global Dmax 108% - 115%** — above ceiling, review flag, plan still returned
- **FIF oscillation detected** (best plan returned)
- **FIF exited via N-beam rule plateau** (normal informational flag — this is how FIF is expected to exit in most cases)
- **FIF limited by cold spot constraint**
- **FIF subfield count at or near maximum**

**Other soft gates:**

- OAR sanity bound exceeded for any in-field OAR (per §18)
- Single-fraction prescription (8 Gy × 1) — always flag for explicit review
- Dosimetrist override of algorithmic technique proposal in Stage 1 (recorded)
- Multi-level target spanning OAR category boundaries
- Effective depth unavailable (geometric fallback used) for lung-path beams
- Lung-path correction heuristic applied (energy biased upward)
- Pacemaker energy override active (always flag with "known limitation" notation)
- Cord rule failure when pacemaker override **is** active (logged as known limitation)
- Post-normalization re-check required additional FIF iterations
- HBB active (informational flag)
- Escalation history present (plan generated after one or more prior escalation attempts)

### 19.3 Why the three-state return is non-negotiable

Two-state loses the most common real-world case. Three-state is the credibility lever.

---

## 20. Decision Log and Reasoning Trace

### 20.1 What gets logged

Every automated decision, every input, every gate evaluation, every FIF iteration, every technique confirmation, every escalation recommendation, every subfield accepted or undone, every temporary structure created or cleaned up. Structured JSON plus human-readable prose.

### 20.2 Config and version provenance

Every decision log entry records:
- Recipe version
- Clinic configuration version (hash + tag)
- Machine configuration
- Library versions (`Pscripts.Planning`, etc.)
- Structure set version / timestamp
- Physician prescription, fractionation, normalization mode
- Pacemaker flag state
- HBB selection

**Configuration retention:** All retired clinic configuration versions are preserved indefinitely in source control. The recipe refuses to start if the active configuration is not in the canonical config repo.

**Decision log persistence:** JSON decision log is attached to the plan as an Aria document attachment.

### 20.3 Format

Structured JSON attached to the plan, human-readable summary in the report (via TreatmentPlanReport), `[CATEGORY]` prefix on each log line for greppability.

### 20.4 Example log entries

```
[RECIPE] T-SpineRecipe v1.0.0, clinic config v2026.04.14-b7e2a1
[STAGE] Stage 1 — Proposal
[INPUTS] Prescription 30 Gy in 10 fx, V100 = 95% normalization
[INPUTS] Machine TrueBeam_1, available energies [6X, 10X, 15X]
[INPUTS] Pacemaker flag: No
[INPUTS] HBB: None
[STRUCTURES] Contract validated: PTV, CTV, SpinalCord (true cord), External, Lung_L, Lung_R present
[STRUCTURES] Geometric sanity: all checks passed; GTV ⊆ CTV ⊆ PTV confirmed
[STRUCTURES] SpinalCord identified as true cord (not PRV)
[LEVEL] Estimator anchors: Heart top at z=8.2, Carina at z=5.1
[LEVEL] PTV centroid at z=10.3 → classified as mid T-spine (T7-T8 approximate)
[PROPOSAL] Posterior-to-target depth 14 cm, anterior-to-target depth 18 cm
[PROPOSAL] Proposed technique: 4-field
[PROPOSAL] Proposed energies: AP 10X, PA 10X, LPO 10X, RPO 10X
[PROPOSAL] Proposed normalization: V100 = 95%
[CONFIRMATION] Dosimetrist confirmed 4-field, energies as proposed, normalization as proposed, HBB none
[STAGE] Stage 2 — Execution begins
[ISO] Placed at (0.0, -2.4, 10.3) = target geometric center
[ISO] Sanity check: inside body contour ✓
[BEAMS] Built 4 beams per arrangement, collimator 0° (default, no HBB)
[MLC] Fit to PTV + 8 mm PTV block margin (no HBB active for this case)
[WEIGHTS] Default 4-field: AP 25%, PA 35%, LPO 20%, RPO 20%
[WEIGHTS] AP/PA analytic adjustment: ratio PA/AP = 1.42 (μ=0.028/cm at 10X, Δd=12.6 cm)
[WEIGHTS] Applied: AP 24.8%, PA 35.2%, LPO 20%, RPO 20%
[DOSE] Initial calculation complete. Global Dmax 109.1%, SpinalCord.Dmax 104.2%
[FIF] N-beam rule: N=4 (eligible: AP, PA, LPO, RPO)
[FIF] Iteration 1: hot region at PA entrance (Dmax 109.1%), dominant contributor PA
[FIF] Iteration 1: temp structure created, subfield synthesized on PA, field weight adjusted
[FIF] Iteration 1: recalc → Dmax 106.8%, ΔDmax 2.3% accepted. Temp structure cleaned up.
[FIF] Iteration 2: hot region at PA entrance (Dmax 106.8%), dominant contributor PA
[FIF] Iteration 2: subfield on PA, ΔDmax 0.9% accepted
[FIF] Iteration 3: hot region at AP/PA central axis (Dmax 105.9%), dominant contributor AP
[FIF] Iteration 3: subfield on AP, ΔDmax 0.7% accepted
[FIF] Iteration 4: hot region at AP entrance (Dmax 105.2%), dominant contributor AP
[FIF] Iteration 4: subfield on AP, ΔDmax 0.3% accepted (at threshold)
[FIF] Iteration 5: hot region at PA entrance (Dmax 104.9%), dominant contributor PA
[FIF] Iteration 5: subfield on PA, ΔDmax 0.1% — BELOW THRESHOLD, undoing
[FIF] Failure counter: 1/4
[FIF] Iteration 6: same hot region, next-ranked beam AP, subfield on AP, ΔDmax 0.0% — undoing
[FIF] Failure counter: 2/4
[FIF] Iteration 7: same hot region, next-ranked beam LPO, subfield on LPO, ΔDmax 0.0% — undoing
[FIF] Failure counter: 3/4
[FIF] Iteration 8: same hot region, next-ranked beam RPO, subfield on RPO, ΔDmax 0.0% — undoing
[FIF] Failure counter: 4/4 — N-beam rule exit
[FIF] Plateau reached. Exiting as converged. Best plan: iteration 4, Dmax 104.9%
[FIF] Best plan tracker: returned iteration 4 state
[FIF] Cold spot check: PTV V100 = 96.4%, acceptable
[NORM] V100 = 95% applied
[POST_NORM] Re-check: global Dmax 105.1%, cord Dmax 103.6%, V100 = 95% ✓
[ACCEPTANCE] Cord rule: SpinalCord.Dmax 103.6% < 105% ✓
[OAR] Lung mean 4.2 Gy, V20 12%, below thresholds
[OAR] Esophagus max 28.4 Gy, below 30 Gy threshold
[OAR] Heart mean 1.8 Gy, below threshold
[GATES] All hard gates passed. FIF exited at Dmax 105.1% (below 106% calibration goal, within clinical ceiling). Informational flag only.
[RESULT] Complete with review needed
```

---

## 21. Reusable Primitives (V1 Implementation Status)

| Primitive | V1 status | Notes |
|---|---|---|
| `BeamArrangementBuilder` | Build | Returns beam intents with clinical role and HBB metadata |
| `BeamIntent` data structure | Build | Includes role, delivery mode, subfield policy, HBB jaw/collimator constraints |
| `DynamicConformalArcBuilder` | Build | Anterior arc for T-spine; wraps Eclipse native control point generation |
| `EnergySelector` | Build | Effective depth preferred, geometric fallback with lung-path bias, pacemaker 6X cap |
| `BeamWeightSolver` | Build | Per-arrangement defaults + analytic AP/PA adjustment |
| `BeamRoleContext` | Build | Semantic context object passed between primitives |
| `MLCFitter` | Build | BEV + PTV block margin; supports static and DCA beams. HBB is transparent to the fitter — when HBB is active, the fitter is simply re-invoked from the HBB-offset isocenter (§13) |
| **`IsocenterHelper`** | **Extend** | **Existing in `Pscripts.Planning`; V1 extends with HBB support (§11.7). Cross-cutting primitive reused by every 3D site.** |
| `FieldInFieldGenerator` (kernel) | Build | Orchestrates policies; highest-leverage primitive |
| `HotspotLocator` (policy) | Build | Single hot region per call; creates and manages temporary isodose structures |
| `BeamContributionEstimator` (policy) | Build | Static beams: standard ray-cast; DCA: total arc field weight (simplified, not per-CP integration) |
| `SubfieldSynthesisPolicy` (2 policies) | Build | Static-beam, DCA (no-op with route-and-undo handling) |
| `ColdSpotGuard` (policy) | Build | Soft-gate cold spot check with bounded retry |
| `ConvergencePolicy` (policy) | Build | 0.3% threshold, N-beam-in-a-row counter, oscillation sliding window, max-iteration ceiling |
| `BestPlanTracker` (policy) | Build | Maintains best-plan-seen state across FIF iterations; returns best plan on any exit |
| `AcceptanceEvaluator` | Build | Three-state result with hard and soft gates |
| `EscalationRecommender` | Build | Generates technique/energy escalation recommendations per §15.11 ladder |
| `DecisionLogger` | Build | Structured + human-readable with version provenance, HBB state, pacemaker state |
| `NormalizationService` | Build | Selectable mode (V100 95% default); triggers post-norm re-check |
| `SetupFieldGenerator` | Build | kV pair, CBCT |
| `MachineCapabilityQuery` | Build | Energies, MLC, jaws, MU limits |
| `PrescriptionHandler` | Build | Dose, fractionation, normalization mode, pacemaker flag, HBB selection |
| `StructureContractValidator` | Build | Metadata + geometric sanity + cord-vs-PRV detection + containment hierarchy |
| `LevelEstimator` | Build | Anatomy-driven (OAR landmarks + PTV centroid), not vertebra-driven |
| `CouchModelInserter` | Build | Inserts couch structure if absent |
| `ReportEmitter` | Build | TreatmentPlanReport integration |
| `RadiologicalPathLengthProvider` | Investigate | Wraps ESAPI API if available; geometric fallback with lung bias |
| `PlanLifecycleManager` | Build | Plan ownership, cleanup on failure (always-remove); tracks recipe-created temporary structures for cleanup |

**Changes from v0.3:**
- `IsocenterHelper` extended for HBB (significant new capability, cross-cutting)
- `BeamContributionEstimator` simplified to total-arc-MU for DCA (dropped "non-trivial" framing)
- `HotspotLocator` absorbs temporary structure lifecycle responsibility

**Clarified in v0.4.1:**
- `MLCFitter` does **not** need HBB awareness as a parameter. HBB is handled entirely by isocenter placement (`IsocenterHelper.PlaceForHBB`), and the MLC fitter is simply re-invoked from the HBB-offset isocenter using the standard PTV block margin. A post-fit jaw sanity check on HBB beams confirms the blocked-side jaw is at 0 cm.

**Removed from v0.3:**
- `StageOrchestrator` — not needed; single-process execution means no handoff primitive is required

### 21.1 What T-spine contributes that other sites inherit

- The technique-family abstraction pattern proved in production
- The FIF kernel + policies architecture with best-plan tracking and plateau-based exit
- **`IsocenterHelper` with HBB support — cross-cutting primitive used by every 3D site**
- The HBB selection pattern (zero, one, or two complementary axes)
- The `BeamRoleContext` and beam intent pattern
- The analytic AP/PA depth-compensated adjustment
- The energy selector with lung-path bias and pacemaker override
- The per-arrangement default weight table pattern
- The three-state acceptance result in production
- The decision log format with version provenance and HBB tracking
- The structure contract pattern including cord-vs-PRV detection
- The DCA / arc primitive with simplified contribution modeling
- The cord rule pattern (Dmax-based acceptance test via DVH query)
- The escalation recommendation pattern
- The anatomy-driven level estimation pattern
- The plan lifecycle management primitive with temporary structure tracking

### 21.2 What T-spine defers to future sites

- `PoseEstimator` — not needed for T-spine
- Optional anatomy-driven aperture modifications (e.g., lens pullback in WBRT)
- Additional arc geometries
- Other clinical override scenarios beyond pacemaker
- Pacemaker contour geometric usage (future enhancement, not V1)

---

## 22. Execution Model and Recipe Lifecycle

### 22.1 Single-process two-stage execution

The recipe runs as a **standalone executable** built on `Pscripts.Planning`. It is launched by the user, opens the patient, acquires its own ESAPI session, runs both stages in the same process, and exits when done. There is no plugin-to-standalone handoff, no inter-process communication, no serialization of state between stages. The plugin-launcher-to-standalone-executable pattern is already solved in the `Pscripts` ecosystem and this recipe reuses it.

**Stage 1 — Proposal (synchronous, fast)**
- Runtime: 20-30 seconds typical (level estimation, depth measurement, technique proposal, structure contract validation)
- UI: blocking — the dosimetrist is at the workstation
- Output: proposal presented via UI; dosimetrist confirms, modifies, or cancels

**Stage 2 — Execution (runs in same process, UI non-blocking)**
- Runtime: 5-10 minutes typical (dose calculation + FIF iterations + normalization + post-norm re-check)
- UI: non-blocking — dosimetrist can minimize the window and do other work
- Same ESAPI session, same patient context, same in-memory state
- Exit: complete, complete-with-review, or aborted-with-reason

The "two-stage" split is about user experience (fast confirmation followed by longer background execution), not about process or session boundaries. All state lives in one executable from start to finish.

### 22.2 Plan ownership and idempotency

- The recipe creates a new plan in Stage 2, named per clinic convention, marked as generated by ProtocolPlanner 3D Automation
- Re-running the recipe creates a new plan with an incremented suffix
- "Same inputs" for idempotency purposes means same patient, structure set version, prescription, machine, clinic config version, and Stage 1 confirmation choices (including HBB and pacemaker flag)

### 22.3 Partial failure and cleanup

If Stage 2 fails mid-execution, the `PlanLifecycleManager` **always removes the partial plan entirely**, including any recipe-created temporary structures (tagged during FIF per §15.1). The decision log records the failure point and the cleanup actions.

If ESAPI does not support clean plan removal, the lifecycle manager implements manual reverse-order traversal of all recipe-created artifacts with error handling at each step. Temporary FIF structures are specifically tagged so they can be found and removed even if the recipe crashed mid-iteration.

### 22.4 Concurrency

The recipe is a standalone executable; two instances launched simultaneously against the same Eclipse installation will compete for ESAPI session access. Default V1 behavior: **detect and fail cleanly**. The second instance detects that an ESAPI session cannot be acquired (or that the patient is already in use) and exits with a clear error message. No queueing in V1.

### 22.5 External change detection

Because Stage 1 and Stage 2 run in the same process and the same ESAPI session, mid-execution patient changes from external sources are less likely than in a cross-process model. However, if the user or another process modifies patient data during Stage 2 (unlikely but possible), the recipe handles it with snapshot-and-recheck:

- At the start of Stage 2, the recipe snapshots the input structures (version timestamps, volumes, centroids)
- At dose calculation time, if any relevant structure has changed, Stage 2 aborts with "external change detected" and triggers cleanup

### 22.6 Configuration version drift

Per §20.2, all retired configuration versions are preserved in source control. When the active configuration version differs from the version used in retrospective validation, the difference is recorded in the decision log and surfaced as a soft flag on the plan.

### 22.7 What is explicitly not in V1

- Hot-swapping configurations during execution
- Resuming a partially-failed recipe run
- Cross-machine recipe execution
- Distributed locking across workstations
- Queueing concurrent recipe invocations

### 22.8 Dose grid resolution requirement

The recipe requires a dose grid resolution of **2.5 mm or finer** (2.0 mm preferred). Coarser grids are refused as a hard gate because the cord Dmax check loses precision at coarser resolutions. The validator reads the grid from the plan's calculation options at the start of Stage 2 (the plan is created in Stage 2, so this check happens there, not in Stage 1).

If the linac's default grid is coarser than 2.5 mm, the recipe overrides the calculation options before initial dose calculation. The override is logged.

---

## 23. Open Questions Summary

In rough order of importance:

1. **Appendix A.7 end-to-end FIF workflow demonstration** — gating the entire §15 design
2. **Clinic validation of technique selection depth thresholds**
3. **Clinic validation of energy selection depth-to-energy table**
4. **Clinic validation of default weight tables per arrangement**
5. **Hybrid technique default weights** — TBD, derive from retrospective data
6. **μ_eff values per energy** for analytic AP/PA adjustment
7. **OAR anchor SI coordinate tables** for the level estimator
8. **OAR sanity bound thresholds by level**
9. **FIF parameters for Phase 4 tuning** — 0.3% threshold may move to 0.5% with data; N-beam-in-a-row rule may need refinements
10. **DCA arc parameters** — exact start/stop angles, arc direction
11. **MLC margin defaults** — 5 vs 6 vs 7 mm
12. **Field length SI margin** — 5 vs 7 vs 10 mm
13. **PTV block margin default value** — clinic validation of the 8 mm strawman against historical 3D plans
14. **Plan naming convention**

---

## 24. Validation Strategy

### 24.1 Retrospective validation set

- 30 "easy" cases: standard single-level, no surgical history
- 20 "median" cases: typical mix including 2-level and technique variety
- 10 "hard" cases: 3-4 level targets, borderline patient sizes
- 10 "exclusion" cases: cases the recipe should refuse
- 5 "pacemaker" cases: pacemaker override active, known-limitation validation
- 5 "HBB" cases: cases where HBB was used clinically, testing recipe HBB behavior
- 5 "hybrid" cases: cases where 4-field at maximum energy was insufficient and the hybrid technique was used (or would have been used had it been available)

### 24.2 Metrics

- Technique proposal acceptance rate (target ≥80%)
- Disagreement analysis
- Edit distance after recipe output
- Wall-clock time (Stage 1 < 30s, Stage 2 < 10 min)
- Plan quality: D95%, global Dmax, `SpinalCord.Dmax`, homogeneity, OAR doses
- Refusal correctness
- Soft gate calibration
- **FIF plateau behavior:** how often FIF exits via N-beam rule vs oscillation vs max iterations; average iteration count; average Dmax at exit; fraction of cases where best-plan tracker returns a non-final iteration
- **Escalation frequency:** fraction of cases requiring escalation; fraction reaching each rung
- **HBB cases:** isocenter placement correctness; jaw constraint correctness; MLC fit correctness
- **Pacemaker cases:** energy cap compliance; known-limitation flag correctness

### 24.3 Pre-clinical gates

1. Phantom validation
2. FIF sandbox validation of kernel behavior on representative test cases
3. Retrospective replanning with blinded review
4. Parallel planning with manual for first ≥10 clinical cases
5. Limited rollout to one dosimetrist before broader release

### 24.4 Regression testing post-V1

Every retrospective case becomes a regression test.

---

## 25. What T-spine Teaches Us About the Framework

1. **Technique-family abstraction is load-bearing on day one.**
2. **The energy selector and beam weight solver are interfaces, not implementations.**
3. **Machine awareness must be enforced at the recipe contract level.**
4. **The structure contract is the integration point that determines whether the recipe even starts.** Since structure generation is upstream of the recipe (physician + Limbus), the contract is about inspection and validation, not generation.
5. **Safety boundaries go in the clinical eligibility envelope** (§3).
6. **The boring last mile is V1 scope.**
7. **The decision log is the product**, not a debug feature.
8. **Refuse-and-explain is pervasive.**
9. **OAR handling without a constraint engine works for palliative sites.** The cord rule is the exception.
10. **DCA support in V1 is a strategic bet** behind a feature gate.
11. **Validation requires retrospective data analysis to set defaults.**
12. **The recipe is a composition of primitives.** Primitives use kernel + policies where site- or technique-specific behavior varies.
13. **Execution model matters as much as planning model.** The single-process two-stage model is the right answer for standalone-executable recipes.
14. **Multi-minute runtime in Stage 2 is acceptable if the output is final.**
15. **The cord rule is the dosimetric driver for T-spine**, checked as an acceptance test via standard DVH query.
16. **FIF is the only dose uniformity tool.** No wedges, no compensators.
17. **FIF runs to plateau, not to a target.** The 0.3% threshold and N-beam-in-a-row rule are the termination primitives.
18. **Clinical overrides (pacemaker) are first-class.**
19. **Escalation on failure is dosimetrist-driven, not autonomous.**
20. **Cross-cutting primitives like `IsocenterHelper` (with HBB) are built once in `Pscripts.Planning` and reused by every site.** T-spine is the first exerciser; WBRT, pelvis, and future sites inherit.
21. **FIF algorithmic details are deliberately deferred to implementation and sandbox iteration.** The strategy document commits to architecture and posture, not to every edge case.
22. **The level estimator is anatomy-driven, not vertebra-driven.** OAR landmarks are the anchors.

---

## 26. V1 Implementation Scope and Sequence

### Phase 0 — ESAPI investigation and document freeze

**A.7 (end-to-end FIF workflow demonstration) should be executed before Phase 1 begins**, not just before Phase 4, because the §15.1 workflow affects Phase 1 primitive contract design. If A.7 reveals the workflow is fictional, the Phase 1 foundational primitives need rework.

### Phase 1 — Foundational primitives

- `MachineCapabilityQuery`
- `PrescriptionHandler` (with pacemaker flag, HBB selection, normalization mode)
- `StructureContractValidator`
- `DecisionLogger`
- `BEVProjector`
- `RadiologicalPathLengthProvider`
- `EnergySelector`
- `BeamRoleContext`
- **`IsocenterHelper` HBB extension** (extend existing helper in `Pscripts.Planning`)

### Phase 2 — Beam construction

- `IsocenterPlacer` (consumes `IsocenterHelper`)
- `BeamArrangementBuilder` (including hybrid technique)
- `MLCFitter` (HBB-transparent; refit from HBB-offset isocenter per §11.5)
- `BeamWeightSolver`
- `LevelEstimator` (anatomy-driven)
- `CouchModelInserter`

### Phase 3 — DCA support

- `DynamicConformalArcBuilder`
- DCA-specific MLC fitting with control points
- DCA feature gate handling

### Phase 4 — Field-in-field (kernel + policies)

**Longest phase, disproportionate investment.** Runs in parallel with sandbox development.

- FIF kernel
- `HotspotLocator` (with temporary structure lifecycle)
- `BeamContributionEstimator`
- Subfield synthesis policies
- `ColdSpotGuard`
- `ConvergencePolicy` with 0.3% threshold and N-beam rule
- `BestPlanTracker`
- ESAPI field weight adjustment via `GetEditableParameters`
- FIF logging integration

**FIF sandbox application** is built in parallel against `Pscripts.Planning`. The sandbox shares the kernel and policies with the production recipe and allows iteration against representative test cases without bogging down ProtocolPlanner. Tuned kernel behavior is integrated into the main recipe after sandbox validation.

### Phase 5 — Acceptance, normalization, reporting

- `AcceptanceEvaluator`
- `EscalationRecommender` (including hybrid escalation rung)
- `NormalizationService` (selectable mode)
- OAR reporting
- `SetupFieldGenerator`
- `ReportEmitter`

### Phase 6 — Recipe assembly and lifecycle management

- `TSpineRecipe` composition layer (including hybrid)
- Stage 1 / Stage 2 in-process split
- Technique proposal algorithm
- Dosimetrist confirmation UI integration
- Clinic configuration loader
- `PlanLifecycleManager` (with temporary structure cleanup)
- End-to-end pipeline tests

### Phase 7 — Validation and rollout

- Retrospective validation per §24
- Phantom validation
- FIF sandbox comparison studies
- Pre-clinical gates
- Limited rollout
- Broader release

### 26.1 What is not in this sequence

- Dosimetrist confirmation UI (parallel workstream)
- Integration with ProtocolPlanner or separate launcher (product decision)
- WBRT, pelvis, or any other site
- VMAT/IMRT/SBRT — categorically excluded
- Wedges or compensators — categorically excluded

---

## 27. Changelog from v0.1 to v0.2

(Preserved — see §28 for v0.2 → v0.3 and §29 for v0.3 → v0.4.)

---

## 28. Changelog from v0.2 to v0.3

(Preserved from v0.3 — see the v0.3 document for the full entry.)

---

## 29. Changelog from v0.3 to v0.4

This changelog documents the targeted patch applied after the third adversarial review round and the fourth clinical clarification pass.

### CC:Gen decoupling

Removed CC:Gen as a named dependency throughout the document:
- **§6.1** structure source column: "Physician + CC:Gen" → "Physician"; "CC:Gen / standard" → "Standard autocontour"
- **§6.3** calibration posture: rewritten to refer to "physician and Limbus output" instead of CC:Gen
- **§6.2** optional structures: per-vertebra row removed; pacemaker contour row added as future-enhancement
- **§7.6** level estimator: rewritten as anatomy-driven using OAR landmarks (carina, heart, esophagus, stomach, diaphragm, kidney) instead of vertebra-driven; PTV centroid + OAR anchors model
- **§10** field length: per-vertebra language removed
- **§25** framework lesson 4: reframed as "the structure contract is the integration point" without CC:Gen
- **§23** open question 12 (per-vertebra contouring template): removed — answered as "not used"
- **§15.1** FIF workflow: added explicit temporary structure lifecycle (step 1.5 create, step 7 cleanup) as recipe-internal ESAPI operations, not CC:Gen integration
- **§3.6** added new "No CC:Gen dependency" clinical boundary subsection
- **§21** primitives table: `HotspotLocator` absorbs temporary structure lifecycle responsibility

### Plateau-based FIF convergence

Removed threshold-based exit framing throughout §15:
- **§15.0** preamble: added explicit "FIF runs to plateau, not to a target Dmax" statement
- **§15.5** rewritten around the N-beam-in-a-row rule with the 0.3% acceptance threshold as the primary convergence mechanism. The rule: subfields with <0.3% ΔDmax are undone; after N consecutive undos (where N = number of eligible static beams), FIF exits as converged. Counter resets on any successful subfield.
- **§15.6** convergence parameters table restructured: removed "FIF stopping target = 105%" row entirely; added "calibration goal 106%" row (not an exit criterion); restructured classification buckets; added explicit note "FIF does not stop because Dmax reaches a target. FIF stops because Dmax is no longer improving."
- **§19.2** soft gates rewritten as FIF outcome classifications applied at exit, not as targeting outcomes
- **§20.4** example log rewritten to show the N-beam-in-a-row plateau exit in action (iteration 5 with four consecutive undo operations)
- **§25** framework lesson 17 added: "FIF runs to plateau, not to a target."

### Single-process execution model clarification

Rewrote §22.1 to reflect the standalone executable reality:
- The recipe is a single process that opens the patient, acquires one ESAPI session, runs both stages, and exits
- There is no inter-stage handoff, no session re-acquisition, no serialization boundary
- The "two-stage" split is a user experience pattern (fast confirmation → longer background execution), not a process model
- `StageOrchestrator` primitive removed from §21 — not needed since there is no handoff
- The plugin-launcher-to-standalone-executable pattern is acknowledged as already solved in `Pscripts`
- §22.5 external change detection simplified because of single-process continuity
- §25 framework lesson 13 updated to reflect single-process correctness

### Half Beam Block (HBB) as a cross-cutting primitive

New §11 section introducing HBB as a dosimetrist-selectable isocenter modifier:
- **§11.2-11.4** describes the HBB mechanics: isocenter offset, jaw-at-zero clipping, collimator constraint (0° default, 90° rare alternative), DCA interaction
- **§11.6** HBB is always dosimetrist-selected, never recipe-proposed
- **§11.7** `IsocenterHelper` API extended with `PlaceForHBB` method
- **§11.2** zero, one, or two axes supported (six single-axis options + twelve two-axis combinations)
- **§11.2** complementarity constraint: two-axis HBB requires different axes; same-axis combinations rejected by validator
- **§4** HBB added to in-scope list
- **§6.4** HBB selection added to Stage 1 user inputs
- **§7.2** HBB is not recipe-proposed
- **§12.1** beam intents carry HBB jaw and collimator constraints
- **§19.1** invalid HBB selection (same-axis) is a hard gate
- **§19.2** HBB active is an informational soft flag
- **§20** decision log records HBB selection
- **§21** `IsocenterHelper` marked as "Extend" (existing primitive, V1 extension)
- **§25** framework lesson 20 added: cross-cutting primitives like `IsocenterHelper` are built once and reused across sites
- **§26** Phase 1 includes `IsocenterHelper` HBB extension

### Hybrid technique as terminal escalation rung

New fourth technique added as the terminal automated rung in §15.11 escalation:
- **§5.1** table expanded to four techniques including hybrid
- **§5.3** technique differences generalized to four variants
- **§12.5** hybrid beam arrangement defined
- **§9.1** hybrid default weights marked as TBD, derived from retrospective data during validation
- **§15.11** escalation ladder extended with hybrid as the final rung before manual
- **§15.4.1** notes that hybrid is the most complex FIF configuration (three static beams + DCA) and stress-tests DCA route-and-undo
- **§24.1** hybrid validation cases added
- **§26** Phase 6 includes hybrid composition

### Explicit FIF implementation scope deferral

New §15.9 and §15.12 sections:
- **§15.9** explicitly defers FIF algorithmic details to Phase 4 implementation, listing specific items that are implementation decisions rather than strategy decisions
- **§15.12** introduces the FIF sandbox as a supporting artifact built in parallel with Phase 4
- **§26** Phase 4 mentions sandbox development explicitly
- **§25** framework lesson 21 added: FIF details deliberately deferred

### Other clarifications and commitments

- **§3.2** cord rule: added "via standard ESAPI DVH query" language acknowledging this as a routine operation, not a voxel-level calculation problem; added "strict inequality" to the < 105% rule
- **§3.5** "no wedges" subsection expanded to explicitly exclude electronic compensators
- **§8.5** pacemaker override: 10 MV default replaced with 6 MV hard cap; pacemaker type (pacemaker / ICD / CRT-D) explicitly treated the same; pacemaker contour acknowledged as future-enhancement for divergence-favorable isocenter placement
- **§14.2** `BeamContributionEstimator` DCA contribution simplified to total arc field weight; per-control-point integration explicitly out of scope
- **§9.2** composition rule clarified: "applied once before dose calculation" made explicit in text
- **§12.4** noted that all four beams in 4-field are added simultaneously before the analytic adjustment and before dose calculation
- **§15.11** escalation ladder rewritten to commit to the full technique escalation: 2-field → 3-field → 4-field → hybrid → manual, with no hard cap on escalation attempts (the ladder has a natural terminal condition)
- **§22.1** runtime targets committed: Stage 1 20-30 seconds, Stage 2 5-10 minutes
- **§22.8** dose grid check moved to Stage 2 (where the plan is created), noted explicitly
- **§16** V100 = 95% committed as the clinic default
- **§7.6** level estimator committed to anatomy-driven approach with OAR landmark anchors

### New primitives in v0.4

- `IsocenterHelper` HBB extension (`PlaceForHBB` method)
- `LevelEstimator` rewritten as anatomy-driven (not a new primitive, but a substantial rewrite)
- `HotspotLocator` absorbs temporary structure lifecycle

### Removed primitives from v0.3

- `StageOrchestrator` (not needed under single-process model)
- The "AP/PA pair" subfield synthesis policy stays withdrawn from v0.3; no change

### Preserved unchanged from v0.3

The following v0.3 decisions remain locked and unchanged:

- Three-state acceptance result
- Decision log as first-class product artifact
- Refuse-and-explain posture
- Boring last mile in V1
- Technique-family abstraction
- Categorical exclusion of VMAT/SBRT/multi-isocenter
- Categorical exclusion of re-treats
- PTV-authoritative position
- DCA in V1 behind feature gate
- Custom HU-to-density ray casting out of scope
- `Pscripts.Aria` not consumed by this recipe
- Kernel + policies FIF pattern
- `BestPlanTracker` as core kernel responsibility
- Post-normalization re-check with bounded re-FIF for global, escalation for cord

---

## 29.1 Patch Changelog: v0.4 → v0.4.1

v0.4 went through a freeze decision review round with three independent adversarial reviewers. All three returned "GO WITH MINOR PATCHES." v0.4.1 applies the convergent edit list from the three reviews plus additional clinical clarifications on the PTV block margin and HBB MLC handling.

### Propagation fixes (reviewer-flagged)

- **§2.2** cord-driven technique selection language softened. The cord rule is the dosimetric driver of evaluation and escalation, not a Stage 1 predictor. Removes the contradiction with §7.4.
- **§4** in-scope bullet for technique recommendation rephrased. Technique recommendation is depth-driven and config-driven; cord compliance is determined post-FIF.
- **§7.1** Stage 1 workflow step 1: removed "cord-to-PTV relationship" from measured inputs. It was dead code — no downstream primitive consumed it, and its presence contradicted §7.4.

### HBB clarifications (clinical input)

- **§6.4** inputs: added **PTV block margin** as a named user-configurable input, typically 8 mm for 3D plans. Surfaced in the Stage 1 UI.
- **§11.2** HBB mechanics: replaced the undefined "plus the PRV margin" phrase with "plus the PTV block margin," the same 8 mm margin used by the `MLCFitter` for BEV projection. There is no "PRV margin" concept in the recipe — the cord rule operates on the true cord (§3.2), not a PRV-expanded volume.
- **§11.4** jaw deviation edge case: removed the specific "1 cm instead of 0 cm" number; replaced with "small geometric deviations are accepted and logged; the deviation magnitude is recorded."
- **§11.5** MLC fitting interaction for HBB: completely rewritten. Previously described as "fit to full target then post-fit override blocked-side leaves." Now described as "refit from the HBB-offset isocenter" — the `MLCFitter` does not need HBB awareness, it is simply re-invoked from the new isocenter position and fits naturally. A post-fit jaw sanity check confirms the blocked-side jaw is at 0 cm.
- **§13** MLC fitting section aligned with the §11.5 refit-from-offset-isocenter framing. MLC fitter margin is named explicitly as the PTV block margin from §6.4.
- **§21** primitives table: `MLCFitter` row updated to reflect that HBB is transparent to the fitter. Added a "Clarified in v0.4.1" note explaining the change.

### Iteration terminology unification (reviewer-flagged)

- **§15.4.1** step 3: reworded to match the "one subfield per iteration" convention from §15.5. The undone subfield counts as a failed iteration; the kernel advances to the next iteration with the next-ranked beam.
- **§20.4** example log: relabeled "Iteration 5" / "Iteration 5 retry" × 3 as "Iteration 5" through "Iteration 8" with the failure counter incrementing across iterations. The N-beam rule exit now fires on iteration 8. This matches the semantic intent of §15.5's "one subfield per iteration" rule.

### Validation set (reviewer-flagged)

- **§24.1**: added "5 'hybrid' cases" bullet to the retrospective validation set list. The §29 changelog claimed this was added during the v0.3 → v0.4 revision, but §24.1 did not actually contain the bullet. Bringing body into alignment with the changelog.

### Phase 0 ESAPI investigation (reviewer-flagged)

- **Appendix A.4.3 (new):** Verify that beam-level jaw settings correctly propagate to all control points of a dynamic conformal arc when HBB is active. This is the Phase 0 question for the §11.4 HBB+DCA interaction claim, which was presented as fact in v0.4 without empirical verification. Bad-answer fallback documented: if propagation does not work, HBB is refused on 2-field PA+DCA and hybrid techniques (3-field and 4-field are unaffected because they do not use DCA).

### Dropped from the edit list

Three reviewer edits were reviewed and dropped because they did not reflect substantive issues:

- **§19.1 dose grid override clarification:** §22.8 already covers the override attempt, and the hard gate fires after override failure. Adding redundant language was unnecessary.
- **§15.6 max iterations "safety brake" relabeling:** the existing table row already says "Safety ceiling; not expected to be reached under normal plateau behavior." Adequate as-is.
- **§14.1 DCA control point spacing sandbox note:** ESAPI handles DCA control point generation and MLC motion limits internally; this is not a freeze-relevant concern.

### Reviewer-suggested edits that clinical input resolved differently

- **Appendix A.3.4 (programmatic leaf override):** The reviewer proposed a Phase 0 question about whether MLC leaves can be programmatically overridden after `FitMLCToStructure` returns. Clinical input clarified that the MLC handling for HBB is "refit from new isocenter," not "post-fit leaf override." The programmatic override question is therefore not needed because the recipe does not do programmatic override. The MLC fitter is simply re-invoked from the HBB-offset isocenter, which is a standard supported operation.

### Summary

v0.4.1 is a 14-edit surgical patch. No structural rework, no section rewrites, no new architecture. All reviewer concerns are addressed either by applying the edit or by documenting why the edit was not needed.

---

## 29.2 Patch Changelog: v0.4.1 → v0.4.2 (FREEZE)

v0.4.1 went through a freeze confirmation review (fifth adversarial round) with three independent reviewers. One returned a clean GO. Two returned GO WITH MINOR PATCHES with a combined 5-item non-overlapping edit list of stale cross-reference cleanup — all propagation misses from the v0.4.1 patch that should have caught them but didn't. v0.4.2 applies those 5 edits and freezes the document.

### Stale cross-reference cleanup

- **§5.2** (shared primitives list): "MLC fitting (BEV projection of target with margin, HBB-aware)" → "MLC fitting (BEV projection of target plus PTV block margin; HBB is handled via isocenter placement, not at the fitter level)"
- **§5.4** (pipeline pseudocode): `fit_mlc_to_each_beam(hbb)  # MLC fitter is HBB-aware` → `fit_mlc_to_each_beam()  # HBB is transparent to the fitter; refit from HBB-offset isocenter`
- **§20.4** (example decision log): `[MLC] Fit to PTV + 6 mm isotropic margin (no HBB clipping)` → `[MLC] Fit to PTV + 8 mm PTV block margin (no HBB active for this case)` — aligns with the 8 mm PTV block margin committed in §6.4
- **§23** (open questions list, item 13): "PRV margin for HBB offset calculation — clinic convention" → "PTV block margin default value — clinic validation of the 8 mm strawman against historical 3D plans" — the PRV margin question is now answered (there is no PRV margin concept; it is the PTV block margin)
- **§26** (Phase 2 primitives): `MLCFitter (with HBB awareness)` → `MLCFitter (HBB-transparent; refit from HBB-offset isocenter per §11.5)`
- **§29** (v0.3 → v0.4 changelog): struck two stale bullets that described the v0.4 post-fit override approach for HBB MLC handling. Specifically removed "§11.5 MLC fitting interaction: `MLCFitter` is HBB-aware via post-fit override" and "§13 MLC fitter takes HBB parameter." These described v0.4 behavior that v0.4.1 reversed, and preserving them as historical bullets was unnecessary because the recipe has not been used yet and there is no need to preserve a historical record of an approach that was never deployed.

### Nothing else changed

v0.4.2 contains zero new architecture, zero new primitives, zero new locked decisions, and zero new open questions. The document content is identical to v0.4.1 except for the 5 propagation cleanup edits above and this §29.2 changelog section. The Document History table is updated to reflect the v0.4.2 freeze.

### Freeze declaration

**v0.4.2 is the frozen T-Spine Automation Strategy document.** No further revisions are planned. The next artifacts in the project are the architectural planning document (which consumes v0.4.2 as input and specifies interface contracts for the primitives named in §21) and the Phase 0 ESAPI investigation (Appendix A, with A.7 as the gating pre-Phase-1 item). The FIF sandbox application begins development in parallel with architectural planning as described in §15.12 and §26 Phase 4.

---

## Appendix A: ESAPI Investigation Checklist

These are empirical questions that must be answered before Phase 1 design work begins. Results are documented in a scratch file and incorporated into the frozen strategy doc before Phase 1 starts.

### A.1 Dose calculation and plan modification

**A.1.1** Does modifying a beam's field weight via `Beam.GetEditableParameters()` require a full dose recalculation?
*Blocks:* FIF latency estimate
*Bad-answer fallback:* FIF iteration budget tightens; Stage 2 runtime extends

**A.1.2** Measured per-calculation time for AAA and Acuros on a representative T-spine plan
*Blocks:* FIF runtime expectations

**A.1.3** Does adding a subfield to a parent beam require full dose recalc?
*Blocks:* FIF loop architecture

### A.2 Radiological path length

**A.2.1** Does ESAPI expose a radiological (water-equivalent) path length method?
*Blocks:* Energy selector design
*Bad-answer fallback:* Geometric depth with lung-path bias

**A.2.2** If yes, what are the precision, coordinate system, and failure modes?

**A.2.3** If no, error magnitude of geometric depth for lung-path beams

### A.3 MLC and subfield APIs

**A.3.1** Does `Beam.FitMLCToStructure` support asymmetric margin per edge?
*Blocks:* MLC fitter, particularly HBB post-fit override

**A.3.2** API for creating a subfield on an existing beam
*Blocks:* Subfield synthesis policy

**A.3.3** ESAPI support for MLC leaf position smoothing

### A.4 DCA / arc APIs

**A.4.1** ESAPI API for creating a dynamic conformal arc with specified start/stop angles
*Blocks:* `DynamicConformalArcBuilder`

**A.4.2** ESAPI MLC motion validation between control points

**A.4.3 (new in v0.4.1)** Verify that beam-level jaw settings correctly propagate to all control points of a dynamic conformal arc when HBB is active. Specifically: create a DCA beam, set the Y jaw to 0 cm at isocenter (HBB-Superior configuration), inspect that all control points inherit the jaw setting, and confirm that the resulting MLC motion respects the clipped half-field across the arc.
*Blocks:* §11.4 HBB+DCA interaction claim
*Bad-answer fallback:* If Eclipse does not propagate jaws to control points under HBB, the recipe must either set the jaw per control point manually (complexity cost) or refuse HBB on cases that use the 2-field PA + DCA technique or the hybrid technique. The 3-field and 4-field techniques are unaffected because they do not use DCA.

### A.5 Plan lifecycle and concurrency

**A.5.1** ESAPI transaction model for plan creation
*Blocks:* `PlanLifecycleManager` design

**A.5.2** Behavior when two processes access the same patient course simultaneously
*Blocks:* Concurrency strategy (standalone executable case)

**A.5.3** ESAPI cleanup API for removing a partially-created plan after failure
*Bad-answer fallback:* Manual reverse-order cleanup, including recipe-tagged temporary structures

### A.6 Structure geometry APIs

**A.6.1** ESAPI methods for checking structure connectivity
*Blocks:* Geometric validation

**A.6.2** API for detecting zero-volume or near-degenerate slices

**A.6.3** API for querying whether a point lies inside a structure volume
*Blocks:* Isocenter sanity check

**A.6.4** Structure set versioning side effects when inserting the couch model

**A.6.5 (new in v0.4)** Cost and behavior of creating a temporary structure from an isodose region in ESAPI. This is a routine operation but needs empirical measurement for FIF latency planning. Specifically: what is the wall-clock cost per isodose-to-structure conversion, does it trigger structure set versioning, and is there a bulk delete operation for removing tagged temporary structures at FIF exit?
*Blocks:* FIF kernel per-iteration cost estimate

### A.7 End-to-end FIF workflow demonstration (HIGHEST PRIORITY — EXECUTE BEFORE PHASE 1)

**A.7.1** Demonstrate end-to-end that the §15.1 FIF workflow is achievable through ESAPI on a representative test plan, before any production code is written. Specifically:

1. Create a test plan with a parent beam
2. Calculate dose
3. Identify a hot region manually
4. Create a temporary structure from an isodose contour (per A.6.5)
5. Create a subfield on the parent beam via the ESAPI API (per A.3.2)
6. Adjust the parent beam's field weight via `GetEditableParameters()` (per A.1.1)
7. Recalculate dose
8. Read back the composite dose distribution
9. Confirm the hot region has been reduced
10. Clean up the temporary structure

If any step fails, **the §15.1 workflow as written is fictional and Phase 4 must be redesigned from scratch, and Phase 1 primitive contracts may need rework.**

**v0.4 update:** A.7 should be executed **before Phase 1 begins**, not just before Phase 4, because the workflow affects Phase 1 primitive contract design.

*Time estimate:* 1-2 days on a test plan with a phantom CT.

### A.8 Decision log persistence

**A.8.1** Where does the JSON decision log live? Aria document, file, or logging service?
*Blocks:* `DecisionLogger` destination

---

## Document History

| Version | Date | Author | Changes |
|---|---|---|---|
| 0.1 | 2026-04-13 | Stephen Eller (with Claude) | Initial draft |
| 0.2 | 2026-04-13 | Stephen Eller (with Claude) | First adversarial review revision |
| 0.3 | 2026-04-14 | Stephen Eller (with Claude) | Second adversarial review revision |
| 0.4 | 2026-04-14 | Stephen Eller (with Claude) | Third adversarial review revision — CC:Gen decoupling; plateau FIF; single-process execution; HBB cross-cutting primitive; hybrid escalation rung; explicit FIF deferral to Phase 4 with sandbox; anatomy-driven level estimator; 0.3% threshold and N-beam-in-a-row rule; 6X pacemaker cap; hybrid technique; freeze candidate |
| 0.4.1 | 2026-04-14 | Stephen Eller (with Claude) | Post-review patch — propagation fixes in §2.2 / §4 / §7.1; PTV block margin as named input (§6.4, §11.2); MLC handling rewritten as refit-from-offset-isocenter (§11.5, §13); `MLCFitter` no longer HBB-aware (§21); iteration terminology unified (§15.4.1, §20.4); hybrid validation cases added (§24.1); Appendix A.4.3 added for HBB+DCA jaw propagation verification. Freeze-ready. |
| **0.4.2** | **2026-04-14** | **Stephen Eller (with Claude)** | **FROZEN.** Freeze confirmation review applied 5 propagation cleanup edits (§5.2, §5.4, §20.4, §23, §26) plus two struck stale bullets from the §29 v0.3→v0.4 changelog that described the v0.4 HBB MLC handling approach (superseded by v0.4.1). Zero new architecture. The document is the freeze version and input to architectural planning. |

---

*v0.4.2 is the FROZEN T-Spine Automation Strategy document. The strategy phase is complete. The next artifact is the architectural planning document, which consumes v0.4.2 as input and specifies interface contracts for the primitives named in §21. Phase 0 ESAPI investigation (Appendix A, especially A.7) runs in parallel with architectural planning. The FIF sandbox application begins development as soon as the kernel interface contracts are settled. This document will not be revised further. Any future changes will be captured in the architectural planning document or in implementation-phase documentation, not in retrofitted strategy revisions.*
