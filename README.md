# IUPPITER Trading Docs

Operational documentation for the IUPPITER MEXC Volume-MM bot — verified safe operating recipes, market analysis playbooks, and incident retrospectives.

## Available Languages

| Language | Path |
|---|---|
| 🇰🇷 한국어 | [`ko/`](./ko/) |
| 🇺🇸 English | [`en/`](./en/) |
| 🇯🇵 日本語 | [`ja/`](./ja/) |
| 🇨🇳 简体中文 | [`zh-CN/`](./zh-CN/) |

## Documents

Each language directory contains the same 4 documents:

| Document | Purpose |
|---|---|
| `operation-guide.md` | General operation manual — start, monitor, stop, alarms |
| `safe-recipe.md` | Self-cross only safe setting (verified, USDT preservation focused) |
| `market-analysis.md` | 4-indicator market flow analysis before bot startup or climb |
| `incident-2026-04-24.md` | Incident retrospective + asymmetric defense system narrative |

## Quick Links — English

- [Operation Guide](./en/operation-guide.md)
- [Safe Recipe](./en/safe-recipe.md)
- [Market Analysis](./en/market-analysis.md)
- [Incident Retrospective](./en/incident-2026-04-24.md)

## Status

- **Verified through**: 2026-04-26 (canary 5fc07f81, 21h live observation, USDT preserved $642 → $658.88)
- **Defense system**: 5-layer asymmetric (Mode FSM + Asymmetric Ladder + Circuit Breakers + Symbol Adaptive Profile + Tick History Aggregator)
- **License**: MIT — feel free to fork, adapt, contribute back

## Contributing

PRs welcome — translation improvements, additional cases, troubleshooting tips.

Edit any markdown file and open a PR. For brand new content (e.g., FAQ, new recipe), please open an issue first to discuss.
