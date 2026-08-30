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

*(full behavioral findings to come after EDA/modeling — this section documents the data
enrichment outcome, which is itself a key finding)*

Only ~7% of listening events (1,340,422 of 19,150,868 plays; 20,230 unique tracks) could be
enriched with Spotify audio features via exact artist+track matching. This is a substantially
lower coverage rate than initially estimated — see Limitations for why.

## Limitations

- Last.fm-1K is 2009-era data — a proxy for modern streaming behavior, not Spotify's actual
  production logs. Findings describe a reproducible pattern on public data, not a claim about how
  Spotify's current algorithm behaves.
- Skip labels are inferred (timestamp-gap heuristic), not ground truth.
- **Feature coverage is low (~7% of plays) and this is a real, verified finding, not a bug**:
  the Spotify reference dataset (Kaggle) is a fixed stratified sample (114 genres × 1,000 tracks
  each), not an exhaustive catalog — most Last.fm tracks, including songs by very popular artists,
  are simply absent from it. An earlier pipeline version reported 52.3% coverage; this was
  incorrect, caused by a duplicate-row bug (the Spotify file lists each track once per genre tag,
  which inflated merge match counts). The bug was found, diagnosed, and fixed — the 7% figure is
  verified two independent ways (pandas merge and raw set-intersection).
- Fuzzy artist/track matching (rapidfuzz) was built and tested but excluded from the final
  pipeline: it recovered only ~0.4% additional play coverage, not enough to justify the added
  complexity and failure surface for a portfolio-scale project.

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
