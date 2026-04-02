# Metrics & RUM APIs – Hreflang / Market Mapping Contract

This document describes the API contract for the metrics and RUM endpoints that now support hreflang-style locale codes. 

All examples assume a local backend running at `http://localhost:4000`.

---

## 1. Shared Concepts

### 1.1. Base URLs

- Technical SEO (static + Firestore):
  - `/api/metrics/technical-seo`
  - `/api/metrics/technical-seo/trends`
- RUM / CWV metrics:
  - `/api/firebase/rum-metrics-p75-multiple`
  - `/api/firebase/linechart-metrics`

### 1.2. Hreflang → Market Mapping

The backend uses `mapHreflangToMarket(input)` (`src/utils/hreflangMapper.js`) to normalize incoming `market` query values into an internal *market slug* that determines the Firestore path (e.g. `technical_seo_trends/{brand}/{market}/...`).

The mapper accepts both hreflang codes and some legacy market strings. It tries, in order:

1. The full, lowercased input (e.g. `"en-us-stl"`).
2. The first two segments joined (e.g. `"en-us"` from `"en-us-stl"`).
3. The last segment (e.g. `"stl"` from `"en-us-stl"`).

If any of those candidates match `MARKET_MAP`, that mapped value is used as the normalized market.

#### 1.2.1. Supported mappings

| Input(s)                            | Normalized market slug |
| ----------------------------------- | ---------------------- |
| `en-ca`, `canada`                  | `canada`               |
| `tr-tr`, `turkey`                  | `turkey`               |
| `ja-jp`, `japan`                   | `japan`                |
| `en-us`, `usa`                     | `usa`                  |
| `en-us-stl`, `stl`                 | `stl`                  |
| `ar`, `ar-ae`, `mena`, `mena-arabic` | `mena`               |
| `en-ae`, `uae`                     | `uae`                  |
| `en-gb`, `en-uk`, `uk`, `united-kingdom` | `united-kingdom` |
| `en-in`, `india`                   | `india`                |

If no mapping exists, the backend treats the market as **unsupported**.

#### 1.2.2. Mapper result shape

The internal helper returns one of:

- Success: `{ market: "<normalized_slug>", matchedKey: "<key_used>" }`
- Error (empty input): `{ error: "MARKET_REQUIRED" }`
- Error (no rule): `{ error: "UNSUPPORTED_MARKET:<original_input>" }`

Only the **error-free** paths are accepted by the APIs described below.

### 1.3. Error responses

Across the endpoints in this document, the following error patterns are relevant:

- Unsupported market / hreflang:
  - Status: `400 Bad Request`
  - Body:  
    ```json
    {
      "error": "UNSUPPORTED_MARKET",
      "details": "UNSUPPORTED_MARKET:<rawMarket>"
    }
    ```

Other validation and internal errors are described under each endpoint.

---

## 2. GET `/api/metrics/technical-seo`

**Purpose:** Returns static, file-backed technical SEO series data for the selected date range. Market is *validated* (if present) but not used to filter data yet.

### 2.1. Query parameters

| Name       | Type   | Required | Description                                                     |
| ---------- | ------ | -------- | --------------------------------------------------------------- |
| `date_range` | string | No       | Range label understood by `resolveRange` (e.g. `7d`, `30d`).   |
| `network`  | string | No       | Optional filter (e.g. `"4G"`, `"3G"`).                          |
| `browser`  | string | No       | Optional filter (e.g. `"Chrome"`, `"Safari"`).                  |
| `device`   | string | No       | Optional filter (e.g. `"desktop"`, `"mobile"`).                 |
| `market`   | string | No       | Hreflang or market string. Only **validated** with the mapper. |

If `market` is provided, it must be one of the supported mappings (see §1.2). Unsupported values result in a 400.

### 2.2. Success response (200)

Body: an array of per-day entries (derived from `technical_seo.json`):

```json
[
  {
    "name": "2025-11-01",
    "metadata": 10,
    "indexing": 3,
    "schema": 4,
    "firebase": 1,
    "total": 18
  }
]
```

