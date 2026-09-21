# For the Health-GPS maintainers

Everything this project found in your code and in your example data, with the command that
reproduces each finding on your binary and your data. [docs/glossary.md](glossary.md) defines this
project's terms.

**What the labels mean.** A bracketed ID such as **P-04** is this project's handle for one finding.
The letter says where the finding came from, not how severe it is:

- **`B-nn`** — a defect in your **source**.
- **`N-nn`** — a way your build can give different results from one run to the next. Most of these are properties rather than bugs, and they matter here only because this project claims determinism.
- **`D-nn`** — a defect in the **example or data packs**.
- **`P-nn`** — a defect this **project** found afterwards, by running the two implementations against each other.

The first three come from an audit of your code and your data carried out on 2026-09-17, before the rewrite and each has an entry in [docs/audit/](audit/) to look up. 
A `P-nn` has no audit entry, because it was found later; its evidence is in [docs/deviations.md](deviations.md) and in this document.

No finding here depends on this repository's implementation being correct. Everything was measured against Health-GPS `3.0.0.0`, built as [docs/build-notes.md](build-notes.md) records, on macOS
26.6.2 with Apple clang 21, against the `imperialCHEPI/healthgps-examples` pack.

## What this is

This repository is a second implementation of Health-GPS in C++20, covering every model your own select. Its purpose is to run
both implementations on the same configurations and seeds and to report where their results differ.

## What the comparison proves, and how

Three of your six examples run to the end in both implementations and agree statistically:
`HLM_France`, `HLM_India` and `KevinHall_FINCH`. The comparison ran **62,030 tests and none
failed**, and no example has a failure budget ([docs/equivalence.md](equivalence.md)). The
comparison is made on *reductions*, not on result rows: every CSV a run writes is collapsed to one
number per variable, scenario, year, sex and output file, because the two implementations' age
bands hold different simulated people. One test compares twenty seeds of one such number against
twenty of yours. Your binary's reductions are checked into this repository, so the comparison
repeats without a build of your code ([docs/equivalence-method.md](equivalence-method.md) §2). The
pass rule is that **a correct build compared against itself has a 1% chance of reporting any
failure at all**, held there by Holm's correction. That rate was measured before adoption: 30 pairs
of twenty-seed runs, **922,564 tests, zero failures against 0.30 expected**
([docs/equivalence-method.md](equivalence-method.md) §4).

This build carries **885 C++ tests**, passing under release, debug and two sanitizer builds, and
**94 tests** over the harness's own statistics. CI runs **16 jobs and all 16 pass**
([docs/SUMMARY.md](SUMMARY.md)).

Five limits apply. `HLM_India` has only been compared at **12,406 people against the 1,240,613 it
ships**. The Kevin Hall energy-balance model has been compared on one country and one pack. The
population impact fraction has never met your binary; its only example cannot run. Your result files
carry six significant digits, so no comparison is tighter than 10⁻⁵ relative. Only `KevinHall_FINCH`
fills the result files split by income category, because nobody in an HLM example has one
([docs/equivalence.md](equivalence.md)).

## What is in this document, and how it is ordered

Two tests decide whether a defect appears here at all. The defect must be yours rather than a choice
this project made. And somebody running your binary must be able to see what it costs: a number that
is wrong in a published file, a run that reports success and applies no policy, or a result that
cannot be reproduced. Everything meeting both tests is below, in four groups.

1. **The findings, plainest first** — the eight the comparison against this implementation measured,
   stated in a paragraph each. Five of the eight are written up in full further down as reports 2,
   5, 6 and 7; the other three have their evidence in [docs/deviations.md](deviations.md).
2. **The seven reports** — the findings that need something from you, each with what it is, which
   files it is in, the command that reproduces it, what happens, what should happen, a suggested
   fix, and the evidence. Report 1 is the pair of examples that cannot run at all.
3. **The two tables after them** — defects that pass both tests but that the comparison did not
   measure, because they were found by auditing your source and by running your binary alone.
4. **The defects established by reading your code** — real, fixed in this build, and never yet seen
   to produce a wrong number.

A last section names what this document leaves out, so that nothing looks as though it was missed.

Two of the eight paragraphs below carry three defects each, so the eight paragraphs hold twelve
entries. Each of those two is one paragraph because it asks one thing of you rather than three.

## The findings, plainest first

**A `NaN` risk factor is counted as zero in the year's mean, and an infinity is not checked at
all.** Three people of 80 kg, one with a `NaN` weight, report 53.3 kg
(`analysis_module.cpp:385-391`). This build stops the run instead, naming the person, the year and
the factor; `--baseline-compat P-10` puts your behaviour back. **P-10**, report 7.

