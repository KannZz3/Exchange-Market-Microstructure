<div align="center">

# Parsing & Analyzing Raw Nasdaq TotalView-ITCH 5.0

### Exchange-Level Limit Order Book Reconstruction & Market Microstructure Analysis

**Raw binary feed → ITCH events → order lifecycle → limit order book → market state → short-horizon price dynamics**

`Python` · `Nasdaq ITCH 5.0` · `Limit Order Book` · `Market Microstructure`

</div>

---

## Overview

This project builds an end-to-end exchange-level market-data pipeline directly from raw **Nasdaq TotalView-ITCH 5.0** binary messages.

Instead of starting from preprocessed OHLC, trade, or quote data, the market state is reconstructed from the underlying sequence of order-level exchange events.

The first-stage implementation studies:

> **AAPL · October 18, 2019 · Nasdaq TotalView-ITCH 5.0**

The objective is deliberately narrow: first validate the complete exchange-level reconstruction pipeline on one liquid stock and one trading day, then extend the framework to larger multi-stock and multi-day datasets.

### Core questions

1. Can a raw binary exchange feed be decoded into a valid limit order book?
2. What economically meaningful market-state variables can be reconstructed from it?
3. Does displayed liquidity imbalance contain short-horizon price information?
4. Is that statistical predictability large enough to be economically tradable?

The project explicitly separates:

> **Statistical predictability ≠ executable trading profitability**

---

## Key Results at a Glance

| Result | Value |
|---|---:|
| Raw ITCH messages processed | **302,347,067** |
| AAPL-related messages | **≈ 1.67 million** |
| Regular-session snapshots | **23,400** |
| Missing best bid / ask | **0 / 0** |
| Crossed or locked book observations | **0 / 0** |
| Mean quoted spread | **0.8289 bps** |
| Median quoted spread | **0.8475 bps** |
| Mean best-bid depth | **207.2 shares** |
| Mean best-ask depth | **271.3 shares** |
| OBI vs. future 1s return correlation | **0.1016** |
| OBI vs. future 5s return correlation | **0.0626** |
| Microprice deviation vs. future 1s return | **0.1161** |
| Continuous-market volume | **4,495,438 shares** |
| Continuous-market VWAP | **$235.9413** |
| Opening Cross | **$234.53 · 1,664,440 shares** |
| Closing Cross | **$236.41 · 1,972,810 shares** |

The central empirical result is a nearly monotonic relationship between **top-of-book imbalance** and subsequent short-horizon midpoint returns.

However, the magnitude of the predicted price movement is substantially smaller than the bid-ask spread, so the result is better interpreted as a **market-making / execution feature** than as standalone aggressive directional alpha.

---

## Data

| Field | Value |
|---|---|
| Exchange | Nasdaq |
| Protocol | TotalView-ITCH 5.0 |
| Date | 2019-10-18 |
| Raw file | `S101819-v50.txt.gz` |
| Target security | AAPL |
| Daily Stock Locate | `14` |

The raw dataset is a gzip-compressed Nasdaq BinaryFILE.

The file is processed as a **stream** rather than decompressed into memory:

```text
2-byte message length
        ↓
ITCH payload
        ↓
Message type
        ↓
Structured exchange event
```

The raw file is not committed to this repository because of its size.

---

## Pipeline Architecture

```text
S101819-v50.txt.gz
        │
        ▼
┌─────────────────────────┐
│ BinaryFILE Reader       │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ ITCH Message Decoder    │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ AAPL Stock-Locate Filter│
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ Order Reference Manager │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ Displayed Limit Order   │
│ Book Reconstruction     │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ 1-Second Market States  │
├─────────────────────────┤
│ Best Bid / Ask          │
│ Mid-Price               │
│ Quoted Spread           │
│ Best-Level Depth        │
│ Order Book Imbalance    │
│ Microprice              │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ Trade / Auction Extract │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ Microstructure Analysis │
└─────────────────────────┘
```

### Design choices

**Streaming parsing**  
The compressed binary file is read message by message, keeping memory usage bounded.

**Integer price representation**  
ITCH `Price(4)` values remain integer-valued internally. Decimal conversion occurs only when analytical outputs are produced.

**Pre-market reconstruction**  
Orders arriving before 09:30 are processed because they may remain active when the regular session begins.

**Regular-session sampling**  
The reconstructed book is sampled once per second over:

`09:30:00 ≤ t < 16:00:00`

giving exactly:

`6.5 × 3,600 = 23,400`

