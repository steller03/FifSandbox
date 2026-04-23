# Phase 0 FIF Sandbox — Design Specification

**Status:** Draft v2 — revised after first adversarial review
**Purpose:** A standalone C# application that demonstrates the v0.4.2 §15.1 FIF workflow end-to-end on a test plan, answering the Appendix A.7 gating question before Phase 1 begins.
**Owner:** Stephen Eller
**Target framework:** C# 7.3 / .NET Framework 4.8 / ESAPI v16.1
**Build platform:** x64 (required for ESAPI standalone executables)
**Pattern:** Standalone executable (not a plugin)

**Compatibility note:** .NET Framework 4.8 is an intentional deviation from Varian's documented 4.6.1 target for ESAPI v16.1. This has been validated across all production Pscripts applications (CC:Gen, ProtocolPlanner, TreatmentPlanReport, CreateQAPlan, PlanScorerV2, BreastHelper). The 4.8 target is a project-wide decision, not an oversight.

**Revision note:** v2 incorporates findings from the first adversarial review round. Key changes: unit-safe hotspot threshold computation, A.4.3 restructured into beam-level and per-CP subtests, explicit try/finally control flow around the mutation window, eligible-parent-beam filter, cord Dmax as observational metric (not clinical rule), temp structure clearing before reuse in multi-iteration variant, and resolution of two open questions.

---

## 1. What This Sandbox Proves

The sandbox exists to answer one question: **does the §15.1 FIF workflow actually work through ESAPI?**

Specifically, the sandbox demonstrates that the following cycle runs successfully on a real patient dataset:

1. Open a patient with a structure set containing a PTV and SpinalCord
2. Calculate initial dose on an existing 3D conformal plan
3. Identify the global hotspot (`Dose.DoseMax3D`, `Dose.DoseMax3DLocation`)
4. Create a temporary CONTROL structure from an isodose region (`ConvertDoseLevelToStructure`)
5. Build a sub-aperture structure (PTV minus hotspot region via `SegmentVolume.Sub`)
6. Add a sibling beam matching a parent beam's geometry (`AddStaticBeam`)
7. Fit the sibling beam's MLC to the sub-aperture structure (`FitMLCToStructure`)
8. Adjust field weights on both parent and sibling beams (`GetEditableParameters` → `WeightFactor` → `ApplyParameters`)
9. Recalculate dose (`CalculateDose`)
10. Read back composite dose and confirm the hotspot decreased
11. Undo the modification if improvement is below threshold (remove sibling beam, restore parent weight, recalculate)
12. Clean up all temporary structures on exit

If all 12 steps succeed, A.7 passes. If any step fails, the sandbox reports exactly which step failed and why, giving the team the information needed to revise the strategy before Phase 1 begins.

### Secondary objective: A.4.3 (HBB+DCA jaw propagation)

The sandbox also includes a secondary test structured as **two subtests**:

1. **Subtest A (beam-level propagation):** Creates a DCA beam, sets the jaw to 0 cm at isocenter via `BeamParameters.SetJawPositions` (beam-level broadcast), then reads back every control point to verify the jaw value propagated. Tests both Y1=0.0 and Y2=0.0 to determine the directional convention empirically.
2. **Subtest B (per-CP fallback):** Only runs if Subtest A fails. Tests whether per-CP jaw edits survive `ApplyParameters`, verifying the manual-per-CP fallback path.

This answers v0.4.2 Appendix A.4.3 and resolves the Y1-vs-Y2 jaw convention question without requiring a separate test harness.

---

## 2. Design Principles

### 2.1 Portability to ProtocolPlanner

The sandbox is a development tool, but its core code — the FIF iteration body, the snapshot-and-restore pattern, the temp structure lifecycle, the hotspot identification logic — should be structured so it can be lifted into `Pscripts.Planning` later with minimal rework.

**What this means in practice:**
- **Separate the FIF logic from the standalone harness.** The iteration body, the `PlanSnapshot` class, the undo pattern, and the metric computation should live in classes that take ESAPI objects as parameters, not in the `Main` method. The `Main` method is the test harness — it opens the patient, sets up the test, calls the FIF logic, and reports results. The FIF logic itself is portable.
- **Use the same patterns the ESAPI skills document.** The sandbox code should read like a direct translation of the iterative-dose-modification skill's Section 12 worked example. When the code ports to `Pscripts.Planning`, an implementer reading both the skill and the code should see a 1:1 mapping.
- **Don't over-engineer for the sandbox.** The sandbox doesn't need dependency injection, configuration files, or a UI. It needs to run, prove the workflow works, and report results. Architectural scaffolding belongs in Phase 1, not Phase 0.

### 2.2 Fail-loud, fail-specific

The sandbox is a verification tool. Every ESAPI operation should be wrapped in checks that report exactly what happened:
- Every `CalculationResult.Success` check should log what step failed
- Every null navigation should log what was null
- Every `CanRemoveStructure` false-return should log which structure couldn't be removed and why
- The sandbox should never silently continue past a failure — it should log, clean up, and exit

### 2.3 Always clean up

The sandbox creates temporary structures and sibling beams. On any exit path — success, failure, or exception — all sandbox-created artifacts must be removed. The `PlanSnapshot` restore pattern handles beams and weights; a separate cleanup pass handles CONTROL structures tagged with the sandbox prefix.