### 2.3. Error responses

- Unsupported market:
  - `400`, body as described in §1.3.
- Generic failure (e.g. JSON file missing):
  - `500`, body: `{ "error": "TECH_SEO_FETCH_FAILED" }`

---

## 3. GET `/api/metrics/technical-seo/trends`

**Purpose:** Returns Firestore-backed, time-series technical SEO trends per category for a given brand + market + date range.

### 3.1. Query parameters

| Name         | Type   | Required | Description                                                                 |
| ------------ | ------ | -------- | --------------------------------------------------------------------------- |
| `brand`      | string | Yes      | Brand slug (e.g. `"lipton"`, `"pukkah"`).                                  |
| `market`     | string | Yes      | Hreflang or market string; normalized using the mapper.                    |
| `startDate`  | string | No       | Explicit start date `YYYY-MM-DD`.                                           |
| `endDate`    | string | No       | Explicit end date `YYYY-MM-DD`.                                             |
| `date_range` | string | No       | Label (e.g. `7d`, `30d`) used if `startDate` / `endDate` are not provided. |

At least one of:

- `startDate` + `endDate`, or
- `date_range`

must resolve to a valid date range.

### 3.2. Validation & errors

- Missing brand or market:
  - Status `400`  
    `{ "error": "BRAND_AND_MARKET_REQUIRED" }`
- Unsupported market (after calling `mapHreflangToMarket`):
  - Status `400`  
    `{ "error": "UNSUPPORTED_MARKET", "details": "UNSUPPORTED_MARKET:<rawMarket>" }`
- Invalid or missing date range:
  - Status `400`  
    `{ "error": "INVALID_DATE_RANGE" }` or `{ "error": "DATE_RANGE_ORDER_INVALID" }`
- Generic failure:
  - Status `500`  
    `{ "error": "TECH_SEO_TRENDS_FETCH_FAILED" }`

### 3.3. Success response (200)

```json
{
  "brand": "lipton",
  "market": "united-kingdom",
  "range": { "startDate": "2025-11-01", "endDate": "2025-11-07" },
  "dates": ["2025-11-01", "2025-11-02"],
  "categories": [
    {
      "id": "indexing",
      "name": "Indexing issues",
      "total": 12,
      "severityTotals": { "error": 4, "warning": 5, "info": 3 },
      "trend": [
        {
          "date": "2025-11-01",
          "count": 10,
          "severity": { "error": 3, "warning": 4, "info": 3 }
        },
        {
          "date": "2025-11-02",
          "count": 12,
          "severity": { "error": 4, "warning": 5, "info": 3 }
        }
      ]
    }
  ]
}
```

`market` in the response is the normalized slug (e.g. `"en-gb"` → `"united-kingdom"`).

---

## 4. GET `/api/firebase/rum-metrics-p75-multiple`

**Purpose:** Returns RUM/CRUX/diagnostics category metrics over a date range, using Firestore data; primarily used to compute P75 metrics across multiple dates.

### 4.1. Query parameters

| Name        | Type   | Required | Description                                                            |
| ----------- | ------ | -------- | ---------------------------------------------------------------------- |
| `brand`     | string | Yes      | Brand slug; defaults to `"fcl"`.                                      |
| `market`    | string | Yes    | Hreflang or market string; defaults `"india"` but validated/normalized. |
| `date_range`| string | Yes      | Range label for `getStartAndEndDate` (e.g. `"7d"`).                   |
| `device`    | string | No       | Device type; normalized to `"desktop"` / `"mobile"` via `normalizeDevice`. |

\*If `market` is omitted, the default `"india"` is used, then normalized by the mapper.

### 4.2. Hreflang behavior

- `const { market: resolvedMarket, error } = mapHreflangToMarket(market)`
- On error:
  - `400`, body: `{ "error": "UNSUPPORTED_MARKET", "details": "UNSUPPORTED_MARKET:<rawMarket>" }`
