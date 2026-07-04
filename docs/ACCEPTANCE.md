# Acceptance criteria — golden fixtures

The design review's sharpest point: **without pinned expected outputs, the "trustworthy
validation" thesis is unfalsifiable.** This file is the definition-of-done. Each fixture is
a real sequence file with a *hand-verified* expected result; the MVP is "done" when the
audit reproduces every expected finding exactly, and the primer engine reproduces every
expected Tm within tolerance. These fixtures double as the **one-click demo**.

Fixtures live in `tests/fixtures/`. Expected findings are asserted in `tests/`.

## Construct fixtures

### `fixture_good.gb` — a clean, annotated plasmid (negative control)
An annotated circular plasmid with an intact CDS, a functional resistance marker, an origin,
and no defects.
- **Expected:** every check returns `ran`; **zero `error` findings.** (A false positive here
  fails the build.)

### `fixture_bad.gb` — an annotated plasmid with two planted defects
Same backbone, with two deliberate errors:
1. a **premature stop codon** inside a labelled CDS, and
2. a **duplicated recognition site** for an enzyme the design would use as a unique cutter.
- **Expected findings (exact):**
  - `error` · reading-frame integrity · premature stop in CDS *<name>* at position *<n>*.
  - `error` · restriction-site uniqueness · *<enzyme>* cuts at 2 sites (expected 1).
  - all other checks return `ran` with no finding.

### `fixture_fusion.gb` — a C-terminal fusion with a retained native stop
An ORF fused to a downstream tag, but the ORF's **native stop codon was not removed**, so the
tag is never translated.
- **Expected:** `error` · native-stop-removal · in-frame stop between ORF *<name>* and tag
  *<name>* at position *<n>*.

### `fixture_unannotated.fasta` — plain sequence, no features (precondition test)
A correct sequence supplied as FASTA (no CDS annotations).
- **Expected:** integrity checks return **`skipped — needs annotation`** (never `ran`, never a
  pass). Topology/alphabet-independent checks may still `run`. This proves a missing
  annotation can never surface as a green all-clear.

### `fixture_methylation.gb` — a site blocked by the prep strain (lab-context test)
A plasmid whose intended digest site overlaps a Dam or Dcm methylation context, propagated in
a `dam+/dcm+` strain from the lab profile.
- **Expected:** `warning` · methylation-vs-strain · *<enzyme>* site at *<n>* is blocked by
  *<Dam|Dcm>* methylation in strain *<name>*; propagate in a *<dam−/dcm−>* strain.

## Primer Tm vectors

A fixed set (≈15–20) of forward/reverse primers spanning length and GC range, each with the
expected binding-region Tm and per-vendor Ta. See [`TM_SPEC.md`](TM_SPEC.md#benchmark) for the
tolerances (±1.5 °C on Tm; exact match on the untailed Ta rule). At minimum include:

| Case | Expectation |
|---|---|
| Matched pair, untailed | Q5 and Taq Ta reproduce the vendor rule; small forward/reverse Tm mismatch |
| Mismatched pair | Tm mismatch flagged; **touchdown gradient** recommended |
| Tailed primer (5′ RE/overlap tail) | binding-region Tm **strictly below** full-length Tm |
| High 3′-GC primer | 3′-end GC flagged as a mispriming risk |

## Build gate

CI runs the audit over every construct fixture and the Tm engine over every primer vector,
using the `FakeLLM` stub (no network). **Green CI = the fixtures reproduce exactly = the MVP
meets its trust bar.** This is the gate a demo, a screener, or a fresh build session can
check in one command.
