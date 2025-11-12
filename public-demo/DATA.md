# Public Data Scope & Data Sources

> This document describes the types of data, sources, and database file distribution policies that Athlemetrics can provide in public demos, and outlines publication strategies for different `.db/.sqlite` files. It covers only high-level information available for external sharing and does not include private scripts or sensitive fields.

## 1. Data Sources & Collection Scope
| Source | URL/Channel | Collected Fields | Primary Use | Compliance Notes |
| --- | --- | --- | --- | --- |
| FBref | https://fbref.com | Standard statistics (goals, shots, passes, possession, goalkeeping, etc.) | Build `player_stats.db` raw tables for Big-5 leagues across multiple seasons, used for per90 calculations | Comply with FBref Robots/ToS, retain only publicly visible fields |
| Transfermarkt | https://www.transfermarkt.com | Player metadata (date of birth, nationality, preferred foot, on-field position), club affiliation | Primary source for `player_metadata` and API `country/position` fields | Reference only public information, avoid scraping restricted pages |
| WhoScored | https://www.whoscored.com | Match events (successful attempts, interceptions, fouls, ratings) | Enrich defensive/discipline features in `player_role_features.db` for scoring model | Use only publicly viewable match summaries, prohibit distributing raw event-level data |
| Official league/club websites | League websites, club press releases | Lineups, announcements, match day rosters | Verify information, fill data gaps (e.g., short-term loans) | Retain source links when referencing |
| Publicly reusable tactical/role research | Published white papers, blogs | Role templates & metric breakdowns | Build `player_role_clusters`, `role_labels` and other cluster descriptions | Use only materials permitted for citation, paraphrase accordingly |

> If integrating commercial data sources such as Opta or StatsBomb, update this table after obtaining authorization and clarify that related data must not be distributed in public repositories.

## 2. Cleaning & Synthesis Pipeline (Overview)
1. **Collection**: Scrape public pages and export CSVs from the above sources via custom crawlers/export scripts, then write to staging tables by season.
2. **Normalization**: Consolidate player IDs from different sources, impute missing values, and standardize all numeric values to per-90 or percentage scales.
3. **Feature Engineering**: Calculate efficiency, intensity, and discipline indices in `player_role_features.db`, as well as UMAP/clustering coordinates for model interpretability.
4. **Modeling & Prediction**: Feed derived features into Random Forest/MLP/XGBoost, output results tables such as `player_predictions`, `goalkeeper_predictions`.
5. **External Delivery**: Provide publicly permissible tables in CSV/SQLite format, and load the same data batch in APIs (`/db/*`, `/chart/*`) to ensure consistency between online demos and documentation.

## 3. Data Download Channels
- **Kaggle Dataset**: https://www.kaggle.com/datasets/athlemetrics/athlemetrics-public-demo
  - Contains the latest public versions of `football_players_prediction.db`, `player_stats.db`, and supporting CSVs (split by season).
  - Kaggle account holders can download without additional approval, making it convenient for users and contributors to reproduce API/model examples locally.
- **GitHub Release / Direct Compressed Archive**: For rapid access to the latest snapshot, download from the Releases section of the main repository or static download link (`https://cdn.athlemetrics.com/public-datasets/latest.zip`). Release notes will indicate version numbers corresponding to `MODEL.md`, `API.md`.

> Contents from both channels are identical; choose either one; Kaggle is better for long-term archival and community discussion, while Release links are more convenient for automated scripts.

## 4. Publicly Shareable Databases & Tables
| File/Table | Content Summary | Recommended Delivery Format | Public Availability | Notes |
| --- | --- | --- | --- | --- |
| `football_players_prediction.db` → `player_predictions` / `goalkeeper_predictions` | Final test/inference results from scoring model, including `rating_pred` and subscores | Complete `.db` or exported `.csv`/`.parquet` (paginated by season) | ✅ Publicly Available | Personally identifiable information removed; contains only public statistics; also serves as data source for RapidAPI `/db/*` |
| `player_stats.db` | Raw snapshot of Big-5 public statistics (11 tables: standard, shots, passes, GCA, goalkeeping, etc.) | Original `.db` (11 tables) or CSV exports by table | ✅ Publicly Available | Although raw data, contains only information queryable from source websites; usable for reproducing experiments |
| `player_role_features.db` → `player_metadata`, `player_role_clusters`, `player_role_umap_embedding`, `player_role_cluster_labels` | Metadata, labels, and visualization coordinates for role clustering | `.db` or split CSV | ✅ Publicly Available | These tables contain only aggregated labels, not sensitive feature vectors |

## 5. Databases Not Included in Public Releases
| File/Table | Handling Requirements | Reason |
| --- | --- | --- |
| `player_role_features.db` → `player_features` | Not included in Kaggle/GitHub release packages | This table records 100+ dimensional private feature vectors; public release may enable reverse-engineering the model |
| `football_players_combined.db` | Not included in Kaggle/GitHub release packages | Aggregates multi-stage cleaning and intermediate features; involves internal process details |

> Download packages for regular users and contributors contain only the content listed in Section 4; restricted databases above are not included in public packages and require no additional handling.

## 6. Data Visualization & API Results
- RapidAPI Playground: https://rapidapi.com/liverpoolynwa0008/api/athlemetrics-player-score-predictor/playground
- Rating Demo / Player Similarity Explorer / Tactical Role Analyzer: All invoke public databases from the above table and provide `/chart/*` PNG visualizations; see `public-demo/API.md` for details.

## 7. Compliance Statement
- Include only public data compliant with source site policies; no paid subscriptions or user privacy data.
- Before publishing any `.db/.csv`, re-verify that restricted tables (especially `player_features`) or unauthorized fields are not included.
- If a partner requests deletion or restriction of a data source, synchronously update this document and the deliverable inventory.