- On success:
  - `resolvedMarket` is passed into `getMultipleDatesCategoryMetrics`.

### 4.3. Success response (200)

The shape is whatever `getMultipleDatesCategoryMetrics` returns; typically an array of per‑date aggregated metrics, e.g.:

```json
[
  {
    "date": "2025-11-01",
    "lcp_p75": 1800,
    "cls_p75": 0.05,
    "inp_p75": 120,
    "urls": 150
  }
]
```

### 4.4. Error responses

- Unsupported market: as in §4.2.
- Generic failure:
  - `500`, body: `{ "error": "Internal Server Error" }`

---

## 5. GET `/api/firebase/linechart-metrics`

**Purpose:** Returns per‑day line chart data for one of `rum`, `crux`, or `diagnostics`, using Firestore.

### 5.1. Query parameters

| Name       | Type   | Required | Description                                                                 |
| ---------- | ------ | -------- | --------------------------------------------------------------------------- |
| `brand`    | string | Yes       | Brand slug; defaults to `"fcl"`.                                           |
| `market`   | string | Yes     | Hreflang or market string; defaults `"india"` but normalized via mapper.   |
| `date_range` | string | No     | Range label; used if `daterange` is absent.                                |
| `daterange`  | string | No     | Alternate range parameter; `date_range ?? daterange` is used.             |
| `device`   | string | No       | Device type; normalized to `"desktop"` / `"mobile"`.                      |
| `category` | string | Yes      | One of `"rum"`, `"crux"`, `"diagnostics"` (case-insensitive).             |

### 5.2. Validation & errors

1. Market:
   - `const { market: resolvedMarket, error } = mapHreflangToMarket(market)`
   - On error → `400` with `UNSUPPORTED_MARKET` as described in §1.3.

2. Category:
   - Missing `category`:
     - `400`  
       `{ "error": "category query parameter is required (rum, crux, diagnostics)" }`
   - Invalid category:
     - `400`  
       `{ "error": "Invalid category \"<category>\". Expected rum, crux, or diagnostics." }`

3. Generic failure:
   - `500`, `{ "error": "Internal Server Error" }`

### 5.3. Success response (200)

Body is the output of `fetchLinechartMetricsByCategory` for the requested category, e.g.:

```json
[
  {
    "date": "2025-11-01",
    "totalUrls": 120,
    "goodPages": 80,
    "badPages": 40,
    "PassPercentage": 66.67,
    "FailPercentage": 33.33,
    "RumMetricsGlance": {
      "lcpPercentages": { "good": 70, "needImprovement": 20, "bad": 10 },
      "clsPercentages": { "good": 85, "needImprovement": 10, "bad": 5 },
      "inpPercentages": { "good": 65, "needImprovement": 25, "bad": 10 }
    }
  }
]
```

The exact shape depends on the category (`RumMetricsGlance`, `CruxMetricsGlance`, or `diagnosticsMetricsGlance`).

---

## 6. Quick Examples

### 6.1. Technical SEO trends for UK (hreflang)

```http
GET /api/metrics/technical-seo/trends?brand=lipton&market=en-gb&date_range=30d
```

- Backend resolves `market` → `"united-kingdom"` and queries Firestore path:  
  `technical_seo_trends/{brand}/{market}/daily/items` → `technical_seo_trends/lipton/united-kingdom/daily/items`.

### 6.2. RUM P75 metrics for Japan (hreflang)

```http
GET /api/firebase/rum-metrics-p75-multiple?brand=pukkah&market=ja-jp&date_range=7d&device=desktop
```

- Backend resolves `market` → `"japan"` and passes that slug to the underlying Firestore model.

### 6.3. Linechart diagnostics metrics for STL (special locale)

```http
GET /api/firebase/linechart-metrics?brand=lipton&market=en-us-stl&category=diagnostics&date_range=30d
```

- Backend resolves `market` → `"stl"` and fetches diagnostics metrics from the corresponding Firestore collections.

---

