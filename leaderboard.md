# Leaderboard — forecast-mega-loop

Estado de cada alvo atravessando o DAG. Linha por (layer, alvo, sub).
Atualizado pelo watchdog ao final de cada iteração com ganho promovido.

**Iter 0008 (H9):** metricas primarias agora **MAE/R²/F1** (PLANO_FINAL Principio 6).
NMAE mantida como secundaria — flaggada `unsafe` quando ymean<1 MWh.

**Iter 0009 (H16):** B6 zero_count_shift v1.1 — `n_test<30 -> downgrade severity 1 nivel`
e sign_flip exige `|corr_train| >= 0.2 AND |corr_test| >= 0.2` (era >0.05). Falso positivo
SE/v3 lag (iter_0004 com n_test=11) automaticamente atenuado: curt_lag7 sign_flip bloqueado,
severities raw=high downgrade para medium. Replay loop pode rodar em janelas curtas sem
gerar req desnecessario ao UlFor.

**Iter 0010 (H3):** PDP carrega sinal alem de gen via residual — **CONFIRMADO**. OLS decomp
`curt ~ gen + pdp` em n=486 dias contemporaneos: r2_extra(pdp_prev) = +0.31 NE, +0.25 SE,
+0.16 S(train) mas COLAPSA test. perm p=0.0, train/test estavel em NE+SE. pdp_prog quase
redundante com gen (corr 0.95, r2_extra <=0.12). Mecanismo: residual(pdp_prev - gen) e' proxy
de curtailment. Implicacao: manter pdp_prev_*; pdp_prog_* candidato a drop. H21+H22 derivadas.

**Iter 0011 (RECON_DELTA):** 7 commits UlFor `4e0fc7b4..c8df4077` absorvidos. Champions
**PROMOVIDOS** no MLflow Registry (ridge_NE / lr_SE / lr_S @champion; ridge_N @staging) +
**endpoint `/api/forecast/d1` LIVE** servindo D+1 (cold 6.5s, warm <50ms). VIF analise
confirma multicolinearidade massiva (38/55 features VIF>=10, cond_num >1e17), mas drop
universal CLEAN ajuda so NE/N — feature_set=full continua default. VIF greedy iterativo
**REFUTADO** (dropa drivers economicos primarios). lr_N instability isolada em fold 4
(jul-set/2025 = blowup 208%) → ridge_curt_n_d1@staging continua a defesa. Sem novos
requests; sem novas hipoteses do loop geradas. Detalhe em `iterations/iter_0011_recon_delta.md`.

**Iter 0012 (H7):** XGB vs LGBM CV walk-forward (5 folds 60d, gap 7d) sobre features
iter_0002 — **REFUTADO_LGBM_SYSTEMATICALLY_BETTER**. LGBM venceu MAE em **10/12 celulas
(83%)**. iter_0002 NE/v3 XGB R²=+0.52 (n=11) era ruido: em CV NE/v3 XGB R² medio=-0.18
vs LGBM +0.26 — inversao total. Maior gap: NE/v2 ΔMAE=+11.299 MWh em favor LGBM. Vetor
de degradacao: distribution shift (KS p<0.0001 NE+SE entre fold1 e foldN; y_mean cai 3x
em NE), XGB sofre mais por splits mais profundos. LGBM continua modelo padrao do loop.
Champions UlFor sao Ridge/LR (iter_0007) — H7 e' diagnostica do replay loop, nao de
producao. Lesson reforca iter_0006/req-0003 (n=11 falsos positivos). Detalhe em
`iterations/iter_0012_h7_xgb_vs_lgbm_cv.md`.

**Iter 0015 (RECON_DELTA):** 7 commits UlFor `c8df4077..5dacb5a2` absorvidos.
**Champion N atualizado**: ridge_curt_n_d1 v1 (full, 55 feat) → **v2 (clean_plus, 31
feat)** @staging, NMAE 86.3±31.8% → **84.8±26.2%** (-1.5pp mean, -5.6pp std). UlFor
H10 (FEATURE_DROPS_N = CLEAN ∪ {cmo_range, ter_verif_lag1, carga_mwmed_rmean7}) **PARCIALMENTE
CONFIRMADA** (nao confundir com nosso H10 ensemble iter_0013). Endpoint `/api/forecast/d1`
agora feature-set aware (loader le `feature_set` do MLflow run params, backward-compat v1).
**Fase 4 observabilidade FECHADA** pelo UlFor: `validate_d1.py` (replay daily MAE/R²/skill)
+ drift PSI nativo numpy/scipy (dual long/recent, Evidently abandonado por conflito plotly 5/6)
+ Telegram alert 4 gatilhos (`skill<0`, `R²<0`, `psi_recent_max>1.0`, `n_feat_drift>10`) +
Dagster asset `forecast/validate_d1` + schedule 07h BRT **STOPPED** ate Breno gerar
`BRAZILGRID_TELEGRAM_BOT_TOKEN/CHAT_ID`. Smoke confirma `lr_S` em colapso operacional
(NMAE 137-184% em 7-14d, skill -37 a -41%) e drift NE alto (psi_recent_max=11.74,
39/45 features) — reproduz B5 distribution shift documentado iter_0012. Sem novos
requests; sem H do loop resolvida. Detalhe em `iterations/iter_0015_recon_delta.md`.

**Iter 0014 (H11):** LGBM quantile regression para incerteza P10/P50/P90 — **REFUTADO_NE**.
CV walk-forward 5x60d gap 7d sobre features iter_0002 em 12 cells; verdict julgado em NE.
Coverage_band_80 mean: NE **43.6%** (0/3 cells in [70%, 90%]) vs 80% nominal —
under-coverage sistemico em todas as 4 subs (SE 45.4%, S 52.1%, N 47.1%). Causa raiz:
LGBM nao modela heteroscedasticidade + distribution shift (iter_0012 KS p<0.0001).
P50 magnitude OK em NE (delta +4.5% vs LGB-mean) mas banda inutil para "P90 conservador"
do operador. **Bonus inesperado**: P50 quantile BATE LGB-mean em magnitude em N (-18%),
S (-14%), SE (-1.6%) — mediana mais robusta que mean em distribuicoes com cauda longa
de zeros. H26 (conformal post-hoc), H27 (P50 substituto, custo zero) e H28 (NGBoost)
derivadas. Detalhe em `iterations/iter_0014_h11_quantile_regression_ne.md`.

