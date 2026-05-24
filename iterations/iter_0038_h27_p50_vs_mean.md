---
alvo: curtailment_d1_point_p50
layer: curtailment
iter_num: 0038
type: hypothesis_test (verdict=CONFIRMADO)
data_utc: 2026-05-25T15:30:00Z
hypothesis: |
  H27 (P3, queued desde iter_0014 — derivada H11 bonus finding):
  "P50 quantile como point estimate substituto em N+S"

  Premissa: H11 (iter_0014) reportou como bonus que LGB-quantile alpha=0.5
  (P50) tem MAE menor que LGB-mean (objective=regression default) em:
    N: -18% MAE mean (0.467k vs 0.578k)
    S: -14% MAE mean (0.97k vs 1.07k)
    SE: -1.6% MAE mean (empate)
    NE: +4.5% MAE mean (P50 perde leve, nao quebra)

  Mecanismo: mediana e' mais robusta que mean para distribuicoes com cauda
  longa de zeros. N tem ~37% dias com curt~0 — mean (MSE) e' arrastado
  pelos picos da cauda; quantile=0.5 minimiza |residuo| direto. KS p<0.0001
  ja confirmou distribution shift em NE+SE (iter_0012 H7), o que explica
  por que o ganho de P50 e' maior em distribuicoes mais skewed (N>S>SE).

  ACEITACAO (a priori, do queue):
    D1: P50 bate LGB-mean em MAE em >=2/3 cells de N+S (>=4/6).
    D2: nao perde >5% MAE em NE+SE (interpretado a nivel de sub-mean).
    CONFIRMADO: D1 AND D2.
baseline_tipo: |
  LGB-mean (objective='regression', defaults identicos a iter_0012 H7,
  iter_0013 H10, iter_0014 H11: n_estimators=300, lr=0.05, num_leaves=31,
  min_child_samples=10, subsample=0.8, colsample_bytree=0.9,
  random_state=0). Predicoes clipped at 0 (curt>=0 hard constraint).

  Tambem reportado por fold:
    - persist_d1 (sanity floor B4)
    - metric_suite completo (mae/r2/f1/nmae/rmse/bias)
baseline_metric: |
  Per cell, MAE mean across 5 folds — LGB-mean baseline:
    NE/v1: 45959   NE/v2: 32181   NE/v3: 31593
    SE/v1:  7752   SE/v2:  7736   SE/v3:  7222
    S/v1:   1157   S/v2:   1064   S/v3:    977
    N/v1:    578   N/v2:    578   N/v3:    553

  Per cell, MAE mean across 5 folds — P50 quantile (alpha=0.5):
    NE/v1: 46396   NE/v2: 34183   NE/v3: 34003
    SE/v1:  7568   SE/v2:  7706   SE/v3:  6967
    S/v1:   1014   S/v2:    963   S/v3:    945
    N/v1:    467   N/v2:    467   N/v3:    458
result_metric: |
  VERDICT: **CONFIRMADO**.

  D1 OK (target): P50 bate LGB-mean em **6/6 cells de N+S** (100% >= 67%
  threshold). Delta MAE mean por sub:
    N:  -17.9% (cells: -18.2, -18.2, -17.1)
    S:  -13.8% (cells: -16.5, -15.6, -9.2)

  D2 OK (constraint): NE+SE preservados a nivel de SUB-mean:
    NE: +4.5% mean (cells: -0.6, +5.6, +8.4) — JUST UNDER threshold +5%.
    SE: -1.6% mean (cells: -1.1, -0.7, -2.9) — empate, leve ganho.

  Bonus diagnostico:
    delta_R2 mean N+S = +0.287 (huge — P50 melhora R² substancialmente em
    distribuicoes com cauda de zeros; LGB-mean tem R² negativo em alguns
    folds N+S, P50 puxa para positivo).
    delta_F1 mean N+S = +0.022 (leve ganho em classificacao p50-event).
    Bias: P50 N+S = 334 MWh vs mean = 210 MWh (P50 underestimate maior em
    magnitude, mas trade-off compensado por MAE -14 a -18%).

  Sanity ressalva NE: a sub-mean passa por margem ESTREITA (+4.5% vs
  +5.0% threshold). Por celula, NE/v2 (+5.6%) e NE/v3 (+8.4%) individuais
  ULTRAPASSAM +5%. Acceptance criteria do queue era a nivel de sub-mean
  (interpretacao mais permissiva), entao passa, mas a 2 das 3 celulas NE
  perdem materialmente — promote em NE viria com asterisco.

  Tabela completa (delta_mae_pct mean per cell, sign convention "<0 = P50 wins"):

    cell    n   MAE_P50   MAE_mean   d%_mean   d%_med    wins   delta_R2
    NE/v1   5    46396     45959      -0.6%    +3.6%     2/5    +0.055
    NE/v2   5    34183     32181      +5.6%    +5.9%     1/5    -0.090
    NE/v3   5    34003     31593      +8.4%    +2.9%     0/5    -0.129
    SE/v1   5     7568      7752      -1.1%    -0.6%     3/5    +0.044
    SE/v2   5     7706      7736      -0.7%    -0.9%     3/5    -0.019
    SE/v3   5     6967      7222      -2.9%    -1.2%     4/5    +0.037
    S/v1    5     1014      1157     -16.5%   -15.0%     3/5    +0.216
    S/v2    5      963      1064     -15.6%   -10.7%     4/5    +0.192
    S/v3    5      945       977      -9.2%   -10.7%     4/5    +0.002
    N/v1    5      467       578     -18.2%   -16.2%     5/5    +0.479
    N/v2    5      467       578     -18.2%   -16.2%     5/5    +0.479
    N/v3    5      458       553     -17.1%   -21.8%     5/5    +0.357
