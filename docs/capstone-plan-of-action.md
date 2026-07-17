# Capstone Plan of Action: SME Alternative Data Scoring & Segmentation

**Purpose:** Reference document for building a scaled-down version of the UniCredit-style alt-data architecture as a Google Data Analytics Certificate capstone. Written for someone doing their first end-to-end data project.

---

## How to use this document

Each phase below is a self-contained milestone. We'll go through them **one at a time, in order** — you don't need to understand phase 5 to start phase 1. When we sit down to work, just tell me which phase we're on (or say "next phase") and I'll walk you through it step by step, explaining any new tool or concept as it comes up.

The Google Data Analytics (GDA) certificate teaches a 6-step data lifecycle: **Ask → Prepare → Process → Analyze → Share → Act**. I've mapped our phases onto that lifecycle so the project structure matches what your certificate taught you and what a grader/reviewer would expect to see.

---

## Phase 0 — Setup (GDA: Ask)
**Goal:** Environment ready, question defined, scope locked.
- [x] Confirm final project question: *"Can alternative data (reviews, web/social signals) improve or supplement traditional SME risk/growth assessment?"*
- [x] Set up Python environment (Jupyter Notebook — I'll help install/configure)
- [x] Create project folder structure (data/, notebooks/, outputs/)
- [x] Write a 3-5 sentence project brief (business problem, who'd use this, why alt-data)

## Phase 1 — Choose & Acquire Data (GDA: Prepare)
**Goal:** One real, public, low-risk dataset in hand.
- [x] Decide dataset: Yelp Open Dataset (reviews) vs. Kaggle SME dataset vs. small hand-collected Google Reviews sample (~20-50 businesses)
- [x] Download and do a first look (row count, columns, obvious gaps)
- [x] Document data source + license/usage terms (this matters for the GDPR/ethics write-up later)

## Phase 2 — Clean & Structure the Data (GDA: Process)
**Goal:** A clean table where every row is correctly attributed — this is "ETL," not NLP yet.
- [x] Handle missing values, duplicates, data types
- [x] Entity resolution: make sure each review/record maps to one company
- [x] Output: one clean structured table, text fields still raw (untouched)

Done — filtered to Philadelphia (largest city in the dataset). Outputs:
`data/business_clean.csv` (14,560 rows) and `data/reviews_clean.csv`
(967,489 rows, dated 2005–2022). Verified against the raw files directly,
including a byte-for-byte spot check of a sampled review against the raw
5.3GB review JSON.

## Phase 3 — Extract Signals from Text (GDA: Process → Analyze bridge)
**Goal:** Turn raw review/social text into structured numeric/categorical features.
- [x] Sentiment scoring (pretrained model or simple LLM call on the small dataset)
- [x] Basic NER / event tagging if relevant (e.g. spaCy) — only if your dataset supports it
- [x] Sanity-check outputs on a handful of rows by hand

Done — VADER sentiment (via `nltk`) + structural text features (word count,
exclamation/question marks, caps ratio) added to `data/reviews_with_sentiment.csv`
(967,489 rows). Sanity check: mean `sentiment_compound` rises monotonically
with star rating (1★: -0.20 → 5★: 0.87), confirming the scores are sensible.

Decision: skipped NER/event tagging for this dataset. Yelp reviews describe
a single customer visit, not business events (hires, expansions, launches) —
the kind of signal NER would meaningfully extract for a press-feed-based
system like the original UniCredit architecture this project scales down
from. Plan: VADER sentiment (via `nltk`, already installed — lexicon-based,
no training needed, well-suited to short informal text) plus basic
structural text features (review length, punctuation, etc.). This will be
noted explicitly in the Phase 7 write-up as a scoped-out limitation driven
by the data source, not a shortcut.

## Phase 4 — Feature Engineering (GDA: Process → Analyze)
**Goal:** One merged feature table combining structured data + NLP-derived features.
- [x] Merge firmographic/structured data with sentiment/event features
- [x] Normalize/standardize numeric features (needed later for clustering)
- [x] Decide on proxy labels if no real default/outcome data exists (flag this as a documented limitation)

Done — `data/business_features.csv` (14,560 rows, 23 columns): business data
merged with review-level sentiment/text features aggregated to business
level (mean/std sentiment, pct negative/positive, structural text means),
plus `category_count`. Proxy risk label `risk_proxy_closed` = 1 if
`is_open` == 0 (72.4% open / 27.6% closed) — explicitly documented as a
proxy, not a real default label, in the notebook and here.

Standardization: deliberately **not** baked into the saved file. K-means
(5a) needs it, XGBoost (5b) doesn't, and fitting a scaler before the 5b
train/test split would leak test information into training rows — so
scaling is deferred to each Phase 5 notebook individually. Demonstrated
inline in the notebook (not saved) to confirm scaling actually changes the
feature distributions as expected.

**Notable finding for the write-up:** closed businesses have slightly
*higher* average sentiment (0.60) than open ones (0.53) — the opposite of
the naive expectation. Real evidence the `is_open` proxy is noisy (closure
often reflects rent/relocation/retirement, not service quality), not a bug.

## Phase 5 — Modeling: Three Use Cases (GDA: Analyze)
Build these independently — none depends on the others being "done" first.

**5a. Segmentation (unsupervised, no LLM/NLP needed here)**
- [x] K-means clustering
- [x] Choose k via elbow method and/or silhouette score
- [x] Name the resulting segments in plain language

Done — `04a_segmentation.ipynb` outputs `data/business_segments.csv`
(14,560 rows: business_id, name, segment, segment_name). Features:
10 numeric (sentiment/text stats, excluding `stars`/`review_count` as
near-duplicates of features already included) + one-hot top-15 categories,
all standardized. Chose k=3 (local silhouette peak, tied with k=8 but
simpler to name; k=10 scored higher but was the untested edge of the
search range). **Caveat for write-up**: silhouette scores were low across
all tested k (0.12-0.19) — segments are real tendencies, not sharply
separated clusters; Philadelphia businesses sit on more of a continuum
than in crisp distinct groups.

Segments: **Struggling / Mixed-Experience Businesses** (28.5%, lowest
sentiment, 42% negative reviews), **Steady Mainstream Favorites** (62.3%,
majority, highest/most consistent sentiment, restaurant/food-heavy),
**High-Traffic Bars & Nightlife** (9.2%, ~3x review volume, 99%
nightlife/96% bars). Note for Phase 5b/6: Segment 0's poor sentiment
profile is an independent cross-check worth revisiting against
`risk_proxy_closed` — it emerged with no visibility into that label.

**Cross-check done (before building 5b):** closure rate by segment is
Segment 0 (worst sentiment) 20.3% (842/4,149), Segment 1 (best sentiment)
29.5% (2,671/9,068), Segment 2 (bars/nightlife) 38.1% (512/1,343) —
**the opposite** of what worse sentiment would predict. This reinforces
the Phase 4 finding that closed businesses have *higher* average sentiment
than open ones. Together these are two independent pieces of evidence that
review sentiment doesn't straightforwardly track `risk_proxy_closed` in
this dataset — a real, honest finding to report, and a sign that
sentiment/text features may turn out to be weak predictors in the Phase 5b
classifier. That would itself be a legitimate result, not a failure.

**5b. Credit/risk proxy classification**
- [x] XGBoost or logistic regression (start simple, upgrade if time allows)
- [x] Train/test split, baseline evaluation (accuracy, precision/recall, or ROC-AUC)
- [x] SHAP values for explainability (this maps to the GDPR Art. 22 requirement in the original case)

Done — `04b_risk_classification.ipynb`. Logistic regression
(`class_weight='balanced'`), 80/20 stratified split, scaler fit on train
only (leakage-safe). `is_open` explicitly excluded from features (perfect
predictor of the target by construction — see Phase 4 warning). Results:
accuracy 0.67, ROC-AUC 0.74, recall 0.72 / precision 0.44 on the "closed"
class — clearly beats the naive baseline (which gets 0.72 raw accuracy by
never predicting closure, 0.00 recall). Confusion matrix: 576/805 actual
closures caught (229 missed), 738 false alarms on the 2,107 that stayed
open. Confirmed via a side-by-side unweighted comparison model that the
0.72 recall comes specifically from `class_weight='balanced'` (no
resampling used) — without it, recall drops to 0.25 while accuracy rises
to 0.76, the same misleading-accuracy trap demonstrated directly. SHAP via
`LinearExplainer` for exact, fast values.

**Key finding — sentiment confirmed 3x as a risk-increasing feature, not
risk-reducing.** `sentiment_compound_mean` is the model's 2nd-strongest
predictor, and higher sentiment predicts *higher* closure risk. This
matches the Phase 4 group-mean finding and the Phase 5a segment cross-tab
— three independent methods agreeing this is real, not noise. Other
drivers make intuitive sense: `cat_restaurants` (highest risk driver,
matches real-world food-service failure rates), `review_count_actual` and
`category_count` reduce risk (more established/diversified = more
stable), `cat_health_and_medical` reduces risk, `cat_shopping`/`cat_food`/
`cat_nightlife` increase it. **Write-up must present the sentiment finding
as "matters in the opposite direction than expected," not "doesn't
matter"** — plausible explanations (nostalgic closure-announcement
reviews, survivorship bias) are floated in the notebook but not resolved
by this dataset; frame as an open question.

**5c. Growth/trend proxy (only if your data has a time dimension)**
- [x] Simple trend/regression on the growth-proxy metric
- [x] If no real time series exists, document this as a scoped-out limitation rather than forcing it

Done — `04c_growth_trend.ipynb` outputs `data/business_growth_trend.csv`
(14,560 rows: business_id, growth_proxy_active_months,
growth_proxy_review_volume_trend). 12,904 businesses (88.6%) got a valid
trend; 1,656 (11.4%, all with 1-5 active months, confirming the threshold
boundary is exactly right) got NaN as designed. Slope fit via
`scipy.stats.linregress` on a *dense* monthly grid (gap months filled
with 0 review count) — not the sparse active-months-only table, which
would have distorted slopes by treating gap months as adjacent.
Sanity-checked by hand: plotted the steepest-rising, steepest-falling, and
near-zero businesses against their fitted lines, confirmed the eyeball
trend matches the computed slope. Distribution: mean slope -0.023,
median -0.0005 (most businesses roughly flat, slight downward skew
overall).

Note: `reviews_clean.csv` does have a real time dimension (`date`, 2005–2022),
so 5c is in scope.

**Design decided (not yet built).** No revenue/foot-traffic data exists in
this dataset, so — same treatment as `risk_proxy_closed` — the growth
target is explicitly a proxy, named accordingly:

- **Target**: `growth_proxy_review_volume_trend` = slope of a simple linear
  regression of monthly review count against time, per business, fit over
  that business's own active period (`reviews_with_sentiment.csv` `date`
  field). Review volume over time stands in for customer-attention/demand
  growth — not revenue growth, which this dataset cannot observe.
