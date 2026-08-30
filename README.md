# Micro-Churn: What Predicts an Early Track Skip?

> Companion piece to [Subscription Churn & Cohort Retention](#) — that project studied churn at the
> subscription level (months); this one studies the same disengagement instinct at the session level
> (seconds).

## Question

What listening-session and track-level factors predict an early skip (a track abandoned within its
first 30 seconds), and what does that reveal about engagement patterns and recommendation quality?

## Data

- **Last.fm Dataset – 1K Users** (Celma, 2010) — ~19M timestamped listening events, 992 users.
  Source: http://ocelma.net/MusicRecommendationDataset/lastfm-1K.html
  License: non-commercial research use, per Last.fm terms.
- **Spotify audio features snapshot** (Kaggle, pre-2024 API deprecation) — track-level audio
  features (danceability, energy, valence, tempo, acousticness, etc.), joined by fuzzy artist/track
  match. Source: [link TBD once downloaded]

Neither dataset is redistributed in this repo — see `data/raw/README.md` for download instructions.

## Method (summary — full detail in notebooks)

1. Reconstruct listening sessions from raw event logs (20-min inactivity gap = new session)
2. Label skips: next track starts within 30s of current track (methodology per [citation])
3. Enrich tracks with audio features via fuzzy artist+track matching (rapidfuzz)
4. EDA: skip rate by session position, time of day, track features
5. Feature engineering: lag features, rolling skip rate, session-position, track features
6. Modeling: baseline logistic regression → XGBoost, with session-aware validation splits
7. Interpretability: SHAP to translate model into engagement-design insights

## Findings

*(to be filled in as the analysis progresses)*

## Limitations

- Last.fm-1K is 2009-era data — a proxy for modern streaming behavior, not Spotify's actual
  production logs. Findings describe a reproducible pattern on public data, not a claim about how
  Spotify's current algorithm behaves.
- Skip labels are inferred (timestamp-gap heuristic), not ground truth.
- Artist/track fuzzy matching introduces some feature coverage gaps — match quality is documented
  in `notebooks/03_feature_enrichment.ipynb`.

## Repo structure

```
data/
  raw/          # downloaded datasets (not committed — see data/raw/README.md)
  processed/    # cleaned/joined intermediate outputs (not committed, regenerate from notebooks)
notebooks/      # numbered, one per pipeline stage
src/            # reusable functions imported by notebooks
reports/
  figures/      # exported plots for the writeup
  tableau/      # Tableau Public dashboard file
```

## Stack

Python (pandas, DuckDB), rapidfuzz, scikit-learn, XGBoost, SHAP, Tableau Public.
