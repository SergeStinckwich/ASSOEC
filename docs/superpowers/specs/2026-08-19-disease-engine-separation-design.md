# Separating the simulation engine from disease specifics

**Date:** 2026-08-19
**Repo:** SergeStinckwich/ASSOEC (`simulation_model/`)
**Goal:** Ebola now, extensible later — a clean seam between the disease-agnostic
simulation engine and the disease-specific logic, with only Ebola wired up, but
such that adding a future disease never requires editing engine code.

---

## 1. Motivation — the coupling today

The model already contains a genuinely reusable core and a partial Ebola
migration, but the seam between them is leaky. Concretely:

- **`disease/disease_model.nls` is supposed to be the generic layer, but it isn't.**
  Its predicates (`is-hospitalised-infection-state?`, `is-severe-infection-state?`,
  `is-infected-infection-status?`, `is-observing-critical-symptoms?`,
  `has-internally-visible-symptoms?`) hardcode literal state strings such as
  `"hospitalised"`, `"infected-stage-2"`, `"safe-burial"`. The "generic" layer
  therefore secretly knows the disease's concrete state names.

- **String dispatch on `disease-fsm-model` / `contagion-model`.** Live branches
  like `if disease-fsm-model = "ebola"` (with dead `"oxford"`/`"assocc"` branches
  commented out) appear in `disease_model.nls`, `contagion/contagion.nls`, and the
  stale `contagion/disease_model_covid.nls`. This is switch-on-type: adding or
  changing a disease means hunting these branches down.

- **Duplicated state lists breed silent bugs.** The same set of states is
  re-typed across five predicates. `is-severe-infection-state?` checks
  `"hospitalized"` and `"non-hospitalized"` (US spelling) while the rest of the
  codebase and the profiles use `"hospitalised"` / `"non-hospitalised"` (UK
  spelling). Those branches therefore **never match** — a latent correctness bug
  that exists precisely because the list is duplicated instead of declared once.

- **Duplication across files.** `contagion/disease_model_covid.nls` is a stale
  near-copy of `disease/disease_model.nls` — two files, one responsibility,
  drifting apart.

- **`contagion.nls` hardcodes state names too.** `is-contagious?` and `is-alive?`
  test literal `"just-contaminated"`, `"removed"`, `"safe-burial"` instead of
  asking the disease.

The good news: the seam already exists *conceptually*. `utils/stochastic_fsm.nls`
is a clean, disease-agnostic FSM engine; `disease/ebola_disease_model.nls` is a
pure FSM transition function; `disease/ebola_profiles.nls` maps abstract roles to
concrete state strings. The refactor makes the engine talk to diseases **only**
through a defined interface, and removes the string dispatch.

---

## 2. The disease interface (the seam)

### 2.1 Property vocabulary (the abstract disease ontology)

The engine understands diseases through a **fixed vocabulary of state
properties**. A disease never identifies itself to the engine; it only tags each
of its concrete states with terms from this vocabulary:

```
susceptible · exposed · infectious · severe · hospitalised ·
critical-symptoms · visible-symptoms · immune ·
dead · buried-safe · buried-unsafe · removed
```

This vocabulary *is* the contract. Engine predicates are defined in terms of these
tags; a disease is "understood" once its states are tagged with them.

### 2.2 Required disease reporters

A disease module must provide exactly these six things, under canonical names (no
disease-specific prefix). Everything else the disease needs (tau values, `p-hosp`,
`CFR`, the transition graph) stays private to the module.

| Canonical reporter | Replaces today | Purpose |
|---|---|---|
| `disease-initial-status` | inline `just-exposed-infection-status` in `set-contaminated-disease-status` | state assigned on contamination |
| `disease-next-state [s]` | `next-state-ebo` | FSM transition; wraps `stochastic-transition-apply` |
| `disease-state-properties [s]` | *new* | the tag set for a concrete state |
| `disease-relative-infectiousness [s]` | `ebo-relative-infection-rate-of-infection-status` | number in [0,1] |
| `disease-contagion-factor [infector susceptible context]` | `ebola-contagion-factor-between` | per-contact contamination risk |
| profile role reporters | `disease/ebola_profiles.nls` (already exist) | abstract role → concrete state string |

### 2.3 The state-property table (per disease)

Each disease declares every state once, in a table. This is the single source of
truth for state semantics and eliminates the duplicated predicate lists.

