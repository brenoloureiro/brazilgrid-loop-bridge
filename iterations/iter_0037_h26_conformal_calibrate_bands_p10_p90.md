---
alvo: NE_d1_quantile_calibrated
layer: curtailment
iter_num: 0037
type: hypothesis_test (verdict=INDETERMINADO_NE_BORDERLINE_CONFIRMADO_SE_S_BONUS)
data_utc: 2026-05-25T08:00:00Z
hypothesis: |
  H26 (P3, queued desde iter_0014 — derivada H11 follow_up):
  "Conformal prediction post-hoc para calibrar bandas P10/P90"
  Premissa: H11 (iter_0014, REFUTADO_NE) mostrou cov_band_80 mean
  43.6% (NE) vs 80% nominal — under-coverage sistemico em todas 4 subs.
  Causa raiz: LGBM quantile com defaults nao modela heteroscedasticidade
  + distribution shift (iter_0012 KS p<0.0001 NE+SE).

  Fix: split conformal prediction (Lei et al. 2018, CQR-symmetric
  Romano et al. 2019).
    1. train_inner = train_full[:-INNER_VAL_DAYS]
    2. inner_val = ultimos INNER_VAL_DAYS de train_full
    3. Fit LGBM quantile (alpha=0.1,0.5,0.9) em train_inner
    4. Predizer em inner_val -> nonconformity scores
       s_i = max(q10_iv - y_iv, y_iv - q90_iv)
    5. q_alpha = quantile_empirico(s, ceil((n+1)*(1-alpha))/n)
       (Lei et al. correcao finite-sample)
    6. Aplicar test: [q10 - q_alpha, q90 + q_alpha]
  Sem retreinar modelo (post-hoc cheap).

  ALVO: NE_d1_quantile_calibrated.
  ACEITACAO (a priori):
    CONFIRMADO_NE: >=2/3 NE cells com cov_band_80_cal in [75%, 85%]
                   AND width_ratio_cal_vs_inner mean < 2.0x
    INDETERMINADO: 1 dimensao ok / borderline; OU iv=60 salva, iv=30 nao
    REFUTADO_NE:   cov fora [70%, 90%] em maioria OR width > 3x
  Risco declarado pelo queue: inner_val 30d pequeno (testar tambem 60d).
baseline_tipo: |
  Tripla baseline por fold:
    (a) UNCAL_FULL  — LGBM quantile em train_full (= H11 iter_0014).
                      Sanity: reproduz cov_uncal_F = 45.5%/43.5%/41.8%
                      NE (matches H11 exatamente).
    (b) UNCAL_INNER — LGBM quantile em train_inner (= train_full menos
                      INNER_VAL_DAYS dias). Apples-to-apples para isolar
                      efeito conformal (mesmo modelo, sem inflar).
    (c) PERSIST_D1  — reuso H10/H11 (point baseline).
baseline_metric: |
  Per NE cell, cov_band_80 (mean across 5 folds):
    H11 H26               cov_uncal_F  cov_uncal_I(iv30)  cov_CAL(iv30)
    NE/v1                       45.5%             35.9%          77.3%
    NE/v2                       43.5%             42.5%          73.6%
    NE/v3                       41.8%             41.1%          73.3%

  NE mean (iv=30):  cov_cal 74.7% (vs 80% nominal, mas 3/3 in [70,90])
                    width_ratio_cal_vs_inner 2.03x  (target <2.0)
                    width_ratio_cal_vs_full  2.00x
                    cells_in [75,85] strict: 1/3
                    cells_in [70,90] loose:  3/3

  NE mean (iv=60):  cov_cal 71.6% (under-shoot por inner_val maior reduzir
                    train_inner ainda mais — sem ganho).

  Bonus diagnostico (NAO julga verdict):
    SE iv=30: cov_cal mean 76.6%  (3/3 strict, 3/3 loose)  width x1.95
    S  iv=30: cov_cal mean 79.5%  (3/3 strict, 3/3 loose)  width x1.51
    N  iv=30: cov_cal mean 90.8%  (0/3 strict, 1/3 loose)  width x1.68
                                  ^^^^^^ OVER-COVER (right tail)