decision: PROMOVE_NS_FLAG_NE
sanity_checks_passed:
  permutation_importance: skipped_inherited
  holdout_temporal_strict: passed_embedded
  leak_detection: skipped_inherited
  baseline_compare: passed_embedded
  distribution_shift: annotated_reuse
  zero_count_n_test_gating: passed
budget_consumido_iter: 0.6
custo_estimado_usd: 0.00
---

# Iter 38 — H27 P50 quantile como point estimate substituto

## Hipotese

H27 (P3, derivada de H11 bonus finding em iter_0014). Testa se o swap de
objective `regression` -> `quantile alpha=0.5` no LGBM e' uniformemente
seguro (drop-in superior) no portfolio de curtailment_d1. A premissa
mecanistica e' que a mediana condicional minimiza MAE diretamente em
distribuicoes assimetricas com massa em zero (cauda longa de eventos
high-curt), enquanto MSE (LGB-mean) sofre arrasto dos picos. N tem ~37%
dias com curt~0, NE tem menos zeros e cauda mais densa.

Aceitacao a priori (do queue):
  D1: P50 bate LGB-mean em >=2/3 (>=4/6) cells de N+S.
  D2: NE+SE nao perde mais que 5% (interpretado a nivel de sub-mean).

## Como foi rodado

**Re-analise** dos artefatos de H11 iter_0014 — sem retreinamento.

H11 ja executou EXATAMENTE este experimento em 12 cells x 5 folds CV
walk-forward (gap 7d, test 60d), com LGBM random_state=0 (bit-exato em
re-runs). Por fold, H11 persistiu:
  - `mae_p50` (LGBM quantile alpha=0.5, predicao clipada em 0)
  - `mae_mean_baseline` (LGBM regression defaults, clipada em 0)
  - `mae_persist_d1` (sanity B4)
  - `metric_suite_p50` e `metric_suite_mean_baseline` completos
    (mae/r2/f1/nmae/rmse/bias)
  - `delta_mae_p50_vs_mean_pct`

H27 difere de H11 apenas na ACCEPTANCE CRITERIA: H11 julgou em NE (com
threshold P50 vs mean <= +10%); H27 julga em N+S (target) com constraint
NE+SE sub-mean <= +5%. Mesma fonte de dados, criterios diferentes.

Re-rodar o LGBM produziria numeros identicos (random_state=0) e custaria
~5 min de compute; re-analise consome <1s e e' bit-exato por construcao.

Script: `scripts/h27_p50_vs_mean.py` (~250 LoC).
Comando: `python scripts/h27_p50_vs_mean.py`
Output: `outputs/iter_0038/h27_p50_vs_mean/{results,summary,verdict,sanity_summary}.json+csv`

## Resultado

**VERDICT: CONFIRMADO.** D1 e D2 ambos passam.

### D1 (target, N+S)

P50 vence LGB-mean em **6/6 cells** de N+S (100% > 67% threshold).
Magnitude do ganho casa exatamente com o queue brief:
  N mean: -17.9% MAE (queue dizia "-18%"). Por celula: -18.2, -18.2, -17.1.
  S mean: -13.8% MAE (queue dizia "-14%"). Por celula: -16.5, -15.6, -9.2.

Por FOLD (mais granular que cell mean), P50 vence em:
  N: 5/5 fold em todas 3 versions = **15/15 folds**
  S: 3/5, 4/5, 4/5 = **11/15 folds (73%)**

R² delta em N+S e' o mais surpreendente: +0.287 mean. LGB-mean tem R²
negativo em varios folds N (mediana arrastada por outliers extremos);
P50 puxa R² para positivo. Em N/v1 fold 5 sample: P50 R²=0.218 vs
LGB-mean R²=-0.676 — magnitude do flip 0.89 absoluta.

### D2 (constraint, NE+SE)

  NE sub-mean delta: +4.5% (just under +5% threshold).
  SE sub-mean delta: -1.6% (empate, leve ganho).

