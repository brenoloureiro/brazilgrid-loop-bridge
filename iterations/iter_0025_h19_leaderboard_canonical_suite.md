---
alvo: leaderboard_canonical_suite_h19_materialization
layer: meta
iter_num: 0025
type: metric_consolidation_manual
data_utc: 2026-05-24T20:30:00Z
hypothesis: |
  H19 (P2 metric, layer=meta) — Extrair MAE/R²/F1 dos champions UlFor para
  o leaderboard. Verdict iter_0016=CONFIRMADO_PARCIAL (extracao parcial via
  parser FINDING + derivacao MAE), iter_0017 COMPLETED (parquet exato via
  req-0007). Esta iter MATERIALIZA o resultado em leaderboard.md com suite
  canonica COMPLETA (Champions+Baselines+Overlay+Sucessores) — Breno pediu
  explicitamente, fora do planner_config.next_iter_should_be normal.
baseline_tipo: |
  Leaderboard atual (HEAD apos iter_0023 daemon) tem suite canonica nos
  champions mas a estrutura cresceu organicamente em snippets historicos
  ("Iter 0022", "Iter 0021", "Iter 0008"...) que misturam champions de
  producao, replays loop deprecados (n=11) e candidatos sucessores. Falta:
  (a) tabela canonica top com champion+suite-completa side-by-side; (b)
  tabela canonica baselines (persist_d1, persist_d7, ma7) com MAE/R²/F1
  exatos do parquet UlFor; (c) `skill_vs_persist_d1` explicito; (d)
  separacao limpa entre "ativo" e "deprecado".
baseline_metric: |
  Apos iter_0017 (req-0007 closure), parquets UlFor
  `cv_summary_mean_{full,clean_plus}.parquet` carregam metricas EXATAS:
    NE  ridge  full        MAE 27317 ± 11920 / R² +0.469 / F1 0.808 / NMAE 33.7%
    SE  lr     full        MAE  6117 ±   873 / R² +0.383 / F1 0.785 / NMAE 46.5%
    S   lr     full        MAE   805 ±   441 / R² +0.371 / F1  NaN  / NMAE 87.2%
    N   ridge  clean_plus  MAE   425 ±   149 / R² +0.170 / F1 0.790 / NMAE 85.3%
  Baselines persist_d1 (mesmo CV, mesma fonte):
    NE 33183 / SE 9273 / S 1230 / N 508 MWh (MAE)
  Antes desta iter o leaderboard tinha esses numeros DENTRO de paragrafos
  longos misturados com narrativa de iter_0017+0018+0021. Suite canonica
  side-by-side ausente; skill_vs_persist nao explicitado.
result_metric: |
  CONFIRMADO_PARCIAL_MATERIALIZADO. leaderboard.md REESCRITO com:

  ## Estrutura final (top-down):

  1. **Champions oficiais UlFor** — 1 linha por sub com 11 colunas:
     modelo, feature_set, n_feat, MAE±std, R²±std, F1±std, RMSE±std,
     NMAE±std, skill_vs_persist (computado), bias±std, nmae_safe.

  2. **Baselines (persist_d1, persist_d7, ma7)** — 3 linhas por sub × 4
     subs = 12 linhas. Mesma suite menos skill+nmae_safe.

  3. **Overlay producao (bias_correction)** — tabela 4-sub mostrando
     default ON/OFF + janela + impacto CV NMAE + 14d real NMAE. Documenta
     que `predicted_curt_mwh_default` em NE+N e' raw - bias_28d/60d.

  4. **Candidatos sucessores** (NAO promovidos):
     - h22_per_fold (iter_0021 UlFor): NE ridge -2.3pp, SE ridge -2.5pp
     - h22_model_aware (iter_0023 UlFor): SE lr preserves family -9.1pp
       NMAE vs h22 universal
     - H14-G (iter_0023 UlFor): N w=14d k=1.5 -31.87pp vs H14-B

  5. **Sucessores via ensemble (H24 loop)** — iter_0022 CONFIRMADO_3SUBS:
     NE -24%, SE -24%, N -19% MAE vs champion-only.

  6. **Historico iter loop (deprecado)** — replays n=11 movidos pra
     secao no rodape com 1 linha por iter.

  7. **Lessons learned + Como atualizar** — protocol para futuros iters.

  ## Skill_vs_persist_d1 (computado direto):

  | sub | MAE_champion | MAE_persist | skill | interpretacao |
  |---|---:|---:|---:|---|
  | NE | 27317 | 33183 | **+17.7%** | champion bate persist CV puro (sem bias_corr) |
  | SE |  6117 |  9273 | **+34.0%** | champion bate persist CV puro |
  | S  |   805 |  1230 | **+34.5%** | champion bate persist CV puro |
  | N  |   425 |   508 | **+16.4%** | champion bate persist CV puro |

  **TODAS as 4 subs**: champion bate persist em MAE no CV (independente de
  bias_correction). Em 14d real, NE+SE batem persist apenas com
  bias_correction ON (iter_0017/0018); S+N tem teto de dados D+1
  documentado (iter_0017 fase4_v3_curtailment_d1_data_ceiling).
