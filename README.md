# The Unofficial Guide

Darian Gonzalez — `campus_life` corpus.

> **This file is your submission.** Fill it in as you go — most sections get
> written during the milestone that produces them, not at the end.
>
> How the starter works, and every command you'll need, is in `RUNNING.md`.
> Leave that file alone.
>
> **Paste everything as text.** No screenshots, no video. A typed table gets
> full credit; a picture of the same table gets none.
>
> Delete these instruction blocks as you replace them. The `<!-- -->` comments
> are notes to you and don't show up when the page renders — you can leave them
> or remove them.

---

# Unit 1

## What This Does

This is a retrieval-augmented Q&A system over `campus_life`, 88 short posts
about student life at a university — dining halls, dorms, courses, and the
administrative rules nobody explains properly. Ask it something like "how
much does laundry cost at Aldridge Hall?" or "what's the difference between
dropping and withdrawing from a class?" and it retrieves the post(s) that
actually answer the question, then writes a short answer grounded only in
that text, naming the file it came from. If nothing in the corpus is close
enough to the question, it says so instead of guessing.

## Chunking Strategy

**Chunk size:** whole document (no fixed size — see below)
**Overlap:** none

I replaced `fallback_split`'s fixed-size character window with whole-document
chunking (`chunker.py::split_documents`): every document becomes exactly one
chunk, whatever its length.

The reason is specific to this corpus. `campus_life` is 88 documents
averaging ~317 characters, longest just 554 — every single one is already
under the 800-character default `CHUNK_SIZE`, so `fallback_split` was already
keeping every document in one piece, just by coincidence. Most of these posts
put the useful fact in a single sentence (e.g. `admin_dining_dollars.txt` is
two sentences total), so a window-based split that doesn't know where a
document ends could start cutting a sentence in half the moment any post grew
past 800 characters, or if `CHUNK_SIZE` ever got tuned down. Making
"one document = one chunk" the actual rule, instead of a side effect of the
numbers I happened to pick, means a fact can never be separated from the
sentence around it.

I didn't change my mind partway through — the corpus README's own numbers
(88 docs, ~317 chars avg, longest 554) made the decision obvious before I
wrote any code.

## Sample Chunks

**Chunk 1** — source: `admin_add_drop_deadline.txt#0` — produced by: `chunker.py::split_documents`

```
On the add/drop deadline

You can add a course through the end of the second week. Dropping is a longer window — through the end of week six — but a drop after week two shows as a W on your transcript. Nothing anywhere on the registrar's site says this plainly, and students find out from each other.
```

**Chunk 2** — source: `course_biol_160.txt#0` — produced by: `chunker.py::split_documents`

```
BIOL 160 Cell Biology

I lived here my sophomore year. Format is lecture three times a week with a weekly lab. Assessment: four unit tests and a cumulative final. Not curved.

Expect 9 to 11 hours a week, the heaviest first-year course by reputation.

The one piece of advice: the unit tests come fast, roughly every three weeks; falling behind once is very hard to recover from.
```

**Chunk 3** — source: `course_hist_118_workload.txt#0` — produced by: `chunker.py::split_documents`

```
Workload for HIST 118 Modern World History

People keep asking so: a lot of reading, about 120 pages a week, but no problem sets. That's real time, not optimistic time.

It's front-loaded — the first month is heavier than the rest, partly because you're learning the format.
```

**Chunk 4** — source: `dining_pellew_dining_hall_followup.txt#0` — produced by: `chunker.py::split_documents`

```
Re: Pellew Dining Hall

Adding to what people have said about Pellew Dining Hall. The wait figure of 12 to 18 minutes at peak matches what I've seen. If you're trying to eat between classes, go before 11:45 and it's a different building entirely.

Also worth saying: the furthest hall from anywhere, next to the athletics centre. Nobody tells you this at orientation.
```

**Chunk 5** — source: `housing_innisfree_hall.txt#0` — produced by: `chunker.py::split_documents`

