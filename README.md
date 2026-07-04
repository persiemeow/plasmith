# Plasmith

**Turn design intent + your lab's reagents into an order-ready, *validated* cloning plan.**

Plasmith is a local web app for molecular cloning. You either (A) state a goal in plain
language against a template sequence — *"insert GFP as a C-terminal fusion after this
gene"* — and get back the DNA to order, the final plasmid, a validity check, and a bench
protocol tuned to the polymerase you actually own; or (B) upload a construct you already
designed and Plasmith **audits it for defects and explains why it might fail at the bench.**

> **Status:** design-complete, pre-MVP. This repository is the specification and build
> plan for a one-week MVP built during the *Built with Claude: Life Sciences* hackathon
> (Jul 7–13). The design has been reviewed by a simulated panel — a bench cloning expert,
> a software reviewer, and a first-time user — and hardened accordingly (see
> [`DECISIONS.md`](DECISIONS.md) and [`docs/`](docs/)).

---

## Why it exists

Generic online calculators and sequence editors (SnapGene, Benchling, Geneious) let you
*draw* a construct, but they don't reliably tell you it's **wrong before you spend two
weeks and a plate of failed colonies finding out.** The three things that actually decide
whether a clone works — is the reading frame intact, will the enzyme cut where you think,
will the primers anneal at the temperature your polymerase wants — are exactly the things
those tools leave to the user's judgement.

Plasmith's bet is that these are **deterministic, checkable, and worth trusting** — so it
computes them in tested code, not in an LLM, and shows its work.

## What makes it different

1. **A trustworthy validation layer.** A registry of pure-function checks — reading-frame
   integrity, restriction-site uniqueness, feature collisions, methylation-vs-strain —
   each returning `ran | skipped | error` so a missing annotation can *never* be shown as
   a green all-clear. (See [`docs/SCOPE.md`](docs/SCOPE.md) for exactly what is and isn't
   validated.)
2. **A polymerase-aware Tm engine, done honestly.** Not "a generic melting temperature."
   A per-vendor nearest-neighbour model with salt/Mg²⁺ correction, computed on the primer's
   *binding region* (not its 5′ tail), producing the annealing temperature *your* enzyme
   wants — benchmarked against the vendor's own calculator. (See
   [`docs/TM_SPEC.md`](docs/TM_SPEC.md).)
3. **Lab-context awareness.** You configure your enzymes, plasmids, kits, strains, and
   polymerases once; Plasmith designs and validates against *your* bench, not a generic one.
4. **A visual sequence workspace.** A SnapGene-like tabbed viewer — **backbone · primers ·
   inserts · finished product** — that **overlays the validation findings on the map**, so a
   flagged stop codon or non-unique cut site is shown *where it happens*, not just listed. Add
   or edit features and primers and the checks re-run live. (Full drag-canvas editing is
   roadmap; see [`docs/SCOPE.md`](docs/SCOPE.md).)

## How it's built — the trust boundary

```
 PRESENTATION (Streamlit)  →  ORCHESTRATION  →  ┌ DOMAIN CORE (deterministic, unit-tested)
                                                │   io · validate · calc(Tm) · design
                                                └ LLM LAYER (bounded)
                                                    intent→spec · rank · draft prose
```

**The rule that defines the whole design:** anything a scientist must *trust* — Tm math,
frame/stop checks, restriction scans, assembly — lives in the deterministic core and is
unit-tested. The **LLM never asserts a number or a pass/fail verdict**; it only interprets
free-text intent, ranks options drawn from deterministic lookups, and writes human-readable
prose. That split is the product's trust story. Full detail in
[`ARCHITECTURE.md`](ARCHITECTURE.md).

## The one-week MVP

A **Mode B audit** of an annotated GenBank plasmid on a single Streamlit page, running ~5
deterministic checks, **plus** a standalone primer-pair Tm panel (Q5 vs Taq) with inline
reconciliation against NEB's public calculator, **plus** the tabbed **Sequence Workspace**
(viewer with findings overlaid + form-based feature/primer editing). It ships with **golden
fixtures** that
double as the definition-of-done *and* the one-click demo, a `FakeLLM` stub so the demo
survives an API outage, and an explicit SCOPE/LIMITATIONS section. Everything decided is in
[`DECISIONS.md`](DECISIONS.md); the acceptance criteria are in
[`docs/ACCEPTANCE.md`](docs/ACCEPTANCE.md).

Explicitly **out of scope for the MVP:** Mode A intent-driven design, Gibson/Golden Gate
junction design, and the virtual gel — these are roadmap, not v1.

## Repository map

| File | What it is |
|---|---|
| [`README.md`](README.md) | You are here |
| [`DESIGN.md`](DESIGN.md) | Product vision — the full feature set and both modes |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Engineering handoff — layers, modules, data model, build order |
| [`DECISIONS.md`](DECISIONS.md) | **Committed defaults** for every open question — build against these |
| [`docs/TM_SPEC.md`](docs/TM_SPEC.md) | The polymerase-aware Tm engine, specified honestly |
| [`docs/ACCEPTANCE.md`](docs/ACCEPTANCE.md) | Golden fixtures + pinned expected findings = definition-of-done |
| [`docs/SCOPE.md`](docs/SCOPE.md) | What the MVP does and does **not** validate |

## Built with Claude

Plasmith uses the Anthropic API (Claude) strictly inside the bounded LLM layer — intent
parsing, option ranking, and protocol prose — while the deterministic core owns every
number and verdict. Built for *Built with Claude: Life Sciences* (Build track).

## License

[MIT](LICENSE) — open source.