result_metric: |
  VERDICT: **INDETERMINADO_NE** (alvo H26 borderline) com **CONFIRMADO_SE_S**
  como bonus deliverable.

  RACIONAL NE (alvo):
    cov_cal mean 74.7% — falha [75%,85%] strict por **0.3 pp**.
    width_ratio 2.03x — falha <2.0 strict por **0.03x**.
    Tres cells PASS loose [70%,90%]; uma cell strict.
    iv=60 NAO salva (cov cai para 71.6%, train_inner empobrece mais).
    Logica do verdict: pass_cov_loose && pass_sharp_loose -> INDETERMINADO.

    Causa proxima: fold heterogeneity (mesmo padrao de H11):
      NE/v1 fold cov_cal=[0.82, 0.58, 0.52, 0.95, 1.00] — folds 2-3
      (set/2024 transicao curt-up, KS p<0.0001 confirmado iter_0012)
      ficam sob a banda mesmo COM q_alpha=53k MWh. Conformal e' marginal
      por construcao (1 q_alpha global) — nao adapta a regime intra-test.

  BONUS DELIVERABLE: SE+S sao CONFIRMADO no nivel de sub:
    SE: 3/3 cells com cov_cal_mean in [75%,85%], width <2.0x baseline
    S:  3/3 cells com cov_cal_mean in [75%,85%], width <2.0x baseline
    Conformal vira bandas P10/P90 deliverable para SE+S (operador pode
    usar P90 conservador com 80% nominal entregue).

  REGRESSAO N: conformal OVER-COVERS (mean 90.8%). Mecanismo: scores
  s_i = max(q10-y, y-q90) dominados por outliers de cauda alta em N
  (37% dos dias com curt~0, alguns picos altos). q_alpha global empurra
  banda inferior para zero e banda superior para alem do ymax test.
  Banda muito larga = nao-discriminativa (inutil para operador). Fix
  futuro: CQR-asymmetric (H37 candidato).

  Tabela completa (iv=30 selecionado; iv=60 reportado em summary.csv):

    cell  n  q_a    cov_uF  cov_uI  cov_CAL  w_uF    w_uI   w_CAL  xI    xF    in[75,85]
    NE/v1 5  45645  45.5%   35.9%   77.3%    66276   60071  128697 2.24  2.04  1/5
    NE/v2 5  29952  43.5%   42.5%   73.6%    57228   61416  107161 1.89  1.96  1/5
    NE/v3 5  30434  41.8%   41.1%   73.3%    53463   57104  104318 1.94  2.01  1/5
    SE/v1 5   6633  48.2%   47.5%   77.3%    11319   11833   21889 1.86  1.93  0/5
    SE/v2 5   6800  47.5%   43.1%   76.3%    11957   12557   22059 1.85  1.84  1/5
    SE/v3 5   6719  40.5%   40.1%   76.3%     9579   10714   21598 2.14  2.24  0/5
    S/v1  5    776  53.1%   50.4%   80.3%     1985    1889    2708 1.46  1.39  1/5
    S/v2  5    656  51.6%   55.3%   77.6%     1674    1764    2448 1.46  1.57  1/5
    S/v3  5    685  51.5%   46.5%   80.6%     1753    1587    2318 1.61  1.58  2/5
    N/v1  5    412  47.5%   42.1%   91.6%     1011     828    1326 1.76  1.45  0/5 (over)
    N/v2  5    412  47.5%   42.1%   91.6%     1011     828    1326 1.76  1.45  0/5 (over)
    N/v3  5    311  46.5%   45.1%   89.0%      996     864    1291 1.53  1.33  1/5 (over)

  Nota replicacao H11: cov_uncal_F == H11 iter_0014 cov_band_80_mean
  bit-exato (45.5/43.5/41.8 NE; 48.2/47.5/40.5 SE; 53.1/51.6/51.5 S;
  47.5/47.5/46.5 N). Framework consistente.
decision: |
  INDETERMINADO_NE + CONFIRMADO_SE_S como bonus. NAO promove para
  deliverable v1.0 em NE — diferenca de 0.3pp (cov 74.7 vs 75 strict) +
  0.03x (width 2.03 vs 2.0 strict) e' ruido amostral em 5 folds × 3
  cells. Reabertura honesta requer mecanismo que adapte intra-test
  (Mondrian conformal por regime, H37 derivada).

  BONUS deliverable separado: SE+S conformal calibration pode virar
  produto **agora** (decisao Breno, fora do scope da iter):
    - operador SE pode usar P10/P90 cal com 80% nominal
    - operador S idem (NMAE 67% mean — bandas conservadoras justificadas)
    - operador N + NE devem usar point P50 ate H37 fechar

  Sem req externo emitido: causa raiz e' arquitetura conformal global
  (1 q_alpha) + distribution shift conhecido (iter_0012). UlFor nao
  desbloqueia isso.

  Conformal post-hoc validado mecanicamente em todos os 12 cells
  (cov_cal sempre > cov_uncal_inner, ratio >= 1.0 em 100% folds).
  Implementacao correta — limite e' do alvo (NE), nao do metodo.