**Four demographic standard deviations have their square root taken twice**, because two loops in
`AnalysisModule::calculate_standard_deviation` finish the same channel. `std_region` is **0.122097**
where the band's spread is 0.745, and `std_ethnicity`, `std_income` and `std_sector` go the same
way. This build reproduces the defect on purpose and a test pins it: these numbers may
already be published, so correcting them is your decision rather than a reimplementation's.
Report 5.

**The food-labelling policy applies its impact more than once** to anybody the policy misses in one
year and reaches in a later one, because `try_emplace` cannot overwrite their existing
"unaffected" mark (`food_labelling_scenario.cpp:47-56`). This build applies the impact once, and
switching your behaviour back with `--baseline-compat P-04` is how the cost was measured:
**+0.209% of mean male BMI on `HLM_France`** and +0.160% on `HLM_India`. **P-04**, with the full
measurement in [docs/deviations.md](deviations.md).

**An intervention selected with the Kevin Hall or static-linear models has no effect, and the run
reports success.** `Scenario::apply` has one call site in your tree, and neither
`kevin_hall_model.cpp` nor `static_linear_model.cpp` calls it. `KevinHall_FINCH` with `simple` and
with `marketing` writes identical files. This build refuses the configuration at load time,
because where the policy *should* apply is the open question this document ends on. **P-05**,
report 2.

**Immigration into an empty age-sex band abandons that band's target silently.** On `HLM_France`,
**197 of your band counts** fall short of the demographic projection, at ages 93 to 100, and every
one of those bands holds zero people. This build keeps the rule — cloning a donor from a
neighbouring age would change the age distribution — but reports the shortfall in every year's run
metrics. **P-01**, with the measurement in [docs/deviations.md](deviations.md).

**Three defects on the population impact fraction path.** A fraction the data store cannot supply is
skipped without any message, and a gap inside a supplied table reads as exactly zero (**P-06**). The
published schema gives the sex column as 0 for female and 1 for male, the opposite of your loader
and your data (**P-07**). In `KevinHall_PIF`, 4 of 15 diseases have no `Smoking` table. This build
makes each of these a located error at load time. The schema is the one that needs your edit,
because your loader and your data already agree with each other.
[docs/deviations.md](deviations.md) carries the evidence for all three.

**Three smaller defects, all fixed here.** `std_income` is written as a column and never filled:
zero in all 4,884 rows of a `KevinHall_FINCH` run (**P-02**). An income distribution is built by
iterating an `std::unordered_map`, so one seed gives different answers under different standard
libraries (`static_linear_model.cpp:1952`, **B-05**). A config with no seed draws one from
`std::random_device` and records it as `0` (`mtrandom.cpp:8`, **B-06**). This build accumulates the
column, samples only from ordered sequences, and requires a seed.

**The energy balance admits a body fat mass below zero, and a year later the weight overflows.** The
partition coefficient `p = C / (C + F)` is undefined at `F = −2.001012658227848 kg`, and your code
floors a negative fat estimate at initialisation only (`kevin_hall_model.cpp:920`). On seed 80 of
`KevinHall_FINCH`, body fat reaches −2.224 kg in 2030 and weight −1.7×10²⁸³ kg in 2031. Body fat
goes below zero on **four seeds in five hundred**, and on three of those four both implementations
finish and write the person out. This build integrates the year only as far as body fat reaches
zero, which leaves 496 of 499 seeds byte-identical and moves three by at most 0.85% in `std_weight`;
`--baseline-compat P-09` puts your behaviour back. **P-09**, report 6, traced in
[docs/findings/seed-80.md](findings/seed-80.md).

## The seven reports

Seven findings need something from you. Each is written in the same shape: what it is, which files
it is in, the command that reproduces it, what happens, what should happen instead, a suggested fix,
and the evidence behind it.

| | Report | Needs |
|---:|---|---|
| 1 | `KevinHall_India` and `KevinHall_PIF` stop in their first simulated year | a decision about a fitted model |
| 2 | An intervention selected with the Kevin Hall or static-linear models has no effect | a decision about a fitted model |
| 3 | `KevinHall_FINCH`'s legacy `static_model.json` names two files the pack lacks | a decision, or a deletion |
| 4 | The baseline exits on a signal on about one `KevinHall_FINCH` run in 250 | a concurrency fix |
| 5 | Four demographic standard deviations have their square root taken twice | a code fix, and a decision about published numbers |
| 6 | The energy balance admits a body fat mass below zero and then overflows | a code fix |
| 7 | A `NaN` risk factor is counted as zero in the year's mean | a code fix |

