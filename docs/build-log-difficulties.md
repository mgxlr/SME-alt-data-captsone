# Capstone Build Log — Design Decisions & Difficulties

Running log of real methodology decisions, snags, and how they were resolved.
Useful both as a personal reference and as raw material for the Phase 7
write-up (methodology honesty, limitations section, "what I'd do differently"
section).

---

## 1. `is_open` data leakage in the risk-proxy feature table
**What happened:** `risk_proxy_closed` (the Phase 5b target) was derived
directly from `is_open`, but `is_open` itself was left in
`business_features.csv` as a regular column. Since
`is_open == 1 - risk_proxy_closed` for every row, using "all columns except
the target" as the feature set would have handed a classification model a
trivial 100%-accuracy shortcut with zero real signal.
**Resolution:** Caught before Phase 5b was built. Added an explicit leakage
warning (markdown + executable assertion) directly after the label definition
in the Phase 4 notebook, documented in CLAUDE.md, and switched the Phase 5b
plan from an exclusion-list feature set ("all columns except target") to an
explicit allowlist (named columns only).
**Takeaway:** This is a real, worth-keeping example for the write-up's
methodology section — demonstrates active leakage awareness, not just
"the model worked."

## 2. No real growth/revenue data — growth-trend proxy needed explicit definition
**What happened:** The original plan's Phase 5c called for a growth model, but
the dataset has no financial or revenue data at all — "growth" had to be
redefined as a proxy before any modeling could start.
**Resolution:** Defined `growth_proxy_review_volume_trend` (slope of monthly
review-count regression over each business's active review period), with an
explicit naming convention (`growth_proxy_*`, never `growth_*`) and a written
caveat that review-volume trend reflects attention/engagement, not confirmed
revenue growth.
**Takeaway:** Name proxies explicitly as proxies, both in code and in the
write-up, rather than letting a variable name imply more than the data
supports.

## 3. Growth-trend proxy: unstable slope for businesses with little review history
**What happened:** After defining `growth_proxy_review_volume_trend` (#2), a
minimum-review-count threshold was initially proposed to guard against noisy
trend slopes — but Yelp's own data already floors every listed business at
≥5 reviews, so a raw count threshold wouldn't exclude anything meaningful.
Investigating further, the real driver of instability turned out to be how
many distinct *months* had review activity, not total review count — e.g. 20
reviews in one week (1 active month) produces an unstable slope, while 10
reviews spread over 3 years doesn't, even though the second business has
fewer total reviews.
**Resolution:** Set a minimum of 6 distinct active months required to compute
`growth_proxy_review_volume_trend`. Businesses below that threshold get
`NaN` for the trend plus a companion column `growth_proxy_active_months`
(so the reason is visible, not silently dropped) — this excludes ~11.4% of
businesses (1,656 of 14,560) rather than the ~40% a stricter 12-month bar
would have cut, while still ruling out mathematically meaningless 1-2-point
"trend lines." No secondary total-review-count floor was added on top, since
active-month count already captures the real risk and a second filter would
mostly just trim further without addressing anything new.
**Takeaway:** Explicit missingness with a visible reason beats a silently
dropped row or a fabricated-looking number. Also worth stating plainly in the
write-up: even above the 6-month floor, a slope fit through a small number of
points is a coarse estimate, not a statistically robust one — the proxy
should be presented as directional, not precise.

## 4. Segmentation: choosing k despite weak, ambiguous silhouette scores
**What happened:** Elbow and silhouette diagnostics for k-means didn't give a
clean answer. Inertia decreased smoothly with no dramatic elbow, and
silhouette scores were low across the entire tested range (0.12-0.19,
nowhere near 1) — meaning segments are real but not sharply separated, not a
sign the clustering failed. Three local silhouette peaks appeared: k=3
(0.164), k=8 (0.164, tied), and k=10 (0.190, but at the very edge of the
tested range 2-10, so it's unclear whether the score would keep rising past
that point or genuinely peak there).
**Resolution:** Chose **k=3**. Reasoning: (1) all three candidates scored in
the same weak silhouette range, so neither k=8 nor k=10 was buying
meaningfully tighter separation, just more segments to explain for the same
underlying signal; (2) k=10 sitting at the edge of the tested range meant it
couldn't be confirmed as a true peak rather than a still-rising trend, and
chasing it further wasn't worth the time cost for a likely-marginal gain;
(3) k=3 aligns with the project's standing "simple + explainable beats
fancier + unexplained" convention — three plain-language segments are
realistically nameable and defensible in a write-up, where eight would need
real justification for each one's distinctness.
**Takeaway:** Weak silhouette scores are informative, not disqualifying — the
right response was to state the caveat explicitly (mixed continuous +
categorical features naturally produce softer separation than pure numeric
clustering) rather than either hide it or over-invest time chasing a
marginally higher score. This caveat needs to make it into the Phase 7
write-up directly, not stay buried in the notebook.

## 5. Segment-vs-closure cross-check confirms sentiment doesn't track risk — and shapes the Phase 5b model choice
**What happened:** Cross-tabbing the three Phase 5a segments against
`risk_proxy_closed` produced a counterintuitive result: Segment 0
("Struggling/Mixed-Experience," worst sentiment, 42% negative reviews) has
the *lowest* closure rate (20.3%), while Segment 2 ("High-Traffic Bars &
Nightlife," decent sentiment) has the *highest* (38.1%). This independently
confirms an earlier Phase 4 finding (closed businesses had higher average
sentiment than open ones, 0.60 vs 0.53) — two separate pieces of evidence
that `risk_proxy_closed` does not track review sentiment the way you'd
naively assume. Plausible explanation: bar/nightlife venues may close for
structural reasons (leases, licensing, ownership churn) unrelated to review
quality, while businesses with a loyal niche or low local competition may
survive despite consistently poor reviews.
**Resolution:** Finding recorded before proceeding. It directly informed the
Phase 5b model choice: when offered XGBoost vs. logistic regression, chose
**logistic regression** — both for matching the project's simple/explainable
convention (interpretable coefficients, no dependency on SHAP alone to
explain it) and because a more complex model is unlikely to meaningfully
outperform a simple one on a signal already shown to be weak/counterintuitive.
The reasoning (not just avoiding an install) was recorded in the notebook.
**Takeaway:** This is a strong, honest result for the write-up: sentiment
being a *weak* predictor of business closure is itself a legitimate finding,
not a failure of the pipeline. Worth stating directly rather than implying
the model "should" have found a strong sentiment-risk link — the data doesn't
support that link, and reporting that plainly is more credible than forcing
a stronger story than the evidence shows.

## 6. Phase 5 complete — all three models built, cross-validated against each other
**What happened:** All three independent Phase 5 tasks finished:
- **5a Segmentation**: 3 named segments (k=3), low silhouette scores (0.12-0.19)
  documented as a real limitation rather than glossed over.
- **5b Risk classification**: logistic regression, leakage-safe (scaler fit on
  train split only, `is_open` excluded), ROC-AUC 0.74. Class imbalance
  (72.4%/27.6%) handled via class weighting, confirmed with full precision/
  recall/accuracy/confusion-matrix numbers rather than a single metric.
  SHAP (LinearExplainer) identified `sentiment_compound_mean` as the 2nd-
  strongest predictor — and higher sentiment predicts *higher* closure risk,
  confirming the #5 finding a third independent way.
- **5c Growth trend**: `business_growth_trend.csv` — 12,904 businesses with a
  valid trend, 1,656 correctly NaN'd per the #3 rule; boundary manually
  verified (all excluded businesses had 1-5 active months, none at 6+).
**Resolution:** All three outputs cross-checked against each other and
against known totals (12,904 + 1,656 = 14,560, matching total business count
exactly) before moving to Phase 6.
**Takeaway:** The headline result of the whole project is the sentiment-vs-
risk reversal — confirmed three independent ways (Phase 4 group means, Phase
5a segment cross-tab, Phase 5b SHAP). This should lead the write-up's
findings section, not sit as a footnote; it's a more interesting and more
defensible result than "the model achieved X accuracy."

---

*Add new entries here as they come up — each one is a legitimate methodology
note, not just a bug log.*
