# Indian Music Label Market Share Analysis
**A five-year reconstruction of India's streaming attention economy using YouTube data.**

---

## Overview

This project builds a complete, end-to-end research pipeline that answers one question: *who controls audience attention in India's music market, and how has that changed over five years?*

It does this by collecting every week of India's YouTube Music Top 100 chart from January 2021 to May 2026 (278 weeks, 28,000 chart rows, 3,337 unique tracks), enriching each track with YouTube metadata, classifying tracks to their parent music label, computing weekly market share, and visualizing the results across five chart types with a parallel institutional-only view.

The pipeline is split into six sequential notebooks. Each notebook has one job, produces one or more output files, and hands off cleanly to the next.

---

## Repository Structure

```
.
├── 01_youtube_charts_extractor.ipynb     # Proof-of-concept single-week fetch
├── 02_historical_chart_ingestion.ipynb   # Full five-year historical collection
├── 03_channel_metadata.ipynb             # YouTube Data API v3 enrichment
├── 04_label_mapping.ipynb                # Channel → parent label classification
├── 05_market_share_analysis.ipynb        # Weekly and overall market share computation
├── 06_visualization_dashboard.ipynb      # Chart generation (matplotlib)
│
├── india_weekly_chart.csv                # Single-week output (NB01)
├── india_youtube_weekly_charts_2021_2026.csv   # Full historical dataset (NB02)
├── india_youtube_enriched.csv            # + YouTube metadata (NB03)
├── india_youtube_labeled.csv             # + label column (NB04)
├── distribution.csv                      # Label distribution audit (NB04)
│
├── weekly_label_market_share.csv         # Weekly share, full market (NB05)
├── overall_label_market_share.csv        # Cumulative share, full market (NB05)
├── weekly_label_market_share_no_indie.csv    # Weekly share, institutional only (NB05)
├── overall_label_market_share_no_indie.csv   # Cumulative share, institutional only (NB05)
│
├── market_share_pivot.csv                # Wide-format pivot for charting (NB06)
├── hhi_trend.csv                         # Weekly HHI series (NB06)
├── market_share_pivot_no_indie.csv       # Pivot, institutional only (NB06)
├── hhi_trend_no_indie.csv                # HHI, institutional only (NB06)
│
└── charts/
    ├── 01_market_share_line_trend.png
    ├── 02_attention_share_stacked_area.png
    ├── 03_latest_week_bar.png
    ├── 04_overall_market_share_bar.png
    ├── 05_hhi_trend.png
    ├── 01_market_share_line_trend_no_indie.png
    ├── 02_attention_share_stacked_area_no_indie.png
    ├── 03_latest_week_bar_no_indie.png
    ├── 04_overall_market_share_bar_no_indie.png
    └── 05_hhi_trend_no_indie.png
```

---

## Data Source & Methodology

**Source:** YouTube Music Charts (India, Weekly Top 100) via the undocumented internal `charts.youtube.com` API, supplemented by the official YouTube Data API v3 for per-video metadata.

**Attention proxy:** Weekly view counts from the chart are used as a proxy for audience attention share. This is a consumption-side measure — it captures what people are actually watching, not what labels are releasing.

**Unit of analysis:** Each row in the master dataset is one track appearance in one weekly chart. A track that charts for 10 consecutive weeks contributes 10 rows, each with that week's view count.

**Market share definition:** For any given week, a label's market share is the sum of views across all its charting tracks divided by the total views across all 100 chart positions that week. Shares are computed independently each week — they do not carry forward.

---

## Notebook-by-Notebook Breakdown

---

### NB01 — `01_youtube_charts_extractor.ipynb`
**Purpose:** Proof of concept. Fetches a single week's Top 100 and saves it to CSV.

**How the API call works:**

YouTube Music Charts does not have a public API. This notebook reverse-engineers the internal endpoint that the YouTube Music website calls when you browse charts. The endpoint is:

```
POST https://charts.youtube.com/youtubei/v1/browse?alt=json
```

