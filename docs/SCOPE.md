# Scope & limitations

A validation tool earns trust by being **explicit about what it does not check.** The worst
failure mode for Plasmith is a green light it cannot back — a scientist reading "no issues"
as "this will work" when a whole class of failure was never examined. This page is the
contract; the UI surfaces a short version of it next to every clean result.

## What the v1 (Mode B) audit validates

- Reading-frame integrity in **labelled** CDS features (premature/internal stop; missing or
  duplicated start/stop).
- Native-stop retention in C-terminal fusions.
- Restriction-site **uniqueness**, topology-correct (circular vs linear).
- Feature collisions (insert/cut inside a resistance marker, origin, or essential feature).
- Unintended repeats / homology that risk mis-assembly or mispriming.
- Dam/Dcm methylation vs the prep strain in the lab profile.
- Primer-pair Tm / annealing temperature for **Q5 and Taq**, with tailed-primer awareness
  (separate primer panel — see [`TM_SPEC.md`](TM_SPEC.md)).
- A tabbed **Sequence Workspace** (backbone / primers / inserts / finished product) that
  renders annotated maps with the findings above **overlaid**. v1 editing is **form-based**
  (feature add/edit/delete, add primer); every edit re-runs the affected checks.

Every check reports `ran | skipped | error`. **`skipped` is shown, never hidden** — e.g. an
unannotated FASTA upload skips the integrity checks rather than passing them.

## What v1 does NOT validate (do not read a clean result as covering these)

- **Assembly-junction fidelity** — Gibson overlap-Tm balance, Golden Gate Type-IIS overhang
  design and ligation fidelity. (Golden Gate failure is dominated by overhang design, which
  v1 does not check — it only flags internal sites.)
- **Mode A design correctness** — intent parsing, primer *design*, and construct assembly are
  out of v1 entirely.
- **Codon quality beyond simple checks** — CAI is host- and reference-dependent; v1 does not
  do full codon optimisation, and does not check cryptic splice sites or 5′ mRNA structure.
- **Mammalian expression special cases** — Kozak strength scoring, 2A/IRES polycistronic
  handling, selenocysteine. (v1's frame/stop checks assume a standard single-ORF reading and
  may mis-call these — hence they are gated to labelled CDS and reported conservatively.)
- **Methylation beyond Dam/Dcm** (e.g. CpG in mammalian contexts).
- **Strain/expression compatibility**, toxicity, or copy-number effects.
- **Fluorescent-variant or "better part" suggestion** — an *opinion* feature kept off the
  pass/fail surface; if added later it is presented as a property vector, not a verdict.
- **Full SnapGene-parity editing** — drag-on-canvas feature handles, in-place sequence
  editing, and multi-user editing are roadmap; the v1 Sequence Workspace edits via forms only.

## Roadmap (post-MVP)

1. Mode A: intent → `EditSpec` → deterministic assembly, primer design, protocol.
2. Assembly-junction design & validation (Gibson overlap balancing; Golden Gate overhang set
   design and fidelity scoring).
3. Expanded validation: codon QC beyond CAI, mammalian Kozak/2A/IRES, plasmid map + virtual gel.
4. More polymerases (each added as a calibrated Tm/Ta pair with its own benchmark row).
5. Reference data: curated variant sets; optional live Addgene.