**Iter 0013 (H10):** Ensemble LGBM + persist_d1 com pesos analiticos derivados de
inner_val 30d (zero leak) **CONFIRMADO_NE_SE + bonus N**. CV walk-forward 5x60d gap 7d,
4 esquemas (equal / inv_mae / inv_mse / opt_alpha). Best ensemble bate LGB-only em
maioria das folds em: **NE 3/3 cells** (-12.996 MWh em v1 = 28%, -4.188 em v2 = 13%,
-4.065 em v3 = 13%), **SE 3/3 cells** (-565..-293 MWh = 4-7%), **N 3/3 cells** (-100
MWh = 17% — bonus, persist forte em N como H10 previa), S 2/3 (v3 +26 MWh irrelevante).
Esquemas vencedores: ens_inv_mae 6 cells, ens_inv_mse 4, ens_equal 2, ens_opt_alpha 0
(sobre-otimiza). Pesos analiticos > grid-search empirico. alpha_opt varia 0.0-1.0
entre folds confirmando adaptacao a regime (NE/v1 fold 4 alpha=0.00 = pura persist
quando LGB MAE=43k vs PER=33k). R² NE/v2 sobe 0.22->0.41; N/v1 -0.33->+0.01. H24
(ensemble sobre champions Ridge/LR) + H25 (stacker Ridge meta-modelo) derivadas a
queue. Detalhe em `iterations/iter_0013_h10_ensemble_v2_persist.md`.

| layer | alvo | sub | baseline (MAE_mwh, CV) | best_metric (MAE/R²/F1, modelo) | NMAE secundario | last_iter | sanity_ok | data_utc |
|---|---|---|---|---|---|---|---|---|
| curtailment | d1_ENE_CNF | NE | persist_d1 (UlFor CV 5 folds — MAE pendente extracao) | **ridge_curt_ne_d1 @champion (R² +0.469±0.098 CV; in-sample R²=0.830)** | NMAE 33.7±8.1% | 0011 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE** | 2026-05-24T08:30Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 (UlFor CV 5 folds) | **lr_curt_se_d1 @champion (R² +0.380±0.139 CV; in-sample R²=0.619)** | NMAE 46.6±13.4% | 0011 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE** | 2026-05-24T08:30Z |
| curtailment | d1_ENE_CNF | S | persist_d1 (UlFor CV 5 folds) | **lr_curt_s_d1 @champion (R² +0.447±0.202 CV; in-sample R²=0.725)** | NMAE 89.6±31.2% (NMAE unsafe em test n=11 — S baixo ymean, iter_0008) | 0011 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE** | 2026-05-24T08:30Z |
| curtailment | d1_ENE_CNF | N | persist_d1 (UlFor CV 5 folds) | **ridge_curt_n_d1 v2 @staging (clean_plus, 31 feat; R² +0.196±0.289 CV; in-sample R²=0.472)** FRAGIL atenuado — v2 reduz std -5.6pp vs v1 full | NMAE 84.8±26.2% (era 86.3±31.8% em v1) | 0015 | nao promovivel ainda — staging only (UlFor H10 PARCIALMENTE CONFIRMADA) | 2026-05-24T12:30Z |
| meta | metric_suite | — | NMAE (Principio 6 violado) | **MAE/R²/F1 primario + NMAE secundario com flag** | 3/4 subs (NE,SE,N) conflict NMAE↔R²/F1 em iter_0002 replay; S NMAE unsafe | 0008 | H9 CONFIRMADO | 2026-05-24T06:00Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | NE | persist_d1 | NMAE 35.7% xgb UlFor v3.3 (superseded por ridge_alpha10) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | SE | persist_d1 | NMAE 46.0% xgb UlFor v3.3 (superseded por lr) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | S | persist_d1 | NMAE 109% xgb UlFor v3.3 (superseded por lr -19.4pp) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | N | persist_d1 | NMAE 72.2% lgbm UlFor v3.3 single fold (CV mostra 99.5±37%) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF | NE | persist_d1 (loop replay n=11) | NMAE 28.2% v2 LGBM original / 31.7% holdout strict | superado por UlFor v3.3 | 0002 | [4/5] | 2026-05-24T01:30Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 (loop replay n=11) | NMAE 36.2% v2 LGBM | superado por UlFor v3.3 | 0002 | [3/5] B3 fail | 2026-05-24T01:30Z |
| meta | h2_off_by_one_pdp | — | dbt_join_correto | corr(PDP[t], gen[t])=0.9118 / corr(PDP[t], gen[t+1])=0.8211 | H2 REFUTADO | 0003 | n=484 dias | 2026-05-24T02:30Z |
| meta | sanity_check_B6 | — | n/a | zero_count_shift + signal_collapse implementado e validado (sintetico high+collapse, real SE/v3 lag sign-flip) | adicionado a default pipeline | 0004 | passou | 2026-05-24T03:00Z |
| meta | sanity_check_B6_v1.1 | — | B6 v1.0 (overconfident em n_test=11) | **B6 + n_test<30 downgrade + sign_flip gate \|corr\|>=0.2** | 5/20 FP sint n=10 / 0 perdas n=60 / SE/v3 lag iter_0002 downgrade high->medium | 0009 | regression + revalidation OK | 2026-05-24T06:45Z |
| meta | h3_pdp_residual_signal | NE | r2_gen_only=0.346 | **r2_gen+pdp_prev=0.654 (r2_extra +0.308; partial_corr +0.69)** | perm p=0.0; test/train delta=+0.001 (estavel); leak ok | 0010 | H3 CONFIRMADO | 2026-05-24T07:30Z |
| meta | h3_pdp_residual_signal | SE | r2_gen_only=0.187 | r2_gen+pdp_prev=0.438 (r2_extra +0.251; partial_corr +0.56) | perm p=0.0; test/train delta=+0.002; leak ok | 0010 | H3 CONFIRMADO | 2026-05-24T07:30Z |
| meta | h3_pdp_residual_signal | S | r2_gen_only=0.053 | r2_gen+pdp_prev=0.210 (r2_extra +0.157 train; +0.001 test!) | perm p=0.0; **dist_shift FAIL** (test colapsa, cobertura 12 usinas) | 0010 | H3 fragil em S | 2026-05-24T07:30Z |
| meta | h7_xgb_vs_lgbm_cv | — | persist_d1 por fold | **LGBM > XGB em 10/12 cells (83%) CV 5x60d gap7d**; NE/v3 R² LGBM +0.257 vs XGB -0.178 (inverteu iter_0002 n=11) | NE/v2 ΔMAE+11.299 MWh, SE/v1 wins XGB so 4/5 mag -210 MWh = 2.8% (irrelevante) | 0012 | H7 REFUTADO; [3/3 sanity B3+B4+B5] | 2026-05-24T09:30Z |
| curtailment | ensemble_lgb_persist_d1 | NE | LGBM-only (CV 5x60d) MAE 32.2k v2 | **best=ens_inv_mae MAE 28.0k v2 (-13%), -28% em v1, -13% em v3**; R² NE/v2 0.22->0.41 | NE/v1 LGB catastrofico 46k corrigido por persist (alpha=0.00 fold 4) | 0013 | H10 CONFIRMADO; [B3+B4+B5 done; B1 inherit; B2/B6 N/A] | 2026-05-24T10:30Z |
| curtailment | ensemble_lgb_persist_d1 | SE | LGBM-only (CV 5x60d) MAE 7.74k v2 | best=ens_inv_mae MAE 7.21k v2 (-7%); -7% v1, -4% v3 | wins 3-4/5 folds; magnitude clinica modesta mas consistente | 0013 | H10 CONFIRMADO | 2026-05-24T10:30Z |
| curtailment | ensemble_lgb_persist_d1 | S  | LGBM-only (CV 5x60d) MAE 1.06k v2 | best=ens_inv_mse MAE 1.06k (~0%); v1 -5%, v3 +3% (irrelevante) | regime instavel; alpha varia 0.00-1.00 entre folds | 0013 | H10 PARCIAL (2/3 confirming) | 2026-05-24T10:30Z |
| curtailment | ensemble_lgb_persist_d1 | N  | LGBM-only (CV 5x60d) MAE 0.58k v1-v2 | **best=ens_inv_mae MAE 0.48k (-17%)**; R² LGB -0.33 -> ens +0.01 | bonus H10 — persist forte em N (skill_LGB_vs_persist negativo); ensemble corrige | 0013 | H10 CONFIRMADO (bonus) | 2026-05-24T10:30Z |

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

