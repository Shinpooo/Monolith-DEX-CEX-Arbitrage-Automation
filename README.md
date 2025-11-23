# Monolith-DEX-CEX-Arbitrage-Automation

## 1. Brief Vision

The goal of this task is to run delta-neutral strategies that capture spreads in a safe, optimized, and trackable way. We focus on predictable execution and maintaining the hedge rather than chasing every micro-second latency edge. The plan is to ship a small MVP fast, test it with limited capital, collect feedback, and iterate based on real execution results.

## 2. Roadmap

### MVP Core infra (2 weeks)
- Strategy clarification (position building, unwind logic, signal definition, order sizing, liquidity checks, config scope).
- MVP minimal strategy engine:
  1. Setup exchange connectivity using CCXT (python): stream orderbooks, private trades/orders, and send orders to Hyperliquid and Bybit.  
  2. Implement a first simple algo using the strategy rules + this connectivity. Single-process, no microservices, in-process DB storage for fills/positions.

### Experiment MVP algo (2 weeks)
- Start with Hyperliquid–Bybit perp/perp only.
- Use existing third-party dashboards (funding, OI, depth) to detect opportunities.
- Trade with very limited capital to validate real execution quality. Check what breaks (liquidity sizing, signals, errors, margin events).
- Monitor pnl, inventory, margin, liquidation ratio from exchange UIs. Manual intervention if something is wrong (unhedged leg, margin spikes).
- Iterate to improve stability, tracked metrics, error handling, liquidity logic. Increase capital progressively.

### Scale the core (2 weeks)
Once MVP shows acceptable stability:
- Split into microservices:
  - Order management service (execution + fills/orders streaming)
  - Market data service (orderbooks + public trades)
  - Strategy engine service (generates trade instructions based on live data)
- Introduce a message queue as a central communication layer between components.

### Periphery work (in parallel) (3+ weeks)
- Data service: consume messages from the MQ and persist to DB.
- In-house opportunity dashboard: internal data (historical funding, live bbo, OI).
- Account tracking module: strat pnl, inventory, margin, positions. Persist + Grafana dashboards.
- Operator UI: start/stop/restart strategies with config. Tradebook (live private fills)

### Next
- Add new venues (exchange connectivity)
- Spot/perp/margin trading
- Improve strategy (quant research, auto-adjusting parameters)
- Latency optimization (colocation, language choice, custom connectors)
- Cross-exchange manual trading interface
- Markouts to evaluate fill quality / toxic flow behaviors

## 3. Prioritization
- **MVP:** Focus on perp–perp arbitrage (Hyperliquid / Bybit) with limited capital and a simple strategy. This validates execution, liquidity checks, and failure scenarios without overengineering.
- **Later Iterations:** Scale architecture, expand venues, add spot–perp/margin, implement dynamic margin management. These features add complexity and risk, so they come after we prove the core loop works.

---

## 4. Risks & Mitigation Plan

| **Risk** | **Severity** | **Mitigation** |
|--------|-----------|------------|
| Slippage / low depth | High | Depth-based sizing, IOC limit order types for taking + retries, avoid pure market orders. |
| Leg imbalance | High | Immediately neutralize (close opposite leg) or rebalance via depth/spread |
| Margin/OI constraints breached | Medium | Alerts + auto-reduction when thresholds hit (reduce exposure, close risky side). |
| Data feed desync / stale orderbooks | Medium | Heartbeats to monitor process state. |
| Auto deleverage (ADL) | High | Difficult to reproduce, detect via execution/position stream, instantly unwind opposite leg and stop strategy to avoid unhedged exposure. |

---

## 5. Technical Design (High-Level)

The end architecture separates Market Data, Strategy, and execution so each component runs independently and avoids shared-state issues.  
A message queue (NATS/Kafka) sits in the middle and it keeps messages idempotent, ordered, and replayable if something goes wrong or we need to investigate a sequence of trades.  
The strategy logic stays stateless and event-driven: it reacts to orderbooks, fills, etc. and send trading signals without holding any state internally.

The database module lives outside of these services and only consumes from the message bus.  
This avoids trading logic blocking because a DB write is slow or the DB crashes, and it makes troubleshooting easier since the DB just records everything (bbos, depth, erros, fills, positions snapshots, execution logs).

---

## 6. UI / Operator Interface
- **Live Grafana dashboard:** margin health, strategy pnl, balances, active positions.
- **Opportunity dashboard:** funding rates, spreads, historical data, orderbook depth.
- **Operator UI:** start/stop/pause/restart strategies with config; show running state and alerts.
- **Manual trading interface:** emergency hedging when something goes wrong; view open orders + “cancel all”.
- **Tradebook:** live flow of executed trades from the MQ to quickly validate behavior.

---

## 7. Testing & Validation
Testing focuses on the critical failure points: liquidity checks, position caps, OI limits, rebalance triggers, and automatic hedge execution.  
We validate hedge correctness on every trade and alert when deviation crosses a configurable threshold.

---
## 8. Open Questions / Assumptions
- First MVP relies on third-party opportunity dashboard which can provide inaccurate data.
- ADL might not be detected through the position update stream. This may vary from an exchange to another.
- The strategy to build a position briefly described in the assignement looks like a taking strategy. This might not be optimal as taker fees are way higher. This can be improved as being maker on one of the two venues.
- Different quotes, some venues quotes with different stablecoins. Which can hide some additional execution costs as stables are not exactly 1-1.
- Leverage can also be adjusted to optimize returns while increasing liquidiation risk. This was not discussed as 1x Leverage was assumed here.



  
