# Causality & Chronological Data Integrity

In systematic quantitative research, the validity of any historical simulation rests entirely on its chronological integrity. A backtest that inadvertently accesses information from the future—even by a single millisecond—will produce artificially inflated performance metrics, rendering the research invalid for live capital allocation.

This document outlines the chronological pitfalls discovered during the framework's evolution and the architectural solutions implemented in the KANG infrastructure to guarantee zero look-ahead bias.

---

## 1. The Anatomy of Look-Ahead Bias (The $T_{open}$ vs $T_{close}$ Paradox)

During the initial development of the legacy system (Alchemist CE), severe discrepancies emerged between historical backtest performance and live forward-testing parity. A forensic audit of the execution pipeline revealed a critical structural flaw: **Temporal Misalignment**.

In standard OHLCV datasets, the timestamp associated with a candle represents the _open_ of the period ($T_{open}$).

- **The Flaw:** The legacy engine set its temporal anchor at $T_{open}$ (e.g., `08:00`) but evaluated the setup validity using the `close` and `high` of the candle, which do not exist until $T_{close}$ (e.g., `12:00`).
- **The Leakage:** Because the engine confirmed the setup using future information, it placed simulated limit orders at `08:00`. These orders frequently filled on the intraday volatility of the forming setup candle itself—an execution that is mathematically impossible in a live environment, as the setup is not confirmed until `12:00`.

## 2. The $T_{close}$ Temporal Anchoring Solution

To eradicate this execution paradox, the KANG research engine enforces strict **Temporal Anchoring**.

The framework decouples the _timestamp of the market state_ from the _timestamp of the decision_.

1. **Setup Generation:** The engine evaluates the Higher Time Frame (HTF) array and identifies the terminal candle $D_i$.
2. **Decision Time Anchor:** The engine calculates the exact millisecond the candle finalizes:
   `decision_time = T_close(D_i) = T_open(D_i) + TimeDelta(HTF_Period)`
3. **Causal Lock:** All subsequent lower-timeframe analysis, indicator calculations, and trade simulations are strictly bound to `decision_time`. The engine is mathematically prohibited from accessing data prior to $T_{close}$.

## 3. Causal Data Slicing & Terminal Candle Blindness

A common issue in array-based market scanners is "Terminal Candle Blindness," where loops iterating through `range(1, len(df) - 1)` fail to process the currently closing candle, leading to off-by-one execution errors.

The KANG framework resolves this by utilizing vectorized NumPy boundaries:

```python
# Slicing Lower Timeframes strictly up to the HTF Decision Time
ltf_idx = np.searchsorted(ltf_times, np.datetime64(decision_time), side="right")
ltf_slice = df_ltf.iloc[:ltf_idx]
```

Using `np.searchsorted` with `side="right"` guarantees that the lower timeframe arrays (e.g., $H4$, $H1$, $M15$) passed to the refinement engines contain the exact fractal structure that formed _during_ the HTF setup candle, and absolutely nothing beyond it.

## 4. Forward Execution Simulation

To capture rich execution metadata (Maximum Favorable/Adverse Excursion) without assuming instantaneous fills, the framework implements a strictly forward-walking simulation loop.

- **Initialization:** The simulation array `m15_future` begins exactly at `decision_time`.
- **Tick-by-Tick Approximation:** The engine iterates through the future array chronologically. It tracks whether the High or Low of the current $M15$ bar breaches the Entry coordinate _before_ it breaches the Target coordinate.
- **Missed TP Tracking:** If the target is hit before the entry limit order is triggered, the simulation immediately halts and flags the outcome as `MISSED_HIT_TP`, preventing the engine from filling a stale order days after the trade idea has already played out.

Causal Timeline Diagram:

- [Causal Execution Timeline](../diagrams/timeline_causality.png)
- A flowchart diagram showing T_open (08:00) -> Candle Formation -> T_close (12:00) -> Order Placed -> Forward Simulation Time -> Trade Exit. This visual proves the mathematical impossibility of past-filling.

## 5. Timezone & Broker Server Parity

Historical Parquet datasets often normalize timestamps to UTC. However, live execution via MetaTrader 5 relies on the broker's specific server time, which is heavily impacted by regional Daylight Saving Time (DST) shifts (e.g., New York EST/EDT transitions).

Failing to align backtest data with live broker server time causes Session Killzones and Daily candle boundaries to drift by 1-2 hours, destroying spatial geometry. The framework utilizes a dedicated `time_utils` module that applies dynamic `pytz` localization to ensure historical Parquet arrays are correctly offset to match the live execution environment seamlessly.

---