## Iter 0008 — H9 metric_suite MAE/R²/F1 substitui NMAE como primaria

PLANO_FINAL UlFor Principio 6 adotado: NMAE rebaixada a secundaria.

Resultados ao aplicar metric_suite sobre iter_0002 LGBM replay (n_test=11):

| sub/ver | MAE | R² | F1_p50 | NMAE | NMAE_safe |
|---|---|---|---|---|---|
| NE/v1 | 14862 | -0.167 | 0.000 | 0.501 | true |
| NE/v2 |  8367 | +0.660 | 0.000 | 0.282 | true |
| NE/v3 | 11830 | -0.002 | 0.000 | 0.399 | true |
| SE/v1 |  3599 | +0.467 | 0.857 | 0.409 | true |
| SE/v2 |  3180 | +0.394 | 1.000 | 0.362 | true |
| SE/v3 |  3406 | +0.544 | 0.857 | 0.387 | true |
| S/v1  |    22 |   NaN  | 0.000 |  -    | **false** (ymean<1) |
| S/v2  |   125 |   NaN  | 0.000 |  -    | **false** |
| S/v3  |    80 |   NaN  | 0.000 |  -    | **false** |
| N/v1  |   269 | -1.402 | 0.800 | 1.011 | true |
| N/v2  |   269 | -1.402 | 0.800 | 1.011 | true |
| N/v3  |   259 | -1.637 | 0.750 | 0.974 | true |

**Conflito de ranking detectado em 3/4 subs** (best-per-sub diverge entre NMAE e R²/F1):

| sub | best por NMAE | best por MAE | best por R² | best por F1 |
|---|---|---|---|---|
| NE | v2 | v2 | v2 | **v1** (sobe!) |
| SE | v2 | v2 | **v3** | v2 |
| S  | unsafe | v1 | n/a | tie zero |
| N  | v3 | v3 | **v1** | **v1** |

Decisao: H9 CONFIRMADO. Ranking unico-criterio (NMAE) viesa inferencia.
Suite MAE+R²+F1 primaria (e.g., NE/v2 vence em magnitude mas NE/v1 captura
melhor eventos high-curt; SE/v3 melhor em variancia explicada mas SE/v2 em
event detection). NMAE secundaria com flag `unsafe` quando ymean<1 MWh —
elimina ruido reportado de "S NMAE 109%" que era artefato de denominador
baixo (test n=11 ymean<1).

Patches:
- `scripts/metric_suite.py` — canonical `compute(y_true, y_pred, y_train)` +
  `format_table()` + standalone runner sobre iter_0002.
- `sanity_checks/baseline_compare.py` (B4) — emite `metric_suite_lgbm` +
  `metric_suite_climatologia_doy`. Skill score mantido (back-compat).
- `sanity_checks/holdout_temporal_strict.py` (B3) — emite
  `metric_suite_strict` + `delta_mae_strict_minus_original` ao lado dos
  campos legados. Import limpa; LGBMRegressor run depende de sklearn no env.

Hipotese derivada criada: **H19** (P2) — extrair MAE+R²+F1 do bakeoff
oficial UlFor (mlflow tabela) para refletir na linha "best_metric"
do leaderboard. Hoje so' NMAE oficial e' conhecida — MAE/R²/F1 dos
champions Ridge/LR estao no MLflow mas nao no checkpoint do loop.
Requer req-0007 ao UlFor ou parse direto do MLflow proxy file.

H10 e H11 estavam bloqueadas em H9 — agora unblocked.

## Iter 0007 — H17 SUPERSEDED + champions Ridge/LR absorved

UlFor self-actionou entre iter_0006 e iter_0007 (~5h, sem novo req do loop):

- commit `76732289` (2026-05-23 22:59): Ridge baseline-controle BATE XGB em NE/S
- commit `4e0fc7b4` (2026-05-24 02:15Z): CV walk-forward 5 folds confirma

Champions per-sub mudaram em 3/4 subs (mean ± std, CV 5 folds 60d):

| sub | champion antes (iter_0006) | champion agora (iter_0007 CV) | delta NMAE | delta R² |
|---|---|---|---|---|
| NE | xgb 35.7% / +0.402 | **ridge_alpha10 33.7±8.1% / +0.469±0.098** | -2.0pp | +0.067 |
| SE | xgb 46.0% / +0.386 | **lr_sklearn 46.6±13.4% / +0.380±0.139** | +0.6pp (tied) | -0.006 |
| S  | xgb 109% / -0.135 | **lr_sklearn 89.6±31.2% / +0.447±0.202** | **-19.4pp** | **+0.58 absoluto!** |
| N  | lgbm 72.2% / +0.041 | ridge_alpha10 86.3±31.8% / +0.196±0.289 FRAGIL | +14.1pp | +0.155 |

