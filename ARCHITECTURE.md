# NVDA Event-Driven Pipeline Architecture

High-level visual overview of the project architecture and data flow.

---

## System Architecture

```mermaid
graph TB
    CONFIG[Config Files] -.->|configure| FETCH
    CONFIG -.->|configure| SIM

    subgraph FETCH[1. Data Fetching]
        NEWS[News Data<br/>Google RSS]
        PRICES[Stock Prices<br/>Polygon API]
    end

    subgraph STORAGE[2. Data Storage]
        CSV[CSV Files<br/>news + prices]
    end

    subgraph TRAIN[3. ML Training]
        SENTIMENT[Sentiment Analysis]
        MODEL[Random Forest<br/>Margin Predictor]
    end

    subgraph SIM[4. Simulation]
        BACKTEST[Backtest Strategy<br/>Entry/Exit Signals]
    end

    FETCH --> STORAGE
    STORAGE --> TRAIN
    TRAIN --> MODEL
    MODEL -.->|predictions| SIM
    STORAGE --> SIM
    SIM --> RESULTS[Trading Plots<br/>Performance Metrics]

    classDef configClass fill:#e1f5ff,stroke:#0288d1,stroke-width:2px
    classDef dataClass fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef processClass fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef modelClass fill:#e8f5e9,stroke:#388e3c,stroke-width:2px

    class CONFIG configClass
    class CSV dataClass
    class FETCH,TRAIN,SIM processClass
    class MODEL,RESULTS modelClass
```

---

## Data Flow

```mermaid
flowchart LR
    INPUT[News Articles<br/>+ Stock Prices]

    PROCESS[Sentiment Analysis<br/>+ Feature Engineering]

    ML[Train ML Model<br/>Random Forest]

    PREDICT[Predict Price Margins<br/>Upper/Lower Bounds]

    TRADE[Generate Signals<br/>Buy/Sell Thresholds]

    INPUT --> PROCESS
    PROCESS --> ML
    ML --> PREDICT
    PREDICT --> TRADE

    classDef inputClass fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef processClass fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef outputClass fill:#e8f5e9,stroke:#388e3c,stroke-width:2px

    class INPUT inputClass
    class PROCESS,ML,PREDICT processClass
    class TRADE outputClass
```

---

## Pipeline Stages

1. **Data Fetching**: Collect news articles and stock prices
2. **Data Storage**: Save raw data as CSV files with timezone normalization
3. **ML Training**: Train Random Forest model to predict price margins from sentiment
4. **Simulation**: Backtest trading strategy with predicted buy/sell signals
