# TakeMeter: classification plan

## Community and purpose

I chose Reddit's r/soccer because its discussions combine live match reactions, player and tactical analysis, club controversies, and rivalry humor, providing varied language and recognizable reasons for participating. TakeMeter will classify comments as reasoned contributions, unsupported takes, banter, or conversational exchanges, helping readers browse explanations, opinions, jokes, and questions according to what they want from a discussion. These distinctions matter because a useful football discussion can include both analysis and social interaction; the classifier will identify contribution types rather than declare opinions correct or fans' comments good or bad.

Both source JSON files identify all 200 entries as belonging to r/soccer. The observed mixture makes classification interesting: the same comment can contain profanity, football knowledge, sarcasm, and an argument, so sentiment or keyword matching alone would miss its purpose.

## Sample reviewed

I read the first 40 nonempty, nondeleted comments in the merged comment file (zero-based data rows 0–41, excluding rows 1 and 34). The sample includes match reactions, discussion of Manchester City's finances and possible sanctions, player-management arguments, questions, and jokes. Labels describe observable content rather than whether I agree with a comment, its factual accuracy, or its popularity.

## Four labels

### Reasoned contribution

Definition: A comment supplies concrete information, evidence, or an explicit reason supporting a football-related claim.

- Clear example: “The article says Felix and Conceição also didn't train.”
- Clear example: “Should have been a red straight away as it was not a ball oriented tackle.”
- Uncertain example: “If this happened to City around 2020 Liverpool would have genuinely won like 3-4 in a row, Klopp wouldn’t have burned out”
- Boundary decision: Label the uncertain example **unsupported take** because it asserts hypothetical outcomes without explaining why they follow; specific names and numbers alone do not constitute support.

### Unsupported take

Definition: A comment expresses a substantive evaluation, prediction, accusation, or recommendation without supplying concrete support or an explanatory reason.

- Clear example: “They should get an additional punishment for showing no remorse after the verdict.”
- Clear example: “We all knew they were cheats. Every single one of us knew it, and yet it happened unimpeded for a decade.”
- Uncertain example: “He needs to grow up. lol. Throwing a hissy fit because he’s not getting playing time lol”
- Boundary decision: Label this **reasoned contribution** because the second sentence explains the criticism through a specific alleged behavior and its trigger. Support need not be verified or persuasive to qualify; `lol` does not automatically indicate banter.

### Banter

Definition: A comment's main contribution is an identifiable joke, sarcastic remark, wordplay, or playful taunt rather than a literal argument or request.

- Clear example: “British press Eddie, best in the world.”
- Clear example: “Messi goodbye match will be against Benin? Lol”
- Uncertain example: The comment comparing sponsorship financing to borrowing $90 from a friend to repay that friend's $100.
- Boundary decision: Label the analogy **reasoned contribution** because it explains a financial mechanism; humorous presentation alone does not make a comment banter.

### Conversational exchange

Definition: A comment primarily asks a genuine question, acknowledges another commenter, or gives an immediate reaction or play-by-play observation without developing a standalone argument.

- Clear example: “Anyone have a stream link?  None are working for me”
- Clear example: “True!”
- Uncertain example: “Spain are so good fuck”
- Boundary decision: Label this **unsupported take** because it makes a standalone judgment about team quality; profanity and brevity do not determine the label.

## Assignment rules

Assign exactly one label according to the following order:

1. If the comment provides concrete information, evidence, or an explicit explanatory reason, choose **reasoned contribution**. This takes precedence when a comment also contains jokes or insults.
2. Otherwise, if an identifiable joke, sarcasm, or playful taunt is its main purpose, choose **banter**.
3. Otherwise, if it makes a standalone evaluation, prediction, accusation, or recommendation, choose **unsupported take**.
4. Otherwise, if it asks a genuine question, acknowledges someone, or reports an immediate reaction or match event, choose **conversational exchange**.

Rhetorical questions are classified by their implied assertion, not as genuine requests. Quoted material counts as support only when the commenter uses it to contribute information or make an argument. Length, profanity, agreement, and vote counts are not labeling criteria.

Stress-test refinements: concrete information means a literal, relevant factual assertion, not a number used only as a punchline. A reason must explain the claim rather than restate it ("he is bad because he is terrible" remains unsupported). Immediate descriptions of match events remain conversational exchanges unless used to explain a broader claim; this exception takes precedence over the concrete-information rule. For mixed questions and assertions, classify the substantive assertion first when it stands independently; otherwise classify the genuine request. If sarcasm cannot be distinguished from a literal statement without missing context, flag the comment for context review rather than inventing intent.

## Exclusivity and coverage check

