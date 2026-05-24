# brazilgrid-loop-bridge

Public, read-only bridge for the **forecast-mega-loop** running inside the
private BrazilGrid monorepo. This repo carries only the artifacts that need
to be visible across sessions and to outside observers — nothing else.

## What lives here

- `iterations/iter_NNNN_<slug>.md` — handoff notes, one per iteration.
- `leaderboard.md` — current best metric per (layer, target).
- `state.json` — snapshot of the loop driver (current iter, active target, budget).

That is it. There is **no model code, no data, no credentials, no clickhouse
queries** in this repo, and there will never be.

## What does NOT live here

- Source code of the loop (`watchdog/`, `sanity_checks/`, `templates/`) — internal.
- Configuration (`config.yaml`) — internal.
- Trained models (`*.joblib`, `*.pkl`) — never sync.
- Datasets (`*.parquet`, `*.csv`) — never sync.
- Credentials, tokens, secrets of any kind — never sync.

## Flow

```
private monorepo  ──(watchdog/heartbeat.sh at end of each iter)──>  this public bridge
        ▲                                                                    │
        │                                                                    │
        └────────────────── never reads from here ───────────────────────────┘
```

The bridge is **strictly one-way**: private → public. The private loop never
pulls from this repo. If a manual edit is made here it will be overwritten
on the next iteration.

## Why bother with a public bridge?

So a second Claude Code session — or a human reading the loop progress —
can see the latest handoff/leaderboard/state without needing access to the
private monorepo, and without me having to copy-paste anything.

## Source of truth

The authoritative copy of every file in this repo lives at
`loops/forecast-mega-loop/` inside the private BrazilGrid monorepo.
