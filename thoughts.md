# Thoughts

Scratch notes: things I noticed reading the tree that are **not** already recorded in
`docs/deviations.md`, `docs/backlog.md`, `docs/upstream-reports.md` or an ADR. Each one was checked
against those documents before it was written down.

Nothing here has been acted on. Nothing here has been *run* either — every claim below comes from
reading code, so the confidence column matters. Items marked "would need a run to confirm" are
exactly that.

Ranked roughly by what I'd look at first.

---

## 1. `policy_start_year`'s documented default is implemented nowhere

**Kind:** correctness, latent. **Confidence:** high — three documents state the rule and no code
implements it.

Three places state the same contract:

| Where | What it says |
|---|---|
| `src/config/types.h:107` | "The calendar year policies start. 0 means `start_time + 2`, as upstream." |
| `src/model/model_input.h:39` | the same sentence |
| `schemas/v2/config/modelling.json` | `"description": "The calendar year policies start. 0 or omitted means start_time + 2."`, `"default": 0` |

The value is read in exactly one place, `src/model/riskfactor/static_linear/model.cpp:307`:

```cpp
const auto policy_start = static_cast<int>(context.inputs().run().policy_start_year);
const bool intervene = context.scenario().type() == sim::ScenarioType::intervention &&
                       context.time_now() >= policy_start;
```

`time_now()` is a calendar year. With `policy_start_year == 0`, `time_now() >= 0` is true in
**every** year — so a config that omits the key gets policies from the **first simulated year**,
not from `start_time + 2`. I grepped for `start_time + 2` across `src/`: the only hits are the two
comments above.

The loader (`src/config/loader.cpp:506-515`) validates the integer and stores it raw; nothing
substitutes. `tests/config/config_loader_test.cpp:446` asserts `EXPECT_EQ(0U, ...)` when the key is
absent — which pins the raw value, not the documented meaning, so the suite agrees the field is 0
and nothing anywhere asks what 0 means.

**How exposed is it today:** narrowly. All four Kevin Hall examples set `2024` explicitly, and the
HLM examples omit it but are on a surface where the field does nothing at all (item 3). So I don't
think any shipped config is currently wrong. The exposure is a StaticLinear-surface config, with
policy coefficients in its model file, that omits the key — and `tools/convert-config` passes the
`modelling` block through unchanged (`src/config/convert.cpp:203`) and never synthesises the field,
so a converted upstream config that omits it stays omitted. `ADR 0018`'s context notes that the
upstream baseline prints `Missing key "policy_start_year"` on HLM examples, so omission is a real
upstream shape rather than a hypothetical one.

**Decide which way it goes**, because both are defensible and the current state is neither:

- implement the documented default (`0 → start_time + 2`) in the loader, where it can be reported
  as an applied-default warning like every other default in this codebase; or
- delete the claim from all three places and say 0 means "from the first year".

The second is arguably better — a magic sentinel that means something other than its value is the
kind of thing this codebase otherwise refuses — but the first is what the schema currently
promises to anyone reading it, and the schema is published.

---

## 2. `active_type_id` does two unrelated jobs, and one of them is undocumented

**Kind:** design wart, with a silent-no-op edge. **Confidence:** high, traced end to end.

Whether a run has an intervention arm at all is decided by one thing — `session.cpp:248` builds
`intervention_modules` iff `config.running.active_intervention.has_value()`, and `session.cpp:426`
branches on that to pick the one-arm or two-arm `Runner::run` overload.

So `running.interventions.active_type_id` is doing two separate jobs:

1. choosing **which policy** runs (live only on the EBHLM surface — B-25), and
2. switching the run from **one arm to two**.

On the Kevin Hall surface only job 2 is live, which is why those four examples ship
`"active_type_id": "simple"` with `"impacts": []`. `ADR 0035` covers the empty impact list properly
— it's upstream's idiom for "no policy here" and accepting it is the right call. What I don't find
recorded anywhere is the **consequence for the other mechanism**:

> `modelling.policy_start_year` cannot produce a policy comparison on its own. It only ever fires
> inside an arm whose `type()` is `intervention`, and the only way to make such an arm exist is to
> name an intervention in a block that does nothing on that surface.

A config with `policy_start_year: 2024` and no `active_type_id` loads clean, runs single-arm, and
the policy **silently never fires** — no warning, no error. That is the same class of thing B-25
is, in the opposite direction, and B-25 is a load-time error here.

**Worth doing, and it needs no upstream ruling** (unlike backlog item 1, which is about where a
policy should *apply* on that surface — a different question). Let *any* configured policy
mechanism be sufficient to build the intervention arm. Then `KevinHall_FINCH`'s config says what it
actually means, the ceremonial `simple` block can go, and a reader stops concluding that the FINCH
runs have no policy in them when in fact they do.

Minimum version if that's too invasive: a diagnostic when `policy_start_year` is set and no
intervention arm will exist, in the same shape as
`check_intervention_reaches_the_model` (`src/config/models/model_loader.cpp:149`).

---

## 3. `policy_start_year` on the HLM surface is silently inert — the unguarded mirror of B-25

**Kind:** missing validation. **Confidence:** high for "nothing checks it"; would need a run to see
what a user actually experiences.

`policy_start_year` is consumed in exactly one place in the whole tree —
`static_linear/model.cpp:307`, inside `StaticLinearModel`. The HLM/EBHLM classes never read it.

So setting `policy_start_year` on an `HLM_France`-shaped config is a no-op with no diagnostic. The
loader validates the *year* (`loader.cpp:506-515`) and never cross-checks it against the configured
model family.

This is structurally identical to B-25 — a policy mechanism configured on a surface that doesn't
implement it — and this project treats that as a **load-time error** worth an ADR of its own. The
asymmetry is just that one mechanism got the check and the other didn't:

| mechanism | on a surface that ignores it |
|---|---|
| `running.interventions.active_type_id` | error, or warning if impacts are empty (ADR 0035) |
| `modelling.policy_start_year` | **nothing** |

The fix is symmetric with what's already there: `applies_the_active_scenario()` is the existing
pattern of "the model answers, the loader doesn't match on a name". The same shape would be a model
answering whether it honours `policy_start_year`. That generalises the ADR 0035 idea from one
boolean about one mechanism to a per-family declaration of which mechanisms it honours — which is
probably the right destination anyway if a fifth model family ever arrives.

No shipped config hits this (the HLM examples omit the key), so it's a trap rather than a live bug.

---

## 4. `docs/design.md` gets the thread default wrong

**Kind:** documentation error. **Confidence:** certain.

`docs/design.md:302` says the thread pool is "sized from `--threads` (default: hardware
concurrency, capped)".

It isn't. The default is **1**, in every other place that states it:

- `src/app/options.h:25` — `std::size_t threads{1}`
- `src/app/options.cpp:47` — help text, "(default 1)"
- `include/hgps/engine.h` — `RunOptions::threads{1}`
- `README.md` — "the default is one"

