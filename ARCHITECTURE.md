# Plasmith — Architecture & Build Handoff

> **For a fresh Claude Code session.** This is a self-contained build brief for an
> open-source hackathon project ("Built with Claude: Life Sciences", Build track).
> It contains no confidential or sensitive material — it's routine molecular-cloning
> design tooling (the category SnapGene / Benchling / Biopython occupy).
>
> **All open questions are now decided — build against [`DECISIONS.md`](DECISIONS.md) and
> proceed.** Do not stop to ask the §9 UI questions or the §10 MVP cut; they are committed
> there. Only surface a question if a default turns out to be genuinely impossible. The Tm
> engine is respecified in [`docs/TM_SPEC.md`](docs/TM_SPEC.md); the definition-of-done and
> golden fixtures are in [`docs/ACCEPTANCE.md`](docs/ACCEPTANCE.md); scope limits are in
> [`docs/SCOPE.md`](docs/SCOPE.md). The domain logic is described plainly below so you pick
> correct libraries and algorithms.

---

## 1. What it is (one paragraph)

Plasmith is a local web app that turns *design intent + a lab's reagent context*
into an *order-ready, validated cloning plan*. The user either (A) imports a
template sequence and states a goal in plain language ("insert GFP as a C-terminal
fusion after this gene"), and Plasmith returns the DNA to order, the final plasmid
sequence, a validity check, and a bench protocol tuned to their polymerase and
reagents; or (B) uploads a construct they already designed and Plasmith audits it
for defects and explains why it might fail. The differentiator is a **trustworthy
validation layer** and a **polymerase-aware Tm/primer engine** — things generic
online calculators and existing editors don't do — driven by **lab-context
awareness**.

## 2. Design decisions already made

- **Form factor:** local web app. Recommended stack: **Python + Streamlit**
  (fastest to build, visual, novice-friendly, easy `streamlit run`). Alternative:
  FastAPI + a light frontend if more control is wanted later.
- **Hosts supported:** *E. coli* and mammalian (HEK/CHO).
- **Assembly methods:** Gibson, restriction–ligation, Golden Gate.
- **LLM:** Anthropic API (Claude) — used only in the bounded layer (§6).
- **MVP cut:** not yet decided — see §10.

## 3. Architecture at a glance (layers)

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENTATION  (Streamlit web UI)                            │
│   Lab Setup · Design wizard · Audit uploader · Results view  │
└───────────────┬─────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────┐
│  ORCHESTRATION  (pipeline runner per mode; session state)    │
└───────────────┬───────────────────────────────┬─────────────┘
                │                                 │
┌───────────────▼──────────────┐   ┌──────────────▼────────────┐
│  DOMAIN CORE (deterministic) │   │  LLM LAYER (bounded)      │
│  — the trust boundary        │   │  intent → structured spec │
│  io · edit · calc · validate │   │  rank/explain suggestions │
│  reference · simulate · design│   │  draft protocol prose     │
└───────────────┬──────────────┘   └──────────────┬────────────┘
                │                                 │
┌───────────────▼─────────────────────────────────▼───────────┐
│  PERSISTENCE (local store: lab profile, saved designs, files)│
├──────────────────────────────────────────────────────────────┤
│  REPORTING (order list · sequence export · plan · map · gel) │
└──────────────────────────────────────────────────────────────┘
```

**Rule of thumb that defines the whole design:** anything a bench scientist must
*trust* (melting-temperature math, reading-frame/stop checks, restriction-site
scanning, sequence assembly) lives in the **deterministic domain core** and is
unit-tested. The **LLM layer** only interprets free-text intent, ranks/explains
options drawn from deterministic lookups, and writes human-readable prose. The LLM
never *asserts a numeric result or a pass/fail verdict.*

## 4. Module / file structure (suggested)

```
plasmith/
  app/                      # Streamlit UI (presentation only)
    Home.py
    pages/
      1_Lab_Setup.py        # configure & persist the lab profile
      2_Design.py           # Mode A wizard
      3_Audit.py            # Mode B uploader
      4_Results.py          # tabs: order list / sequence / map / gel / report / protocol
  core/
    io/                     # parse & write sequence files (FASTA, GenBank)
    edit/                   # sequence operations: insert / replace / fuse; assembly
    calc/                   # Tm (polymerase-aware), GC, primer metrics
    validate/               # rule engine + individual checks (see §7)
    reference/              # catalogs: plasmids, codon-usage tables, part variants
    simulate/               # in-silico assembly + virtual gel + plasmid map
    design/                 # primer design; assembly-method selection
  llm/                      # bounded LLM functions (see §6)
  store/                    # local persistence (SQLite or JSON)
  report/                   # build outputs (order list, exports, plan)
  data/                     # bundled reference data shipped with the app
  tests/                    # unit tests for the deterministic core
  requirements.txt
  README.md
```

## 5. Core data model (entities)

- **LabProfile** — the reusable context: `enzymes[]` (restriction enzymes on
  hand), `plasmids[]` (owned plasmids/sequences, with uploaded files),
  `kits[]` (assembly kits), `strains[]` (cloning/expression strains),
  `polymerases[]` (drives the Tm engine).
- **SequenceRecord** — `id, name, sequence, features[], source_format`.
- **DesignRequest** — `template_ref, goal_text, sourcing_choice
  (synthesize_gblock | use_owned_plasmid), method (gibson|restriction|golden_gate)`.
- **EditSpec** — structured result of parsing the goal:
  `operation (insert|replace|fuse), target_locus, payload_part, orientation, position (5'|3')`.
- **Part** — `name, sequence, type, source`.
- **PrimerSet / OrderItem** — the buy list.
- **ValidationReport** — `findings[]`, each `{severity, location, message,
  suggested_fix}`.
- **CloningPlan** — `pcr_conditions, protocol_steps[], controls[], diagnostics[]`.

## 6. LLM layer (bounded — this is where the model is allowed to reason)

| Function | Input | Output | Guardrail |
|---|---|---|---|
| `parse_intent` | free-text goal + template features | structured **EditSpec** | schema-validated; user confirms before execution |
| `rank_alternatives` | a **deterministic** candidate list (e.g. variant catalog) | ranked list + rationale | never invents members not in the list |
| `draft_protocol` | the deterministic CloningPlan fields | readable protocol prose | numbers come from the plan, not the model |
| `explain_findings` | the deterministic ValidationReport | plain-language "why it might fail" | never adds/removes findings |

Everything else (Tm, GC, frame/stop, restriction scans, assembly) is deterministic
code in `core/` and is unit-tested. This split is the product's trust story.

## 7. The validation rule engine (the differentiator)

Design it as a **registry of pluggable checks**. Each check is a pure function:

```
check(construct, lab_context) -> list[Finding]      # Finding = {severity, location, message, suggested_fix}
```

A runner executes all registered checks and aggregates findings. New checks are
added independently without touching the runner. Checks to implement (grouped):

- **Reading-frame integrity:** premature/internal stop in a labelled ORF;
  frameshift vs a tag or downstream ORF; a labelled ORF that is actually
  frameshifted or truncated; missing/duplicated start or stop.
- **Copy-fidelity:** labelled ORF mismatches a reference sequence (the copy-error case).
- **Expression context:** ribosome-binding-site spacing from the start (bacterial);
  promoter position; Kozak context (mammalian).
- **Codon usage:** rare/unoptimized codons; low codon-adaptation index after optimization.
- **Alternatives:** align an ORF to a reference set and surface better-suited variants
  (deterministic catalog lookup; LLM only ranks/explains — see §6).
- **Cloning feasibility:** restriction-site **uniqueness** (a chosen enzyme that
  would cut at multiple sites, not one); internal sites that break the chosen
  method (e.g. Golden Gate); unintended repeats/homology that cause mis-assembly
  or mispriming; feature collisions (insert/cut landing in a resistance marker,
  origin, or other essential feature).
- **Primer QC:** high overall GC and high GC at the 3′ end; forward/reverse Tm
  mismatch for the chosen polymerase; multiple binding sites → recommend a
  touchdown gradient; primer self-priming / dimers / hairpins.

## 8. Suggested libraries

- **Biopython** — FASTA/GenBank parsing (`Bio.SeqIO`), restriction analysis
  (`Bio.Restriction`), Tm baseline (`Bio.SeqUtils.MeltingTemp`, incl.
  salt/Mg²⁺-corrected nearest-neighbour models), codon tables.
- **pydna** — in-silico assembly (Gibson, restriction/ligation), primer design,
  simulated PCR and gels.
- **dna_features_viewer** — plasmid/linear feature maps.
- **matplotlib** — virtual gel rendering.
- **Streamlit** — UI. **SQLite/JSON** — local store. **anthropic** — LLM layer.
- Polymerase-aware Tm: wrap Biopython's corrected nearest-neighbour Tm and add a
  per-polymerase annealing offset from a small config table (Q5, Phusion, Taq,
  KAPA HiFi, …) — this table is the honest core of the "real PCR conditions" claim.

## 9. UI questions to ask the user before building

- Single scrolling page vs a step wizard for Design mode?
- Lab Setup: entry forms, file upload for owned plasmids, or both? Where persisted
  (local SQLite vs a folder of files the user can see)?
- Results view: which tabs matter in v1 — order list, final sequence, plasmid map,
  virtual gel, validity report, protocol?
- Download formats to offer (GenBank / FASTA / plain text; plan as Markdown/PDF)?
- Show plasmid map + virtual gel in v1, or defer to v2?

## 10. Open build decisions (resolve with the user)

- **Reference data sources:** live Addgene access vs a curated catalog bundled in
  `data/`; codon-usage tables (e.g. Kazusa) bundled vs fetched; a variant reference
  set for the "suggest a better-suited variant" feature.
- **Polymerases** to ship first (pick 2–3 for the Tm table).
- **Output file formats** to prioritize.
- **The one-week MVP cut:** the full spec above is more than a week. A defensible
  MVP: Mode B (audit an uploaded construct) + a subset of §7 checks + the
  polymerase-aware Tm engine — it proves the "trustworthy validation" thesis with
  the least surface area. Confirm with the user.

## 11. Build order (suggested)

0. `core/models` **first** — freeze the data schemas (`Construct` with an explicit
   `topology` field, `Finding` with `severity` and a `ran|skipped|error` status,
   `Location`). Everything else builds against these. (See `DECISIONS.md` #9–#10.)
1. `core/io` (GenBank in/out via Biopython) + the **golden fixtures** from
   [`docs/ACCEPTANCE.md`](docs/ACCEPTANCE.md) (`fixture_good.gb`, `fixture_bad.gb`, …).
2. `core/validate` runner + the **plasmid-applicable** check set with unit tests —
   frame/stop, native-stop-removal, RE-uniqueness (topology-correct), feature-collision,
   unintended-repeat, Dam/Dcm-vs-strain. *(Note: the earlier draft listed primer-QC checks
   here — those belong to the separate Tm panel, not the Mode B plasmid audit. See
   `DECISIONS.md`.)*
3. `core/calc` polymerase-aware Tm per [`docs/TM_SPEC.md`](docs/TM_SPEC.md) + the standalone
   primer Tm panel (Q5 vs Taq), benchmarked against the primer vectors in `ACCEPTANCE.md`.
4. Streamlit **Audit** page end-to-end (upload → findings table → explanation) + the Tm panel.
5. `store` JSON lab profile + wire lab context (strain genotype) into the methylation check.
6. `llm/` layer last, behind the deterministic core, with a `FakeLLM` stub used in CI and as
   the live-demo fallback (`DECISIONS.md` #11).
7. **Deferred to roadmap (not v1):** Mode A design, `core/edit`/`core/simulate`, virtual gel
   — see [`docs/SCOPE.md`](docs/SCOPE.md).