- **Minimum-data guard**: only computed for businesses with **≥6 distinct
  active months** (calendar months with at least one review). Below that,
  a monthly trend line is fit through too few points to mean anything —
  11.4% of businesses (1,656 / 14,560) fall under this floor. Those get
  `NaN` for the trend, plus a companion column `growth_proxy_active_months`
  recording why, rather than being silently dropped (keeps the one-row-per-
  business grain intact for the segmentation/risk models) or given a
  fabricated number.
- **Threshold chosen at 6 months, not 12**: 12 months would null out ~40%
  of businesses — too much data loss for what's already the lowest-priority
  of the three Phase 5 tasks. 6 months excludes only the mathematically
  degenerate cases (including some businesses with literally 1 active
  month, where no trend can be fit at all) while keeping the sample largely
  intact.
- **No secondary total-review-count floor.** A business with exactly 6
  reviews spread across 6 months is still a legitimate (if noisy) 6-point
  trend — an extra count-based filter would just trim further without
  addressing the actual cause of instability (few time points), so it was
  left out as unneeded complexity.
- **Write-up caveat (do not skip)**: even at the 6-month floor, a slope
  fit through ~6 points is a coarse estimate, not a statistically solid
  one. Phase 7 must say this explicitly — `growth_proxy_review_volume_trend`
  should never be presented as more reliable than it is, especially for
  businesses near the minimum threshold.