It also contradicts the table two paragraphs above it in the same document ("Sequential by
default") and the §5 rationale that follows it, which argues *for* a default of one on the grounds
that the baseline gained nothing from threading. So it reads like a leftover from an earlier
intention rather than a considered claim.

One-line fix. Flagging it because `design.md` is described as the architectural contract and is
first on the reading path, so a wrong number there propagates.

---

## 5. `docs/backlog.md` item 11 is stale — `person.sector` *is* assigned

**Kind:** stale documentation. **Confidence:** high.

Backlog item 11 (`docs/backlog.md:227`) says:

> `person.sector` (urban/rural) is assigned nowhere; the channel appears if the mapping declares
> the factor.

But `StaticLinearModel::initialise_sector` assigns it — `src/model/riskfactor/static_linear/income.cpp:199`,
drawing against the model's `RuralPrevalence` table — and it is called from two places in
`static_linear/model.cpp` (lines 262 and 317, initialisation and newborns). `model.cpp:136` also
declares it in `AssignedAttributes`, which is what makes the output channel appear. The design doc
lists sector as one of the things `income.cpp` owns.

What's true is that **the HLM surface** assigns no sector — its models don't have the concept. So
the item is probably a correct observation about HLM that got written as a global one, or it
predates the static linear work.

Low stakes, but worth correcting rather than deleting: the backlog is explicitly the document that
"makes the next decision easy", and an item claiming a feature is missing when it exists is the
kind of thing that gets someone to implement it twice.

---

## 6. `clone_for_immigration` hand-enumerates fields and misses two

**Kind:** latent hazard. Currently benign — I checked. **Confidence:** high that it's benign today,
high that it's fragile.

`Engine::clone_for_immigration` (`src/sim/engine.cpp:217`) copies a donor person field by field.
It copies `age, gender, region, ethnicity, ses, sector, income, income_continuous,
physical_activity, risk_factors, diseases`.

It does **not** copy `income_adjustment_stratum` or `has_income_adjustment_stratum`
(`src/model/person.h:70-71`), so every immigrant arrives with `has_... == false`.

**Why it doesn't currently matter:** `assign_adjustment_strata`
(`static_linear/income.cpp:168-183`) clears every person's flag and re-derives the whole
population's strata each year, and it runs inside `StaticLinearModel::update_risk_factors` — which
runs *before* the dynamic model (`risk_factor_model.cpp:396-397`), which is what reads the flags
(`kevin_hall/model.cpp:98,110`). Migration happens earlier still, in the demographic step
(`sim/engine.cpp` `update_population`). So immigrants are re-stratified before anything looks.

**Why I'd still change it:** the omission is safe only because of an ordering relationship that
nothing states and no test asserts. Reorder the two models, or add a reader of these fields that
runs earlier, and immigrants silently fall out of every stratum-specific calibration pass — which
would show up as a small drift in stratified output and nothing else. A hand-maintained field list
next to a growing struct is the mechanism; `Person` has gained fields before.

Cheapest fix is to make the omission impossible rather than to add the two lines: copy-construct
and then reset the identity/lifecycle fields, so a new member is carried by default and *not*
carrying it is the thing you have to write on purpose. That's the inverse of the current default,
and the current default is the dangerous one.

---

## 7. Emigration selects by slot order; immigration selects at random

**Kind:** question, possibly a modelling wart. **Confidence:** low on whether it matters — this one
genuinely needs a run.

In `apply_net_migration` (`src/sim/engine.cpp`) the two directions are not symmetric:

- **immigration** (line ~284) builds the candidate list of same-age, same-sex people and draws
  uniformly: `context.random().next_int(candidates.size())`
- **emigration** (line 290) takes "the first `-net` active people of this age and sex, **in slot
  order**" — no draw at all

Deterministic is good and clearly intentional. But slot order is not a neutral ordering: the
initial cohort occupies slots `1..N` by index (D9), and newborns and immigrants take **recycled**
slots from the free list (`ADR 0017`). So within an age-sex band, "lowest slot" correlates with how
the person entered the cohort, and emigration systematically removes the earliest-entering members
of each band.

Whether that biases anything depends on whether entry route correlates with risk factors within a
band, which I can't tell by reading — an immigrant is a clone of someone already present, so it
might wash out entirely.

**What I'd want to know, in order:**

1. Does the upstream baseline use the same rule? Every other migration decision in this file has a
   comment saying how it relates to the baseline (the immigration candidate list cites audit N-8);
   this one has none, which is either "same as upstream, not worth saying" or an unexamined gap.
2. If it *is* the same rule, it belongs in `docs/deviations.md` as a kept-deliberately entry the way
   B-21 is, rather than being invisible.
3. Is emigration by slot order distinguishable from emigration at random, in the outputs? That's a
   `seed_scan.py`-shaped question and the machinery to answer it already exists.

I'd put this last of the seven. It may well be nothing. But it's the only place I found where a
selection rule is silently non-uniform and nothing in the repo says whether that was a decision.

---

# Not a defect here — a question about the model

Different in kind from the seven above. Items 1-7 are things I could change in this repository.
This one is a property of the **model and the shipped configurations**, it is reproduced faithfully
here, and the only thing I would change on this side is observability. It is the thing I would
most want to ask Imperial about.

## 8. Every calibrated band mean is pinned to a constant for the whole horizon

**Kind:** question for upstream, plus a cheap observability gap here. **Confidence:** high on the
mechanism and on the config settings, both read directly. Not verified by running anything.

### The mechanism

`FactorsMean` is a CSV per sex — first column `age`, one column per risk factor, one row per age
(`src/io/csv_reader.cpp:326-352`). It has **no year axis**, and neither does the type it loads into
(`SexAgeFactorTable`, indexed by sex, factor, age).

It does two jobs. As the **anchor** for generation in StaticLinear, where the model produces a
multiplier on it rather than a value (`static_linear/factors.cpp:180-181`). And as the
**calibration target**, `adjust_risk_factors`, whose whole computation is:

```cpp
deltas[age] = expected - mean;            // risk_factor_model.cpp:305
```

— then added to every person in that (sex, age) band. An unweighted additive shift, so the band's
simulated mean lands on the table value **exactly**, by construction.

It runs **every year**, on all three surfaces:

| Model | Where |
|---|---|
| EBHLM | `hlm_model.cpp:176` at initialisation, `:203` every year after `update_exposure` |
| StaticLinear | `static_linear/model.cpp:193, 207, 244, 248` — whole population, income, per income quintile |
| KevinHall | `kevin_hall/model.cpp:361` newborns, `:408` every adult every year, **immediately after the energy balance ran** |

### The part worth asking about

The only time-variation available is a trend multiplier on read
(`risk_factor_model.cpp:125-146`) — `UpfTrend`, geometric and capped at `trend_steps`; or
`IncomeTrend`, decaying from the second year. Both are **one scalar per factor**: not per age, not
per sex.

And it is switched off in every example that runs:

| Example | `project_requirements.trend` | Runs |
|---|---|:-:|
| HLM_France | `{enabled: false, type: null}` | yes |
| HLM_India | `{enabled: false, type: null}` | yes |
| KevinHall_FINCH | `{enabled: false, type: null}` | yes |
| KevinHall_PIF | `{enabled: false, type: null}` | no |
| KevinHall_India | `{enabled: true, type: income_trend}` | no |

Independently of the config, the EBHLM path passes `apply_trend = false` at both of its
calibration sites, so on the HLM surface the target is static whatever the config says.

**So on `HLM_France`, which simulates 2010 to 2050, every calibrated factor's band mean is shifted
onto the same fixed number in all 41 years.** The population-wide mean still moves, because the age
structure moves — but that is demography, not risk factors evolving. On the Kevin Hall surface it
is sharper still: forty years of mechanistic energy balance, overwritten at the band level annually.

### What is *not* pinned, in fairness

- **The arm comparison.** Deltas are computed by the baseline and replayed by the intervention from
  the journal, so both futures get numerically identical shifts and the calibration cancels in
  `intervention - baseline`. That is the model's actual claim and it is unaffected.
- **Dispersion** (`std_*`), the **joint distribution** across a person's factors, and therefore
  **disease incidence, prevalence, mortality and DALYs**, which follow from individual values
  through relative-risk tables. Shifting a band's mean does not decide who is in its tail.

Calibrating to known margins is ordinary practice — post-stratification, raking. The objection is
not to doing it. It is that nothing distinguishes a pinned output from a modelled one.

### Evidence that the pinning is total, from this repo

`docs/SUMMARY.md` finding 5, from the seed-80 work: the weight calibration sets each band's mean to
the table, so one man's 22 kg deficit was **spread onto his 46 band-mates and the mean restored
exactly**. Only the dispersion moved. A person whose weight had gone catastrophically wrong left no
trace in the mean.

### Why it matters: the current setting is dominated

There are three coherent things to do with a calibration target, and the one in use is the weakest
of them. "Absolute values" below means magnitudes — 27.3 kg/m2 — as against the difference between
two arms. (Not to be confused with this codebase's other meaning of *level*, the HLM hierarchy
level in `risk_factors[].level`.)

| | What it does | What the reported means then are |
|---|---|---|
| **A** | calibrate to the base year **only**, then let the simulation run | a genuine **model output**, and testable against observed later years |
| **B** | calibrate to **year-indexed** observed means | **tracking reality** through the observed period |
| **current** | calibrate to the base year, **every year, for the whole horizon** | the base-year survey, restated at the end of the run |

A and B buy opposite things — A buys falsifiability, B buys realism — and both are defensible. The
current arrangement buys neither: the means are not a model output, because they are pinned, and
they are not reality, because they are pinned to a table that stopped moving. On `HLM_France` that
is a 2010-era mean reported as a 2050 figure, in all 41 years.

**And the format is not the obstacle.** The `FactorsMean` CSV already has `age` as its first
column; a `year` column, or a file per year, is a small change, and on this side it is one axis on
`SexAgeFactorTable` and one row selection in `expected_from`. Where multi-year survey data exists,
a single base-year table plus **one scalar per factor** discards nearly all of it — that scalar
cannot express a trend that differs by age or by sex, which is most of what such data says.

**The two counterarguments, and what they actually support.**

1. *The data runs out before the horizon does.* True — surveys covering, say, 2006-2020 against a
   run to 2050 would fix ten years and leave thirty unchanged. Still strictly better, because it
   puts a visible line between "calibrated to observation" and "extrapolated under an assumption".
   There is no such line today.
2. *Calibrating to observed means makes the model unfalsifiable over the observed period.* This one
   is genuinely strong: pin the means to observed 2015 values and 2015 can no longer serve as
   validation. But it is an argument for **A** — calibrate once and let go — and therefore an
   argument against calibrating every year for four decades. It does not defend the status quo.

### The questions

1. Which published outputs depend on calibrated **absolute values**, and which on differences and
   distributions? A benign answer exists — "the levels were never the claim" — and if that is the
   answer it is fine. It is not written down anywhere.
2. Is the trend being off in all three runnable examples **intentional**, or a switch nobody set?
   `KevinHall_India` is the only example that turns it on and it cannot run, which is at least
   consistent with nobody having exercised it.
3. Where do the `FactorsMean` numbers come from, and from which survey year? The files are not
   vendored here (`docs/examples.md` — configs only), so this repository cannot answer it.

### The one thing actionable on this side

**Record the calibration delta.** It is already computed at `risk_factor_model.cpp:305` and thrown
away. A band whose delta is large, or growing year on year, is a band where the model is drifting
and the calibration is covering for it — which is exactly the signal finding 5 says nobody had.
Emitting it as a metric alongside the existing `UncalibratedFactor.<name>` would cost almost
nothing and would turn "how much work is the calibration doing?" from an unanswerable question into
a column.

Worth pairing with something that lets a reader of a results CSV tell a pinned mean from a modelled
one. Today `mean_bmi` and `std_bmi` sit in adjacent columns, one is pinned by construction and the
other is not, and nothing says so.

---

## Checked and *not* worth writing up

Recording these so nobody re-derives them:

- **Newborns without a region or ethnicity** — handled; `demographic.cpp:267-271` assigns both
  before the risk-factor models run, with a comment saying why.
- **`ResultWriter` opening files after the run has done its work** — no; the writer is constructed
  before the engines (`session.cpp:414`), so a bad output path fails before any simulation.
- **`policy_start_year` double-counting with the `interventions` block** — they're genuinely
  independent and `static_linear/policies.cpp:1-6` says so explicitly. Both can be active at once
  by design.
