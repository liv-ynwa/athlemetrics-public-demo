# Athlemetrics Public API Guide

> This guide covers the publicly exposed scoring prediction, database retrieval, and visualization interfaces for reuse across Rating Demo, RapidAPI Playground, or custom applications. Complete interactive documentation is available at https://api.athlemetrics.com/redoc.

## Online Reference Entry Points (Highly Recommended to Try First)
- Rating Demo: https://platform.athlemetrics.com/rating-demo/ (same origin as `POST /predict_features` endpoint)
- RapidAPI Playground: https://rapidapi.com/liverpoolynwa0008/api/athlemetrics-player-score-predictor/playground
- ReDoc (production API full specification): https://api.athlemetrics.com/redoc
- Other products: Player Similarity Explorer, Tactical Role Analyzer (same account system, cross-validate responses)

## Basic Information

### Base URL & Environments
| Environment | Purpose | Base URL Example |
| --- | --- | --- |
| Production | Rating Demo, local integration | `https://api.athlemetrics.com` |
| RapidAPI Free Tier | No backend deployment required, rate-limited | `https://athlemetrics-player-score-predictor.p.rapidapi.com` |
| Local Debug | Run `uvicorn app:app --reload` | `http://localhost:8000` |

> RapidAPI and production endpoints follow the same OpenAPI specification; only authentication method and rate limits differ.

### Authentication & Required Request Headers
- RapidAPI:
  - `X-RapidAPI-Key: <your-key>`
  - `X-RapidAPI-Host: athlemetrics-player-score-predictor.p.rapidapi.com`
- Direct production connection: public GET requests allowed by default; `POST /predict_features` should be placed behind your own API Gateway with `X-API-Key`. For formal API keys, submit a request via the "Contact" form in Rating Demo.
- All JSON requests must include `Content-Type: application/json`.

### Common Conventions
- Response encoding is UTF-8 JSON; chart endpoints return `image/png` binary.
- Time and season fields use strings (example: `"2023-24"`).
- Numeric values are consistent decimal floats; documentation examples show 2 decimal places, but actual precision is retained.
- Error codes follow HTTP semantics: `400` validation failure, `404` no matching record, `429` / `503` platform rate limiting or maintenance.

## Quick Start
1. Select `POST /predict_features` in RapidAPI Playground, fill in sample payload, and observe the response.
2. Open Rating Demo, switch roles/models to compare with API results.
3. For custom frontend builds, reuse cURL or JavaScript snippets below; to browse all schemas, visit https://api.athlemetrics.com/redoc.

## Endpoints Quick Reference
| Group | Method | Path | Summary | Main Output |
| --- | --- | --- | --- | --- |
| Prediction | POST | `/predict_features` | Input role and raw season data, return normalized score and subscores | JSON (score + dimensions) |
| Database | GET | `/db/player/search` | Fuzzy search player ratings by name | JSON list |
| Database | GET | `/db/player/{id}` `/db/goalkeeper/{id}` | Fetch single player rating by ID or season | JSON object |
| Database | GET | `/db/predictions` | Unified filtering (league/team/season/nationality/position) | JSON list + pagination |
| Raw Data | GET | `/data/league/{league}` `/data/team/{team}` etc. | Return cleaned raw statistics rows | JSON list |
| Raw Data | GET | `/data/league/{league}/csv` | Download CSV snapshot for a league | `text/csv` |
| Charts | GET | `/chart/*` series | Generate bar, radar, trend charts, etc. (for Rating Demo) | `image/png` |

---

## 1. Prediction Score: `POST /predict_features`
- Function: Based on role (Attacker / Midfielder / Defender / Goalkeeper) and raw season/period statistics, automatically derive per90 features and output composite score with multiple subscores.
- Query parameter: `model_version` (default `rf_model_per90`, specifies model package version for canary releases).
- Headers: `Content-Type: application/json`, with authentication headers if required.

### 1.1 Request Body Structure (RawOutfieldUnion)
#### Common Fields
| Field | Description |
| --- | --- |
| `role` | Role, enum: `Attacker`, `Midfielder`, `Defender`, `Goalkeeper` |
| `model` | Sub-model used, enum: `mlp`, `xgb`, `rf`; switchable in Rating Demo |
| `minutes_played` | Total season minutes played; must be ≥ 1 to derive per90 |

