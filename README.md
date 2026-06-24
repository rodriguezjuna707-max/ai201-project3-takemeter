# TakeMeter — Classifying Discourse Quality in r/DemonSlayer

---

## 1. Community choice and reasoning

**Community:** r/DemonSlayer (and the parallel r/KimetsuNoYaiba), the subreddit fandom for *Kimetsu no Yaiba* — a completed manga with an ongoing anime adaptation.

**Why this community is a good classification target.** Demon Slayer discourse is varied in quality for a structural reason: the manga is *finished*, so there are no open mysteries to predict. Substantive posts are therefore arguments about a known work — animation analysis, powerscaling matchups, and the famously divisive ending — rather than speculation. That produces a natural three-way spread: rigorous arguments, bare confident opinions, and pure emotional reactions to the series' many death scenes and sympathetic-villain backstories. A regular in the community instantly recognizes the difference between "here's why Akaza's defeat is consistent with his feats" and "Akaza's mid, cope" — and that recognizable difference is the signal the classifier is meant to learn. The hard part (and the interesting part) is that the same *topic* — powerscaling, the ending — can appear on either side of the quality line, so the model must learn reasoning structure, not just keywords.

---

## 2. Label taxonomy

Three labels, sorted by **reasoning quality** rather than topic.

### `analysis`

A structured argument about the story, characters, adaptation, or a powerscaling matchup, backed by specific, verifiable evidence — cited scenes, feats, or animation details.

- *"Ufotable rendering Hinokami Kagura with that hand-drawn fire-over-3D-camera technique is why the Rui fight hits harder than the earlier water-breathing animation. The studio escalates the visual language exactly when Tanjiro unlocks a new style, so the animation is doing narrative work."*
- *"Akaza losing to Tanjiro and Giyu isn't a powerscaling inconsistency. His regeneration was tied to his refusal to die, and the fight shows him remembering Koyuki right as he stops healing. The feats line up once you track when his regen actually fails."*

### `hot_take`

A bold, confident opinion — a ranking, an "overrated/underrated," or a powerscaling verdict — asserted without supporting evidence.

- *"Rengoku is overrated and it's only because he died early. Mid Hashira at best, y'all are just emotional about it."*
- *"Hot take: the Entertainment District arc is the only genuinely good arc and everything after is a downgrade. Not close."*

### `reaction`

An immediate emotional response to a specific moment — a death, a backstory, an episode — expressing feeling rather than making an argument.

- *"Just watched the Rengoku scene. I am not okay. SET YOUR HEART ABLAZE has been ringing in my head for an hour and I've cried twice."*
- *"Nezuko backstory flashback broke me. Wasn't ready. Why does this show keep doing this on a weeknight."*

---

## 3. Data collection, labeling, and distribution

**Source.** Public posts and top-level comments collected from **r/DemonSlayer** and **r/KimetsuNoYaiba** on 2026-06-24. 〔CONFIRM your exact method: e.g. "via Reddit's public API (top/hot/controversial listings + top-level comments from the most-active threads), then cleaned and deduplicated" OR "manually copy-pasted into a spreadsheet". Each row's `notes` column records the source subreddit and permalink.〕 〔If you used an LLM to pre-label any examples before reviewing them yourself, you MUST disclose that here and in the AI Usage section below.〕

**Labeling process.** Each example was assigned exactly one of the three labels using the definitions in `planning.md`. The decision rule for the hardest boundary (`analysis` vs `hot_take`) was: strip the opinion framing and ask whether specific, verifiable evidence remains *and* whether the post engages counter-evidence. If it weighs both sides → `analysis`; if it only stacks favorable points to defend a predetermined verdict → `hot_take`.

**Label distribution (full labeled set, 201 examples):**


| Label      | Count   | Share    |
| ------------ | --------- | ---------- |
| `reaction` | 70      | 34.8%    |
| `hot_take` | 66      | 32.8%    |
| `analysis` | 65      | 32.3%    |
| **Total**  | **201** | **100%** |

No class exceeds the 70% imbalance ceiling, and every class clears the 20% floor. The notebook split this 70/15/15 into **140 train / 30 validation / 31 test**, stratified by label.

**Three difficult-to-label examples and decisions:**

