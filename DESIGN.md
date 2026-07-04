# Plasmith — Functional Design (working draft)

> This is the **product vision** — the full feature set and both modes. It is intentionally
> broader than the one-week MVP: the committed MVP cut and every other decision now live in
> [`DECISIONS.md`](DECISIONS.md), and the deterministic engine specs are in [`docs/`](docs/).
> Read this for *what Plasmith is meant to be*; read `DECISIONS.md` for *what v1 actually builds*.

## What it is

An **order-ready, validated cloning planner**. You give it *intent + your lab's
context*; it returns a *validated cloning plan* — the DNA to order, the final
plasmid sequence, a design-validity check, and a bench protocol tuned to the
reagents you actually have.

**Differentiator vs SnapGene / Benchling:** the smart validation layer (catches
design mistakes before they cost bench time), a polymerase-aware primer/Tm
engine (real PCR conditions, not a generic Tm), and lab-context awareness (it
designs against *your* enzymes, plasmids, kits, and strains).

## Scope decided so far (from interview)

- **Form:** local web app (e.g. Streamlit — visual, low-maintenance)
- **Hosts:** E. coli + mammalian (HEK/CHO)
- **Assembly methods:** Gibson, restriction–ligation, Golden Gate (all three)
- **MVP cut:** TBD — see Open Questions

## Lab context / local memory (set once, reused across designs)

The user configures their lab once; Plasmith stores it locally and designs
against it. Inputs:

1. **Restriction enzymes** available
2. **Lab-owned plasmids & sequences** — an uploadable GenBank/FASTA library
3. **Cloning kits** available (e.g. Gibson/HiFi assembly, Golden Gate kits, T4 ligase)
4. **Cloning / expression strains** available (e.g. DH5α, BL21, Stbl3; mammalian lines)
5. **PCR polymerases** available *(implied — drives the Tm engine; confirm)*

This context shapes primer design, method selection, protocols, diagnostic
checks, and recommended controls.

## Two ways to use it

### Mode A — Design (intent-driven)

1. **Import a template** — FASTA or GenBank (or start from a reference / Addgene plasmid).
2. **State the goal in plain language**, e.g.
   - "Replace the Cas9 here with Cas12a"
   - "Insert GFP after this gene as a fusion" (N- or C-terminal)
3. **Sourcing question:** synthesize a **gBlock** for cloning (which assembly
   method?) — *or* use a **pre-existing plasmid you own** (upload the plasmid
   carrying, e.g., Cas12a or GFP).
4. **Outputs:**
   1. **DNA to order** — primers and/or gene blocks
   2. **Final assembled plasmid sequence**
   3. **Validity check** against the design-error criteria (below)
5. **Cloning plan:**
   - **PCR conditions** tuned to the user's polymerase (Tm, touchdown gradient)
   - **Fusion / assembly protocol** or annealing/ligation protocol
     (experience-based suggestions)
   - **Recommended controls**
   - If needed: **colony-PCR primers** or **diagnostic restriction digests**,
     using only enzymes the lab has (from local memory)

### Mode B — Audit (validate an existing design)

- User uploads a sequence file they already designed.
- Plasmith auto-checks the plasmid for defects and concerns.
- Outputs a report: **what's wrong and why it might fail at the bench.**

## The validation layer (design-error checks)

**Sequence / ORF integrity**
- Premature / internal **stop codon** in a labelled ORF
- **Frameshift** — insert not in frame with a tag or downstream ORF; a labelled
  ORF that is actually frameshifted or produces a truncated product
- **Wrongly-copied ORF** — mismatch vs a reference sequence (the copy-error case)
- Missing / duplicated **start or stop**
- **Tag** frame, orientation, and **5′/3′ position**; linker sanity; whether the
  tag position could affect protein integrity/function — cross-check Addgene /
  reference fusion designs

**Expression elements**
- **RBS spacing** from the start codon (E. coli)
- **Promoter** position / distance
- **Kozak** sequence (mammalian)
- **Rare / unoptimized codons**; low CAI after codon optimization
- **Auto-align an ORF to public data and suggest better alternatives**
  (e.g. a brighter GFP variant suited to the use case)