Reports 6 and 7 are the two this build has already guarded against. Both are still upstream defects,
because a guard added in a reimplementation does nothing for anybody running yours. Reports 6 and 7
are also the two worth sending first, because both are defects in code you are running today and
neither needs a decision from anybody: one is a missing floor that the same file already applies
fifty lines earlier, and the other is deleting four lines that turn a `NaN` into a zero.

---

## 1. `KevinHall_India` and `KevinHall_PIF` stop in their first simulated year

**Summary.** Both examples give a newborn a weight below the lower bound their own configuration
sets, so neither example can complete a run in any implementation.

**Affected.**

- `KevinHall_India/new_config.json` and `KevinHall_PIF/new_config.json`, at
  `modelling.risk_factors.Weight.range[0]` = **3.319358** kg.
- `KevinHall_India/weight_quantiles_NCDRisk_male.csv` and `..._female.csv`, and the identical pair
  in `KevinHall_PIF/`.
- The check that stops the run: `KevinHallModel::validate_weight_in_config_range`,
  `kevin_hall_model.cpp:1191-1193`.

**Reproduce.**

```bash
HealthGPS.Console -f KevinHall_India/new_config.json
```

**What happens.** The run stops in the first simulated year, on every seed. Your binary dies on an
uncaught exception. This build stops with an error naming the person, the value and the bound.

| Example | Implementation | Who failed | Weight | Floor |
|---|---|---|---:|---:|
| `KevinHall_India` | baseline | (not named) | 3.1084 kg | 3.319358 kg |
| `KevinHall_India` | this build | person 1, male, age 0 | 3.159 kg | 3.319358 kg |
| `KevinHall_PIF` | baseline | (not named) | 3.1084 kg | 3.319358 kg |
| `KevinHall_PIF` | this build | person 5, male, age 0 | 2.544 kg | 3.319358 kg |

**What should happen.** Both examples run to their horizon. A pack should not ship a weight bound
and a birth-weight curve that contradict each other.

**Suggested fix.** Either the bound or the curve is wrong, and only you can say which. A 2.5 kg
newborn is an ordinary low-birthweight baby, which suggests the configured 3.32 kg is a mean rather
than a minimum; the same configured range has 92.56 kg at the top, which is not a maximum either.
Changing either number here would mean inventing a number for a fitted model.

A second, smaller problem is in the same file. `KevinHall_India/new_config.json` carries both a
deprecated root `trend_type` of `"UPFTrend"` and a `project_requirements.trend.type` of
`"income_trend"`. Your own loader refuses that combination: *"Deprecated root-level trend_type
and/or income_categories are not allowed when project_requirements is present"*. The file is
therefore unloadable by the validator shipped beside it.

**Evidence.** A newborn's weight is an expected birth weight multiplied by a quantile drawn from the
shipped curve: 9,995 multipliers per file, spanning 0.668034 to 1.916892 for males and 0.655385 to
1.902204 for females. The product falls below the bound on ordinary draws rather than tail ones, and
the failing person is the first or the fifth of the cohort, so no seed, cohort size or horizon
avoids it.

The two failures are one defect rather than two: `India.DataFile.csv` and both weight-quantile files
are byte-for-byte identical between the two example directories.

This blocks two of your six examples, the whole Kevin Hall and India data family, and the only
population-impact-fraction comparison there could be. `KevinHall_PIF` loads completely in this
build, all 69 fraction tables and both scenarios, and then stops on this.

---

## 2. An intervention selected with the Kevin Hall or static-linear models has no effect

**Summary.** A configuration that selects an intervention alongside either of those models runs to
completion, reports success, and produces output byte-identical to a run with no policy at all.

**Affected.** `Scenario::apply` has exactly one call site in the whole tree,
`dynamic_hierarchical_linear_model.cpp:110`. Neither `static_linear_model.cpp` nor
`kevin_hall_model.cpp` calls it.

**Reproduce.** With any two of the six intervention types the example defines:

```bash
# in KevinHall_FINCH/new_config.json, set interventions.active_type_id = "simple", then run:
HealthGPS.Console -f KevinHall_FINCH/new_config.json
# set it to "marketing", run again with the same seed, and compare the two result files
```

**What happens.** The two result files are identical. Nothing in the output, the log or the exit
code says the policy was never applied.

