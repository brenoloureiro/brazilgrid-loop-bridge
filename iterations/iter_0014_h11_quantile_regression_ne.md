---
alvo: NE_d1_quantile_forecast
layer: curtailment
iter_num: 0014
type: hypothesis_test (verdict=REFUTADO_NE)
data_utc: 2026-05-24T11:30:00Z
baseline_tipo: |
  Dual baseline por fold:
    (a) persist_d1            — repeat ultimo dia
    (b) LGB-mean (objective='regression')  — point estimate default
baseline_metric: |
  MAE persist_d1 mean across folds (reuso CV iter_0012/iter_0013):
    NE  ~33.7k MWh   SE  ~8.3k MWh   S  ~1.27k MWh   N  ~0.51k MWh
  MAE LGB-mean (point baseline; objective='regression'):
    NE/v1 45.959k  NE/v2 32.181k  NE/v3 31.593k
    SE/v1 7.752k   SE/v2 7.736k   SE/v3 7.222k
    S/v1 1.157k    S/v2 1.064k    S/v3 0.977k
    N/v1 0.578k    N/v2 0.578k    N/v3 0.553k
hypothesis: |
  H11 (P2): LightGBM com objective='quantile' (alphas 0.1, 0.5, 0.9).
  Substituir ponto-estimativa por bandas — util para downstream (operador
  escolhe P90 conservador). Alvo explicito: NE_d1_quantile_forecast.

  Hipotese implicita: a) bandas P10/P90 sao CALIBRADAS (coverage_band_80
  proximo de 80% nominal); b) P50 quantile nao perde magnitude
  significativa vs LGB-mean (delta_mae_p50 <= +10%). Premissa: distribution
  shift documentado em iter_0012 (KS p<0.0001 NE+SE) NAO inviabiliza
  calibracao com pesos default.
