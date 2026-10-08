# TakeMeter

A four-way classifier for r/soccer comments. It labels a comment by **what kind of contribution it makes** — not whether the opinion is correct or popular — so a reader can filter a thread for explanation, opinion, jokes, or conversation.

## Demo video

**[▶ Watch the demo](https://drive.google.com/file/d/1rf8lB0ulQccYxWxdzsWUeNNhYw_y3Qxe/view?usp=sharing)**

<!-- Replace PASTE_YOUR_VIDEO_URL_HERE with the real link once uploaded. -->

Covers five test-set comments classified live with label and confidence, one correct prediction explained, one incorrect prediction explained, and a walkthrough of the evaluation report below.

---

## Label taxonomy

Four labels, one per comment. Full rubric, assignment order, and boundary rules: [`planning.md`](planning.md).

### reasoned contribution
> Supplies concrete information, evidence, or an explicit reason supporting a football-related claim.

- *"The article says Felix and Conceição also didn't train."*
- *"Should have been a red straight away as it was not a ball oriented tackle."*

### unsupported take
> Expresses a substantive evaluation, prediction, accusation, or recommendation **without** supplying concrete support or an explanatory reason.

- *"They should get an additional punishment for showing no remorse after the verdict."*
- *"We all knew they were cheats. Every single one of us knew it, and yet it happened unimpeded for a decade."*

### banter
> The main contribution is an identifiable joke, sarcastic remark, wordplay, or playful taunt rather than a literal argument or request.

- *"British press Eddie, best in the world."*
- *"Messi goodbye match will be against Benin? Lol"*

### conversational exchange
> Primarily asks a genuine question, acknowledges another commenter, or gives an immediate reaction or play-by-play observation without developing a standalone argument.

- *"Anyone have a stream link?  None are working for me"*
- *"True!"*

### Assignment order

Comments routinely satisfy more than one definition, so the labels are applied as an **ordered cascade**, not four independent judgements:

1. Provides concrete information, evidence, or an explicit explanatory reason → **reasoned contribution** (this wins even if the comment also contains jokes or insults)
2. Otherwise, an identifiable joke/sarcasm/taunt is its main purpose → **banter**
3. Otherwise, makes a standalone evaluation, prediction, accusation, or recommendation → **unsupported take**
4. Otherwise, asks a genuine question, acknowledges someone, or reports an immediate reaction → **conversational exchange**

Rhetorical questions are classified by their implied assertion. Length, profanity, vote count, agreement, and the token `lol` are explicitly **not** labeling criteria.

---

## Community and task

**Community: r/soccer.** I chose it because its threads mix modes that are usually studied separately — live match reaction, tactical and player analysis, club-governance controversy (the Manchester City case runs through much of this data), transfer speculation, and rivalry humor. Crucially, *a single comment routinely contains several of these at once*: profanity, real football knowledge, sarcasm, and an argument, in four lines. That makes it a poor fit for sentiment or keyword matching and a good fit for a taxonomy about what a comment is *doing*.

**What the tool is for.** TakeMeter is a browsing filter. A reader who wants to understand *why* a result happened should be able to surface explanation; a reader who wants the jokes should be able to surface those. It sorts contributions by kind.

**What it is deliberately not for.** The labels do not encode quality, correctness, or agreement. *Unsupported take* is not an insult and *reasoned contribution* is not an endorsement — a comment can be well-reasoned and wrong, or a bare opinion and correct. `planning.md` commits to this explicitly: these labels "do not support automatic removal of comments or ranking fans by opinion quality," and low-confidence predictions should stay visible with an uncertainty indicator rather than being silently filtered out.

### Design decisions behind the rubric

These are the choices a reader needs in order to judge the label set, each with the reasoning:

1. **Function, not topic or sentiment.** Labels are verbs. The same claim about City's finances can be any of the four labels depending on whether it explains, asserts, jokes, or asks. This is the whole premise, and — as the [Reflection](#reflection-what-i-defined-vs-what-the-model-learned) documents — it is also the thing the model failed to learn.
2. **An ordered cascade rather than four independent judgements.** Support → humor → standalone assertion → conversational function, with rule 1 overriding the rest. Mixed-mode comments are the norm here, not the exception, so a tie-breaking order was necessary for annotation to be reproducible at all. (It also turned out to be unlearnable by a softmax classifier; see the Reflection.)
3. **Support need not be true, verified, or persuasive.** Only explanatory. This keeps the classifier from drifting into fact-checking, which is a different and much harder task, and keeps annotation from turning into a referendum on football opinions.
4. **Circular reasons don't count.** "He's bad because he's terrible" restates rather than explains, so it stays an *unsupported take*. Added after stress-testing exposed the loophole.
5. **Explicit non-criteria.** Length, profanity, vote count, agreement, and the token `lol` are all declared irrelevant to the label. Each was a confound I noticed myself being pulled by while reading the sample.
6. **One label per comment, no `other` class.** Uncertain cases get recorded in a notes column instead, so that a vague escape hatch doesn't quietly absorb the hard boundary cases the rubric exists to resolve.
7. **Context-dependent comments are flagged, not guessed.** Where sarcasm cannot be distinguished from literal speech without the parent comment, the rubric calls for a context-review flag. This matters because the deployed model sees comment text only.

---

## Dataset

265 labeled rows in [`labeled_comments.csv`](labeled_comments.csv):

- **185 real** r/soccer comments, collected from two public r/soccer comment exports of 100 each and merged after cleaning.
- **80 synthetic** comments (40 banter, 40 conversational exchange) generated to rebalance the two minority classes. These occupy **data rows 186 to 265** of the CSV: rows 186 to 225 are banter, rows 226 to 265 are conversational exchange. Rows 1 to 185 are entirely real.

Split 70/15/15, stratified, `random_state=42` → **185 train / 40 validation / 40 test**.

> **Known defect in this run, discussed in detail below:** the augmentation was logged at the time with an explicit constraint, recorded in `planning.md`, that synthetic rows be used *for training only* and that real comments be split *before* synthetic rows are added. The notebook instead ran a single stratified random split over the whole augmented file. **12 of the 40 test examples (30%) are AI-generated.** All headline numbers below are reported on that contaminated test set, because that is the run that produced them — the [Real vs. synthetic](#the-biggest-finding-30-of-the-test-set-is-synthetic) section separates the two.

### How the data was collected and annotated

**Collection.** Two public r/soccer comment exports, 100 comments each, merged into a single working file. Removing `[removed]`, `[deleted]`, and empty entries left **185 usable comments** — short of the 200-usable target in `planning.md`, and no additional collection round was run to close the gap.

**Annotation.** All 185 real comments were labeled in one pass against the frozen rubric, with a notes column for difficult cases; **90 of the 185 rows carry a difficulty or interpretation note**, which is itself a signal about how much of this task lives at the boundaries. Two automated bot messages were identified in notes for removal before any human-discourse modeling.

**The central data limitation:** every label in `labeled_comments.csv` is an **AI-generated draft**. `planning.md` specified that a second annotator independently label ≥40 comments, that raw agreement and Cohen's kappa be computed before adjudication, and that validation/test gold labels be produced by a human without seeing AI suggestions. **None of that was done.** No human correction pass, no agreement measurement, no independent gold set. Consequently every model score in this report is measured against unvalidated labels, which caps what any of the numbers can mean — a model error and an annotation error are not distinguishable here. This is disclosed in full in [AI usage](#ai-usage).

### Label distribution

| Label | Real only (185) | | Final CSV (265) | |
|---|---:|---:|---:|---:|
| reasoned contribution | 80 | 43.2% | 80 | 30.2% |
| unsupported take | 48 | 25.9% | 48 | 18.1% |
| conversational exchange | 35 | 18.9% | 75 | 28.3% |
| banter | **22** | **11.9%** | 62 | 23.4% |

The real-data distribution is the one that matters for interpreting results: *reasoned contribution* outnumbers *banter* **3.6 to 1**, and banter is under 12% of the collected data. No label exceeds the 70% threshold `planning.md` set as a trigger for mandatory re-collection, but the skew is severe enough that it drove the augmentation decision — and, as the [run history](#training-run-history) shows, was initially blamed for a failure it did not actually cause.

### Three difficult-to-label examples and the decisions made

These are genuine boundary cases from the sample, each recorded with the two labels in contention and the rule that resolved it. They are the cases the rubric exists for.

**1.** *"If this happened to City around 2020 Liverpool would have genuinely won like 3-4 in a row, Klopp wouldn't have burned out"*
**reasoned contribution vs. unsupported take → labeled `unsupported take`.**
The comment is specific — real clubs, a real manager, a numeric claim — and specificity reads like evidence. But it asserts a *hypothetical outcome* without explaining why that outcome would follow. The rule written from this case: **names and numbers alone do not constitute support.** Support has to do explanatory work, not merely be concrete.

**2.** *"He needs to grow up. lol. Throwing a hissy fit because he's not getting playing time lol"*
**unsupported take vs. reasoned contribution → labeled `reasoned contribution`.**
Surface features point the other way: it is insulting, informal, and contains `lol` twice. But the second sentence explains the criticism by naming a specific alleged behaviour and its trigger — that is a reason, so rule 1 applies. Two sub-rules came from this case: **support need not be verified or persuasive to qualify**, and **`lol` does not automatically indicate banter.** Labeling it *unsupported take* would have meant scoring tone rather than function.

**3.** *"More like: Your friend owes you $100 / He has $10, so he asks you for $90 / He then gives you the $100…"* (the sponsorship-financing analogy, test-set row 37)
**banter vs. reasoned contribution → labeled `reasoned contribution`.**
The comment is formatted as a joke and reads as one. It is also a working explanation of a circular-financing mechanism. Rule 1 outranks rule 2, so it is a reasoned contribution: **humorous presentation alone does not make a comment banter.** The converse case was stress-tested separately and resolved the other way — *"Our striker has scored 400 goals in my dreams"* is banter, because the number is a punchline rather than a factual assertion. The distinction is whether the humour *carries* content or *is* the content.

A fourth case worth recording because it fixed a rule: *"Spain are so good fuck"* was labeled **unsupported take**, not *conversational exchange* — it is a standalone judgement about team quality, and **profanity and brevity do not determine the label**.

**Why augmentation happened.** The pre-augmentation distribution was badly skewed — reasoned contribution 80, unsupported take 48, conversational exchange 35, banter 22 — leaving banter at 11.9% of the data. Rather than collect more real banter, 80 synthetic rows were generated for the two minority classes. The [error analysis](#the-biggest-finding-30-of-the-test-set-is-synthetic) shows this was the wrong call, and why.

### Training/test composition by origin

| Label | Train (real / synthetic) | Test (real / synthetic) |
|---|---|---|
| reasoned contribution | 56 / 0 | 12 / 0 |
| unsupported take | 34 / 0 | 7 / 0 |
| conversational exchange | 25 / 27 | 5 / 7 |
| banter | **14 / 29** | **4 / 5** |

---

## Models

| | Baseline | Fine-tuned |
|---|---|---|
| Model | `openai/gpt-oss-120b` via Groq, zero-shot | `distilbert-base-uncased` (66M params) |
| Supervision | Rubric in the system prompt, 1 example per label | 185 training examples |
| Parse failures | 0 / 40 | n/a |

**Hyperparameters changed from the notebook defaults** (3 epochs, lr 2e-5): trained for **10 epochs at lr 5e-5** with `warmup_steps=12`, `weight_decay=0.01`, batch size 16, `load_best_model_at_end` on validation accuracy.

These values were arrived at through AI consultation rather than a swept search — Gemini first, for a breakdown of the classification report, then Claude; the settings reported here are the end state of that process. See [Instance 6](#instance-6--classification-report-interpretation-and-hyperparameter-setting-gemini-then-claude). No held-out hyperparameter search was run, and no alternative configuration was trained, so these are a consulted choice, not a tuned optimum.

The arithmetic does support the direction of the change: 185 training examples at batch size 16 is ~12 optimizer steps per epoch, so the notebook default of 3 epochs is only ~35 steps — too few for a freshly initialized classification head to converge. At 10 epochs the run takes 120 steps, which matches the `[120/120]` reported in the notebook.

### Baseline: prompt and collection method

The baseline is **zero-shot** — no training examples, the rubric supplied entirely in the system prompt. The prompt is 1,804 characters and mirrors `planning.md`: one plain-language definition per label, one example each, then an explicit block of borderline-case rules carrying the same decisions recorded in [Three difficult-to-label examples](#three-difficult-to-label-examples-and-the-decisions-made).

```text
You are classifying comments from the subreddit r/soccer.
Assign each comment to exactly one of the following categories.

Reasoned contribution: The comment supports a football-related claim with concrete
  information, evidence, or an expl[anatory reason].
  Example: "Should have been a red straight away as it was not a ball oriented tackle."
Unsupported take: The comment gives an evaluation, prediction, accusation, or
  recommendation without concrete suppor[t].
  Example: "They should get an additional punishment for showing no remorse after the verdict."
Banter: The comment's main point is a joke, sarcastic remark, wordplay, or playful taunt
  rather than a literal argum[ent or request].
  Example: "British press Eddie, best in the world."
Conversational exchange: The comment asks a genuine question, acknowledges another
  commenter, or gives a quick react[ion].
  Example: "Anyone have a stream link? None are working for me"

Decision rules for borderline cases:
- Names, numbers, or hypotheticals alone are not support. "If this happened to City around 2020 …"
- A reason counts even if it is unverified or unpersuasive. "He needs to grow up. Throwing a hissy fit …"
- Humor does not make a comment Banter if it still explains something. An analogy explaining a financing scheme …
- Brevity and profanity do not decide the label. "Spain are so good fuck" is Unsupported take because it is a standa[lone judgement].
- Words like "lol" do not automatically mean Banter.

Respond with ONLY the label name.
Do not explain your reasoning.
Valid labels:
Reasoned contribution
Unsupported take
Banter
Conversational exchange
```

*Transcribed from the executed notebook, where long lines are clipped at the page margin in the exported copy; `[bracketed]` text completes a clipped clause from the rubric it was copied from, and `…` marks a clipped example. The structure, rules, and label list are verbatim.*

**How results were collected.** Each of the 40 test comments was sent individually to `openai/gpt-oss-120b` via the Groq API at `temperature=0`, with the prompt above as the system message and `Classify this post:\n\n{text}` as the user message. `max_tokens` was deliberately left unset — this model reasons before answering, and a tight cap returns an empty string instead of a label. Responses were lowercased and stripped, then matched against the label strings **longest-first**, so that a label which is a substring of another cannot be matched by mistake. Anything unmatched would be recorded as `None` and excluded from the metrics; **0 of 40 responses failed to parse**, so the baseline is scored on the full test set. A 0.1 s delay between calls respected free-tier rate limits.

Two design points worth noting. The baseline sees **the same 40-row test split** as the fine-tuned model, so the comparison is like-for-like — including the 30% synthetic contamination, which inflates both scores. And because the rubric is supplied in-context rather than learned, the baseline is effectively a direct test of *whether the label definitions are clear enough to follow* — which makes its 0.725 accuracy, against the fine-tuned model's 0.650, a meaningful result about the task rather than only about the models.

### Training run history

The reported model is not the first run. At least three were trained, and the earlier two **mode-collapsed** — the model learned to predict one class almost always:

| Run | Data | Test rows | Config | Fine-tuned result |
|---|---|---|---|---|
| A | 185 real rows | 28 | 3 ep, 2e-5, **warmup 50** | **0.500** (14/28) — collapsed; baseline on this split was 0.643 |
| B | 265 rows (augmented) | 40 | 3 ep, 2e-5, **warmup 50** | **0.400** (16/40), macro F1 **0.25** — collapsed |
| **C (reported)** | 265 rows (augmented) | 40 | **10 ep, 5e-5, warmup 12** | **0.650** (26/40), macro F1 **0.615** |

Both failures were the same pathology. Run A predicted *reasoned contribution* for **24 of 28** samples; run B for **36 of 40**. Both assigned *every* banter comment and *every* unsupported take to that one class and never emitted either label once — per-class F1 of 0.00 for both. At 0.400, run B is only 10 points above always guessing the majority class, which is essentially what it was doing.

**The cause was a warmup schedule longer than the training run.** This is arithmetic, not inference:

| Run | Train rows | Steps/epoch (bs 16) | Total steps | `warmup_steps` | Consequence |
|---|---|---|---|---|---|
| A | 130 | 9 | **27** | 50 | warmup never completes — LR peaks at **1.08e-5**, 54% of nominal |
| B | 186 | 12 | **36** | 50 | warmup never completes — LR peaks at **1.44e-5**, 72% of nominal |
| C | 186 | 12 | **120** | 12 | warmup done at 10% of run; 90% of training at full LR |

With linear warmup, the learning rate ramps from zero and reaches its nominal value only at `warmup_steps`. In runs A and B that point was never reached: training ended while the LR was still ramping, so the model never trained at 2e-5 at all. A freshly initialized classification head given ~30 steps at a fraction of the intended LR does the only thing it can — it learns the class prior and predicts the largest class. That is textbook majority-class collapse, and the fix is more effective training, which is exactly what run C's 120 full-LR steps provided.

This matters for reading the AI advice: the problem was that **warmup was already too long relative to the run**, and the recommendation received was to *raise* it to 100–150 and *lower* the LR to 1e-5 — both strictly in the wrong direction. See [Instance 6](#instance-6--classification-report-interpretation-and-hyperparameter-setting-gemini-then-claude).

Two things follow that matter more than the hyperparameters:

1. **Augmentation did not fix the problem it was added for — and this was a falsified prediction, not just a disappointment.** The synthetic rows were generated on the explicit diagnosis that mode collapse was caused by class imbalance, with the prediction that balancing the classes would resolve it. Run B is already post-augmentation. It still collapsed, and *banter* — the class the synthetic rows existed to rescue — still scored 0.00. The hypothesis was tested and failed. The 80 synthetic rows bought no minority-class performance and left behind [30% test contamination](#the-biggest-finding-30-of-the-test-set-is-synthetic). Full provenance of that recommendation is in [Instance 6](#instance-6--classification-report-interpretation-and-hyperparameter-setting-gemini-then-claude).
2. **The reported run is a survivor.** Runs A and B were discarded after inspecting their results, and run C's hyperparameters were chosen in response to those results. The test set was therefore consulted during model selection, which is exactly the leakage `planning.md` set out to prevent when it said to "freeze the rubric, model, and any confidence threshold using training/validation data before testing." The 0.650 figure is mildly optimistic for that reason, on top of the contamination.

### The reported run

The run overfits. Training loss fell from 1.39 to **0.046**, while validation loss bottomed out at **0.838 (epoch 6)** and then rose to 0.951 by epoch 10. Validation accuracy plateaued at 0.725 from epoch 6 on, so `load_best_model_at_end` restored the epoch-6 checkpoint. Note that the selected checkpoint scored 0.725 on validation but **0.650 on test** — checkpoint selection on 40 held-out examples is itself noisy.

---

# Evaluation Report

## Overall accuracy

| Model | Accuracy | Macro F1 | Weighted F1 | 95% CI (Wilson) |
|---|---|---|---|---|
| Zero-shot baseline (gpt-oss-120b) | **0.725** (29/40) | **0.71** | 0.72 | [0.572, 0.839] |
| Fine-tuned DistilBERT | **0.650** (26/40) | **0.615** | 0.640 | [0.495, 0.779] |
| Majority class (always *reasoned contribution*) | 0.300 (12/40) | 0.115 | — | — |

**Fine-tuning made the model worse by 7.5 accuracy points and ~9.5 macro-F1 points.** With n=40 the two confidence intervals overlap across almost their entire range, so the regression is not statistically distinguishable from zero — but there is certainly no evidence that fine-tuning helped. Both models clear the majority-class baseline comfortably.

## Per-class metrics

**Fine-tuned DistilBERT** (accuracy 0.650)

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| unsupported take | 0.60 | 0.43 | 0.50 | 7 |
| conversational exchange | 0.71 | 0.83 | 0.77 | 12 |
| reasoned contribution | 0.69 | 0.75 | 0.72 | 12 |
| banter | 0.50 | 0.44 | **0.47** | 9 |
| **macro avg** | 0.63 | 0.61 | **0.61** | 40 |
| **weighted avg** | 0.64 | 0.65 | 0.64 | 40 |

**Zero-shot baseline** (accuracy 0.725, 40/40 responses parseable)

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| unsupported take | 0.45 | 0.71 | 0.56 | 7 |
| conversational exchange | 0.80 | 1.00 | 0.89 | 12 |
| reasoned contribution | 0.83 | **0.42** | 0.56 | 12 |
| banter | 0.88 | 0.78 | **0.82** | 9 |
| **macro avg** | 0.74 | 0.73 | **0.71** | 40 |
| **weighted avg** | 0.77 | 0.72 | 0.72 | 40 |

The two models fail in **opposite directions**, which is the most informative part of this comparison:

- **Banter**: baseline F1 0.82, fine-tuned 0.47. The prompted model recognizes jokes; the fine-tuned model largely does not.
- **Reasoned contribution**: baseline recall 0.42 with precision 0.83 — it is very reluctant to call something "reasoned," and when it does it is usually right. The fine-tuned model is the reverse (recall 0.75, precision 0.69): it hands out *reasoned contribution* freely.
- **Unsupported take**: the baseline over-predicts it (precision 0.45 — it is the dumping ground for anything it won't call reasoned). The fine-tuned model under-predicts it: it emitted *unsupported take* only 5 times against 7 true instances.

## Confusion matrix — fine-tuned model

Rows = true label, columns = predicted label.

| True \ Predicted | unsupported take | conversational exchange | reasoned contribution | banter | **Total** |
|---|---|---|---|---|---|
| **unsupported take** | **3** | 1 | 1 | 2 | 7 |
| **conversational exchange** | 1 | **10** | 0 | 1 | 12 |
| **reasoned contribution** | 1 | 1 | **9** | 1 | 12 |
| **banter** | 0 | 2 | **3** | **4** | 9 |
| **Total predicted** | 5 | 14 | 13 | 8 | 40 |

Diagonal = 26/40 = 0.650. 14 errors.

![Fine-tuned model confusion matrix](<confusion_matrix (1).png>)

Reading it:

- **banter → reasoned contribution (3) is the single largest off-diagonal cell**, and it is strongly directional: the reverse cell (reasoned → banter) holds 1, and banter is *never* predicted as *unsupported take* (0).
- **Banter is involved in 9 of the 14 errors** (5 as the true label, 4 as the wrong prediction) despite being only 9/40 of the test set. It is the weakest class on both precision and recall.
- *conversational exchange* is the model's default guess — predicted 14 times against 12 true instances, and it absorbs errors from all three other classes.

---

## Error analysis

### Step 1 — AI-assisted pattern surfacing, and what survived it

As required, the misclassified examples were run through an LLM to surface candidate themes before I wrote this analysis. I then checked each proposed theme against the actual examples and the confusion matrix. **Three of five proposed patterns survived unchanged, one was narrowed, and one was discarded.**

| # | Pattern proposed by the LLM | Verdict after checking |
|---|---|---|
| 1 | *banter ↔ reasoned contribution is the dominant confusion* | **Verified.** 3 errors in that one cell, the largest off-diagonal; banter touches 9/14 errors. |
| 2 | *The model reads sarcasm literally* | **Narrowed.** True for 2–3 cases, but it does not explain `Correct name xd` or `lrn2internet`, which contain no sarcasm at all. Split into two distinct failure modes below. |
| 3 | *Short, low-information posts fail more often* | **Discarded — not supported.** See below. |
| 4 | *Errors cluster at low confidence* | **Verified.** Mean confidence across all 14 errors is **0.55** (range 0.37–0.68); the three correct examples sampled scored 0.80, 0.85, 0.92. **No error exceeded 0.68, and no sampled correct prediction fell below 0.80**, so the two groups do not overlap at all. Based on only 3 correct-prediction confidences, so the threshold itself is not established. |
| 5 | *Evaluative vocabulary gets pulled toward unsupported take* | **Rejected as stated.** The model **under**-predicts *unsupported take* (5 predictions vs. 7 true), so it is not a magnet class. Only 2 errors land there. |

**Why I discarded pattern 3 (post length).** It is the kind of claim that sounds right and dissolves on inspection. Errors do skew shorter, median 8 words against 11 for correct predictions, but the share of very short items barely differs: **29% of the 14 errors are ≤5 words against 23% of the 26 correct predictions**. That is a gap of under two examples, which is noise. More decisively, the **counterexamples are the shortest items in the set**: `Goal.` (1 word), `GOOOOAL!` (1 word), and `Video anyone?` (2 words) are all correct, the last at 0.92 confidence. Length is also confounded with class, since banter and conversational exchange are both short and the real driver is class. Length is a correlate of the real problem, not the problem.

**What the LLM pass did not find, and I only caught by checking the data myself:** the train/test contamination described below. It is the largest single effect in this evaluation, and no amount of reading error text would have surfaced it — it required cross-referencing the test rows against the generated row ranges. This is worth recording as a limit of the technique: pasting in misclassified examples surfaces patterns *within the errors*, and is blind to how the evaluation set was built.

**Verification.** All 14 errors are recovered from the notebook output. I reproduced the stratified split locally and confirmed that each error text appears in the rebuilt test set with the gold label the notebook reported, and that the 14 errors reconstruct the confusion matrix cell for cell. Earlier drafts of this report worked from a clipped output that showed only 11; the three missing predictions were inferred from the matrix at the time, and the full list confirms all three inferences were right.

### Step 2 — All 14 errors

| # | Comment | True | Predicted | Conf. |
|---|---|---|---|---|
| 1 | "Wow! Crazy bidding wars. I think I'm in for a long painful 5+ years of Arsenal selling and ending up 3rd or 4th." | unsupported take | reasoned contribution | 0.65 |
| 2 | "Always takes us 10 mins to get going" | unsupported take | banter | 0.49 |
| 3 | "Correct name xd" | banter | conversational exchange | 0.66 |
| 4 | "Lmao instant response" | conversational exchange | banter | 0.56 |
| 5 | "Manchester City or AC Milan.... oooh, tough choice." | banter | reasoned contribution | 0.68 |
| 6 | "It happens, Lamine/Barna were poor the whole WC, look at them now" | reasoned contribution | conversational exchange | 0.40 |
| 7 | "Very underrated point" | conversational exchange | unsupported take | 0.61 |
| 8 | "lrn2internet" | banter | conversational exchange | 0.44 |
| 9 | "Pretty baffling how fraud at this scale can be managed for a decade… Guessing the funding was from a parent company of a parent company…" | reasoned contribution | unsupported take | 0.51 |
| 10 | "Ballon dor voting finished 3 days ago" | reasoned contribution | banter | 0.65 |
| 11 | "This match is sponsored by the snooze button." *(generated)* | banter | reasoned contribution | 0.49 |
| 12 | "They need to do a Rangers: to the bottom of the pyramid" | unsupported take | banter | 0.62 |
| 13 | "Thank goodness.  The Gunners were in need of something like this." | unsupported take | conversational exchange | 0.58 |
| 14 | "He did it on purpose as he knew the title would have been taken away anyway. #respect" | banter | reasoned contribution | 0.37 |

Every error is a collected comment except #11, which is one of the generated rows. Confidence across all 14 runs from 0.37 to 0.68, mean **0.55**.

### Step 3 — Three errors in depth

---

#### Error 5 — "Manchester City or AC Milan.... oooh, tough choice."
**True: banter → Predicted: reasoned contribution (0.68)**

This is the clearest instance of the dominant `banter → reasoned contribution` cell, and the model's confidence (0.68, its joint-highest on any error) shows it is not hedging.

**Why the boundary is hard.** Every *surface* feature here points at *reasoned contribution*. It names two real clubs, it is syntactically a declarative comparison, it has no `lol`, no emoji, no profanity, no question mark. The joke lives entirely in `oooh, tough choice.` — a sarcastic marker whose meaning depends on knowing that a player choosing between those two clubs is *not* a hard decision in the speaker's view. That is world knowledge plus pragmatic inversion. DistilBERT has 66M parameters and saw 185 examples; it has no mechanism for inversion, so it falls back on the strongest lexical signal it has, which is "named football entities + declarative comparison."

This is confirmed by the directionality in the matrix: banter is misread as *reasoned contribution* 3 times but never once as *unsupported take*. The model is not randomly confused about jokes — it specifically routes entity-bearing, assertion-shaped jokes to the class it associates with entities and assertions.

**Labeling problem or data problem?** A **data problem**, and the gold label is right. `planning.md` anticipated exactly this case: its rule *"humorous presentation alone does not make a comment banter"* and its converse were stress-tested during planning, and the synthetic probe `"Our striker has scored 400 goals in my dreams."` is the same construction — a factual-looking claim whose content is a punchline. The rubric is sound; the training set just never taught the boundary.

---

#### Error 10 — "Ballon dor voting finished 3 days ago"
**True: reasoned contribution → Predicted: banter (0.65)**

The mirror image of Error 5, and it isolates the variable cleanly.

This comment is a flat, checkable fact deployed to correct someone — a textbook *reasoned contribution* under rule 1 of the rubric ("supplies concrete information"). The model called it a joke, confidently.

**Why?** Put it beside Error 5 and the pattern is exact: the model has learned a **register heuristic rather than a function heuristic**. Error 5 is long-ish, entity-rich, punctuated with ellipses → "reasoned." Error 10 is six words, terse, no hedging, conversational cadence → "banter." The actual rubric distinction is about *what the comment does* (does it supply information?), and on that axis the two are swapped. The model is reading style where the label is defined by function.

Length is involved here, but note the direction cuts against proposed pattern 3: this short post was pushed *into* banter, while short posts elsewhere (`Correct name xd`, `lrn2internet`) were pushed *out of* banter into conversational exchange. Brevity isn't a consistent force; it just lets register dominate because there is nothing else to go on.

**Labeling problem or data problem?** **Data.** 56 real *reasoned contribution* examples were available for training, but `planning.md` notes they skew toward long, explanatory comments about City's finances and tactical arguments. A six-word factual correction is an underrepresented shape of that class.

---

#### Error 9 — "Pretty baffling how fraud at this scale can be managed for a decade…"
**True: reasoned contribution → Predicted: unsupported take (0.51)**

> Pretty baffling how fraud at this scale can be managed for a decade. How hard can it be to trace where the money is coming from?
>
> Guessing the funding was from a parent company of a parent company of a parent company and so on, ultimately run by the same owners as City

Lowest-confidence error but one of the most diagnostic, because this is the boundary `planning.md` itself predicted would be hardest.

**Why the boundary is hard.** This comment genuinely contains both signals, in that order. It **opens** like an unsupported take: an evaluative reaction (`Pretty baffling`) and a rhetorical question. Only in the second paragraph does it do the work that earns *reasoned contribution* — it proposes a concrete mechanism (nested parent companies with common ultimate ownership) that would explain how the money was untraceable. The explanatory content is at the end, after the model has already seen 25 tokens of evaluative framing.

The rubric handles this explicitly: rule 1 takes precedence, so a comment that supplies an explanatory reason is *reasoned contribution* even when it also emotes, and `planning.md` also fixed that `"support need not be verified or persuasive to qualify"` — `Guessing...` is hedged, but it is still a proposed mechanism. The model learned no such precedence; it appears to have weighted the opening.

**Labeling problem or data problem?** This one is the closest to a genuine **boundary** problem rather than either. I re-read it against the frozen rubric and the gold label holds — but it holds *because of an ordering rule*, which is precisely the kind of distinction that a bag-of-features classifier cannot represent. The 0.51 confidence is the model honestly reporting a near-tie. If the rubric needs a tightening anywhere, it is here: the definition should state that hedged mechanisms (`guessing`, `probably`) count as support, since that is doing a lot of silent work.

**Corroboration:** the baseline fails the same way on this class — its *reasoned contribution* recall is 0.42, and its *unsupported take* precision is 0.45, meaning it too dumps reasoned comments into *unsupported take*. **Both models, trained and prompted completely differently, confuse the same boundary in the same direction.** That is strong evidence the difficulty is in the task definition, not in DistilBERT.

---

### The biggest finding: 30% of the test set is synthetic

Cross-referencing the 40 test rows against the generated rows (CSV data rows 186 to 265):

| | Test n | Errors | Accuracy |
|---|---|---|---|
| **Real** r/soccer comments | 28 | **13** | **0.536** |
| **Synthetic** generated comments | 12 | **1** | **0.917** |

Exact, not estimated: all 14 errors are matched to specific test rows, and exactly one of them is a generated comment.

The 12 generated rows that landed in the test set, listed so the finding can be checked against `labeled_comments.csv` directly:

| Test position | Label | Text |
|---:|---|---|
| 1 | conversational exchange | That answers my question, thanks. |
| 8 | conversational exchange | Is the draw before or after the late match? |
| 9 | banter | Someone tell the striker the goal is not a suggested destination. |
| 10 | conversational exchange | The away end is bouncing right now. |
| 13 | conversational exchange | Fair enough, I see what you mean now. |
| 18 | conversational exchange | What time is kickoff in the UK? |
| 24 | banter | This match is sponsored by the snooze button. |
| 25 | banter | We are collecting yellow cards like they come with loyalty points. |
| 27 | banter | Put the crossbar on the payroll, it is doing all the defending. |
| 32 | banter | We have signed so many backup keepers we can field a starting eleven of them. |
| 33 | conversational exchange | GOOOOAL! |
| 34 | conversational exchange | Full time. See you in the next match thread. |

Positions are 0-based in the test split, reproducible with `random_state=42` at both split calls.

Per class, the split is stark:

| Label | Real recall | Synthetic recall |
|---|---|---|
| conversational exchange | **3/5 = 60%** | **7/7 = 100%** |
| banter | **0/4 = 0%** | **4/5 = 80%** |

**The headline 0.650 is 0.536 on real Reddit comments, lifted to 0.650 by a 30% slice of generated text the model finds far easier (0.917 on it).** The reported banter F1 of 0.47 is propped up entirely by that slice: the model got **all four real banter comments wrong**, so its real-world banter recall is **zero**. Every point of banter credit in this report comes from comments I generated myself.

**Why this happened, mechanically.** Training banter was 14 real examples vs. 29 synthetic. The real ones are irreducibly weird Reddit register — `Rogue fax machine part 2 electric boogaloo`, `Izi Pizi`, `Bros moving like Seth MacFarlane`, `*Panic on the streets of Openshaw*`. The synthetic ones are uniform, well-formed observational jokes on a single template: `Put the crossbar on the payroll, it is doing all the defending.` / `We are collecting yellow cards like they come with loyalty points.` / `We have signed so many backup keepers we can field a starting eleven of them.`

Two-thirds of the banter signal therefore pointed at one narrow comedic register. The model learned that register — and it scores 80% on held-out examples of it. It did not learn banter.

Worse, **that template is declarative, entity-bearing, and assertion-shaped** — structurally indistinguishable from *reasoned contribution* on the features a small model can extract. This is very likely why `banter → reasoned contribution` is the largest error cell: the augmentation intended to fix the minority class actively taught the confusion that dominates the confusion matrix.

The augmentation also diluted the real signal rather than reinforcing it. The dataset contains 22 real banter comments in total; the stratified split put only **14 of them in training**, with the other 8 held out. Augmentation raised the banter training count from 14 to 43 without adding a single new real example — so the class hit a healthy-looking size while its real-world coverage stayed at 14, far below the ~50-per-label target in `planning.md`.

This is not a surprise defect; `planning.md` stated the correct protocol in advance ("split the real examples by thread, then add synthetic examples only to training"). The notebook's single stratified `train_test_split` over the augmented CSV silently overrode it.

### Against the success criteria in `planning.md`

| Requirement (prototype gate) | Target | Measured | Verdict |
|---|---|---|---|
| Macro F1 | ≥ 0.75 | 0.615 | **FAIL** |
| F1 for every label | ≥ 0.60 | banter 0.47, unsupported take 0.50 | **FAIL** |
| Macro F1 vs. majority baseline | ≥ +0.10 | 0.615 vs. 0.115 (+0.50) | PASS |
| ≥ 5 gold examples per test label | ≥ 5 | min 7 (unsupported take) | PASS |
| Independent human annotation agreement | ≥ 80% raw | not measured — all labels are AI drafts | **INSUFFICIENT EVIDENCE** |
| Test set is real community data | required | 70% real | **FAIL** |

**The fine-tuned model does not meet the prototype bar.** Under the project's own decision procedure — all applicable gates must pass — this run is a fail, and the annotation-agreement gate was never measured, so even the passing metrics rest on unvalidated gold labels. The zero-shot baseline (macro F1 0.71) is closer to the bar but also misses it, and was scored on the same contaminated test set.

---

## Reflection: what I defined vs. what the model learned

The gap here is not that the model got 14 examples wrong. It is that the model and I were doing two different tasks that happen to agree 65% of the time.

**I defined the labels by communicative function. The model learned register.** The entire intellectual content of the rubric in `planning.md` is the claim that *the same content can belong to any of the four classes depending on what it does*. A joke that explains a financing mechanism is a reasoned contribution. A factual-sounding number used as a punchline is banter. A six-word correction is a reasoned contribution; a sixty-word rant is an unsupported take. The labels are verbs, not topics.

What the model learned instead is a prototype for each class built out of surface style: long, entity-dense, multi-clause → *reasoned contribution*; terse and casual → *banter*; question marks and acknowledgement formulas → *conversational exchange*. On that axis the model is actually quite good. It is just a different axis from the one I defined.

The sharpest evidence is that `planning.md` contains an explicit list of features that **must not** determine the label — and the model keyed on nearly every one of them:

| Rule I wrote | What the model did | Error |
|---|---|---|
| "specific names and numbers alone do not constitute support" | Read `5+ years… 3rd or 4th` as support | 1 |
| "`lol` does not automatically indicate banter" | Read `Lmao` as a banter marker | 4 |
| "Length… [is] not [a] labeling criterion" | Read a 6-word factual correction as banter | 10 |
| "brevity and profanity do not determine the label" | Read terse casual phrasing as banter | 2 |

This is consistent with discarding "short posts fail more often" earlier, and the distinction matters. That discarded claim was about *error rates* — short posts are not disproportionately wrong, and `Goal.`, `GOOOOAL!` and `Video anyone?` are all correct. The claim here is different: when a comment is short, there is little else for the model to go on, so **register decides by default** — and register points in whatever direction the training data happened to associate it with. That is why brevity pushes `Ballon dor voting finished 3 days ago` *into* banter while pushing `Correct name xd` *out of* it. Length is not a cause of error; it is a condition under which the model's real criterion becomes the only criterion.

I did not write those rules arbitrarily. I wrote them because, during the stress-testing pass, *those were the confounds I could see myself being tempted by*. I encoded them as warnings in prose — and prose warnings are exactly the part of a rubric that does not survive contact with a training set. The model never read `planning.md`. It only ever saw 185 (text, label) pairs, and in those pairs the confounds and the labels were correlated. I told the annotator what to ignore and then handed the model a dataset in which the thing to ignore was predictive.

**The rubric is a decision procedure; the model can only learn a similarity function.** My assignment rules are an ordered cascade: check for support first, then humor, then standalone assertion, then conversational function, with rule 1 explicitly overriding the rest. That is an algorithm executed over a whole comment. A softmax classifier has no representation of precedence — it pools evidence and normalizes. So for any comment that carries two signals, the rubric says "the earlier rule wins" and the model says "whichever signal is stronger wins." Error 9 is this exact failure: an evaluative opening followed by an explanatory mechanism, where the rubric is unambiguous that the explanation decides it, and the model returned 0.51 — an honest near-tie between two signals that my rubric says should never have been weighed against each other. This is structural, not a data-volume problem. More examples would shift the boundary; they would not give the model a notion of precedence.

**What it overfit to.** Most concretely, the synthetic comedic template. I intended *banter* to name a function — the point of the comment is the joke. The model learned one generator's dialect of that function, and scores 80% on held-out samples of that dialect against **0% on real Reddit banter**, where it missed all four. The augmentation added 29 rows to the class and roughly zero new coverage of the concept, because all 29 rows were voiced by the same writer. Worse, that voice is declarative and entity-bearing, so the banter boundary I accidentally taught *overlaps* the reasoned-contribution boundary — which is why `banter → reasoned contribution` is the largest error cell. I did not merely fail to teach banter; I taught a version of it that collides with another class.

**What it missed.** Pragmatic inversion, entirely. Sarcasm, rhetorical questions, and implicature are load-bearing in my rubric — rhetorical questions are supposed to be classified by their implied assertion, and sarcasm is supposed to route to banter. None of this has a lexical signature; it requires modeling what a speaker means against what they said. A 66M-parameter encoder with 185 examples has no mechanism for it, and the matrix shows the absence cleanly: banter is predicted as *reasoned contribution* three times and as *unsupported take* **zero** times. Assertion-shaped jokes are never recognized as jokes — they are sorted by how much they look like substance.

**One flaw that is mine, not the model's.** Three of my four labels are defined by something present — evidence, a joke, a question. *Unsupported take* is defined by something **absent**: an assertion with no support. A class defined by absence has no positive feature signature, and both models struggle with it in telling opposite directions: the fine-tuned model under-predicts it (5 predictions against 7 true instances, 0.43 recall), while the zero-shot baseline over-predicts it into a dumping ground (0.45 precision). When two systems with nothing in common both fail on the same class in complementary ways, the problem is likelier in the definition than in either system. *Unsupported take* is currently the residue left over after the other three rules decline — and I asked a classifier to learn a residue.

**What this means for the tool.** TakeMeter was meant to let a reader filter a thread for explanation versus opinion versus jokes. The boundary the model actually learned would sort that thread largely by length and formality. That is not worthless — it correlates with what I wanted — but it fails in the specific way that would most damage trust: because sarcasm routes *into* the reasoned-contribution class, the "show me the substance" filter would quietly seed itself with jokes. That is also why `planning.md` set a 0.85 precision gate on reasoned contribution for deployment, and why 0.69 is not close.

The honest summary: I built a taxonomy about intent and trained it on a dataset where intent was confounded with style, then measured it on a test set where 30% of the text was written by a generator rather than by the community. The model learned the most predictive thing available to it. The gap between that and my definitions is a measure of how much of my rubric lived only in prose.

---

## What would need to change

Ordered by expected impact:

1. **Re-split before anything else, and assert the split in code.** Partition the 185 real comments into train/val/test *first*, add the 80 synthetic rows to train *only*, and re-score. This costs nothing and is the only way to know what the real numbers are. Every figure in this report should be treated as provisional until that re-run. Then make the rule self-enforcing with one line that fails loudly rather than silently:

   ```python
   SYNTHETIC_ROWS = range(185, 265)          # 0-based; CSV data rows 186-265
   synthetic = set(df.iloc[SYNTHETIC_ROWS]["text"])
   leaked = set(test_df["text"]) & synthetic
   assert not leaked, f"{len(leaked)} synthetic rows leaked into the test set"
   ```

   The spec stated this constraint correctly and nothing checked it; an assertion is the difference between a rule and a comment.
2. **Collect real banter instead of generating it.** Banter needs ~50 real examples spanning its actual range — meme references, sarcasm, wordplay, terse taunts (`lrn2internet`, `Correct name xd`), and deadpan jokes. 14 real examples cannot cover a class this stylistically diverse, and generated substitutes made the dominant error cell worse rather than better.
3. **Target the `banter → reasoned contribution` boundary with hard pairs.** Mine or write minimal pairs that hold entities and declarative structure constant and vary only intent: `Manchester City or AC Milan.... oooh, tough choice.` (banter) against `Manchester City and AC Milan both bid, and he picked City for the Champions League football.` (reasoned). The model currently has no example that separates these.
4. **Add short factual corrections to *reasoned contribution*.** `Ballon dor voting finished 3 days ago` is a shape of that class with almost no representation. Training it only on long explanatory comments taught the model that reasoning means length.
5. **Tighten the rubric on hedged support.** State explicitly that a hedged mechanism (`guessing`, `probably`, `I think it's because…`) counts as support. Error 9 depends on this and the current wording leaves it implicit.
6. **Run the annotation-agreement check `planning.md` specifies** — 40 comments, second annotator, Cohen's kappa, before adjudication. All 265 labels are currently unreviewed AI drafts, which caps how much any model score can mean.
7. **Lower expectations for DistilBERT on this task, or change model.** The baseline beat it while seeing zero training examples. Sarcasm detection and function-over-register classification are exactly where a 66M-parameter encoder with 185 examples is weakest. If the fine-tuned model still loses after a clean re-split, the honest conclusion is that a prompted large model is the better tool for this taxonomy — and the fine-tuning result is evidence about task difficulty, not a failed experiment.

---

## Sample Classifications

Five test-set posts run through the fine-tuned model, with its predicted label and softmax confidence. Three correct, two wrong.

| Post (truncated) | True label | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| Video anyone? | conversational exchange | conversational exchange | 0.92 | yes |
| That answers my question, thanks. *(synthetic)* | conversational exchange | conversational exchange | 0.85 | yes |
| You're obviously angry.  That's understandable. | unsupported take | unsupported take | 0.80 | yes |
| Wow! Crazy bidding wars. I think I'm in for a long painful 5+ years of Arsenal selling and... | unsupported take | reasoned contribution | 0.65 | no |
| Always takes us 10 mins to get going | unsupported take | banter | 0.49 | no |

`You're obviously angry. That's understandable.` is the most interesting correct prediction, and at 0.80 it is not a lucky guess. It is a standalone judgement about another commenter with no supporting reason offered — rule 3 of the rubric — and the model resisted two plausible traps: it did not read `That's understandable` as an acknowledgement (which would have made it *conversational exchange*), and it did not treat the second-person address as conversational. That is the model applying the function test correctly on a real comment.

The two correct *conversational exchange* rows are high-confidence but less informative: `Video anyone?` is an unambiguous question, and `That answers my question, thanks.` is a synthetic row, i.e. generated by the same process that produced 27 of the 52 training examples for that class — which, as shown above, is close to a free win.

The confidence column separates the two groups cleanly: the three correct predictions score 0.80–0.92, while the two errors score 0.65 and 0.49. That gap holds across the whole error set — no misclassification in this run exceeded 0.68 confidence (see pattern 4 in the error analysis). Both wrong rows are *unsupported take* comments that the model pushed into other classes — they appear as Errors 1 and 2 in the verified-errors table above, and both sit in the `unsupported take` row of the confusion matrix, which has the worst recall (0.43) of any class.

---

## Spec reflection

### One way the spec guided the implementation

**Pre-committing the metrics and the pass/fail gates, before any model existed, is what made this report able to say "fail."**

`planning.md` fixed three things in advance: that **macro F1 is the primary metric** (so that a common class like *conversational exchange* could not mask failure on *banter*), that per-label F1 and a confusion matrix would be reported for every label rather than just averages, and that success required **macro F1 ≥ 0.75 with every label ≥ 0.60** — with an explicit decision procedure stating that all gates must pass and "a strong average cannot compensate for a failing label."

That mattered because the result was genuinely ambiguous-looking. 0.650 accuracy on a 4-class problem is 2.2× the majority baseline, and it would have been easy to write that up as a reasonable prototype. Against pre-registered gates it is unambiguous: macro F1 0.615 fails, banter 0.47 fails, unsupported take 0.50 fails. The spec also pre-committed to *reporting support counts and uncertainty*, which is why the Wilson intervals are in the report — and those intervals are what showed the 7.5-point gap between the two models is not statistically separable at n=40. Deciding what would count as success while I still wanted the project to succeed removed my ability to move the target afterwards.

The ordered cascade did similar work at annotation time. Because rule 1 (support wins) was frozen before labeling, I could adjudicate Error 9 against a fixed rule rather than rationalize it after seeing the prediction — and conclude the gold label was right and the model was wrong, rather than the reverse.

### One way the implementation diverged, and why

**The spec specified a split protocol for synthetic data, and the implementation violated it.**

`planning.md` states the rule explicitly: *"Before retraining, split the real examples by thread, then add synthetic examples only to training. Validation and test sets must contain real, independently human-labeled comments."* `planning.md` even names the exact failure mode: *"Randomly splitting the augmented CSV would test partly on generated language and could overstate real-community performance."*

The implementation did precisely that. The starter notebook's pipeline runs a single stratified `train_test_split` over whatever CSV is uploaded, and I uploaded the augmented 265-row file. The result is that **30% of the test set is AI-generated**, and the prediction in the spec was accurate almost to the word — real-data accuracy is ~55% against ~85% on the synthetic slice.

**Why it happened** is worth being precise about, because it is not that I forgot the rule. The spec assumed I controlled the split; the notebook owns the split and exposes no hook for a pre-partitioned dataset. Honoring the spec required doing the partition *before upload* — splitting the real 185 rows myself, appending synthetic rows to the train portion only, and either uploading pre-split files or passing an index. That is a small amount of work, and I didn't notice that the notebook's convenience had quietly overridden a constraint I had already written down. The spec was right, the tooling made the wrong thing the default, and I followed the tooling.

Three further divergences, all failures to execute rather than deliberate changes:

- **The primary metric was not used for model selection.** `planning.md` designates macro F1 as primary "so a common category cannot hide poor performance on banter." The notebook's `compute_metrics` returns only accuracy, so per-epoch macro F1 was never computed, and `metric_for_best_model="accuracy"` selected the checkpoint. Gemini explicitly recommended switching to `f1_macro` and I overrode it (see [Instance 6](#instance-6--classification-report-interpretation-and-hyperparameter-setting-gemini-then-claude)) — the spec and the AI tool agreed with each other against what was actually run.
- **The test set was consulted during model selection.** The spec says to freeze the model "using training/validation data before testing." In practice two earlier runs were trained, their *test* confusion matrices inspected, and the hyperparameters revised in response (see [Training run history](#training-run-history)). Validation data existed and should have carried that decision.
- **The 200-usable-comment target** was not met (185 collected, no second round), and the **annotation-agreement protocol** — second annotator, ≥40 comments, Cohen's kappa before adjudication — was never run, leaving all 265 labels as unreviewed AI drafts.

The generalizable lesson: a constraint written in a planning document is only enforced if something in the pipeline enforces it. Both of my most serious divergences are places where the spec stated a rule correctly and nothing checked it. A single assertion, that no test row falls in the generated row range, would have caught the contamination at run time, and that check is now item 1 in [What would need to change](#what-would-need-to-change).

---

## AI usage

AI tools were used at six stages, across three different systems (Codex, Gemini, Claude). **All annotation was AI-assisted — see Instance 2, which is the most consequential disclosure in this report.** Instance 6 covers the only stage where an AI tool's output directly shaped the trained model rather than the dataset or the write-up.

### Instance 1 — Rubric stress-testing (Codex, planning stage)

**What I directed it to do.** I gave it the four label definitions, the assignment order, and the hard-case descriptions, and asked it to generate eight fictional r/soccer-style comments *sitting on the boundary between two labels* — specifically covering humorous explanations, short judgements, circular reasons, rhetorical vs. genuine questions, and context-dependent sarcasm — then to name the two competing labels for each and say which rule resolved it.

**What it produced.** Eight probes with proposed adjudications, reproduced in full as a table in `planning.md`. Three exposed real holes in the rubric as it stood: a joke statistic (`"Our striker has scored 400 goals in my dreams."`) could pass as factual support; a circular reason (`"He's bad because he's terrible."`) could pass as an explanation; and immediate match observations overlapped with reactions.

**What I changed or overrode.** I accepted the three weaknesses and wrote the fixes into the assignment rules — concrete information must be "a literal, relevant factual assertion, not a number used only as a punchline," a reason "must explain the claim rather than restate it," and match-event descriptions stay *conversational exchange* unless used to support a broader claim. I **overrode its handling of ambiguity**: where it proposed a confident label for `"Great substitution, boss."`, I changed the outcome to *requires context review*, because sarcasm and literal praise are genuinely indistinguishable there from text alone. That became a standing rubric rule — flag rather than invent intent. I also ruled that all eight probes are **excluded from training and evaluation data**; they are rubric tests, not community data, and they appear nowhere in `labeled_comments.csv`.

### Instance 2 — Annotation pre-labeling (Codex) — full disclosure

**What I directed it to do.** The original plan was narrow: pre-label development examples in batches of 20, each with one proposed label, a short rationale quoting the relevant wording, and a `context_needed` flag, which I would then review row by row against the original text. I subsequently directed it to draft labels for the **entire** merged comment file in one pass instead.

**What it produced.** All **185 labels (100% of the real dataset)**, plus difficulty notes on 90 rows, and it correctly flagged two automated bot messages in the *conversational exchange* class.

**What I changed or overrode — nothing, and that is the problem.** No human correction pass was performed. There is no `human_final_label` column, no `human_changed_label` count, no independently annotated agreement subset, and no Cohen's kappa. The provenance fields the spec required (`ai_prelabel_used`, `ai_label`, `ai_rationale`, `human_final_label`) were not carried into the final two-column CSV.

**Why this matters for every number in this report.** This diverged from the plan in a way that damages evaluation independence specifically: `planning.md` required validation and test gold labels to be produced by a human *without seeing* the AI suggestions, precisely to avoid anchoring. Because the same tool labeled train, validation, and test, **the test set is not an independent check** — a systematic labeling bias would be learned by the model and then rewarded by the scoring. Where this report says the model was "wrong," it strictly means *the model disagreed with an unreviewed AI draft label*. I re-read the specific error examples analyzed above against the frozen rubric and believe those gold labels are correct, but that is spot-checking, not the measured agreement the spec called for. **AI drafts are presented here as drafts, not as validated gold.**

### Instance 3 — Synthetic augmentation (Codex)

**What I directed it to do.** Generate 40 banter and 40 conversational-exchange comments consistent with the rubric, to rebalance the two minority classes.

**What it produced.** 80 rows, appended to the CSV as data rows 186 to 265 and logged at the time with their row ranges and a constraint restating the training-only rule.

**What I changed or overrode.** Only mechanical checks — uniqueness and label validity. **I did not review the generated text for stylistic diversity, and should have.** All 29 synthetic banter rows that landed in training are variations on a single observational-joke template, which the error analysis identifies as a direct cause of the largest error cell. The provenance manifest is the one part of this stage that worked as intended: it is what made the contamination finding possible after the fact.

### Instances 4–5 — Error analysis and report metrics (Claude, Opus 5)

**What I directed it to do.** (4) Surface common themes across the misclassified examples. (5) Recover the per-class metrics and error texts, which existed only inside the executed notebook PDF, and compute statistics the notebook did not report.

**What it produced.** (4) Five candidate patterns. (5) A local reproduction of the stratified split, plus macro F1, Wilson confidence intervals, a majority-class baseline, and the real-vs-synthetic breakdown.

**What I changed or overrode.** Of the five proposed patterns, **three were verified, one was narrowed, and one was discarded**: "short posts fail more often" dissolved on checking, because the shortest items in the test set (`Goal.`, `GOOOOAL!`, `Video anyone?`) are all correctly classified. I also rejected "evaluative language is pulled toward *unsupported take*" after checking the matrix column — the model *under*-predicts that class. Every surviving pattern is backed by a count, and the confusion matrix was verified to be arithmetically consistent with both the reported per-class metrics and the 0.650 accuracy before being used. Most importantly, **the single largest finding in this report — the 30% synthetic test contamination — was not produced by the AI pattern pass.** It required cross-referencing the test rows against the generated row ranges, which is a check on *how the evaluation was built* rather than on the errors themselves. That is a real limitation of "paste your errors into an LLM" as a technique, and worth recording.

### Instance 6 — Classification-report interpretation and hyperparameter setting (Gemini, then Claude)

**Scope note:** this was not a single question. The consultation ran to six pages of transcript covering metric interpretation, two mode-collapsed runs, the system prompt, **the decision to generate synthetic data**, and three successive hyperparameter configurations. It is the most load-bearing AI consultation in the project, because unlike the others it changed both the dataset and the trained model.

**What I directed it to do.** I pasted the `classification_report` output and asked **Gemini** what it meant; then, as runs failed, I pasted the fine-tuned confusion matrices, the label distribution, my system prompt, and my `TrainingArguments`, asking in turn "how can i correct it?" and for a review of each. I later put the same material to **Claude**. The hyperparameters finally used, 10 epochs, lr 5e-5, `warmup_steps=12` — are the end state of that process.

**Disclosure that belongs here rather than in the dataset section: the synthetic augmentation was Gemini's recommendation.** Given the 185-row distribution, it diagnosed the collapse as class imbalance ("the optimizer took the path of least resistance… it could achieve a mathematically safe error rate simply by guessing the majority class") and recommended, as its preferred option, *"Synthetic Data Generation (Recommended): Use a larger, highly capable LLM to generate 30 to 50 new, diverse examples of 'banter' and 'conversational exchange'."* The 80 synthetic rows in `labeled_comments.csv` exist because of that advice. It was a reasonable diagnosis that turned out to be **wrong**: run B was trained on the balanced data and collapsed anyway (see [Training run history](#training-run-history)). One word of the advice also went unheeded — **"diverse"** — and the lack of diversity in what was generated is the direct cause of the largest error cell in the final model.

**What it produced.** Gemini analyzed the confusion matrix of an **earlier, mode-collapsed run** (see [Training run history](#training-run-history)) and reported, accurately: the model predicted *reasoned contribution* **36 times out of 40**, dumped **100% of unsupported takes (7/7)** and **100% of banter (9/9)** into that one class, never once predicted *unsupported take* or *banter*, got 4/12 conversational exchanges, and scored **40% accuracy (16/40)**. It also noticed the test set had grown from 28 to 40 rows and inferred correctly that data had been added. I re-derived all of these numbers and they are exactly right, including the implied **macro F1 of 0.25**.

It then gave a diagnosis and two prescriptions: that the pattern "usually points to the optimizer getting stuck early in training," that `warmup_steps` should be raised **to 100 or 150** to let the optimizer stabilize, and that model selection should use **`f1_macro`** so that scoring zero on two classes is penalized.

**What I changed or overrode.** Both prescriptions were overridden. The final notebook uses `warmup_steps=12` and `metric_for_best_model="accuracy"`. One of those overrides was right and one was wrong:

- **Overriding the warmup advice was correct, and the advice was unworkable.** With 185 training examples at batch size 16, an epoch is ~12 optimizer steps. The 3-epoch configuration that produced the collapse ran **~36 steps in total**, so a 100-step warmup is **278% of the entire run** — the learning rate would never finish warming up, and the model would train even less than it already was. Even in the final 10-epoch configuration (120 steps), warmup=100 would spend **83% of training** below peak LR. The diagnosis was also inverted: this was not an optimizer stuck in a local minimum, it was a **freshly initialized classification head that had barely been trained**, and an undertrained head defaults to the largest class — which is exactly *reasoned contribution* (56 of 185 training rows). The cure for undertraining is more training, not more warmup. Raising epochs 3→10 and LR 2e-5→5e-5 resolved the collapse and took accuracy from 0.400 to 0.650.
- **Class weights were recommended with working code, and were not implemented — which was the right call for the wrong reason.** Gemini supplied a `WeightedLossTrainer` subclass and a computed weight vector `[0.57, 0.96, 1.32, 2.10]`, noting the weights assume label IDs `0=Reasoned, 1=Unsupported, 2=Conversational, 3=Banter`. The *values* are correct for this distribution. The *order is not this project's.* The notebook's actual `LABEL_MAP` is `0=unsupported take, 1=conversational exchange, 2=reasoned contribution, 3=banter`, so pasting that array as written would have assigned **0.57 to unsupported take** (should be 0.96), **0.96 to conversational exchange** (should be 1.32), and **1.32 to reasoned contribution** (should be 0.57) — up-weighting the very majority class that was causing the collapse by more than 2×. Gemini did hedge the risk in a parenthetical ("ensure the order matches your model's label2id mapping"), but the concrete artifact it handed over was misaligned with the label map it had already been shown. The standard `Trainer` was kept, so the bug never shipped.

- **Overriding the `f1_macro` advice was a mistake, and Gemini was right.** `planning.md` designates macro F1 as the **primary metric** precisely so a dominant class cannot mask failure on *banter*, and Gemini independently identified the same trap in sharper terms — it called `metric_for_best_model="accuracy"` "a critical trap… actively encouraging your model to collapse into the majority class," warning that `load_best_model_at_end` "will likely load the checkpoint that simply surrendered." Selecting checkpoints on accuracy contradicts my own spec and the advice I had in hand. It also had a concrete cost: the notebook's `compute_metrics` returns only accuracy, so **per-epoch macro F1 was never computed at all** and could not have been used for checkpoint selection even if wanted. Validation accuracy then tied at 0.725 across epochs 6, 7, 9 and 10, and the tie was broken by epoch order rather than by the metric I had committed to. This is logged as a spec divergence in [Spec reflection](#spec-reflection).

**The most important thing this exchange reveals is about the augmentation, not the hyperparameters.** Gemini's note that the test set grew 28 → 40 dates this run precisely: 185 real rows yield a 28-row test set at 15%, and 265 augmented rows yield 40. So this collapsed run is **after** the 80 synthetic rows were added. Augmentation was introduced to fix the class imbalance, and at that point it had **not fixed it** — the model was still predicting one class 90% of the time, and *banter*, the class the synthetic data was meant to rescue, still scored a flat zero. What actually fixed the collapse was the hyperparameter change. The augmentation's measurable legacy is therefore the 30% test-set contamination and the `banter → reasoned contribution` error cell, with no demonstrated benefit on the problem it was added to solve.

**Why this instance is disclosed separately.** It is the only stage where AI output fed back into the **trained artifact** rather than into data or prose: these settings produced the 0.650 model that every number in this report describes. Two consequences follow, and both are already visible in the results.

First, **no hyperparameter search was run.** One configuration was consulted and trained. The report therefore cannot claim these settings are good, only that they are the ones used — and the [Models](#models) section now says so explicitly.

Second, and more importantly, **the training curve suggests the advice was not fully acted on.** The run reaches best validation loss at **epoch 6 (0.838)** and then degrades to 0.951 by epoch 10 while training loss collapses to 0.046 — textbook overfitting across the final four epochs. `load_best_model_at_end` rescued the epoch-6 checkpoint, so the final score was not damaged, but 10 epochs at 5e-5 is past the useful point for 185 examples and the curve says so plainly. A sensible reading of the classification report *before* the final run would likely have suggested early stopping or fewer epochs. This is a limitation of consulting a tool about metrics without then letting the measured curve revise the setting.

A note on method that applies to this instance specifically: asking two models the same question and taking the second answer is not validation. Gemini and Claude agreeing would not make a setting correct, and their disagreeing would not tell me which to trust. The only thing that adjudicates a hyperparameter is a held-out comparison, which was not run here.

### Summary

| Stage | Tool | What it did | What I changed / overrode |
|---|---|---|---|
| Rubric stress-testing | Codex | 8 boundary probes; exposed 3 rule weaknesses | Accepted 3 fixes; overrode one adjudication to *context review*; excluded all probes from data |
| **Annotation** | Codex | **Drafted all 185 labels (100%)** | **Nothing — no human review pass was run** |
| Augmentation | Codex | 80 synthetic rows + SHA-256 provenance | Uniqueness/validity checks only; did not review for diversity |
| Error-pattern surfacing | Claude (Opus 5) | 5 candidate patterns | 3 verified, 1 narrowed, 1 discarded; contamination found independently |
| Report metrics | Claude (Opus 5) | Split reproduction, macro F1, Wilson CIs, real/synthetic split | Verified split against notebook; verified matrix against reported metrics |
| **Augmentation decision** | Gemini | Diagnosed collapse as imbalance; **recommended generating synthetic minority-class data** | Followed — but the prediction was falsified (run B collapsed anyway), and "diverse" was not verified |
| **Hyperparameters** | **Gemini, then Claude** | Diagnosed two collapsed runs; advised `warmup_steps=100-150`, lr 1e-5, class weights, `f1_macro` | **Warmup/LR overridden — correct** (warmup already exceeded the run). **Class weights dropped — array was misordered for this label map.** **`f1_macro` overridden — a mistake** |

Every pattern reported above is backed by a counted example. LLM explanations of *why* the model erred are hypotheses about the task, not evidence about the model's internals.

---

## Repository contents

| File | What it is |
|---|---|
| `README.md` | This evaluation report. |
| `planning.md` | Community choice, label rubric, assignment rules, metrics, and success criteria. |
| `labeled_comments.csv` | 265 labeled examples. Data rows 1 to 185 are real r/soccer comments; rows 186 to 265 are synthetic (186 to 225 banter, 226 to 265 conversational exchange). |
| `evaluation_results.json` | Accuracy numbers exported from the notebook. |
| `confusion_matrix (1).png` | Confusion matrix plot for the fine-tuned model. |

Source comment exports, the executed notebook, and the AI consultation transcripts were used during development and are not part of this submission. The findings that rely on them are reported in full above, and the numbers are reproducible from `labeled_comments.csv` alone, since the split is deterministic and the synthetic rows are identified by row range.

## Reproducing

Open the fine-tuning notebook in Colab with a T4 GPU, upload `labeled_comments.csv`, set a `GROQ_API_KEY` in Colab Secrets, and run all cells. Note that reproducing the split exactly requires `random_state=42` at both split calls; fine-tuning itself is not seeded, so per-class numbers will shift by an example or two between runs — another reason to treat n=40 results as provisional.