H17 (P0 promover XGB SE/S a FASE 4) -> **SUPERSEDED_BY_ULFOR_RIDGE_LR_CV**:
premissa XGB invalidada, mas a INTENCAO (promover algo a FASE 4) e' valida —
champions Ridge/LR sao os candidatos reais. Loop NAO emite req-0004 pois
UlFor JA executa @champion registry plan + OOT 2x agendado (autopilot).

Lessons:
- Queue e snapshot temporal. UlFor pode resolver em sessao paralela entre iters.
- Recon-style absorve em vez de duplicar trabalho.
- Single fold n=60d (iter_0006) overestima xgb e subestima lr — CV 5 folds
  e ground truth oficial pos-iter_0007.

H18 nova (P1 methodology, blocked por req-0005): auditar champions Ridge/LR
via B1-B6 antes de FASE 4 final.

## Iter 0009 — H16 B6 robustez n_test (CONFIRMADO)

Patch sanity_checks/zero_count_shift.py: dois mitigations ortogonais.

**(1) `n_test < 30 -> downgrade severity 1 nivel`** (high->medium->low->none).
Severity original preservada em `severity_raw`; flag novo
`downgraded_due_to_small_n_test` marca downgrades para audit.

**(2) `sign_flip` exige `|corr_train| >= 0.2 AND |corr_test| >= 0.2`**.
Gate antigo era `>0.05` em ambos — pega ruido amostral em n=11. `corr_test`
em [0.05, 0.20] com n=11 nao discrimina sinal real de ruido. Flag novo
`sign_flip_blocked_by_min_abs_corr` documenta o que o gate antigo teria
flaggado e o novo bloqueia.

**Regression test sintetico** (`scripts/b6_regression_n_test_threshold.py`):
DGP fraco-positivo (lag1 ~ AR(1), corr alvo ~ +0.25), n_train=365, 20 seeds.

| cenario | old_sign_flip | new_sign_flip | new_downgraded | criterio aceite |
|---|---|---|---|---|
| n_test=10 (small) | 5/20 | 0/20 | 20/20 | 5 FP eliminados |
| n_test=60 (large) | 0/20 | 0/20 | 0/20 | zero regressao em VP |

Todos 4 criterios de aceitacao passam -> verdict **CONFIRMADO**.

**Revalidation iter_0002 runs** (snapshot da causa pratica que motivou H16):

| sub/ver (n_test=11) | n_high | n_med | n_collapse | n_downgrade | sf_blocked |
|---|---|---|---|---|---|
| NE/v1 | 0 (era 0) | 0 (era 4) | 2 | 4 | 0 |
| NE/v3 | 0 (era 0) | 0 (era ~4) | 2 | 8 | 0 |
| SE/v3 | 0 (era 0) | 4 (era 4) | 3 (era 4) | 9 | 1 (curt_lag7) |
| N/v3  | 0 | 0 | 1 | 5 | 3 |

- SE/v3: features raw=high downgrade para medium (preserva sinal mas atenua alarme);
  curt_lag7 sign_flip bloqueado (`ct=0.196 < 0.2`). UlFor req-0003 ja' tinha
  confirmado que esses flips eram amostrais em n=60 — patch agora alinha o
  diagnostico do loop com aquela verdade.
- NE/v1-v3: lag sign_flips PERSISTEM (`|ct|=0.66/0.42`, ambos >>0.2) — gate
  novo nao mascara warnings legitimos onde a correlacao train e' forte.
- N/v3: 3 sign_flips bloqueados — sub com mais falso positivo amostral.

H20 derivada (P3): expor n_test no leaderboard + flag `low_confidence_n_test` no
bake-off runner.

### Lessons learned (cumulativas)

- B6 v1.0 (iter_0004): util mas overconfident em janelas curtas. Gera req
  externa de baixa-confianca.
- B6 v1.1 (iter_0009): downgrade calibrado + gate de sign_flip alinhado com
  corr de UlFor (n>=60). Loop agora pode rodar B6 em replay n=11 sem ruido.
- Padrao geral: sanity checks devem expor `n_test` e ajustar severities por
  potencia estatistica — replica-se em B5 (PSI), B1 (leak corr).

## Iter 0010 — H3 PDP residual signal (CONFIRMADO)

Hipotese H3 testada com OLS contemporaneo `curt ~ gen + PDP` em n=486 dias
(2024-12-01 -> 2026-05-01, sub NE/SE/S; N excluido por cobertura PDP local
zero). Crosswalk inline via mapeamento_conjunto_pdp -> obt_conjunto (~208/606
usinas mapeadas, vs UlFor 82% de 606).

### Resultados por (sub, variant)

| sub | variante | r2_gen | r2_gen+pdp | **r2_extra** | partial_corr | perm p | train→test delta |
|---|---|---|---|---|---|---|---|
| **NE** | pdp_prev | 0.346 | 0.654 | **+0.308** | +0.686 | 0.000 | +0.001 (estavel) |
| NE | pdp_prog | 0.346 | 0.460 | +0.115 | +0.418 | 0.000 | +0.052 |
| **SE** | pdp_prev | 0.187 | 0.438 | **+0.251** | +0.556 | 0.000 | +0.002 (estavel) |
| SE | pdp_prog | 0.187 | 0.192 | +0.005 | -0.074 | 0.098 | +0.000 |
| S  | pdp_prev | 0.053 | 0.210 | +0.157 | +0.408 | 0.000 | **-0.182** (fragil!) |
| S  | pdp_prog | 0.053 | 0.083 | +0.030 | +0.178 | 0.000 | -0.027 |

### Sanity checks (queue requeridos: leak, perm, dist_shift)

- **leak**: PASS. corr(pdp[t], curt[t]) > corr(pdp[t], curt[t-1]) em 5/6
  combinacoes. PDP e' forward-looking, nao back-cast (publicado D-1 -> safe).
- **perm** (500 shuffles): 5/6 com p=0.0 (observado MUITO acima do p99 null
  ~0.005-0.016). Apenas SE/pdp_prog com p=0.098 (mas r2_extra so 0.005,
  irrelevante).
- **dist_shift** (split 80/20 temporal): NE+SE estaveis (|delta|<0.003).
  **S falha** (test r2_extra colapsa de 0.18 para 0.001) — regime change
  curt-S baixo no Dez/2025-Mai/2026 + cobertura PDP-S so 12 usinas (todas
  eolicas, nenhuma solar).

### Interpretacao tecnica

Separacao PDP_prev vs PDP_prog e' a chave:

- **PDP_prog tracking-very-tight de gen** (corr 0.95 NE, 0.75 SE, 0.93 S).
  Sinal redundante com geracao realizada. Programado D-1 ja' incorpora
  dispatch real; mudou pouco apos a operacao.