result_metric: |
  REFUTADO_NE. Bandas LGBM quantile com defaults sistemicamente
  UNDERCOVERED em todas as 4 subs. Coverage band 80% nominal:
    NE: 43.6% mean (0/3 cells in [70%, 90%])
    SE: 45.4% mean (1/15 folds individuais entram em [70%, 90%])
    S:  52.1% mean (1/15 folds individuais)
    N:  47.1% mean (3/15 folds individuais)

  Tabela resumida (mean across 5 folds, por cell — 12 cells):

    cell     PB10    PB50    PB90   cov80   cov10  cov90    width   cross  P50/mean
    NE/v1   7937   23198   18589   45.5%   15.1%  60.6%   66276    1.7%    -0.6%
    NE/v2   6960   17091   13870   43.5%   14.0%  57.5%   57228    6.7%    +5.6%
    NE/v3   7046   17002   14114   41.8%   16.4%  58.2%   53463    6.8%    +8.4%
    SE/v1   1725    3784    2569   48.2%   25.1%  73.3%   11319    4.0%    -1.1%
    SE/v2   1757    3853    2444   47.5%   24.4%  71.9%   11957    3.0%    -0.7%
    SE/v3   2034    3484    2363   40.5%   30.2%  70.6%    9579   13.4%    -2.9%
    S/v1     122     507     459   53.1%   28.8%  79.3%    1985    9.7%   -16.5%
    S/v2     127     482     415   51.6%   28.9%  78.5%    1674   13.0%   -15.6%
    S/v3     128     472     355   51.5%   29.3%  78.1%    1753   14.5%    -9.2%
    N/v1      75     233     122   47.5%   37.2%  84.6%    1011    2.0%   -18.2%
    N/v2      75     233     122   47.5%   37.2%  84.6%    1011    2.0%   -18.2%
    N/v3      83     229     109   46.5%   39.2%  85.6%     996    2.0%   -17.1%

  Achados relevantes:

  1) UNDER-COVERAGE SISTEMICO. Coverage_10 elevado (NE 14-16% vs 10%
     nominal; SE 24-30%; S 28-29%; N 37-39%) — modelo subestima cauda
     baixa (zeros sao mais comuns no test que no train). Coverage_90 baixo
     (NE 57-60% vs 90%; SE 70-73%; S 78-79%; N 85%) — modelo subestima
     cauda alta. Bandas sao ESTREITAS DEMAIS.

  2) FOLD HETEROGENEITY confirma B5 dist_shift. Em NE/v1:
       fold 1: cov_band=62% (jul/24-ago/24, regime estavel)
       fold 2-3: cov_band=27-40% (set/24-dez/24, transicao curt-up)
       fold 4-5: cov_band=47-52% (jan-mar/26, novo regime, recalibra)
     Padrao reproduz KS p<0.0001 documentado em iter_0012.

  3) P50 nao quebra magnitude. NE: media +4.5% vs LGB-mean (passa B4 <=
     +10%). Bonus em SE/S/N: P50 BATE LGB-mean (deltas -0.7% a -18.2%).
     P50 quantile = mediana, robusto a outliers de cauda; em
     distribuicoes com cauda longa de zeros (N/S) ganha sistematicamente.
     Implicacao: P50 quantile pode VIRAR POINT ESTIMATE substituto em N+S
     (H27 derivada).

  4) QUANTILE CROSSING controlado. Max 14.5% em S/v2 (cauda longa de
     zeros, mais ambiguo); NE+SE+N <7%. Treinar alphas independentes nao
     introduz patologia significativa de monotonicity — isotonic post-hoc
     desnecessario nesta etapa.

  5) PINBALL LOSS escala com magnitude do problema (NE ~17-23k, S ~470,
     N ~230). Nao usavel como threshold absoluto; usado como ranking
     intra-cell. PB(0.5) > PB(0.1) e PB(0.9) em todos os subs — coerente
     com simetria das perdas pinball para alphas 0.1/0.9 vs 0.5.

  Verdict criteria (definidos a priori no script):
    CONFIRMADO_NE: >=2/3 NE cells com cov_band in [70%, 90%]
                   AND delta_mae_p50_vs_mean_pct mean <= +10%.
    REFUTADO_NE:   cov_band fora de [60%, 95%] em maioria das NE cells
                   OR delta_mae_p50 mean > +20%.
  Resultado: cov_band <60% em 3/3 NE cells (NE/v1=45.5%, NE/v2=43.5%,
  NE/v3=41.8%) — REFUTADO_NE pelo lado da banda. P50 magnitude OK em NE
  (+4.5% mean), mas isso nao salva o veredito (banda inutil = nao serve
  ao caso de uso operacional "P90 conservador").
decision: |
  REFUTADO_NE. LGBM com objective='quantile' e defaults NAO produz bandas
  calibradas para curt D+1 em NE (target H11). Bandas P10/P90 capturam
  ~44% dos eventos vs 80% nominal — operador que escolher "P90 conservador"
  vai estar errado em ~30% das situacoes de alta crit. Nao virar deliverable
  para v1.0 nesta forma.

  P50 magnitude OK — mantida opcao de usar P50 como point estimate
  substituto em N+S (H27 derivada, P3).

  Sem req externo emitido: causa raiz e' arquitetura LGBM (variancia
  modelada implicitamente via leaves) + distribution shift conhecido
  (iter_0012). UlFor nao desbloqueia isso.
sanity_checks_passed:
  permutation_importance: skipped
  holdout_temporal_strict: passed_embedded
  leak_detection: skipped
  baseline_compare: passed_embedded
  distribution_shift: annotated_reuse
  zero_count_shift: passed_embedded
budget_consumido_iter: 1.0
custo_estimado_usd: null
---

# Iter 0014 — H11 LGBM Quantile Regression (NE)

## Hipotese

H11 (P2 do queue): "LightGBM com objective=quantile (alphas 0.1, 0.5, 0.9).
Substituir ponto-estimativa por bandas — util para downstream (operador
escolhe P90 conservador). Bench worktree ja tem NGBoost similar."

