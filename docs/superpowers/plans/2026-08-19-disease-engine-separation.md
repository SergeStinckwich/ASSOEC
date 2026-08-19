# Disease/Engine Separation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Separate the disease-agnostic simulation engine from Ebola-specific logic so the engine depends only on a fixed interface, and adding a future disease never edits engine code.

**Architecture:** The engine understands diseases through a fixed *property vocabulary* (`susceptible`, `infectious`, `severe`, `hospitalised`, `dead`, `removed`, …). Each disease declares its concrete states in a per-disease *state-property table* and provides six canonical reporters (`disease-next-state`, `disease-contagion-factor`, `disease-initial-status`, `disease-relative-infectiousness`, `disease-state-properties`, plus profile role reporters). Engine predicates become one-line table lookups; all `if disease-fsm-model = "…"` string dispatch is removed. Disease selection is compile-time via a single `active_disease.nls` include.

**Tech Stack:** NetLogo 7.0.4 (`covid-sim.nlogox` is the active model), `.nls` includes, the `table` extension (already loaded), NetLogo BehaviorSpace/headless for CI.

**Reference spec:** `docs/superpowers/specs/2026-08-19-disease-engine-separation-design.md`

---

## Verification model (read first)

NetLogo has no unit-test framework and `validation/testing.nls` is fully commented out. This plan adds a tiny in-model test runner and uses two verification methods throughout:

- **Predicate tests (primary):** Open `simulation_model/covid-sim.nlogox` in NetLogo 7, then in the **Command Center** type:
  ```
  run-disease-engine-tests
  ```
  Expected on success: `disease-engine tests: ALL PASS (NN checks)`. On failure a check prints `FAIL: …` and the run aborts with `error`. Because the whole model compiles before the Command Center is usable, a green run also proves every `.nls` still compiles.

- **Whole-model regression (per the spec §6):** a fixed-seed short run capturing five metrics (`#infected`, `#hospitalised`, `#dead-people`, `#safely-buried`, `#unsafely-buried`). Captured once as a baseline in Task 1, re-checked in Task 8.

The tests are *characterization tests of the intended behavior* defined by the property table in the spec. The pre-refactor code is expected to FAIL them (that is the "red"), because it contains the known `"hospitalized"`/`"non-hospitalised"` spelling bug. Each refactor step drives them back to green.

> **Two model files:** All edits target `simulation_model/covid-sim.nlogox` (the active Ebola v7 file). `covid-sim.nlogo` (v6, legacy COVID) is out of scope — do not edit it. Its stale `__includes` differs and is handled in Task 7.

---

## File structure

Created:
- `simulation_model/validation/disease_engine_tests.nls` — the test runner (Task 1)
- `simulation_model/disease/diseases/ebola/ebola_properties.nls` — state-property table + relative-infectiousness (Task 2)
- `simulation_model/disease/active_disease.nls` — the single disease-selection include (Task 7)
- `simulation_model/disease/diseases/ebola/README.md` + `_template/README.md` — the interface contract (Task 7)

Renamed:
- `simulation_model/disease/disease_model.nls` → `simulation_model/disease/disease_engine.nls` (Task 7)

Moved (Task 7):
- `disease/ebola_profiles.nls`, `disease/ebola_disease_model.nls`, `contagion/ebola_contagion_factor.nls` → `disease/diseases/ebola/`
- `contagion/disease_model_covid.nls`, oxford/assocc remnants → `disease/diseases/_archive/`

Modified:
- `simulation_model/environment_dynamics.nls:1` (include list — Tasks 1, 7)
- `simulation_model/disease/disease_model.nls` (predicates + dispatch — Tasks 3, 4)
- `simulation_model/contagion/contagion.nls` (contagion seam — Task 5)

---

## Task 1: Test runner + regression baseline

**Files:**
- Create: `simulation_model/validation/disease_engine_tests.nls`
- Modify: `simulation_model/environment_dynamics.nls:1` (add the test file to includes)

