# SME Alternative Data Scoring & Segmentation

**Can alternative data — online reviews and firmographic signals — supplement traditional financial statements in assessing small and medium enterprises (SMEs)?**

A Google Data Analytics Certificate capstone, scaled down from a UniCredit-style bank alt-data architecture. Built on the Yelp Open Dataset (public, licensed, no scraping), scoped to a single city (Philadelphia, 14,560 businesses, 967,489 reviews spanning 2005–2022).

---

## Executive summary

Across three independent analyses (a simple group comparison, an unsupervised clustering cross-tab, and a supervised model's explainability values), review sentiment turned out to predict business closure in the opposite direction from what "alternative data improves risk assessment" would naively suggest: businesses with worse reviews were less likely to have closed.

The risk model still achieves moderate predictive power (ROC-AUC 0.74) using review volume, category, and text-pattern features. bad reviews does not predict business failure. This dataset does not say that either.

Three models were built, each independent of the others:
- **Segmentation** (k-means, k=3): named, interpretable business segments from review behavior and category mix.
- **Risk classification** (logistic regression): predicts a *proxy* for business closure — not real credit default — with documented, leakage-safe methodology.
- **Growth trend** (linear regression on review volume): a *proxy* for demand/attention growth — not real revenue growth — with an explicit minimum-data threshold to avoid reporting noise as signal.

Every model output in this write-up is named as what it actually is: a proxy, not a real financial outcome. That distinction is treated as a first-class part of the deliverable, not a footnote.

---

## Architecture overview

The original context this project is scaled down from is a bank-grade alt-data credit architecture: live web/social scraping, licensed vendor data feeds, continuous ingestion pipelines, and production model-serving infrastructure feeding real lending decisions.

This capstone builds the same conceptual pipeline: raw signal → structured features → models → decisions — at a small scope t with one public dataset:


Yelp Open Dataset (JSON, public)
        │
        ▼
01_data_cleaning.ipynb        — filter to Philadelphia, deduplicate, chunked read of the 5.3GB review file
        │
        ▼
02_nlp_signals.ipynb          — VADER sentiment + structural text features (per review)
        │
        ▼
03_feature_engineering.ipynb  — aggregate to one row per business, merge structured + NLP features,
        │                        define the risk_proxy_closed label
        ▼
04a_segmentation.ipynb ─┐
04b_risk_classification.ipynb ─┼─  three independent models
04c_growth_trend.ipynb ─┘
        │
        ▼
05_visualization.ipynb        — presentation-ready charts, exported to outputs/


Every stage after cleaning is independent of the others where the plan allows it. Each writes its own output file rather than mutating a shared table.

What's explicitly out of scope, per the original plan: real bank-grade infrastructure, live social/web scraping, licensed data vendor integration, and production deployment. 
---

## Data source and build-vs-buy reasoning

**Source**: the Yelp Open Dataset (`yelp_academic_dataset_business.json`, `yelp_academic_dataset_review.json`). Public, licensed for this use, downloaded once as a static snapshot. No scraping, no live API calls, no vendor contracts.
---

## Methodology

**Data cleaning:** The review file was too large to load into memory at once, so I read it in chunks and filtered down to Philadelphia businesses as I went, rather than loading everything first.

**NLP approach:** Before deciding how to extract signal from the review text, I looked at a sample of the reviews themselves. They're short, first-person accounts of a single visit, not the kind of press or business-announcement text the original architecture's event-detection approach (hiring signals, expansion announcements) was designed for. So I scoped that part out and used sentiment scoring plus basic text features instead, since that actually matches the kind of text I have.

**Feature engineering:** When I built the risk label from the "open/closed" field, I checked the resulting feature table and noticed the original open/closed column was still sitting in it alongside the label I'd derived from it. Since one is just the inverse of the other, leaving it in would let any model "predict" risk by cheating off that column instead of learning anything from the actual data. I removed it before training anything, and added a check that will flag it automatically if it ever ends up back in the table.

**Growth proxy:** There's no revenue or transaction data in this dataset, so I defined growth as a proxy instead. The trend in how many reviews a business gets per month over time. When I looked at the data, I noticed some businesses had all their reviews bunched into a very short window, which would make a "trend" meaningless. So I only calculated the trend for businesses with a reasonable spread of active months, and left the rest blank.

**Segmentation:** I tested a range of cluster counts to find natural groupings in the business data. None of them separated cleanly with the separation scores staying weak across the whole range I tried. Since the higher options weren't performing better, I went with three clusters, since that's the number I can actually name and explain clearly.

**Headline finding:** Working through the results, I noticed something I didn't expect: businesses that had closed actually had *better* average reviews than ones still open. I checked this three separate ways: a simple average comparison, a cross-check against the clusters, and the trained model's own feature importance. All three agreed. That consistency told me it's a real pattern, not a fluke of any one method. I don't know the exact cause from this data alone, so I'm reporting it as an open question rather than guessing at an explanation.

**Risk model:** I trained a logistic regression model, adjusting for the imbalance between open and closed businesses, and it performs meaningfully better than chance, though far from perfectly. I noticed its raw accuracy actually looks worse than simply guessing "still open" every time. I focused on recall instead, which is the metric that matters for this question.

**Growth results:** Among the businesses with enough history to measure, slightly more were trending down than up, with most showing only modest movement either way.

**Limitations:** Every risk and growth number here is a proxy, not a confirmed financial outcome. I don't have real default or revenue data to check against. The analysis is also limited to one city and one snapshot in time, and the segments, while real, aren't well separated from each other.

**Ethics:** I only used public review data, didn't scrape anything, and nothing in this project tries to identify individual reviewers. I chose a model whose reasoning can be explained after the fact, and none of this is built to make an automated decision on its own. 

**At bank scale:** The biggest gap I'd want to close is real outcome data. Actual defaults or revenue, instead of the proxies I had to use here. Everything else I'd want to add (more cities, more data sources, formal fairness testing) matters, but comes second to that.