**What should happen.** Either the policy applies to the selected models, or the configuration is
refused at load time.

**Suggested fix.** The fix depends on an answer only you have: is a policy *meant* to apply
alongside those models? If it is, then **where** it applies changes the result — before the energy
balance runs, after it, or to the intake targets — and that is a modelling decision with a fitted
model behind it. If the inertness is a defect, the configuration should be refused rather than run.
This build refuses it, naming the intervention, the number of impacts it declares and the dynamic
model file.

**Evidence.** All six intervention types were run with those models. In every pair, the baseline
future and the intervention future wrote identical files. The single call site can be found by grep
in both source trees. This blocks the comparison of four of your six examples with any policy that
does anything.

---

## 3. `KevinHall_FINCH`'s legacy `static_model.json` names two files the pack does not contain

**Summary.** The legacy configuration for that example refers to two data files that are not in the
example directory, so the legacy path cannot be loaded as shipped.

**Affected.** `KevinHall_FINCH/static_model.json`, which names
`Finch_residual_policy_covariance.csv` and `policyeffect_model.csv`. The pack ships seven
scenario-prefixed variants of each instead, `S1_` through `S7_`.

**Reproduce.**

```bash
HealthGPS.Console -f KevinHall_FINCH/config.json
```

**What happens.** The load fails: *"Could not find file …/KevinHall_FINCH/
Finch_residual_policy_covariance.csv"*, followed by *"Failed with message: Could not load input file
info."*

**What should happen.** The legacy configuration loads, or the pack no longer ships it.

**Suggested fix.** Point `static_model.json` at one of the seven scenario variants, or delete the
file as superseded by `new_static_model.json`, which names the `S1_` pair. Either answer is useful;
the second saves the next reader the hour this one cost.

**Evidence.** Recorded as audit finding D-02 in
[docs/audit/02-data-and-examples.md](audit/02-data-and-examples.md). The missing files are part of
the example rather than of the pinned data release. This build reads the example's modern
`new_config.json` instead and records that as a scope decision ([ADR
0030](decisions/0030-policy-scenario-selection-for-the-broken-finch-example.md)).

---

## 4. The baseline exits on a signal on about one `KevinHall_FINCH` run in 250

**Summary.** A small fraction of `KevinHall_FINCH` runs die on `SIGTRAP` or `SIGABRT`, and every one
of them succeeds when re-run with the same binary, configuration and seed.

**Affected.** `repository.cpp:56-58` against `:63` and `:134`. A read path on `std::map diseases_`
takes no lock, and races an insert on another thread that does, reached from `disease.cpp:79`. Audit
findings B-01 and B-02, in [docs/audit/04-baseline-issues.md](audit/04-baseline-issues.md).

**Reproduce.** Run the example in a loop and count. At about one failure in 250, sixty runs is
roughly an hour and may show none:

```bash
for i in $(seq 1 60); do HealthGPS.Console -f KevinHall_FINCH/new_config.json || echo "FAILED $i"; done
```

**What happens.** The process exits on a signal, having written nothing usable. A re-run of the same
seed succeeds.

**What should happen.** A run either completes or fails deterministically.

**Suggested fix.** Take the lock on the read path, or populate the disease repository before the
parallel section rather than lazily from inside it.

**Evidence.** Two measurements, and the larger is the better one. Of **500** single-threaded
`KevinHall_FINCH` runs, **2 exited on a signal** and both succeeded on the retry. An earlier
measurement of **180** runs made for this project's equivalence comparison saw **4** — three
`SIGTRAP`, one `SIGABRT` — which is the one in forty-five quoted in earlier versions of this report.
`HLM_France` ran 80 times over the same comparisons without a single failure, so the FINCH example
is what provokes it. ThreadSanitizer names the race directly: locked write at `repository.cpp:134`
against unlocked read at `repository.cpp:57`, same address, both via `disease.cpp:79`.

This build has not exited on a signal once across the same runs, which is what running the two
scenarios sequentially buys ([ADR
0009](decisions/0009-sequential-scenarios-and-the-migration-journal.md)). The comparison harness
retries a baseline run up to three times and prints every retry, so the flake stays visible in the
record.

---

## 5. Four demographic standard deviations have their square root taken twice

**Summary.** Each output column's accumulated squared deviations are finished in one of two loops. A
column named in both loops is finished twice, so its standard deviation is the square root of a
number that is already a standard deviation.