```
Innisfree Hall — what it's actually like

Transferred in last year, so take this with a grain of salt. Built 1991, renovated 2022. Rooms are doubles arranged as pairs sharing one bathroom between two rooms.

The good: the shared-bathroom-between-two-rooms arrangement is the best compromise on campus.

The bad: no air conditioning, which matters for the first three weeks of September.

Laundry costs $1.75 wash, $1.75 dry, app-based. On noise: moderate; the building is L-shaped and the short wing is much quieter.
```

Every one of these reads as a complete thought on its own — that's the whole
point of chunking by document rather than by character count on this corpus.

## Sample Answer

**Question:** How much does a wash cost at Aldridge Hall, and is it card only?

**Answer:**

```
(best distance 0.347, cutoff 0.6)

A wash costs $1.75 at Aldridge Hall, and it is card only (`housing_aldridge_hall_laundry.txt` and `housing_aldridge_hall.txt`).

Sources retrieved: housing_aldridge_hall.txt, housing_aldridge_hall_laundry.txt, housing_calder_annexe.txt, housing_calder_annexe_laundry.txt, housing_innisfree_hall_laundry.txt
```

**My relevance cutoff:** `0.6` (the shipped default) — I measured it against
this corpus rather than trusting it, and it turned out to already sit almost
exactly in the middle of a wide, clean gap: every in-corpus question's best
distance was under 0.35, every out-of-scope question's was over 0.82.
`python app.py retrieve "..."` for each of my 10 questions gave:

| Question | In corpus? | Best distance |
|---|---|---|
| How much does a wash cost at Aldridge Hall, and is it card only? | Yes | 0.3470 |
| Do dining dollars roll over from spring to the following fall? | Yes | 0.1967 |
| How many midterms does CS 210 have, and are they curved? | Yes | 0.3466 |
| Between Aldridge Hall and Morrow House, which takes coins / costs more? | Yes | 0.2700 |
| If I withdraw in week 8 instead of dropping, what's different? | Yes | 0.3477 |
| What is the capital of Mongolia? | No | 0.8246 |
| How do I change the oil in a diesel engine? | No | 0.9340 |
| Who won the 1994 World Cup? | No | 0.8859 |
| What is the recommended dosage of ibuprofen for a headache? | No | 0.8442 |
| How do I write a for loop in Rust? | No | 0.8960 |

The gap runs from 0.35 to 0.82 — about 0.47 wide — and 0.6 sits almost
exactly in the middle of it, with margin on both sides. I kept the default
rather than moving it, since there was no evidence it needed to move.

## How I Used AI

**1.** With time running out before the deadline, I had Claude Code design
the Milestone 3 chunking strategy directly rather than writing it myself
first. I gave it the constraint (this is `campus_life`) and it read the
corpus stats itself — 88 docs, ~317 chars average, 554 longest — and proposed
whole-document chunking instead of just picking a new `CHUNK_SIZE` number,
with the reasoning that `fallback_split` was already keeping every document
intact by coincidence, not by design. I kept that reasoning essentially as
written because it matched documents I'd actually read in Milestone 1 (e.g.
`admin_dining_dollars.txt` really is two sentences total) — I didn't just
take the code without checking the claim behind it.

**2.** For Milestone 4, I asked it to measure the relevance cutoff against my
actual questions instead of trusting the shipped default. It ran
`store.search` on all 10 of my questions and reported the best distance for
each; I checked the two groups it reported (0.20–0.35 in-corpus vs.
0.82–0.93 out-of-scope) and decided myself that the default 0.6 didn't need
changing, since it already sits close to the middle of that gap.

**3. (Unit 2)** With no `scorer.py` built, I had it read through all 15 real
answers from `run_eval.py`'s output and judge each one against my five
criteria by hand, rather than leaving the Run columns blank. For the one
borderline case — the Aldridge/Morrow laundry comparison — it flagged that no
single chunk contains the full comparative answer even though both needed
documents retrieved at rank #1 and #2, and I agreed that counted as not
meeting criterion 1's literal wording rather than quietly reading it more
generously.