fonte:
  parquets:
    - C:/Projetos/brazilgrid-ulfor/experiments/bakeoff_curtailment_multisub/outputs/cv_summary_mean_full.parquet
    - C:/Projetos/brazilgrid-ulfor/experiments/bakeoff_curtailment_multisub/outputs/cv_summary_mean_clean_plus.parquet
    - C:/Projetos/brazilgrid-ulfor/experiments/bakeoff_curtailment_multisub/outputs/cv_summary_per_fold_full.parquet (RMSE+bias agregados)
    - C:/Projetos/brazilgrid-ulfor/experiments/bakeoff_curtailment_multisub/outputs/cv_summary_per_fold_clean_plus.parquet
  ulfor_commits_extract:
    - 6b21ffdf (req-0007 closure, F1_p50 + per-fold parquets)
    - c8e27784 (champion N v2 clean_plus, iter_0015)
    - 2daa5d40 (H22_model_aware SE candidate, iter_0023)
    - 1ba9cb39 (H14-G N candidate, iter_0023)
    - c5004bac (CHAMPION_DECISION_MATRIX, iter_0023)
  zero_retrain: true
budget_horas: 1.4
sanity_checks_required: []
sanity_checks_done:
  - leaderboard_complete: true
  - persist_baseline_present: true
  - skill_vs_persist_computed: true
  - rmse_extracted_per_fold: true
  - bias_extracted_per_fold: true
  - nmae_safe_flagged: true
---

# Iter 0025 — H19 leaderboard canonical suite (MATERIALIZADO)

## Objetivo

Materializar H19 (extracao MAE/R²/F1/RMSE/skill/bias dos champions UlFor)
em uma estrutura de leaderboard CANONICA, separando claramente:

- **Status quo producao** (champion + overlay bias_correction)
- **Baselines** (persist_d1, persist_d7, ma7) com mesma suite
- **Candidatos sucessores** aguardando decisao Breno (h22, h22_MA, H14-G,
  ensemble H24)
- **Historico iter loop deprecado** (replays n=11 iter_0002..0006)

H19 verdict iter_0016 era CONFIRMADO_PARCIAL com MAE_derived (~10% slack);
iter_0017 req-0007 fechou com numeros EXATOS no parquet. O que estava
faltando era estrutura — leaderboard cresceu organicamente com snippets
por iter. Breno pediu explicitamente padronizacao com suite canonica
side-by-side.

Esta iter executa **PHASE B/C/E/F** do plan original (PHASE D state.json
DEFERRED — explicado abaixo).

## Champions absorvidos + fonte