The payload identifies the client as `WEB_MUSIC_ANALYTICS` (the internal client name YouTube Music's analytics interface uses) and specifies India (`gl: IN`) as the geography, weekly granularity, and a specific end date. The API returns a deeply nested JSON object.

**The `find_trackviews` function:**

The response JSON is not a flat structure — the track list is buried several levels deep and its exact path can shift. Rather than hardcode a path, the function recursively walks the entire JSON tree (handling both dicts and lists) until it finds a key called `trackViews`. This is defensive: if YouTube restructures their response slightly, the extractor still works.

**Extraction:**

For each track in `trackViews`, the notebook pulls: rank, song name, artists (joined as a comma-separated string), weekly view count, the encrypted video ID, and weeks on chart. These are assembled into a DataFrame, sorted by rank, and saved.

**Output:** `india_weekly_chart.csv`

---

### NB02 — `02_historical_chart_ingestion.ipynb`
**Purpose:** Scales NB01 to the full five-year history by looping over every weekly date from 2021-01-07 to 2026-05-14.

**Date generation:**

The notebook generates all 278 Thursday dates in the study window using Python's `datetime` and `timedelta`. Each date is formatted as `YYYYMMDD` and passed as the `chart_params_end_date` parameter in the API query string.

**Fetch loop:**

Each week is fetched by calling `fetch_chart(date)`, which is identical in structure to NB01 but parameterized. The loop is wrapped in a `try/except` so a single failed week does not abort the entire run. Failed weeks are printed and skipped; successful weeks are appended to a list of DataFrames.

**Additional fields captured:**

NB02 captures two fields NB01 didn't: `previous_rank` (prior week's chart position) and `percent_views_change` (week-over-week view change as reported by YouTube). These are not used in the final analysis but are preserved in the dataset for future use — a sign that the researcher was thinking about momentum analysis.

**Rate limiting:**

A 1-second sleep between requests keeps the collection polite and avoids triggering YouTube's rate limits over a 278-request run.

**Known limitation:**

There is no checkpointing. If the loop fails at week 200, the run must restart from week 1. For a production pipeline this would be addressed by saving progress incrementally (e.g., writing each week to disk as it succeeds).

**Concatenation:**

All per-week DataFrames are concatenated with `pd.concat(ignore_index=True)` and saved as a single master file.

**Output:** `india_youtube_weekly_charts_2021_2026.csv` — 28,000 rows × 9 columns.

---

### NB03 — `03_channel_metadata.ipynb`
**Purpose:** Enriches the chart dataset with video-level metadata from the official YouTube Data API v3. This is what provides the `channel_title` field that NB04 uses for label classification.

**Why this step is necessary:**

The chart data from NB02 contains video IDs and artist names, but not the uploading channel. Channel title is the primary signal for label classification. To get it, every unique video ID in the dataset must be looked up via the YouTube Data API.

**Deduplication:**

3,337 unique video IDs are extracted from the 28,000-row dataset. The API is called once per unique video, not once per chart row — this reduces API quota usage by roughly 88%.

**Batching:**

The YouTube Data API allows fetching metadata for up to 50 video IDs per request. The `batched()` helper yields 50-item slices of the ID list. 3,337 IDs → 67 batches. Total wall-clock time in the logged run: approximately 36 seconds.

**Session with retry logic:**

Rather than using bare `requests.get()`, the notebook builds a `requests.Session` with a `urllib3.util.retry.Retry` object attached. This configures automatic retries (up to 3) with exponential backoff (factor 1.5) on HTTP 429 (rate limit), 500, 502, 503, and 504 responses. This is production-grade API client behaviour.

**Fields extracted per video (`parse_item`):**

| Field | Source |
|---|---|
| `video_id` | item.id |
| `channel_id` | snippet.channelId |
| `channel_title` | snippet.channelTitle |
| `video_title` | snippet.title |
| `description` | snippet.description |
| `published_at` | snippet.publishedAt |
| `tags` | snippet.tags (list) |
| `category_id` | snippet.categoryId |
| `default_language` | snippet.defaultLanguage |
| `duration` | contentDetails.duration (ISO 8601) |
| `youtube_total_views` | statistics.viewCount |
| `youtube_like_count` | statistics.likeCount |
| `youtube_comment_count` | statistics.commentCount |

**Missing video handling:**

If the API returns fewer items than were requested in a batch, the notebook identifies which IDs are missing (set difference between requested and returned) and logs them with a warning. Missing videos are typically deleted or set to private since the chart was collected.

**Merge:**

The 3,337-row metadata DataFrame is left-joined onto the 28,000-row chart DataFrame on `video_id`. Row count is verified to not have changed post-merge (a guard against accidental many-to-many joins from duplicate video IDs in the metadata response).

**Output:** `india_youtube_enriched.csv` — 28,000 rows × 21 columns.

---

### NB04 — `04_label_mapping.ipynb`
**Purpose:** Maps each row's `channel_title` to its parent music label. This is the most methodologically significant notebook in the pipeline — it is where the market share figures are constructed.

**Classification architecture:**

Two lookup structures are defined:

`EXACT_MAP` is a Python dict mapping exact channel title strings to label names. It covers all known channels for 20+ institutional labels including all regional sub-channels (e.g., `T-Series Apna Punjab`, `T-Series Tamil`, `Lahari Music | T-Series` all map to `T-Series`). This is checked first and is case-sensitive.

`_PATTERN_MAP` is a list of compiled regex patterns checked in order if the exact match fails. For example, any channel title containing `t-series` (case-insensitive) maps to T-Series. Word boundary anchors are used where needed (e.g., `\bYRF\b` to avoid false matches on strings that merely contain those letters).