**Affected.** `AnalysisModule::calculate_standard_deviation` in `analysis_module.cpp`, and the same
shape in `AnalysisModule::calculate_income_based_standard_deviation`,
`analysis_module.cpp:1735-1800`.

**Reproduce.** Run the example and read the band with the most people in it. `std_region` should be
the spread of a 1-to-4 category over that band, and is the square root of that spread over the head
count instead:

```bash
HealthGPS.Console -f KevinHall_FINCH/new_config.json -T 1
```

**What happens.** In a `KevinHall_FINCH` result file, `std_region` is **0.122097** in a band of 50
people whose `mean_region` is 1.38. Region takes the values 1 to 4, so that band's spread is about
0.745, and `sqrt(0.745 / 50)` is 0.122097.

**What should happen.** Each channel's accumulated squared deviations are finished exactly once.

**Suggested fix.** Finish each channel once: either drop the explicit demographic list where the
name is already in the mapping, or mark a channel as finished when `divide_by_count_sqrt` writes it
back.

**Evidence.** `divide_by_count_sqrt` lower-cases the name and writes back in place:

```cpp
const double sum = series.at(sex, std_channel).at(age);
series.at(sex, std_channel).at(age) = std::sqrt(sum / count);
```

The first loop walks every mapping entry:

```cpp
for (const auto &factor : context.mapping().entries()) {
    divide_by_count_sqrt(factor.key().to_string(), core::Gender::female, age, count_F);
    divide_by_count_sqrt(factor.key().to_string(), core::Gender::male, age, count_M);
}
```

The second is an explicit list of demographic channels: `age`, `age2`, `age3`, `gender`, `region`,
`ethnicity`, `sector`, `income`, `income_category`, `physical_activity`. A channel reached by both
gets `sqrt(sqrt(sum_of_squares / n) / n)`.

`KevinHall_FINCH` declares `Region`, `Ethnicity`, `Income`, `income_category`, `Age`, `Age2`, `Age3`
and `Gender` as level-0 risk factors, which means they are drawn before anything that depends on
them, so all eight are in both lists. Four of the eight are zero either way: `mean_age` is the
band's own age and `mean_gender` the file's own sex, so those deviations are zero before anything is
square-rooted, and `income_category` is constant within a stratum file. The four that carry a number
are `std_region`, `std_ethnicity`, `std_income` and, where a pack assigns it, `std_sector`.
`std_bmi` is in the mapping only, is finished once, and is correct. No mean is affected and no other
column is.

This build **reproduces** the behaviour rather than correcting it, in the whole-population series
and in the income-stratified one, because a standard deviation is a number somebody may have
published and the correction is a decision for the model's owners. The test
`AnalysisIncomeSeries.ADemographicThatIsAlsoADeclaredFactorHasItsSquareRootTakenTwice` holds the
reproduced behaviour in place, so that nobody corrects it here by accident.

---

## 6. The energy balance admits a body fat mass below zero, and a year later the weight overflows

**Summary.** `KevinHallModel::kevin_hall_run` integrates body fat straight through zero. Below
−2.001 kg the model's own partition coefficient is undefined, and the next year's weight is a number
like −1.7×10²⁸³ kg.

**Affected.**

- `KevinHallModel::kevin_hall_run`, the yearly step, in `kevin_hall_model.cpp`.
- `initialise_kevin_hall_state`, `kevin_hall_model.cpp:920`, which floors a negative body fat
  estimate at `F = 0.2 * BW` and is the only place that floor is applied.
- `validate_weight_in_config_range`, `kevin_hall_model.cpp:1191-1193`.

**Reproduce.** The rate is about one seed in five hundred of `KevinHall_FINCH`, and your binary's
cohorts are not this build's, so the reproduction is the arithmetic rather than a seed. Feed
`kevin_hall_run`'s own statements this state, taken from one real person in one real run:

```
age 24, male, BW_0 55.848205648994693, H 171.67238876297318, PAL 1.7771963443015526,
CI_0 405.23122522563074, CI 583.79583355280693, EI_0 3030.6442817726597, EI 4828.6449430126222,
G_0 0.92450226451031092, Na_0 4.505747672185052, Na 4.4136891989841027,
ECF_0 17.202140148866199, F_0 -2.2324094172489493, L_0 37.457816538689301, K -3048.8515237582928
```

**What happens.** The call returns `BW = -1.7019180456941172e+283`. The yearly step is the
closed-form solution of a two-compartment relaxation, one of whose coefficients is

```
p = C / (C + F),     C = 10.4 * rho_L / rho_F = 2.001012658227848 kg
```