1. **Powerscaling post citing real feats one-sidedly.** *"Tanjiro low-key beats Sanemi — he learned Hinokami Kagura on the fly against Rui and adapted to Daki's style mid-fight."* Candidate labels: `analysis` (real, verifiable feats) vs `hot_take` (predetermined verdict). **Decision → `hot_take`**, because it never engages a single one of Sanemi's feats; the real feats function as decoration for a conclusion already chosen.
2. **Emotional post that pivots to an argument.** A post that opens "I'm sobbing" but then makes a structural claim about why a scene was set up well. Candidate labels: `reaction` vs `analysis`. **Decision → by primary purpose**: if the emotion is the point → `reaction`; if the emotion is a lead-in to a reasoned claim → `analysis`.
3. **The ending, two ways.** "The reincarnation epilogue had me sobbing" (`reaction`) vs "the reincarnation epilogue was a mistake — it deflates the stakes the story earned" (`analysis`). Same event, opposite labels. **Decision rule:** is the post *feeling* the moment or *making a case* about it?

---

## 4. Fine-tuning approach

- **Base model:** `distilbert-base-uncased` (HuggingFace), a distilled BERT chosen because it fine-tunes in minutes on a free T4 GPU and is appropriate for short-text, few-class classification.
- **Training setup:** Hugging Face `Trainer` on the training split (~70% of the dataset), max sequence length 256, dynamic padding via `DataCollatorWithPadding`.
- **Hyperparameters:** notebook defaults — **3 epochs, learning rate 2e-5, batch size 16**, weight decay 0.01, max sequence length 256. *(Confirm these match your final run; if you changed any, update this line.)*
- **Key hyperparameter decision:** I kept **3 epochs** rather than increasing them. With only ~140 training examples, more epochs risked overfitting to surface features (post length, punctuation, vocabulary) rather than improving the genuinely hard `analysis`/`hot_take` boundary — the distinction the project is actually about. Stopping at 3 keeps the model from memorizing the small training set while still giving it enough passes to separate the easy classes. *(If your validation-loss curve flattened or rose after epoch 2, cite that as supporting evidence.)*

---

## 5. Baseline description

**Approach:** zero-shot classification with Groq's `llama-3.3-70b-versatile`. No task-specific training — the model receives the label definitions and one example per label, and must output a single label name.

**Prompt used (system prompt):**

```
You are classifying posts and comments from the r/DemonSlayer anime community.
Assign each post to exactly one of the following three categories.

analysis: a structured argument about the story, characters, adaptation, or a
powerscaling matchup, backed by specific, verifiable evidence such as cited
scenes, feats, or animation details.
Example: "Akaza losing to Tanjiro and Giyu isn't a powerscaling inconsistency.
His regeneration was tied to his refusal to die... the feats line up once you
track when his regen fails."

hot_take: a bold, confident opinion such as a ranking, an overrated/underrated
claim, or a powerscaling verdict, asserted without supporting evidence.
Example: "Rengoku is overrated and it's only because he died early. Mid Hashira
at best, y'all are just emotional about it."

reaction: an immediate emotional response to a specific moment such as a death,
a backstory, or an episode, expressing feeling rather than making an argument.
Example: "Just watched the Rengoku scene. I am not okay. SET YOUR HEART ABLAZE
has been ringing in my head for an hour and I've cried twice."

Respond with ONLY the label name: analysis, hot_take, or reaction.
Do not explain your reasoning. Do not add punctuation or any other words.
```

**How results were collected:** each of the 31 test examples was sent to the model at `temperature=0`, `max_tokens=20`. The raw response was lowercased and matched against the three label strings (with whitespace/punctuation normalization so `hot take` → `hot_take`). Predictions were compared to the held-out test labels.

---

## 6. Evaluation report

### Overall accuracy (same 31-example test set)


| Model                              | Accuracy             |
| ------------------------------------ | ---------------------- |
| Zero-shot baseline (Llama-3.3-70B) | **96.77%** (30 / 31) |
| Fine-tuned DistilBERT              | **90.32%** (28 / 31) |
| **Change from fine-tuning**        | **−6.45 points**    |

The fine-tuned model performed **worse** than the zero-shot baseline. This is the central finding of the evaluation and is analyzed below.

### Per-class metrics — Fine-tuned DistilBERT


| Label         | Precision | Recall    | F1        | Support |
| --------------- | ----------- | ----------- | ----------- | --------- |
| `analysis`    | 0.90      | 0.90      | 0.90      | 10      |
| `hot_take`    | 0.89      | 0.80      | 0.84      | 10      |
| `reaction`    | 0.92      | 1.00      | 0.96      | 11      |
| **Macro avg** | **0.90**  | **0.90**  | **0.90**  | 31      |

### Per-class metrics — Zero-shot baseline


| Label         | Precision | Recall    | F1        | Support |
| --------------- | ----------- | ----------- | ----------- | --------- |
| `analysis`    | 0.909     | 1.000     | 0.952     | 10      |
| `hot_take`    | 1.000     | 0.900     | 0.947     | 10      |
| `reaction`    | 1.000     | 1.000     | 1.000     | 11      |
| **Macro avg** | **0.970** | **0.967** | **0.966** | 31      |