| sub | modelo | feature_set | MAE (MWh) | R² | F1_p50 | RMSE (MWh) | NMAE | skill | bias (MWh) | fonte |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| NE | ridge_alpha10 | full | 27 317 ± 11 920 | +0.469 ± 0.098 | 0.808 ± 0.170 | 34 516 ± 13 773 | 33.7 ± 8.1% | +17.7% | −13 549 ± 14 503 | `cv_summary_mean_full.parquet` |
| SE | lr_sklearn | full | 6 117 ± 873 | +0.383 ± 0.094 | 0.785 ± 0.108 | 8 098 ± 1 124 | 46.5 ± 13.5% | +34.0% | −839 ± 3 043 | `cv_summary_mean_full.parquet` |
| S | lr_sklearn | full | 805 ± 441 | +0.371 ± 0.164 | NaN | 1 289 ± 513 | 87.2 ± 22.6% | +34.5% | +42 ± 255 | `cv_summary_mean_full.parquet` |
| N | ridge_alpha10 | clean_plus | 425 ± 149 | +0.170 ± 0.185 | 0.790 ± 0.048 | 535 ± 134 | 85.3 ± 27.6% | +16.4% | +70 ± 236 | `cv_summary_mean_clean_plus.parquet` |

RMSE e bias agregados localmente do `per_fold` parquet (UlFor parquet mean
nao expoe RMSE/bias diretamente — extraidos via groupby por (sub, model)
com pandas).

Skill_vs_persist computado in-place: `1 - MAE_champion / MAE_persist_d1`
no MESMO CV (5 folds × 60d gap 7d).

## Conflitos de ranking entre metricas

**Zero conflitos** ao nivel champion-vs-baseline: todas as 4 subs concordam
que champion > persist_d1 em MAE+R²+F1+RMSE+NMAE+skill. Ranking unico.

Conflitos PERSISTEM intra-sub (ridge vs lr, full vs clean_plus) mas
documentados em handoffs anteriores:
- NE em clean_plus: ridge (NMAE 33.2%) vs lr (34.7%) — ridge venceu por
  NMAE+std; F1 empate 0.802 vs 0.799. Champion mantido ridge_full.
- SE em h22_per_fold: ridge venceu sobre lr_full (R² +0.390 vs +0.302).
  iter_0023 H22_model_aware corrige isso preservando familia LR
  (R² +0.381 com lr+h22_MA).
- N em clean_plus: ridge venceu lgbm marginalmente (MAE 425 vs 455).

## Skill vs persist por sub

```
NE: +17.7%  champion 27 317 vs persist 33 183  delta -5 866 MWh
SE: +34.0%  champion  6 117 vs persist  9 273  delta -3 156 MWh
S : +34.5%  champion    805 vs persist  1 230  delta -425 MWh
N : +16.4%  champion    425 vs persist    508  delta -83 MWh
```

Em todos os 4 subs champion bate persist no CV PURO (sem bias_correction
overlay). Em 14d real:
- NE: raw NMAE 59.2% > persist 54.1% (perde 5pp), MAS corrected 49.2%
  bate persist por 4.9pp (bias_correction_28d ON default desde iter_0017).
- SE: data ceiling — nenhum ML bate persist em 14d real no regime atual
  (iter_0017 UlFor H21 evidenciou; bias_correction nao destrava).
- S: lr_S em colapso operacional em 7-14d (validate_d1 skill -37 a -41%).
- N: smoke pred 246 → default 267 com bias_correction_60d ON
  (iter_0018); 14d real nao reportado consolidado ainda.

## Pendencias (qualquer metrica nao extraida)

1. **bias_correction overlay em CV**: parquet UlFor reporta champion *raw*.
   Valores corrigidos so' em logs 14d real. Para fechar — candidato a
   **req-0008** ao UlFor (P3 ~30min): rodar `validate_d1` em janela
   CV-completa e logar `mae_corrected_per_fold` ao lado de `mae_per_fold`.
2. **F1_p50 em S**: NaN estrutural (P50_train=0). Alternativa: threshold
   P75 ou P90 — **H32 emergente** (P3 ~0.5h) candidata. NAO criada
   formalmente nesta iter; deixada como nota.