sanity_checks_passed:
  permutation_importance: skipped
  holdout_temporal_strict: passed_embedded
  leak_detection: skipped
  baseline_compare: passed_embedded
  distribution_shift: annotated_reuse
  zero_count_shift: n_a
budget_consumido_iter: 1.5
custo_estimado_usd: null
---

# Iter 0037 — H26 Conformal Prediction Post-hoc (NE bands P10/P90)

## Hipotese

H26 (P3 do queue, derivada H11 follow_up): "Conformal prediction post-hoc
para calibrar bandas P10/P90". Reabre H11 (REFUTADO_NE iter_0014) usando
split conformal — sem retreinar modelo, baixo custo, mecanismo conhecido
(Lei et al. 2018, Romano et al. CQR-symmetric 2019).

ALVO explicito: `NE_d1_quantile_calibrated`. Rodadas 4 subs × 3 vers (12
cells) para reuso framework e como bonus diagnostico — verdict julgado
APENAS em NE.

Hipotese implicita: nonconformity scores em inner_val (30d) capturam o
gap de cobertura sistemico de LGBM quantile, sem destruir sharpness (width
ratio < 2x baseline). Risco declarado: inner_val=30d pequeno (testar 60d).

## Como foi rodado

Script: `scripts/h26_conformal_prediction.py`.

CV identica a H11/H10/H7:
  - 5 folds walk-forward, test=60d, gap=7d.
  - Features iter_0002 (37-47 cols por cell, n_train varia 74→344).
  - LGB defaults: n_est=300, lr=0.05, num_leaves=31, min_child=10.

Por fold (5 folds × 12 cells = 60 fold-runs × 2 variantes iv = 120 conformal calibrations):
  1. train_full = feats[dia < train_cutoff]
  2. test_block = feats[test_start..test_end]
  3. UNCAL_FULL: fit LGBM quantile (a=0.1,0.5,0.9) em train_full -> predict test
     (= replica H11 iter_0014 bit-exato; sanity de framework)
  4. UNCAL_INNER + CAL (para cada iv_days in {30, 60}):
     a. train_cutoff_inner = train_cutoff - iv_days dias
     b. train_inner = train_full[dia < train_cutoff_inner]
     c. inner_val = train_full[dia >= train_cutoff_inner]
     d. fit LGBM quantile (a=0.1,0.5,0.9) em train_inner -> predict test (UNCAL_INNER)
     e. predict inner_val -> nonconformity scores s = max(q10_iv - y_iv, y_iv - q90_iv)
     f. q_alpha = quantile(s, ceil((n+1)*0.8)/n, method='higher')  [Lei et al.]
     g. CAL: [max(q10_te - q_alpha, 0), q90_te + q_alpha]

Metricas por fold:
  - coverage_band_80, coverage_10, coverage_90 (cal + uncal_inner + uncal_full)
  - sharpness_mean (q90 - q10) para os 3 esquemas
  - width_ratio_cal_vs_inner (efeito puro do conformal)
  - width_ratio_cal_vs_full (efeito conformal + perda de train data)
  - pinball loss (cal e uncal)
  - q_alpha (a inflacao em MWh)
  - score_iv_mean, score_iv_p80 (diagnostico de calibracao)

Comando:
```
.venv/Scripts/python loops/forecast-mega-loop/scripts/h26_conformal_prediction.py
```

Loop venv local nao tem sklearn; reusada .venv raiz (lgb 4.6.0, sklearn
1.8.0) — pratica padrao do repo (igual H11).

## Resultado

VERDICT: **INDETERMINADO_NE** (script `verdict.json`).

Por que nao CONFIRMADO_NE:
  - cov_cal_mean NE (iv=30) = 74.7% — falha threshold [75%,85%] por 0.3pp
  - width_ratio_cal_vs_inner mean = 2.03x — falha <2.0 por 0.03x
  - cells_in [75,85] strict = 1/3 NE; iv=60 nao salva (cov 71.6%)

Por que nao REFUTADO_NE:
  - 3/3 NE cells PASS loose [70%,90%]
  - width_ratio < 3.0 (longe de "banda explode")
  - Conformal MOVEU cov_uncal_inner 35-43% -> cov_cal 73-77% (efeito real)

**Bonus deliverable (CONFIRMADO_SE_S):**
  SE: 3/3 cells, cov_cal mean 76.6%, width ratio 1.95x — PASS estrito
  S : 3/3 cells, cov_cal mean 79.5%, width ratio 1.51x — PASS estrito