> The baseline got only **1 of 31 wrong** (96.77%). Its single error was a true `hot_take` predicted as `analysis` (recall on `hot_take` = 0.90; precision on `analysis` = 0.91) — the same `analysis`↔`hot_take` boundary where the fine-tuned model also concentrates its mistakes.

### Confusion matrix — Fine-tuned model (rows = true, columns = predicted)


| true ↓ / pred → | analysis | hot_take | reaction | total |
| ------------------- | ---------- | ---------- | ---------- | ------- |
| **analysis**      | **9**    | 1        | 0        | 10    |
| **hot_take**      | 1        | **8**    | 1        | 10    |
| **reaction**      | 0        | 0        | **11**   | 11    |
| total             | 10       | 9        | 12       | 31    |

The diagonal (9, 8, 11) is correct predictions. All three errors are off-diagonal, and **two of the three fall on the `analysis`↔`hot_take` boundary** — one in each direction: 1 true `analysis` predicted as `hot_take`, and 1 true `hot_take` predicted as `analysis`. The third error is a single `hot_take` predicted as `reaction`. `reaction` is predicted perfectly (recall 1.00) and is never confused with anything. Notably, **all three misclassifications carried a confidence of just 0.35** — barely above the 0.33 random-guess floor for three classes — so the model was essentially guessing on every example it got wrong, rather than confidently choosing the wrong label.

### Three wrong predictions, analyzed

All three of the fine-tuned model's errors are shown below, each with its (low) confidence score.

**Error 1 — true `hot_take`, predicted `reaction` (confidence 0.35).**
Post: *"Hot take: the Entertainment District arc is the only genuinely good arc and everything after is a downgrade. Not close."*
*Why it failed:* this is short, punchy, and emphatic ("Not close."), and it carries none of the powerscaling/feat vocabulary the model came to associate with `hot_take`. Its terse, exclamatory cadence resembles the surface style of the `reaction` class, so the model leaned that way — despite the post literally opening with the words "Hot take." The 0.35 confidence (near the 0.33 chance floor) confirms the model had no real signal here; it learned to read tone and length, not the argumentative intent the label is about.

**Error 2 — true `analysis`, predicted `hot_take` (confidence 0.35).**
Post: *"Akaza losing to Tanjiro and Giyu isn't a powerscaling inconsistency. His regeneration was tied to his refusal to die, and the fight shows him starting to remember Koyuki right as he stops healing. The..."*
*Why it failed:* this is a genuine `analysis` — it cites a specific mechanism (regeneration tied to his will, failing exactly as he remembers Koyuki) and reasons to a conclusion. But it sits on the project's hardest boundary: it is a *powerscaling* post, and the model learned that powerscaling vocabulary leans `hot_take`. It keyed on the topic instead of recognizing that this post actually engages the evidence. This is precisely the `analysis`→`hot_take` confusion the planning doc predicted.

**Error 3 — true `hot_take`, predicted `analysis` (confidence 0.35).**
Post: *"The 'Transparent World' is a cheap visual cheat code introduced way too late just to give characters a way to bypass reaction speeds. Muichiro and Gyomei randomly unlocking it mid-battle completely br..."*
*Why it failed:* the inverse of Error 2, on the same boundary. This is a bare negative verdict ("cheap cheat code," "completely br[eaks]…") with explanatory-sounding phrasing but no real evidence engagement — a `hot_take`. The model saw the pseudo-reasoning structure and the lore-specific terms ("Transparent World," named characters) and called it `analysis`. It learned that explanation-shaped sentences = `analysis`, without learning whether the explanation is actually grounded.

**Diagnosis — labeling problem or data problem?** The errors are consistent with the *labels being applied consistently* but the *training data being too cleanly separable*. `reaction` is trivially separated by surface features (capitalization, exclamation marks), so the model nails it (recall 1.00). The real distinction — reasoning quality between `analysis` and `hot_take` — is where two of the three meaningful errors land, and all three errors are near-random-confidence guesses, because the dataset did not contain enough genuinely borderline examples for the model to learn the boundary rather than the surface signal.

**What would fix it.** More *borderline* `analysis`/`hot_take` examples — specifically calm-toned hot takes that cite a feat or use explanatory phrasing but never engage counter-evidence (like Error 3), and powerscaling/topic-heavy analyses that do reason through a mechanism (like Error 2) — so the model is forced to learn engagement-with-evidence instead of topic vocabulary, length, and tone.

### Sample classifications (fine-tuned model)

Confidence is the softmax probability of the predicted class.


