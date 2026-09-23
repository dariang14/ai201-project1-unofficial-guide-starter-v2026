# Acceptance criteria — The Unofficial Guide

Five criteria that say what "working" means for this system, written in unit 1
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"Retrieval works"* is an opinion. *"For at
least 4 of my 5 test questions, the top results include a chunk containing the
answer"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter or looser one. A reason that says something about your corpus or your
pipeline earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## 1. Retrieved chunks contain the answer

For at least 4 of my 5 test questions, the retrieved chunks include one that
contains the answer.

**Why this target:**
Three of my five questions (laundry cost, dining dollars rollover, CS 210
midterms) each have their answer sitting in one short document, so retrieval
should be easy for those. The other two are multi-hop: they each need the
system to pull back chunks from *two different* documents at once (Aldridge
+ Morrow laundry pages; add/drop + withdrawal deadlines), and nothing in the
corpus explicitly compares the two, so I expect at least one of those two to
strain retrieval. 4 of 5 leaves room for exactly one multi-hop miss without
meaning retrieval is broken.

---

## 2. Every answer names a source

Every answer the system produces names at least one source document.

**Why this target:**
`generate.py`'s `GROUNDING_INSTRUCTION` explicitly tells the model to name
the filename it used, and the relevance gate in `gate.py` refuses a question
outright rather than letting the model answer from nothing. So every
non-refusal answer has already been tied to at least one retrieved document
before generation even runs — there's no code path where the model answers
without being handed a source. This should hold at 5 of 5 as long as the
model follows the system instruction, which is the thing worth checking.

---

## 3. The relevance gate stops out-of-corpus questions

When I ask a question my documents clearly don't cover, the relevance gate
stops it and the system returns "I don't have enough information about that" —
in at least 4 of 5 tries.

<!-- The five questions are the ones in `OUT_OF_SCOPE` at the bottom of
     `questions.py`, and `run_eval.py` puts them through the gate and writes
     what happened into your run log. Swap them for your own if you'd rather —
     just keep five of them, or the "4 of 5" above has nothing to be 4 of. -->

**Why this target:**
My `OUT_OF_SCOPE` questions (world capitals, diesel engines, a 1994 World Cup
result, ibuprofen dosage, a Rust for-loop) are all completely unrelated to a
university campus, so I expect their embedding distances against 88 short
posts about dorms and dining halls to be clearly bad matches. I'm holding
back one from 5/5 because I haven't measured actual distances yet — that's
Milestone 4 — and it's possible a word like "course" in the Rust question
overlaps enough with corpus vocabulary to pull a deceptively close chunk.

---

## 4. Chunking doesn't split a document that didn't need splitting

At least 4 of 5 sampled chunks are exactly one whole source document, with no
sentence cut across a chunk boundary.

**Why this target:**
The longest document in `campus_life` is 554 characters
(`housing_old_brewhouse.txt`), and the average is about 317 — both well
under the default 800-character `CHUNK_SIZE`. At the default settings,
chunking should be close to a no-op: almost every document should become a
single chunk on its own. If I sample 5 chunks and see even one cut
mid-sentence, that tells me the chunker is splitting on raw character count
rather than respecting document boundaries, since nothing in this corpus is
actually long enough to need it.

---

## 5. The cited source is the correct one, not just any source

For at least 4 of my 5 test questions, the document(s) named in the answer
are among the document(s) that actually contain the answer — not just some
retrieved document, and not just any file the model felt like naming.

**Why this target:**
Criterion 2 only checks that an answer names *a* source; it says nothing
about whether that source is the right one. My two multi-hop questions (the
Aldridge/Morrow laundry comparison and the drop/withdrawal comparison) each
need two documents combined, so it's easy for the system to retrieve the
right chunks but have the model only cite one of them, or to retrieve a
plausible-looking but wrong document (a different dorm's laundry page, say)
and cite that instead. This matters more for this corpus than raw retrieval
hit-rate, because someone would actually act on a wrong dorm's laundry price
or a wrong course's exam policy.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 2 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 1. Retrieved chunks contain the answer

         For at least 4 of my 5 test questions, the retrieved chunks include
         one that contains the answer.

         **Why this target:** ...

         > **Revised in unit 2:** For at least 4 of 5 questions, the top three
         > results contain the answer.
         >
         > **Why revised:** I couldn't judge "the chunks include one that
         > contains the answer" the same way twice — I scored two questions
         > differently on Monday than on Wednesday. The new version is
         > something I can actually check.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said 4 of 5 but got 2 of 5, so 2 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.

     The whole reason the originals stay visible is so someone can see what you
     said before you knew the answer.
     ───────────────────────────────────────────────────────────────────────── -->