The main overlaps in the sample are humorous explanations, short judgments versus reactions, and criticisms that give a reason. The ordered rules above resolve these by prioritizing support, then humor, then standalone assertions, then conversational functions. A short factual contribution can be reasoned, while a long unsupported assertion can remain an unsupported take.

The four labels appear to cover the activities in the reviewed sample, but 90% coverage of all 200 comments has not yet been verified. During annotation, record uncertain cases in a separate notes column rather than creating an `other` label; if more than 10% of usable comments do not fit, revise the definitions. Exclude `[removed]`, `[deleted]`, and empty entries; comments whose meaning depends on a missing parent should be flagged for context review rather than guessed.

For hard cases, record the two candidate labels and the reason for uncertainty, then apply the ordered rules. If the parent comment or thread title is needed, inspect that context to resolve intent and record that dependency; a model using only comment text will be evaluated separately on these cases. A second annotator should independently label at least 40 usable comments spanning the four categories, discuss disagreements, and revise the rubric before finalizing the dataset. Every usable example will receive one final label; unresolved examples will remain outside training until reviewed, with their number reported rather than silently discarded.

## Data collection and annotation

Start with the two r/soccer comment exports, which contain 100 comments each, and their combined text in the merged working file. Recover comment IDs, thread IDs, and permalinks from the JSON so examples remain traceable and train/test separation can be enforced; do not assume that two comments with identical short text are duplicates unless their IDs match.

The initial target is **200 usable annotated comments, approximately 50 per label**. The existing 200 rows include deleted entries, so additional collection may be needed even before balancing. Sample across match threads, transfer discussions, tactical or player discussions, and club-governance news, using multiple teams and threads rather than concentrating on one controversy. Collect public comments from r/soccer through an available permitted export or Reddit access method, retaining only metadata needed for annotation and evaluation.

After annotating the first 200 usable examples, inspect class counts. If a label has fewer than 40 examples, collect more comments from relevant thread types until it approaches 50, while applying the same definitions and recording that this extra sampling is targeted. Do not invent examples, duplicate rare examples, or relabel comments to reach a quota. If a label remains scarce, report its support and collect more data before claiming reliable performance for it; merge labels only if annotation shows their distinction is unclear, not merely because one is rare.

Keep a randomly sampled audit set representative of the community's natural label distribution, since balancing the development data changes class proportions. Split by thread, keeping replies and duplicate comment IDs in the same partition, to prevent shared context from leaking across training and evaluation. Aim for a roughly 60/20/20 train/validation/test split with all labels represented where thread grouping permits; use validation data for model choices and leave the test set untouched until final evaluation.

## AI Tool Plan

AI assistance will focus on testing the rubric, suggesting annotations, and examining errors; this project does not require AI-generated implementation code. The tool for these activities will be the current Codex assistant, with the displayed model identifier, date, prompts, and outputs recorded for disclosure. Human review remains necessary: the boundary decisions below are AI proposals, not independent human validation.

### Label stress-testing

Give the assistant the four definitions, assignment order, and hard-case descriptions above, then ask: "Generate eight fictional r/soccer-style comments at the boundary between two labels. Include humorous explanations, short judgments, circular reasons, rhetorical and genuine questions, and context-dependent sarcasm. Identify the competing labels and explain which rule resolves each case; flag cases that require missing context."

The following eight synthetic examples were generated during planning. They are rubric probes, not collected community posts, and will not enter training or evaluation data.

| Synthetic boundary comment | Competing labels | Proposed decision and boundary |
|---|---|---|
| "World-class defending: three unmarked runners in the box. That's why we conceded." | Reasoned contribution / banter | Reasoned contribution: the stated defensive failure explains the goal despite sarcasm. |
| "He's bad because he's terrible." | Reasoned contribution / unsupported take | Unsupported take: circular wording supplies no explanatory reason. |
| "Spain are brilliant." | Unsupported take / conversational exchange | Unsupported take: a standalone quality judgment, even though short. |
| "What a goal!" | Conversational exchange / unsupported take | Conversational exchange: immediate event-specific celebration without a broader evaluation. |
| "Anyone know why the manager benched him?" | Conversational exchange / unsupported take | Conversational exchange: a genuine request with no independent accusation. |
| "Why start him? He's lost possession five times and their chances all came down his side." | Reasoned contribution / conversational exchange | Reasoned contribution: observations support an implied criticism; the question is rhetorical. |
| "Our striker has scored 400 goals in my dreams." | Banter / reasoned contribution | Banter: the number belongs to a fictional punchline rather than factual support. |
| "Great substitution, boss." | Banter / unsupported take | Requires context review: it could be literal praise or sarcasm; text alone cannot determine intent. |