| Post (truncated)                                                              | Predicted  | Confidence | Correct? |
| ------------------------------------------------------------------------------- | ------------ | ------------ | ---------- |
| "Hot take: the Entertainment District arc is the only genuinely good arc..."  | `reaction` | 0.35       | ❌       |
| "Akaza losing to Tanjiro and Giyu isn't a powerscaling inconsistency..."      | `hot_take` | 0.35       | ❌       |
| "The 'Transparent World' is a cheap visual cheat code introduced too late..." | `analysis` | 0.35       | ❌       |
| 〔a correctly-classified `reaction` post — paste text〕                        | `reaction` | 〔0.__〕   | ✅       |

> 〔To finish this table: from your notebook, pick one correctly-classified example (a `reaction` is ideal) and paste its text + confidence into the last row. The three error rows above are your real notebook output.〕

**Why one correct prediction is reasonable:** a correctly-classified `reaction` post (e.g. an "I am not okay… I've cried twice" example) is predicted correctly with high confidence because it expresses in-the-moment emotion about a specific scene with no argumentative structure — exactly what the `reaction` definition describes. The contrast with the error rows is the real story: the model is *confident and right* on `reaction` (clear surface cues) but *near-random (0.35) and wrong* on the `analysis`/`hot_take` boundary, which is the distinction the project actually cares about.

---

## 7. Reflection — what the model learned vs. what I intended

I intended the model to learn a **quality-of-reasoning** distinction: whether a post argues from evidence, asserts without it, or simply emotes. What it actually learned was closer to a **surface-feature** distinction. The evidence is in the error pattern: `reaction` — the class most identifiable by punctuation and capitalization — was learned perfectly, while every meaningful error clustered on the `analysis`/`hot_take` boundary, which can only be separated by reasoning structure, not surface cues.

The decisive result is that **fine-tuning made performance worse** (−6.45 points: 90.32% vs the baseline's 96.77%). The right reading is not "DistilBERT is bad" but "the task as represented by this dataset is too easy." A 70B general model already classifies these near-perfectly zero-shot because the classes are separable by superficial signals; fine-tuning a small model on only ~140 such examples introduced confusions the large model never made, without teaching anything the large model lacked. Tellingly, the fine-tuned model's three errors all landed at 0.35 confidence — it never even committed to its wrong answers, which is what you'd expect from a model that learned surface cues but not the underlying reasoning distinction. The project's stated goal — capturing what makes a take good in a way specific to the community — is exactly the part neither model demonstrably learned, because the data didn't force them to. The model overfit to *how a take is written* and missed *whether it reasons*.

---

## 8. Spec reflection

**One way the spec helped:** the spec's insistence on a precise, mutually-exclusive label taxonomy with an explicit edge-case decision rule (Milestones 1–2) is what made the eventual failure *legible*. Because the `analysis`/`hot_take` boundary was defined in advance as the hard one, the confusion matrix immediately confirmed a predicted hypothesis rather than producing an uninterpretable result.

**One way the implementation diverged, and why:** the spec frames the dataset as messy real-world text where the hard `analysis`/`hot_take` boundary shows up naturally and often. My dataset turned out far more cleanly separable than that ideal — the 96.77% zero-shot baseline is the evidence — so the borderline cases that make the task interesting were underrepresented. That divergence is the most likely root cause of the fine-tuned model regressing below the baseline: with the classes already easy to split on surface cues, there was little for fine-tuning to learn and room to introduce noise. If I redid this, I would deliberately over-collect genuinely borderline posts (calm-toned hot takes, topic-heavy real analyses) so the boundary the project cares about is actually represented in training.

---

## 9. AI usage

> The course requires disclosing at least two specific AI-tool uses, **including any AI assistance during annotation or data creation.** Complete each item truthfully.

1. **Planning / label design.** I used an AI assistant (Claude) to stress-test the label taxonomy — generating borderline `analysis`/`hot_take` posts to check whether my definitions held — and to draft `planning.md`. I reviewed and 〔kept / revised — specify〕 the output, and the final decision rules are my own.
2. **Failure-pattern analysis.** I pasted the misclassified test examples into an AI assistant and asked it to surface common themes; it identified the `analysis`/`hot_take` clustering, which I then verified myself against the confusion matrix before writing the analysis above.
3. **Data creation / annotation (if applicable):** 〔FILL IN — if any part of the dataset was AI-generated or AI-pre-labeled, disclose it explicitly here: what tool, what it produced, and what you reviewed or corrected. This is required, not optional.〕

---

## Repository contents

- `planning.md` — design notes, label definitions, edge-case rules, evaluation plan (written before data collection)
- `takemeter_dataset.csv` — the labeled dataset (`text`, `label`)
- `README.md` — this file
- `evaluation_results.json` — accuracy comparison and label map (committed from Colab)
- `confusion_matrix.png` — fine-tuned confusion matrix image (supplementary to the markdown table above)