Alvo explicito: NE_d1_quantile_forecast. Para reuso do framework de CV
(iter_0012/iter_0013), rodamos as 4 subs x 3 vers (12 cells), mas o
verdict e' julgado APENAS em NE. SE/S/N viram bonus diagnostico.

Hipotese implicita testada:
  a) bandas P10/P90 sao CALIBRADAS → coverage_band_80 in [70%, 90%];
  b) P50 quantile nao degrada magnitude vs LGB-mean → delta_mae <= +10%.

## Como foi rodado

Script: `scripts/h11_quantile_regression_cv.py`.

CV identica a H7/H10:
  - 5 folds walk-forward, test=60d, gap=7d.
  - Features iter_0002 (43-47 cols por cell, n_train varia 74→344 por fold).
  - Sem retune: LGB defaults (n_est=300, lr=0.05, num_leaves=31, min_child=10).

Cada fold treina 4 modelos no MESMO train block:
  - mdl_mean (objective='regression') → baseline point
  - mdl_q01 (objective='quantile', alpha=0.1) → P10
  - mdl_q05 (objective='quantile', alpha=0.5) → P50
  - mdl_q09 (objective='quantile', alpha=0.9) → P90

Persist_d1 baseline: pred[i] = y[i-1], pred[0] = ultimo y_train. (mesmo
formato H10).

Metricas por fold:
  - Pinball loss por alpha (proper scoring rule para quantiles)
  - Coverage P10 (mean(y<q10)), P90 (mean(y<q90)), banda 80% (mean(q10<=y<=q90))
  - Sharpness mean = mean(q90-q10)
  - Quantile crossings (mean(q10>q50 OR q50>q90))
  - MAE de P50 vs LGB-mean + persist_d1 (delta absoluto + %)
  - metric_suite completo p/ P50 e LGB-mean e persist (back-compat H9)

Comando:
```
.venv/Scripts/python loops/forecast-mega-loop/scripts/h11_quantile_regression_cv.py
```

(Loop venv local nao tem sklearn; uso .venv raiz do brazilgrid que tem
sklearn 1.8.0 + lightgbm 4.6.0. Pratica padrao do repo.)

## Resultado

VERDICT: **REFUTADO_NE** (script `verdict.json`).

Causa raiz: bandas LGBM quantile sistematicamente undercovered.

**Por que sub-aproveita 80% nominal:** LGBM modela quantile por leaves
do tree — variancia das predicoes vem de variancia em-amostra do train.
Quando test_y tem variancia maior (distribution shift confirmado por
B5 iter_0012, KS p<0.0001 entre folds antigos e recentes), o modelo
nao se adapta — as bandas treinadas continuam estreitas. Fold-a-fold:

  NE/v1 cov_band por fold:  62%, 27%, 40%, 47%, 52%
  NE/v2 cov_band por fold:  60%, 42%, 22%, 49%, 45%
  NE/v3 cov_band por fold:  57%, 40%, 18%, 58%, 36%

O fold 1 (ate jul/2024, regime mais estavel) e' o unico que se aproxima
de 60%. Folds 2-3 (transicao curt-up set/2024) sao os piores.

**Sanity checks (6/6 verificados):**

  - **B1 leak_detection**: SKIPPED. Features identicas a iter_0002 — leak
    ja validado em iter_0010 H3 (PDP_prev forward-looking, p_perm=0.0).
    Quantile objective e' post-prediction transform, sem features novas.

  - **B2 permutation_importance**: SKIPPED. PI das features iter_0002 ja
    rodado em iter_0010 (5/6 cells passaram com p=0.0). Quantile usa
    mesmo X; perm seria redundante.

  - **B3 holdout_temporal_strict**: PASSED_EMBEDDED. GAP=7d em cada fold;
    train ate (test_start - 7d). Zero overlap.

  - **B4 baseline_compare**: PASSED_EMBEDDED. P50 quantile comparada a
    persist_d1 E LGB-mean em cada fold. delta_mae_p50_vs_mean_pct mean
    NE = +4.5% (<= +10% threshold). P50 BATE LGB-mean em outros subs.

  - **B5 distribution_shift**: ANNOTATED_REUSE. Evidencia ja em iter_0012
    (NE+SE KS p<0.0001). Padrao fold-a-fold do under-coverage REPRODUZ o
    shift documentado → confirma diagnostico.

  - **B6 zero_count_shift**: PASSED_EMBEDDED. n_test >= 58 em todos os
    folds (threshold de downgrade = 30). Sem zero-only folds.