These probes exposed three weaknesses in the earlier rules: factual match observations overlapped with reactions, circular explanations could pass as reasoning, and joke statistics could pass as evidence. The stress-test refinements in the assignment rules resolve these now. Before full annotation, I will personally review these decisions and a fresh batch of 5–10 probes, revising any remaining ambiguous boundary; genuinely missing context will be tracked rather than treated as a definition failure that can always be solved from text alone.

### Annotation assistance

I plan to use Codex to pre-label development examples in batches of 20 after the rubric is finalized, requesting one proposed label, a short rationale quoting the relevant wording, and a context-needed flag for each comment. I will review every proposal against the original text, retain or correct it, and record a human final label; an AI suggestion alone will not count as a completed annotation. The independently annotated agreement subset and validation/test gold labels will be labeled by humans without viewing AI suggestions, avoiding anchoring and keeping evaluation independent of pre-labeling.

Track `comment_id`, `ai_prelabel_used`, `ai_tool`, `ai_model`, `ai_run_date`, `prompt_version`, `rubric_version`, `ai_label`, `ai_rationale`, `context_needed`, `human_final_label`, and `human_changed_label`. Keep the original AI prediction even when corrected, and retain prompts and responses in an annotation log. The final AI usage section will disclose the number and proportion of examples pre-labeled, the tool/model used, and how many proposals were changed.

### Current annotation artifact and AI usage

#### Synthetic augmentation update

I decided to augment the two smaller categories synthetically. `labeled_comments.csv` now contains 265 rows with only `text` and `label`: the original 185 rows, unchanged, followed by 40 synthetic banter comments and 40 synthetic conversational exchanges. Current totals are reasoned contribution 80 (30.19%), unsupported take 48 (18.11%), banter 62 (23.40%), and conversational exchange 75 (28.30%). This is a more balanced distribution, not an exactly equal one; synthetic rows do not count toward the target for 200 collected real examples.

I had Codex generate these additions from the rubric: banter focuses on recognizable fictional jokes without explanatory support, while conversational exchanges contain genuine questions, acknowledgments, or immediate match reactions. All synthetic additions were checked for unique text and valid labels, but have not been independently human-reviewed. I am keeping a provenance record of every generated text, label, data-row number, and SHA-256 text hash, so provenance survives reordering while the main CSV stays at two columns. The row ranges below carry the same information. Data rows 186–225 are synthetic banter and 226–265 are synthetic conversational exchanges, excluding the header.

Before retraining, split the real examples by thread, then add synthetic examples only to training. Validation and test sets must contain real, independently human-labeled comments. Randomly splitting the augmented CSV would test partly on generated language and could overstate real-community performance; use the synthetic row ranges to exclude generated rows from evaluation. Compare the same held-out real set with and without augmentation to measure whether minority-class F1 improves. No trainer is present in the repository, and no retraining result has been produced. Small class counts alone do not prove why a model failed, and these additions do not guarantee improved generalization.

The following paragraphs describe the initial, pre-augmentation artifact and remain as an annotation history.

I had Codex create one complete, unsplit `labeled_comments.csv` from the merged comment file, which contained 185 cleaned comments rather than the original 200 rows. All 185 labels (100%) are AI-generated drafts, not human-reviewed gold annotations; `annotation_status` explicitly records this, and `ai_prelabel_used` is true for every row. The required columns are `text`, `label`, and `notes`; comment IDs, thread IDs, and source filenames are also retained. The conversational-exchange category includes two automated bot messages, identified in notes for removal before human-discourse modeling.

The draft distribution is 80 reasoned contributions, 48 unsupported takes, 22 banter, and 35 conversational exchanges. Ninety rows contain difficult-case or interpretation notes, including provisional decisions needing parent-context review. No human corrections or independent agreement measurements have been recorded, and the 200-usable-example target has not been met. This chat is the prompt/output record for the initial labeling pass; the tool was Codex, while an exact underlying model identifier was not independently recorded.

The post-labeling imbalance check on all 185 CSV rows gives reasoned contribution 43.24%, unsupported take 25.95%, conversational exchange 18.92%, and banter 11.89%. No label exceeds the specified 70% threshold, so this check does not require additional collection before proceeding. These are draft-label counts and must be recalculated after human review or exclusions; the earlier collection targets remain separate requirements.

This full-file drafting pass differs from the earlier plan to pre-label only development batches. To preserve independent evaluation, humans must annotate the agreement subset and any validation/test examples without seeing this file's AI labels or notes. AI drafts must not be presented as independently annotated evaluation gold, and unresolved context-dependent decisions must be reviewed before those examples are used.