**Cloning feasibility**
- **Restriction-site uniqueness** — a chosen enzyme that would cut at *multiple*
  sites instead of a single site
- Internal restriction sites that **break the chosen method** (e.g. Golden Gate BsaI)
- **Unintended repeats / homology** → Gibson mis-assembly or PCR misprime
- **Feature collisions** — insert or cut lands inside a resistance gene, ori, or
  other essential feature

**Primer QC (design mode)**
- High overall **GC** and high **GC at the 3′ end**
- **Tm mismatch** between forward and reverse primers for the specific
  polymerase / enzyme mix
- **Multiple binding sites** → recommend touchdown-PCR conditions to suppress
  off-target priming
- **Primer self-priming / self-amplification** (dimers, hairpins)

## The primer / Tm engine

- **Polymerase-specific Tm:** salt / Mg²⁺ correction + the enzyme's recommended
  annealing offset → sets primer length and the touchdown gradient.
- Because online calculators disagree and SnapGene ignores buffer ions and
  polymerase speed, this engine is a core differentiator.
- **Which polymerases first:** TBD — pick 2–3 (e.g. NEB Q5, Phusion, Taq, KAPA HiFi).

## Outputs

- **Order list** — primers and/or gene blocks
- **Final plasmid sequence** — file format TBD (GenBank / FASTA / SnapGene `.dna`)
- **Validity report**
- **Cloning plan** — PCR conditions, assembly/fusion protocol, controls,
  diagnostic colony-PCR primers or restriction digests
- **Virtual gel / plasmid map** *(from the original brief — confirm for MVP)*

## Sequence viewer & editor (SnapGene-like workspace)

A visual workspace for looking at and editing the DNA, organised as **four tabs** so the
user follows the construct from parts to product:

1. **Backbone** — the vector: origin, resistance marker, MCS, promoters.
2. **Primers** — every primer, its binding site on the template, orientation, and per-vendor
   Tm (feeds the Tm engine in [`docs/TM_SPEC.md`](docs/TM_SPEC.md)).
3. **Inserts** — the gene block(s) / fragment(s) to be cloned in.
4. **Finished product** — the in-silico assembled construct (backbone + insert(s)), i.e. the
   plasmid you would actually order/verify.

Each tab shows an **annotated map** (circular for plasmids, linear for fragments/primers) +
a **feature table** + the **sequence**. The differentiator vs SnapGene: **validation findings
are overlaid directly on the map** — a premature stop, a non-unique cut site, a feature
collision, a methylation-blocked site are highlighted *in place*, so the audit becomes
spatial, not just a list.

**Edit operations (v1 — form/side-panel based, not free-drag):**
- **Features:** add, edit, or delete a feature (name, type, start/end, strand).
- **Primers:** add a primer by pasting a sequence *or* by selecting a region to derive one;
  see its binding site and Tm immediately.
- Every edit **re-runs the affected checks live**, so the viewer and the validation layer are
  the same surface — this coupling is the point.

**Deferred to roadmap:** full SnapGene-parity interactive canvas editing (click-drag feature
handles, in-place sequence editing, drag-to-select), and real-time multi-user editing. See
[`docs/SCOPE.md`](docs/SCOPE.md). Candidate libraries for that stage: OpenVectorEditor
(`@teselagen/ove`) or SeqViz embedded as a Streamlit custom component; v1 renders with
`dna_features_viewer`.

## Open questions (next interview batch)

- **Data sources:** Addgene (do you have API access, or curate a local set?);
  codon-usage tables (Kazusa?); RBS/promoter references; a fluorescent-protein
  variant reference for the "suggest a brighter GFP" feature (e.g. FPbase)
- **Output file formats:** GenBank vs SnapGene `.dna` vs FASTA — which matter?
- **LLM vs deterministic boundary:** where the model reasons (parsing intent,
  suggesting alternatives, drafting protocols) vs where deterministic code owns
  the answer (Tm math, frame/stop checks, restriction-site scanning) — you want
  validation *trustworthy, not hallucinated*
- **The one-week MVP cut** vs the full vision above