market-state snapshots.

**Auction preservation**  
Cross Trade (`Q`) messages are retained independently of the continuous-book cutoff so both the Opening Cross and post-16:00 Closing Cross are captured.

---

## From Exchange Events to Market State

A limit order book is not a static dataset.

It is a state generated by the ordered sequence of exchange events.

For every active order, the reconstruction maintains:

```text
Order Reference Number
Side
Price
Remaining Shares
```

### Core ITCH messages

| Type | Message | State Effect |
|---|---|---|
| `R` | Stock Directory | Maps Stock Locate to ticker |
| `A` | Add Order | Creates displayed order |
| `F` | Add Order + MPID | Creates displayed order |
| `E` | Order Executed | Reduces remaining quantity |
| `C` | Executed with Price | Reduces quantity + records execution price |
| `X` | Order Cancel | Partially removes quantity |
| `D` | Order Delete | Removes remaining order |
| `U` | Order Replace | Removes old ID and creates new order |
| `P` | Non-displayed Trade | Transaction only; does not alter displayed book |
| `Q` | Cross Trade | Opening / closing / other auction transaction |
| `B` | Broken Trade | Invalidates a reported trade |
| `I` | NOII | Auction imbalance information |

Order Reference Numbers connect modification messages to the corresponding displayed order.

---

## Market-State Variables

Let:

- `B_t` = best bid
- `A_t` = best ask
- `Qᵇ_t` = displayed quantity at best bid
- `Qᵃ_t` = displayed quantity at best ask

### Mid-price

$$M_t=\frac{A_t+B_t}{2}$$

### Quoted spread

$$S_t=A_t-B_t$$

Spread in basis points:

$$S_t^{\mathrm{bps}}=\frac{A_t-B_t}{M_t}\times 10{,}000$$

### Top-of-book imbalance

$$I_t=\frac{Q_t^b-Q_t^a}{Q_t^b+Q_t^a},\qquad -1\le I_t\le1$$

Interpretation:

- `I_t > 0` → greater displayed depth on the bid
- `I_t < 0` → greater displayed depth on the ask

### Microprice

$$MP_t=\frac{A_tQ_t^b+B_tQ_t^a}{Q_t^b+Q_t^a}$$

Unlike the simple midpoint, the microprice incorporates relative displayed liquidity and therefore shifts toward the side of the book with lower available depth.

---

## Reconstruction Validation

The reconstruction is accepted only if the resulting market state is internally consistent.

| Integrity Check | Result |
|---|---:|
| Expected snapshots | 23,400 |
| Actual snapshots | **23,400** |
| Missing best bid | **0** |
| Missing best ask | **0** |
| Negative spread | **0** |
| Locked book | **0** |
| Negative bid depth | **0** |
| Negative ask depth | **0** |
| Broken trade matches | **0** |
| Raw AAPL `Q` messages | **2** |
| Recorded `Q` trades | **2** |
| Opening Cross captured | **1** |
| Closing Cross captured | **1** |

A useful end-to-end consistency condition is:

$$N_Q^{raw}=N_Q^{recorded}=2$$

Both Nasdaq auction crosses are therefore preserved by the final pipeline.

---

## What Generates Exchange Activity?

AAPL generated approximately **1.67 million ITCH messages**.

| Event | Count |
|---|---:|
| Add Order (`A`) | 769,396 |
| Add Order with MPID (`F`) | 2,670 |
| Delete (`D`) | 731,241 |
| Replace (`U`) | 98,898 |
| Execute (`E`) | 54,748 |
| Execute with Price (`C`) | 717 |
| Cancel (`X`) | 1,288 |
| Non-displayed Trade (`P`) | 10,156 |
| Cross Trade (`Q`) | 2 |
| NOII (`I`) | 420 |

The event stream is dominated by **liquidity creation and removal**, not completed transactions.

Approximate shares of AAPL messages:

```text
Add Orders      ≈ 46.1%
Deletes         ≈ 43.8%
Replacements    ≈  5.9%
```

This illustrates a fundamental difference between exchange-level data and conventional trade/OHLC data:

> A large fraction of economically relevant market activity occurs **before a transaction ever takes place**.

---

## Market Characteristics

Across all 23,400 regular-session snapshots:

| Metric | Result |
|---|---:|
| Mean mid-price | **$235.8620** |
| Mean quoted spread | **0.8289 bps** |
| Median quoted spread | **0.8475 bps** |
| Average best-bid depth | **207.2 shares** |
| Average best-ask depth | **271.3 shares** |
| Mean top-of-book imbalance | **−0.0378** |