```netlogo
;; ebola_properties.nls — the ONLY place Ebola state semantics live
to build-disease-state-property-table
  set disease-state-property-table table:make
  declare-state "healthy"           ["susceptible"]
  declare-state "just-contaminated" ["exposed"]
  declare-state "infected-stage-1"  ["infectious"]
  declare-state "infected-stage-1.5"["infectious"]
  declare-state "infected-stage-2"  ["infectious" "severe" "critical-symptoms" "visible-symptoms"]
  declare-state "hospitalised"      ["infectious" "severe" "hospitalised" "critical-symptoms" "visible-symptoms"]
  declare-state "non-hospitalised"  ["infectious" "severe" "critical-symptoms"]
  declare-state "h-to-immune"       ["infectious" "hospitalised"]
  declare-state "dead-infectious"   ["infectious" "dead"]
  declare-state "safe-burial"       ["dead" "buried-safe"]
  declare-state "unsafe-burial"     ["infectious" "dead" "buried-unsafe"]
  declare-state "immune"            ["infectious" "immune"]   ; preserves current 0.5 relative rate
  declare-state "removed"           ["removed"]
end

to declare-state [name tags]
  table:put disease-state-property-table name tags
end
```

> **Note on `"immune"`:** today `ebo-relative-infection-rate-of-infection-status`
> returns `0.5` for the immune state and the predicates treat `"immune"` as
> infected. That is preserved above by tagging `immune` as `infectious` so
> behavior is unchanged by the refactor. Whether immune agents should really be
> contagious is a **modeling question left for a follow-up**, deliberately out of
> scope here so the refactor stays behavior-preserving.

`disease-relative-infectiousness` stays a small explicit reporter in the disease
module (it returns graded numbers like `0.734`, not booleans), reading the same
state names. It is not folded into the boolean table.

---

## 3. Engine-side predicates (rewritten against the interface)

Timed-state unwrapping is centralized in **one** helper instead of repeated in
every predicate (`if is-timed-state? s [...]` appears five times today):

```netlogo
to-report concrete-state-of [s]
  ifelse is-timed-state? s [ report state-of-timed-state s ] [ report s ]
end

to-report state-has? [s tag]
  report member? tag disease-state-properties concrete-state-of s
end
```

Every engine predicate then collapses to a one-liner reading the table:

```netlogo
to-report is-infected?                 report state-has? infection-state "infectious" end
to-report is-hospitalised-infection-state? [s]  report state-has? s "hospitalised" end
to-report is-severe-infection-state? [s]        report state-has? s "severe" end
to-report is-observing-critical-symptoms?       report state-has? infection-state "critical-symptoms" end
to-report has-internally-visible-symptoms?      report state-has? infection-state "visible-symptoms" end
to-report is-immune?      report state-has? infection-state "immune" end
to-report is-susceptible? report state-has? infection-state "susceptible" end
```

And in `contagion.nls`:

```netlogo
to-report is-contagious?  report state-has? infection-status "infectious" end
to-report is-alive?
  report not state-has? infection-status "removed"
     and not state-has? infection-status "buried-safe"
     and not state-has? infection-status "buried-unsafe"
end
to-report risks-of-contamination [infector susceptible context]
  report disease-contagion-factor infector susceptible context   ; no string dispatch
end
```

Note this rewrite *fixes* the `"hospitalized"`/`"non-hospitalized"` typo bug as a
side effect, because "severe" is now declared once in the table rather than
re-typed.

### 3.1 Behavior-preservation check

`is-infected-infection-status?` today is subtle: it reports `true` for `immune`
and `h-to-immune` and `unsafe-burial`, and `false` for `safe-burial`. The table
tags above reproduce this exactly:
- `immune`, `h-to-immune`, `unsafe-burial`, `dead-infectious` → tagged
  `infectious` → `is-infected?` true. ✅
- `safe-burial` → not `infectious` → false. ✅
- `just-contaminated` (exposed) → not `infectious` → false, matching today. ✅

The `is-infected?` engine metric must key off the `"infectious"` tag (not a
separate "is this the infected set" list) to preserve current counts. This
equivalence must be verified state-by-state during implementation (see §6).

---

## 4. File layout

The seam becomes a directory boundary. Engine files know the *vocabulary and the
six reporters*; disease files know *Ebola*.

```
simulation_model/
  disease/
    disease_engine.nls            # RENAMED from disease_model.nls. 100% disease-agnostic:
                                   #   metrics globals, kill-person, update loop,
                                   #   concrete-state-of / state-has?, all predicates.
                                   #   Documents the property vocabulary (the contract).
    active_disease.nls            # NEW. ONLY an __includes list selecting the active disease.
                                   #   Swapping disease = edit this one line.
    diseases/
      ebola/
        ebola_profiles.nls        # role -> concrete-state strings (exists)
        ebola_disease_model.nls   # transition FSM + tau/p reporters (exists)
        ebola_properties.nls      # NEW: state-property table + relative-infectiousness
        ebola_contagion_factor.nls# MOVED from contagion/ (Ebola-specific)
      _template/
        README.md                 # "to add a disease: implement these 6 reporters + table"
      _archive/
        # oxford / assocc remnants + old disease_model_covid.nls, kept for reference
  contagion/
    contagion.nls                 # disease-agnostic spread mechanics; calls disease-contagion-factor
  utils/
    stochastic_fsm.nls            # unchanged — the true engine core
```