**Classification priority:**

1. If `channel_title` is NaN → `"Independent"` (fallback)
2. If exact match in `EXACT_MAP` → that label
3. If regex match in `_PATTERN_MAP` → first matching label
4. Otherwise → `"Independent"` (fallback)

**Performance:**

`apply_labels()` runs the exact map as a vectorized pandas `.map()` operation across the entire series before falling back to row-wise `.apply()` only for unmatched entries. This is significantly faster than a pure row-wise loop on 28,000 rows.

**Coverage results (from logged output):**

- 28,000 total rows
- 12,522 rows (44.7%) matched a known institutional label
- 15,478 rows (55.3%) fell through to the `"Independent"` fallback

**Audit output:**

The `report()` function prints the full label distribution and lists all 947 unique channel titles that were classified as Independent. This list is the key transparency mechanism — it allows any analyst to inspect what is being called "independent" and verify or challenge the classification.

**Important methodological note:**

The `"Independent"` label is a fallback bucket, not a confirmed classification. It contains genuine independent artists (Anuv Jain, AP Dhillon, Armaan Malik) but also YouTube auto-generated topic channels (`Anirudh Ravichander - Topic`), smaller distributors (`Artiste First`), and regional channels not in the map. The 52.8% independent market share figure in the final presentation is more precisely described as *non-major-institutional* share. The directional finding — that non-institutional content commands the majority of attention — is robust, but the exact number should be footnoted accordingly.

**Output:** `india_youtube_labeled.csv` — 28,000 rows × 22 columns (adds `label`). Also `distribution.csv`.

---

### NB05 — `05_market_share_analysis.ipynb`
**Purpose:** Computes weekly and cumulative market share from the labeled dataset. Produces four output files: full market and institutional-only, each in weekly and overall form.

**Input validation:**

Before any computation, the notebook checks that the required columns (`week`, `label`, `views`) are present and raises a `ValueError` if not. View counts are coerced to numeric and rows with unparseable values are dropped with a warning (0 rows dropped in the actual run).

**`compute_weekly_share()`:**

1. Groups by `(week, label)` and sums views → `label_views` per label per week.
2. Groups by `week` alone and sums `label_views` → `total_weekly_views` per week.
3. Joins the two on `week` (preserving all label rows).
4. Computes `market_share = label_views / total_weekly_views` and `market_share_pct` (rounded to 2dp).
5. Sorts by week ascending, then share descending.

This produces a long-format DataFrame where each row is one label in one week — 3,368 rows for 280 weeks × up to 21 labels.

**`compute_overall_share()`:**

Groups by `label`, sums all views across all weeks, sorts descending, and computes share against the grand total. This gives the five-year cumulative picture: Independent 52.82%, T-Series 17.74%, Saregama 5.95%, etc.

**`compute_labeled_share()`:**

Filters out the `"Independent"` label, then re-runs both computations from scratch on the filtered DataFrame. This is important: shares are recalculated against the new (reduced) total, so they sum to 100% within the institutional universe rather than being relative to the full market. This is what produces the institutional split view in the presentation (T-Series 37.6%, Saregama 12.6%, etc.).

**Output files:**

| File | Description |
|---|---|
| `weekly_label_market_share.csv` | Long-format weekly share, full market |
| `overall_label_market_share.csv` | Cumulative share, full market |
| `weekly_label_market_share_no_indie.csv` | Long-format weekly share, institutional only |
| `overall_label_market_share_no_indie.csv` | Cumulative share, institutional only |

---

### NB06 — `06_visualization_dashboard.ipynb`
**Purpose:** Generates all charts from the market share data. Produces five chart types, run twice — once for the full market (including Independent) and once for the institutional-only view — yielding 10 PNG files total.

**Label colours:**

A fixed `LABEL_COLOURS` dict maps each label to a hex colour. This ensures colour consistency across all five chart types — T-Series is always `#E63946` (red), Saregama always `#2A9D8F` (teal), etc. Unmapped labels fall back to `#888888`.

**Chart types:**

`chart_line_trend()` — Multi-line chart of weekly market share per label over the full study period. Each label is one line. Uses 1.6px linewidth for readability at chart scale. X-axis formatted as quarterly date ticks (`MMM YYYY`).

`chart_stacked_area()` — Stacked area chart of the same data. Shows how the total attention pool is divided over time. Alpha 0.85 on fills. This is the clearest chart for visualizing structural shifts.

`chart_latest_week_bar()` — Bar chart of the most recent week's market share, top 15 labels. Value labels are placed above each bar with a +0.3 offset. The latest week is determined dynamically from `df["week"].max()`.

`chart_overall_bar()` — Bar chart of cumulative five-year market share, top 15 labels. Recalculates share from the weekly data at render time (rather than reading from the overall CSV) so this chart is always internally consistent with the weekly data it's charting from.

