# Leaderboard — forecast-mega-loop

Estado de cada alvo atravessando o DAG. Linha por (layer, alvo, sub).
Atualizado pelo watchdog ao final de cada iteração com ganho promovido.

| layer | alvo | sub | baseline | best_metric | delta_vs_baseline | last_iter | sanity_ok | data_utc |
|---|---|---|---|---|---|---|---|---|
| curtailment | d1_ENE_CNF | NE | persist_d1 | **NMAE 35.7% (UlFor v3.3 oficial, n=60d, R² +0.402)** | UlFor oficial sub-level | 0006 | UlFor n=60 | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 | **NMAE 46.0% (UlFor v3.3 oficial, n=60d, R² +0.386, PROMOVIVEL FASE 4)** | UlFor oficial + req-0003 verdict | 0006 | UlFor n=60 | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF | S | persist_d1 (113.7%) | **NMAE 109% xgb UlFor v3.3 (ML AGORA bate persist!)** | skill +0.04 vs persist_d1 | 0006 | UlFor n=60 | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF | N | persist_d1 | NMAE 72.2% lgbm (UlFor v3.3, sem-lags vence) | UlFor oficial | 0006 | UlFor n=60 | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF | NE | persist_d1 (loop replay n=11) | NMAE 28.2% v2 LGBM original / 31.7% holdout strict | superado por UlFor v3.3 | 0002 | [4/5] | 2026-05-24T01:30Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 (loop replay n=11) | NMAE 36.2% v2 LGBM | superado por UlFor v3.3 | 0002 | [3/5] B3 fail | 2026-05-24T01:30Z |
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

## Iter 0005 — Self-planning upgrade

- Loop opera em modo self-planning a partir desta iter
- 4 componentes novos: hypotheses_queue.md (15 H), planner.py, quality_gate.py, status.sh
- run.sh reescrito como continuous loop com budget caps + --dry-run
- Smoke test (3 iters dry-run): planner seleciona H9 (P1), bridge sync funcionou
- UlFor processou TODAS 3 requests durante esta iter (be2c9186 patch + b7acfdcd PDP fix)
- Loop continuo NAO INICIADO — aguarda autorizacao manual do Breno apos revisao

## Iter 0006 — Recon delta UlFor (numbers oficiais v3.3 absorved)

UlFor v3.3 sub-level oficial pos PDP gap fix (test 2026-03-23 -> 2026-05-21, n=60d, gap 7d):

| sub | NMAE pre-fix | NMAE pos-fix | R² pre | R² pos | observacao |
|---|---|---|---|---|---|
| NE | 36.0% | **35.7%** | +0.375 | **+0.402** | +2.7pp var explicada |
| SE | 46.2% | **46.0%** | +0.373 | **+0.386** | bias -2017 -> -1896 |
| S | 117% | **109%** | -0.133 | -0.135 | **ML quebra teto persist=113.7%!** |
| N | 72.2% | 72.2% | +0.022 | +0.041 | lgbm slight melhor |

Decisoes oficiais UlFor:
- **SE/v3 PROMOVIVEL para FASE 4** (req-0003 verdict)
- **S agora aprendivel como regressor** (era nao_aprendivel)
- Lags por sub: NE=KEEP, S=KEEP (ajudam), N=REMOVE (atrapalham), SE=indiferente

B6 lessons learned (req-0001 + req-0003 responses):
- B6 com test n=11 deu falso positivo (sign-flip em SE) que n=60d nao confirma
- Causa raiz iter_0002 NE delta -4.2pp era amostragem (14/16 deltas <2pp com n=60)
- H16 nova (P1): ajustar B6 para downgrade severity quando n_test < 30

Hipoteses fechadas nesta iter: H6 (B6 sign-flip refutado por UlFor)
Hipoteses adicionadas: H16 (B6 threshold-by-n), H17 (P0 promover SE/S v3.3 a FASE 4)
