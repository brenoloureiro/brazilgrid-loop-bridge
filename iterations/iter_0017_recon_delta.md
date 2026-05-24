---
alvo: recon_delta_ulfor_post_5dacb5a2
layer: meta
iter_num: 0017
type: recon_delta
data_utc: 2026-05-24T14:30:00Z
ulfor_head_inicio: 5dacb5a2
ulfor_head_fim: 5c7963d4
commits_absorvidos: 15
novos_requests: 0
requests_fechados: 1   # req-0007
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.5
---

# Iter 0017 — RECON_DELTA UlFor (5dacb5a2 → 5c7963d4)

## Objetivo

Absorver 15 commits novos da sessao UlFor entre o checkpoint 07:15Z
(`5dacb5a2`, HEAD na entrada do iter_0016) e o checkpoint atual
(`5c7963d4`, 06:25Z do dia seguinte — UlFor rodou tres sprints autopilot
em paralelo enquanto o loop processava H19 + agora abre H17). Recon-only —
sem testar hipotese de modelagem, so reconcilia state/queue/leaderboard
+ fecha req-0007 (UlFor respondeu).

## PHASE A — Commits inspecionados

Em ordem cronologica (mais antigo primeiro):

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `13f4a4de` | feat — Fase 4 step 2 (dashboard MLflow vitrine) | Pagina `dataops/vitrine/forecast.html` + JS + endpoint proxy `/api/mlflow/validation-runs`. Mostra status NE/SE/S/N (NMAE/R²/skill/PSI + champion) + timelines 30 ultimas validations + tabela runs. CSS-puro consistente com vitrine. **Nao afeta nossa interpretacao** — apenas surface UX para validate_d1 (que ja absorvemos iter_0015). | absorvido |
| 2 | `8a30a29e` | feat — Fase 4 step 3 (rastreabilidade end-to-end) | `/api/forecast/d1` agora retorna `mlflow_url` clicavel, `training_git_sha`, `experiment_id/name`, `training_ran_at`, `inference_git_sha`. Divergencia training vs inference sha = sinal de modelo treinado em codigo antigo. Smoke 4 subs OK. **Nao afeta interpretacao** — observabilidade que ajuda futuros sanity checks B1-B6 (H18). | absorvido |
| 3 | `5ea5a412` | chore — checkpoint 08:15Z | Marker. "Fase 4 STEPS 2+3 fechadas (dashboard MLflow + rastreabilidade)" — Fase 4 da PLANO_FINAL fechou as 3 etapas: step 1 validate_d1+drift+Telegram (iter_0015), step 2 dashboard (commit 1), step 3 rastreabilidade (commit 2). | absorvido via 1+2 |
| 4 | `f64cfbb7` | coord — receive req-0007 | Inbound do loop iter_0016 ao canal `coordination/loop_requests.md` (P2 data_publish: F1_p50 por fold + per-fold MAE parquet). | absorvido (rastro do envio) |
| 5 | `f697449d` | feat — UlFor H13 closure S REFUTADA | Opcao D do checkpoint 08:15Z. CV 5x60d + 14d real (2026-05-08..21): `clean_plus` PIOROU S em CV (-0.28 R² lr, -0.14 ridge vs full). Em 14d real, TODOS ML perdem para persist_d1: persist 99.9% NMAE/R²-0.095 > ridge+clean_plus 137.4%/-0.942 > **lr+full champion 159.6%/-1.915** (champion atual catastrofico em janela seca-Mai/26). MANTER `lr_curt_s_d1@champion v1` (CV-validado, melhor config em CV — Principio #5 PLANO_FINAL: NAO promover ML que perde a persist). Nova UlFor H18 aberta (S underperforma persist estruturalmente). **CHAMPION S INALTERADO**, mas FRAGIL operacional reconfirmado (alinha com FRAGIL atenuado iter_0015). | absorvido |
| 6 | `6b21ffdf` | feat — **req-0007 DONE** + F1_p50 + per-fold parquet | UlFor implementou AMBAS as lacunas em 1 sprint. (a) `_f1_p50()` portado de `loops/forecast-mega-loop/scripts/metric_suite.py` para `bakeoff_d1.py:metrics()`. `metrics(y_train=...)` retorna `f1_p50`, `threshold_p50`, `ymean_test` ao lado de mae/rmse/nmae/bias/r2. MLflow log + CV_SUMMARY atualizados. Back-compat preservada. (b) Per-fold parquet dump em `experiments/bakeoff_curtailment_multisub/outputs/cv_summary_per_fold_{feature_set}.parquet` (schema do acceptance + 6 extras). 4 arquivos publicados: `clean_plus + full` × `per_fold + mean`. **F1_p50 results clean_plus**: NE ridge/lr=0.80 (best, persist=0.75), SE xgb=0.71 (persist=0.69), N ridge=0.79 (persist=0.72), S=NaN (P50=0 esperado). **CRITICO**: req-0007 OPEN → DONE. H19 follow-up agora UNBLOCKED — proxima iter pode parsear parquet `full` e fechar gap F1 do leaderboard. | absorvido |
| 7 | `d93100ae` | feat — UlFor H19 root-cause SE/fold4 R²=-2.5 | Ablation single-feature add-back em SE/fold4 (test 2025-07-27..09-24): `ter_verif_rmean7` sozinha restaura 99% do gap clean→full (R² -2.53 → -0.48 ridge; -3.12 → -0.07 lr). Dropada por `FEATURE_DROPS_CLEAN` (corr +0.92 com `ter_verif_lag1`). Em seca-2025Q3, rolling-7d captura trend termico que lag1 pontual nao captura; OLS nao recupera via lag1, GBDT sim via nao-linear. **VIF/corr nao basta para guiar drops em sub com regimes sazonais fortes**. Reforco da licao FINDING_VIF_GREEDY. **NAO confundir com nosso H19** (extracao MAE/R²/F1 champions, iter_0016 CONFIRMADO_PARCIAL). | absorvido |
| 8 | `8708c329` | chore — checkpoint 05:00Z | Marker (H13 closed + req-0007 + H19 SE finding). | absorvido via 5+6+7 |
| 9 | `ea8ca325` | feat — UlFor H20 closure CONFIRMADA | Follow-up direto do H19 root-cause. Novo feature-set `clean_plus_v2 = clean_plus − {drop ter_verif_rmean7}` (39 features global). Re-CV 5x60d: SE lr **−11.27pp NMAE / +0.62 R²** (major win). NE ridge +0.26pp / −0.02 R² (within bound). S lr +1.34pp (within bound). N ridge +0.35pp (within bound). Fold 4 SE: R² lr -3.12 → -0.07, ridge -2.53 → -0.48. **CHAMPIONS NAO ALTERADOS** — todos seguem em `full` por consistencia com runs MLflow historicos. `clean_plus_v2` e' alternativa analitica daqui pra frente, base de H21 ulfor (re-treinar champion SE em v2 e comparar 14d). | absorvido |
| 10 | `e56aa5f3` | feat — UlFor H18-A REFUTADA | Investigacao motivada pelo checkpoint 05:00Z (Opcao H18-A). H18 ulfor abriu: S underperforma persist estruturalmente. H18-A testa drop global de 4 envenenadoras ridge_S+full (`curt_rmean7`, `pdp_prev_solar_mwh`, `val_net_mwmed`, `curt_lag14`). Surgical drop CV 5x60d: NMAE 97.9% → **103.5% (+5.55pp PIORA)**, R² +0.23 → +0.15, stdev 18.7% → 29.2% (+10.5pp mais instavel). Por fold: fold 0/3/4 neutra/melhora; fold 1 +13.13pp, fold 2 +23.55pp (PIORA muito). Mesmas features uteis em 3/5 folds. **Mesma licao do H19**: aggregate "envenenadora em ≥2 folds" nao guia drop global. `pdp_prev_solar_mwh` em particular = Top-3 IMPORTANTE fold 1 (+58.3pp ao manter), Top-3 ENVENENADORA fold 0 (-2.7pp). Papel inverso por regime. **UlFor H18 fica aberta** (H18-B per-fold FS, H18-C target alt log1p/binario, H18-D mixture-of-experts por regime — nao no autopilot). lr_S+full mantido. | absorvido |
| 11 | `108772a5` | chore — checkpoint 05:30Z | Marker (H20 NET WIN + H18-A REFUTADA). | absorvido via 9+10 |
| 12 | `4223aa4c` | feat — UlFor H21 REFUTADA (14d real lr_SE v2) | Follow-up imediato H20. Treino lr_SE ate 2026-05-07, holdout 14d 2026-05-08..21. `clean_plus_v2` **perde feio**: persist 69.6% NMAE / lr+full champion 77.4% / lr+clean_plus_v2 **92.7% (+15.31pp vs full)**. Ganho H20 estava concentrado em fold 4 (seca-2025Q3); regime Mai/26 (transicao chuvoso→seco, 3 dias zero curt) favorece FULL. **Champion `lr_curt_se_d1@v1 full` PERMANECE**. NAO promove. Observacoes: NENHUM ML bate persist_d1 NMAE em 14d (**teto-de-dados D+1 estendido do NE ao SE no regime atual** — alinha com Fase 4 v3 ceiling iter_0018 do bigforecaster, ver `MEMORY.md::fase4_v3_curtailment_d1_data_ceiling`). lr+full DOMINA em F1_p50 (0.714 vs persist 0.545). Bias estrutural +5500-7100 MWh em todos ML — abre janela H14 ulfor. | absorvido |
| 13 | `41d8d952` | feat — UlFor H14 + H14-C PRODUTIZAR NE | Bias correction rolante 28d: `corrected_D = max(0, pred_D − mean(pred − actual)[ultimos 28d causais])`. **14d real (2026-05-08..21)**: NE **−9.94pp NMAE (59.2 → 49.2%, R² −0.348 → −0.135)**, bate persist (54.1%) em 4.9pp — **primeira vez no projeto**. SE +1.61pp (regime change joga correcao no rumo errado). S +10.81pp (catastrofico, ymean ~32 MWh amplifica ruido). N +2.15pp (raw ja bate persist por 36.9pp; bias estrutural negligivel). **CV walk-forward 5x60d** (cobre 2025-07..2026-05): NE media **−6.90pp, wins 4/5, loses 0/5 → PRODUTIZAR**; SE 0/5 wins → NAO; S +0.95 mean → NAO; N media −28.63pp wins 3/5 loses 2/5 → MAYBE (fold 4 −120pp pulls mean; fold 0/1 regridem +2pp). Decisao analitica: NE def. produtiza; N risky; SE/S nao tocar. **Champions NAO alterados** — bias e' post-processing layer. | absorvido |
| 14 | `586eeae6` | feat — route `/api/forecast/d1` expoe `bias_correction_mw` | Implementa decisao H14/H14-C em PRODUCAO (loader.py + route). `BIAS_CORRECTION_DEFAULTS = {NE: True, SE: False, S: False, N: False}` + janela 28d. Response enriquecida: `predicted_curt_mwh` (raw, back-compat), `predicted_curt_mwh_corrected` (raw − bias_28d, sempre exposto), `predicted_curt_mwh_default` (raw OR corrected per-sub default), `bias_correction.{bias_mw, applied_in_default, recommended_apply, window_days, n_used, window_start, window_end, applicable, source}`. Smoke e2e: NE bias=+7319 MWh → raw 62329 → corrected 55010 (perto persist 59149). SE bias=−284 not applied. S bias=−167 not applied. N bias=−29 not applied. **NE NMAE D+1 cai estruturalmente de 33.7% CV (sem correcao) para esperado ~30% (extrapolando ganho 14d) com bias_correction ativa por padrao**. `analytics_api` NAO deployada ainda (sem risco prod). | absorvido |
| 15 | `5c7963d4` | chore — checkpoint 06:25Z | Marker (3 sprints: H21 REFUTADA + H14 analitica + H14 produtizado). | absorvido via 12+13+14 |

## PHASE B — O que mudou na nossa interpretacao do leaderboard

### Mudancas de PRODUCAO

1. **NE D+1 bias-corrected por padrao** (commits 13+14). Champion `ridge_curt_ne_d1@v1 full` continua **TREINADO igual**, mas a route agora aplica subtracao do bias rolante-28d antes de retornar `predicted_curt_mwh_default`. Em 14d real: −9.94pp NMAE (59.2 → 49.2%), bate persist por 4.9pp. CV 5x60d: −6.90pp mean, wins 4/5 loses 0/5. **Primeiro modelo do projeto a bater persist D+1 em janela real para curt total NE**. Coluna `best_metric` da linha NE do leaderboard mantem MAE/R²/NMAE do CV oficial (que NAO inclui bias correction), mas adiciona nota inline.
2. **N champion ainda `ridge_curt_n_d1@v2 staging`** (clean_plus, 31 feat). Inalterado nesta janela.
3. **SE/S champions inalterados** — H21 ulfor REFUTOU clean_plus_v2 em 14d real (SE +15.31pp), H13 ulfor REFUTOU clean_plus em CV+14d (S regrediu).

### Mudancas de OBSERVABILIDADE

4. **Fase 4 PLANO_FINAL UlFor 100% FECHADA**: step 1 validate_d1+drift+Telegram (iter_0015), step 2 dashboard MLflow vitrine (`forecast.html`), step 3 rastreabilidade end-to-end no endpoint (`training_git_sha`, `mlflow_url`, etc.). Telegram alert ainda STOPPED ate Breno gerar `BRAZILGRID_TELEGRAM_BOT_TOKEN/CHAT_ID` (carry-over iter_0015).

### Mudancas de DADO

5. **req-0007 DONE** (commit 6): F1_p50 + per-fold parquet dump publicados. F1_p50 clean_plus: NE ridge/lr=**0.80**, SE xgb=0.71, N ridge=**0.79**, S=NaN (P50=0). Parquet path: `experiments/bakeoff_curtailment_multisub/outputs/cv_summary_per_fold_{clean_plus,full}.parquet` (~13KB cada, 140 rows = 4 subs × 7 modelos × 5 folds, schema MAE/RMSE/NMAE/bias/R²/F1_p50/ymean_test/threshold_p50/n_train/n_test/janelas). **H19 follow-up agora codavel local**: parser parquet → preencher F1_p50 na linha leaderboard de cada champion + remover caveat 1a ordem ~10-15% MAE-em-MWh (substituir derivado por per-fold real ao agregar).

### Mudancas de METODOLOGIA absorvida (insights UlFor)

6. **`ter_verif_rmean7` load-bearing em SE/seca-2025Q3** (commit 7). Reforco VIF-only insuficiente. Util ao desenhar feature drops do loop.
7. **Drop global por "envenenadora em ≥2 folds" NAO funciona** (commit 10). `pdp_prev_solar_mwh` em particular tem papel inverso por regime (Top-3 IMPORTANTE fold 1, Top-3 ENVENENADORA fold 0). Apenas per-fold ou regime-aware FS faz sentido.
8. **Teto-de-dados D+1 estendido NE→SE** (commit 12). NENHUM ML bate persist NMAE em 14d real SE (Mai/26 regime transicao). Alinha com `MEMORY.md::fase4_v3_curtailment_d1_data_ceiling` ja documentado.
9. **Bias correction rolante e' uma camada ortogonal aos features** (commits 13+14). Mecanismo: causal mean(pred − actual) em janela curta corrige drift de modelo lento. Aplicavel a qualquer modelo (Ridge, LR, LGBM ensemble do nosso H10) e qualquer sub onde o sinal e' estavel. NAO testavel via CV 5x60d historico sem reimplementar a logica de aplicacao causal (UlFor implementou).

## PHASE B — Mudancas concretas em arquivos

- **state.json**:
  - `iter_atual`: 16 → 17
  - `alvo_ativo`: `leaderboard_consistency_post_h9` → `recon_delta_ulfor_post_5dacb5a2`
  - `iter_no_alvo_atual`: 1 → 1 (novo alvo)
  - `ultimo_handoff`: → `iter_0017_recon_delta.md`
  - `ulfor_session_sync.iter0017_inicio_head` = `5dacb5a2`, `iter0017_fim_head` = `5c7963d4`, `novos_commits_durante_iter0017` = [15 entries], `delta_resumo_iter0017` = {champion changes, fase 4 close, req-0007 done, insights}
  - `open_requests`: remove `req-0007`
  - `closed_requests`: add `req-0007` com `ulfor_verdict` resumido + paths dos arquivos publicados
  - `best_ml_oficial_ulfor_cv_5folds.NE`: add `bias_correction_productized = {default: true, window_days: 28, expected_nmae_delta_14d_pp: -9.94, expected_nmae_delta_cv_pp: -6.90, wins_loses_cv: "4/5/0", commit: "41d8d952+586eeae6"}`
  - `planner_config.notas_iter0017` documenta o recon
- **leaderboard.md**: bloco no topo (iter 0017 RECON_DELTA) + nota inline na linha NE (`bias_correction_mw default ON, prod 14d empirical 49.2% NMAE`) + nota inline `req-0007 DONE` na celula `sanity_ok` da linha NE/SE/S/N. Adicionar linha "meta | fase_4_observabilidade_complete" com status 3/3 steps + Telegram pendente.
- **hypotheses_queue.md**: `last_updated` → 2026-05-24T14:30:00Z. **Nenhuma H do loop fechada nesta iter** (UlFor numera Hs internos, ja foi nota nomenclatura iter_0015). H19 status mantido `done CONFIRMADO_PARCIAL` mas adicionado `external_unblock_note: req-0007 DONE — parser parquet now possible, follow-up cheap close to full CONFIRMADO`.

## PHASE C — Qual hipotese fica na fila para iter_0018

`planner_config.next_iter_should_be` mantido como **H21** (P2 feature engineering `pdp_residual = pdp_prev − gen`), derivada de H3 iter_0010, codavel localmente sem dep externa.

**Mas:** atratividade da H19-followup (`parse cv_summary_per_fold_full.parquet` e preencher F1_p50 nas 4 linhas champions do leaderboard) subiu muito — req-0007 DONE destrava, custo <0.5h. Recomendacao: planner considere H19-followup como warmup do iter_0018 (5-30min) **antes** de abrir H21 (1.5h). Se exceder budget, fechar so' H19-followup e deferir H21 para iter_0019.

**Alt nova candidata (P3):** **H29** = "Aplicar tecnica UlFor H14 (bias_correction rolante 28d) sobre LGBM-ensemble do nosso H10 e medir delta NMAE no CV 5x60d em todas 4 subs". Mecanismo ortogonal (mean shift correction vs. mixing com persist) — pode compor ganhos. Custo ~1h. Nao criada no queue ainda (alt menos prioritaria que H19-followup e H21).

## Sanity checks

N/A — recon, sem hipotese de modelagem. 6 sanity checks default nao aplicaveis a iter type `recon_delta`.

## Budget

- Estimado: 0.5h (15 commits, deep-read em 6, parse 1 commit big-diff para validar req-0007 closure)
- Real: ~0.5h
- Acumulado dia (UTC 2026-05-24): iter 17, ~0.5h (cap diario USD 150 NAO atingido)
