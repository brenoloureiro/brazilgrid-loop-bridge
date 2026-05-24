---
alvo: NE_d1_ngboost_quantile
layer: curtailment
iter_num: 0039
type: hypothesis_test (verdict=INDETERMINADO_PINBALL_DEGRADA)
data_utc: 2026-05-25T18:30:00Z
hypothesis: |
  H28 (P3, queued desde iter_0014 — derivada H11 alt-modelo):
  "NGBoost (Duan et al. 2020) modela distribuicao parametrica (Normal/Lognormal)
   — sigma(x) e' funcao explicita de X, lidando com heteroscedasticidade que
   LGBM quantile nao captura. Resolve under-coverage cronica em NE?"

  Premissa: H11 (iter_0014, REFUTADO_NE) detectou cov_band_80 mean 43.5% em
  NE (3 cells) vs 80% nominal — under-coverage sistemica. LGBM quantile treina
  3 modelos independentes por alpha; sigma nao acopla q10/q90 a estrutura
  de variancia condicional. NGBoost natural gradient otimiza joint likelihood
  de (loc, scale) — sigma e' funcao explicita de X.

  ACEITACAO (a priori do queue H28):
    CONFIRMADO: NGBoost (Normal OR LogNormal) com defaults atinge
                cov_band_80 in [70%, 90%] em pelo menos 2/3 NE cells
                AND pinball_loss(P50) <= LGBM quantile P50 (mesmas cells)
    INDETERMINADO: 1 dimensao OK / borderline
    REFUTADO: cov fora [60%, 95%] em maioria NE OR pinball P50 degrada >20%
baseline_tipo: |
  LGBM quantile (objective='quantile', alpha in {0.1, 0.5, 0.9}, 3 modelos
  independentes). Hyperparams identicos a iter_0012 H7 / iter_0013 H10 /
  iter_0014 H11: n_estimators=300, lr=0.05, num_leaves=31, min_child_samples=10,
  subsample=0.8, colsample_bytree=0.9, random_state=0. Predicoes clipped at 0.

  Baseline carregado bit-exato de
  outputs/iter_0014/h11_quantile_regression_ne/results.json por fold.

  Tambem reportado por fold:
    - persist_d1 (sanity floor B4 implicito)
    - metric_suite_p50 (mae/r2/f1/nmae/rmse/bias) sobre NGB P50
baseline_metric: |
  Per cell NE, mean across 5 folds (LGBM quantile H11 iter_0014):
    NE/v1: pinball_0.5=23198 cov_band_80=45.5% width=66276 MAE_P50=46396
    NE/v2: pinball_0.5=17091 cov_band_80=43.5% width=57228 MAE_P50=34183
    NE/v3: pinball_0.5=17002 cov_band_80=41.8% width=53463 MAE_P50=34003

  Per cell NE, mean across 5 folds (NGBoost Normal, fit em y_raw):
    NE/v1: pinball_0.5=24962 cov_band_80=67.3% width=114942 MAE_P50=49924
    NE/v2: pinball_0.5=19948 cov_band_80=76.9% width=109258 MAE_P50=39896
    NE/v3: pinball_0.5=19820 cov_band_80=76.3% width=109068 MAE_P50=39640

  Per cell NE, mean across 5 folds (NGBoost LogNormal-equiv, fit em log1p(y_raw)):
    NE/v1: pinball_0.5=32169 cov_band_80=89.9% width=245029 MAE_P50=64337
    NE/v2: pinball_0.5=29523 cov_band_80=87.1% width=254437 MAE_P50=59046
    NE/v3: pinball_0.5=28986 cov_band_80=87.8% width=256507 MAE_P50=57971
