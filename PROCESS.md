# Process Log

This file tracks the real working process behind this project — including the wrong turns,
because the reasoning behind a correction is more useful (to me, in an interview, and to anyone
reading this) than a clean-looking pipeline that hides how it actually got built.

## Session 1 — Data audit

Before writing any real logic, I checked whether the two raw files actually matched their
documented schema, and whether a direct ID join was even possible.

- Confirmed `artist_id`/`track_id` in the Last.fm file use MusicBrainz IDs, while the Spotify
  file uses Spotify's own ID system — two completely different ID spaces. A direct ID join was
  never going to work, regardless of missingness.
- Checked null rates: `artist_name`/`track_name` are 100% filled in the Last.fm file, which is
  what made a text-based join (rather than ID-based) the correct call from the start.

## Session 2 — Session reconstruction & skip labeling

Built session boundaries (20-minute inactivity gap = new session) and skip labels (next track
starts within 30 seconds) using SQL window functions (`LAG`/`LEAD`) over the full ~19.15M-row
event log. Skip rate came out to 0.8% — a heavily imbalanced target, which changes what metrics
matter later (precision/recall/PR-AUC, not accuracy).

## Session 3 — Feature enrichment: where the real debugging happened

This was the session that took the longest, and the one with the most worth documenting.

**Initial (wrong) result:** an early version of the exact-match pipeline reported a 52.3% match
rate between Last.fm tracks and the Spotify features file. This looked plausible at the time and
I moved forward planning fuzzy-matching around it.

**What actually happened:** while building the final full-dataset join, the row count came back
at 29,645,224 — 10.5 million rows *more* than the original 19,150,868-row sessionized dataset.
That's the kind of error that's easy to miss if you only check summary statistics and never
sanity-check row counts against a known baseline.

**Root cause:** the Kaggle Spotify features file lists the same track multiple times — once per
genre tag it's associated with (e.g., Radiohead's "Creep" appears separately under alt-rock,
alternative, indie, and rock). Merging on `(artist, track)` against a file with duplicate keys
multiplies matching rows, silently inflating both the row count *and* the reported match rate.

**How I verified the fix, rather than just trusting a new number:** after deduplicating the
features file (keeping the highest-popularity row per track) and rebuilding the match, the
reported rate dropped to ~1%. A number changing that much deserved independent verification, not
just acceptance — so I checked it two ways:
1. A fresh pandas `.merge()` on the deduplicated, verified-correct data
2. A raw Python `set` intersection of `(artist, track)` tuples — a method with no dependency on
   pandas' merge internals at all

Both landed in the same range (0.6%–1.3%), confirming the low number was real, not a new bug.
The original 52.3% had been wrong the entire time.

**Scope decision — dropping fuzzy matching:** I had already built a two-stage fuzzy-matching
function (rapidfuzz — match artist name first, then track name within that artist's catalog) as
a way to recover additional coverage beyond exact matching. Once corrected, it recovered only
about 0.4% additional play coverage. Given that gain, I made the call to exclude it from the
final pipeline — the added complexity (a slower, harder-to-explain matching stage, more failure
surface) wasn't earning its keep for the coverage it bought. Final enrichment is exact-match
only: ~7% of plays (1,340,422 of 19,150,868), 20,230 unique tracks.

**Why the low coverage isn't itself a bug:** separately confirmed that the Spotify reference file
is a fixed stratified sample — exactly 114 genres × 1,000 tracks each (std = 0.0 tracks per
genre) — not an exhaustive catalog. Even massively popular artists (Radiohead, Pink Floyd, The
Beatles) only have a handful of their most iconic songs represented. No amount of better matching
recovers a track that was never included in the reference data to begin with.

## Lessons carried into later sessions

- Always sanity-check row counts against a known baseline after any join — a summary statistic
  alone (like a match *rate*) can look fine while hiding a multiplied denominator.
- When a number changes a lot after a fix, verify it a second way before trusting it.
- Not every technique that's built needs to ship in the final pipeline — measuring a technique's
  actual contribution before keeping it is itself part of the job.