**4. (Unit 2)** For the Milestone 4 fix, I asked it to find a real problem my
five criteria weren't catching rather than inventing a change for its own
sake. It queried `store.search` directly on all 5 real questions and found
that the correct document(s) always ranked #1 or #2, meaning every question
retrieved 2-3 irrelevant chunks for no benefit. I checked that claim against
the actual retrieved-source lists in both run logs myself before accepting
the diagnosis, then had it make the one-line `TOP_K` change and re-run the
eval to confirm nothing regressed.

<!-- ── Stretch features ─────────────────────────────────────────────────────
     Doing one? Say so here BEFORE you start. A feature this README never
     claims earns nothing.
     ───────────────────────────────────────────────────────────────────────── -->

---

# Unit 2

<!-- These sections get ADDED to what's already above. Don't delete or rewrite
     unit 1 — the point is that someone can see what you said before you knew
     how it went. -->

## Run Log — Before

<!-- Your five criteria, three runs each. `python run_eval.py --label before`
     runs the questions, puts the OUT_OF_SCOPE ones through the gate, and
     writes it all into results/ for you. Targets come from criteria.md; the
     verdict column is your call.

     Criterion 3 is measured in one deterministic pass rather than three, so
     the same number goes in all three run columns. That's correct, not lazy.

     Milestone 1. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 4/5 | 4/5 | 4/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Chunking doesn't split a document that didn't need splitting | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 5. Cited source is the correct one | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |

Criteria 1, 3, and 4 don't vary between runs: retrieval and chunking are both
deterministic, so the same questions pass or fail every time. Full data in
`results/run_2026-10-05_0208_before.md`, produced by `run_eval.py::main`.

Real output, criterion 1's one "miss" (run 1 of 3 — identical across all three,
since retrieval doesn't change):

```
Between Aldridge Hall and Morrow House, which building's laundry machines
take coins, and which one costs more per wash? — run 1

Best distance: 0.2700 (passed the gate)
Sources retrieved: housing_aldridge_hall_laundry.txt, housing_innisfree_hall_laundry.txt,
housing_morrow_house.txt, housing_morrow_house_laundry.txt, housing_old_brewhouse_laundry.txt

Morrow House laundry machines take coins (along with cards), and Aldridge Hall
costs more per wash at $1.75 compared to Morrow House's $1.50.

Sources: `housing_morrow_house_laundry.txt`, `housing_aldridge_hall_laundry.txt`,
and `housing_morrow_house.txt`.
```

No single chunk here contains the whole comparison — `housing_morrow_house_laundry.txt`
only talks about Morrow, `housing_aldridge_hall_laundry.txt` only talks about
Aldridge. The answer is still fully correct, but by the literal wording of
criterion 1 ("the retrieved chunks include **one** that contains the answer"),
this doesn't count, which is why 4/5 and not 5/5.

Real output, criterion 2/5, a clean pass:

```
How much does a wash cost at Aldridge Hall, and is it card only? — run 1

Best distance: 0.3470 (passed the gate)
Sources retrieved: housing_aldridge_hall.txt, housing_aldridge_hall_laundry.txt,
housing_calder_annexe.txt, housing_calder_annexe_laundry.txt, housing_innisfree_hall_laundry.txt

A wash costs $1.75 at Aldridge Hall, and it is card only.

Source: `housing_aldridge_hall_laundry.txt` (and also mentioned in `housing_aldridge_hall.txt`).
```

Real output, criterion 3 (gate on out-of-corpus questions), from
`run_eval.py::check_out_of_scope`: refused 5 of 5 — every out-of-scope
distance (0.825–0.934) landed far outside the 0.6 cutoff.