- **PDP_prev e previsao independente** (publicada D-1), recurso esperado.
  A discrepancia `pdp_prev - gen` correlaciona com curtailment porque
  gen = previsao - restricao. residual e' proxy direta de curt.

### Decisao para o modelo

- **Manter `pdp_prev_eolica_mwh`, `pdp_prev_solar_mwh`** (NE+SE definitivo,
  S condicional). Valor agregado robusto.
- **Considerar dropar `pdp_prog_eolica_mwh`, `pdp_prog_solar_mwh`** —
  colinearidade ~0.95 com `ger_*_mwh`, sinal residual <=0.12.

### Follow-ups

- **H21 (P2 feature)**: engineering `pdp_residual_mwh = pdp_prev - gen` como
  1 canal denso (vs 2 brutos), sanity B1 leak + B2 perm + B4 baseline.
- **H22 (P3 model)**: GBDT-only `curt ~ gen + pdp_prev` vs OLS para medir
  gap nao-linear. Se GBDT gap >= 5pp R², ha interacoes que justificam
  manter features brutas em vez de engineering.

Sem req externo necessario. Crosswalk + parser inline funciona; tabela
`feat_pdp_renovavel` no UlFor (cobertura 82%) ja' faz a agregacao
materializada — H21/H22 podem rodar la' diretamente com mais cobertura.

## Iter 0011 — RECON_DELTA UlFor (4e0fc7b4 -> c8df4077)

7 commits absorvidos. Champions de candidates do iter_0007 viraram
producao real:

| commit | acao | impacto |
|---|---|---|
| `d1fe9777` | CV walk-forward 5x60d (impl) | numeros oficiais iter_0007 materializados; MLflow 140 runs + 28 CV_SUMMARY |
| `83bc79c2` | promote_champions MLflow | ridge_NE/lr_SE/lr_S @champion, ridge_N @staging |
| `3ac5916a` | VIF + CLEAN bake-off | H4 ulfor CONFIRMADA estrutural; CLEAN ajuda NE/N, HURTS SE/S (R² S cai 0.35); FULL default |
| `4d6dd73a` | checkpoint marker 04:00Z | — |
| `d2bf38e4` | VIF greedy iterativo | H8 ulfor REFUTADA — multicolin estat != redundancia preditiva |
| `a7edb1ef` | endpoint /api/forecast/d1 | **PRODUCAO LIVE** — cold 6.5s, warm <50ms, cache 1h modelo + 15min predicao |
| `c8df4077` | investigate_lr_N instability | fold 4 (jul-set/2025) = blowup 208% FULL -> 128% CLEAN; envenenadoras: cmo_range, taxa_penetracao, ter_verif_lag1, carga_mwmed_rmean7; ridge_curt_n_d1@staging continua a defesa |

### O que mudou na nossa interpretacao

- **Champions agora estao em PRODUCAO** (nao mais "candidate aguardando OOT 2x").
  MLflow Registry com aliases setados. Endpoint /api/forecast/d1 servindo
  os 4 subs. linhas `curtailment | d1_ENE_CNF | <sub>` da tabela top
  atualizadas para refletir status PROMOVIDO + in-sample R² + nota LIVE.
- **VIF nao destrava drop universal**. Multicolinearidade massiva confirmada
  (38/55 features VIF>=10, cond_num >1e17), mas drop padrao CLEAN ajuda
  so NE+N. feature_set=full continua default. CLEAN candidato Staging
  NE proximo round (potencial robustez +1pp stdev / NMAE -0.5pp).
- **lr_N instabilidade isolada por fold**: fold 4 (jul-set/2025, train=221d,
  mais antigo) puxa stdev=65.4%. CLEAN reduz stdev 52.4->22.7pp.
  Envenenadoras especificas N: cmo_range, taxa_penetracao, ter_verif_lag1,
  carga_mwmed_rmean7. Em CLEAN ainda aparecem curt_lag1/curt_rmean7
  instaveis (N tem muitos zeros estruturais -> OLS extrapola mal).
  ridge_curt_n_d1@staging continua. Possivel `feature_set=clean_plus_n`
  proximo round.

### Reqs / hipoteses

- Sem novos requests pendentes. Os 3 antigos (req-0001/2/3) continuam DONE.
- Sem novas hipoteses do **loop** geradas. UlFor numera seus proprios H1/H2/H4/H8/H9
  no PLANO_FINAL — **nao confundir**. Em particular: commit `c8df4077` diz
  "H9 RESPONDIDA" referindo-se ao H9 ulfor (lr_N), **nao** ao nosso H9
  (metric_suite MAE/R²/F1, iter_0008 CONFIRMADO).
- Status H18 (sanity B1-B6 sobre champions Ridge/LR) ainda blocked por
  req-0005 — sanity local requer ou (a) UlFor publicar predicoes em
  parquet acessivel ou (b) loop ganhar acesso ao MLflow tracking URI.
  Candidato a req-0004 (dump MLflow CV_SUMMARY) **nao** emitido nesta
  iter — auto-pesado, evitar duplicar trabalho ja resumido em FINDING.

### Proxima iter