which is undefined at `F = −C`. Below that point `p` is negative, `a1*b2 − a2*b1` is negative, `tau`
is negative, and `exp(-365.0 / tau)` grows away from the steady state instead of decaying towards
it. At a `tau` of −0.559 days, `365 / tau` is −652, so one call multiplies the state by e to the
power of 652.

**What should happen.** The model is evaluated only where it is defined. A body fat mass that has
reached zero does not continue downwards into a region where the model's own coefficient has changed
sign.

**Suggested fix.** Apply the floor that `initialise_kevin_hall_state` already applies at
`kevin_hall_model.cpp:920`, for the same stated reason, to the yearly update as well. This build
instead integrates the year only as far as the model is defined, to the instant body fat reaches
zero, which is `exp(-t*/tau) = F* / (F* - F0)`, with lean tissue taken to the same instant ([ADR
0049](decisions/0049-the-energy-balance-is-integrated-only-where-it-is-defined.md)).

**Evidence.** On seed 80 of `KevinHall_FINCH`, the year before the overflow, the same person's step
returns `F = -2.2240198893612395` from an ordinary `F_0` of 5.86 kg, after a 30% fall in energy
intake and a rise in physical activity in the same year, both of which the fitted models draw. The
whole trace is [docs/findings/seed-80.md](findings/seed-80.md).

In **500 seeds** of `KevinHall_FINCH`, body fat went below zero four times and the weight overflowed
once. The sign of the runaway is what makes it worth reporting at that rate:
`validate_weight_in_config_range` **throws** below the configured minimum, and **prints a warning
and returns** above the maximum. A runaway upward therefore leaves your binary exiting **zero**,
with an impossible number in the results file. That was confirmed directly by running
`KevinHall_FINCH` with `Weight.range` set to `[1, 50]`: **67,668 `[WEIGHT RANGE WARNING]` lines,
exit code 0**, results written.

This build's guard changes nothing over the same 500 seeds: 499 are byte-identical with it on and
off, and the five-hundredth is the difference between a results file and no results file. Every
occurrence is recorded in the JSON manifest written beside the results, naming the person and the
year.

---

## 7. A `NaN` risk factor is counted as zero in the year's mean

**Summary.** `AnalysisModule::calculate_historical_statistics` substitutes `0.0` for a `NaN` risk
factor and divides by the whole head count, so the reported mean is wrong and nothing says so.

**Affected.** `analysis_module.cpp:385-391`.

**Reproduce.** Any model that can produce a `NaN` will do it; the shortest path is a risk factor
whose `range` is absent and whose linear model produces `log(0)`. The eight lines are unambiguous
and faster to read:

```cpp
double factor_value = 0.0;
if (entity.risk_factors.contains(item.first)) {
    factor_value = entity.risk_factors.at(item.first);
    if (std::isnan(factor_value)) {
        factor_value = 0.0;
    }
}
item.second[entity.gender] += factor_value;
```

**What happens.** A `NaN` becomes a zero, is added to the sum, and the sum is divided by the whole
head count. A cohort of three people weighing 80 kg, one of whom has a `NaN` weight, reports a mean
weight of **53.3 kg**. Nothing in the output, the log or the exit code records the substitution.
`std::isnan` is false for `±inf`, so an infinite risk factor is not examined at all and the band's
mean becomes infinite, which at least is visible.

**What should happen.** A value that cannot describe a person does not reach an output file. The run
stops, or the value is excluded and the exclusion is reported.

**Suggested fix.** Delete the four lines that substitute zero, and test `std::isfinite` instead of
`std::isnan`. The check costs one instruction on a value already in a register.

**Evidence.** `calculate_historical_statistics` is the last code that sees a value together with the
person it belongs to. A substitution there turns a defect anywhere upstream of it, including report
6, into a results file that is quietly wrong rather than a run that fails loudly.

This build stops the run with an error naming the person, the year, the factor and the value. Values
with a physical meaning are bounded as well as checked for finiteness: a weight must be between a
gram and a tonne, and energy intake must not be negative. Those bounds sit far outside any
configured modelling range, so crossing one is evidence of a defect rather than of an unusual draw.
The behaviour is switchable with `--baseline-compat P-10`, so the difference it makes can be
measured ([ADR 0050](decisions/0050-no-output-carries-a-number-that-cannot-exist.md)).

---

## More defects in your code, which the comparison did not measure