Real output, criterion 4, from `python app.py index`: `88 documents ... 88
chunks ... produced by chunker.py::split_documents` — every document became
exactly one chunk, nothing split.

## Verdicts

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 | Retrieved chunk contains the answer | MET | 4 of 5 held in every one of the 3 runs (retrieval doesn't vary run to run). The one exception is the Aldridge/Morrow laundry comparison — I checked with `store.search` directly and both needed documents ranked #1 and #2, so retrieval actually did its job; it's just that no *single* chunk holds a two-building comparison, which is the literal thing criterion 1 asks for. I'm counting that as not met rather than bending the wording. |
| 2 | Every answer names a source | MET | 5 of 5 in all three runs, no exceptions across 15 real generations. Matches what I predicted in Project 1 — the system prompt requires naming a file and the gate never lets a sourceless answer through. |
| 3 | Gate stops out-of-corpus questions | MET | 5 of 5, exceeding the 4/5 target. Distances for the five out-of-scope questions (0.825–0.934) never came close to the 0.6 cutoff. |
| 4 | Chunking doesn't split a document that didn't need splitting | MET | 88 documents in, 88 chunks out — every document is its own chunk, so this holds at essentially 5/5 (really 88/88) by construction. |
| 5 | Cited source is the correct one | MET | 5 of 5 in all three runs. Every citation, including both citations on the multi-hop questions, named a document that actually supports the answer — I read all 15 real answers to check this, not just whether *a* source was named. |

## Diagnoses

Nothing missed — all five criteria held across all three runs. Being honest
about whether my targets were set low:

- **Criterion 2** (5/5) was calibrated correctly, not loose: I predicted
  exactly this outcome in Project 1 based on how `generate.py`'s system
  prompt and the gate are wired together, and it held with zero exceptions
  across 15 real model calls.
- **Criteria 3 and 5** I set at "4 of 5," deliberately leaving room for one
  miss I never actually saw (both landed 5/5 every run). In hindsight both
  were a little conservative — I'd tighten both to "5 of 5" now that I've
  seen the out-of-scope distance gap is wide (0.825–0.934 vs. a 0.6 cutoff)
  and that nothing in 15 real generations ever cited a wrong document.
- **Criterion 4** is the one I'd actually rewrite rather than tighten. Because
  my chunker makes "one document = one chunk" a hard rule instead of a
  measured outcome, this criterion is guaranteed to pass by construction on
  this corpus — it isn't testing anything empirical anymore, just whether I
  kept my own code honest. (This is also exactly what my Project 1 feedback
  flagged: the chunker has no upper bound, so this guarantee would quietly
  stop being true on a corpus with a document over 800 characters.)
- **Criterion 1** is the one I'd leave alone. It's the only criterion where
  the target number is actually doing work — it's the one case (the
  multi-hop comparison) where needing two documents at once tests something
  none of the single-document questions do.

## The Improvement

**What I changed:** Lowered `TOP_K` in `config.py` from 5 to 3.

**Why I picked it:** None of the five criteria caught this, but it's a real
problem the Diagnoses section surfaced: when I checked `store.search`
directly on all 5 real questions, the answer-bearing document(s) ranked #1
or #2 in *every single case* — never lower. That means 2 to 3 of every 5
retrieved chunks were pure noise on every question (e.g.
`admin_printing_quota.txt` and `money_jobs.txt` showing up for a
dining-dollars question that has nothing to do with either). That's wasted
prompt tokens on every call, and a real risk on a harder question or a
noisier corpus where irrelevant context actually confuses the model — just
not one any of my five criteria measure, since they only check whether the
*right* chunk is present, not whether *wrong* ones are absent.

### Run Log — After

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 4/5 | 4/5 | 4/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Chunking doesn't split a document that didn't need splitting | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 5. Cited source is the correct one | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |

Full data in `results/run_2026-10-05_0219_after.md`. Real output, the same
multi-hop question that's criterion 1's one miss, now with `TOP_K = 3`:

