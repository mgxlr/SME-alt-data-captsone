# SME Alternative Data Scoring & Segmentation

A Google Data Analytics Certificate capstone testing whether alternative data — online reviews and firmographic signals — can supplement traditional financial statements in assessing small and medium enterprises (SMEs). Built on the Yelp Open Dataset, scoped to Philadelphia (14,560 businesses, 967,489 reviews, 2005–2022). Scaled down from a bank-grade alt-data architecture — see `docs/write-up.md` for what that means in practice.

**Headline finding:** across three independent analyses, businesses with worse reviews were *less* likely to have closed, not more — the opposite of what "bad reviews predict business failure" would suggest.

For the full write-up — methodology, findings, limitations, GDPR/ethics, and what a bank-scale version would need — see **[`docs/write-up.md`](docs/write-up.md)**.

## Repo structure

```
sme-alt-data-capstone/
├── README.md                        this file
├── requirements.txt                 pinned Python dependencies
├── data/
│   ├── Yelp JSON/                   raw dataset (not included — see Setup)
│   └── *.csv                        cleaned/derived outputs, written by the notebooks
├── notebooks/                       numbered analysis notebooks, run in order (see below)
├── outputs/                         exported chart PNGs from 05_visualization.ipynb
└── docs/
    ├── write-up.md                  the full capstone write-up
    ├── capstone-plan-of-action.md   phase-by-phase project plan and status
    └── build-log-difficulties.md    methodology decisions and difficulties log
```

## Setup

1. **Python environment** (3.10+ recommended):
   ```
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```
2. **NLTK sentiment data** (one-time download, not a pip package):
   ```
   python3 -c "import nltk; nltk.download('vader_lexicon')"
   ```
3. **Raw data**: download the [Yelp Open Dataset](https://www.yelp.com/dataset) and place `yelp_academic_dataset_business.json` and `yelp_academic_dataset_review.json` in `data/Yelp JSON/`. Not included in this repo — the review file alone is ~5.3GB.

## Running the notebooks

Run in this order — each depends on outputs from the one(s) before it:

| # | Notebook | Depends on | Produces |
|---|---|---|---|
| 1 | `01_data_cleaning.ipynb` | raw Yelp JSON | `business_clean.csv`, `reviews_clean.csv` |
| 2 | `02_nlp_signals.ipynb` | `reviews_clean.csv` | `reviews_with_sentiment.csv` |
| 3 | `03_feature_engineering.ipynb` | `business_clean.csv`, `reviews_with_sentiment.csv` | `business_features.csv` |
| 4 | `04a_segmentation.ipynb`, `04b_risk_classification.ipynb`, `04c_growth_trend.ipynb` | `business_features.csv` (04c also needs `reviews_with_sentiment.csv`) | `business_segments.csv`, `business_growth_trend.csv` — independent of each other, any order |
| 5 | `05_visualization.ipynb` | outputs of steps 3–4 | chart PNGs in `outputs/` |

Open with `jupyter notebook` or `jupyter lab`, or execute headlessly, e.g.:
```
jupyter nbconvert --to notebook --execute --inplace notebooks/01_data_cleaning.ipynb
```
Notebook 1 takes several minutes (it scans the full 5.3GB review file in chunks); the rest run in under a minute each.