3. **Holdout 14d real para candidatos h22_per_fold NE+SE**: pendente em
   **H31 emergente** (req-0008 alternativo). Validation gap em
   CHAMPION_DECISION_MATRIX iter_0023 PARCIALMENTE fechado pelo commit
   UlFor `99af14b7` (validation 14d REFUTOU 2/7 promoves) mas h22_per_fold
   NE+SE ainda nao validados separadamente.
4. **Skill_ens_vs_persist (H24)**: iter_0022 reporta delta_vs_MOD mas nao
   `1 - MAE_ens / MAE_persist` explicitamente. Derivavel do parquet local
   `outputs/iter_0022/`. Nao executado nesta iter (fora do envelope).

## PHASE D (state.json) — DEFERRED

Originalmente planejado adicionar:
- Top-level `leaderboard_oficial_v33_champions` block (machine-readable)
- Top-level `best_ml` block (para quality_gate R3)
- `planner_config.notas_iter0025`

Mas durante esta iter o **daemon loop estava ativamente modificando
state.json em iter_0024** (recon_delta de 2 commits UlFor adicionais). Para
evitar race condition e merge conflict, PHASE D foi DEFERIDA ate proxima
iter (iter_0026 ou followup).

**Acao recomendada**: ao retomar, adicionar os 2 blocos top-level COM os
dados desta iter (champions_metrics_consolidated ja existe em state.json
desde iter_0017 como `hypotheses_verdict.H19_extract_champion_mae_r2_f1.champions_metrics_consolidated`
— pode-se promover para top-level + adicionar RMSE/bias/skill).

Quality gate R3 (state.best_ml.warn check) continua passando porque
`state.best_ml` ainda nao existe — heuristica idempotente.

## Proxima iter recomendada

Daemon esta em modo recon_delta. Como UlFor commitou novos commits
(observed em iter_0024 `951b8c7b`: 4427a718..5d41d063 absorvido + commit
99af14b7 absorved retroactively no iter_0023), o backlog UlFor esta caught
up por ora.

**Candidatos para iter_0026** (planner sees this iter as just-completed):

- **(A)** PHASE D do iter_0025 (state.json block addition) — 0.3h, fecha
  pendencia infra. Recomendado.
- **(B)** H31 emergente: req-0008 ao UlFor pedindo holdout 14d real
  isolado em h22_per_fold NE+SE (independente do que iter_0024 absorveu
  via commit 99af14b7). Custo loop 0.5h, custo UlFor ~30min.
- **(C)** H29 emergente: aplicar bias_correction per-sub (NE@28d, N@60d)
  sobre H10 ensemble LGBM+persist no replay loop. Custo 1h. Ortogonal a
  H10/H24, pode compor ganhos.
- **(D)** H27 (P3 P50 quantile substituto, custo zero) ou H30 (Ridge pdp
  residual CV, custo 1h) — ambos do queue antigo.

Recomendacao: **(A) + (B)** combinados em 1h.

## Lessons learned

- **Suite canonica side-by-side > narrativa cumulativa**. Leaderboard
  cresceu 1098 linhas com snippets historicos — buscar champion atual
  custava ~5min de scrolling. Estrutura nova (62 linhas top-table + 60
  linhas notes) caracteriza estado em <1min de leitura.
- **Daemon loop + sessao manual paralela** = race condition real em
  state.json. Solucao adotada: defer PHASE D, commit so leaderboard +
  iter file + queue annotation. Daemon owns state.json mid-iter.
- **Skill_vs_persist e' metrica indispensavel** ao lado de MAE — operador
  precisa saber "champion bate baseline simples?" antes de adotar.
  Faltava no leaderboard ate hoje (so' implicito via NMAE comparison).
- **F1_p50 NaN em S** e' artefato fundamental do protocolo P50(train)=0
  quando target tem muitos zeros. Nao e' "falha do modelo" — e' "metrica
  binarizada em threshold zero nao discrimina". Documentar como
  esperado, considerar threshold alternativo (P75) em sub com cauda
  longa de zeros.
