# Plasmith — Committed Decisions

> **This file removes the single biggest blocker the design review found:** the original
> handoff told a fresh build session to *"ask the user the UI questions and agree an MVP
> cut before writing code,"* which stalls autonomous work. Every open question from
> `ARCHITECTURE.md §9–§10` and `DESIGN.md`'s Open Questions now has a **committed default**
> below. **Build against these and proceed** — only surface a question if a default turns
> out to be genuinely impossible.

## MVP cut

**The one-week MVP is Mode B (audit an uploaded construct)** — a single scrolling Streamlit
page that runs the deterministic check set on an annotated GenBank plasmid, plus a separate
primer-pair Tm panel. It proves the two theses (trustworthy validation + polymerase-aware
Tm) with the least surface area. Mode A (intent-driven design) is **out of scope for v1**.

## Decision table

| # | Question (source) | Decision | Rationale |
|---|---|---|---|
| 1 | Fresh session ask UI questions first? (`ARCH` line 8) | **No** — build against this file; only ask if a default is impossible | Autonomy is the whole point of a handoff |
| 2 | MVP cut (`ARCH §10`, `DESIGN`) | **Mode B audit spine + standalone primer Tm panel** | Least surface area that proves both theses |
| 3 | Single page vs wizard (`ARCH §9`) | **Single scrolling Audit page** | Wizard is a Mode A / post-MVP concern |
| 4 | Persistence (`ARCH §9`) | **One JSON lab profile + a `data/` folder of uploaded files** | Human-readable, no schema migration; drop SQLite for v1 |
| 5 | Output formats (`ARCH §10`, `DESIGN`) | **GenBank + FASTA via Biopython only** | SnapGene `.dna` is a proprietary binary rabbit hole |
| 6 | Results tabs; map + gel in v1? (`ARCH §9`) | **Findings table + Tm panel + the Sequence Workspace** (tabbed viewer, below); defer virtual gel | Viewer makes the audit visual/demo-able; gel adds little |
| 6b | DNA viewer/editor scope (`ARCH §12`, `DESIGN`) | **v1 = tabbed viewer (backbone/primers/inserts/product) with findings overlaid + form-based feature CRUD and add-primer**; full drag-canvas editing = roadmap | A read+light-edit viewer is cheap and differentiated; SnapGene-parity editing is a multi-week build we won't win on |
| 7 | Polymerases to ship first (`ARCH §10`, `DESIGN`) | **Q5 + Taq only** | Maximal, defensible Ta delta; two well-documented NEB models to benchmark against |
| 8 | Reference data source (`ARCH §10`, `DESIGN`) | **Bundle a tiny curated set in `data/`** | No live Addgene, no variant reference set for v1 |
| 9 | FASTA has no CDS features — how do integrity checks behave? | **Require annotated GenBank for integrity checks**; on FASTA run only alphabet/topology-independent checks and return an explicit `skipped — needs annotation` | A missing annotation must never read as a pass |
| 10 | Plasmid topology (unaddressed in both docs) | **`topology` is a first-class field on `Construct`; plasmids default to circular; thread into every `Bio.Restriction` call** | Linear-vs-circular changes site counts and cut logic |
| 11 | LLM in the MVP: model id + outage behavior (`ARCH §8`) | **Pin an explicit model id; ship a `FakeLLM` used in CI and as the live-demo fallback** | Deterministic core owns all numbers/verdicts, so the trust story survives an API outage |
| 12 | How Tm base value is computed vs vendor offset (`ARCH §8`) | **Per-vendor NN + salt/Mg model calibrated to match that vendor's own calculator; Ta = per-vendor rule, computed on the primer binding region** | See [`docs/TM_SPEC.md`](docs/TM_SPEC.md) — the naive "universal Tm + fudge factor" is a category error |
| 13 | License / openness | **MIT, public repo** | Hackathon is open-source; screener-visible |

## The check set for the Mode B audit (v1)

The original build order named two **primer-QC** checks as highest-value — but Mode B audits
a *plasmid, which has no primers*. Corrected v1 check set (all plasmid-applicable, each
returns `ran | skipped | error`):

1. **Reading-frame integrity** — premature/internal stop in a labelled CDS; missing/duplicated start or stop.
2. **Native-stop-removal for C-terminal fusions** — assert no in-frame stop between an ORF and a downstream tag (low-cost, high-frequency bug).
3. **Restriction-site uniqueness** — topology-correct; flag an enzyme that cuts at multiple sites.
4. **Feature collision** — an insert or cut landing inside a resistance marker, origin, or other essential feature.
5. **Unintended repeat / homology** — regions that would cause Gibson mis-assembly or PCR mispriming.
6. **Dam/Dcm methylation vs prep-strain** — cross-reference site context + enzyme methylation-sensitivity + the strain genotype in the lab profile. *(This is the lab-context moat made real.)*

The **primer Tm panel** is a separate surface: paste a forward + reverse primer, get Q5 and
Taq annealing temperatures with tailed-primer awareness and NEB reconciliation.

## Deferred to the roadmap (NOT v1)

Mode A intent parsing/design · Gibson overlap-Tm balance & Golden Gate overhang-fidelity
design · virtual gel · **full interactive canvas editing of the Sequence Workspace
(OpenVectorEditor / SeqViz; drag feature handles, in-place sequence editing, multi-user)** ·
codon QC beyond CAI · mammalian Kozak/2A/IRES special-casing · fluorescent-variant
suggestion · live Addgene. See [`docs/SCOPE.md`](docs/SCOPE.md).
