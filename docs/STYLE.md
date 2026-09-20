# How the documents in this repository are written

This file is the writing rule for every document here. It exists because the project's credibility
rests on somebody outside it being able to read these documents and check them. A reader who has to
decode a sentence before they can check it will usually stop instead.

The intended reader is a policy economist or a software engineer who has never seen this repository
and has one pass to spend on it. They know their own field. They do not know this project's terms,
its finding IDs, or which of its documents came first.

The rules are numbered so that a review comment can name one.

---

## The rules

**1. One idea per sentence.** If a sentence carries two claims, split it into two sentences. A
reader who disagrees with half of a sentence has nowhere to put the disagreement.

**2. The thing first, the qualification second.** Write "The energy balance admits a negative body
fat mass, which is undefined in its own partition coefficient", not "Undefined in its own partition
coefficient, and admitted anyway, is a negative body fat mass". The reader should know what the
sentence is about by its fifth word.

**3. Never open a sentence with a pronoun standing for a concept.** "It", "this", "that" and "they"
at the start of a sentence must refer to a person or a named object in the previous clause, not to
the previous idea. Name the thing again instead. Repetition is cheaper than ambiguity.

**4. Define every project term on first use on that page, or link the glossary.** Terms such as
*surface*, *family*, *deviation*, *stored reference*, *lattice series* and *compatibility flag* mean
something specific here. Give the reader a definition in the sentence, or link
[docs/glossary.md](glossary.md). A page is the unit, not a document: a reader arriving at a heading
from a link has not read the paragraphs above it.

**5. An aphorism may cap an explanation, never replace one.** A memorable line after a full
explanation is useful. The same line in place of the explanation leaves the reader with a mood and
no fact. At most one aphorism per document.

**6. No em-dash chains.** At most one em-dash pair in a sentence. Two or more turn the sentence into
a list of asides that the reader has to reassemble. A second aside belongs in a second sentence.

**7. No sentence that survives its own deletion without loss.** Delete the sentence and read the
paragraph again. If nothing is missing, the sentence was decoration. Three examples of the
pattern, all of them written in this repository: "What moves these numbers is runner weather", "a
fixed point a summary of its own tree cannot reach", and "the shape of thing somebody puts in a
paper".

**8. No "found by running rather than reading" framing.** Say what was done and what it showed. How
the finding was arrived at is a fact about the method, and it belongs in a sentence that states the
method, not in a flourish attached to the finding.

**9. Numbers, identifiers and tables stay.** Every measured number, every `B-nn` finding ID, every
`file.cpp:123` reference and every table in these documents is evidence. Plain writing means plain
sentences around the evidence. It never means less evidence. If a passage is too long, cut the
explanation of the evidence and link the document that holds it, never the number itself.

**10. British spelling.** *Behaviour*, *initialise*, *modelling*, *analyse*, *licence* as a noun.
Code, file names and quoted output keep whatever spelling they have.

---

## Three examples, taken from these documents

### 1. The opening of `docs/briefing.md`

**Before.**

> A second implementation of Health-GPS, in C++20, compared against yours run for run. Not a fork
> and not a proposed replacement: it exists to find out what the model does when two independent
> implementations are held against each other.

Neither sentence has a subject and a verb in the ordinary place, and "held against each other" asks
the reader to guess what was compared. Rules 1 and 2.

**After.**

> This repository is a second implementation of Health-GPS, written in C++20. It is not a fork of
> your code and not a proposed replacement for it. Its purpose is to run both implementations on
> the same configurations and the same seeds, and to report where their results differ.

### 2. A finding in `docs/briefing.md`

**Before.**

> It is the last place that knows whose a value is, and it turns any upstream defect into a wrong
> mean.

The sentence opens on a pronoun standing for a paragraph, and "knows whose a value is" is not a
sentence. Rules 1 and 3.

**After.**

> `AnalysisModule::calculate_historical_statistics` is the last code that sees a value together with
> the person it belongs to. A substitution made there turns a defect anywhere upstream of it into a
> mean that is wrong and looks ordinary.

### 3. A claim about the test suite in `README.md`

**Before.**

> A further 94 test the equivalence harness's own statistics, because a mistake there says PASS
> rather than producing a wrong number.

The reason clause is compressed to the point of being a puzzle. Rule 1.

**After.**

> A further 94 tests cover the equivalence harness's own statistics. The harness is what decides
> whether the two implementations agree, so a mistake in it does not produce a visibly wrong
> number. It reports agreement that is not there.

---

## What this file does not govern

The internal record — [docs/SUMMARY.md](SUMMARY.md), [docs/equivalence.md](equivalence.md),
[docs/equivalence-method.md](equivalence-method.md), [docs/performance.md](performance.md),
[docs/deviations.md](deviations.md) and the ADRs under [docs/decisions/](decisions/) — was written
for readers who already have the context, and it was not rewritten against these rules. New text
added to those files should still follow them.

Source comments are not covered here. A comment is read next to the code it describes, which is a
different kind of context from a document read alone.