Sub-mean passa, mas NE merece asterisco:
  NE/v1: -0.6% (P50 quase empate)
  NE/v2: +5.6% (P50 perde acima de threshold por celula)
  NE/v3: +8.4% (P50 perde claramente)

Se interpretasse o threshold a nivel de CELULA (mais conservador), NE/v2
e NE/v3 falhariam. O queue brief diz "nao perde >5% em NE+SE" sem
qualificar — interpretei como sub-mean, alinhado com o agregador padrao
do loop (cell-mean -> sub-mean -> verdict).

Mecanismo do underperform de P50 em NE: NE e' o sub com maior magnitude
absoluta (MAE base ~32-46k MWh), com cauda densa de eventos
high-curtailment (gargalo NE->SE recorrente). Quando ha cauda densa, o
mean ainda e' uma estatistica util (E[y|x] util quando y nao tem outliers
solitarios); P50 ignora informacao da cauda. Em N/S, onde a cauda e'
pontual e a massa esta perto de 0, mediana ganha. Esse e' o mecanismo
estatistico classico.

### Sanity (B1-B6)

Herdado de H11 iter_0014 (mesmo X, mesmo split, mesmo seed):
  B1 leak: skipped_inherited (PDP_prev validado iter_0010 H3, 500 shuffles p=0)
  B2 perm: skipped_inherited (mesmo motivo)
  B3 holdout strict: **passed_embedded** (gap=7d em todos 60 folds)
  B4 baseline: **passed_embedded** (P50 vence persist_d1 em 12/12 cells)
  B5 dist_shift: annotated_reuse (KS p<0.0001 NE+SE — explica mecanismo)
  B6 n_test gating: **passed** (n_test 58-60, threshold >=30)

## Decisao

**PROMOVE_NS_FLAG_NE.** Recomendacao operacional:

1. **N e S**: usar P50 (LGBM `objective='quantile', alpha=0.5`) como point
   estimate default. Ganho universal (15/15 folds N, 11/15 folds S),
   custo zero (mesmo treino, mesmo X, swap de hiperparametro). Magnitude
   relevante (-14 a -18% MAE).

2. **SE**: P50 ou mean sao essencialmente equivalentes (-1.6% mean).
   Podemos adotar P50 por consistencia com N+S; sem ganho material.

3. **NE**: manter LGB-mean (status quo). Sub-mean passa por margem
   estreita (+4.5% vs +5% threshold), mas 2/3 celulas NE individualmente
   perdem >5%. NE tem cauda densa que e' EXATAMENTE o regime onde mean
   tem vantagem teorica sobre mediana. Esse e' um achado mecanistico:
   NE precisa de modelo de cauda (ridge canonico H10/H22 ou H37 CQR
   asymmetric), nao mediana.

Promote pertence ao UlFor/forecasting prod (`loader.py` defaults +
MLflow Registry). Loop nao executa promote por convencao — Breno decide.

## Proximo passo

1. **Anotar promote candidato** no leaderboard: P50 e' o novo canonical
   point estimate para N+S (NE mantem LGB-mean ridge canonico).

2. **H27 -> done, iter_handled=0038.** Nenhuma hipotese derivada
   automaticamente: o gap NE -> "cauda densa precisa de cauda-aware
   model" ja esta endereçado pelas hipoteses queued **H28 (NGBoost)**
   e **H37 (CQR-asymmetric + Mondrian, criada no iter_0037)**. Nao
   criar duplicacao.

3. **Sinal para UlFor (informativo, NAO request)**: a substituicao P50
   em N+S e' uma melhoria sem custo e sem mudanca de pipeline (apenas
   o argumento `objective`/`alpha` muda no fit). Custo de implementacao
   estimado em < 30 LoC mais retraining (~minutos). Promote pertence
   ao Breno; nao abrimos request.

## Apendice: por que re-analise e' a abordagem correta

Tres alternativas consideradas:

1. **Re-rodar LGBM com seeds variados** (n_seeds=10) para gerar
   intervalos de confianca por celula. Custo: ~50 min compute,
   refuta-ria CONFIRMADO se algum fold reverte sinal. **Rejeitado**:
   H11 ja rodou random_state=0 e o sinal e' enorme em N (15/15 folds
   100% wins), nao seed-dependent. Em S (11/15) ja ha variance natural
   reportada por fold. Seeds extras nao mudam verdict (apenas precision
   das stds reportadas).

2. **Re-rodar bake-off com features atualizadas** (post-iter_0002, ex:
   iter_0027 H8 features de intercambio). **Rejeitado**: H27 e' sobre
   *objective* swap, nao feature set. Mudar X mistura dois fatores
   confundidores. H11 e' o experimento bem desenhado.

3. **Re-analise** (escolhido). 1s de compute. Bit-exato. Aplica
   acceptance criteria de H27 (foco N+S, sub-mean) em vez de H11 (foco
   NE, cell-mean). Defensivel: nada novo no modelo, so no julgamento.
