# Acceptance criteria — Provenance Guard

Five criteria that say what "working" means for this service, written in unit 7
**before** you built it.

**All five are yours.** None are given.

An acceptance criterion names a number or something plainly observable. *"The
detector is accurate"* is an opinion — and perfect AI detection isn't possible
anyway, so it's an opinion about something nobody can deliver. *"Of 10 pieces I
wrote myself, at most 1 is labelled high-confidence AI"* is a criterion.

Under each, write a sentence or two on **why that target**. *"I allow at most 1
false positive in 10 because on a writing site a wrong accusation costs more
than a missed detection"* is a real answer.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## Pick numbers you can defend

**Cover at least three of these five areas.** They're prompts, not a form.

| Area | A question it could answer |
|---|---|
| False positives | How often may a human's writing get called AI? |
| Score spread | How far apart should clearly-AI and clearly-human text score? |
| Label coverage | Is every one of your three labels actually reachable? |
| The appeal path | Does an appeal always change status and always get recorded? |
| Bad input | What happens when `text` is missing, empty, or enormous? |

Two things worth knowing before you pick numbers:

- **The expensive mistake is one-sided.** Missing some AI text costs the site a
  little. Accusing a real writer costs that person a lot. A criterion that
  treats both errors as equally bad is describing a system nobody would ship.
- **Unit 8 tests across three trials, and the target has to hold across all
  three.** Your first signal runs a model, so identical input can score
  differently twice. A target of "at most 1 in 10" against trials of 2, 1, 2 is
  a **miss**.

---
## 1. False positives on ordinary human writing

Of 10 pieces of ordinary human writing I submit, at most 1 comes back with guess `ai`, in each of three trials.

**Why this target:** On a writing site a wrong accusation costs the writer more than a missed AI submission costs the site. One in ten is the most I would accept if the writing were mine. Zero would pretend the model does not flag plain sentences. Three in ten is a system I would not want on my own work. Unit 8 runs three trials, and the model score can move, so the target has to hold in all three.

---

## 2. Score spread

On 5 pairs of clearly AI text and clearly human text, the combined score of the AI piece is at least 0.20 higher than the human piece in at least 4 of the 5 pairs, in each of three trials.

**Why this target:** If those two kinds of text land on the same number, the three labels are names for one score. A gap of 0.20 is visible in the JSON. A gap of 0.05 would pass even when the signals barely move, and that target would be too easy to miss failing.

---

## 3. Label coverage

In one set of 6 submissions I choose, each of the three guesses `human`, `unsure`, and `ai` appears at least once.

**Why this target:** A label no score can reach is a sentence in the README that the service never shows a reader. Six submissions is enough to aim two at each band. If one guess is missing, the ranges have a gap.

---

## 4. The appeal path

Of 5 appeals sent with a `content_id` that already exists in the log, all 5 return status `under_review`, and all 5 append a log line whose `reasoning` is the text that was sent. One appeal sent with a `content_id` that is not in the log returns HTTP 404 and appends no log line.

**Why this target:** The appeal is the only thing a writer can do after a wrong label. If it fails quietly, the false positive in the trace has no way out. Five is enough to see that it is not a one-off. The 404 case is there because a typo in the id looks like a broken endpoint when the log simply has no such row.

---

## 5. Bad input

Three requests, one with `text` missing, one with `text` empty or only whitespace, and one with `text` of 8001 characters, each return HTTP 400. None of the three appends a log entry with status `decided`.

**Why this target:** The trace refuses those three cases. The model only reads about 400 tokens, so scoring a long paste would judge a slice and present it as a judgment of the whole piece. A 500, or a `decided` row on empty text, means the route crashed or invented a judgment. All three have to fail the same way, or the contract is not the one I wrote.



---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 8 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath, like this:

         ## 2. Uncertain cases land in the unsure band

         The system handles uncertain cases well.

         **Why this target:** ...

         > **Revised in unit 8:** Text I judged borderline scores between
         > 0.4 and 0.6 in at least 4 of 5 cases.
         >
         > **Why revised:** "well" gave me nothing to check. I couldn't
         > reach a verdict from it at all.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "At most 1 of 10 of my own pieces flagged as AI" → "at most 3
            of 10", because 1 of 10 turned out to be hard.

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