#### Outfield (Forwards/Midfielders/Defenders) Required Statistics
| Field | Type | Description |
| --- | --- | --- |
| `goals`, `shots`, `shots_on`, `assists` | int | Core attacking data for conversion rate and goal contribution |
| `tackles`, `tackles_blocks`, `tackles_interceptions` | int | Defensive intensity for `defensive_efficiency_per90` |
| `duels_total`, `duels_won` | int | Ground duel indicators, further derive win rate/intensity |
| `dribbles_attempts`, `dribbles_completed` | int | Dribble attempts and successful dribbles |
| `passes`, `passes_key` | int | Pass and key pass counts |
| `fouls_committed`, `fouls_drawn` | int | Fouls committed and drawn data |
| `yellow_cards`, `red_cards` | int | Discipline fields; generates `discipline_score` when provided |

#### Goalkeeper Additional Fields
| Field | Type | Description |
| --- | --- | --- |
| `goal_saves` | int | Number of saves |
| `goals_conceded` | int | Goals conceded (for calculating `goal_save_rate_per90`) |
| Other fields | same as above | Goalkeepers still require passing, dueling, foul data beyond goalkeeper metrics |

> All fields use season totals; server automatically converts to per90, efficiency, intensity, discipline index, etc. No need to manually provide `_per90` or `_score` fields.

### 1.2 Example: Attacker Request & Response
```bash
curl -X POST "https://api.athlemetrics.com/predict_features?model_version=rf_model_per90" \
  -H "Content-Type: application/json" \
  -d '{
    "role": "Attacker",
    "model": "rf",
    "minutes_played": 2430,
    "goals": 18,
    "shots": 92,
    "shots_on": 47,
    "assists": 9,
    "tackles": 25,
    "tackles_blocks": 12,
    "tackles_interceptions": 14,
    "duels_total": 410,
    "duels_won": 205,
    "dribbles_attempts": 120,
    "dribbles_completed": 74,
    "passes": 1650,
    "passes_key": 55,
    "fouls_committed": 22,
    "fouls_drawn": 58,
    "yellow_cards": 5,
    "red_cards": 0
  }'
```

Example response:
```json
{
  "role": "Attacker",
  "model": "rf",
  "model_version": "rf_model_per90",
  "prediction": 4.87,
  "goals_score": 5.415704832167999,
  "shots_score": 9.482113991188555,
  "tackles_score": 6.049003987003068,
  "assists_score": 4.584295167832001,
  "discipline_score": 3.7754066879814543,
  "dribbles_score": 3.7754066879814543,
  "duels_score": 3.7754066879814543,
  "passes_score": 10,
  "foul_score": 3.7754066879814543
}
```

Field descriptions:
- `prediction`: Main score, between 0–1; displayed as 0–100 in Rating Demo.
- `*_score`: Optional subscores; exact fields depend on role and model (goalkeepers return `goalsave_score`).
- `model_version`: Facilitates version comparison consistent with `MODEL.md` changelog.

### 1.3 JavaScript/Browser Example
```js
async function predict(apiBase, token, payload) {
  const res = await fetch(`${apiBase}/predict_features?model_version=rf_model_per90`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': token ?? ''
    },
    body: JSON.stringify(payload)
  });
  if (!res.ok) {
    const detail = await res.json().catch(() => ({}));
    throw new Error(`HTTP ${res.status}: ${detail.detail ?? 'Unknown error'}`);
  }
  return res.json();
}
```

---

## 2. Player Prediction Database: `/db/*`
These endpoints directly read from `player_predictions` and `goalkeeper_predictions` tables, used in Rating Demo to search players or perform batch retrieval.

| Method | Path | Key Parameters | Description |
| --- | --- | --- | --- |
| GET | `/db/player/search` | `name` (required, fuzzy match) | Return multiple prediction records by name (include `id`, `league`, `team`, `rating_pred`, etc.) |
| GET | `/db/goalkeeper/search` | `name` | Same as above, for goalkeepers |
| GET | `/db/player/{player_id}` | `season` (optional) | Return complete row for single player; if `season` provided, return only that season's data |
| GET | `/db/goalkeeper/{goalkeeper_id}` | same as above | Goalkeeper version |
| GET | `/db/league/{league_name}/player_predictions` | `limit`, `offset` | All player predictions for a league; supports pagination |
| GET | `/db/team/{team_name}/player_predictions` | `limit`, `offset` | Aggregated by team |
| GET | `/db/season/{season}/player_predictions` | `limit`, `offset` | Aggregated by season |
| GET | `/db/predictions` | `role` (player/goalkeeper), optional `league`, `team`, `season`, `country`, `position`, `limit`, `offset` | Unified filter endpoint; default returns first page of all records |

