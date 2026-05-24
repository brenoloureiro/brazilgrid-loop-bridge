# Leaderboard — forecast-mega-loop

Estado de cada alvo atravessando o DAG. Linha por (layer, alvo).
Atualizado pelo watchdog ao final de cada iteração com ganho promovido.

| layer | alvo | baseline_tipo | baseline_nmae | best_nmae | iter_vencedor | sanity_ok | data_utc |
|---|---|---|---|---|---|---|---|
| curtailment | curt_d1_NE_ENE_CNF | persist_d1 (replay n=11) | 0.388 | 0.282 (v2 LGBM) | 0002 | [4/5] (B5 warn) | 2026-05-24T01:30Z |
| curtailment | curt_d1_SE_ENE_CNF | persist_d1 (replay n=11) | 0.633 | 0.362 (v2 LGBM) | 0002 | [3/5] (B3 fail, B5 warn) | 2026-05-24T01:30Z |
| curtailment | curt_d1_S_ENE_CNF | persist_d7 (replay n=11) | — (ymean=0) | — | — | nao_aprendivel | 2026-05-24T01:30Z |
| curtailment | curt_d1_N_ENE_CNF | persist_d1 (replay n=11) | 0.342 | 0.974 (v3 LGBM) | — (ML pior) | [3/5] | 2026-05-24T01:30Z |

---

## Legenda

- **baseline_tipo**: persist_d1 = repete curt(D). persist_d7 = curt(D-6).
  ma7 = media movel 7d. climatologia_doy = media historica do mesmo dia-do-ano.
- **baseline_nmae**: NMAE do melhor baseline. NE/SE: persist_d1. S: ymean=0.
- **best_nmae**: melhor NMAE ML do bakeoff. v2 vence em NE+SE. v3 nao-promovivel.
- **iter_vencedor**: 0002 = ablacao confirma v2 > v3 em NE+SE.
- **sanity_ok**: [n/5] = quantos sanity checks passaram limpo.
  - B1 leak_detection: 0 leaks fortes (PASS)
  - B2 permutation_importance: PI rodou (PASS — mas confirma dominancia PDP suspeita)
  - B3 holdout_temporal_strict: NE/v3 +6.7pp piora, SE/v3 +19.2pp piora (FAIL p/ v3)
  - B4 baseline_compare: skill matrix produzida (PASS)
  - B5 distribution_shift: NE/v3 47 PSI alerts (>43 v2) (WARN)
- **data_utc**: ISO 8601 UTC do registro.

## Notas

- Test n=11 dias (limitado pela staleness do CH local em feat_intercambio + feat_carga + feat_termico).
- Replay v3 = ABLACAO sobre features (v3 = all, v2 = -PDP, v1 = -PDP -PREV); usa
  mesmos hyperparametros do bakeoff_d1.py UlFor (random_state=0).
- N/v3 LGBM `best_nmae=0.974` e PIOR que persist_d1=0.342 — ML nao consegue
  com janela atual. Persistencia simples ainda vence em N.

## Decisao iter 0002

- **v3 NAO PROMOVE** para FASE 4 do UlFor. Voltar para v2 ou v3.1+/v3.3 (PDP NE off).
- Antes de re-promover v3 generalizado: fix PDP gaps Apr/2026 + auditar
  off-by-one no JOIN PDP `addDays(t.dia, 1)`.