Each of these is yours, each has a demonstrated consequence, and none of them needs a decision about
a fitted model. They are separated from the eight findings above because they were found by auditing
your source and by running your binary alone, rather than by holding the two implementations against
each other. None is written up as a report, because none of them needs anything from you beyond a
fix.

| | What | Where | What it costs |
|---|---|---|---|
| **P-03** | `project_requirements.two_stage.use_logistic` is read into the configuration structure and then consulted nowhere. The two-stage first step runs whenever the model file has a `logistic_regression` entry, whatever the flag says | the configuration structure and every use of the field | the FINCH pack sets the flag to `false`, ships the file, and is fitted to the behaviour with the first step **on**. Running `KevinHall_FINCH` both ways moves `mean_redmeat` by **30%**. This build lets the model file decide, because that is what the data was fitted to, and reports the disagreement as a warning naming which one decided |
| **B-02** | a lock-free read path on the shared disease map races a locked insert made from another thread | `repository.cpp:56-58` against `:134`, both reached from `disease.cpp:79` | about **one `KevinHall_FINCH` run in 250** exits on `SIGTRAP` or `SIGABRT` having written nothing usable, and the same seed succeeds on a re-run: 2 of 500 runs measured here. ThreadSanitizer names the race directly. Written up as report 4 |
| **B-01** | the baseline and intervention scenarios run on concurrent threads and publish result rows asynchronously | `runner.cpp:85-92`, `event_monitor.h:47-53` | three runs of one seed produced three different file hashes, and sorting made them byte-identical. No value moved, so the cost is that two runs of one configuration cannot be compared as files |
| **P-08** | `data_root_path` is required by the population impact fraction schema and has its `${VAR}`s expanded, then is overwritten with the data store's own root before anything reads it | `repository.cpp:118` | a configuration that points the field anywhere else has been misled, with nothing said. The field has never done anything |
| **B-09** | the `-s/--storage` option cannot be used: the program errors unless exactly one of `-s` and `config.data` is present, and the published schema makes `data` required | `program.cpp:130-135` against `schemas/v1/config.json` | running `-s <dir>` with an otherwise valid config gives *"Invalid configuration - : Required property 'data' not found."* |
| **B-08** | `output.file_name` is ignored unless the configured name contains a `{…}` token | `configuration.cpp:231-261` | configuring `"file_name": "result.json"` produces `HealthGPS_result_<timestamp>.json` |
| **B-11** | 35 of your 471 tests skip silently, on an undocumented path relative to `__FILE__` that matches the layout of neither data repository | `KevinHallHeight.Test.cpp:21-26` and four sibling files | `[ PASSED ] 436 tests. [ SKIPPED ] 35 tests`. A green run proves less than it appears to, across 7.4% of the suite |

## Defects in the example data packs

These belong to whoever owns `healthgps-examples` and `healthgps-data`. The first two change which
numbers a run draws; the rest cost a reader time rather than accuracy.

| | What | What it costs |
|---|---|---|
| **P-11** | `pif-data-v5` ships `diseases/COPD/`, holding nothing but a `PIF/` subtree, for a disease its own registry calls `pulmonary` | `COPD` and `pulmonary` carry **different** `Smoking` tables for what is clinically one disease: maxima 0.00014 and 0.032, and 150 non-zero cells against 2,880. Only `pulmonary` is reachable, because that is what the registry lists, so the `COPD` tables are either dead or the ones that were meant |
| — | `France.DataFile.csv` declares `Age` as an integer column and gives **80 of its 40,000 rows** a fractional age. `std::stoi` stops at the first non-digit, so each one is truncated toward zero in silence | truncation is what the data means, because the file's own `Age1` column carries `floor(Age)`. The silence is the part that is wrong. This build performs the same arithmetic and reports it once per column, with a count and the first offending value |
| **D-02** | `KevinHall_FINCH/static_model.json` names `Finch_residual_policy_covariance.csv` and `policyeffect_model.csv`, and the pack ships seven scenario-prefixed variants of each instead | the legacy configuration cannot be loaded as shipped. Written up as report 3 |
| **D-04** | the six examples name **five mutually exclusive data releases**, and nothing in either repository records which release a given working copy holds | no single checkout satisfies all six examples |
| **D-08** | a missing model key prints `Missing key "policy_start_year"` with no file name, no severity marker, and no statement of the value used instead | a silent default in a policy model is invisible to whoever reads the output |
| **D-06** | `diseases/PIF_Mappings.xlsx` is the only spreadsheet in a machine-read tree. No code reads it and no manifest names it | human documentation that will drift from the CSVs it describes, with nothing able to catch the drift |

