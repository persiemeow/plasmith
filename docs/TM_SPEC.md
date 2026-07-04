# Tm / annealing-temperature engine — specification

The polymerase-aware Tm engine is Plasmith's headline differentiator. The design review
flagged the original one-line description as a **methodological category error**, so this
document respecifies it honestly. If this engine is not correct, the differentiator is a
liability rather than a moat.

## The mistake to avoid

The original brief described the engine as *"Biopython's corrected nearest-neighbour Tm +
a per-polymerase annealing offset from a small config table."* That is wrong in two ways:

1. **A vendor's annealing rule is calibrated to that vendor's own Tm model, not to a
   universal Tm.** NEB's "Ta = Tm + 3 °C" for Q5 is only meaningful *relative to the Tm
   that NEB's calculator produces.* Bolt a different Tm model underneath the same offset
   and the recommendation drifts by several degrees. You cannot mix one lab's Tm with
   another's offset.
2. **Tm computed on the full oligo is wrong for any tailed primer.** Overlap primers
   (Gibson), Type-IIS primers (Golden Gate), and restriction-site-tailed primers do **not**
   anneal along their 5′ tails in the first PCR cycles — only the 3′ binding region hybridises
   to the template. Melting the whole oligo overstates the effective Tm.

## The model

For each primer and each supported polymerase, compute:

**1. Base duplex Tm on the binding region only.**
- Use the SantaLucia (1998) unified nearest-neighbour parameter set for ΔH/ΔS.
- Apply a salt correction and an explicit divalent-cation (Mg²⁺) + dNTP correction —
  Owczarzy et al. (2008) — because PCR buffers are Mg²⁺-dominated, not monovalent.
- Compute on the **3′ binding region** that actually hybridises to the template, excluding
  any 5′ tail. Record the tail separately; it raises Tm only after it is incorporated
  (cycle ≥ 2), which the touchdown recommendation accounts for.

**2. Per-vendor annealing temperature `Ta = f(Tm, vendor)`.**
- The base Tm model above must be **calibrated to reproduce each vendor's own calculator**
  within tolerance before its offset rule is applied (see Benchmark). Ship one calibrated
  `(Tm-model, Ta-rule)` pair per vendor — never a shared Tm with per-vendor offsets.
- **v1 vendors: Q5 (NEB) and Taq.** Chosen for the largest, most defensible Ta delta and
  because both have well-documented public models to benchmark against.

**3. Pair-level output.**
- Report forward Tm, reverse Tm, the **Tm mismatch**, and the recommended Ta per polymerase.
- When the pair is mismatched or a primer has multiple binding sites, recommend a
  **touchdown gradient** (start ~Tm+high, step down) rather than a single Ta.

## Config-table shape

```
polymerases:
  Q5:
    tm_model:   {nn: santalucia_1998, salt: owczarzy_2008, mono_mM, mg_mM, dNTP_mM}
    ta_rule:    Ta = Tm_binding + 3        # NEB Q5, relative to the calibrated model above
    primer_conc_nM
  Taq:
    tm_model:   {nn: santalucia_1998, salt: owczarzy_2008, mono_mM, mg_mM, dNTP_mM}
    ta_rule:    Ta = Tm_binding - 5        # generic Taq guidance
    primer_conc_nM
```

The numbers in `ta_rule` and the buffer concentrations are the **honest core of the "real
PCR conditions" claim** — they must be sourced (vendor protocol / manual) and cited in the
config, not invented.

## Benchmark (definition-of-done for this engine)

The engine is "done" only when, on a fixed test set of ~15–20 primers of varied length and
GC:

- the **binding-region Tm matches the vendor's own public calculator** within **±1.5 °C**
  for each supported vendor, and
- the recommended **Ta matches the vendor's stated rule** exactly for untailed primers, and
- a **tailed primer** yields a binding-region Tm strictly below its full-length Tm (proving
  the tail is excluded).

The test primers and their expected values are pinned in
[`ACCEPTANCE.md`](ACCEPTANCE.md#primer-tm-vectors).

## Explicitly not claimed

Plasmith does **not** claim a universal "true" Tm, does not model secondary structure's
effect on annealing beyond flagging hairpins/dimers, and does not extend the vendor rules
to polymerases it has not calibrated. Adding a polymerase means adding a calibrated pair and
a benchmark row — not reusing another enzyme's offset.

### References
- SantaLucia, J. (1998). *A unified view of polymer, dumbbell, and oligonucleotide DNA
  nearest-neighbor thermodynamics.* PNAS 95(4):1460–1465.
- Owczarzy, R. et al. (2008). *Predicting stability of DNA duplexes in solutions containing
  magnesium and monovalent cations.* Biochemistry 47(19):5336–5353.
- NEB Tm Calculator / Q5 & Taq product protocols (cite exact version in the config table).
