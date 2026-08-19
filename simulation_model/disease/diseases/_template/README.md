# Adding a disease

A disease is a folder under `disease/diseases/<name>/` providing:

1. `*_profiles.nls`   — role->concrete-state reporters (e.g. `just-exposed-infection-status`).
2. `*_disease_model.nls` — the FSM transition function plus the canonical reporters
   `disease-next-state [s]`, `disease-initial-status`, `disease-relative-infectiousness [s]`.
3. `*_properties.nls` — `build-disease-state-property-table` tagging each concrete state
   with the engine vocabulary: susceptible, exposed, infectious, severe, hospitalised,
   critical-symptoms, visible-symptoms, immune, dead, buried-safe, buried-unsafe, removed.
   Also `disease-state-properties [s]`.
4. `*_contagion_factor.nls` — `disease-contagion-factor [infector susceptible context]`.

Then point `disease/active_disease.nls` at the new folder. No engine file changes.
