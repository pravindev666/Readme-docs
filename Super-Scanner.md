# Alpha-Zeta Super Scanner -  Guide

> **Project Type:** Momentum-Based Stock Screening & Trading System  
> **Target Universe:** Nifty 500 Stocks  
> **Technology Stack:** Python, Streamlit, yfinance, GitHub Actions, Random Forest ML  
> **Key Achievement:** +32.8% ROI on backtested 1-2 week swing trades

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [Data Pipeline](#data-pipeline)
4. [Core Logic & Algorithms](#core-logic--algorithms)
5. [Technical Indicators](#technical-indicators)
6. [Scoring Engine (Filter 1)](#scoring-engine-filter-1)
7. [Safety Filters](#safety-filters)
8. [Backtesting Methodology](#backtesting-methodology)
9. [Backtest Results](#backtest-results)
10. [Key Interview Q&A](#key-interview-qa)

---

## Executive Summary

The Alpha-Zeta Super Scanner is a **professional-grade momentum engine** designed for the Indian equity market. It scans the entire Nifty 500 universe to identify high-probability breakout candidates using a combination of:

- **Technical Analysis**: RSI, EMA, ATR, Hurst Exponent
- **Volume Analysis**: Turnover-based liquidity filters
- **Momentum Scoring**: Filter 1 Ensemble (Momentum × 100 + Volume × 2.0)
- **Risk Management**: Automatic position sizing with 10% allocation limits

---

## System Architecture

### High-Level Architecture Diagram

```mermaid
graph TB
    subgraph "Data Layer"
        A[GitHub Actions<br/>Scheduled Workflow] -->|6x daily| B[fetch_nifty_data.py]
        B -->|yfinance API| C[nifty500_ohlcv.csv<br/>~5.8 MB]
        C --> D[metadata.json<br/>Freshness Tracker]
    end
    
    subgraph "Application Layer"
        E[Streamlit App<br/>streamlit_app.py] -->|Reads| C
        F[CLI App<br/>app.py] -->|Reads| C
    end
    
    subgraph "Processing Engine"
        E & F --> G[DataEngine<br/>Symbol Fetching]
        G --> H[TechnicalCore<br/>Indicator Calculations]
        H --> I[FormulaFactory<br/>Scoring Engine]
        I --> J[Safety Filters<br/>Risk Guards]
        J --> K[Results<br/>Ranked Opportunities]
    end
    
    subgraph "Output"
        K --> L[Top 20 Stocks Table]
        K --> M[CSV Export]
        K --> N[Trade History Log]
    end
```

### Component Layer Architecture

```mermaid
graph LR
    subgraph "Layer 1: Data"
        A1[DataEngine] --> A2[get_nifty_symbols]
        A1 --> A3[clean_yf_data]
        A1 --> A4[fetch_stock_data]
        A1 --> A5[load_csv_data]
    end
    
    subgraph "Layer 2: Technical"
        B1[TechnicalCore] --> B2[calculate_rsi]
        B1 --> B3[calculate_ema]
        B1 --> B4[calculate_atr]
        B1 --> B5[get_hurst]
        B1 --> B6[calculate_indicators]
    end
    
    subgraph "Layer 3: Scoring"
        C1[FormulaFactory] --> C2[generate_all<br/>Filter 1 Ensemble]
    end
    
    subgraph "Layer 4: UI"
        D1[AlphaZetaScanner] --> D2[setup_menu]
        D1 --> D3[run_scanner]
        D1 --> D4[log_top_picks]
    end
```

---

## Data Pipeline

### Where Data is Fetched From

| Source | Description | Primary/Fallback |
|--------|-------------|------------------|
| **NSE Official** | `niftyindices.com/IndexConstituent/ind_nifty500list.csv` | Primary for symbols |
| **Yahoo Finance** | `yfinance` Python library | Primary for OHLCV |
| **Pre-fetched CSV** | `data/nifty500_ohlcv.csv` | Primary for Streamlit |
| **Static Fallback** | Hardcoded top 8 stocks | Emergency fallback |

### Where Data is Stored

```
Alpha_Zeta_Super_Scanner/
├── data/
│   ├── nifty500_ohlcv.csv      # 500 stocks × 200 days OHLCV (~5.8 MB)
│   ├── nifty500_live.csv       # Hourly updated data
│   ├── metadata.json           # Last update timestamp
│   └── live_metadata.json      # Live data freshness
├── trade_history.csv           # All scanner executions
├── persistent_pick_history.csv # Top-ranked picks over time
└── backtest_v4_data.pkl        # Cached backtest data (~8.4 MB)
```

### GitHub Actions Data Fetch Workflow

```mermaid
sequenceDiagram
    participant GH as GitHub Actions
    participant Script as fetch_nifty_data.py
    participant NSE as NSE Website
    participant YF as Yahoo Finance
    participant Repo as Repository
    
    GH->>Script: Trigger (11:00 AM, 3:15 PM IST)
    Script->>NSE: GET Nifty 500 symbols
    NSE-->>Script: Symbol list (500 stocks)
    
    loop Each Symbol
        Script->>YF: Download OHLCV (200 days)
        YF-->>Script: Price data
    end
    
    Script->>Script: Merge with existing CSV
    Script->>Script: Remove duplicates
    Script->>Repo: Commit nifty500_ohlcv.csv
    Script->>Repo: Commit metadata.json
```

### Workflow Schedule (IST)
| Time | Purpose |
|------|---------|
| 11:00 AM | Early market scan |
| 3:15 PM | Near-close confirmation |

---

## Core Logic & Algorithms

### Data Flow Diagram

```mermaid
flowchart TD
    A[Start Scanner] --> B[Load Nifty 500 Symbols]
    B --> C{Data Source?}
    C -->|CSV Available| D[Read from CSV<br/>Instant Load]
    C -->|No CSV| E[Fetch from yfinance<br/>~80 min for 500 stocks]
    
    D --> F[Loop: Each Stock]
    E --> F
    
    F --> G[calculate_indicators]
    G --> H{Pass Safety Filters?}
    
    H -->|Price < SMA50| I[SKIP: Downtrend]
    H -->|RSI > 70| J[SKIP: Overbought]
    H -->|Low Turnover| K[SKIP: Illiquid]
    H -->|All Pass| L[Calculate Score]
    
    L --> M[FormulaFactory.generate_all]
    M --> N{Score > 0?}
    
    N -->|Yes| O[Add to Results]
    N -->|No| P[SKIP: Weak Signal]
    
    O --> Q[Sort by Score DESC]
    Q --> R[Output Top 20]
```

---

## Technical Indicators

### Indicators Calculated (in `TechnicalCore.calculate_indicators`)

| Indicator | Formula | Purpose |
|-----------|---------|---------|
| **RSI** | 100 - (100 / (1 + RS)) | Momentum oscillator (avoid >70) |
| **EMA** | Exponential weighted mean | Trend direction |
| **ATR** | Average True Range | Volatility measurement |
| **Hurst Exponent** | Polyfit on log-log scale | Trend persistence (>0.5 = trending) |
| **SMA 50** | 50-day simple moving average | Primary trend filter |
| **Volume Ratio** | Current Vol / Avg Vol | Institutional activity |
| **ROC (Rate of Change)** | (Current - Past) / Past × 100 | Price momentum |
| **Bollinger Squeeze** | (BB_Std × 4) / BB_Mid | Volatility compression |
| **TD Sequential Count** | Consecutive days > 4-day prior | Exhaustion counter |

### Indicator Calculation Code Flow

```mermaid
graph TD
    A[Raw OHLCV Data] --> B[clean_yf_data]
    B --> C{Valid Data?}
    C -->|No| D[Return None]
    C -->|Yes| E[Extract Prices & Volumes]
    
    E --> F[Performance Metrics]
    F --> F1["r_s: Short Return (3-5 days)"]
    F --> F2["r_m: Medium Return (10-21 days)"]
    F --> F3["r_l: Long Return (21-63 days)"]
    
    E --> G[Volatility]
    G --> G1[short_vol: 5-day annualized]
    G --> G2[medium_vol: 10-day annualized]
    
    E --> H[Technical Signals]
    H --> H1[RSI: 14-21 period]
    H --> H2[EMA: Distance from average]
    H --> H3[ATR: True range average]
    H --> H4[Hurst: Trend persistence]
    
    E --> I[Volume Analysis]
    I --> I1[vol_ratio: Recent vs Average]
    I --> I2[avg_vol: Turnover value]
    
    F1 & F2 & F3 & G1 & G2 & H1 & H2 & H3 & H4 & I1 & I2 --> J[Return Metrics Dict]
```

---

## Scoring Engine (Filter 1)

### The Championship Formula

The core scoring formula (historically validated with **+32.8% ROI**):

```python
Score = (Momentum_20 × 100) + (Volume_Intensity × 2.0)
```

Where:
- **Momentum_20** = 20-day price return (r_l / 100)
- **Volume_Intensity** = Current volume / 42-day average volume × 2.0

### Why This Formula Works

```mermaid
pie title Score Composition
    "Momentum (33%)" : 33
    "Volume Intensity (66%)" : 66
```

| Component | Weight | Rationale |
|-----------|--------|-----------|
| **Momentum** | 33% | Confirms the stock is already moving up |
| **Volume** | 66% | Institutional footprint - money flow confirmation |

> **Key Insight:** A price move WITHOUT volume = potential trap  
> A price move WITH massive volume = confirmed institutional trend

---

## Safety Filters

### Filter Sequence (The Shield)

```mermaid
flowchart TD
    A[Stock Candidate] --> B{Price > SMA 50?}
    B -->|No| C[REJECT: Downtrend<br/>Never catch falling knives]
    B -->|Yes| D{RSI < 70?}
    
    D -->|No| E[REJECT: Overbought<br/>Avoid buying tops]
    D -->|Yes| F{Turnover > Min?}
    
    F -->|No| G[REJECT: Illiquid<br/>Exit risk too high]
    F -->|Yes| H{Safe Mode?}
    
    H -->|Yes| I{0 <= Short Return <= 10%?}
    I -->|No| J[REJECT: Too Hot<br/>Cooling filter]
    I -->|Yes| K[PASS: Valid Candidate]
    
    H -->|No| K
```

### Filter Statistics from Actual Scan

| Filter | Typical Rejection % | Purpose |
|--------|---------------------|---------|
| **Trend (SMA50)** | ~30-40% | Avoids downtrends |
| **Exhaustion (RSI)** | ~10-15% | Avoids buying peaks |
| **Liquidity (Turnover)** | ~20-30% | Ensures tradability |
| **Cooling (Safe Mode)** | ~5-10% | Prevents FOMO entries |

---

## Backtesting Methodology

### Types of Backtests Conducted

| Backtest Version | Type | Hold Period | SL/TP | Special Features |
|------------------|------|-------------|-------|------------------|
| **V10 (2025)** | Weekly scanning | 10 days | 10% SL, 20% TP | Random Forest brain integration |
| **Brutal Gauntlet** | Stress test | 5 days | N/A | 0.5% slippage, real friction |
| **V11-V30** | Iterative refinement | Variable | Variable | Testing parameter variations |
| **Infinite Hold** | Buy & hold reference | ∞ | None | Benchmark comparison |
| **Multi-TF** | Multiple timeframes | 3-30 days | Dynamic | Timeframe optimization |

### Backtest Architecture

```mermaid
graph TB
    subgraph "Data Preparation"
        A[Historical OHLCV<br/>backtest_v4_data.pkl] --> B[Date Range Selection]
        B --> C[Symbol Universe<br/>Random 50 per week]
    end
    
    subgraph "Signal Generation"
        C --> D[TechnicalCore.calculate_indicators]
        D --> E[FormulaFactory.generate_all]
        E --> F{Score > Threshold?}
        F -->|Yes| G[Add to Candidates]
        F -->|No| H[Skip]
    end
    
    subgraph "Position Management"
        G --> I[Sort by Score DESC]
        I --> J[Buy Top N<br/>Max 5 positions]
        J --> K[Track Daily P&L]
        K --> L{Exit Condition?}
        L -->|SL Hit| M[Exit with Loss]
        L -->|TP Hit| N[Exit with Profit]
        L -->|Time Limit| O[Exit at Market]
        L -->|No| K
    end
    
    subgraph "Performance Metrics"
        M & N & O --> P[Calculate ROI]
        P --> Q[Win Rate]
        P --> R[Max Drawdown]
        P --> S[Calmar Ratio]
    end
```

---

## Backtest Results

### V10 Verification Backtest (2025)

**Configuration:**
- Period: Jan 2025 - Dec 2025
- Hold Period: 10 days
- Stop Loss: 10%
- Take Profit: 20%
- Scanning: Weekly (Mondays)

**Results Summary:**

| Metric | Value |
|--------|-------|
| Total Trades | 68 |
| Win Rate | ~40% |
| Exit by SL | 11 trades (-10% each) |
| Exit by TP | 0 trades |
| Exit by Time | 57 trades |

**Sample Trades (from `results_v10_2025.csv`):**

| Date | Symbol | Return | Exit Reason |
|------|--------|--------|-------------|
| 2025-01-13 | TRENT | -11.1% | Stop Loss |
| 2025-03-26 | TRENT | +10.9% | Time Exit |
| 2025-04-25 | TITAN | +11.3% | Time Exit |
| 2025-05-14 | ZENTEC | +11.8% | Time Exit |
| 2025-11-18 | DATAPATTNS | +13.2% | Time Exit |

---

### Brutal Gauntlet Stress Test (2025)

**Configuration:**
- Slippage: 0.5% total friction per trade
- Entry/Exit Applied: 0.25% each side
- Weekly Rotation: Top 3 stocks per week

**Results Summary:**

| Metric | Value |
|--------|-------|
| Total Trades | 139 |
| Best Trade | NETWEB +20.7% |
| Worst Trade | POWERINDIA -14.6% |
| Avg Weekly Return | Variable |

**Top Performers:**

| Symbol | Score | Return |
|--------|-------|--------|
| NETWEB | 59.56 | +20.7% |
| COCHINSHIP | 23.49 | +19.1% |
| ATHERENERG | 23.95 | +18.3% |
| DATAPATTNS | 42.49 | +16.8% |
| POWERINDIA | 20.04 | +17.1% |

---

### Performance Comparison

```mermaid
xychart-beta
    title "Strategy Performance Comparison"
    x-axis [V10-Backtest, Brutal-Gauntlet, Buy-Hold-Nifty]
    y-axis "Annual ROI %" 0 --> 40
    bar [32.8, 25.0, 15.0]
```

---

## Key  Q&A

### 1. "How does the scanner fetch data?"

**Answer:**
The system has a **dual-source architecture**:

1. **Primary (Cloud/Production):** Pre-fetched CSV from GitHub Actions
   - Stored in `data/nifty500_ohlcv.csv` (~5.8 MB)
   - Updated twice daily (11 AM, 3:15 PM IST)
   - 200 days of history for 500 stocks

2. **Fallback (Development):** Direct yfinance API
   - Real-time data from Yahoo Finance
   - Used when CSV unavailable
   - Slower (80+ minutes for full scan)

---

### 2. "Where is the data stored?"

**Answer:**
```
data/
├── nifty500_ohlcv.csv    # Main OHLCV storage (Symbol, Date, OHLC, Volume)
├── metadata.json         # {"last_updated": "2026-01-31", "total_stocks": 500}
└── nifty500_live.csv     # Hourly refresh variant
```

Additionally:
- `trade_history.csv`: All scan results over time
- `backtest_v4_data.pkl`: 8.4 MB pickle cache for backtesting

---

### 3. "Explain the scoring formula"

**Answer:**
```python
Score = (r_l / 100 × 100) + (vol_ratio × 2.0)
      = Momentum_20 + Volume_Bonus
```

- **r_l**: 21-day price return (%)
- **vol_ratio**: Today's volume ÷ 42-day average

**Why 2× volume weight?**
- Volume is the "institutional footprint"
- High volume + high momentum = confirmed institutional accumulation
- High momentum alone = potential retail trap

---

### 4. "What safety measures prevent bad trades?"

**Answer:**
Four-layer protection:

1. **Trend Guard (SMA 50)**: Only buy if price > 50-day average
2. **Exhaustion Guard (RSI < 70)**: Avoid overbought stocks
3. **Liquidity Guard (Turnover)**: Minimum ₹100M+ daily turnover
4. **Cooling Guard (Safe Mode)**: Short-term gain 0-10% only

---

### 5. "What types of backtests did you run?"

**Answer:**

| Type | Purpose | Result |
|------|---------|--------|
| **V10 Verification** | Validate RF model on 2025 data | 68 trades, 40% win rate |
| **Brutal Gauntlet** | Stress test with slippage | 139 trades, realistic friction |
| **Multi-Timeframe** | Optimize hold periods | 1-2 weeks optimal |
| **Infinite Hold** | Buy-and-hold benchmark | Lower than momentum |

---

### 6. "What is the Hurst Exponent and why use it?"

**Answer:**
```python
Hurst = polyfit(log(lags), log(tau), 1)[0] × 2.0
```

- **H > 0.5**: Trending market (momentum works)
- **H = 0.5**: Random walk (avoid trading)
- **H < 0.5**: Mean-reverting (contrarian works)

The scanner uses Hurst to detect if a stock is in a **trending regime** where momentum strategies succeed.

---

### 7. "Why GitHub Actions for data fetching?"

**Answer:**
**Problem:** Streamlit Cloud IPs are often blocked by Yahoo Finance
**Solution:** Pre-fetch data on GitHub's infrastructure

| Benefit | Explanation |
|---------|-------------|
| No IP blocks | GitHub Actions has different IP range |
| Instant loads | CSV reads in <1 second |
| Consistent data | All users see identical prices |
| Free tier | 2000 min/month, using ~360 min |

---

### 8. "How does position sizing work?"

**Answer:**
```python
allocation_per_stock = capital × 0.10  # 10% max per position
qty = int(allocation_per_stock / spot_price)
```

If capital = ₹1,00,000:
- Max allocation = ₹10,000 per stock
- For a ₹500 stock → Buy 20 shares
- For a ₹5,000 stock → Buy 2 shares

---

### 9. "What are Entry Range and Exit Target?"

**Answer:**
Dynamic ranges based on ATR (Average True Range):

```python
entry_buffer = max(spot × 0.01, ATR × 0.25)  # If high score
entry_range = f"{spot} - {spot + entry_buffer}"

exit_buffer = max(target × 0.02, ATR × 0.5)
exit_range = f"{target} - {target + exit_buffer}"
```

This gives traders a **zone** rather than a fixed price, accounting for intraday volatility.

---

### 10. "What's the recommended trading time?"

**Answer:**
**The 3:15 PM Rule:**

| Time | Action | Why |
|------|--------|-----|
| 9:15-10:00 AM | WAIT | Fake-out zone, institutions selling |
| 12:00 PM | MONITOR | Trend forming but not confirmed |
| **3:15-3:25 PM** | **ENTER** | Institutions hold overnight, 95% confirmed |
| After Market | PLAN | Build next-day watchlist |

---

## Technical Architecture Summary

```mermaid
mindmap
  root((Alpha-Zeta<br/>Scanner))
    Data Pipeline
      GitHub Actions
      yfinance API
      NSE CSV
      Local PKL Cache
    Technical Core
      RSI
      EMA
      ATR
      Hurst Exponent
      SMA 50
      Volume Ratio
    Scoring Engine
      Filter 1 Ensemble
      Momentum × 100
      Volume × 2.0
    Safety Filters
      Trend Guard
      RSI Guard
      Liquidity Guard
      Cooling Guard
    Output
      Streamlit Dashboard
      CLI Interface
      CSV Export
      Trade History
    Backtesting
      V10 Verification
      Brutal Gauntlet
      Multi-Timeframe
      30+ Variations
```

---

## Quick Reference Card

| Question | One-Line Answer |
|----------|-----------------|
| Data Source | NSE + yfinance → CSV → App |
| Data Storage | `data/nifty500_ohlcv.csv` (5.8 MB) |
| Core Formula | Score = Momentum×100 + Volume×2.0 |
| Key Filters | SMA50, RSI<70, Turnover, Cooling |
| Backtest ROI | +32.8% (1-2 week swings) |
| Best Timeframe | 3:15 PM entry, 1-2 week hold |
| Backtest Types | V10, Brutal Gauntlet, Multi-TF |
| Trade Count | 68 (V10), 139 (Gauntlet) |

---

*Document generated for interview preparation. All code and results from Alpha_Zeta_Super_Scanner project.*
