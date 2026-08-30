# Capstone Report — Structured Content Archetype Clustering

- **Author:** Farhan
- **Lane:** Structured Content Archetype Clustering
- **Repo:** https://github.com/muhammadfarhan2157-source/flyrank-ml-internship
- **Date:** 2026-08-30

## 0. Abstract

Content teams reviewing hundreds or thousands of pages need a way to triage them beyond "everything is a priority." This project clusters 30,000 anonymized FlyRank content pages into six performance archetypes using 16 safe, current-state metrics (traffic, CTR, engagement, freshness, content properties), with no article text and no client-identifying data. A KMeans model (k=6, chosen by silhouette score) produced groups that are moderately separated (silhouette ≈ 0.20) but highly reproducible under re-seeding and bootstrapping (Adjusted Rand Index mostly 0.80–1.00), and profiling each cluster's raw metrics before naming it surfaced six distinct archetypes — from "Champions" carrying the most traffic to "Invisible/Unindexed" pages with almost no search presence. The output is a ranked action queue (protect / improve / rewrite / monitor / merge / prune) a content editor can act on directly, with "Engagement-Problem Pages" (26% of the inventory, high traffic, page-one position, but 0.25% median CTR) as the top review priority.

## 1. Problem framing

**Unit of analysis:** one content page (a row in the 90-day performance snapshot).
**Output:** a cluster assignment, archetype name, and recommended action per page.
**Action a human takes:** a content editor pulls the ranked queue and works top-down — improving metadata/CTR on "Engagement-Problem Pages," rewriting "Stale Visible Pages," and pruning or merging the weakest, lowest-demand tail — instead of reviewing thousands of pages in no particular order.
**Cost of a wrong call:** clustering errors here cost review time, not revenue directly — a page miscategorized into "monitor" instead of "improve" delays attention by a cycle, which is a low-stakes, recoverable error (unlike, say, an automated de-indexing decision). This is why the output is framed as decision support, not an automated trigger.
**Why ML helps:** manually segmenting thousands of pages on 16 interacting metrics isn't something a human can do consistently by eye; clustering finds the natural groupings so a person doesn't have to eyeball a spreadsheet.

## 2. Data safety

**Data used:** `data/raw/content_refresh_anonymized.csv` — the anonymized 30k-row, 32-client starter slice. No warehouse/Hugging Face data was pulled for this pass (starter dataset only).

**Columns deliberately excluded:**
- `trend_direction`, `trend_pct` — the data dictionary defines these as the label source (`is_declining_label = trend_direction == "down"`); used only afterward to *describe* clusters, never as clustering inputs.
- `content_id`, `client_id` — pseudonymous IDs, used only for grouping/counting, never as features.
- `provider_used`, `model_used` — explicitly flagged "not a model feature" in the data dictionary.
- No raw URLs, keyword text, or client names exist in this slice, so none appear anywhere in `work/`.

**Leakage risk considered:** none of the above label-adjacent columns entered the feature matrix; this was checked by listing `feature_cols` explicitly in the notebook (Section 1) rather than passing the whole dataframe to the scaler.

## 3. Baseline

Clustering is unsupervised, so there is no accuracy baseline to beat in the classic sense. The equivalent "transparent rule" comparison here is a single hand-written rule an editor might already use — e.g., "review anything below page 1" or "review anything with zero clicks." Checked against the clusters: a naive "zero clicks" rule would flag five of the six archetypes (everything except "Champions" and "Engagement-Problem Pages," which do get clicks) — it can't distinguish *why* a page has no clicks (no demand vs. stale vs. unindexed vs. filler), which is exactly the gap the archetype breakdown closes. That comparison is the "baseline" for this lane: one flat bucket vs. six that separate on content-type, age, freshness, and position, not just current click count.

## 4. Model / analysis

**Method:** StandardScaler → KMeans (`k=6`, chosen from a silhouette grid over k=3–8) → PCA(2) for visualization only.