## Phase 6 — Visualize Results (GDA: Share)
- [x] Cluster plots (segmentation)
- [x] Feature importance / SHAP plot (credit model)
- [x] Trend chart (growth model, if applicable)
- [x] Clean, labeled, presentation-ready charts (matplotlib/seaborn/plotly)

Done — `05_visualization.ipynb`, 5 charts exported to `outputs/`:
1. `segment_size_and_closure_rate.png` — segment sizes + closure rate by
   segment (the key 5a/5b cross-validated finding)
2. `risk_shap_feature_importance.png` — top 10 SHAP features, diverging
   color by risk direction
3. `risk_roc_curve.png` — full ROC curve (AUC 0.74), not built in 5b
4. `sentiment_by_closure_status.png` — the headline finding (box plot,
   means 0.53 open vs 0.60 closed)
5. `growth_trend_distribution.png` — 56% declining / 44% growing review
   volume among eligible businesses

Used the `dataviz` skill's validated colorblind-safe reference palette
(fixed categorical order blue/aqua/yellow for segments; status green/red
for open/closed; diverging blue/red for SHAP direction) rather than
matplotlib defaults. Risk model (SHAP values, ROC curve) is re-derived
from scratch in this notebook with identical code/random_state rather
than saved/loaded from 5b — confirmed reproducible (AUC matched exactly:
0.7427981852859589 both times).

