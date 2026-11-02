# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

Provenance Guard is a small service a writing site can call when someone submits their own writing. The site sends the text. The service sends back a guess, a score, a sentence a reader can understand, and an id the writer can use to push back.

A submission goes through seven steps.

1. The text arrives at `POST /submit` as JSON with `text` and `creator_id`. If `text` is missing, empty, or only whitespace, the route stops. It returns HTTP 400 and writes a rejection to the audit log. There is no guess and no label, because there is nothing to judge. The same refusal happens when `text` is longer than 8000 characters. The model only reads about 400 tokens, so scoring a pasted novel would judge a short slice and present that slice as a judgment of the whole piece.

2. The route makes a `content_id`. The appeal needs that id. Without it the writer has nothing to point at.

3. Signal one runs. `detector.model_signal` asks a local GPT-2 model how predictable the text is. Text the model finds easy to predict scores higher. That number is `model_score`, from 0 to 1. Higher means more likely AI.

4. Signal two runs. `stylometry.style_signal` does not read the text. It measures the shape: how much sentence length varies, unique words divided by total words, and how much punctuation there is. Those become `style_score`, also from 0 to 1, higher meaning more likely AI.

5. `scoring.combine_signals` averages the two scores. If the lower score is under 0.60, the average is capped at 0.64, so one dissenting signal cannot produce a high-confidence AI label.

6. `scoring.score_to_label` turns the confidence into a short guess, `human`, `unsure`, or `ai`, and a full sentence. The reader sees the sentence. At or below 0.35 the label is high-confidence human. At or above 0.65 it is high-confidence AI. Between those cuts the label is unsure. The sentences themselves are written in Milestone 5.

7. `audit.log_decision` appends one line to `logs/audit.jsonl`. The line has the time, the content id, the creator id, the guess, both signal scores, the combined score, the label, and status `decided`. The HTTP response carries `content_id`, `guess`, `confidence`, `label`, `model_score`, and `style_score`. A writer who thinks the label is wrong sends `POST /appeal` with that `content_id` and their reasoning. The original line stays. A new line sets the status to `under_review`. The service does not score the text again.

### A false positive

Say a student submits this, and they wrote every line.

> I went to the library after class. I sat down and wrote my poem. It is about my mother and the kitchen at home. I wrote every line myself.

The sentences are short and the words are common. Signal one measures predictability, so this kind of plain writing can score like machine writing. Signal one returns 0.6661 on this poem. Signal two returns 0.5058, because the text is too short for word variety to count and there is no comma or dash. The average is 0.5859. The lower signal is under 0.60, so the score stays in the unsure band. Signal one alone is already over 0.65. If the label followed only the model, the reader would see an AI label on a poem the person wrote.

The writer can take the `content_id` from the response and `POST /appeal` with a reason. The log keeps the original decision and adds a line with status `under_review`. Nothing is re-scored. A person has to read the appeal.

A harder version of the same mistake is eight short human sentences, "I walked to the store. I bought some bread," and so on. Signal one scores that 0.8863. Signal two scores it 0.5673. A plain average is about 0.73, which would be the AI label. The agreement rule caps it at 0.64, so the label stays unsure. That cap is the decision. Some edited AI text will also stay unsure, and that is the cost.

### Endpoint contracts

`POST /submit`

Request fields:

| Field | Type | Meaning |
|---|---|---|
| `text` | string | The writing being judged |
| `creator_id` | string | Who submitted it |

Success is HTTP 200:

| Field | Type | Meaning |
|---|---|---|
| `content_id` | string | Id for a later appeal |
| `guess` | string | `human`, `unsure`, or `ai` |
| `confidence` | number | 0 to 1. Higher means more likely AI |
| `label` | string | The full sentence a reader sees |
| `model_score` | number | Signal one, 0 to 1 |
| `style_score` | number | Signal two, 0 to 1 |

Failure is HTTP 400 when `text` is missing, empty, or longer than 8000 characters. The body is `error` set to `bad_input` and a short `message`. There is no `content_id`. The audit log gets a rejection line, not a decision.

`POST /appeal`

Request fields:

| Field | Type | Meaning |
|---|---|---|
| `content_id` | string | From an earlier submit response |
| `reasoning` | string | The writer's own words |

Success is HTTP 200, and only if that `content_id` is already in the log:

| Field | Type | Meaning |
|---|---|---|
| `content_id` | string | The same id |
| `status` | string | `under_review` |
| `message` | string | A short confirmation that a person will look, and that the text was not scored again |

The appeal does not send `creator_id`. The log copies `creator_id` from the original decision. The new log line has the time, `content_id`, `creator_id`, `event` set to `appeal`, `status` set to `under_review`, and `reasoning`. The original decision line is not overwritten.

Missing `content_id` or empty `reasoning` is HTTP 400 with `error` set to `bad_input`. A `content_id` that is not in the log is HTTP 404 with `error` set to `not_found`, and no new log line.

`POST /ping` and `GET /log` already work. I am not changing them.

---

## Signals and Scoring

### Signal one — the model signal

**What it measures:** How predictable the text is to a local GPT-2 model. `detector.model_signal` turns perplexity into a score from 0 to 1. Low perplexity means the model was not surprised, and the score goes up. Higher means more likely AI.

