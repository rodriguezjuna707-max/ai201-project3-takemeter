# TakeMeter — Planning

A fine-tuned text classifier that sorts r/DemonSlayer discourse by quality: is a post *reasoning from the work*, *asserting a vibe*, or *feeling a moment*?

---

## 1. Community

**Community:** r/DemonSlayer (with r/KimetsuNoYaiba as a secondary source for the same fandom), the subreddit for *Kimetsu no Yaiba* — a completed manga and ongoing anime adaptation.

**Why this community.** The brief asks for discourse that is active, text-heavy, and varied in quality. Demon Slayer delivers that variance for a specific structural reason: the manga is *finished*. There are no mysteries left to predict, so substantive posts aren't theories about what comes next — they're arguments about a known work. That produces a clean three-way spread of post types. Some posts make real cases (about animation choices, powerscaling, the divisive ending); some assert confident rankings with no support; and a large slice are pure emotional reactions, because the series is built on devastating death scenes and sympathetic-villain backstories. A regular in this community instantly recognizes the difference between "here's why Akaza's defeat is consistent with his feats" and "Akaza's mid, cope." That recognizable difference is exactly the signal the classifier should learn.

**Why it's a good classification task.** The hard boundary (evidence-backed argument vs. confident assertion) is genuinely subtle — the same topic, powerscaling, can produce a post on either side of the line. A model has to learn *reasoning structure*, not just topic keywords, which is what makes the task non-trivial and the failure modes interesting.

---

## 2. Labels

Three labels, sorted by reasoning quality rather than topic.

### `analysis`
A structured argument about the story, characters, adaptation, or a powerscaling matchup, backed by specific, verifiable evidence — cited scenes, manga panels, animation details, or established feats.

- *"Ufotable rendering Hinokami Kagura with that hand-drawn fire-over-3D-camera technique is why the Rui fight hits so much harder than the earlier water-breathing animation. The studio escalates the visual language exactly when Tanjiro unlocks a new style, so the animation is doing narrative work, not just looking good."*
- *"Akaza losing to Tanjiro and Giyu isn't a powerscaling inconsistency. His regeneration was tied to his refusal to die, and the fight shows him starting to remember Koyuki right as he stops healing. The defeat is psychological and his backstory sets it up — the feats line up once you track when his regen actually fails."*

### `hot_take`
A bold, confident opinion — a ranking, an "overrated/underrated," or a powerscaling verdict — asserted without supporting evidence. The claim might be correct; the post states it rather than argues it.

- *"Rengoku is overrated and it's only because he died early. Mid Hashira at best, y'all are just emotional about it."*
- *"Hot take: the Entertainment District arc is the only genuinely good arc and everything after is a downgrade. Not close."*

### `reaction`
An immediate emotional response to a specific moment — a death, a backstory, an episode — expressing feeling rather than making an argument.

- *"Just watched the Rengoku scene. I am not okay. SET YOUR HEART ABLAZE has been ringing in my head for an hour and I've cried twice."*
- *"Nezuko backstory flashback broke me. Wasn't ready. Why does this show keep doing this on a weeknight."*

---

## 3. Hard edge cases

### Primary edge case: `analysis` vs. `hot_take` in powerscaling
The most common genuine ambiguity is a powerscaling post that **cites real feats but uses them in motivated, one-sided reasoning.**

> *"Tanjiro low-key beats Sanemi. He learned Hinokami Kagura on the fly against Rui and adapted to Daki's whole fighting style mid-fight — that kind of in-combat growth is exactly what wins against a power-brawler like Sanemi."*

The feats are specific and verifiable (the Rui fight, the Daki adaptation), which smells like `analysis`. But the conclusion only stacks favorable evidence and never engages a single one of Sanemi's feats — the signature of a `hot_take` dressed up as analysis.

**Decision rule.** Strip the opinion framing and ask not just *is there evidence* but *does the post engage the counter-evidence?* If it weighs both sides — the opponent's feats too — and reasons to a verdict → `analysis`. If it only assembles favorable feats to defend a predetermined conclusion → `hot_take`. The post above never mentions what Sanemi can do, so the real feats are functioning as decoration → **`hot_take`**.

### Secondary edge case: `reaction` vs. `hot_take`
A reaction often carries a mild judgment ("that was peak"). **Rule:** if the post's *primary purpose* is venting emotion in the moment → `reaction`; if it's asserting a debatable, rankable claim → `hot_take`. Same event, opposite labels: "the reincarnation epilogue had me sobbing" is `reaction`; "the reincarnation epilogue was a mistake, it deflates the stakes" is `analysis` if it reasons or `hot_take` if it's a bare complaint.

### Handling during annotation
When a post gives me genuine pause, I apply the relevant rule above, record the post text + the two candidate labels + the decision + the rule that decided it in the `notes` column of the CSV, and carry the 3+ hardest into the README's difficult-examples section. A post that resists both rules is a signal to tighten a definition before continuing, not to invent an `other` bucket.

> Note: the example posts above are constructed to define the boundaries. Per Milestone 3, I'll read 30–40 real r/DemonSlayer posts during collection and replace the edge-case examples with genuine ones found in the wild — a real powerscaling post is the most reliable place to find a true borderline.

---

## 4. Data collection plan

**Source.** Public posts and top-level comments from r/DemonSlayer and r/KimetsuNoYaiba. Public content only, no authenticated or private channels. Collected manually by copy-paste into a spreadsheet to stay close to the data (the brief estimates ~1–2 hours for 200), reading each one rather than scraping in bulk.