result_metric: |
  VERDICT: **INDETERMINADO_PINBALL_DEGRADA**.

  D1 cov calibration: **PASS** (em ambas dists).
    Normal: 2/3 NE cells em [70%, 90%] (NE/v2=76.9%, NE/v3=76.3%; NE/v1=67.3% off-lo).
            Sub-mean cov=73.5%.
    LogNormal: 3/3 NE cells em [70%, 90%] (87.1-89.9%, todas borderline alto).
            Sub-mean cov=88.3%.
    NGBoost LIFTS cov_band_80 em +21-46 pp vs LGBM (massive).

  D2 pinball preservation: **FAIL** (em ambas dists).
    Normal: delta_pinball_05 mean = +14.4% (NE/v1 +6.5%, NE/v2 +19.6%, NE/v3 +17.1%).
    LogNormal: delta_pinball_05 mean = +55.5% (NE/v1 +31.7%, NE/v2 +70.3%, NE/v3 +64.6%).
    Threshold a priori (D2): pinball P50 <= LGBM. Ambas dists ABOVE 0.

  Hard-refute floor (cov fora [60%, 95%] em >50% cells OR pinball >+20% em ambas dists):
    Normal: cov fora [60%, 95%] em 0/3 cells. Pinball +14.4% (< +20%). NAO dispara.
    LogNormal: cov fora [60%, 95%] em 0/3 cells (todas em [70%, 90%]).
               Pinball +55.5% (>>+20%). LogNormal-only dispararia hard-refute
               mas Normal salva (regra "ambas dists" no codigo: ambas > +20%).

  Tabela completa per cell-dist (mean across 5 folds):

    cell-dist               n     PB50  cov80  width   MAE_P50  dPB%  dcov_pp dw%      in_band
    NE/v1/Normal            5    24962  67.3%  114942   49924   +6.5%  +21.8pp  +85.9%  1/5
    NE/v1/LogNormal         5    32169  89.9%  245029   64337  +31.7%  +44.4pp +277.8%  2/5
    NE/v2/Normal            5    19948  76.9%  109258   39896  +19.6%  +33.5pp +106.0%  3/5
    NE/v2/LogNormal         5    29523  87.1%  254437   59046  +70.3%  +43.6pp +327.3%  3/5
    NE/v3/Normal            5    19820  76.3%  109068   39640  +17.1%  +34.5pp +118.8%  3/5
    NE/v3/LogNormal         5    28986  87.8%  256507   57971  +64.6%  +46.0pp +348.3%  3/5

  Bonus diagnostico:
    - Quantile monotonicity: 0% crossings em TODAS as 30 folds NGB
      (vs LGBM quantile ~0% tambem; ambos preservam ordem). Parametrico
      garante q10<=q50<=q90 por construcao — vantagem teorica neutralizada
      por LGBM ja' ser bem-comportado nas features iter_0002.

    - Width inflation: NGBoost Normal +85-119% width; LogNormal +278-348%.
      LogNormal cobre quase tudo (88.3% sub-mean) mas a custo de
      widths absurdos — banda inutil para operacao (ex: NE/v1/LogNormal
      width=245k MWh com y_te_mean=92k MWh, ratio 2.7x).

    - Per-fold variance NGBoost Normal: cov_band_80 oscila por fold
      (NE/v1: 63.3, 41.7, 48.3, 88.1, 95.0 — std=24pp). Calibracao
      media OK mas instavel por fold. Mesmo problema que LGBM tem.

    - Velocidade: n_estimators=100 lr=0.01 NGB roda ~4s/fold (60s total
      18 fits efetivos). Bem dentro de budget.
decision: NAO_PROMOVE_E_FECHA_CAMINHO_PARAMETRICO_DEFAULT
sanity_checks_passed:
  permutation_importance: skipped_inherited
  holdout_temporal_strict: passed_embedded
  leak_detection: skipped_inherited
  baseline_compare: passed_embedded
  distribution_shift: annotated_reuse
  zero_count_n_test_gating: passed
budget_consumido_iter: 0.5
custo_estimado_usd: 0.00
---

# Iter 39 — H28 NGBoost vs LGBM quantile em NE

## Hipotese

H28 (P3, derivada de H11 iter_0014 alt-modelo). NGBoost (Duan et al. 2020) e'
gradient boosting com gradiente natural sobre likelihood parametrica — modela
loc(x) E scale(x) explicitamente. A premissa em H28: a under-coverage cronica
de NE (43.5% vs 80% nominal em H11) e' efeito de LGBM treinar 3 modelos
quantile independentes, sem acoplar scale(x) ao input. NGBoost natural
gradient acopla — deveria calibrar.

Aceitacao a priori (do queue):
  D1 cov: >=2/3 NE cells com cov_band_80 in [70%, 90%] em pelo menos UMA dist.
  D2 pinball: pinball P50 NGB <= LGBM (sub-mean).
  CONFIRMADO: D1 AND D2.

## Como foi rodado

Mesmo CV walk-forward do H11: 5 folds, test window 60d, gap 7d, MIN_TRAIN=60,
MIN_TEST=5. Mesmas features iter_0002 (NE/v1, NE/v2, NE/v3). Bit-paired vs
LGBM quantile per fold (H11 iter_0014 results.json carregado direto).

