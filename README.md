# Netflix Content Intelligence & Recommendation Platform

A local data engineering pipeline that ingests Netflix catalog data through a **Medallion Architecture**, builds analytical dashboards exposing content strategy gaps, and serves a **content-based recommendation engine** with an interactive UI — all in a single Jupyter notebook.

---

## The problem

OTT platforms face three persistent, expensive challenges:

**Content discovery failure** — users who can't find what to watch churn. Netflix has publicly acknowledged that improving recommendations directly impacts retention.

**Blind content acquisition** — without data on genre saturation, regional gaps, and catalog freshness, content spend is guided by gut feeling rather than evidence. Which genres are oversaturated? Which emerging markets are underserved?

**Catalog staleness** — libraries age invisibly. A platform can have 6,000 titles and still feel empty if 70% of them are pre-2015 content in the same three genres.

This project addresses all three by turning raw catalog CSVs into actionable intelligence and a working recommender.

---

## Architecture

```
CSV files (Kaggle)
     │
     ▼
┌──────────┐     ┌───────────┐     ┌──────────┐
│  BRONZE   │────▶│  SILVER    │────▶│  GOLD    │
│ Raw → Pqt │     │ Clean+Join │     │ Agg tbls │
│ + lineage │     │ + features │     │ 4 tables │
└──────────┘     └─────┬─────┘     └────┬─────┘
                       │                 │
                       ▼                 ▼
                 ┌───────────┐    ┌────────────┐
                 │  ML Model  │    │ Dashboards │
                 │ TF-IDF +   │    │  8 Plotly  │
                 │ Cosine Sim │    │   charts   │
                 └─────┬─────┘    └────────────┘
                       │
                       ▼
                ┌─────────────┐
                │ Interactive  │
                │ Recommender  │
                │  (widgets)   │
                └─────────────┘
```

**Medallion layers in detail:**

| Layer | Input | Operations | Output |
|-------|-------|-----------|--------|
| Bronze | 5 raw CSVs | Schema validation, ingestion timestamps, source tracking | `.parquet` files with lineage metadata |
| Silver | Bronze parquets | Type casting, null handling, dedup, join all 5 tables, feature engineering (content age, decade, desc length) | Single enriched parquet: 6,200+ titles × 25+ columns |
| Gold | Silver parquet | Pre-aggregated analytics: genre trends, country volume, content freshness, rating distribution | 4 business-ready parquet tables |

---

## Tech stack

| Component | Technology |
|-----------|-----------|
| Ingestion & transformation | Pandas, NumPy |
| Storage format | Parquet (PyArrow) |
| ML / NLP | Scikit-learn (TF-IDF, cosine similarity, MultiLabelBinarizer), SciPy |
| Visualization | Plotly (interactive charts, dark theme) |
| Interactive UI | ipywidgets (Combobox search, sliders, live output) |
| Runtime | Jupyter Notebook / JupyterLab |

No cloud services, no API keys, no external dependencies beyond pip packages.

---

## Dataset