Caught and fixed during review: an early version of the growth-trend
histogram used `.clip()` to handle outliers, which piled every outlier
into one fake bin exactly at the percentile boundary — looked like a
real cluster of businesses but was a charting artifact. Fixed by
filtering (excluding) outliers from the view instead, with the excluded
count stated in the caption rather than hidden.

## Phase 7 — Write-Up (GDA: Share → Act)
**Goal:** The narrative deliverable — as important as the code for a portfolio piece.
- [x] Architecture overview (mirroring this context doc, scaled to what you actually built)
- [x] Build-vs-buy reasoning (why you used public data, not a live scraper)
- [x] GDPR/ethics section (data minimization, explainability, human-in-the-loop, limitations of proxy labels)
- [x] Key findings in plain language for a non-technical reader
- [x] Honest "what I'd do differently at bank scale" closing section

Done — `docs/write-up.md`. Leads with the sentiment-reversal finding as the
project's headline result (adapted from `docs/build-log-difficulties.md`
#5/#6), not a model-accuracy number. Methodology section adapts the
leakage catch (#1), growth-proxy definition (#2), 6-month threshold
reasoning (#3), and k=3 segmentation decision (#4) from the build log
directly. Every cited number was re-verified against the actual data
files before finalizing (segment sizes/closure rates, sentiment means,
growth trend split all matched exactly).

## Phase 8 — Final Polish & Presentation (GDA: Act)
- [x] Clean up notebook(s) so they run top-to-bottom without errors
- [x] README for the repo/folder
- [ ] Optional: short slide deck or one-pager summarizing the project for a portfolio/LinkedIn (skipped per instruction)

Done — fresh restart-and-run-all confirmed on all 7 notebooks in dependency
order (01 → 02 → 03 → 04a → 04b → 04c → 05), zero failures. All data
outputs and chart PNGs re-verified byte-consistent with prior runs after
the fresh execution. `README.md` added at repo root (project summary,
headline finding, repo structure, setup + run-order instructions, links
to `docs/write-up.md` without duplicating it). `requirements.txt` added
and verified by installing into a clean, isolated virtualenv from
scratch — confirmed every import used across all notebooks resolves.
No hardcoded/machine-specific paths found in any notebook (grepped).

Note: one chart (`segment_size_and_closure_rate.png`) initially showed a
stale timestamp after the first sequential background run despite that
run reporting success — investigated directly (checked cell execution
counts, confirmed no error output, isolated by deleting the file and
re-running just that notebook), traced to a filesystem-visibility quirk
in that specific background task rather than a defect in the notebook
code. Re-verified clean on a direct re-run; content confirmed unchanged
and correct.

---

## Explicitly out of scope (per project context doc)
Real bank-grade infrastructure, live scraping at scale, licensed vendor data, production deployment. Don't let scope creep pull us back toward these.

## Where we are now
**Phase 8 — done (slide deck skipped per instruction). The capstone is complete: all 8 phases finished, all notebooks verified to run cleanly from a fresh clone, README and requirements.txt in place.**

---

*Update this checklist as we go — just ask me to mark items done or adjust phases, and I'll keep this file current.*