`iter_0012` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`). Derivada de H3 iter_0010, codavel
localmente, sem dep externa. Alt: H10, H22, H19.

## Iter 0013 — H10 Ensemble LGBM + persist_d1 (CONFIRMADO_NE_SE)

Hipotese H10 (P2): ensemble simples LGBM + persist_d1 com pesos derivados de
skill em CV pode dominar LGBM puro em subs onde persistencia carrega muito
sinal. Alvo explicito do queue = NE+SE; detail H10 destaca persist forte em N.

CV walk-forward 5 folds (60d cada, gap 7d) sobre features iter_0002 nas 12
cells (4 subs x 3 vers). Inner split adicional: ultimos 30d do train viram
inner_val para derivar pesos do ensemble SEM leak de test. 4 esquemas de peso:
`ens_equal`, `ens_inv_mae` (w∝1/MAE), `ens_inv_mse` (BMA Gaussian),
`ens_opt_alpha` (grid 0..1 step 0.05 argmin MAE inner_val).

### Resultados (best ensemble vs LGB-only, deltas MAE em MWh; negativo=ganho)

| cell  | LGB MAE | PER MAE | best ens MAE | best_scheme    | delta vs LGB | wins  | R² LGB | R² best |
|-------|--------:|--------:|-------------:|----------------|-------------:|-------|-------:|--------:|
| NE/v1 |  45.959 |  33.722 |       32.963 | ens_inv_mse    |     -12.996  | 5/5   | -0.634 | +0.187  |
| NE/v2 |  32.181 |  33.837 |       27.993 | ens_inv_mae    |      -4.188  | 4/5   | +0.217 | +0.412  |
| NE/v3 |  31.593 |  33.837 |       27.528 | ens_inv_mae    |      -4.065  | 3/5   | +0.257 | +0.423  |
| SE/v1 |   7.752 |   8.328 |        7.186 | ens_equal      |        -565  | 3/5   | -0.135 | -0.001  |
| SE/v2 |   7.736 |   8.328 |        7.211 | ens_inv_mae    |        -525  | 4/5   | -0.081 | +0.004  |
| SE/v3 |   7.222 |   8.328 |        6.929 | ens_inv_mse    |        -293  | 3/5   | +0.074 | +0.104  |
| S/v1  |   1.157 |   1.266 |        1.104 | ens_equal      |         -53  | 3/5   | -0.291 | -0.151  |
| S/v2  |   1.064 |   1.268 |        1.058 | ens_inv_mse    |          -6  | 3/5   | -0.168 | -0.118  |
| S/v3  |   0.977 |   1.268 |        1.003 | ens_inv_mse    |         +26  | 3/5   | -0.007 | -0.005  |
| N/v1  |   0.578 |   0.513 |        0.478 | ens_inv_mae    |        -100  | 4/5   | -0.334 | +0.012  |
| N/v2  |   0.578 |   0.513 |        0.478 | ens_inv_mae    |        -100  | 4/5   | -0.334 | +0.012  |
| N/v3  |   0.553 |   0.513 |        0.475 | ens_inv_mae    |         -78  | 4/5   | -0.164 | +0.028  |

Sub-summary (cells confirming / total):
- **NE 3/3** — best delta -12.996 MWh em v1 (LGB catastrofico 46k corrigido
  por persist via fold 4 alpha=0.00 = pura persist). v2 (cell alvo H10)
  -4.188 MWh = 13% reduction.
- **SE 3/3** — magnitude menor mas wins 3-4/5 folds.
- **N 3/3 (bonus)** — confirmacao do mecanismo H10 detail: persist forte
  em N corrige LGB-pior-que-persist. R² LGB -0.33 -> ensemble +0.01.
- **S 2/3** — v3 +26 MWh (2.7%, irrelevante em escala 1k MWh).

### Esquemas de peso — ranking por frequencia de win

| esquema         | n_cells_won | nota                                                     |
|-----------------|------------:|----------------------------------------------------------|
| ens_inv_mae     |           6 | mais robusto (NE/v2-v3, SE/v2, N/v1-v2-v3)              |
| ens_inv_mse     |           4 | BMA Gaussian (NE/v1, SE/v3, S/v2-v3)                    |
| ens_equal       |           2 | surpresa positiva (SE/v1, S/v1)                          |
| ens_opt_alpha   |           0 | grid-search sobre-otimiza inner_val 30d, generaliza pior |

**Insight**: pesos analiticos proportional-to-precision >> minimo empirico
em inner_val pequeno. Consistente com BMA classico.

### Adaptacao do alpha por fold

alpha_opt varia 0.0-1.0 entre folds da mesma cell — confirma que pesos
respondem a regime shift documentado em iter_0012 B5 (KS p<0.0001 NE+SE).

| cell  | alphas por fold (1..5)                | interpretacao                              |
|-------|---------------------------------------|--------------------------------------------|
| NE/v1 | 0.70, 0.30, 0.70, 0.00, 0.15          | fold 4 LGB MAE=43k vs PER=33k -> pura persist |
| NE/v2 | 0.80, 0.70, 0.75, 0.75, 0.50          | LGB confiavel, peso LGB > peso persist     |
| NE/v3 | 0.75, 0.75, 0.90, 0.80, 0.50          | LGB carry, mix em fold 5 (PER>LGB)         |
| SE    | 0.55-0.90                             | LGB carry com pequeno hedge persist        |
| S/v*  | 0.00 a 1.00                           | regime instavel; fold 1 LGB suprime curto  |
| N     | 0.20-1.00                             | LGB e persist similares em folds 1-2       |

### Sanity checks (queue requeridos: baseline, holdout)

- **B1 leak**: INHERITED iter_0002 features ja auditadas; persist_d1 =
  y_d1[i-1] D-1 safe por construcao.
- **B2 perm**: N/A — ensemble e' meta-modelo de 2 ponteiros, sem features.
- **B3 holdout strict**: DONE_VIA_CV — 12 cells x 5 folds = 60 holdouts,
  gap 7d entre train e test, inner_val 30d sem overlap com test.
- **B4 baseline_compare**: DONE_INTEGRADO — persist_d1 e' componente direto
  do ensemble; skill vs persist computado em todas as 12 cells (skill +0.07
  a +0.21).
- **B5 dist_shift**: INHERITED iter_0012 + evidencia adicional
  `weights_distribution.csv` (alpha varia 0.0-1.0 entre folds = ensemble
  responde a shift).
- **B6 zero_count**: N/A — nao introduz features.

Coverage total documentada em
`outputs/iter_0013/h10_ensemble_v2_persist/sanity_summary.json`.

### Decisao

CONFIRMADO_NE_SE. Ensemble (LGBM + persist_d1, pesos inv_mae ou inv_mse)
deve ser POST-PROCESSING DEFAULT para forecast curt D+1 no loop. Sem req
externo emitido — champions UlFor sao Ridge/LR; H10 testou LGBM (replay
loop). H24 deriva: aplicar mesmo esquema sobre champions Ridge/LR.

### Hipoteses derivadas

- **H24** (P2): mesmo ensemble aplicado a champions Ridge/LR UlFor — ganho
  similar? Implementacao codavel local (Ridge_alpha10 + LR_sklearn no
  replay sobre features iter_0002).
- **H25** (P3): stacker Ridge meta-modelo sobre [LGB, persist, ma7,
  climatologia] supera weighted average analitico?

### Proxima iter

`iter_0014` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`). Alt: H24 (ensemble champions, derivada
de hoje), H11 (quantile, unblocked por H9), H22 (GBDT vs OLS gap), H19
(extrair MAE/R²/F1 champions).


## Iter 0014 — H11 LGBM Quantile Regression NE (REFUTADO_NE)

Hipotese H11 (P2): LightGBM com objective='quantile' (alphas 0.1, 0.5, 0.9).
Substituir ponto-estimativa por bandas — util para downstream (operador
escolhe P90 conservador). Alvo explicito do queue = NE.

CV walk-forward 5 folds (60d cada, gap 7d) sobre features iter_0002 nas 12
cells (4 subs x 3 vers; verdict julgado APENAS em NE; SE/S/N viram bonus).
Cada fold treina 4 modelos no mesmo train: mdl_mean (objective='regression'
= baseline point) + mdl_q01 + mdl_q05 + mdl_q09 (quantile per alpha).