**Disease selection is compile-time**, which is honest for NetLogo (it cannot
conditionally `__includes` files at runtime). `active_disease.nls` is the single
switch. There is no `if disease-fsm-model = "..."` anywhere in engine code.

---

## 5. Engine rewrite checklist

1. Rename `disease/disease_model.nls` → `disease/disease_engine.nls`.
2. Replace hardcoded state-string lists in the five predicates with `state-has?`
   one-liners (§3).
3. Remove `if disease-fsm-model = "ebola"` dispatch from
   `set-contaminated-disease-status`, `is-immune?`, `is-susceptible?`,
   `next-infection-state`; call the canonical `disease-*` reporters directly.
4. Delete `contagion/disease_model_covid.nls` (stale duplicate; archive a copy).
5. `contagion.nls`: `risks-of-contamination` → `disease-contagion-factor`;
   `is-contagious?` / `is-alive?` → `state-has?`.
6. Move Ebola files under `disease/diseases/ebola/`; move oxford/assocc remnants
   under `disease/diseases/_archive/`.
7. Add `disease/active_disease.nls` and update the top-level `__includes` in
   `covid-sim.nlogo` / `covid-sim.nlogox` to pull in `disease_engine.nls`,
   `active_disease.nls`, and the contagion engine, instead of the current
   disease-specific includes.

### Out of scope (YAGNI, noted for later)

- **Runtime plugin registry** (Approach C): compiling all diseases in and
  dispatching via anonymous-procedure tables for runtime switching. Not needed for
  "Ebola now."
- **GUI choosers.** `disease-fsm-model` and `contagion-model` widgets in the
  `.nlogo`/`.nlogox` become vestigial. Editing NetLogo GUI XML by hand is
  error-prone; leave the widgets in place, stop reading them in code, and clean up
  the GUI in a separate pass.
- **The `immune`-is-contagious modeling question** (see §2.3).

---

## 6. Sequencing (each step compiles and runs)

The model is large and has no automated test suite, so every step is additive or
behavior-preserving and independently runnable in NetLogo.

1. **Add, don't replace.** Add `ebola_properties.nls` (table + `declare-state`)
   and the `concrete-state-of` / `state-has?` helpers. Nothing calls them yet;
   the model still runs unchanged.
2. **Swap predicates one at a time.** Rewrite each predicate to use `state-has?`,
   keeping a way to compare. Validate equivalence with the state-by-state check
   in §3.1 (enumerate every state the Ebola FSM can produce and assert old vs new
   predicate agree). This is where the `"hospitalized"` typo fix is confirmed.
3. **Introduce canonical reporters as aliases.** Define `disease-next-state`,
   `disease-contagion-factor`, `disease-initial-status`,
   `disease-relative-infectiousness` as thin wrappers over the existing
   `*-ebo` / `ebola-*` reporters. Switch engine call sites to the canonical names.
   Then inline-rename the originals and drop the aliases.
4. **Remove string dispatch and the duplicate file.** Delete the
   `= "ebola"/"oxford"/"assocc"` branches; delete `disease_model_covid.nls`.
5. **Reorganize files** into the §4 layout; add `active_disease.nls`; archive
   COVID remnants; update the top-level `__includes`.

### Validation at each step

There is no test harness, so validation is: (a) the model loads without a compile
error in NetLogo, and (b) a short fixed-seed run produces the same epidemic curve
(`#infected`, `#hospitalised`, `#dead-people`, `#safely-buried`,
`#unsafely-buried` over N ticks) before and after the step. Capturing one
reference run at the start gives a regression baseline for the whole refactor.

---

## 7. What success looks like

- `disease_engine.nls` and `contagion.nls` contain **zero** concrete state-name
  string literals and **zero** `= "<disease>"` comparisons.
- Adding a hypothetical second disease requires creating one folder under
  `diseases/` with a profiles file, a transition function, a property table, a
  relative-infectiousness reporter, and a contagion-factor reporter — and editing
  the single `active_disease.nls` include line. No engine file is touched.
- The `"hospitalized"`/`"non-hospitalised"` spelling bug is gone because "severe"
  is declared once.