NGBoost hyperparams (defaults H28 + n_estimators reduzido):
  - n_estimators=100 (vs LGBM 300 — H28 explicito por concern de velocidade)
  - learning_rate=0.01
  - minibatch_frac=1.0
  - natural_gradient=True
  - random_state=0
  - Dist=Normal sobre y_raw (curva 1) | Normal sobre log1p(y_raw) (curva 2,
    LogNormal-equivalente — Dist=LogNormal direto exigiria y_train>0
    estritamente; log1p e' robusto a zeros)

Quantiles extraidos via `dist.ppf(alpha)` por sample (parametric inverse
CDF). P50 point = dist.ppf(0.5). Para LogNormal-equiv, todas as previsoes
sao re-exponenciadas via `expm1(pred)` e clipadas em 0.

Script: `scripts/h28_ngboost_vs_lgbm.py` (~430 LoC).
Comando: `python scripts/h28_ngboost_vs_lgbm.py`
Output: `outputs/iter_0039/h28_ngboost_vs_lgbm/{results,summary,verdict,sanity_summary}.{json,csv}`
Wallclock: ~25s total (3 cells x 2 dists x 5 folds = 30 fits).

## Resultado

**VERDICT: INDETERMINADO_PINBALL_DEGRADA.** D1 pass, D2 fail.

### D1 (cov calibration) — PASS

NGBoost LIFTA dramaticamente cov_band_80 NE em ambas dists:
  LGBM baseline: 43.5% sub-mean (3 cells media)
  NGB Normal:    73.5% sub-mean (+30pp absoluto, 2/3 cells em [70, 90])
  NGB LogNormal: 88.3% sub-mean (+45pp, 3/3 cells em [70, 90], borderline alto)

Por cell (in_band threshold [70%, 90%]):
  NE/v1: Normal=67.3% (NAO), LogNormal=89.9% (SIM, borderline)
  NE/v2: Normal=76.9% (SIM), LogNormal=87.1% (SIM)
  NE/v3: Normal=76.3% (SIM), LogNormal=87.8% (SIM)

Ambas dists individualmente passam D1 (Normal 2/3, LogNormal 3/3). NE/v1
Normal a 67.3% e' a unica cell fora de banda em NE para Normal —
borderline (3pp abaixo do threshold inferior).

Mecanismo confirmado: sigma(x) explicito acopla incerteza ao input.
Distribution shift em NE+SE detectado por KS p<0.0001 (iter_0012 H7) e'
justamente o regime onde isso ajuda — heteroscedasticidade alta.

### D2 (pinball preservation) — FAIL

Pinball P50 (≡ 0.5 * MAE) DEGRADA em ambas dists:

  Normal:    sub-mean delta_pinball_05 = +14.4%
             (NE/v1 +6.5%, NE/v2 +19.6%, NE/v3 +17.1%)
  LogNormal: sub-mean delta_pinball_05 = +55.5%
             (NE/v1 +31.7%, NE/v2 +70.3%, NE/v3 +64.6%)

Threshold a priori: pinball P50 NGB <= LGBM. Ambas dists ABOVE 0,
ambas ABOVE +14.4% mean. Normal fica abaixo do hard-floor (+20%) mas
ainda viola D2; LogNormal explode (+55.5%) mas escapa do hard-floor
"em ambas dists" porque Normal salva.

Mecanismo de degradacao P50:
  - Pinball P50 ≡ 0.5 * MAE — degradacao do MAE significa que a mediana
    estimada esta mais longe da verdade.
  - LGBM quantile com alpha=0.5 minimiza pinball P50 diretamente
    (sao loss-aligned). NGBoost minimiza NLL Normal(mu, sigma) — a
    media mu MINIMIZA MSE da media, nao MAE da mediana. Para distribuicao
    skewed (curt NE tem cauda densa), media != mediana.
  - LogNormal-equiv piora ainda mais: a transformacao log1p amortece
    cauda mas re-exponencia inflama erros pequenos em y_raw. P50 do
    log1p re-exponenciado sub-estima sistematicamente em magnitude alta.

### Width inflation (bonus diagnostico)

NGBoost paga a calibracao com bandas muito mais largas:
  LGBM: width_mean = 53-66k MWh (NE)
  NGB Normal: 109-115k MWh (+86-119%)
  NGB LogNormal: 245-257k MWh (+278-348%)

Para operador: banda LogNormal com width=245k MWh em uma sub onde
y_te_mean=92k MWh tem ratio 2.7x — banda nao informativa
("a verdade esta entre 0 e 2.7x a media"). Coverage 100% e' trivial
nesse regime; o ganho real e' em sharpness conditional, nao bruta.

NGBoost Normal mantem ratio width/mean ~1.2 — bem mais util operacionalmente.
Mas pinball P50 +14.4% e' um preco real.

### Sanity (B1-B6)

Herdado/embutido (mesmo X, mesmo split):
  B1 leak: skipped_inherited (PDP_prev validado iter_0010 H3, 500 shuffles p=0)
  B2 perm: skipped_inherited (mesmo motivo)
  B3 holdout strict: **passed_embedded** (gap=7d em todos 30 folds)
  B4 baseline: **passed_embedded** (comparacao explicita vs LGBM e persist_d1)
  B5 dist_shift: annotated_reuse (KS p<0.0001 NE+SE — explica mecanismo)
  B6 n_test gating: **passed** (n_test 58-60, threshold >=30)

## Decisao

**NAO_PROMOVE_E_FECHA_CAMINHO_PARAMETRICO_DEFAULT.** Recomendacao:

1. **NAO promover NGBoost (Normal nem LogNormal) com defaults** como
   substituto de LGBM quantile em NE. D2 viola — pinball P50 degrada
   14-55% em sub-mean.

2. **LogNormal-equivalente esta encerrado.** +55% pinball degradacao,
   +278-348% width inflation. Re-exponenciacao log1p amplifica erros.

3. **NGBoost Normal merece consideracao parcial:** consegue +30pp cov,
   pinball +14% (acima do threshold mas nao catastrofico). Trade-off
   real: cov_quality vs point_loss. Se downstream usar SO a banda
   (operador conservador) e ignorar point estimate, Normal e' superior
   a LGBM-quantile. Mas a pratica atual usa P50 como point — entao
   NAO promove.

4. **Caminho parametrico com defaults esta FECHADO no replay loop**
   para NE D+1 curt. Hipoteses queued que cobririam variacoes:
   - **H37 (CQR-asymmetric + Mondrian)** — proximo grande swing em cov:
     conformal prediction com bin per sub. Esperado fechar gap NE sem
     tocar P50 (CQR e' post-hoc, mantem o modelo base de point).
   - **H30 (Ridge_alpha10 residual)** — encerrou H3-family em iter_0020
     OLS; rodar Ridge fecha o capitulo basis-rotation.
   - Hyperparameter sweep NGBoost (n_estimators=300, lr=0.05) NAO
     fica como follow-up automatico — H28 testou DEFAULTS conforme
     queue brief. Se Breno quiser explorar, abrir hipotese P4.

5. **H28 -> done, iter_handled=0039.** Sem follow-up automatico.

Promote pertence ao Breno (loader.py defaults + MLflow Registry).
Loop nao executa promote. Aqui o promote e' NEGATIVO (nao alterar
producao); reportado como leitura clara para arquivo.

## Proximo passo

1. **Atualizar leaderboard**: adicionar linha "NGBoost defaults NE
   D+1 — INDETERMINADO_PINBALL_DEGRADA. Calibra cov +30pp mas degrada
   P50 +14%. Nao promove."

2. **H37 prioridade efetiva sobe** (sem mudar P3 formal): proximo
   teste de under-coverage NE deve ser conformal asymmetric, nao
   re-arquitetura de modelo base.

3. **Sinal informativo para UlFor**: o caminho parametrico-default
   nao bate o caminho post-hoc-calibration (H26 conformal symmetric
   ja' indeterminado em iter_0037; H37 asymmetric+Mondrian e' a
   proxima tentativa). UlFor nao precisa rodar nada — Breno decide
   se mantem H37 queued ou se abre nova frente.

## Apendice A: por que NGBoost defaults nao basta

NGBoost minimiza NLL Normal(mu(x), sigma(x)). Isso e' MSE-equivalente
em mu (mean E[y|x]) e sigma calibrada por likelihood. O point estimate
"natural" e' mu — a media. Quando ja sabemos (H27 iter_0038) que P50
(mediana) bate LGB-mean (media) em N+S (cauda longa de zeros), trocar
para parametrico que estima MEAN nao ajuda P50.

Para NGBoost minimizar P50 (mediana) diretamente, precisariamos de Dist
mais sofisticada (Laplace?) — nao queued. Defaults Normal/LogNormal sao
o teste minimo definido por H28. Resultado: caminho default nao basta.

## Apendice B: LogNormal-equiv via log1p Normal vs LogNormal direto

Tentei Dist=LogNormal direto (ngboost.distns.LogNormal) com y_raw>0 mas
y NE tem zeros (alguns dias sem curt) — LogNormal NLL diverge. Solucao
adotada: Dist=Normal sobre target log1p(y_raw), re-exp via expm1 no
predict. Equivalente matematico (Normal em log-space = LogNormal em
raw-space sobre suporte > 0) com tolerancia a zeros.

Resultado: log1p Normal cobre 100% em alguns folds (e.g. NE/v1 fold 1)
mas widths absurdos. Re-exp magnifica scale — quando sigma(x) e'
estimada em log-space e re-exponenciada, multiplica cauda por
exp(sigma)^2/2 (variance amplification). Banda inutil.

A conclusao operacional fecha: nem Normal nem LogNormal-equiv defaults
trazem ganho neto.

## Apendice C: budget reportado

  - Setup (uv add ngboost, install deps): ~30s
  - Script writing: ~5 min
  - Run: ~25s wallclock (30 fits)
  - Handoff + verdict analysis: ~10 min
  - Total iter: ~0.5h (well under 2.5h budget)
