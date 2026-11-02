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

4. Signal two runs. `stylometry.style_signal` does not read the text. It measures the shape: how much sentence length varies, unique words divided by total words, and how much punctuation there is. Those become `style_score`, also from 0 to 1, higher meaning more likely AI. This function is still a stub. Milestone 4 builds it.

5. `scoring.combine_signals` turns the two scores into one confidence from 0 to 1. The rule is not chosen yet. I will write it down before coding it. A case where the two signals disagree is what that rule has to handle.

6. `scoring.score_to_label` turns the confidence into a short guess, `human`, `unsure`, or `ai`, and a full sentence. The reader sees the sentence. The starter file currently cuts at 0.35 and 0.65. At or below 0.35 the label is high-confidence human. At or above 0.65 it is high-confidence AI. Between those cuts the label is unsure. Those cuts are placeholders until I decide where a wrong accusation should start.

7. `audit.log_decision` appends one line to `logs/audit.jsonl`. The line has the time, the content id, the creator id, the guess, both signal scores, the combined score, the label, and status `decided`. The HTTP response carries `content_id`, `guess`, `confidence`, `label`, `model_score`, and `style_score`. A writer who thinks the label is wrong sends `POST /appeal` with that `content_id` and their reasoning. The original line stays. A new line sets the status to `under_review`. The service does not score the text again.

### A false positive

Say a student submits this, and they wrote every line.

> I went to the library after class. I sat down and wrote my poem. It is about my mother and the kitchen at home. I wrote every line myself.

The sentences are short and the words are common. Signal one measures predictability, so this kind of plain writing can score like machine writing. I have now scored this paragraph. Signal one returns `model_score` 0.6661. Signal two is still the stub, which returns 0.5, so an equal mix of 0.6661 and 0.5 lands on about 0.58, inside the placeholder unsure band. The model score alone is already above the placeholder AI cut of 0.65. If the label followed signal one by itself, the reader would see the AI label on a poem the person wrote.

The writer can take the `content_id` from the response and `POST /appeal` with a reason. The log keeps the original decision and adds a line with status `under_review`. Nothing is re-scored. A person has to read the appeal.

The setup sample, which was bland generated prose, came back at 0.8864. This poem came back at 0.6661. Both are high. The thresholds have to keep a poem at 0.6661 from being called high-confidence AI just because the sentences are short.

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

**What it measures:** The shape of the text, not what it says. Three measures in `stylometry.py`. `sentence_length_spread` is how much sentence length varies. `type_token_ratio` is unique words divided by total words. `punctuation_density` is how much punctuation there is relative to length. `style_signal` is supposed to combine them into one score from 0 to 1, higher meaning more likely AI. Those functions are stubs today.

**Why that might differ between human and AI writing:** Machine text often keeps a steadier sentence length and cleaner punctuation than a person drafting. Word variety can shift too. I am not treating that as settled until I run the same pieces through both signals in Milestone 4.

**What it can't see:** It cannot read. Typos, broken sentences, and a pasted quotation change the shape and move the score without changing who wrote it. That is cheap to do on purpose.

### The combining rule

**The rule:** Not chosen yet. Milestone 4. I will write the rule in words before I change the code. The starter currently averages the two scores. When the signals disagree, that average hides the disagreement in the middle. I have not decided whether that is what I want.

**The numbers:** Placeholder only. `WEIGHT_MODEL_SIGNAL` is 0.5 and `WEIGHT_STYLE_SIGNAL` is 0.5 in `config.py`. The label cuts in that same file are 0.35 and 0.65, also placeholders.

**Where it lives:** `scoring.py::combine_signals`

**A case where my two signals split, and what my rule does with it:** Not yet. Signal two does not vary yet, so there is no real split to record. The false-positive paragraph above is the case the rule has to be able to handle once both signals are real.



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