### 2.4 Observable output

The sandbox should produce a structured text report (to console and optionally to a file) that walks through each step with:
- Step name
- Pass/fail
- Key metrics (e.g., "DoseMax3D before: 11023 cGy, after: 10785 cGy, delta: -2.16% of Rx")
- Timing (wall-clock per step, total runtime)
- Any diagnostic messages from `BeamCalculationLog` on failure

This report is the artifact that proves A.7 passed or identifies where it failed.

---

## 3. Application Structure

### 3.1 Entry point

```
FifSandbox.exe
├── Program.cs              — Main, Application lifecycle, patient open/close
├── FifSandboxRunner.cs     — Orchestrates the A.7 test sequence
├── HbbDcaTestRunner.cs     — Orchestrates the A.4.3 test (secondary)
├── IterationEngine.cs      — The portable FIF iteration body
├── PlanSnapshot.cs         — Snapshot capture and restore (from iterative-dose-modification skill)
├── HotspotAnalyzer.cs      — Hotspot identification and beam contribution ranking
├── TempStructureManager.cs — Temp structure create/find-or-create/cleanup
├── SandboxReport.cs        — Structured output formatting
└── SandboxConfig.cs        — Stub MRN, structure IDs, thresholds, iteration budget
```

### 3.2 Class responsibilities

**`Program.cs`** — the standalone executable bootstrap. Opens the ESAPI `Application`, opens the patient by MRN, finds the target plan, runs `FifSandboxRunner`, then runs `HbbDcaTestRunner`, reports results, closes the patient, disposes the application. This is the only class that touches `Application` lifecycle.

**`FifSandboxRunner.cs`** — orchestrates the A.7 test sequence. Takes an `ExternalPlanSetup` and a `SandboxConfig`. Calls `BeginModifications` once. Runs the iteration loop using `IterationEngine`. Captures the `PlanSnapshot` before the first iteration. Reports pass/fail for each of the 12 steps. Restores the snapshot on any failure or at test completion so the plan is left in its original state.

**`HbbDcaTestRunner.cs`** — orchestrates the A.4.3 secondary test via two subtests. Subtest A creates a conformal arc beam and tests beam-level jaw propagation via `SetJawPositions` (tests both Y1=0.0 and Y2=0.0). Subtest B (only if A fails) tests per-CP jaw edits as a fallback. Logs initial and final jaw positions so the physicist can determine the directional clipping convention. Removes the test beam and restores the plan on completion.

**`IterationEngine.cs`** — the portable iteration body. This is the class that eventually moves to `Pscripts.Planning`. It implements one iteration cycle: identify hotspot → create temp structure → build sub-aperture → add sibling beam → fit MLC → adjust weights → recalculate → check improvement → accept or undo. Takes an `ExternalPlanSetup`, a `PlanSnapshot`, and iteration parameters. Returns an `IterationResult` (accepted/undone/failed, metrics, diagnostics).

**`PlanSnapshot.cs`** — the snapshot class from the iterative-dose-modification skill's Section 11. Capture: beam IDs, weight factors, structure IDs, normalization value. Restore: remove added beams, restore weights, remove added structures, recalculate.

**`HotspotAnalyzer.cs`** — wraps the hotspot identification and beam contribution ranking patterns from `beam-dose-deep.md`. Takes a plan with valid dose, returns the hotspot location, the hotspot `DoseValue`, and a ranked list of beam contributions at the hotspot.

**`TempStructureManager.cs`** — wraps the temp structure lifecycle from `structure-ops.md` and `isodose-analysis-deep.md`. Creates CONTROL structures with the sandbox prefix, implements find-or-create idempotency, converts isodose regions to structures, builds sub-aperture structures via `SegmentVolume.Sub`, and cleans up all sandbox-prefixed structures on exit.

**`SandboxReport.cs`** — formats and outputs the structured test report. Each step is logged with pass/fail, timing, and metrics. The report is written to console and optionally to a file path from `SandboxConfig`.

**`SandboxConfig.cs`** — a simple data class holding:
- `PatientMrn` — stubbed as `"11223344"`, replaced in testing
- `PlanId` — the plan to run against (**required**, no default — the sandbox must target a specific known plan for trustworthy evidence)
- `TargetStructureId` — the PTV structure ID (e.g., `"PTV_Spine"`)
- `CordStructureId` — the spinal cord structure ID (e.g., `"SpinalCord"`)
- `ImprovementThresholdPct` — minimum improvement per iteration as % of Rx (e.g., `0.3`)
- `MaxIterations` — iteration budget (e.g., `5` for the sandbox)
- `SubfieldWeightFraction` — fraction of parent MU to allocate to the subfield (e.g., `0.10`)
- `HotspotThresholdFraction` — isodose level for hotspot structure, as a fraction of Rx (e.g., `1.05` for 105%). **This is a multiplier, not a percentage.** The hotspot threshold is computed as `plan.TotalDose * HotspotThresholdFraction`, preserving the `DoseValue` unit envelope. See §4 Step 4 for the unit-safe computation.
- `TempStructurePrefix` — prefix for sandbox-created structures (e.g., `"SB_"`)
- `ReportFilePath` — optional file path for the output report (null = console only)
- `DoseDeltaToleranceAbsolute` — tolerance for dose-restoration comparisons, in the session's canonical dose unit (whatever `plan.TotalDose.Unit` is — typically cGy). Example: `1.0` (meaning 1.0 cGy if the session unit is cGy, or 0.01 Gy if the session unit is Gy).
- `JawPositionToleranceMm` — tolerance for A.4.3 jaw position verification (e.g., `0.01` mm)