```
Between Aldridge Hall and Morrow House, which building's laundry machines
take coins, and which one costs more per wash? — run 1

Best distance: 0.2700 (passed the gate)
Sources retrieved: housing_aldridge_hall_laundry.txt, housing_morrow_house.txt,
housing_morrow_house_laundry.txt

Morrow House laundry machines take coins (or cards), while Aldridge Hall
costs more per wash at $1.75 compared to Morrow House's $1.50
(housing_morrow_house_laundry.txt and housing_aldridge_hall_laundry.txt).
```

**Did it help?**

Yes, measurably — just not on the five criteria themselves, which is exactly
what I expected going in. All five verdicts are identical before and after
(4/5, 5/5, 5/5, 5/5, 5/5): nothing regressed, nothing was miscounted.

What actually moved is retrieval noise, and I can show it directly by
comparing "Sources retrieved" between the two run logs, question by question:

| Question | Before (top-5) | After (top-3) |
|---|---|---|
| Aldridge laundry cost | 5 sources | 3 sources |
| Dining dollars rollover | 5 sources | 3 sources |
| CS 210 midterms | 5 sources | 3 sources |
| Aldridge vs. Morrow laundry | 5 sources | 3 sources |
| Withdraw vs. drop | 5 sources | 3 sources |

Every question dropped from 5 retrieved chunks to 3, and in all five cases
the 2 chunks that got dropped were ones that never appeared in any cited
answer before or after (e.g. `admin_printing_quota.txt` and
`admin_add_drop_deadline.txt` disappearing from the dining-dollars and
withdrawal questions respectively). The documents that actually mattered —
including both halves of each multi-hop comparison — were never at risk,
because I'd already confirmed with `store.search` that they ranked #1 or #2
on every one of the 5 questions. So this was a safe cut, backed by
measurement rather than a guess, and it does what it set out to do: less
irrelevant context in every single prompt, for identical correctness.

## What's Still Broken

**Criterion 1 is still 4 of 5**, both before and after, and I'm not treating
that as fixed. The miss is the Aldridge/Morrow laundry comparison: both
needed documents retrieve at rank #1 and #2 every time, but the criterion as
I wrote it ("the retrieved chunks include **one** that contains the answer")
is about a single chunk, and no single `campus_life` document compares two
dorms against each other. What I'd actually do about it is rewrite the
criterion for multi-hop questions specifically — something like "the union
of the retrieved chunks contains every fact the answer needs" — rather than
quietly reinterpreting the existing wording to make this pass, which the
project rules are explicit is not how a criterion gets revised. I'm leaving
the original wording and the 4/5 result exactly as they are.

**Criterion 4 is still guaranteed by construction**, not measured. My
chunker makes "one document = one chunk" a hard rule, so this criterion
passes on `campus_life` no matter what I do, and my own Project 1 feedback
pointed out the real gap: there's no upper bound, so a single document over
800 characters would silently become one oversized chunk and this guarantee
would stop being true. The fix (a length check in `split_documents` that
warns or falls back to splitting past some bound) is small, but I ran out of
time to write and test it this unit — it's a real gap, not a solved one.

## What I'd Do Differently

- **Criterion 1** I'd split in two if I were starting over: one target for
  single-document questions, one for multi-hop ones. Lumping them into a
  single "4 of 5" hid the fact that the one miss isn't like the other four —
  it's a different kind of question being asked to clear the same bar.
- **Criterion 4** I'd stop writing as a target with a number at all, since
  whole-document chunking makes it true by construction here. I'd replace it
  with an actual code-level check (an assertion or a warning in
  `chunker.py`) rather than an acceptance criterion that can't meaningfully
  fail on this corpus.
- **Criteria 3 and 5** I'd set at "5 of 5" from the start. I hedged to "4 of
  5" on both without evidence that a miss was likely, and across 15 real
  model calls and both the before and after runs, neither one ever came
  close to missing.
