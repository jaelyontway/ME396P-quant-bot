# NVDA Event‑Driven Pipeline

Quant Bot stitches together Google News scraping, ML‑based margin prediction, and an NVDA trading simulation. You can either run the full controller for a cached date or trigger individual stages while iterating on a single component.

- **Controller‑driven demo** – type a date from `data/demo_data/`, and `controllerProgram.py` copies the CSVs, executes EAST, and launches the simulation.
- **Modular workflow** – use the scripts under `data_fetching/`, `src/east/`, `training/`, and `simulation/` independently when you only need a subset of the pipeline.

If you prefer a picture before diving in, the diagrams in `ARCHITECTURE.md` outline the end‑to‑end data flow.

---

## Quick Start

### Option A – Full demo (cached data, no API calls)
1. Follow the [Environment Setup](#environment-setup).
2. `python controllerProgram.py`
3. Enter any folder name that exists under `data/demo_data/` (for example `2025-10-14`). The script copies the demo CSVs into place, runs EAST, and spawns the simulator automatically.

### Option B – Manual stage run
1. Fetch new data (optional): `python data_fetching/data_fetching.py --config config/config.yaml`
2. Copy/rename the resulting `news_*.csv` → `feedcsv.csv` and `nvda_prices_*.csv` → `feedcsv2.csv` in the repo root.
3. Run EAST to generate `src/east/output.txt`: `python src/east/run_east_model.py`
4. Drop `simulation/NVDA_prices.csv` and `simulation/output.txt` into the simulation folder (or just re‑use the controller’s copies) and execute `python simulation/simulation_program.py`.

`quick_start.md` contains the minimal commands used to replicate the canonical Oct 1–20 window and points to commonly referenced CSVs.

---

## Repository Layout

| Path | Purpose |
| --- | --- |
| `config/config.yaml` | Polygon key, keyword defaults, and simulation overrides. |
| `controllerProgram.py` | Glue script that moves cached data, runs EAST, and launches the simulator. |
| `data/` | Canonical news/price CSVs, demo data (`data/demo_data/`), and derived datasets. |
| `data_fetching/` | Google News + Polygon orchestrator (`data_fetching.py`). |
| `data_processing/` | Helpers such as `normalize_news_times.py` for timezone cleanup. |
| `news_data/` | Clean article bodies (`clean_content_in.csv`) for feature engineering. |
| `src/east/` | Event clustering + sentiment stack (EAST) that produces trading margins. |
| `training/` | Random Forest training utilities plus sentiment/margin helpers. |
| `simulation/` | Plotting/exit‑signal scripts (`simulation_program.py`, helpers, configs). |
| `models/` | Saved predictors like `margin_predictor.pkl`. |

All shared dependencies live in `requirements.txt`.

---

## Environment Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

The requirement set covers the analysis stack (`numpy`, `matplotlib`, `scikit-learn`) plus the sentiment/LLM helpers (`vaderSentiment`, `sentence-transformers`, `openai`) imported by downstream scripts.

---

## Configure Credentials and Defaults

Edit `config/config.yaml` (or copy it) and fill in:

- `apis.stocks.key` with your Polygon key.
- `defaults.keywords`, `defaults.news_date`, `defaults.price_date`, etc. to define the date windows and search filters.

The same YAML config powers `data_fetching/data_fetching.py`, the training layer, and the simulation helpers. You can also set `STOCK_API_KEY` in the environment to override the key at runtime.

---

## Fetch News + Prices (optional when using demos)

```bash
cd data_fetching
python data_fetching.py \
  --config ../config/config.yaml \
  --news-date 2025-10-10 \
  --news-end-date 2025-10-11 \
  --price-date 2025-10-12 \
  --price-hours-after 48 \
  --keywords government shutdown nvidia
```

Common flags:
- `--news-date` / `--news-end-date` – inclusive Google News range (defaults live in YAML).
- `--price-date` – anchor day for NVDA prices (falls back to the news date).
- `--price-hours-after` – length of the post-event window; pair with `--price-trading-days 0` to use hours instead of sessions.
- `--price-trading-days` – fetch whole trading days (set to `0` to disable).
- `--interval-minutes` – Polygon bar resolution.
- `--skip-news` – reuse news CSVs and only re-pull prices.
- `--news-timezone` – force the timezone used when normalizing news timestamps.

Each run emits `data/NVDA_<DATE>/` with `news_*.csv`, `nvda_prices_*.csv`, and metadata JSON. Re-run `combine_price_runs.py` if you need to merge multiple price pulls (see `quick_start.md` for the Oct 1–20 canonical file).

---

## Run the EAST Model

The EAST stack inside `src/east/` clusters related headlines, scores sentiment, and derives directional margins around the latest NVDA price.

```bash
cd src/east
python run_east_model.py  # expects feedcsv.csv + feedcsv2.csv in repo root
```

Inputs:
- `feedcsv.csv` – news CSV (from `data_fetching` or `data/demo_data`).
- `feedcsv2.csv` – intraday price CSV aligned to the same event window.

Outputs:
- `src/east/output.txt` – upper/lower margin levels, sentiment score, recent price snapshot.
- Console log with the generated signal, coverage stats, and sentiment details.

---

## Predict Margins / Train ML

1. **Create training data** – join the news and price features (clean bodies live in `news_data/clean_content_in.csv`) or reuse the prepared CSVs in `data/training_data/`.
2. **Train**:
   ```bash
   python training/train_ml_model.py \
     --training-data training_data.csv \
     --output models/margin_predictor.pkl \
     --test-size 0.2
   ```
3. **Use predictions** – `training/src/integrate_margins.py` loads the fetched CSVs and calls `margin_predictor.predict_margins` (rule-based by default, ML-powered when the model exists).

---

## Simulation Workflows

### Controller (`controllerProgram.py`)
- Requests a demo date from `data/demo_data/NVDA_<DATE>/`.
- Copies `news_<DATE>.csv` → `feedcsv.csv` and `nvda_prices_<DATE>.csv` → `feedcsv2.csv`.
- Runs `src/east/run_east_model.py` and copies `src/east/output.txt` into `simulation/output.txt`.
- Writes the date to `simulation/targetSimDate.txt`, syncs `config/config.yaml` into `simulation/{config,nvidia-config}.yaml` (patching the date), mirrors the price CSV to `simulation/NVDA_prices.csv`, and launches `simulation/simulation_program.py`.

### Manual simulation (if you skip the controller)
1. Set the desired date inside `simulation/targetSimDate.txt`.
2. Place `NVDA_prices.csv` (and optionally `output.txt` from EAST) inside `simulation/`.
3. Ensure `simulation/config.yaml` is populated (copy from `config/config.yaml` if needed).
4. Run:
   ```bash
   cd simulation
   python simulation_program.py
   ```
   The script visualizes the upper/lower margins, labels the first exit signal, and prints realized vs. hold returns.

`preSimulationController.py` and `simPriceHelper.py` can automate CSV prep if you want a lighter-weight alternative to the main controller.

---

## Reference Material

- `quick_start.md` – literal commands to pull Oct 1–20 data plus tips on using canonical CSVs.
- `ARCHITECTURE.md` – mermaid diagrams for the architecture, directory layout, and sequence flow.
- `feedcsv.csv` / `feedcsv2.csv` – working copies of the latest news/price pull used by EAST.
- `simulation/targetSimDate.txt` – the simulation date override (auto-managed by the controller).

Use these docs when onboarding new contributors or when you need to verify the end‑to‑end data shapes before making code changes.