**Portable classes take primitive parameters, not `SandboxConfig`.** The portable classes (`IterationEngine`, `PlanSnapshot`, `HotspotAnalyzer`, `TempStructureManager`) accept ESAPI objects and individual values as constructor/method parameters. `SandboxConfig` is consumed only by the harness code (`FifSandboxRunner`, `HbbDcaTestRunner`) which unpacks it into the primitives the portable classes need. This ensures `SandboxConfig` does not leak into the portable surface when the classes move to `Pscripts.Planning`.

### 3.3 What is NOT in the sandbox

- No UI (no WPF, no WinForms, no dialogs)
- No clinic configuration files
- No recipe-specific logic (no technique selection, no energy selection, no level estimation)
- No decision logging beyond the sandbox report
- No HBB isocenter offset logic (A.4.3 tests jaw propagation only, not the `IsocenterHelper` extension)
- No `Pscripts.Planning` dependency (the sandbox uses ESAPI directly — the portable classes are designed to move to `Pscripts.Planning` later, but they don't depend on it now)
- No multi-patient or batch processing
- No prescription modification
- No normalization modification

---

## 4. The A.7 Test Sequence

The `FifSandboxRunner` executes these steps in order. Each step is individually pass/fail reported. The sequence stops on the first hard failure; soft failures (e.g., improvement below threshold) are reported but the sequence continues.

### Step 1: Validate inputs and preflight

- Confirm the plan is an `ExternalPlanSetup`
- **Preflight: check that the plan is not approved.** An approved plan will throw on `BeginModifications`. Check `PlanSetup.ApprovalStatus` and refuse to run with a clear "plan is approved — cannot modify" message if it's not in a modifiable state. Report this distinctly from ordinary plan-content failures.
- Confirm `StructureSet` is not null
- Find the target structure by `SandboxConfig.TargetStructureId` — fail if missing or empty
- Find the cord structure by `SandboxConfig.CordStructureId` — fail if missing or empty
- **Build the eligible-parent-beam list.** An eligible beam is one that satisfies ALL of: `!beam.IsSetupField`, `beam.MLC != null`, `beam.MLCPlanType == MLCPlanType.Static`. Fail if no eligible beams exist. Log the eligible beam IDs and the ineligible beam IDs with reasons for exclusion.
- Confirm the plan has a prescription (`TotalDose` is not undefined)
- Log: structure IDs found, total beam count, eligible beam count, prescription dose (with unit)

### Step 2: Initial dose calculation

- **Call `BeginModifications()` here** (once, before any write operation in the entire test sequence). This is early enough to catch write-permission failures before doing substantive work, and late enough that Step 1's read-only validation has already passed.
- If `IsDoseValid` is false, call `CalculateDose()`
- Check `CalculationResult.Success` — hard fail if false (collect `BeamCalculationLog`)
- **Log the calculation model and dose grid settings** for run-to-run comparability: `plan.PhotonCalculationModel`, dose grid resolution from calculation options, any other relevant model parameters
- Log: `DoseMax3D` (with unit), `DoseMax3DLocation`, `SpinalCord.Dmax` via `GetDoseAtVolume(cord, 0.03, AbsoluteCm3, Absolute)` (with unit)
- Capture the initial `PlanSnapshot`

### Step 3: Identify the global hotspot

- Read `Dose.DoseMax3D` and `DoseMax3DLocation`
- **Rank only eligible beams** (from the eligible list built in Step 1) by their contribution at the hotspot using the per-beam point dose pattern. Skip ineligible beams entirely — they cannot be modified by the FIF workflow and should not be considered as parent candidates.
- Log: hotspot location, hotspot dose, top 3 eligible beam contributions with beam IDs and absolute doses
- Identify the dominant eligible contributor beam (highest absolute dose at the hotspot among eligible beams)
- If no eligible beam has a non-undefined dose contribution at the hotspot, fail with a diagnostic

### Step 4: Create hotspot temporary structure

- **Unit-safe hotspot threshold computation.** Compute the threshold as a `DoseValue` that preserves the unit envelope of `plan.TotalDose`:
  ```
  DoseValue hotThreshold = plan.TotalDose * config.HotspotThresholdFraction;
  ```
  Where `HotspotThresholdFraction` is a dimensionless multiplier (e.g., `1.05` for 105% of Rx). The resulting `hotThreshold` inherits the unit of `plan.TotalDose` — if the prescription is in cGy, the threshold is in cGy; if in Gy, it's in Gy. **Do not extract `.Dose` and re-wrap with a hardcoded unit.** Log both the Rx dose and the threshold with their units.
- Find-or-create a CONTROL structure with ID `"{prefix}HotSpot"` (e.g., `"SB_HotSpot"`). **If the structure already exists from a prior run, clear its geometry before reuse** — call `ClearAllContoursOnImagePlane` on every slice, or remove and recreate the structure. Do not depend on `ConvertDoseLevelToStructure` overwriting existing geometry (the behavior is empirically unverified — see §11).
- Call `structure.ConvertDoseLevelToStructure(plan.Dose, hotThreshold)` — passing the `DoseValue` directly
- Verify the structure is not empty (`!structure.IsEmpty && structure.HasSegment`)
- Log: hotspot structure ID, volume, threshold dose (with unit), whether it was created fresh or cleared-and-reused

### Step 5: Build sub-aperture structure

- Find-or-create a CONTROL structure with ID `"{prefix}SubAp"` (e.g., `"SB_SubAp"`). **If the structure already exists, clear its geometry before reuse** (same discipline as Step 4).
- Assign `subAperture.SegmentVolume = target.SegmentVolume.Sub(hotspot.SegmentVolume)`
- Verify not empty
- Log: sub-aperture volume

### Step 6: Add sibling beam

- Get the dominant contributor beam from Step 3
- Create a new static beam matching the parent's geometry:
  - Same `ExternalBeamMachineParameters` (machine, energy, dose rate, technique)
  - Same collimator angle, gantry angle, couch angle, isocenter
  - Placeholder jaws (will be overwritten by `FitMLCToStructure`)
- Assign a sandbox-specific beam ID: `"{prefix}FiF_{iteration}"` (e.g., `"SB_FiF_1"`)
- Log: sibling beam ID, parent beam ID, parent geometry

### Step 7: Fit MLC to sub-aperture

- Preflight: `sibling.GetStructureOutlines(subAperture, inBEV: true)` — verify non-empty
- Call `sibling.FitMLCToStructure` with the 6-parameter overload:
  - `FitToStructureMargins(0.0)` — zero margin (the sub-aperture already has the right shape)
  - `optimizeCollimatorRotation: false` — preserve parent geometry
  - `JawFitting.FitToRecommended`
  - `OpenLeavesMeetingPoint.OpenLeavesMeetingPoint_Middle`
  - `ClosedLeavesMeetingPoint.ClosedLeavesMeetingPoint_Center`
- Log: fit completed, no exceptions

### Step 8: Adjust field weights

- Compute the subfield weight: `parentWeight * SandboxConfig.SubfieldWeightFraction`
- Get fresh `BeamParameters` for the sibling, set `WeightFactor`, `ApplyParameters`
- Get fresh `BeamParameters` for the parent, reduce `WeightFactor` by the same delta, `ApplyParameters`
- Log: parent weight before/after, sibling weight, total weight (should equal original parent weight)

### Step 9: Recalculate dose

- Call `CalculateDose()`
- Check `CalculationResult.Success` — hard fail if false (collect `BeamCalculationLog`)
- Log: calculation success, wall-clock time for the recalc

### Step 10: Evaluate improvement

- Read `Dose.DoseMax3D` (post-modification)
- **Unit-safe improvement computation** (from iterative-dose-modification skill §6):
  - Assert `previousMax.Unit == newMax.Unit == plan.TotalDose.Unit` — throw on mismatch
  - Check `IsUndefined()` on all three — break if any is undefined
  - Compute: `improvementRaw = previousMax.Dose - newMax.Dose` (positive = improvement)
  - Compute: `improvementPctOfRx = (improvementRaw / plan.TotalDose.Dose) * 100.0`
- **Cord Dmax as observational metric.** Read `SpinalCord.Dmax` via `GetDoseAtVolume(cord, 0.03, AbsoluteCm3, Absolute)`. Report the value with unit. **Do not classify as pass/fail against a threshold.** The sandbox proves the ESAPI mechanics work; clinical threshold evaluation belongs in the production recipe, not in the sandbox.
- Log: previous `DoseMax3D` (with unit), new `DoseMax3D` (with unit), improvement as % of Rx, cord Dmax (with unit, observational only)

### Step 11: Test the undo path

Regardless of whether the modification improved the plan, test the undo:
- Remove the sibling beam (`RemoveBeam`)
- Restore the parent's `WeightFactor` from the snapshot
- Recalculate dose
- Verify `DoseMax3D` matches the initial value within `SandboxConfig.DoseDeltaToleranceAbsolute`
- Log: undo completed, restored `DoseMax3D`, delta from initial (expected to be within tolerance)

### Step 12: Cleanup and final report

- Remove all sandbox-prefixed structures (`TempStructureManager.CleanupAll`)
- Restore from the initial `PlanSnapshot` if any state was left modified
- Final recalculate to ensure the plan is left in its original state
- Log: cleanup completed, final `DoseMax3D` matches initial (within `SandboxConfig.DoseDeltaToleranceAbsolute`)
- Write the full report

### Control flow: try/finally around the mutation window

The entire mutation window (Steps 4-11) is wrapped in a try/finally:

```
Step 1: Validate inputs (read-only, no try/finally needed)
Step 2: BeginModifications + initial dose calc + snapshot capture

try
{
    Step 4: Create hotspot structure
    Step 5: Build sub-aperture
    Step 6: Add sibling beam
    Step 7: Fit MLC
    Step 8: Adjust weights
    Step 9: Recalculate
    Step 10: Evaluate
    Step 11: Test undo
}
finally
{
    Step 12: PlanSnapshot.Restore + TempStructureManager.CleanupAll + final recalc + report flush
}
```

The finally block **always** runs, regardless of how the try block exited (success, hard failure, exception). This guarantees that:
- All sandbox-created beams are removed
- All weight factors are restored to their pre-sandbox values
- All sandbox-prefixed temp structures are cleaned up
- The dose grid is recalculated to reflect the restored state
- The report is flushed to console/file even on failure

An outer try/finally in `Program.cs` wraps the patient open/close lifecycle:

```
try
{
    Application.OpenPatientById(mrn)
    FifSandboxRunner.Run(plan, config)
    HbbDcaTestRunner.Run(plan, config)   // only if A.7 cleanup succeeded
}
finally
{
    // Do NOT call SaveModifications — discard all changes
    Application.ClosePatient()
    Application.Dispose()
}
```

This two-level try/finally structure ensures cleanup at both the test level (sandbox-created artifacts) and the application level (patient context).

### Multi-iteration variant (Steps 3-10 in a loop)

After the single-iteration test passes, the sandbox runs a multi-iteration variant:
- Capture a fresh snapshot
- Loop Steps 3-10 for `SandboxConfig.MaxIterations` iterations (or until plateau)
- **At the start of each iteration, clear existing temp structure geometry before reuse** (same clearing discipline as Steps 4-5 in the single-iteration test). Do not depend on `ConvertDoseLevelToStructure` overwriting existing geometry — explicitly clear or recreate the temp structures each time.
- On each iteration, log the same metrics
- On plateau (improvement below threshold), break and report
- On completion, restore from snapshot and clean up
- The multi-iteration variant is wrapped in the same try/finally structure as the single-iteration test

This exercises the full iterative workflow (though the sandbox doesn't implement the N-beam counter — it uses a simple "break on first failure to improve" rule, which is sufficient to verify the ESAPI mechanics even though the production kernel will have the full counter logic).

**A.4.3 dependency:** The A.4.3 test only runs after the A.7 sequence (both single-iteration and multi-iteration) has completed and cleanup has restored the plan to its initial state. If A.7 cleanup failed, A.4.3 is skipped with a "SKIPPED (A.7 cleanup failure)" report.

---

## 5. The A.4.3 Test Sequence

The `HbbDcaTestRunner` executes after the A.7 sequence completes successfully. It operates on the same plan. **It only runs if A.7 cleanup restored the plan to its initial state.** If A.7 cleanup failed, A.4.3 is skipped.

The test is structured as **two subtests** that probe different jaw-write paths:

- **Subtest A (beam-level propagation):** the real A.4.3 question — does a beam-level jaw write propagate to all DCA control points?
- **Subtest B (per-CP fallback):** if Subtest A fails, does per-CP jaw editing at least work? This tests the manual-per-CP fallback path from v0.4.2 Appendix A.4.3's bad-answer fallback.

### Step 1: Create a conformal arc beam

- `plan.AddConformalArcBeam` with:
  - Same machine params as an existing beam in the plan
  - Collimator 0°
  - Gantry start 50°, stop 310° (anterior arc matching v0.4.2 §14.1)
  - `GantryDirection.Clockwise` (or `CounterClockwise` — doesn't matter for this test)
  - Couch 0°
  - Same isocenter as an existing beam
  - A reasonable control point count (e.g., 10-20)
- Assign beam ID: `"{prefix}DCA_Test"`
- Log: beam created with control point count

### Step 2 — Subtest A: Beam-level jaw write via `SetJawPositions`

This subtest answers the core A.4.3 question. It uses `BeamParameters.SetJawPositions(VRect<double>)` — the **beam-level broadcast method** that sets the same jaw positions across all control points in a single call.

- Get `BeamParameters` for the DCA beam
- **Set jaws at the beam level** using the broadcast method:
  ```
  bp.SetJawPositions(new VRect<double>(currentX1, testJawValue, currentX2, currentY2))
  ```
  Where `testJawValue` is the jaw position being tested (0.0 mm for HBB simulation). **Test both Y1 = 0.0 and Y2 = 0.0** to resolve open question #4 (which jaw clips which direction in IEC BLD coordinates). Run the subtest twice — once with Y1 = 0.0 and once with Y2 = 0.0 — and report which convention produces the expected clipping.
- `beam.ApplyParameters(bp)`
- **Read back every control point's `JawPositions`** and verify the test jaw value is present on all CPs within `SandboxConfig.JawPositionToleranceMm`
- Log: control point count, test jaw value per CP, pass/fail per CP

**Subtest A pass criterion:** ALL control points have the expected jaw value within tolerance after a single beam-level `SetJawPositions` call.

**Directional convention resolution:** The sandbox does not encode the IEC BLD sign convention into its test logic. Instead, it **logs the initial jaw positions** (from `AddConformalArcBeam`) and the **final jaw positions** (after `SetJawPositions`) for both the Y1=0.0 and Y2=0.0 test runs. The delta report (e.g., "Y1 moved from -60.0 to 0.0, Y2 unchanged at 52.0") gives the physicist the raw data to determine which jaw clips which direction from clinical knowledge of the IEC BLD coordinate system. The sandbox proves propagation; directional interpretation is the physicist's responsibility.

**Subtest A result interpretation:**
- **PASS:** Eclipse propagates beam-level jaw settings to all DCA control points. HBB can use the beam-level jaw write and trust that all CPs inherit it. The production recipe can use `SetJawPositions` at the beam level for HBB on DCA beams.
- **FAIL:** Beam-level jaw writes do not propagate to DCA control points. Proceed to Subtest B.

### Step 3 — Subtest B: Per-CP jaw write (diagnostic fallback)

This subtest only runs if Subtest A fails. It tests whether per-CP jaw editing works as a fallback.

- Get fresh `BeamParameters` for the DCA beam
- **Set jaws per control point** by looping `bp.ControlPoints`:
  ```
  foreach (ControlPointParameters cpp in bp.ControlPoints)
  {
      VRect<double> currentJaws = cpp.JawPositions;
      cpp.JawPositions = new VRect<double>(currentJaws.X1, testJawValue, currentJaws.X2, currentJaws.Y2);
  }
  ```
- `beam.ApplyParameters(bp)`
- **Read back every control point's `JawPositions`** and verify the test jaw value is present on all CPs within tolerance
- Log: control point count, test jaw value per CP, pass/fail per CP

**Subtest B result interpretation:**
- **PASS:** Per-CP jaw edits survive `ApplyParameters`. The production recipe's fallback path (set jaws per-CP manually when beam-level propagation doesn't work) is viable. The recipe must loop CPs instead of using `SetJawPositions`, which adds complexity but is functionally equivalent.
- **FAIL:** Neither beam-level nor per-CP jaw edits work for DCA beams under HBB. The recipe must refuse HBB on 2-field PA+DCA and hybrid techniques (per the v0.4.2 bad-answer fallback).

### Step 4: Cleanup

- Remove the DCA test beam (`RemoveBeam`)
- Recalculate dose to restore the plan
- Log: cleanup completed

### Summary table

| Subtest A | Subtest B | Conclusion |
|---|---|---|
| PASS | (not run) | Beam-level jaw propagation works. HBB on DCA is safe with `SetJawPositions`. |
| FAIL | PASS | Beam-level propagation broken, but per-CP fallback works. Recipe must loop CPs. |
| FAIL | FAIL | Neither path works. Recipe must refuse HBB on DCA techniques. |

---

## 6. ESAPI Skills Referenced

The sandbox code should be a direct translation of patterns from these skills:

| Skill | What the sandbox uses from it |
|---|---|
| `iterative-dose-modification-patterns.md` | Section 12 worked example (the iteration loop skeleton), Section 11 `PlanSnapshot`, Section 10 undo pattern, Section 6 unit-safe delta, Section 4 `IsDoseValid` discipline |
| `mlc-fitting-and-fif-workflows.md` | Pattern A steps 1-10 (the one-shot FIF body), the sibling beam creation pattern, `FitMLCToStructure` 6-parameter overload, `GetEditableParameters`/`WeightFactor`/`ApplyParameters` |
| `beam-dose-deep.md` | "Hotspot Analysis" section, "Per-Beam Point Dose Query" section, beam contribution ranking pattern |
| `dose-query.md` | `DoseMax3D`, `DoseMax3DLocation`, `GetDoseAtVolume` for cord Dmax, `DoseValue` unit handling |
| `isodose-analysis-deep.md` | `ConvertDoseLevelToStructure` for hotspot structure creation |
| `structure-ops.md` | `AddStructure`, `RemoveStructure`, `CanRemoveStructure`, `SegmentVolume.Sub` |
| `calculation-models.md` | `CalculateDose()`, `CalculationResult.Success` |
| `error-diagnostics-deep.md` | `BeamCalculationLog` diagnostic collection on failure |
| `api-limitations.md` | `IsDoseValid` silent-failure trap, collection iteration safety, thread affinity |
| `batch-standalone-deep.md` | Standalone executable pattern: `Application.CreateApplication`, `OpenPatientById`, `SaveModifications`, `ClosePatient`, `Dispose` |
| `beam-geometry-deep.md` | `AddConformalArcBeam` for A.4.3, `ExternalBeamMachineParameters` construction |
| `control-points-deep.md` | `BeamParameters.ControlPoints` jaw inspection for A.4.3, `GetEditableParameters`/`ApplyParameters` for jaw writes |
| `plan-creation.md` | `AddStaticBeam` for sibling beam creation |

---

## 7. Test Dataset Requirements

The sandbox runs against a patient with MRN `"11223344"` (stub — replace in testing).

### Required dataset characteristics

- A CT image set with a structure set containing:
  - A PTV structure (ID configurable via `SandboxConfig.TargetStructureId`)
  - A SpinalCord structure (ID configurable via `SandboxConfig.CordStructureId`)
  - An EXTERNAL / BODY structure
  - At least one lung structure (not strictly required for the sandbox, but realistic for T-spine)
- An existing 3D conformal plan on the structure set with:
  - At least 2 static beams (PA + one other), each with an MLC
  - A prescription (dose per fraction, number of fractions)
  - A calculation model assigned (AAA or AcurosXB)
  - Valid dose (or the ability to calculate dose — the sandbox recalculates as its first step)
- The plan should NOT be approved (write operations fail on approved plans)
- The plan should be on a machine the sandbox's Eclipse installation has configured

### What the sandbox does NOT require

- Any specific fractionation
- Any specific beam arrangement (2-field, 3-field, 4-field — any will work)
- Any specific energy
- HBB to already be set up (the sandbox tests HBB mechanics independently)
- Any Pscripts infrastructure

---

## 8. Expected Output

### Console output (sample)

```
=== Phase 0 FIF Sandbox ===
Patient: 11223344
Plan: TSpine_Test
Target: PTV_Spine | Cord: SpinalCord
Build: x64 / .NET 4.8 / ESAPI 16.1

--- A.7: End-to-End FIF Workflow ---

Step  1: Validate inputs .............. PASS
         Plan approval status: Unapproved (modifiable)
         Beams: 4 total (AP, PA, LPO, RPO)
         Eligible for FIF: 4 (all static MLC)
         Rx: 3000 cGy / 10 fx (unit: cGy)
Step  2: Initial dose calculation ..... PASS  (3.2s)
         Calc model: AAA_16.1.0 | Grid: 0.25 cm
         DoseMax3D: 3312 cGy (110.4% Rx) | Cord Dmax: 3098 cGy (103.3% Rx)
Step  3: Identify global hotspot ...... PASS
         Location: (12.3, -45.6, 78.9) mm
         Top eligible contributor: PA (892 cGy/fx, 34.2%)
Step  4: Create hotspot structure ..... PASS
         Threshold: 3150 cGy (105.0% of 3000 cGy Rx)
         SB_HotSpot: 4.2 cc (created new)
Step  5: Build sub-aperture ........... PASS
         SB_SubAp: 82.1 cc (cleared and rebuilt)
Step  6: Add sibling beam ............. PASS
         SB_FiF_1 (parent: PA, gantry: 180.0°)
Step  7: Fit MLC to sub-aperture ...... PASS
Step  8: Adjust field weights ......... PASS
         PA: 0.35 → 0.315 | SB_FiF_1: 0.035
Step  9: Recalculate dose ............. PASS  (3.4s)
Step 10: Evaluate improvement ......... PASS
         DoseMax3D: 3245 cGy (108.2% Rx) | Delta: -2.23% of Rx
         Cord Dmax: 3062 cGy (102.1% Rx) [observational]
Step 11: Test undo path ............... PASS  (3.1s)
         Restored DoseMax3D: 3312 cGy | Delta from initial: 0.00 cGy (within 1.0 cGy tolerance)
Step 12: Cleanup ...................... PASS
         Removed: SB_HotSpot, SB_SubAp, SB_FiF_1

--- Multi-iteration variant (5 iterations max) ---

Iter 1: DoseMax3D 3312 → 3245 cGy | Delta: -2.23% of Rx | ACCEPT
Iter 2: DoseMax3D 3245 → 3198 cGy | Delta: -1.57% of Rx | ACCEPT
Iter 3: DoseMax3D 3198 → 3172 cGy | Delta: -0.87% of Rx | ACCEPT
Iter 4: DoseMax3D 3172 → 3164 cGy | Delta: -0.27% of Rx | BELOW THRESHOLD (0.30%)
Plateau reached after 4 iterations.
Final DoseMax3D: 3172 cGy (105.7% Rx) | Cord Dmax: 3028 cGy [observational]
Restored to initial state: DoseMax3D 3312 cGy (within 1.0 cGy tolerance) | PASS

--- A.4.3: HBB+DCA Jaw Propagation ---

Step  1: Create conformal arc ......... PASS
         SB_DCA_Test: 50°→310° CW, 15 control points

Subtest A (beam-level SetJawPositions):
  Y1=0.0 test: SetJawPositions → ApplyParameters → readback
         15/15 control points have Y1 = 0.0 mm (within 0.01 mm) .... PASS
  Y2=0.0 test: SetJawPositions → ApplyParameters → readback
         15/15 control points have Y2 = 0.0 mm (within 0.01 mm) .... PASS
  Subtest A: PASS — beam-level jaw propagation confirmed

Subtest B: SKIPPED (Subtest A passed)

Step  4: Cleanup ...................... PASS

=== SUMMARY ===
A.7  FIF end-to-end:        PASS (all 12 steps)
A.7  Multi-iteration:       PASS (plateau at iteration 4)
A.4.3 Subtest A (beam-level): PASS (Y1 and Y2 both propagate)
A.4.3 Subtest B (per-CP):     SKIPPED
Total runtime: 48.7s
```

### Failure output (sample — Step 9 fails)

```
Step  9: Recalculate dose ............. FAIL
         CalculationResult.Success = false
         [PA/Dose] Grid does not cover target structure
         [SB_FiF_1/Dose] Grid does not cover target structure

--- finally block: emergency cleanup ---
         Restored from snapshot: PASS
         Removed temp structures: SB_HotSpot, SB_SubAp
         Final DoseMax3D: 3312 cGy (within 1.0 cGy tolerance)

=== SUMMARY ===
A.7  FIF end-to-end:        FAIL at Step 9 (CalculateDose failed)
A.4.3 HBB+DCA propagation: SKIPPED (A.7 cleanup incomplete)
```

---

## 9. Coding Standards

- C# 7.3 strict (no switch expressions, no nullable reference types, no using declarations)
- .NET Framework 4.8 (see compatibility note in header)
- **Solution platform: x64** (required for ESAPI standalone executables)
- String interpolation for all string building (no `string.Format`)
- `StringComparison.OrdinalIgnoreCase` for all ID comparisons
- `.FirstOrDefault()` + null check, never `.First()`
- `struct` types use `.Any()` guard, not null checks on `FirstOrDefault`
- Null-check every ESAPI navigation property
- Check `IsDoseValid` before every dose query
- Check `CalculationResult.Success` on every `CalculateDose()` call
- Full enum value names (`OpenLeavesMeetingPoint.OpenLeavesMeetingPoint_Middle`)
- **Unit-normalization policy:** all dose arithmetic operates on `DoseValue` objects, never on extracted `.Dose` doubles re-wrapped with a hardcoded unit. The unit envelope of `plan.TotalDose` is the canonical unit for the session. If a conversion to cGy is needed for reporting, it happens at the report boundary, not inside the computation path. See §4 Step 4 for the canonical pattern.
- No magic numbers — all thresholds, margins, tolerances, and IDs come from `SandboxConfig`
- XML doc comments on all public members
- Methods under 30 lines (sandbox relaxation of the usual 20-line rule, since some ESAPI workflows are inherently sequential)

---

## 10. Portability Notes

When the sandbox proves A.7 and the project moves to Phase 1:

**Classes that port to `Pscripts.Planning`:**
- `IterationEngine.cs` → becomes the core of the FIF kernel
- `PlanSnapshot.cs` → becomes `Pscripts.Planning.PlanSnapshot`
- `HotspotAnalyzer.cs` → becomes `Pscripts.Planning.HotspotLocator` (the v0.4.2 policy)
- `TempStructureManager.cs` → becomes part of `Pscripts.Planning.PlanLifecycleManager`

**Classes that stay as test infrastructure:**
- `Program.cs` — standalone harness, not needed in production
- `FifSandboxRunner.cs` — test orchestration, replaced by the recipe composition layer
- `HbbDcaTestRunner.cs` — one-off test, not needed after A.4.3 passes
- `SandboxReport.cs` — replaced by the production `DecisionLogger`
- `SandboxConfig.cs` — replaced by the production clinic configuration system

**The mapping from sandbox classes to v0.4.2 primitives:**

| Sandbox class | v0.4.2 primitive | Phase |
|---|---|---|
| `IterationEngine` | `FieldInFieldGenerator` (kernel) | Phase 4 |
| `PlanSnapshot` | `BestPlanTracker` (policy) | Phase 4 |
| `HotspotAnalyzer` | `HotspotLocator` (policy) + `BeamContributionEstimator` (policy) | Phase 4 |
| `TempStructureManager` | `PlanLifecycleManager` (cross-cutting) | Phase 6 |
| Step 10 evaluation | `AcceptanceEvaluator` | Phase 5 |
| Step 11 undo | `ConvergencePolicy` (the undo-and-advance logic) | Phase 4 |

---

## 11. Open Questions

1. **`AddConformalArcBeam` control point count parameter.** v0.4.2 §14.1 says "default Eclipse auto-generated" control points. Does `AddConformalArcBeam` take a `cpCount` parameter, or does Eclipse determine it? The sandbox should use whatever the API supports; the test only cares about jaw propagation, not about the number of CPs. **Status: OPEN — verify against ESAPI Online Help or empirically.**

2. ~~Does `ConvertDoseLevelToStructure` on a reused structure overwrite or append?~~ **RESOLVED in v2.** The sandbox no longer depends on the answer. §4 Step 4 and §4 Multi-iteration variant now explicitly clear temp structure geometry before reuse. The overwrite/append behavior is still empirically unknown but is no longer a design dependency.

3. **Does `RemoveBeam` throw if the beam has already been removed?** The cleanup path may call `RemoveBeam` on a beam that was already removed during undo. The sandbox guards with a `plan.Beams.Any(b => string.Equals(b.Id, beamId, StringComparison.OrdinalIgnoreCase))` check before calling `RemoveBeam`. **Status: OPEN — verify empirically. The guard is in place regardless.**

4. ~~Jaw position convention for HBB-Superior.~~ **ELEVATED into the A.4.3 test design in v2.** §5 Subtest A now tests both Y1 = 0.0 and Y2 = 0.0 and reports which convention produces the expected clipping. The sandbox will resolve this empirically rather than depending on a pre-determined answer.

5. ~~`SaveModifications` timing.~~ **RESOLVED in v2.** The sandbox uses Option B: restore state, do NOT call `SaveModifications`, and close the patient via `ClosePatient` which discards all unsaved modifications. This is confirmed by Varian's standalone executable guidance: `SaveModifications` writes changes to the database; `ClosePatient` discards them. The sandbox is a read-modify-restore-close cycle that leaves the patient unchanged. This is safe for repeated runs on the same patient.

---

## 12. What Success Looks Like

**A.7 passes** if all 12 steps complete without hard failure AND the multi-iteration variant demonstrates measurable hotspot reduction across at least 2 iterations before plateau.

**A.4.3 passes** if Subtest A (beam-level `SetJawPositions`) demonstrates that all control points on the DCA beam inherit the jaw position within tolerance. If Subtest A fails but Subtest B (per-CP jaw edits) passes, A.4.3 is a **partial pass** — the per-CP fallback path works but the recipe must use the more complex per-CP loop instead of the simpler beam-level broadcast. The summary table in §5 documents all three outcome combinations.

**Phase 0 is complete** when both tests pass (or A.4.3 partially passes with a documented fallback) on a representative T-spine phantom case. The team can then proceed to Phase 1 primitive implementation with confidence that the underlying ESAPI mechanics work as v0.4.2 assumed.

**The sandbox is not disposable.** Its portable classes become the starting point for the Phase 4 FIF kernel, and the test report format becomes the template for regression tests that verify the kernel still works after refactoring.