Example:
```bash
curl "https://api.athlemetrics.com/db/predictions?role=player&league=Premier%20League&season=2023-24&limit=25"
```

Response fields (excerpt):
```json
[
  {
    "id": 10234,
    "name": "Mohamed Salah",
    "league": "Premier League",
    "team": "Liverpool",
    "season": "2023-24",
    "rating_pred": 0.89,
    "goals_score": 0.94,
    "assists_score": 0.81,
    "duels_score": 0.73,
    "minutes_played": 2430,
    "goals_per90": 0.67,
    "shots_per90": 3.41,
    "passes_key_per90": 1.98,
    "country": "Egypt",
    "position": "RW"
  },
  ...
]
```

> Result fields match database columns; can cross-reference with `MODEL.md` to understand derived feature meanings. For column information, first call `/data/schema`.

---

## 3. Raw Data Snapshot: `/data/*`
These endpoints directly expose cleaned `player_stats_combined` table for verifying input data or creating custom visualizations.

| Method | Path | Description |
| --- | --- | --- |
| GET | `/data/league/{league}` | Return raw statistics rows for specified league, supports `limit` and `offset` pagination |
| GET | `/data/team/{team}` | Same as above, filtered by team |
| GET | `/data/season/{season}` | Filter by season (season can be `2023` or `2023-24`, consistent with database) |
| GET | `/data/player/{player_id}` | Complete row for single player |
| GET | `/data/league/{league}/csv` | Download league results as CSV (`Content-Disposition: attachment`) |
| GET | `/data/schema` | Return table names and column structure for building custom queries |

---

## 4. Charts/Visualizations: `/chart/*`
All chart endpoints return `image/png`; Rating Demo displays directly via `<img>`. Common options below (all require `model` to be `player` or `goalkeeper`):

| Path | Function | Key Parameters |
| --- | --- | --- |
| `/chart/bar/{model}/{player_id}` | Single-game or season metrics bar chart | `metrics` (comma-separated fields), `width/height/scale` |
| `/chart/radar/{model}/{player_id}` | Ability radar chart | `metrics` (default includes main subscores) |
| `/chart/trend/{model}/{player_id}` | Season trend line chart | Automatically uses `rating_pred` over seasons |
| `/chart/compare/{model}` | Two-player/goalkeeper metric comparison | Query: `p1`, `p2`, `metric` |
| `/chart/histogram/{model}` | Metric distribution histogram | Query: `metric`, `league`, `season`, `bins` |
| `/chart/boxplot/{model}` | Metric boxplot by category | Query: `metric`, `by`, `season` |
| `/chart/heatmap/{model}` | Metric correlation heatmap | Query: `metrics`, `season` |
| `/chart/stacked/{model}/{player_id}` | Season stacked contribution | `metrics` |
| `/chart/parallel/{model}/{player_id}` | Parallel coordinates chart | `metrics` |

Example (radar chart):
```bash
curl -L "https://api.athlemetrics.com/chart/radar/player/10234?metrics=goals_score,shots_score,assists_score,dribbles_score,passes_score,duels_score,rating_pred" \
  -o radar.png
```

> Chart endpoints mainly return image byte streams without caching; for direct frontend display, add your own CDN or client-side caching strategy.

---

## 5. Errors & Rate Limiting
| Status Code | Scenario | Solution |
| --- | --- | --- |
| 400 | Missing fields, `minutes_played < 1`, model/role not in allowed list | Correct request body or query parameters |
| 404 | No matching player/goalkeeper or chart data row | Check `id`, `league`, `season` spelling |
| 422 | Pydantic validation failure (field type error) | Confirm field types/are integers |
| 429 | RapidAPI free tier or platform rate limit | Retry later, or upgrade RapidAPI plan |
| 500 | Database connection or model inference error | Record `X-Request-ID` and provide feedback via Rating Demo page |

Default rate limits: RapidAPI free tier ~5 req/s; production environment recommended to stay below 20 req/s with retry/backoff logic.

---

## 6. Further Reading
- Model & output interpretation: `public-demo/MODEL.md`
- Data scope & sources: `public-demo/DATA.md`
- Complete interactive specification: https://api.athlemetrics.com/redoc
- For deeper player rating samples, trigger the same interface requests directly in Rating Demo / Player Similarity Explorer.

