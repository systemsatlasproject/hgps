# Seven bug reports for the Health-GPS maintainers

Seven findings belong to whoever owns the Health-GPS code and the example data packs rather than to
this repository. A `B-nn` or `D-nn` label below is the identifier an earlier audit of that code gave
the defect ([docs/audit/](audit/)). Each one is written in the same shape: what it is, which files
it is in, the command that reproduces it, what happens, what should happen instead, a suggested fix,
and the evidence behind it.

Everything here was measured against Health-GPS `3.0.0.0`, built as
[docs/build-notes.md](build-notes.md) records, on macOS 26.6.2 with Apple clang 21, against the
`imperialCHEPI/healthgps-examples` pack. No report depends on this repository's implementation being
correct: every one reproduces with your binary and your data.

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
because a guard added in a reimplementation does nothing for anybody running yours.

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
The behaviour is switchable with `--baseline-compat B-30`, so the difference it makes can be
measured ([ADR 0050](decisions/0050-no-output-carries-a-number-that-cannot-exist.md)).

---

## Related records in this repository

[docs/deviations.md](deviations.md) lists every behaviour where this build differs from yours
deliberately, each with its evidence and the test that pins it. Each deviation that changes a number
has a switch that restores your behaviour, so its effect can be measured ([ADR
0041](decisions/0041-deliberate-deviations-are-switchable.md)).