- [ ] **Step 1: Create the test runner**

Create `simulation_model/validation/disease_engine_tests.nls` with the expected-behavior truth table taken from spec §3.1. This file references only reporters that will exist by the end of Task 3 (`is-hospitalised-infection-state?`, `is-severe-infection-state?`, `is-infected-infection-status?`) plus new ones from Task 2/3 (`state-has?`). To let Task 1 compile *now*, the truth-table checks that need `state-has?` are written but the runner is guarded so it can be extended incrementally.

```netlogo
;; disease_engine_tests.nls — characterization tests for the disease/engine seam.
;; Run from the Command Center:  run-disease-engine-tests

globals [ #disease-engine-checks ]

to assert-equal [actual expected msg]
  set #disease-engine-checks #disease-engine-checks + 1
  if actual != expected [
    print (word "FAIL: " msg " — expected " expected " got " actual)
    error (word "disease-engine test FAILED: " msg)
  ]
end

;; every concrete Ebola state the FSM can occupy
to-report all-ebola-states
  report (list
    "healthy" "just-contaminated" "infected-stage-1" "infected-stage-1.5"
    "infected-stage-2" "hospitalised" "non-hospitalised" "h-to-immune"
    "dead-infectious" "safe-burial" "unsafe-burial" "immune" "removed")
end

to run-disease-engine-tests
  set #disease-engine-checks 0
  test-property-predicates
  print (word "disease-engine tests: ALL PASS (" #disease-engine-checks " checks)")
end
```

- [ ] **Step 2: Add the characterization checks**

Append the `test-property-predicates` procedure encoding the intended truth values (from spec §3.1). Each row asserts the four property predicates for one state.

```netlogo
;; expected: [ state  infected?  hospitalised?  severe?  critical-symptoms? ]
to test-property-predicates
  foreach (list
    ;  state                infected  hosp   severe  critical
    (list "healthy"           false   false  false   false)
    (list "just-contaminated" false   false  false   false)
    (list "infected-stage-1"  true    false  false   false)
    (list "infected-stage-1.5" true   false  false   false)
    (list "infected-stage-2"  true    false  true    true)
    (list "hospitalised"      true    true   true    true)
    (list "non-hospitalised"  true    false  true    true)
    (list "h-to-immune"       true    true   false   false)
    (list "dead-infectious"   true    false  false   false)
    (list "safe-burial"       false   false  false   false)
    (list "unsafe-burial"     true    false  false   false)
    (list "immune"            true    false  false   false)
    (list "removed"           false   false  false   false)
  ) [ row ->
    let s        item 0 row
    assert-equal (is-infected-infection-status? s)  (item 1 row) (word "is-infected? " s)
    assert-equal (is-hospitalised-infection-state? s) (item 2 row) (word "is-hospitalised? " s)
    assert-equal (is-severe-infection-state? s)     (item 3 row) (word "is-severe? " s)
  ]
end
```

> Note: `critical-symptoms` and `visible-symptoms` are exercised in Task 3 Step 4 once those predicates take a state argument; for now the table documents the intended values in the comment.

- [ ] **Step 3: Register the test file in the include chain**

In `simulation_model/environment_dynamics.nls` line 1, add `"validation/disease_engine_tests.nls"` to the `__includes` list. Current line:

```netlogo
 __includes ["contagion/contagion.nls" "gathering_points.nls" "public_measures/public_measures.nls" "general_environment_dynamics.nls" "activity_model.nls"  "disease/disease_model.nls" "economy_model.nls"]
```

New line:

```netlogo
 __includes ["contagion/contagion.nls" "gathering_points.nls" "public_measures/public_measures.nls" "general_environment_dynamics.nls" "activity_model.nls"  "disease/disease_model.nls" "economy_model.nls" "validation/disease_engine_tests.nls"]
```

- [ ] **Step 4: Verify it compiles and the test is RED**

Open `simulation_model/covid-sim.nlogox` in NetLogo 7. In the Command Center run:

```
run-disease-engine-tests
```

Expected: a `FAIL:` line and an aborting error — most likely `FAIL: is-severe? hospitalised` or an `error "not implemented for: hospitalised"` thrown from the current `is-severe-infection-state?` (the `"hospitalized"` typo bug). This confirms the tests exercise real behavior and the current code does not satisfy the intended contract.

- [ ] **Step 5: Capture the regression baseline**

In NetLogo, with a fixed seed, run a short deterministic scenario and record the five metrics. In the Command Center:

```
random-seed 42
setup-ebola
repeat 200 [ go ]
print (list ticks #infected #hospitalised #dead-people #safely-buried #unsafely-buried)
```

Copy the printed list into a new file `simulation_model/validation/disease_engine_baseline.txt` (one line, plus a comment noting seed=42, 200 ticks, and today's date). This is the regression oracle for Task 8.

- [ ] **Step 6: Commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git add simulation_model/validation/disease_engine_tests.nls simulation_model/validation/disease_engine_baseline.txt simulation_model/environment_dynamics.nls
git commit -m "test: add disease-engine characterization tests + regression baseline"
```

---

## Task 2: State-property table + lookup helpers (additive)

**Files:**
- Create: `simulation_model/disease/diseases/ebola/ebola_properties.nls`
- Modify: `simulation_model/disease/disease_model.nls` (declare the table global, include the new file, add helpers)

This task is purely additive — nothing calls the new helpers yet, so the model behaves identically and the Task 1 test stays RED.

- [ ] **Step 1: Create the property table file**

Create `simulation_model/disease/diseases/ebola/ebola_properties.nls`:

```netlogo
;; ebola_properties.nls — the ONLY place Ebola state semantics live.
;; Tags come from the engine's fixed property vocabulary (see disease_engine).

to build-disease-state-property-table
  set disease-state-property-table table:make
  declare-state "healthy"            ["susceptible"]
  declare-state "just-contaminated"  ["exposed"]
  declare-state "infected-stage-1"   ["infectious"]
  declare-state "infected-stage-1.5" ["infectious"]
  declare-state "infected-stage-2"   ["infectious" "severe" "critical-symptoms" "visible-symptoms"]
  declare-state "hospitalised"       ["infectious" "severe" "hospitalised" "critical-symptoms" "visible-symptoms"]
  declare-state "non-hospitalised"   ["infectious" "severe" "critical-symptoms"]
  declare-state "h-to-immune"        ["infectious" "hospitalised"]
  declare-state "dead-infectious"    ["infectious" "dead"]
  declare-state "safe-burial"        ["dead" "buried-safe"]
  declare-state "unsafe-burial"      ["infectious" "dead" "buried-unsafe"]
  declare-state "immune"             ["infectious" "immune"]
  declare-state "removed"            ["removed"]
end

to declare-state [name tags]
  table:put disease-state-property-table name tags
end

;; canonical disease reporter: the tag list for a concrete state
to-report disease-state-properties [s]
  report table:get disease-state-property-table s
end
```

- [ ] **Step 2: Declare the table global and helpers in the engine**

In `simulation_model/disease/disease_model.nls`, add `disease-state-property-table` to the `globals [ … ]` block (lines 4-17), and add the include + two helper reporters near the top (after line 1's `__includes`). Add to the `__includes` on line 1:

```netlogo
__includes [ "disease/ebola_profiles.nls" "disease/ebola_disease_model.nls" "disease/diseases/ebola/ebola_properties.nls"]
```

Add the global (inside the existing `globals` block):

```netlogo
  disease-state-property-table
```

Add the helpers (place them just above `to-report is-hospitalised-infection-state?`):

```netlogo
;; unwrap a timed-state [state time] down to its concrete state string
to-report concrete-state-of [s]
  ifelse is-timed-state? s [ report state-of-timed-state s ] [ report s ]
end

;; does state s carry property tag `tag` from the disease's property table?
to-report state-has? [s tag]
  report member? tag disease-state-properties concrete-state-of s
end
```

- [ ] **Step 3: Build the table during setup**

The table must be populated before any predicate reads it. In `simulation_model/setup/setup-ebola.nls`, inside `to setup-ebola`, add a call to `build-disease-state-property-table` immediately after `reset-ticks` (line ~10). Add this line:

```netlogo
  build-disease-state-property-table
```

Also call it at the top of `run-disease-engine-tests` so the tests work without a full setup. Edit `simulation_model/validation/disease_engine_tests.nls`, in `run-disease-engine-tests`, add as the first line of the body:

```netlogo
  build-disease-state-property-table
```

- [ ] **Step 4: Verify it still compiles (behavior unchanged, test still RED)**

Open `covid-sim.nlogox`. Command Center:

```
run-disease-engine-tests
```

Expected: still fails at a predicate assertion (the predicates are not yet rewritten), but NO compile error and NO "table not found" error — proving the table builds and `state-has?` is callable. Optionally verify directly:

```
build-disease-state-property-table print state-has? "hospitalised" "severe"
```

Expected: `true`.

- [ ] **Step 5: Commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git add simulation_model/disease/diseases/ebola/ebola_properties.nls simulation_model/disease/disease_model.nls simulation_model/setup/setup-ebola.nls simulation_model/validation/disease_engine_tests.nls
git commit -m "feat: add Ebola state-property table and state-has? lookup helpers"
```

---

## Task 3: Rewrite classification predicates to use the table (GREEN)

**Files:**
- Modify: `simulation_model/disease/disease_model.nls` (five predicates)
- Modify: `simulation_model/validation/disease_engine_tests.nls` (add symptom checks)

- [ ] **Step 1: Replace the five predicate bodies**

In `simulation_model/disease/disease_model.nls`, replace the bodies of these predicates. Full replacements:

```netlogo
to-report is-hospitalised-infection-state? [s]
  report state-has? s "hospitalised"
end

to-report is-severe-infection-state? [s]
  report state-has? s "severe"
end

to-report is-infected-infection-status? [s]
  report state-has? s "infectious"
end

to-report is-observing-critical-symptoms?
  report state-has? infection-status "critical-symptoms"
end

to-report has-internally-visible-symptoms?
  report state-has? infection-status "visible-symptoms"
end
```

Leave `is-infected?` (line 183) as-is — it already delegates to `is-infected-infection-status? infection-status`, which now reads the table.

- [ ] **Step 2: Extend the tests with symptom predicates**

In `simulation_model/validation/disease_engine_tests.nls`, add a fifth column check for critical symptoms inside `test-property-predicates` (after the `is-severe` assert), using the same rows:

```netlogo
    assert-equal (state-has? s "critical-symptoms") (item 4 row) (word "critical? " s)
```

- [ ] **Step 3: Verify GREEN**

Open `covid-sim.nlogox`. Command Center:

```
run-disease-engine-tests
```

Expected: `disease-engine tests: ALL PASS (NN checks)`. The `"hospitalized"` typo bug is now gone because "severe" is declared once in the table.

- [ ] **Step 4: Commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git add simulation_model/disease/disease_model.nls simulation_model/validation/disease_engine_tests.nls
git commit -m "refactor: drive engine classification predicates from property table"
```

---

## Task 4: Canonical disease reporters + remove string dispatch

**Files:**
- Modify: `simulation_model/disease/disease_model.nls` (dispatch sites)
- Modify: `simulation_model/disease/ebola_disease_model.nls` (expose canonical name)

- [ ] **Step 1: Add canonical reporters as thin aliases**

In `simulation_model/disease/ebola_disease_model.nls`, add canonical wrappers at the end (keeping the Ebola-specific `next-state-ebo`/`tau`/`p` internals private):

```netlogo
;; ---- canonical disease interface (engine calls only these) ----
to-report disease-next-state [s]
  report next-state-ebo s
end

to-report disease-initial-status
  report just-exposed-infection-status
end

to-report disease-relative-infectiousness [s]
  report ebo-relative-infection-rate-of-infection-status s
end
```

- [ ] **Step 2: Remove `disease-fsm-model` dispatch from the engine**

In `simulation_model/disease/disease_model.nls`, replace these four procedures. Full replacements:

```netlogo
to set-contaminated-disease-status
  set-infection-status disease-initial-status
  set time-when-infected ticks
end

to-report is-immune?
  report state-has? infection-status "immune"
end

to-report is-susceptible?
  report state-has? infection-state "susceptible"
end

to-report next-infection-state
  report disease-next-state infection-state
end
```

This deletes the commented `assocc`/`oxford` branches, the live `if disease-fsm-model = "ebola"` guards, and the dead `error "unimplemented"` fall-through in `next-infection-state`.

- [ ] **Step 3: Add susceptible/immune checks to the tests**

In `simulation_model/validation/disease_engine_tests.nls`, add a small procedure and call it from `run-disease-engine-tests` (after `test-property-predicates`):

```netlogo
to test-susceptible-immune
  assert-equal (state-has? "healthy" "susceptible") true  "susceptible healthy"
  assert-equal (state-has? "immune"  "immune")      true  "immune immune"
  assert-equal (state-has? "healthy" "immune")      false "healthy not immune"
end
```

Add `test-susceptible-immune` to the body of `run-disease-engine-tests`.

- [ ] **Step 4: Verify GREEN + no dispatch remains**

Command Center in `covid-sim.nlogox`:

```
run-disease-engine-tests
```

Expected: `ALL PASS`. Then confirm the string dispatch is gone from the engine file:

```bash
cd /Users/stinckwich/Projects/ASSOEC
grep -n "disease-fsm-model" simulation_model/disease/disease_model.nls
```

Expected: no output.

- [ ] **Step 5: Commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git add simulation_model/disease/disease_model.nls simulation_model/disease/ebola_disease_model.nls simulation_model/validation/disease_engine_tests.nls
git commit -m "refactor: replace disease-fsm-model string dispatch with canonical disease reporters"
```

---

## Task 5: Clean the contagion seam

**Files:**
- Modify: `simulation_model/contagion/contagion.nls`
- Modify: `simulation_model/contagion/ebola_contagion_factor.nls` (expose canonical name)

- [ ] **Step 1: Expose the canonical contagion reporter**

In `simulation_model/contagion/ebola_contagion_factor.nls`, add at the end (keeping the existing `ebola-contagion-factor-between` as the private implementation):

```netlogo
;; canonical disease interface
to-report disease-contagion-factor [infector susceptible context]
  report ebola-contagion-factor-between infector susceptible context
end
```

- [ ] **Step 2: Rewrite the three contagion-seam reporters**

In `simulation_model/contagion/contagion.nls`, replace `risks-of-contamination` (lines 119-122), `is-contagious?` (lines 191-193), and `is-alive?` (lines 195-197). Full replacements:

```netlogo
to-report risks-of-contamination [infector susceptible context]
  report disease-contagion-factor infector susceptible context
end
```

```netlogo
to-report is-contagious?
  report state-has? infection-status "infectious"
end

to-report is-alive?
  report not state-has? infection-status "removed"
     and not state-has? infection-status "buried-safe"
     and not state-has? infection-status "buried-unsafe"
end
```

This removes the `if contagion-model = "ebola"` dispatch and the hardcoded `"just-contaminated"`/`"removed"`/`"safe-burial"` literals.

- [ ] **Step 3: Add contagion checks to the tests**

In `simulation_model/validation/disease_engine_tests.nls`, add and register:

```netlogo
to test-contagion-predicates
  ;; is-contagious? == "infectious" tag (spec §2.3 preserves immune-as-contagious)
  assert-equal (member? "infectious" disease-state-properties "just-contaminated") false "exposed not contagious"
  assert-equal (member? "infectious" disease-state-properties "infected-stage-2")  true  "stage-2 contagious"
  assert-equal (member? "infectious" disease-state-properties "safe-burial")       false "safe-burial not contagious"
  assert-equal (member? "infectious" disease-state-properties "unsafe-burial")     true  "unsafe-burial contagious"
  ;; is-alive?
  assert-equal (member? "removed" disease-state-properties "removed")              true  "removed is removed"
  assert-equal (member? "buried-safe" disease-state-properties "safe-burial")      true  "safe-burial buried-safe"
end
```

Add `test-contagion-predicates` to `run-disease-engine-tests`.

- [ ] **Step 4: Verify GREEN + regression check**

Command Center in `covid-sim.nlogox`:

```
run-disease-engine-tests
```

Expected: `ALL PASS`. Then re-run the seeded regression and compare to the baseline from Task 1 Step 5:

```
random-seed 42
setup-ebola
repeat 200 [ go ]
print (list ticks #infected #hospitalised #dead-people #safely-buried #unsafely-buried)
```

Expected: identical to `simulation_model/validation/disease_engine_baseline.txt`. If it differs, STOP — a predicate's tag set does not match prior behavior; diff against spec §3.1 before continuing.

- [ ] **Step 5: Commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git add simulation_model/contagion/contagion.nls simulation_model/contagion/ebola_contagion_factor.nls simulation_model/validation/disease_engine_tests.nls
git commit -m "refactor: drive contagion seam through canonical disease-contagion-factor + property tags"
```

---

## Task 6: Delete the stale COVID duplicate

**Files:**
- Delete: `simulation_model/contagion/disease_model_covid.nls`

- [ ] **Step 1: Confirm it is not in the active include chain**

```bash
cd /Users/stinckwich/Projects/ASSOEC/simulation_model
grep -rn "disease_model_covid" . | grep -i includ
```

Expected: no output (it is only referenced by its own header line, never included).

- [ ] **Step 2: Delete it**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git rm simulation_model/contagion/disease_model_covid.nls
```

- [ ] **Step 3: Verify GREEN**

Open `covid-sim.nlogox`, Command Center:

```
run-disease-engine-tests
```

Expected: `ALL PASS` (deleting an unincluded file changes nothing).

- [ ] **Step 4: Commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git commit -m "chore: remove stale duplicate disease_model_covid.nls"
```

---

## Task 7: Reorganize into engine/disease directory layout

This task is pure file movement + include-path updates. Do it in one commit so the model is never left with a broken include.

**Files:**
- Rename: `disease/disease_model.nls` → `disease/disease_engine.nls`
- Create: `disease/active_disease.nls`, `disease/diseases/ebola/README.md`, `disease/diseases/_template/README.md`
- Move: ebola files into `disease/diseases/ebola/`; archive oxford/assocc remnants
- Modify: `environment_dynamics.nls`, `disease/disease_engine.nls`, `contagion/contagion.nls` include lines

- [ ] **Step 1: Create directories and move disease files**

```bash
cd /Users/stinckwich/Projects/ASSOEC/simulation_model
mkdir -p disease/diseases/ebola disease/diseases/_template disease/diseases/_archive
git mv disease/ebola_profiles.nls disease/diseases/ebola/ebola_profiles.nls
git mv disease/ebola_disease_model.nls disease/diseases/ebola/ebola_disease_model.nls
git mv contagion/ebola_contagion_factor.nls disease/diseases/ebola/ebola_contagion_factor.nls
git mv disease/disease_profiles.nls disease/diseases/_archive/disease_profiles.nls
```

(`disease/diseases/ebola/ebola_properties.nls` is already in place from Task 2.)

- [ ] **Step 2: Rename the engine file**

```bash
cd /Users/stinckwich/Projects/ASSOEC/simulation_model
git mv disease/disease_model.nls disease/disease_engine.nls
```

- [ ] **Step 3: Create `active_disease.nls` (the single selection point)**

Create `simulation_model/disease/active_disease.nls`:

```netlogo
;; active_disease.nls — SELECTS THE ACTIVE DISEASE.
;; To switch diseases, change only the include paths below to another
;; folder under disease/diseases/ that provides the six canonical reporters.
__includes [
  "disease/diseases/ebola/ebola_profiles.nls"
  "disease/diseases/ebola/ebola_disease_model.nls"
  "disease/diseases/ebola/ebola_properties.nls"
  "disease/diseases/ebola/ebola_contagion_factor.nls"
]
```

- [ ] **Step 4: Strip disease includes out of the engine and contagion files**

In `simulation_model/disease/disease_engine.nls`, replace the line-1 `__includes` (which currently pulls the ebola files directly) with an empty engine include:

```netlogo
;; disease_engine.nls — disease-AGNOSTIC. Understands only the property
;; vocabulary and the six canonical disease-* reporters. No disease files here.
```

(Delete the old `__includes [ "disease/ebola_profiles.nls" … ]` line entirely.)

In `simulation_model/contagion/contagion.nls`, replace line 1:

```netlogo
;; contagion.nls — disease-AGNOSTIC spread mechanics. Calls disease-contagion-factor.
```

(Delete the old `__includes [ "contagion/ebola_contagion_factor.nls"]` line.)

- [ ] **Step 5: Wire the new files into the top-level chain**

In `simulation_model/environment_dynamics.nls` line 1, rename `disease/disease_model.nls` → `disease/disease_engine.nls` and add `disease/active_disease.nls`. Final line:

```netlogo
 __includes ["contagion/contagion.nls" "gathering_points.nls" "public_measures/public_measures.nls" "general_environment_dynamics.nls" "activity_model.nls"  "disease/disease_engine.nls" "disease/active_disease.nls" "economy_model.nls" "validation/disease_engine_tests.nls"]
```

- [ ] **Step 6: Write the interface contract docs**

Create `simulation_model/disease/diseases/_template/README.md`:

```markdown
# Adding a disease

A disease is a folder under `disease/diseases/<name>/` providing:

1. `*_profiles.nls`   — role→concrete-state reporters (e.g. `just-exposed-infection-status`).
2. `*_disease_model.nls` — the FSM transition function + the canonical reporters
   `disease-next-state [s]`, `disease-initial-status`, `disease-relative-infectiousness [s]`.
3. `*_properties.nls` — `build-disease-state-property-table` tagging each concrete
   state with the engine vocabulary: susceptible, exposed, infectious, severe,
   hospitalised, critical-symptoms, visible-symptoms, immune, dead, buried-safe,
   buried-unsafe, removed. Also `disease-state-properties [s]`.
4. `*_contagion_factor.nls` — `disease-contagion-factor [infector susceptible context]`.

Then point `disease/active_disease.nls` at the new folder. No engine file changes.
```

Create `simulation_model/disease/diseases/ebola/README.md` with a one-line pointer: `Ebola implementation of the disease interface. See ../_template/README.md.`

- [ ] **Step 7: Verify GREEN after the move**

Open `covid-sim.nlogox` in NetLogo 7 (this fully recompiles all moved includes). Command Center:

```
run-disease-engine-tests
```

Expected: `ALL PASS`. Then a seeded regression check identical to baseline:

```
random-seed 42
setup-ebola
repeat 200 [ go ]
print (list ticks #infected #hospitalised #dead-people #safely-buried #unsafely-buried)
```

Expected: matches `disease_engine_baseline.txt`.

- [ ] **Step 8: Confirm the engine is literal-free**

```bash
cd /Users/stinckwich/Projects/ASSOEC/simulation_model
grep -nE '"(hospitalised|infected-stage|safe-burial|just-contaminated|removed|immune)"' disease/disease_engine.nls contagion/contagion.nls
grep -n "disease-fsm-model\|contagion-model" disease/disease_engine.nls contagion/contagion.nls
```

Expected: no output from either grep (all concrete state names and disease switches now live only in the disease folder).

- [ ] **Step 9: Commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git add -A
git commit -m "refactor: reorganize into disease_engine + diseases/ebola with single active_disease selector"
```

---

## Task 8: Legacy model + final regression sign-off

**Files:**
- Modify: `simulation_model/covid-sim.nlogo` (legacy v6 — include path only) OR document it as unsupported

- [ ] **Step 1: Decide the legacy v6 file's fate**

`covid-sim.nlogo` (v6) still references the old `disease_model.nls` path indirectly and `setup/setup.nls`. It is not part of the Ebola effort. Confirm the team's intent:

```bash
cd /Users/stinckwich/Projects/ASSOEC
git log -1 --format="%ci %an" -- simulation_model/covid-sim.nlogo
```

If it is dead (June 2026, COVID-only), add a top-of-Code-tab comment marking it legacy rather than editing includes. If it must keep loading, mirror the `environment_dynamics.nls` include change (already done there — the v6 file shares that include file, so no separate edit is needed unless it overrides includes). Verify by opening it in NetLogo; if it throws a missing-include error for `disease_model.nls`, it means it was pinned to the old name — in that case restore compatibility by leaving a one-line `disease/disease_engine.nls` include note. Record the decision in the commit message.

- [ ] **Step 2: Full regression, both seeds**

Open `covid-sim.nlogox`. Command Center, run two seeds and compare against the baseline pattern (seed 42 must match exactly; seed 7 is a sanity run):

```
random-seed 42
setup-ebola
repeat 200 [ go ]
print (list "seed42" #infected #hospitalised #dead-people #safely-buried #unsafely-buried)
random-seed 7
setup-ebola
repeat 200 [ go ]
print (list "seed7" #infected #hospitalised #dead-people #safely-buried #unsafely-buried)
```

Expected: seed42 line matches `disease_engine_baseline.txt`; seed7 line completes without error.

- [ ] **Step 3: Optional — headless CI experiment**

To run the predicate tests from the terminal (CI), create a BehaviorSpace experiment via NetLogo's GUI (Tools → BehaviorSpace → New), name it `disease-engine-tests`, set **setup commands** to `run-disease-engine-tests`, **go commands** to `stop`, **stop condition** to `true`, metrics empty, repetitions 1. Save the model, then:

```bash
"/Applications/NetLogo 7.0.4/netlogo-headless.sh" --model "/Users/stinckwich/Projects/ASSOEC/simulation_model/covid-sim.nlogox" --experiment disease-engine-tests
```

Expected: exit code 0 and no `FAILED` in output. A failing predicate aborts with a non-zero exit — usable as a CI gate.

- [ ] **Step 4: Final commit**

```bash
cd /Users/stinckwich/Projects/ASSOEC
git add -A
git commit -m "chore: finalize disease/engine separation; regression verified; mark legacy v6 model"
```

---

## Self-review notes (author)

- **Spec coverage:** §1 problems → fixed across Tasks 3-6; §2.1 vocabulary → Task 2 table + Task 7 template; §2.2 six reporters → Tasks 2/4/5; §2.3 state-property table → Task 2; §3 predicates → Task 3; §3.1 behavior-preservation → Task 1 truth table + Task 5/7 regression; §4 layout → Task 7; §5 checklist → Tasks 3-7; §6 sequencing → task order matches; §7 success criteria → Task 7 Step 8 greps + Task 8 regression.
- **Out-of-scope items** (runtime registry, GUI choosers, immune-as-contagious modeling) are intentionally not tasked, per spec §5.
- **Type/name consistency:** `disease-state-property-table`, `disease-state-properties`, `concrete-state-of`, `state-has?`, `disease-next-state`, `disease-initial-status`, `disease-relative-infectiousness`, `disease-contagion-factor`, `run-disease-engine-tests`, `build-disease-state-property-table` used consistently across all tasks.
