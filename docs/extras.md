# What the rewrite delivers

[docs/upstream-reports.md](upstream-reports.md) is every defect this project found in the baseline's
code and data, written up so that whoever owns them can fix them without help. This file is the
other half of the account: what this implementation does **better**, measured rather than asserted.

It states results rather than methods. Where a number here comes from a change to how something is
built, the change itself is in this repository and is not described here — this is the summary a
reader outside the project gets, and the reasoning behind each result is a longer conversation.

---

## 1. Speed and memory

Best of five against the baseline binary, on the same configurations and the same seeds, in one
session on one machine.

| Example | Wall | CPU | Peak memory |
|---|---|---|---|
| `HLM_France` | **2.0× faster** | **3.2× less** | **37% less** (85.6 → 52.7 MiB) |
| `KevinHall_FINCH` | **3.5× faster** | **5.5× less** | **2.5× less** (197.8 → 78.1 MiB) |
| `HLM_India`, 1,240,613 people | level (2,068 s against 2,079 s) | **2.0× less** | **1.6× less** (3,011 → 1,872 MiB) |

**The CPU column is the one that matters.** The baseline runs its two scenarios on two threads. This
build runs them sequentially on one, and is still faster on wall clock on two of the three examples
and level on the third. On `HLM_India` the baseline spends two cores to finish at the moment this
build finishes on one.

That India row is also the first time the baseline had been run on that example at full scale, on
any machine, in any run of this project. It exits 0 and writes the same 16,564 rows to each of the
same four files.

## 2. Two thirds of the memory on the largest example is available to the baseline too

`KevinHall_FINCH` peaks at 198 MiB in the baseline and peaked at 195 MiB here — the same number,
because both implementations had made the same structural choice. Changing it took this build to
**77.5 MiB, a 60% cut**, and most of the CPU gap came with it. **No output moved.**

The same opportunity is in the baseline's tree, and taking it needs no modelling decision and no
agreement with anybody. Two smaller ones are written up in full as **B-13** and the **B-05**
extension in [docs/upstream-reports.md](upstream-reports.md), because those are defects rather than
design choices and belong in the reports.

## 3. Determinism as a guaranteed property

- **Byte-identical output at any thread count**, asserted in the test suite.
- **Byte-identical output across runs of the same seed.** Three same-seed runs of the baseline
  produced three different result files, identical only after sorting (**B-01**).
- **A trial run's results do not depend on how many runs precede it.**
- **A fixed, stated evaluation order** everywhere floating-point summation could otherwise vary.
- The contract is fourteen clauses and it is **enforced by the compiler**, not by review: the
  language of the codebase makes the non-deterministic version of each operation unavailable rather
  than discouraged.

## 4. The concurrency defect is gone by construction

Not fixed by locking — removed, by arranging that the racing access cannot occur. **Zero** signal
exits across 500 `KevinHall_FINCH` runs, against the baseline's two in the same 500 (report 4).

## 5. Failures are located and loud

- An input mistake produces a **located diagnostic naming the file, the field and the line**, and
  every such problem in a configuration is reported in one pass rather than one per run.
- **A misspelled name is an error, not a plausible number.** Names are resolved and checked before a
  run starts.
- **Every applied default is reported**, naming the field and the value used.
- **No output carries a number that cannot exist.** A value that is not finite, or not a possible
  description of a person, stops the run naming the person, the year and the term, instead of
  becoming a quietly wrong mean.
- An undefined environment variable in a path is an error naming it, rather than an empty string
  that silently relocates a run's output (**N-17**).

## 6. Testing

| | This build | The baseline |
|---|---:|---:|
| C++ tests | **885** | 471 |
| of which skip silently | **0** | **35** (7.4% of the suite, **B-11**) |
| build configurations they pass under | **4** — release, debug, ASan+UBSan, ThreadSanitizer | release |
| tests over the comparison harness's own statistics | **94** | — |
| CI jobs | **16**, Linux and macOS, clang and gcc | — |

The 94 harness tests matter more than the number suggests. A mistake in a comparison harness reports
PASS rather than producing a visibly wrong result, so this one is tested against itself **and
against a deliberately wrong build** — it has to detect a defect that was put there on purpose.

## 7. Portability

This build compiles on **macOS and Linux** as shipped. The baseline needs four source changes to
build on macOS at all (**B-10**), and [docs/build-notes.md](build-notes.md) records every one of
them, which is most of the work of fixing it. That one is a gift rather than a claim.

## 8. A library, not a program

The engine is a library with a **published API of four calls**, and that is the entire surface. The
command line, the local server and the browser frontend are all ordinary clients of it, none of them
privileged, none of them able to reach past it.

Progress is an **event stream the simulation cannot observe**, so watching a run cannot change its
results.

## 9. A browser frontend

Configure a run from the published schema, validate it and see a diagnostic land on the field that
caused it, start it, watch it stream year by year, chart and tabulate the results, switch between
output families, and download the CSV the run actually wrote.

**48 unit tests and 21 end-to-end tests in a real browser**, both in CI. Served out of the same
binary as the engine, so there is no second thing to deploy.

## 10. Runs that can be reproduced and audited

- **A config pins its own data release verifiably**, so "which data produced this result" is a
  question with an answer. The six upstream examples name five mutually exclusive data releases and
  nothing records which one a given working copy holds (**D-04**).
- **Every run writes a manifest** beside its results: the seed actually used, the options in force,
  the inputs and their hashes.
- **A result file is byte-comparable between runs** of the same configuration.
- Results are written at a precision that round-trips, rather than at six significant digits — which
  is what puts a 10⁻⁵ floor under every comparison against the baseline.

## 11. A documented configuration format, and a converter for the old one

A published JSON Schema that a test keeps in step with the loader, and a converter that reads the
upstream examples as they ship, reports what it changed and why, and resolves the contradiction that
makes `KevinHall_India/new_config.json` unloadable by the validator shipped beside it.

## 12. The comparison harness, which is useful on its own

It runs both implementations over many seeds, reduces every result file each of them writes to a
comparable series, applies a pass rule whose false-positive rate is **stated and was measured before
it was adopted**, and prints a verdict. That rule is the reason the equivalence claims in
[docs/upstream-reports.md](upstream-reports.md) mean something: the previous one gave between 0 and
45 failures from the same build against the same baseline depending on which twenty seeds were
drawn.

A companion answers a different question — *did these five hundred runs finish, and is what they
wrote sane?* — by reading every output file rather than the exit code. That is how the
four-in-five-hundred body-fat finding was established, on an example where a runaway weight leaves
both implementations exiting zero.

---

## What is on offer here

Three of these need nothing from this project to be useful. **§2** is a change the baseline's own
tree can take, worth 60% of one example's memory and most of its CPU, moving no output. **§7** is
four source changes already written down. **§5** is a position on diagnostics that costs nothing to
adopt incrementally.

The rest are properties of this implementation rather than patches, and they are worth stating for
one reason: an implementation that cannot reproduce itself byte for byte cannot be used to establish
anything about another one. Everything in
[docs/upstream-reports.md](upstream-reports.md) rests on the guarantees in §3 and §6.