### Resultados (mean across 5 folds, por cell — 12 cells)

| cell  |  PB10  |  PB50  |  PB90  | cov80 | cov10 | cov90 | width  | cross | P50/mean Δ |
|-------|-------:|-------:|-------:|------:|------:|------:|-------:|------:|-----------:|
| NE/v1 |   7937 |  23198 |  18589 | 45.5% | 15.1% | 60.6% |  66276 |  1.7% |   -0.6%    |
| NE/v2 |   6960 |  17091 |  13870 | 43.5% | 14.0% | 57.5% |  57228 |  6.7% |   +5.6%    |
| NE/v3 |   7046 |  17002 |  14114 | 41.8% | 16.4% | 58.2% |  53463 |  6.8% |   +8.4%    |
| SE/v1 |   1725 |   3784 |   2569 | 48.2% | 25.1% | 73.3% |  11319 |  4.0% |   -1.1%    |
| SE/v2 |   1757 |   3853 |   2444 | 47.5% | 24.4% | 71.9% |  11957 |  3.0% |   -0.7%    |
| SE/v3 |   2034 |   3484 |   2363 | 40.5% | 30.2% | 70.6% |   9579 | 13.4% |   -2.9%    |
| S/v1  |    122 |    507 |    459 | 53.1% | 28.8% | 79.3% |   1985 |  9.7% |  -16.5%    |
| S/v2  |    127 |    482 |    415 | 51.6% | 28.9% | 78.5% |   1674 | 13.0% |  -15.6%    |
| S/v3  |    128 |    472 |    355 | 51.5% | 29.3% | 78.1% |   1753 | 14.5% |   -9.2%    |
| N/v1  |     75 |    233 |    122 | 47.5% | 37.2% | 84.6% |   1011 |  2.0% |  -18.2%    |
| N/v2  |     75 |    233 |    122 | 47.5% | 37.2% | 84.6% |   1011 |  2.0% |  -18.2%    |
| N/v3  |     83 |    229 |    109 | 46.5% | 39.2% | 85.6% |    996 |  2.0% |  -17.1%    |

cov80 nominal = 80%; cov10 nominal = 10%; cov90 nominal = 90%.
Δ = (MAE_P50 - MAE_LGB_mean) / MAE_LGB_mean (negativo = P50 melhor).

### Coverage por sub (mean)

| sub | cov_band_80 | cov_10 | cov_90 | folds in [70%, 90%] | diagnostico                        |
|-----|------------:|-------:|-------:|--------------------:|------------------------------------|
| NE  |      43.6%  | 15.2%  | 58.8%  |  0/15               | severe under-coverage              |
| SE  |      45.4%  | 26.6%  | 71.9%  |  1/15               | severe under-coverage              |
| S   |      52.1%  | 29.0%  | 78.6%  |  1/15               | under-coverage (medio P10 alto)    |
| N   |      47.1%  | 37.9%  | 84.9%  |  3/15               | melhor cov_90, mas cov_10 explode  |

### Fold heterogeneity (cov_band por fold, NE/v1 exemplo)

| fold | window           | cov_band | LGB mean MAE | y_te mean | comentario                |
|------|------------------|---------:|-------------:|----------:|---------------------------|
|   1  | 2025-09→2025-11  |    62%   | 39383        |       51k | regime mais estavel       |
|   2  | 2025-11→2026-01  |    27%   | 62065        |       72k | transicao curt-up         |
|   3  | 2026-01→2026-03  |    40%   | 58850        |       67k | transicao continua        |
|   4  | 2026-03→2026-05  |    47%   | 43185        |       46k | novo regime, estabiliza   |
|   5  | 2026-05→2026-07  |    52%   | 26311        |       32k | regime estavel novo       |

Reproduz B5 distribution shift documentado em iter_0012 (NE+SE KS p<0.0001).

### Sanity checks (queue requeridos: holdout, baseline, dist_shift)

- **B1 leak**: SKIPPED — features identicas iter_0002, ja auditadas em
  iter_0010 (PDP_prev forward-looking, p_perm=0.0).
- **B2 perm**: SKIPPED — PI iter_0010 cobre 5/6 cells com p=0.0.
- **B3 holdout strict**: PASSED_EMBEDDED — gap=7d em todas folds; n_test
  58-60 em todas as folds, zero overlap.
- **B4 baseline_compare**: PASSED_EMBEDDED — P50 comparada a persist_d1 E
  LGB-mean por fold. delta_mae_p50_vs_mean_pct mean NE = +4.5% (passa).
  Bonus: P50 BATE LGB-mean em N/S/SE (-18% a -1%).
- **B5 dist_shift**: ANNOTATED_REUSE — evidencia iter_0012 (KS p<0.0001).
  Padrao fold-a-fold do under-coverage confirma diagnostico.
- **B6 zero_count**: PASSED_EMBEDDED — n_test >= 58 (threshold downgrade=30),
  sem zero-only folds.

Coverage total em `outputs/iter_0014/h11_quantile_regression_ne/sanity_summary.json`.

### Decisao

**REFUTADO_NE.** Bandas LGBM quantile com defaults sistemicamente
UNDERCOVERED em todas as 4 subs (cov_band 41-53% vs 80% nominal).
Operador escolhendo "P90 conservador" estaria errado em ~30% dos casos
criticos. Nao virar deliverable v1.0 nesta forma.

Causa raiz: LGBM nao modela heteroscedasticidade explicita (variancia
vem de variancia em-amostra), e' insuficiente sob distribution shift
fold-a-fold documentado em iter_0012. Sem req externo — UlFor nao
desbloqueia mudanca de arquitetura.

P50 magnitude OK em NE (delta +4.5% vs LGB-mean, passa B4); bonus em
N/S/SE: P50 BATE LGB-mean (delta -1% a -18%) — mediana mais robusta
que mean em distribuicoes com cauda longa de zeros.

### Hipoteses derivadas

- **H26** (P3): conformal prediction post-hoc — calibra banda LGBM via
  nonconformity score do inner_val. Goal cov_band in [75%, 85%] sem
  retreinar modelo. Likely-fix se atacarmos H11 de novo.
- **H27** (P3): P50 quantile como POINT ESTIMATE substituto em N+S.
  P50 BATE LGB-mean em magnitude com custo zero (objective swap). Ganho
  transversal pequeno mas universal nos subs com cauda longa.
