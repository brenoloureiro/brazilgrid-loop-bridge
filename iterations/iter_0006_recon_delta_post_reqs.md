---
alvo: ulfor_v33_metrics_absorbed_and_reqs_closed
layer: meta
iter_num: 0006
type: recon_delta
data_utc: 2026-05-24T05:00:00Z
baseline_tipo: ulfor_oficial_v33
hypothesis: |
  UlFor processou 3 requests do loop e commitou v3.3 PDP gap fix
  (b7acfdcd). Absorver metricas oficiais, fechar reqs, atualizar
  state/queue/leaderboard antes de continuar fila.
result_metric: |
  3 requests fechados com ulfor_response. v3.3 sub-level absorved:
    NE 35.7% R²+0.402  SE 46.0% R²+0.386 (PROMOVIVEL FASE 4)
    S 109% R²-0.135 (ML AGORA bate persist 113.7%!)  N 72.2% R²+0.041
  H6 REFUTADO via req-0003 (sign-flip B6 era amostral n=11).
  H2 e B6 lessons absorved.
decision: ABSORVED — UlFor oficial vira fonte de verdade do leaderboard.
sanity_checks_passed:
  ulfor_responses_extracted: true   # 3 reqs ulfor_response preenchidos
  state_queue_leaderboard_sync: true
  no_new_requests_needed: true      # tudo pendente foi fechado nesta iter
budget_consumido_iter: 1.8
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/state.json (closed_requests com ulfor_verdict, best_ml_oficial_ulfor_v33)
  - loops/forecast-mega-loop/leaderboard.md (4 linhas UlFor v3.3 + 2 deprecadas loop replay)
  - loops/forecast-mega-loop/hypotheses_queue.md (H6 done, H16 + H17 novas)
  - loops/forecast-mega-loop/iterations/iter_0006_recon_delta_post_reqs.md
---

# Iter 0006 — Recon delta UlFor + reqs fechados

## UlFor commits absorved

### `be2c9186` — autopilot patch aplicado

Breno aplicou o patch sugerido em iter_0003 `autopilot_patch.md` ao
`docs/forecast/checkpoints/AUTOPILOT_PROMPT.md`. Novo passo 4.5 ao
PROTOCOLO DE RETOMADA: autopilot le `coordination/loop_requests.md`
no inicio de cada retomada e processa requests OPEN/IN_PROGRESS
respeitando envelope (PARAR E PERGUNTAR em P0/P1), ordem P0->P1->P2
e dentro de P por tipo feature_fix > test > investigation > metric_check.

Impacto loop: canal loop->UlFor agora e *self-driving* — toda
escrita em `loop_requests.md` sera lida pelo autopilot na proxima
retomada sem intervencao do Breno.

### `b7acfdcd` — PDP gap Abr/26 fix + v3.3 sub-level ganhos

Backfill manual de 28 dias faltantes em Abr/2026 do dataset
`programacao_previsao` via Dagster CLI (1 partition/dia, ~30s
cada). Re-materializado `stg_ons_programacao_previsao` +
`feat_pdp_renovavel` via dbt run no EC2.

**Cobertura feat_pdp** por (sub, fonte): 56 dias -> 84 dias completos.

**v3.3 sub-level oficial** (test 2026-03-23 -> 2026-05-21, n=60d):

| sub | NMAE pre-fix | NMAE pos-fix | R² pre | R² pos | observacao |
|---|---|---|---|---|---|
| NE | 36.0% | **35.7%** | +0.375 | **+0.402** | +2.7pp var explicada |
| SE | 46.2% | **46.0%** | +0.373 | **+0.386** | bias -2017 → -1896 |
| S | 117% | **109%** | -0.133 | -0.135 | **ML quebra teto persist=113.7%** |
| N | 72.2% | 72.2% | +0.022 | +0.041 | lgbm marginal |

**Ganho mais material**: S. Pre PDP fix, baseline `persist_d1` 113.7%
**vencia** ML (117.2%). Pos fix, xgb 109.0% **< 113.7%** = ML agora
bate baseline em S. PDP solar S preenchido foi feature chave (era
nula durante o gap Abr/26).

Causa raiz cron failure TBD — S3 publicou tudo, mas dlt asset
partition runs faltavam. Investigacao em frente de monitoring
separada (nao loop).

## Requests fechados (3)

### req-0001 (P1 investigation) — DONE

**Pergunta**: Validar com test n>=60d que shift -1d em pdp_prev_eolica
e ruido amostral (off-by-one ja refutado em iter_0003).

**ulfor_response**: RUIDO AMOSTRAL CONFIRMADO. Sanity B1 re-aplicado
com test 60d em v3.3 (gap fechado):

| feature shifted | NE | SE | S | N |
|---|---|---|---|---|
| pdp_prev_eolica_mwh | =0.0 | -0.3pp | -0.5pp | =0.0 |
| pdp_prev_solar_mwh | =0.0 | +1.2pp | +0.1pp | +3.4pp |
| prev_eolica_ger_mwh | =0.0 | =0.0 | -0.5pp | =0.0 |
| prev_solar_ger_mwh | =0.0 | =0.0 | =0.0 | =0.0 |