## Defects established by reading your code rather than by measurement

Nine more are recorded in [docs/deviations.md](deviations.md) and fixed in this build. Each one is a
code path rather than a measurement, so none of them has been shown to produce a wrong number, and
ranking the effort of fixing them is yours.

`Identifier::operator==` compares only the cached 64-bit hash while the defaulted `operator<=>`
compares the string, so `a == b` and `a < b` can hold at once (**B-04**). Eleven `<cctype>` calls
take a plain signed `char`, which is undefined behaviour for any byte at or above `0x80`
(**B-03**). `next_int` is documented half-open and implemented inclusive, and all four call sites
compensate (**B-07**). `Population::add` is `noexcept` while calling `emplace_back` and `at()`, so
an allocation failure becomes `std::terminate` (**B-12**), and it rescans the population from index
0 on every call, once per migrant per age per sex per year (**B-13**). `next_empirical_discrete`
accepts two empty vectors and then reads `values.back()` (**B-14**). The polar normal draw rejects
`p >= 1.0` but not `p == 0.0`, so `log(0)` can carry a `NaN` into a risk factor (**B-15**).
`next_double` uses `std::generate_canonical`, which has returned exactly 1.0 on some
implementations, and the integer sampler would then return `max + 1` (**N-05**). Population
reductions accumulate into a shared table under a mutex, which makes them race-free but not
order-stable (**N-07**).

## What this document leaves out

Four kinds of thing are missing on purpose. Naming them is cheaper than leaving you to wonder
whether they were found.

**Choices this project made.** Scenarios run sequentially, result rows have a defined order, the
timestamp lives in the JSON metadata rather than in the file name, `project_requirements` is
required, and an intervention that no configured model would apply is refused at load time. Those
are this repository's decisions and not claims about your code
([docs/deviations.md](deviations.md), *Design differences*).

**Differences that move no number.** One exception type split into two, an explicit Cholesky
decomposition in place of Eigen, a maintained population counter, and a column order that follows
the configuration rather than the alphabet. A reader comparing the two trees will notice them, and
none of them changes a result.

**Housekeeping.** A production log line reading `FINISHED ALL THE LOADING REQUIRED CUTIEPIE :)`
(**B-16**); 157 `MAHIMA:` comments, a dead debug block, and a top-level directory named after a
person (**B-17**, **D-07**); a project version of `3.0.0.0` against a vcpkg manifest saying
`1.2.2.0`, where the results file reports the former (**B-18**); three atomic progress counters
incremented and never read (**B-19**); and 177 MB of committed simulation output, which is 62% of
the examples repository (**D-05**). All confirmed, none of them a defect in a result.

**Portability.** Your tree does not build on macOS: an `#error "Unsupported platform"`, plus
`std::execution::par` and `std::osyncstream`, neither available in Apple libc++ (**B-10**). This
project wrote four shims rather than a report, because whether to support macOS is your decision
and the shims are recorded in [docs/build-notes.md](build-notes.md).

## The question we cannot answer

**Where should a policy that changes a nutrient intake be applied in the Kevin Hall model: before
the energy balance runs, after it, or to the intake targets?**

Each answer gives a different number, and the choice belongs to whoever fitted the model. **P-05**
and report 2 hide the question, by making such a policy do nothing in your build. This build refuses
the configuration at load time instead. Until the question is answered, four of your six examples
can only be compared with no policy active.

## How to reproduce the comparison

The comparison needs no binary of yours, because the reductions are checked in:

```bash
tests/equivalence/run.py --example HLM_France --seeds 20 --use-reference
```

Passing `--refresh-reference` runs your binary instead; [docs/build-notes.md](build-notes.md)
records the reference build of Health-GPS `3.0.0.0`.

## What is being offered

Every finding above is yours rather than this repository's, and the seven reports are where to
start. Everything else recorded about the two implementations is in
[docs/deviations.md](deviations.md), with the evidence for each finding and the test that pins this
build's behaviour. Every deviation that changes a number carries a switch that puts your behaviour
back, so its cost is measured rather than argued
([ADR 0041](decisions/0041-deliberate-deviations-are-switchable.md)). All of it is yours under
BSD-3-Clause.

## Related records in this repository

[docs/deviations.md](deviations.md) lists every behaviour where this build differs from yours
deliberately, each with its evidence and the test that pins it. Each deviation that changes a number
has a switch that restores your behaviour, so its effect can be measured ([ADR
0041](decisions/0041-deliberate-deviations-are-switchable.md)).