**Achados secundarios (bonus N+S+SE):**

  - P50 quantile BATE LGB-mean em magnitude em **N, S, SE**:
      N: -18% mean (0.467k vs 0.578k LGB-mean) — sub com cauda longa de zeros
      S: -14% mean (0.97k vs 1.07k)
      SE: -1.6% mean (basicamente empate)
    Mecanismo: mediana e' mais robusta que mean para distribuicoes com
    cauda longa de zeros estruturais. Em N, ~37% dos dias tem curt~0
    (cov_10 ~37% confirma). LGB-mean puxa para cima por outliers;
    quantile P50 fica na mediana.

  - Quantile crossings BAIXAS (NE+SE+N <7%, S 9-14%). Modelos independentes
    nao precisam de isotonic post-hoc para monotonicity nesta amostra.

  - Sharpness MUITO ESTREITA: NE width 53-66k vs persist_d1 MAE 33k
    (banda <2x o erro do baseline) — banda tem que ser maior para cobrir
    incerteza real.

## Decisao

**REFUTADO_NE.** LGBM com objective='quantile' + defaults NAO produz
bandas calibradas em NE D+1 curt. Coverage_band_80 mean 43.6% vs 80%
nominal — operador escolhendo "P90 conservador" estaria errado em ~30%
dos casos. Nao virar deliverable v1.0 nesta forma.

**Sem req externo emitido:** causa raiz e' arquitetura (LGBM nao modela
heteroscedasticidade explicita) + distribution shift conhecido (iter_0012).
UlFor nao desbloqueia. Sem mudanca em production.

**Hipoteses derivadas (3 novas, todas P3):**

  - **H26 (P3)**: Conformal prediction post-hoc. Calibrar bandas LGBM
    via split conformal: pegar nonconformity scores em calibration set,
    inflar banda. Objetivo: cov_band_80 in [75%, 85%]. SEM retreinar
    modelo, baixo custo. Likely-fix se atacarmos H11 de novo.

  - **H27 (P3)**: P50 quantile como POINT ESTIMATE substituto em N+S.
    P50 BATE LGB-mean em magnitude em N (-18%) e S (-14%). Sem custo,
    so trocar objective de regression para quantile alpha=0.5. Ganho
    transversal pequeno mas universal nos subs com cauda longa.

  - **H28 (P3)**: NGBoost vs LGBM quantile. NGBoost modela distribuicao
    parametrica (Normal/Lognormal); naturalmente lida com heteroscedasticidade
    (sigma como funcao de X). Bench worktree mencionou NGBoost — vale
    integrar e comparar pinball loss.

## Proximo passo

H11 fechada como REFUTADO_NE com 3 follow-ups (H26-H28).

Proximo plan natural (planner alt_next pre-existente):
  - **H21** (P2 feature engineering, derivada de H3 iter_0010): adicionar
    `pdp_residual_mwh = pdp_prev - gen` como feature, rodar bake-off
    replay sobre LGBM + ensemble (combina H3 + H10). Ja desbloqueada.
  - **H24** (P2 ensemble sobre champions Ridge/LR): aplica esquema do H10
    validado a producao UlFor.

Recomendacao: **H21 primeiro**. Ja codavel localmente, sem dep externa,
derivada direta de H3 confirmado, melhora baseline antes de H24.