### Spread distribution

- **32.5%** of snapshots: 1-cent spread
- **47.9%**: 2-cent spread
- **19.5%**: 3 cents or wider

The opening regime is materially different from the remainder of the session:

| Period | Mean Spread |
|---|---:|
| 09:30–09:35 | **≈ 3.09 bps** |
| After 09:35 | **≈ 0.80 bps** |

The first minutes therefore contain substantially greater price-discovery and liquidity-adjustment activity than the normal continuous session.

---

# Empirical Result: Order Book Imbalance

Future midpoint return over horizon `h` is defined as:

$$r_{t,t+h}=\left(\frac{M_{t+h}}{M_t}-1\right)\times10{,}000\ \mathrm{bps}$$

Current top-of-book imbalance is divided into five states.

| State | N | Mean OBI | Future 1s | Future 5s |
|---|---:|---:|---:|---:|
| **Strong Sell** | 4,336 | −0.8315 | **−0.0621 bps** | **−0.0655 bps** |
| **Sell** | 5,201 | −0.3783 | **−0.0535 bps** | **−0.0584 bps** |
| **Neutral** | 6,142 | +0.0079 | **+0.0131 bps** | **+0.0219 bps** |
| **Buy** | 4,166 | +0.3923 | **+0.0575 bps** | **+0.0982 bps** |
| **Strong Buy** | 3,554 | +0.8452 | **+0.0846 bps** | **+0.1259 bps** |

The conditional means exhibit a nearly monotonic ordering:

```text
Strong Sell
    <
Sell
    <
Neutral
    <
Buy
    <
Strong Buy
```

### Interpretation

A stronger displayed bid-side imbalance is associated with a more positive subsequent midpoint movement.

A stronger ask-side imbalance is associated with a more negative subsequent movement.

However:

> **The median one-second return is zero in every imbalance bucket.**

Therefore, order-book imbalance does not deterministically forecast the next price move.

It changes the **conditional distribution** of short-horizon outcomes.

---

## Predictive Strength and Decay

### Order-book imbalance

$$Corr(I_t,r_{t,t+1s})=0.1016$$

$$Corr(I_t,r_{t,t+5s})=0.0626$$

Predictive strength decreases as the horizon increases from one to five seconds.

This suggests that a meaningful part of the information contained in the current top-of-book state is **short-lived**.

### Microprice

The microprice contains slightly more one-second information:

$$Corr\left(\frac{MP_t-M_t}{M_t},r_{t,t+1s}\right)=0.1161$$

Hence, in this sample:

```text
Microprice deviation
        >
Raw top-of-book imbalance
```

in terms of contemporaneous short-horizon correlation.

The relationship is measurable, but most one-second price variation remains unexplained by the current top-of-book state.

---

## Continuous Trading and Auctions

### Continuous Market

| Metric | Result |
|---|---:|
| Printable non-cross trades | **65,046** |
| Volume | **4,495,438 shares** |
| VWAP | **$235.9413** |

### Nasdaq Opening Cross

| Metric | Result |
|---|---:|
| Price | **$234.53** |
| Volume | **1,664,440 shares** |

### Nasdaq Closing Cross

| Metric | Result |
|---|---:|
| Price | **$236.41** |
| Volume | **1,972,810 shares** |

Auction transactions are intentionally separated from continuous-market volume because `Q` messages represent discrete cross mechanisms rather than ordinary continuous executions.

The auction volumes also illustrate that economically significant liquidity can concentrate in discrete exchange events.

---

# Statistical Edge vs. Trading Edge

The strongest positive imbalance bucket produces an average one-second midpoint movement of approximately:

$$0.0846\ \mathrm{bps}$$

The average quoted spread is:

$$0.8289\ \mathrm{bps}$$

Therefore:

$$\frac{0.0846}{0.8289}\approx10\%$$

The average predicted directional move is only about **one-tenth of the quoted spread**.

This is the crucial economic result.

Even before including:

- fees,
- rebates,
- latency,
- queue position,
- partial fills,
- slippage,
- market impact,
- adverse selection,

the expected midpoint movement is materially smaller than the cost of aggressively crossing the market.

### What the signal is *not*

It is not evidence of a standalone:

```text
OBI > threshold
→ send market order
→ earn positive expected return
```

strategy.

### Where it may matter

#### 1. Quote Skew

