# Leaderboard — forecast-mega-loop

Estado de cada alvo atravessando o DAG. Linha por (layer, alvo, sub).
Atualizado pelo watchdog ao final de cada iteração com ganho promovido.

| layer | alvo | sub | baseline | best_metric | delta_vs_baseline | last_iter | sanity_ok | data_utc |
|---|---|---|---|---|---|---|---|---|
| curtailment | d1_ENE_CNF | NE | persist_d1 (replay n=11) | NMAE 28.2% (v2 LGBM, original split) | skill +0.27 vs persist_d1 | 0002 | [4/5] (B5 warn) | 2026-05-24T01:30Z |
| curtailment | d1_ENE_CNF | NE | persist_d1 (replay n=11) | NMAE 31.7% (v2 LGBM, holdout estrito gap 7d) | skill +0.18 vs persist_d1 | 0002 | [4/5] | 2026-05-24T01:30Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 (replay n=11) | NMAE 36.2% (v2 LGBM, original split) | skill +0.43 vs persist_d1 | 0002 | [3/5] (B3 fail strict) | 2026-05-24T01:30Z |
| curtailment | d1_ENE_CNF | S | persist_d7 (n=11) | nao_aprendivel (ymean_test=0) | — | 0002 | — | 2026-05-24T01:30Z |
| curtailment | d1_ENE_CNF | N | persist_d1 (n=11) | persist_d1 ainda vence (ML NMAE 97% > persist 34%) | — | 0002 | [3/5] | 2026-05-24T01:30Z |
| meta | h2_off_by_one_pdp | — | dbt_join_correto | corr(PDP[t], gen[t])=0.9118 / corr(PDP[t], gen[t+1])=0.8211 | H2 REFUTADO | 0003 | n=484 dias | 2026-05-24T02:30Z |
| meta | sanity_check_B6 | — | n/a | zero_count_shift + signal_collapse implementado e validado (sintetico high+collapse, real SE/v3 lag sign-flip) | adicionado a default pipeline | 0004 | passou | 2026-05-24T03:00Z |

---

## Legenda

- **baseline**: tipo do baseline + tamanho do test set onde foi medido
- **best_metric**: melhor metrica conhecida (modelo + janela)
- **delta_vs_baseline**: skill score `1 - MAE_model/MAE_baseline`. Positivo = melhor que baseline.
- **last_iter**: iter que produziu o numero atual
- **sanity_ok**: `[n/5]` = quantos sanity checks (B1 leak, B2 PI, B3 holdout strict, B4 baseline, B5 dist shift) passaram limpo. WARN = parcial.
- **data_utc**: ISO 8601 UTC do registro

## Notas

- **Test n=11 dias** no replay (limite imposto pela staleness do CH local em feat_intercambio + feat_carga + feat_termico). UlFor roda em ambiente com test n=60d.
- **NE/v3 nao-promovivel** ainda (iter_0002): regressao vs v2 por distribution shift PDP, NAO por off-by-one (confirmado iter_0003).
- **SE/v3 ganho aparente NAO sobrevive holdout estrito** (+19.2pp NMAE com gap 7d).
- **N permanece nao-aprendivel** com janela atual: ML NMAE 97% > persist_d1 34%.

## Iter 0003 — H2 validation

Iter 0003 nao produziu novo modelo. Validou H2 (off-by-one PDP) empiricamente:
correlacao(PDP_prev[t], gen[t]) = 0.9118 > correlacao(PDP_prev[t], gen[t+1]) =
0.8211. **H2 REFUTADO.** dbt model + bakeoff JOINs estao corretos.

Causa real do "shift -1 melhora NMAE" do iter_0002: provavelmente ruido
amostral (n=11) — request req-0001 enviado ao UlFor para confirmar com
n>=60d.

## Iter 0003 — Canal loop->UlFor

Estabelecido em commit UlFor `1fb2bb50`. Primeiros requests:
- req-0001 (P1 investigation): validar "shift -1 e ruido" com n>=60d
- req-0002 (P0 feature_fix): forward-fill em vez de coalesce(0) para gaps PDP Apr/2026

## Iter 0004 — B6 zero_count_shift + req-0003

- B6 implementado em sanity_checks/zero_count_shift.py + integrado a __init__.py
- Validacao sintetica: caso canonico coalesce(0) flagged severity=high + signal_collapse=magnitude_collapse
- Validacao real (snapshot iter_0002 v3): caught lag-feature sign-flip em SE/v3 e NE/v3:
  - SE/v3: curt_lag1 +0.252->-0.273, curt_lag7 +0.196->-0.146, ger_eolica +0.277->-0.291 (TODOS sign flip)
  - NE/v3: curt_lag1 +0.664->-0.229, curt_lag7 +0.415->-0.341 (sign flip)
- B6 e diagnostico da CAUSA RAIZ do "SE colapso strict +19.2pp" do iter_0002
- req-0003 enviado ao UlFor (P1 investigation, evidencia B6): commit `9c1484e2`
