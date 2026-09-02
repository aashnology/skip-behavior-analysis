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
3. Enrich tracks with audio features via exact and fuzzy artist+track matching (rapidfuzz)
4. EDA: skip rate by session position, time of day, track features
5. Feature engineering: lag features, rolling skip rate, session progress, time encoding, leakage-safe user historical skip rate
6. Modeling: baseline logistic regression → XGBoost, with a session-aware, time-based validation split
7. Interpretability: SHAP, translated into engagement-design insights

## Findings

### Data enrichment
Only ~7% of listening events (1,340,422 of 19,150,868 plays; 20,230 unique tracks) could be
enriched with Spotify audio features via exact artist+track matching. This is a substantially
lower coverage rate than initially estimated — see Limitations for why. This low coverage directly
shapes several downstream findings below.

### Session position & time of day
- Skip rate climbs steadily the longer a session runs, with a sharp jump for sessions of 20+
  tracks. Most sessions are short; long "marathon" sessions are a distinct, more skip-prone
  listening mode. (An earlier version of this analysis grouped by `session_id` alone; since
  `session_id` resets per user, this incorrectly merged unrelated sessions from different users —
  caught via a sanity check, fixed by grouping on `user_id` + `session_id` together.)
- Skip rate is highest overnight (hours 0–1 UTC) and lowest hours 4–6, roughly a 2x spread.
  (Timestamps are UTC, not per-user local time, so this reflects an aggregate mix of listener
  timezones.)

### Skip cascade
Skipping is not an independent event. Skip rate is roughly 50–60x higher immediately after a
skipped track (16% early in a session, 41% late in a session) than after a non-skipped track
(0.3–0.7%). This effect holds independently of session position, and the two effects compound — a
track played late in a session, right after a skip, has over 100x the skip probability of one
played early with no recent skip.

### Audio features
On the ~7% of plays with a matched Spotify audio-features record, skip rate roughly doubles from
the least to the most popular quartile of tracks — counterintuitive, and plausibly a familiarity
effect (well-known tracks get skipped more readily to reach something new) rather than a quality
signal. Energy and loudness show the opposite pattern. This finding is limited to the matched
subset and may not generalize.

### Modeling

| Model | Features | AUC |
|---|---|---|
| Logistic Regression | Behavioral only | 0.921 |
| Logistic Regression | Behavioral + audio | 0.9215 |
| XGBoost | Behavioral only | **0.962** |

Adding sparse audio features gave no meaningful improvement over behavioral features alone
(AUC +0.0002) — consistent with their ~7% coverage. XGBoost meaningfully outperformed logistic
regression: at a comparable recall (~78%), its precision (0.41) is more than double logistic
regression's (0.20). At a high-confidence threshold (0.95), XGBoost correctly flags a skip about
half the time it predicts one, while still catching 74% of actual skips.

Train/test split is time-based (train through mid-Oct 2008, test after) rather than user-based,
since one feature (`user_historical_skip_rate`) is a causally-expanding average that only stays
meaningful under temporal separation. Skip rate rises across this period (a real, gradual trend —
verified via monthly breakdown — not a split artifact), so absolute skip-rate figures differ
between train and test by design.

### Interpretability (SHAP)
- `rolling_skip_rate_5` dominates the model (92% of feature importance). Its relationship to skip
  probability is a step function, not linear: the first recent skip already produces most of the
  risk signal; additional recent skips add only modest further risk.
- `session_progress` has negligible effect across most of a session, except a sharp drop at the
  very last track — likely reflecting that the session is ending, not that the track was judged
  and skipped.
- A user's overall historical skip rate matters, but far less than recent in-session behavior, and
  doesn't meaningfully interact with it.
- Time of day, day of week, and genre change are essentially noise once behavioral-momentum
  features are included.

## Engagement-design implications

1. **React to the first skip, not a pattern.** Since one recent skip already carries most of the
   risk signal, a recommendation system doesn't need to wait for repeated skipping before
   adjusting — real-time re-ranking triggered by a single skip is well-supported by the data.
2. **Intervene early in a session.** Skip risk compounds with session length, so the
   highest-leverage moment to adjust recommendations is in the first few tracks, before a session
   enters its higher-skip-rate later stretch.
3. **Weight in-session behavior over long-run user profile.** A user's overall skip tendency is a
   comparatively weak, non-interacting signal — what's happening in *this* session matters more.
4. **Audio content signals underperform behavioral signals for skip prediction**, at least at this
   level of match coverage — suggesting engineering effort is better spent on session-context
   modeling than on richer audio metadata, for this specific use case.

## Limitations

- Last.fm-1K is 2009-era data — a proxy for modern streaming behavior, not Spotify's actual
  production logs. Findings describe a reproducible pattern on public data, not a claim about how
  Spotify's current algorithm behaves.
- Skip labels are inferred (timestamp-gap heuristic), not ground truth.
- **Feature coverage is low (~7% of plays) and this is a real, verified finding, not a bug**: the
  Spotify reference dataset (Kaggle) is a fixed stratified sample (114 genres × 1,000 tracks each),
  not an exhaustive catalog — most Last.fm tracks, including songs by very popular artists, are
  simply absent from it. An earlier pipeline version reported 52.3% coverage; this was incorrect,
  caused by a duplicate-row bug (the Spotify file lists each track once per genre tag, which
  inflated merge match counts). The bug was found, diagnosed, and fixed — the 7% figure is
  verified two independent ways (pandas merge and raw set-intersection).
- Fuzzy artist/track matching (rapidfuzz) was built and tested but excluded from the final
  pipeline: it recovered only ~0.4% additional play coverage, not enough to justify the added
  complexity and failure surface for a portfolio-scale project.
- Audio-feature-derived model features (deltas, rolling averages) inherit the ~7% match-rate
  limitation and are sparse enough (1.8%–30% non-null) that their standalone predictive value is
  limited, as confirmed by the modeling ablation above.
- Timestamps are UTC; time-of-day findings reflect an aggregate mix of listener timezones, not
  local time-of-day behavior.

## Stack

Python (pandas, DuckDB), rapidfuzz, scikit-learn, XGBoost, SHAP.

## Setup

pip install -r requirements.txt

Run notebooks in numeric order (`01` through `06`). Each notebook loads its inputs from
`data/processed/`; earlier notebooks must be run first to generate these intermediate files.
