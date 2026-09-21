# 0051 — The audit's finding IDs are frozen; this project's own findings get their own prefix

## Status

Accepted, 2026-09-21. Extends [ADR 0041](0041-deliberate-deviations-are-switchable.md), which named
every compatibility flag after the finding it restores; renaming the findings therefore renames the
flags. Supersedes nothing.

## Context

Every defect this project records carries a short ID. The prefix says where the finding came from:
`B-nn` the baseline's source, `N-nn` a baseline nondeterminism source, `D-nn` the data and example
packs, `R-nn` the earlier rewrite the audit also reviewed. All four were assigned by the audit
carried out on 2026-09-17, before this project started ([docs/audit/](../audit/)), which numbered
`B-01`–`B-19`, `N-1`–`N-20`, `D-01`–`D-08` and `R-01`–`R-15`.

Later build runs found defects of their own and **minted new IDs inside the audit's `B-` and `D-`
namespaces**: `B-21` to `B-30`, and a second `D-04`. Nothing recorded that this had happened. Three
consequences, each of which a reader hit before this record existed:

- **`B-nn` meant two different things.** `B-15` is an audit finding with an entry to look up;
  `B-29` is not, and the audit's index gives no hint that it is missing rather than stale. The
  header of [docs/deviations.md](../deviations.md) stated that its `B-nn` IDs came from
  `docs/audit/04-baseline-issues.md`, which was false for ten of them, including the three that
  name compatibility flags.
- **`D-04` named two unrelated findings.** In the audit it is *the examples pin five mutually
  exclusive data releases*; in `docs/deviations.md` it is *an unregistered `diseases/COPD/`
  directory*. Both were cited by those IDs, in documents that link to each other.
- **A gap with no record.** There was no `B-20`. Nothing said whether a finding had been retired or
  the number was simply skipped.

The audit is a dated artefact with its own internal cross-references, in the same class as an ADR:
written once, about a tree as it stood on a particular day. Renumbering it to close the gap would
invalidate every citation inside it and destroy the one property that makes it usable as evidence.

## Decision

**The audit's IDs are frozen.** `B-01`–`B-19`, `N-nn`, `D-01`–`D-08` and `R-01`–`R-15` keep the
numbers the audit gave them, and no future finding is added to those ranges.

**Findings this project makes get `P-nn`** — *this Project, after the audit*. A `P-nn` has no audit
entry by construction; its evidence is its row in [docs/deviations.md](../deviations.md) and the
test named there. New findings continue from `P-12`.

The eleven existing post-audit findings were renumbered:

| was | is | what |
|---|---|---|
| B-21 | **P-01** | immigration into an empty age-sex band abandons that band's target silently |
| B-22 | **P-02** | `std_income` is emitted as a column and never filled |
| B-23 | **P-03** | `two_stage.use_logistic` is read into the config and consulted nowhere |
| B-24 | **P-04** | the food-labelling policy applies its impact more than once |
| B-25 | **P-05** | an intervention on the Kevin Hall or static-linear model has no effect |
| B-26 | **P-06** | a missing PIF table is a silent no-op; a gap inside one reads as zero |
| B-27 | **P-07** | the published PIF schema inverts the sex column |
| B-28 | **P-08** | `data_root_path` is required, expanded, then overwritten before use |
| B-29 | **P-09** | the energy balance integrates body fat straight through zero |
| B-30 | **P-10** | a `NaN` risk factor is counted as zero in the year's means |
| D-04 *(deviations)* | **P-11** | an unregistered `diseases/COPD/` directory holding only a `PIF/` subtree |

`N-1`–`N-9` were padded to `N-01`–`N-09`, so that every prefix pads to two digits and IDs sort
lexically. The audit's `N` numbering is otherwise untouched.

## Consequences

**Three compatibility flags were renamed**, because ADR 0041 requires a flag to be named after the
finding it restores, so that a flag in a config or a manifest leads straight to the entry explaining
it. This is a **breaking change to a published interface**:

| was | is | restores |
|---|---|---|
| `B-24` | `P-04` | the food-labelling policy's repeated impact |
| `B-29` | `P-09` | the energy balance integrating past a body fat mass of zero |
| `B-30` | `P-10` | counting a value that is not a number as zero in the year's means |

The rename reaches the `baseline_compat` array in a config, `--baseline-compat` on the command line,
`CompatFlag` and `LoadOptions::baseline_compat` in the published API
([ADR 0032](0032-library-and-a-thin-cli.md)), the `baseline_compat` property in `schemas/v2`, the
flags recorded in every run manifest ([ADR 0034](0034-a-run-manifest-beside-the-results.md)), and
the flag list the frontend renders.

**A configuration written before this record fails to load**, rather than running with the flag
silently ignored: an unrecognised name is an error and always has been
([ADR 0041](0041-deliberate-deviations-are-switchable.md)), and the error names the flag it did not
recognise. The migration is the table above. No stored equivalence reference carries a flag name, so
nothing in `tests/equivalence/reference/` needed regenerating, and the comparison confirmed it
rather than assuming it.

**Two defects were corrected on the way.** `src/data/store.cpp` cited `docs/deviations.md D-40`, an
ID that has never existed in this repository, for what is now `P-07`. `schemas/v2/config.json` said
`"B-24" is the only one so far`, which stopped being true when the tenth run added two more flags.

**What this costs.** Five prefixes instead of one, and a reader must learn that four of them are the
audit's and one is not. That is the price of keeping the audit citable; the alternative renumbers a
dated document that other documents quote by number.

## Alternatives

- **Renumber everything into one continuous scheme.** One prefix and no explanation needed. It
  rewrites `docs/audit/`, whose prose cites its own findings by number throughout, and it removes
  the distinction between a finding somebody else confirmed before this project began and one this
  project made about its own comparison. Rejected: the audit's value is that it predates the work it
  is used to justify.
- **Keep the flag names as `B-24`, `B-29` and `B-30`** and renumber only the findings. No breaking
  change. It leaves three `B-` numbers in circulation that match no finding, which is a smaller
  version of the problem this record exists to remove, and it breaks ADR 0041's rule that a flag
  name leads to its entry. Rejected on those grounds; the interface has no consumers outside this
  repository yet, which is the cheapest this change will ever be.
- **Leave the numbering alone and document the irregularity.** Tried first, in
  `docs/upstream-reports.md`: it took a paragraph to explain that `B-21` and above are not audit
  findings, that there is no `B-20`, and that finding nothing in the audit for `B-29` means the ID
  is newer rather than wrong. A scheme needing a paragraph of apology is the defect, not the
  documentation of it.