`chart_hhi()` — Line chart of the Herfindahl-Hirschman Index over time with a shaded area fill. Two dashed reference lines at HHI = 0.15 ("Moderate") and HHI = 0.25 ("Concentrated") provide the economic concentration benchmarks used in the analysis takeaways.

**HHI computation (`compute_hhi()`):**

HHI is computed weekly as the sum of squared market shares across all labels in that week: `HHI = Σ(sᵢ²)` where `sᵢ` is label i's share expressed as a decimal. A value above 0.25 indicates a highly concentrated market; below 0.15 indicates moderate competition. The chart shows the full market HHI has consistently exceeded 0.25, driven primarily by the Independent fallback bucket's dominant share.

**The `generate_charts()` function:**

Takes a DataFrame, a label list, output paths for pivot and HHI CSVs, and a filename suffix. Builds the wide-format pivot (labels as columns, weeks as rows), computes HHI, saves both intermediate CSVs, then calls all five chart functions. Called twice from `main()` — once with the full dataset (suffix `""`) and once with the no-indie dataset (suffix `"_no_indie"`).

**Known bug — RecursionError:**

Notebook 06 has a recursion bug in the `generate_charts()` function. The function defines a local `_save` closure that is intended to replace the module-level `save` function (via `mod.save = _save`), so that chart filenames automatically include the suffix. However, `_save` calls `save(fig, name + suffix)` — and because `save` in the local scope at that point *is* `_save`, this creates infinite recursion. The charts were produced (the logged output confirms the pivot and HHI CSVs saved successfully, and the presentation contains the correct charts), meaning the bug was either present in a prior version of the notebook that ran successfully, or was introduced after the charts were generated. It does not affect the analytical results. The fix is to capture the original `save` function in a local variable before reassigning: `_orig_save = save; def _save(fig, name): _orig_save(fig, name + suffix)`.

---

## Known Limitations

**Independent bucket is a fallback, not a confirmed classification.** Any channel not matched by `EXACT_MAP` or `_PATTERN_MAP` is labelled Independent. This includes some label-affiliated or distributor-managed channels that weren't in the mapping. The 52.8% independent share figure is best described as *non-major-institutional* share. The institutional analysis is unaffected.

**No checkpointing in NB02.** A failure mid-loop requires a full restart. For a production pipeline, each week should be written to disk as it succeeds.

**YouTube Music Charts internal API is undocumented.** The `charts.youtube.com` endpoint used in NB01 and NB02 is not a public API. It can change without notice. If the response structure changes, `find_trackviews()` will fail silently (returning `None`) rather than raising an error.

**YouTube Data API quota.** NB03 consumes 67 quota units (1 per batch of 50 IDs). The default daily quota is 10,000 units, so a single run is well within limits. Re-running the full pipeline repeatedly in one day could approach limits if the dataset grows.

**View counts as attention proxy.** Weekly view counts measure consumption volume but not depth of engagement. A track with 10M views from passive autoplay may score higher than a track with 5M views from active search. This is a structural limitation of any chart-based market share methodology.

**Study window starts January 2021.** Pre-2021 data is not available through the YouTube Music Charts interface, so the pipeline cannot capture pre-streaming-era dynamics or the COVID-19 consumption shock of 2020.

---

## Dependencies

```
requests
pandas
matplotlib
urllib3
```

All notebooks run on Python 3.13. A YouTube Data API v3 key is required for NB03 (set `API_KEY` in the config section). The `charts.youtube.com` endpoint in NB01 and NB02 requires no authentication.

---

## Running the Pipeline

Run notebooks in order. Each notebook reads the output of the previous one.

```
NB01 → india_weekly_chart.csv (optional, for testing)
NB02 → india_youtube_weekly_charts_2021_2026.csv
NB03 → india_youtube_enriched.csv
NB04 → india_youtube_labeled.csv
NB05 → weekly/overall market share CSVs
NB06 → charts/*.png
```

NB01 is a standalone proof of concept and does not need to be run before NB02. NB02 through NB06 must be run in sequence.

---

## Key Findings

| Metric | Value |
|---|---|
| Study period | Jan 2021 – May 2026 |
| Weekly observations | 278 |
| Total chart rows | 28,000 |
| Unique tracks | 3,337 |
| Unique channels | 1,013 |
| Institutional labels tracked | 20 |
| Independent / non-institutional share (5yr) | 52.8% |
| T-Series overall share | 17.7% (full market) / 37.6% (institutional only) |
| Saregama overall share | 6.0% (full market) / 12.6% (institutional only) |
| T-Series baseline share trend | ~45% institutional (2021) → ~30% (2026) |
| HHI range (full market) | 0.20 – 0.46 |
| HHI range (institutional only) | 0.13 – 0.41 |