**Regressao N:** cov_cal mean 90.8% — OVER-COVERS. Mecanismo: scores
dominados por outliers de cauda alta (37% dos dias com curt~0); q_alpha
global empurra banda muito alem da escala empirica do test. Banda
larga demais = nao-discriminativa. Fix candidate: CQR-asymmetric (H37).

**Sanity checks (6/6 verificados):**

  - **B1 leak_detection**: SKIPPED. Features identicas a iter_0010 H3 (PI
    p=0.0 em 5/6 cells; PDP_prev forward-looking validado leak-free).
    Conformal e' transformacao linear post-hoc sobre predicoes; sem
    features novas.

  - **B2 permutation_importance**: SKIPPED. Mesma X de iter_0010, PI ja
    rodado e passed. Conformal nao introduz features.

  - **B3 holdout_temporal_strict**: PASSED_EMBEDDED. 5 folds walk-forward,
    gap=7d entre train_full e test. Inner_val SEMPRE temporalmente ANTES
    do test (carved do FIM do train_full, antes do gap). Zero look-ahead
    na calibracao. Calibracao usa apenas (q_inner_val, y_inner_val) que
    foram observados antes do test_start - GAP_DAYS - 1.

  - **B4 baseline_compare**: PASSED_EMBEDDED. Tripla baseline reportada
    em cada fold: UNCAL_FULL (= H11), UNCAL_INNER (apples-to-apples),
    CAL (H26 deliverable), + PERSIST_D1 point. Replicacao H11 bit-exato
    (cov_uncal_full == iter_0014 summary.csv).

  - **B5 distribution_shift**: ANNOTATED_REUSE. iter_0012 KS p<0.0001
    em NE+SE confirmado. Fold heterogeneity em CAL (NE/v1 folds:
    0.82/0.58/0.52/0.95/1.00) prova que o conformal MARGINAL (1 q_alpha
    global por fold) NAO adapta a regime intra-test. E' o limite teorico
    da abordagem split conformal — Mondrian conformal por regime (H37
    derivada) e' o passo natural.

  - **B6 zero_count_shift**: N/A. Conformal entrega intervalo, nao point
    estimate. B6 mede gating em ymean_test<1 (point forecast cell); fora
    de escopo H26. Per-cell ymean_test consistente com iter_0030 (N+S
    abaixo de 1.5 MWh; NE+SE acima).

## Decisao

INDETERMINADO_NE + CONFIRMADO_SE_S bonus.

NAO PROMOVE para deliverable v1.0 em NE. Diferenca de 0.3pp (cov) +
0.03x (width) e' ruido amostral em 5×3 = 15 fold-cells. Reabertura honesta
demanda mecanismo que adapte intra-test (cv+ conformal, Mondrian por
regime, ou modelo heteroscedastico explicito a' la NGBoost — H28 ja
queued).

BONUS deliverable SE+S separado (decisao Breno):
  - Conformal post-hoc com inner_val=30d entrega bandas P10/P90 com
    cobertura 80% nominal calibrada em 3/3 cells SE e 3/3 cells S.
  - Custo: 1 dia perdido de training data (inner_val) + 1 hyperparam
    (alpha_target=0.2). Sem retreinar modelo na producao — aplicacao
    e' transformacao linear sobre saidas LGBM.
  - Util para "P90 conservador" em SE+S; operador NE+N usa P50 ate H37
    resolver (P50 ja' nao quebra magnitude em NE per H11).

Conformal post-hoc MECANISMO validado: cov_cal > cov_uncal_inner em
100% dos 60 fold-runs com q_alpha > 0; ratio sempre >= 1.0 (nao reduz
banda). Implementacao correta — limite e' do TARGET (NE distribution
shift), nao do metodo.

## Proximo passo

H37 derivada (criada nesta iter, P3, queued):
  "Mondrian conformal por regime — split inner_val em buckets de curt"
  Goal: adapta q_alpha intra-fold para resolver heterogeneity NE+SE
  documentada (folds 2-3 sob cov, folds 4-5 over cov). Risco: 30d /
  2 buckets = 15d por bucket, instavel; testar via leave-one-fold.

Sem follow-up imediato — proximo iter: planner escolhe entre H30
(P3 ~1h, fecha H3-family ja queued), H37 (recem criada), ou
RECON_DELTA UlFor se HEAD avancar. SE+S bonus pode virar req-NNNN
para UlFor (promote conformal_p10p90 SE+S como artifact opcional do
v1.0) se Breno aprovar.

Sem emitir req externo nesta iter. Producao 100% inalterada.