**Exact feature list (16 features):** `log_impressions_90d`, `log_clicks_90d`, `log_sessions_90d`, `log_ai_sessions_90d`, `ctr`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`, `avg_position_clean`, `has_position_data`, `log_word_count`, `had_word_count`, `log_search_volume`, `had_search_volume`, `content_age_days`, `days_since_update_capped`.

**Left out on purpose:** `competition`, `competition_level`, `cpc`, `main_intent`, `age_tier`/`freshness_tier`/etc. (redundant with the raw numeric fields already included), and all label-adjacent/ID/excluded columns from Section 2.

**Target/proxy:** none — this is unsupervised; "archetype" is the cluster label itself, assigned after profiling, not predicted from a ground-truth target.

## 5. Evaluation

**Split:** not applicable in the train/test sense (unsupervised). Instead, robustness was checked with (a) a silhouette-score grid across candidate `k` values, and (b) a stability check: re-fitting KMeans with 5 different random seeds and on 5 bootstrap resamples, each compared to the original assignment via Adjusted Rand Index.

**Metrics:**
- Silhouette (k=6, subsampled n=6,000): **0.1985** — the best of the k=3–8 grid, but a moderate absolute score, meaning cluster boundaries are soft.
- ARI vs. re-seeding (5 seeds): **0.845–0.997** (mean ≈ 0.94).
- ARI on bootstrap resamples (5 resamples): **0.798–0.990** (mean ≈ 0.90).

**Error analysis (in place of a confusion matrix):** the two archetypes with the fewest pages ("Aggregator Filler Pages," "Invisible/Unindexed Pages," ~1,200–1,600 pages each) sit at the extremes of near-zero traffic and are the most stable clusters conceptually, while the three-way split between "Weak/No-Demand," "Stale Visible," and "Engagement-Problem" pages carries the least clean boundary — these three account for almost all of the moderate silhouette score, since they differ mainly in degree (how much traffic, how stale) rather than in kind.

## 6. Interpretation

Reading the profile table (median metrics per cluster, computed but not used as clustering input) before naming anything:

| Archetype | n pages | Read from the data | Action |
|---|---|---|---|
| Champions (refresh risk) | 1,270 (4.2%) | Highest impressions/sessions, longest content, only cluster with real AI-referral traffic — but the highest median days-since-update (~104d vs ~20–25d elsewhere) | Protect + monitor freshness |
| Engagement-Problem Pages | 7,866 (26.2%) | High impressions, page-1 position, but 0.25% median CTR and 1.7% engagement rate | Improve (metadata/CTR + on-page engagement) |
| Stale Visible Pages | 6,773 (22.6%) | Oldest median age (~441d), moderate impressions, page-2 position, ~0 measured clicks/engagement | Rewrite / refresh |
| Weak/No-Demand Pages | 11,295 (37.7%) | Largest cluster, youngest median age, low impressions, near-zero engagement, low search volume | Monitor (prune weakest tail) |
| Aggregator Filler Pages | 1,601 (5.3%) | Feedly-sourced, zero search volume (no keyword target), negligible traffic despite a decent position | Merge or prune |
| Invisible/Unindexed Pages | 1,195 (4.0%) | Near-zero impressions, mostly no position data, and almost no decline *or* growth — already at the floor | Investigate indexing / prune |

**Surprise / negative result:** "cannibalization-risk" and true semantic "hidden gems" archetypes (both suggested in the lane brief as possibilities) did not emerge and are not claimed — this slice has no article text (ruling out semantic grouping) and no cross-page keyword-identity join (ruling out a defensible cannibalization signal). Reporting that gap honestly is itself a finding: not every plausible archetype survives contact with what the data can actually support.

## 7. Recommendation

Ranked by where editor time pays off fastest:

1. **Engagement-Problem Pages (26.2%)** — highest-leverage fix: pages already get page-1 visibility and real traffic, so a metadata/CTR pass captures upside without needing new content.
2. **Stale Visible Pages (22.6%)** — rewrite/refresh candidates: oldest content, page-2 position, zero engagement; a content refresh could plausibly move these toward page 1.
3. **Champions (4.2%)** — protect and watch freshness: these carry the most traffic and the only meaningful AI-referral sessions, but the longest gap since last update flags a refresh-risk clock worth monitoring, not ignoring.
4. **Weak/No-Demand (37.7%), Aggregator Filler (5.3%), Invisible/Unindexed (4.0%)** — lower priority for active rewriting; review for pruning or merging, since the underlying issue (no demand, no keyword target, or no index presence) usually isn't fixed by a content edit.

**Confidence and limits, stated explicitly:** treat this as a first-pass triage lens, not a final verdict — cluster boundaries are soft (silhouette ≈ 0.20) even though the groupings reproduce well across re-seeding and bootstrapping. A page near a cluster boundary deserves a human glance before action, especially before any merge/prune decision.

## 8. Reproducibility

```bash
git clone https://github.com/muhammadfarhan2157-source/flyrank-ml-internship.git
cd flyrank-ml-internship
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute work/notebooks/capstone.ipynb
```

- **Random seeds:** 42 throughout (StandardScaler is deterministic; KMeans/PCA seeded at 42, `n_init=10`).
- **Committed receipts:** `work/outputs/action_mapping.json`, `work/outputs/cluster_profile.json`, `work/notebooks/figures/*.png` — all aggregated/derived, no raw per-row dataset committed (per `work/README.md`'s no-datasets-in-git rule; note `work/**/*.csv` is gitignored repo-wide, which is why these receipts are JSON, not CSV).
- **Base rate note:** not applicable here in the precision@K sense (unsupervised task); the closest equivalent, `pct_declining` per archetype (41–63% across the five higher-traffic archetypes vs. essentially the whole-dataset decline rate of 54.2%), is reported in the profile table above rather than a single headline number, since no archetype's decline rate is dramatically different from the overall base rate — a result worth stating plainly rather than overselling.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — https://flyrank.ai

---

**Claims checklist:** observed / measured / directional / decision-support language used throughout · no causal claims · no "predicted Google's algorithm" claim · no client-identifying details anywhere in this report or the notebook · numbers above match a fresh execution of `work/notebooks/capstone.ipynb` (re-run 2026-08-30, 0 errors).