**Why that might differ between human and AI writing:** Text a model wrote is often text a model finds easy to continue. Common words in ordinary sentences are easy to predict.

**What it can't see:** Predictability is not authorship. A person writing in a second language, a younger writer, or anyone who stays with words they are sure of will look predictable too. Dense writing with rare words often goes the other way and looks human, because the model fails to predict those words. The people this signal is most likely to accuse are the ones with the least room to argue.

### Signal two — the style signal

**What it measures:** The shape of the text, not what it says. Three measures in `stylometry.py`, averaged into `style_signal`. Higher means more likely AI. `sentence_length_spread` is the standard deviation of sentence lengths divided by the mean. Low spread points toward AI, so the score flips it. One sentence has no spread and counts as the even case. `type_token_ratio` is unique words divided by total words. Under 40 words this measure is ignored and contributes 0.5, because a short text looks varied just by being short. On longer text, a ratio near 0.45 points toward AI and a ratio near 0.85 points toward a person. `punctuation_density` counts commas, semicolons, colons, and dashes per word, not full stops. About 0.08 marks per word points toward AI. Bare text and heavily punctuated text point toward a person.

**Why that might differ between human and AI writing:** Generated prose tends toward even, mid-length sentences. A person is lumpier, a short sentence next to a long one. On the four calibration pieces that showed up. Clearly AI text scored style 0.6170. A lumpy human note scored 0.2712. Formal human writing scored 0.4223. Lightly edited AI scored 0.5154.

**What it can't see:** It cannot read. Typos, broken sentences, and a pasted quotation change the shape and move the score without changing who wrote it. That is cheap to do on purpose.

### The combining rule

**The rule:** Average the two scores, equal weights. Both already point the same way. Higher means more likely AI. If the lower score is under 0.60, cap the average at 0.64. One signal that is not actually on the AI side of 0.60 cannot drag the result into the high-confidence AI band. The label then stays unsure, or human if the average is low enough.

**The numbers:** `WEIGHT_MODEL_SIGNAL` 0.5, `WEIGHT_STYLE_SIGNAL` 0.5, `AGREEMENT_FLOOR` 0.60, cap 0.64. Label cuts stay 0.35 and 0.65. All of those live in `config.py`.

**Where it lives:** `scoring.py::combine_signals`

**A case where my two signals split, and what my rule does with it:** Eight short sentences a person could write. "I walked to the store. I bought some bread. I walked back home. I made some toast. I sat at the table. I ate the toast. I washed the plate. I went to my room." Signal one scores 0.8863. Signal two scores 0.5673. The plain average is about 0.73, over the AI cut. The style score is under 0.60, so `combine_signals` returns 0.6400 and the label stays unsure.

The same rule on typos does less than I wanted. Adding misspellings to the AI paragraph drops signal one from 0.8907 to 0.2406, because the model is surprised by the odd words, while signal two stays 0.6170, because the sentence lengths did not change. The average is 0.4288, which is unsure. Typos fool the model. The style signal cannot read, so it does not notice them either. Neither signal gets to call that text AI on its own, and neither calls it human.

| Piece | model_score | style_score | combined |
|---|---|---|---|
| Clearly AI | 0.8907 | 0.6170 | 0.7539 |
| Clearly human | 0.3969 | 0.2712 | 0.3340 |
| Formal human | 0.3788 | 0.4223 | 0.4006 |
| Lightly edited AI | 0.5274 | 0.5154 | 0.5214 |
| Plain poem | 0.6661 | 0.5058 | 0.5859 |
| Even human sentences | 0.8863 | 0.5673 | 0.6400 |



---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human |  |  |
| unsure |  |  |
| high-confidence AI |  |  |

**Why I worded the "unsure" one this way:**
<!-- It's the hardest of the three. It has to admit uncertainty without
     sounding like an accusation, because the person reading it may well have
     written every word themselves. -->



---

## Sample Run

<!-- One submission and one appeal, pasted as text: the request, the response,
     and the log entries. -->

**A submission**

Milestone 3, one signal only. `confidence`, `guess`, `label`, and `style_score` are still placeholders. The model score is real. The same poem scored 0.6661, which is above the placeholder AI cut of 0.65, so plain writing can already look like a machine to signal one.

```bash
$ curl -X POST http://127.0.0.1:5000/submit \
  -H "Content-Type: application/json" \
  -d '{"text": "I went to the library after class. I sat down and wrote my poem. It is about my mother and the kitchen at home. I wrote every line myself.", "creator_id": "ada"}'
```

```json
{
  "confidence": null,
  "content_id": "ff6c79cd6f8f4d269434af9f76ce1a9a",
  "guess": "unset",
  "label": "Label not written yet.",
  "model_score": 0.6661,
  "style_score": null
}
```

**An appeal**

```bash
$ curl -X POST http://127.0.0.1:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "...", "reasoning": "..."}'
```

```json

```

**What the log shows afterwards**

```bash
$ curl "http://127.0.0.1:5000/log?limit=3"
```

```json

```

---

## How I Used AI

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** ___ per minute, ___ per day

**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

```

```

---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