### Failure analysis

After evaluation, give Codex an error table containing comment IDs, comment text, human gold labels, predicted labels, and available context flags, asking it to group errors by pattern and cite the supporting IDs. Look for sarcasm read literally, profanity mistaken for an unsupported take, circular or implicit reasoning, brief reactions mistaken for judgments, missing parent context, and recurring team/thread vocabulary. Ask for counterexamples and distinguish possible annotation errors from model errors instead of accepting every proposed pattern.

I will verify each proposed pattern by rereading every cited error against the frozen rubric, counting how many errors support it, and checking at least three correct predictions with similar wording where available. Record verified counts and representative examples, and mark small groups as tentative rather than general conclusions. AI explanations are hypotheses, not evidence about the model's internal reasoning; the evaluation write-up will include only patterns supported by inspected examples. Any change prompted by final test errors will be evaluated on new untouched data, as specified in the final decision procedure.

## Evaluation metrics

- **Macro F1 is the primary metric:** it weights all four labels equally and combines precision and recall, preventing a common category such as conversational exchange from hiding poor performance on banter or reasoned contributions.
- **Per-label precision, recall, F1, and support:** precision for reasoned contributions measures whether an analysis-focused filter actually retrieves useful information or reasoning, while recall measures how much of that content it misses. Banter precision and recall reveal whether the model confuses jokes with literal opinions. Report these measures for every label rather than just the average.
- **Confusion matrix:** inspect reasoned contribution versus unsupported take and banter versus unsupported take, where the rubric predicts the hardest boundaries. Use these errors to identify whether the model misses explanatory reasons or misreads sarcasm.
- **Accuracy and majority-class baseline:** report overall accuracy as a secondary measure and compare with always predicting the most common training label; also compare macro F1 with that baseline. Accuracy alone could reward ignoring less frequent labels.
- **Annotation agreement:** report raw agreement and Cohen's kappa on the independently annotated subset before adjudication. This measures whether the label definitions are reproducible; inconsistent human labels would limit the meaning of model scores.

Report sample counts and uncertainty alongside scores, including confidence intervals where the sample permits. With only about 40 held-out examples under the initial split, per-label estimates will be unstable, so the initial evaluation is a prototype result. Evaluate context-dependent examples separately and ensure any context supplied to the model matches what will be available in the intended tool.

## Definition of success

I'll call this a useful prototype if it reaches macro F1 of 0.75, F1 of at least 0.60 on every label, and beats the majority-class baseline by at least 0.10 macro F1 on the same held-out test set. Alongside that, at least 90% of the comments I sample should fit the rubric, and my independently annotated comments should agree at least 80% of the time. If either data check fails I'd rather fix the task than tune the model. These are the bars I'm setting now, before I've measured anything.

To run it as an optional browsing filter in the community I want more: macro F1 of 0.80, F1 of 0.70 on every label, and precision of at least 0.85 on reasoned contributions, since that's the filter people would actually lean on. I'd check that on a fresh audit set of at least 400 usable comments from threads I haven't touched, with at least 50 of each label. If a label comes up short I'll keep sampling naturally rather than topping it up deliberately, which would help me diagnose but wouldn't tell me it's ready. I also want the lower bound of a 95% confidence interval on macro F1 to sit at or above 0.75, bootstrapped over whole threads, 2,000 replicates, seed written down.

### How I'll decide

Everything that applies has to pass. A good average doesn't excuse a failing label. I'll compare unrounded scores, compute macro F1 over the same four labels every time, and give a label zero F1 if it never gets predicted. If any label has fewer than five gold examples in the test set, that's insufficient evidence rather than a pass.

Rubric coverage means the number of comments I gave a final label, divided by every nonempty, nondeleted comment I sampled, with unresolved cases still in the denominator. Agreement gets measured before any discussion, on at least 40 independently labeled comments, and anything unresolved counts as a disagreement. I'll freeze the rubric, the model, and any confidence threshold on training and validation data before touching the test set, then score every test comment including the ones the model is unsure about. If test results end up driving changes, the next decision needs a fresh test set.

At the end I want one table: each requirement, what it measured, and pass, fail, or insufficient evidence. Prototype success means every prototype gate passes. Deployment means every deployment gate passes, plus the same coverage and agreement checks on the audit data. Missing data counts as not ready.

One last thing, and it matters more to me than the numbers. People should be able to see the original comment and correct the prediction. Low-confidence predictions stay visible with some uncertainty marker instead of being quietly filtered out. The point is helping someone find the kind of discussion they're after, with an error rate they can live with. These labels don't justify removing comments or ranking fans by the quality of their opinions.