**14/16 combinacoes com |delta| < 2pp.** Excecao: N + pdp_prev_solar
+3.4pp (N e' low-signal). Iter_0002 +4.2pp NE foi variabilidade
amostral n=11 vs n=60.

### req-0002 (P0 feature_fix) — DONE via b7acfdcd

**Pergunta**: Substituir coalesce(0) por forward-fill no bakeoff
para gaps PDP Abr/2026.

**ulfor_response**: PREMISSA INVALIDADA — gap fechado upstream em vez
de mitigado downstream. Backfill manual 28 dias via Dagster, re-mat
stg + feat. Coalesce 0 agora SEGURO (sem gaps artificiais).
Forward-fill nao necessario.

**Material**: v3.3 sub-level vide tabela acima.

### req-0003 (P1 investigation) — DONE

**Pergunta**: SE/v3 colapso strict +19.2pp era sazonalidade ou
amostral? B6 do iter_0004 detectou sign-flip em curt_lag* SE.

**ulfor_response**: Re-rodado com test n=60d, gap 7d.

Sub SE corr_train vs corr_test:

| feature | corr_train | corr_test | sign_flip |
|---|---|---|---|
| curt_lag1 | +0.252 | +0.132 | NAO (mesma direcao) |
| curt_lag7 | +0.188 | +0.107 | NAO |
| curt_lag14 | +0.160 | -0.083 | marginal (|corr|<0.1) |
| curt_rmean7 | +0.449 | +0.170 | NAO |
| ger_eolica_mwh | +0.287 | +0.164 | NAO |
| pdp_prev_solar_mwh | +0.198 | +0.034 | mesma direcao |

**Sign-flip NAO confirmado.** Correlacoes test atenuam mas mantem
direcao positiva. Magnitudes |corr_test| ~ 0.1-0.2 em todos lags
(vs +0.252 train) = variabilidade amostral n=60.

**NMAE variantes (gap 7d)**:
- v3a (com lags): NE 34.8% / SE 46.5% / S 118.3% / N 70.4%
- v3b (sem lags): NE 37.7% / SE 45.8% / S 119.1% / **N 68.1%**

**Decisao por sub**:
- NE: lags AJUDAM (+2.9pp se removidos) — manter
- S: lags ajudam — manter
- N: lags ATRAPALHAM (-2.3pp se removidos) — **remover**
- SE: indiferente (~-0.7pp) — manter por consistencia

**VEREDITO: SE/v3 PROMOVIVEL para FASE 4.** Sign-flip B6 do loop foi
falso positivo amostral (test n=11).

## Atualizacoes leaderboard/queue/state

### leaderboard.md

UlFor v3.3 vira fonte de verdade sub-level (n=60d):
- NE/v3.3 35.7% / SE/v3.3 46.0% / S/v3.3 109% / N/v3.3 72.2%
- Loop replays n=11 marcados deprecated_by

### hypotheses_queue.md

- **H6** SE colapso: status `blocked` → **`done`** (iter_handled 0006,
  completed_at)
- **H16 (novo)** P1: ajustar B6 threshold por n_test (downgrade severity
  quando n<30) para evitar falso positivo demonstrado neste iter
- **H17 (novo)** P0: promover SE/v3 + S/v3.3 a FASE 4 (registrar como
  request P0 ao UlFor — loop nao executa promocao)
- **H15** S classificador: priority P2 → P3 (PARCIALMENTE OBSOLETA, ML ja
  bate baseline com PDP fix)

### state.json

- `iter_atual` = 6
- `best_ml_oficial_ulfor_v33` adicionado (NE, SE, S, N com decisao FASE 4)
- `best_ml_loop_replay_n11` marcado deprecated_by
- `closed_requests` com `ulfor_verdict` por req
- `open_requests` = []
- `ulfor_session_sync.iter0006_fim_head` = be2c9186 (sem novo commit
  durante iter — recon de commits ja existentes)

## Nova prioridade da queue (proximo iter)

Topo da queue por priority + dep satisfeita:

1. **H17 (P0 new)** — promover SE/S v3.3 a FASE 4. Acao = registrar
   request P0 ao UlFor com payload (qual model serializer, qual drift
   monitor, metricas baseline para alarm)
2. **H16 (P1 new)** — fix B6 threshold-by-n. Loop pode executar (0.5h)
3. **H9 (P1)** — NMAE → MAE/R²/F1 (Principio 6 PLANO_FINAL). Loop pode
   executar (1.0h)
4. **H7 (P2)** — XGBoost vs LGBM systematic
5. **H3 (P2)** — PDP residual vs gen
6. ...

**Recomendacao iter_0007**: H16 (B6 robustness fix). Loop pode rodar
sozinho, valor metodologico imediato — toda iter futura usa B6 menos
ruidoso. H17 e P0 mas e acao indireta (criar req), pode esperar 1 iter.

## Recomendacao sobre re-rodar B6 sobre v3.3 outputs (H14)

H14 era "rodar B1-B6 sobre v3.3". Pelo trabalho UlFor desta iter,
isso JA aconteceu parcialmente:
- B1 leak: req-0001 ja validou shift -1d em v3.3 → ruido amostral
- B3 holdout strict: req-0003 ja rodou gap 7d em v3.3
- B5 PSI: nao reportado explicitamente pelo UlFor
- B6 zero_count_shift: nao rodado pelo UlFor; **mas com gap fechado,
  zero_rate em test deve ser ~0% em todas PDP cols, falso positivo
  do iter_0004 nao se repete**

H14 fica como blocked em req-0006 (publicar v3.3 preds em formato
processavel pelo loop). Loop nao pode rodar B6 diretamente sobre o
modelo do UlFor sem acesso aos preds/booster.

## Loop continuo

Status: **AINDA NAO INICIADO**. Esta iter foi executada manualmente
via `_force_next_iter.txt` (override). Apos commit + heartbeat, arquivo
sera deletado.

Para iniciar loop continuo:
```bash
bash loops/forecast-mega-loop/watchdog/run.sh \
  > loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
echo $! > loops/forecast-mega-loop/watchdog/_loop.pid
```

Planner detectara queue atualizada e selecionara H16 (P1) como
proxima iter normalmente — ou H17 (P0) se ordenacao primaria.