- **H28** (P3): NGBoost vs LGBM quantile. Distribuicao parametrica
  (Normal/Lognormal) lida com heteroscedasticidade que LGBM quantile
  nao captura. Bench worktree ja tem NGBoost similar (per H11 detail).

### Proxima iter

`iter_0015` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`, derivada H3 confirmado). Alt: H24
(ensemble champions Ridge/LR), H27 (P50 substituto, derivada hoje,
ganho baixo custo), H26 (conformal, fix de H11).

## Iter 0015 — RECON_DELTA UlFor (c8df4077 -> 5dacb5a2)

7 commits absorvidos. Champion N teve upgrade marginal de versao (v1 full ->
v2 clean_plus). Fase 4 observabilidade (validate_d1 + drift PSI + Telegram
alert) FECHADA pelo UlFor em paralelo aos iters 0012-0014 do loop:

| commit | acao | impacto |
|---|---|---|
| `0481a5a6` | checkpoint marker 05:15Z | "H8 ulfor refutada, endpoint operacional, H9 ulfor respondida" — todos Hs UlFor internos (Fase 3), nao do loop |
| `c8e27784` | H10 ulfor clean_plus + endpoint feature-set aware | **MUDANCA DE CHAMPION N**: ridge_curt_n_d1 v2 (clean_plus, 31 feat) @staging substitui v1 (full, 55 feat). NMAE 86.3±31.8% -> **84.8±26.2%** (-1.5pp mean, **-5.6pp std**). lr_N nao promovido. UlFor H10 = CLEAN ∪ {cmo_range, ter_verif_lag1, carga_mwmed_rmean7} PARCIALMENTE CONFIRMADA. `loader.py` agora le `feature_set` do MLflow run params e aplica drops em `build_inference_row` (backward-compat v1 via param `55_feat_*_v3.3 -> full`) |
| `c0e80193` | validate_d1.py foundation | **FASE 4 STEP 1**. Replay-predict ultimos N dias usando features as-of D + comparacao vs realized y_d1 e baseline persist D-1. MLflow experiment `ulfor-validation-d1`. Smoke 14d: NE 49.2% +9% skill, SE 55.4% +20%, **S 137% -37% (alerta)**, N 60.5% +45% |
| `26617ba5` | checkpoint marker 06:15Z | "Fase 3 fechada, Fase 4 iniciada" |
| `9d7652de` | Dagster asset + schedule 07h BRT | Asset `forecast/validate_d1` + schedule `forecast_validate_d1_daily` (10h UTC, **STOPPED**). Subprocess pro venv raiz (mlflow nao em dataops). Smoke 7d: NE 40.5% +29%, SE 71.7% +37%, **S 184.3% -41% (alerta)**, N 56.6% +59% |
| `9873e3c8` | drift PSI + Telegram alert | **FASE 4 STEPS 2+3 (Opcao C closure)**. Evidently ABANDONADO (conflito `evidently>=0.5 depends on plotly<6` vs nosso pin `plotly>=6.5`). PSI nativo numpy/scipy: bins adaptativos n/3 max 10, Laplace smoothing, dual `psi_long`+`psi_recent`, SEASONAL_FEATURES excluidas. Telegram standalone com 4 gatilhos. Window default 60d. Smoke: **NE psi_recent_max=11.74 39/45 features** drift |
| `5dacb5a2` | checkpoint marker 07:15Z | "Fase 4 step 1 fechada (Dagster+drift+Telegram)" |

### O que mudou na nossa interpretacao

1. **Champion N evoluiu silenciosamente**. Iter_0011 marcou ridge_N@staging
   v1 (full); 1 commit depois, v2 clean_plus. Tracking versao do champion
   no leaderboard agora obrigatorio (nao so' modelo + NMAE — incluir
   `feature_set` + `versao MLflow`). Linha N atualizada com `v2 (clean_plus, 31 feat)`.

2. **Pipeline observabilidade Fase 4 inteira PRONTA**. Champions tem 3
   camadas de validacao continua quando schedule for ativado: replay
   diario (`validate_d1`), drift PSI (`compute_drift`), Telegram alert
   (`alert.py`). **Acao Breno**: gerar `BRAZILGRID_TELEGRAM_BOT_TOKEN`
   + `BRAZILGRID_TELEGRAM_CHAT_ID` (mesma convencao `sintegre_freshness_alert`)
   + ativar `forecast_validate_d1_daily` no Dagster UI.

3. **Smoke validate_d1 confirma fragilidade lr_S**. Janelas 7d e 14d:
   skill_vs_persist NEGATIVO (-37 a -41%), NMAE 137-184%. CV 5x60d
   ainda mostra +0.447 R² mas operacionalmente o modelo colapsa em
   janelas curtas. Valida flag "FRAGIL" no leaderboard.

4. **Drift PSI confirma B5 distribution shift iter_0012**. NE smoke 60d:
   psi_recent_max=11.74 (vs industry-std threshold 0.2), 39/45 features
   driftando. Top: rolling-means de carga e CMO. Threshold calibrado em
   `>1.0` (vs std 0.2) sera recalibrado em 2 semanas de operacao real.

### Reqs / hipoteses

- Sem novos requests pendentes. Os 3 antigos (req-0001/2/3) continuam DONE
  (verificado contra `git show 5dacb5a2:coordination/loop_requests.md`).
- Sem novas hipoteses do **loop** geradas. Mudancas absorvidas referem-se
  a Hs UlFor internos (PLANO_FINAL). **Critico**: UlFor H10 (clean_plus
  FEATURE_DROPS_N) != nosso H10 (LGBM+persist ensemble, iter_0013).
  Disambig persistente em `state.json.nota_nomenclatura`.
- **H18 (sanity B1-B6 sobre champions)**: bloqueio segue, mas urgencia
  diminui — pipeline validate_d1 + drift PSI da cobertura empirica
  continua que reduz valor marginal do B1-B6 audit local.
- **H19 (extrair MAE/R²/F1 dos champions)**: atratividade cresce — daily
  summary JSON do validate_d1 tem MAE/R²/skill por sub. Quando schedule
  ativar + loop ganhar acesso MLflow URI, custo H19 cai significativamente.

### Proxima iter

`iter_0016` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`). Derivada de H3 iter_0010, codavel
localmente, sem dep externa. Razoes inalteradas desde iter_0014/0015.
Considerar usar ensemble post-processing (H10 nosso, iter_0013 confirmado)
+ LGBM (iter_0012 confirmado GBDT padrao) para baseline final do bake-off
H21. Alt: H27 (P50 substituto custo zero), H24 (ensemble Ridge/LR), H19
(MAE/R²/F1 dos champions — agora parseavel via summary JSON UlFor).

