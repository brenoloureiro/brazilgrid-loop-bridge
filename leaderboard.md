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

| layer | alvo | sub | baseline (MAE_mwh, CV) | best_metric (MAE/R²/F1, modelo) | NMAE secundario | last_iter | sanity_ok | data_utc |
|---|---|---|---|---|---|---|---|---|
| curtailment | d1_ENE_CNF | NE | persist_d1 (UlFor CV 5 folds — MAE pendente extracao) | **ridge_curt_ne_d1 @champion (R² +0.469±0.098 CV; in-sample R²=0.830)** | NMAE 33.7±8.1% | 0011 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE** | 2026-05-24T08:30Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 (UlFor CV 5 folds) | **lr_curt_se_d1 @champion (R² +0.380±0.139 CV; in-sample R²=0.619)** | NMAE 46.6±13.4% | 0011 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE** | 2026-05-24T08:30Z |
| curtailment | d1_ENE_CNF | S | persist_d1 (UlFor CV 5 folds) | **lr_curt_s_d1 @champion (R² +0.447±0.202 CV; in-sample R²=0.725)** | NMAE 89.6±31.2% (NMAE unsafe em test n=11 — S baixo ymean, iter_0008) | 0011 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE** | 2026-05-24T08:30Z |
| curtailment | d1_ENE_CNF | N | persist_d1 (UlFor CV 5 folds) | **ridge_curt_n_d1 @staging (R² +0.196±0.289 CV; in-sample R²=0.472)** FRAGIL — lr_N investigado iter_0011, fold 4 blowup 208% | NMAE 86.3±31.8% | 0011 | nao promovivel ainda — staging only | 2026-05-24T08:30Z |
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
