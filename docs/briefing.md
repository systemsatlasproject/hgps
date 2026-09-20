# A briefing for the Health-GPS authors

For a reader with twenty minutes. Project terms are defined in [docs/glossary.md](glossary.md).

## What this is

This repository is a second implementation of Health-GPS in C++20, covering the model surface your
own examples use. It is not a fork of your code and not a replacement for it. Its purpose is to run
both implementations on the same configurations and seeds and to report where their results differ.

## What it proves, and how

Three of your six examples run to the end in both implementations and agree statistically:
`HLM_France`, `HLM_India` and `KevinHall_FINCH`. The comparison was **62,030 tests over two
examples' stored references, none failing, with no failure budget**
([docs/equivalence.md](equivalence.md)). One test covers one variable, scenario, year and sex in one
output file. The pass rule is a **family-wise false-positive rate of 1%** under Holm's correction,
measured before it was adopted by comparing one build against itself: **30 pairs, 922,564 tests,
zero failures against 0.30 expected** ([docs/equivalence-method.md](equivalence-method.md) §4).

This build carries **885 C++ tests**, passing under four build configurations, and **94 tests** over
the comparison harness's own statistics. CI runs **16 jobs and all 16 pass**
([docs/SUMMARY.md](SUMMARY.md)).

Five limits apply. `HLM_India` has only ever been compared at **12,406 people against the 1,240,613
it ships**. The Kevin Hall surface has been compared on one country and one pack. The population
impact fraction has never met your binary, because its only example cannot run. Your result files
carry six significant digits, so no comparison is tighter than 10⁻⁵ relative. Only `KevinHall_FINCH`
fills the income-stratified files, because nobody in an HLM example has an income category
([docs/equivalence.md](equivalence.md)).

## What the comparison found in your code

Eight findings, plainest first. Evidence for each is in [docs/deviations.md](deviations.md), and
seven are written up in [docs/upstream-reports.md](upstream-reports.md).

**A `NaN` risk factor is counted as zero in the year's mean, and an infinity is not checked at
all.** Three people of 80 kg, one with a `NaN` weight, report a mean of 53.3 kg
(`analysis_module.cpp:385-391`). **B-30**, report 7.

**Four demographic standard deviations have their square root taken twice**, because two loops in
`AnalysisModule::calculate_standard_deviation` finish the same channel. `std_region` is **0.122097**
where the band's spread is 0.745, and `std_ethnicity`, `std_income` and `std_sector` go the same
way. This build reproduces it, because the correction is yours. Report 5.

**The food-labelling policy applies its impact more than once** to anybody who fails an early
coverage draw and passes a later one. The measured effect is **+0.209% of mean male BMI on
`HLM_France`** and +0.160% on `HLM_India`. **B-24**.

**An intervention on the Kevin Hall or static-linear surface has no effect, and the run reports
success.** `Scenario::apply` has one call site in your tree, and neither `kevin_hall_model.cpp` nor
`static_linear_model.cpp` calls it. `KevinHall_FINCH` with `simple` and with `marketing` produces
identical result files. **B-25**, report 2.

**Immigration into an empty age-sex band abandons that band's target silently.** On `HLM_France`,
**197 of your band counts** fall short of the demographic projection, and every one of those bands
holds zero people (**B-21**).

**Three defects on the population impact fraction path.** A fraction the data store cannot supply is
a silent no-op, and a gap inside a supplied table reads as exactly zero (**B-26**). The published
schema gives the sex column as 0 for female and 1 for male, the opposite of your loader and your
data (**B-27**). In `KevinHall_PIF`, 4 of 15 diseases have no `Smoking` table.

**Three smaller defects.** `std_income` is written as a column and never filled: zero in all 4,884
rows of a `KevinHall_FINCH` run (**B-22**). An income distribution is built by iterating an
`std::unordered_map`, so one seed gives different income categories under different standard
libraries (`static_linear_model.cpp:1952`, **B-05**). A config with no seed draws one from
`std::random_device` and records it as `0` (`mtrandom.cpp:8`, **B-06**).

**The energy balance admits a body fat mass below zero, and a year later the weight overflows.** The
partition coefficient `p = C / (C + F)` is undefined at `F = −2.001012658227848 kg`, and your code
floors a negative fat estimate at initialisation only (`kevin_hall_model.cpp:920`). On seed 80 of
`KevinHall_FINCH`, body fat reaches −2.224 kg in 2030 and weight −1.7×10²⁸³ kg in 2031, a number
your own statements reproduce. Body fat goes below zero on **four seeds in five hundred**, and on
three of those four both implementations finish and write the person out. **B-29**, report 6, traced
in [docs/findings/seed-80.md](findings/seed-80.md).

## The two example packs that do not run

`KevinHall_India` and `KevinHall_PIF` stop in their first simulated year, in both implementations
and on every seed. Both put the lower bound of `Weight` at **3.319358 kg**, and both give newborns a
weight below it from the shipped quantile curve on ordinary draws. Your binary stops at 3.1084 kg
and this build at 3.159 kg and 2.544 kg, on the first and the fifth person of the cohort, so no seed
or cohort size avoids it.

The two failures are one defect, because `India.DataFile.csv` and both weight-quantile files are
byte-for-byte identical between the directories. Either the bound or the curve is wrong, and
changing either means changing a fitted model. Report 1.

## The question we cannot answer

**Where should a policy that changes a nutrient intake be applied on the Kevin Hall surface: before
the energy balance runs, after it, or to the intake targets?**

Each answer gives a different number, and the choice belongs to whoever fitted the model. **B-25**
above hides the question today, by making such a policy a no-op in your build. This build refuses
the configuration at load time instead. Until the question is answered, four of your six examples
can only be compared with no policy active.

## How to reproduce the comparison

The comparison needs no binary of yours, because the reductions are checked in:

```bash
tests/equivalence/run.py --example HLM_France --seeds 20 --use-reference
```

Passing `--refresh-reference` runs your own binary instead; [docs/build-notes.md](build-notes.md)
records the reference build of Health-GPS `3.0.0.0`.

## What is being offered

Seven of the findings above belong to you rather than to this repository, and
[docs/upstream-reports.md](upstream-reports.md) writes each up with the command that reproduces it
on your binary and your data. Every deviation carries a switch that puts your behaviour back, so its
cost can be measured ([docs/deviations.md](deviations.md)). All of it is yours under BSD-3-Clause.