**Volume and target distribution.** At least 200 examples. Target a roughly balanced split so no class dominates and the model can't win by always predicting the majority:

| Label | Target share | Approx. count |
|---|---|---|
| `analysis` | ~33% | ~66 |
| `hot_take` | ~33% | ~66 |
| `reaction` | ~34% | ~68 |

**Where each label is easiest to find.** `reaction` concentrates in episode-discussion and just-finished threads; `hot_take` in ranking, "unpopular opinion," and powerscaling threads; `analysis` in longer top-level posts and adaptation/animation discussion. I'll deliberately pull from a mix of thread types so the dataset isn't skewed by where I looked.

**If a label is underrepresented after 200.** The brief flags >70% in one label as an imbalance problem; my own target is at least ~20% per class. `analysis` is the likeliest to come up short, since it's rarer in casual threads. If so, I'll do a targeted second pass into the thread types above (longer top-level posts, animation/ending discussion) and collect more of the thin class specifically — rather than down-sampling the others and losing total volume.

**File format.** One complete labeled CSV (not pre-split) with columns `text`, `label`, and `notes`. The notebook handles the 70/15/15 train/validation/test split automatically.

---

## 5. Evaluation metrics

**Accuracy alone is not enough.** On a 3-class task, accuracy hides *which* distinction the model failed to learn, and it rewards a model that nails the two easy classes while never learning the hard boundary. If `analysis` is even slightly the smallest class, a model that collapses it into `hot_take` could still post a healthy-looking accuracy.

**Headline metric: macro-averaged F1.** Macro-F1 averages the per-class F1 scores with equal weight regardless of class frequency, so a class the model quietly fails on can't be masked by the larger classes. This is the right summary number for a task where the *hard* class may also be the *smaller* one.

**Per-class precision, recall, and F1.** Reported for every label, for both the fine-tuned model and the zero-shot baseline. Per-class F1 is what tells me whether each distinction is being learned (the brief's "all F1 ≥ 0.70 = learning all distinctions" heuristic).

**Confusion matrix.** The diagnostic that matters most for this specific task, because my whole design predicts the errors will cluster in one cell: true `analysis` predicted as `hot_take`. A large off-diagonal count at (row=`analysis`, col=`hot_take`) is a directional, actionable signal that the model learned topic keywords (powerscaling, the ending) but not reasoning structure. I'll write the matrix out as a markdown table in the README, not just commit the PNG.

**Why these and not others.** The task is a quality-of-reasoning judgment where one boundary is intrinsically harder than the other two; macro-F1 + per-class breakdown + a confusion matrix together answer "did it learn the hard boundary or just the easy ones?" — which raw accuracy cannot.

---

## 6. Definition of success

**Beat the baseline meaningfully.** The fine-tuned model must clearly exceed the zero-shot Llama-3.3-70b baseline on macro-F1 — not by a point or two of noise. If fine-tuning barely beats the baseline, that's a finding to report (labels too easy or too noisy), not a success.

**Specific thresholds:**
- **Overall accuracy ≥ 0.70** on the held-out test set (well above the 0.33 random-guess floor for 3 classes).
- **Macro-F1 ≥ 0.70.**
- **Per-class F1 ≥ 0.65 for every label**, including `analysis` — this is the real bar. A model that hits high overall accuracy but has `analysis` F1 near 0.4 has failed at the one distinction the project is actually about, and I'd call that *not* good enough regardless of the headline number.
- **The `analysis`→`hot_take` confusion is the metric to watch.** Success means that off-diagonal cell is a minority of `analysis` predictions, not the majority.

**"Good enough" for deployment.** For a real community tool — say, surfacing high-effort posts or flagging low-effort hot takes — I'd want per-class F1 ≥ 0.75 on the hard boundary specifically, plus calibrated confidence (a 90%-confident prediction should be right meaningfully more often than a 60%-confident one), so the tool can defer to a human on low-confidence cases instead of mislabeling them. The thresholds above are the bar for *this project*; deployment would raise the bar on the `analysis`/`hot_take` boundary and add the calibration requirement.

---

## 7. AI Tool Plan

This is a data-and-judgment project, not an implementation project, so AI tools help in three specific places.

**Label stress-testing — doing this.** I gave an LLM my three definitions and the powerscaling edge case and asked it to generate borderline posts between `analysis` and `hot_take`. Posts I couldn't classify cleanly drove me to add the "does it engage the counter-evidence?" tiebreaker to the decision rule *before* annotating 200. I'll re-run this if I revise any definition.

**Annotation assistance — deciding explicitly: limited use, fully reviewed.** I may use an LLM to pre-label a batch using my exact `planning.md` definitions, but only as a first pass — I will read and correct every pre-assigned label, since skimming produces noisy training data. Any pre-labeled examples will be tracked (a flag in the `notes` column) and disclosed in the README's AI usage section. If pre-labeling proves unreliable on the hard boundary, I'll drop it and label everything by hand.

**Failure analysis — doing this.** After evaluation I'll paste the misclassified test examples into an LLM and ask it to surface patterns (a recurring label pair, post length, sarcasm, low-information posts), then verify each pattern myself by re-reading the examples before writing it up. The AI surfaces candidates; I confirm or discard them. Both what I keep and what I discard go in the evaluation report.

*This section will be updated before starting any stretch features.*
