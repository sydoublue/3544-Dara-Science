# 3544-Data-Science
# 🌐 Multi-Platform Social Breakout Prediction

> Cross-platform graph-based forecasting of incoming message bursts and growth in online social networks.

**Supported platforms**: CollegeMsg (Facebook-like), Math Overflow, Bluesky, Koo, Voat.

This repository provides a complete pipeline to:

1. **Unify** heterogeneous temporal interaction data from five different social platforms into a single schema.
2. **Extract** graph structural features (degree, PageRank, clustering, reciprocity) using trailing‑window snapshots.
3. **Engineer** weekly time‑series features (lags, rolling statistics, interaction‑type composition, sentiment).
4. **Predict** two targets:
   - **Regression**: percentage change in incoming messages next week.
   - **Classification**: whether a user will experience a “breakout week” (increase >200% or >2 standard deviations above their personal mean).

The whole process is designed to handle **platform heterogeneity** without losing the individual identity of each network.

---

## 📦 Datasets

| Platform              | Source                                                                                        | Nodes  / Edges (approx.)         | Period      |
| --------------------- | --------------------------------------------------------------------------------------------- | -------------------------------- | ----------- |
| **CollegeMsg**        | [SNAP](https://snap.stanford.edu/data/CollegeMsg.html)                                        | 1,899 / 59,835                    | 193 days    |
| **Math Overflow**     | [SNAP](https://snap.stanford.edu/data/sx-mathoverflow.html) (a2q, c2q, c2a)                   | Large                            | 2,350 days  |
| **Bluesky, Koo, Voat**| [MADOC (Zenodo)](https://zenodo.org/records/14637314)                                          | 23.1M users across platforms     | 2012–2024   |

All data is publicly available. We automatically stream and parse the raw files during the data ingestion stage – no manual download is required (see [Getting Started](#getting-started)).

---

## 🧱 Project Structure

├── data/                   # (ignored) Raw & processed data files
├── notebooks/              # Exploratory & visualization notebooks
├── src/
│   ├── ingestion.py        # Stage 1: schema unification
│   ├── features.py         # Stage 2: weekly aggregation & target building
│   ├── graph_features.py   # Stage 3: graph structural features
│   ├── modelling.py        # Stage 4: XGBoost training, evaluation, SHAP
│   └── utils.py            # Helper functions
├── configs/                # YAML/JSON configuration for windows, splits
├── outputs/                # Figures, model checkpoints, results
├── README.md
├── requirements.txt
└── LICENSE


---

## 🚀 Pipeline Overview

### Stage 1 – Data Ingestion & Schema Unification
We bring every data source into a single table with columns:

| Column            | Description                                              |
| ----------------- | -------------------------------------------------------- |
| `src_user`         | Who initiated the interaction                            |
| `dst_user`         | Who received it                                          |
| `timestamp_utc`    | UNIX timestamp (seconds)                                 |
| `platform`         | One of `collegemsg`, `mathoverflow`, `bluesky`, `koo`, `voat` |
| `interaction_type` | `message`, `a2q`, `c2q`, `c2a`, `comment`, `repost`, `post` |

- **CollegeMsg**: loaded directly from SNAP `.txt.gz`.
- **MathOverflow**: three separate files loaded and concatenated to preserve `interaction_type`. The union file is intentionally ignored.
- **MADOC**: Parquet files streamed from Zenodo; `user_id`, `parent_user_id`, `publish_date` are mapped to the unified schema; rows with no receiver (standalone posts) are dropped to keep the data as interaction networks. Sentiment columns (`sentiment_vader`, `strict_filter`) are carried forward for later use.

### Stage 2 – Weekly Aggregation & Target Construction
- Count incoming and outgoing edges per **(platform, user, calendar week)**.
- Compute interaction‑type composition (e.g. fraction of comments, reposts).
- Build lag (1‑week, 2‑week) and rolling (4‑week mean & std) **using `.shift(1)` before the rolling window** to avoid data leakage.
- Construct the targets:
  - **Regression**: `pct_change_next_week = (next_week_count - this_week_count) / (this_week_count + 1)`
  - **Classification**: `is_breakout = (next > rolling_mean + 2*rolling_std) OR (next > 3 * (this_week_count + 1))`
- Drop cold‑start rows and terminal weeks.

### Stage 3 – Graph Feature Extraction
Instead of a single static graph, we use **trailing‑window snapshots**:
- For each prediction week *t*, consider only edges from weeks `[t-W, t-1]`.
- Build directed graphs with `igraph` (fast even for hundreds of thousands of nodes).
- Extract per‑user metrics:
  - `in_degree`, `out_degree`
  - `pagerank` (directed)
  - `clustering` (local undirected)
  - `reciprocity_personal` (fraction of contacts who also message back)
- Features are computed at a few snapshot boundaries and forward‑filled until the next snapshot.

### Stage 4 – Modelling & Evaluation
**Feature matrix** includes:
- Temporal position, user tenure
- Lag & rolling activity features
- Outgoing/incoming ratios
- Interaction type composition (zeros for platforms missing certain types)
- Graph structural features
- Sentiment (MADOC only)
- Platform one‑hot encoding

**Baselines**:
- *Persistence* for regression (predicted change = 0)
- *Always‑no‑breakout* for classification

**Model**: XGBoost
- Regression: `reg:squarederror`, evaluated with MAE / RMSE
- Classification: `binary:logistic` with `scale_pos_weight`, evaluated with AUROC, F1, and Precision‑Recall AUC (breakout events are rare).

**Validation**: Strict temporal split **per platform** (first 70% weeks training, next 15% validation, last 15% test). No global shuffling.

**Interpretation**: SHAP values are extracted to rank the most important features globally and per platform, answering whether graph features (PageRank, clustering) add predictive power beyond simple activity lags.

---

## 🧪 Getting Started

### 1. Clone and install dependencies
```bash
git clone https://github.com/your-username/your-repo.git
cd your-repo
pip install -r requirements.txt

---

## Citation
Pietro Panzarasa, Tore Opsahl, and Kathleen M. Carley. "Patterns and dynamics of users' behavior and interaction: Network analysis of an online community." Journal of the American Society for Information Science and Technology 60.5 (2009): 911-932.
Ashwin Paranjape, Austin R. Benson, and Jure Leskovec. "Motifs in Temporal Networks." In Proceedings of the Tenth ACM International Conference on Web Search and Data Mining, 2017.
Mitrovic Dankulov, M., Tomašević, A., Maletic, S., Andjelkovic, M., Vranic, A., Cvetkovic, D., Stupovski, B., Vudragovic, D., Major, S., & Bogojević, A. (2025). MADOC: Multi-Platform Aggregated Dataset of Online Communities (1.0.0) [Data set]. Zenodo. https://doi.org/10.5281/zenodo.14637314