Use imbalance or microprice to move internal fair value away from the simple midpoint.

#### 2. Adverse-Selection Control

A passive seller may reduce or reprice ask exposure when the book becomes strongly bid-heavy, while a passive buyer may react symmetrically to strong ask-side pressure.

#### 3. Execution Timing

Short-lived liquidity imbalance may help determine whether an execution algorithm should become more or less aggressive.

These are research hypotheses—not demonstrated profitable strategies.

---

# Main Takeaway

The main contribution is not a single correlation coefficient.

It is the complete transformation:

```text
Raw Exchange Bytes
        ↓
Exchange Events
        ↓
Individual Orders
        ↓
Limit Order Book
        ↓
Market State
        ↓
Predictive Information
        ↓
Economic Interpretation
```

For AAPL on October 18, 2019, the evidence supports five conclusions:

1. **Raw ITCH messages can be reconstructed into an internally consistent displayed limit order book.**
2. **Top-of-book imbalance contains measurable short-horizon directional information.**
3. **Microprice contains slightly stronger one-second information than raw imbalance.**
4. **The predictive effect is short-lived and materially smaller than the bid-ask spread.**
5. **Its most plausible use is therefore in market making and execution rather than standalone aggressive directional trading.**

---

## Limitations

This is intentionally a first-stage study.

| Limitation | Implication |
|---|---|
| One stock | No cross-sectional generalization |
| One day | No temporal robustness claim |
| Nasdaq only | Not consolidated U.S. market activity |
| 1-second sampling | Sub-second information is compressed |
| Top-of-book analysis | Deeper LOB information is not yet used |
| No queue model | Passive execution probability is unknown |
| No fees / rebates | Net trading economics are incomplete |
| No latency model | Real-time implementability is not established |
| No market impact | Capacity is not evaluated |

The empirical relationship should therefore be interpreted as a **microstructure result**, not proof of a deployable trading strategy.

---

## Next Research Stages

### Phase I.5 — Deeper ITCH Microstructure

Extend the current event-level framework to:

- `100 ms / 500 ms / 1 s` horizons
- event-time rather than clock-time analysis
- multi-level LOB depth
- Order Flow Imbalance (OFI)
- signal-decay curves
- queue-position modeling
- execution-aware PnL
- adverse-selection measurement
- NOII and auction dynamics

The central question becomes:

> **At what horizon does exchange-level order-book information contain the most economic value?**

### Phase II — Multi-Stock / Multi-Day Validation

Expand:

```text
1 stock × 1 day
```

into:

```text
N stocks × T days
```

Large-scale datasets such as WRDS TAQ can then be used to study related quantities including:

- quoted spread
- effective spread
- realized spread
- price impact
- volume
- volatility
- NBBO dynamics

ITCH and TAQ contain different information sets, so TAQ should be used for **large-sample external validation**, not treated as a substitute for full order-level ITCH reconstruction.

---

## Repository Structure

```text
.
├── Part I — Exchange-level Engineering.ipynb
├── README.md
│
├── S101819-v50.txt.gz          # local only; not committed
│
└── output/
    ├── message_stats.csv
    ├── market_state.csv
    ├── trades.csv
    ├── trade_summary.csv
    ├── validation_summary.csv
    ├── imbalance_bucket_stats.csv
    │
    └── figures/
        ├── 01_midprice.png
        ├── 02_spread.png
        ├── 03_imbalance.png
        └── 04_imbalance_future_return.png
```

> The notebook contains the complete reproducible workflow from binary decoding through empirical microstructure analysis.

---

## Reproduce the Analysis

### Requirements

```bash
pip install numpy pandas matplotlib jupyter
```

### Run

1. Obtain `S101819-v50.txt.gz`.
2. Place it in the project directory.
3. Launch:

```bash
jupyter lab
```

4. Open:

```text
Part I — Exchange-level Engineering.ipynb
```

5. Run the notebook from top to bottom.

Expected pipeline:

```text
Raw BinaryFILE
    ↓
AAPL Identification
    ↓
LOB Reconstruction
    ↓
23,400 One-Second Market States
    ↓
Trade & Auction Extraction
    ↓
Integrity Validation
    ↓
Microstructure Analysis
    ↓
CSV + Figure Outputs
```

---

## Disclaimer

This repository is an educational and research project in exchange-level market-data engineering and market microstructure.

Results are based on a single historical stock-day sample and do **not** constitute investment advice or evidence of a deployable trading strategy.
