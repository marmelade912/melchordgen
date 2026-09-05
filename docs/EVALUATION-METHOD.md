# How melchordgen is evaluated

The hard problem is not producing notes that are in key. It is producing a part that a listener would keep over a familiar sample. That is a taste question, so the project treats taste as something to measure rather than argue about.

## Blind rating rounds

1. **Fixed reference loops.** A small set of the producer's own loops is held constant across rounds so results are comparable over time.
2. **Candidate clips.** For each round, generated parts (and, early on, hand-authored control clips) are rendered over the reference loops. Clips are labelled with opaque IDs; the listener does not know which generator version or rule set produced which clip.
3. **Rating.** The listener rates each clip on a fixed scale with a short written reason. Early rounds also captured pairwise preferences ("keep A or B").
4. **Scoring.** Per generator version, the pass rate (share of clips at or above the acceptance threshold) is reported with a 95% confidence interval computed from the number of ratings, so a version with three lucky clips cannot look better than one with thirty solid ones.
5. **Decision rule.** A change to the generator or the constraint layer is accepted only if its interval clears the current baseline. Ties and overlaps are rejected or re-run with more clips. This is a pre-registered gate: the threshold is written down before the round, not after.
6. **Drift check.** Because there is one rater, a subset of earlier clips is re-rated in later rounds to measure how much the rater's own standard moved. Results are read against that drift.

## What the early rounds produced

- Fourteen hand-authored clips over the producer's sample established the quality floor ("competent but average") that the real generator has to beat.
- The written reasons were distilled into a short list of explicit rules that the constraint layer now enforces: default to mid register and moderate density, allow chromatic notes only when they resolve, use extended chords only when voiced well, prefer cohesion over busyness.
- Every later generator change is evaluated against that floor with the interval method above.

## Why this matters beyond music

The same discipline applies to any system whose output is judged by people: fix the reference set, blind the rater, state the acceptance threshold in advance, report an interval rather than a single number, and re-test the rater against old items to catch drift. It is the difference between "it sounds better to me today" and a result someone else can check.