Source: [Netflix Movies and TV Shows](https://www.kaggle.com/datasets/shivamb/netflix-shows) (Kaggle)

| File | Records | Description |
|------|---------|-------------|
| `netflix_titles.csv` | 6,236 | Core catalog — title, type, rating, duration, description, release year |
| `netflix_directors.csv` | 4,852 | Director ↔ show_id mapping |
| `netflix_cast.csv` | 44,311 | Cast member ↔ show_id mapping |
| `netflix_category.csv` | 13,670 | Genre/category ↔ show_id mapping |
| `netflix_countries.csv` | 7,179 | Country ↔ show_id mapping |

---

## Quick start

### 1. Clone and set up

```bash
# Unzip the project
unzip netflix_content_intelligence_pipeline.zip
cd netflix_project

# Create a virtual environment (recommended)
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install pandas numpy scikit-learn plotly ipywidgets kaleido scipy pyarrow
```

### 2. Verify data

The CSVs should be at:
```
netflix_project/
├── data/
│   └── raw/
│       ├── netflix_titles.csv
│       ├── netflix_directors.csv
│       ├── netflix_cast.csv
│       ├── netflix_category.csv
│       └── netflix_countries.csv
├── netflix_content_intelligence_pipeline.ipynb
└── README.md
```

If your files are elsewhere, update `DATA_DIR` in the Configuration cell.

### 3. Run

```bash
jupyter notebook netflix_content_intelligence_pipeline.ipynb
```

Run all cells top to bottom. The pipeline creates `data/bronze/`, `data/silver/`, `data/gold/`, and `models/` directories automatically.

> **JupyterLab users:** if ipywidgets don't render, run:
> `jupyter labextension install @jupyter-widgets/jupyterlab-manager`

---

## What each section does

### Section 1 — Bronze layer (raw ingestion)
Reads each CSV, attaches ingestion metadata (`_ingested_at`, `_source_file`, `_record_count`), writes to Parquet with Snappy compression. Validates schemas and reports null counts per table.

### Section 2 — Silver layer (cleaning + joining)
- Casts `duration_minutes`, `duration_seasons`, `release_year` to proper types
- Parses `date_added` into datetime, extracts `year_added` and `month_added`
- Normalizes ratings (strips suffixes)
- Engineers features: `content_age`, `decade`, `desc_length`, `type_flag`
- Aggregates each lookup table (directors, cast, genres, countries) into lists per `show_id`
- Joins all 5 tables into a single enriched DataFrame
- Deduplicates on `show_id`

### Section 3 — Gold layer (analytics tables)
Produces four pre-aggregated tables:
- **genre_trend** — title count by genre × year × type
- **country_volume** — title count and avg release year by country × type
- **content_freshness** — title count and avg duration by decade × type
- **rating_distribution** — title count by rating × type

### Section 4 — Analytical dashboards
Eight interactive Plotly visualizations:
1. Movies vs TV Shows split (donut chart)
2. Top 15 genres by title count
3. Content additions over time (area chart)
4. Top 20 content-producing countries
5. Rating distribution by content type
6. Genre × year heatmap (top 12 genres)
7. Genre co-occurrence pairs (top 40)
8. Genre opportunity map — catalog depth vs recent growth % (scatter)
9. Country content profile — volume vs freshness (scatter)

The last two are the actual business insights: they identify where Netflix is underinvesting.

### Section 5 — Recommendation engine
- Builds a "soup" per title: description + genres + top directors + rating
- Fits TF-IDF (8,000 features, bigrams) on the soup
- One-hot encodes genres via MultiLabelBinarizer
- Combines matrices with tunable weighting (70% TF-IDF, 30% genre)
- `recommend(title)` — single-title similarity search
- `recommend_from_multiple(titles)` — averages feature vectors across multiple liked titles, returns blended recommendations
- Cosine similarity computed on-the-fly (no full 6K×6K matrix in memory)

### Section 6 — Interactive UI
- 5 Combobox dropdowns with type-ahead search across the full catalog
- Content type filter (Any / Movie / TV Show)
- Results count slider (5–30)
- Outputs a styled DataFrame with similarity scores, genres, and release year

### Section 7–8 — Genre clustering + content gap analysis
- Genre co-occurrence matrix revealing which genres naturally pair together
- Scatter plots mapping opportunity zones: high growth + low catalog depth = content gap

---

## Recommendation engine — how it works

The engine uses **content-based filtering**. Given a title (or set of titles), it:

1. Looks up the title's combined feature vector (TF-IDF on text + one-hot genres)
2. Computes cosine similarity against every other title in the catalog
3. Ranks by similarity, optionally filters by content type
4. Returns top-N results

For multi-title input, it averages the feature vectors first, then runs similarity. This produces recommendations that blend the characteristics of all input titles rather than over-indexing on any single one.

**Limitations (be honest about these):**
- Content-based only — no collaborative signal (requires user interaction data)
- Descriptions vary in quality; short/generic descriptions reduce similarity accuracy
- No temporal weighting — a 2010 title is treated the same as a 2021 title
- The TF-IDF vocabulary is frozen at fit time; new titles require re-fitting

---

## Project structure (after running)

```
netflix_project/
├── data/
│   ├── raw/                         # Original CSVs
│   ├── bronze/                      # Parquet + lineage metadata
│   │   ├── titles.parquet
│   │   ├── directors.parquet
│   │   ├── cast.parquet
│   │   ├── categories.parquet
│   │   └── countries.parquet
│   ├── silver/                      # Enriched joined table
│   │   └── netflix_enriched.parquet
│   └── gold/                        # Analytics aggregations
│       ├── genre_trend.parquet
│       ├── country_volume.parquet
│       ├── content_freshness.parquet
│       └── rating_distribution.parquet
├── models/                          # (reserved for serialized models)
├── netflix_content_intelligence_pipeline.ipynb
└── README.md
```

---

## Productionization roadmap

This is a local prototype. To take it further:

| Step | What | Why |
|------|------|-----|
| Live data source | Replace CSVs with TMDB API or a web scraper on a schedule | Static data = stale recommendations |
| Collaborative filtering | Add user watch/rating data, build a hybrid recommender (content + collaborative) | Content-based alone plateaus; collaborative captures taste patterns |
| Model serving | Wrap `recommend()` in a FastAPI endpoint, containerize with Docker | Decouples model from notebook; enables frontend integration |
| Pipeline orchestration | Migrate Bronze → Silver → Gold to Airflow or Prefect DAGs | Scheduling, retries, alerting, lineage tracking |
| Frontend | React/Next.js app consuming the API | Actual user-facing product |
| A/B testing | Log recommendations served vs clicked, measure CTR | Prove the engine adds value over random or popularity-based suggestions |
| Vector DB | Move embeddings to Pinecone/Weaviate for sub-ms retrieval at scale | Cosine similarity on 6K titles is fine; on 600K it's not |

---

## License

This project uses the publicly available Netflix dataset from Kaggle. The code is provided as-is for educational and portfolio purposes.
