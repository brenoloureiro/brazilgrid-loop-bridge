---
alvo: h25_stacker_ridge
layer: curtailment
iter_num: 0035
type: hypothesis_test (verdict=REFUTADO_NO_GAIN)
data_utc: 2026-05-25T06:30:00Z
hypothesis: |
  H25 (P3, queued desde iter_0014 — derivada H10 follow_up):
  "Stacker meta-modelo (Ridge sobre base preds) supera weighted average?"
  Premissa: H10 (iter_0013) mostrou que pesos analiticos (inv_mae,
  inv_mse) > grid-search empirico ens_opt_alpha em inner_val 30d. H25
  sobe um nivel: stacker via Ridge regression sobre features =
  [LGB_pred, persist_d1_pred, ma7_pred, climatologia_doy_pred] no
  inner_val pode capturar interacoes lineares simples (peso de persist
  depende do nivel do LGB, etc).
  Risco declarado pelo queue: inner_val 30d e' pequeno para Ridge
  robusto (4 baselines + intercept = 5 parametros) — usar shrinkage
  forte (alpha alto) ou inner_val maior (60d, ao custo de perder fold).
  Aceitacao: stacker bate H10 best em maioria das cells por >=2% MAE
  reduction (significancia clinica).
baseline_tipo: |
  H10 inv_mae ensemble (canonical do iter_0013, peso analitico:
  w_lgb = inv_mae_lgb_val / (inv_mae_lgb_val + inv_mae_persist_val);
  w_persist = 1 - w_lgb). 2 bases (LGB + persist_d1).
  Tambem reportados: LGB-only, persist_d1, ma7, climatologia_doy.
baseline_metric: |
  Per cell (MAE mean across 5 folds), H10 inv_mae baseline:
    NE/v1: 33220   NE/v2: 27993   NE/v3: 27528
    SE/v1:  7199   SE/v2:  7211   SE/v3:  6937
    S/v1:   1108   S/v2:   1062   S/v3:   1006
    N/v1:    478   N/v2:    478   N/v3:    475
result_metric: |
  Per cell (MAE mean, best Ridge variant) vs H10 inv_mae baseline:
    NE/v1: 33927 (+2.1%, ridge_no_intercept_a10, wins 2/5)
    NE/v2: 31305 (+11.8%, ridge_no_intercept_a10, wins 2/5)
    NE/v3: 32049 (+16.4%, ridge_no_intercept_a10, wins 1/5)
    SE/v1:  8686 (+20.6%, ridge_no_intercept_a10, wins 0/5)
    SE/v2:  7944 (+10.2%, ridge_no_intercept_a10, wins 2/5)
    SE/v3:  7748 (+11.7%, ridge_no_intercept_a10, wins 2/5)
    S/v1:   1523 (+37.4%, ridge_no_intercept_a10, wins 3/5)
    S/v2:   1528 (+44.0%, ridge_no_intercept_a10, wins 3/5)
    S/v3:   1911 (+90.0%, ridge_no_intercept_a10, wins 2/5)
    N/v1:    672 (+40.7%, ridge_no_intercept_a10, wins 1/5)
    N/v2:    672 (+40.7%, ridge_no_intercept_a10, wins 1/5)
    N/v3:    563 (+18.5%, ridge_a100, wins 2/5)
  ---
  Per-sub mean_pct_delta vs H10 (across versions):
    NE: +10.1%   SE: +14.2%   S: +57.1%   N: +33.3%
  Confirma >=2% reducao em 0/12 cells (threshold majoritario = 7/12).
  Ridge stacker NUNCA bate H10 inv_mae — refutacao limpa e robusta.
decision: |
  REFUTADO_NO_GAIN. Premissa de H25 (stacker linear sobre 4 bases supera
  weighted average analitico em janela curta) NAO se sustenta.

  Mecanismo confirmado empiricamente:
    1. Ridge com intercept (alpha=0.1..100): MAE explode 50x-200x em
       NE/v1 e cells com gen alto (1.7M MAE — overfit catastrofico
       inner_val). Mesmo alpha=100 nao regulariza o suficiente em
       n_val=30 com 4 colineares.
    2. Ridge SEM intercept (alpha=10): variante menos pior — clipa a
       coef pesos mas ainda overfit MAE_test/MAE_inner_val media
       64-283% por cell (single fold extremo: NE/v1 fold com 28280%
       degradacao).
    3. H10 inv_mae: 2 escalares MAE_lgb_val + MAE_persist_val → 2 pesos
       sum-to-1. SEM intercept, SEM 4 features, SEM minimizacao
       empirica de erro em inner_val. Robusto a shift por construcao.

  Convergencia com lessons:
    - H10 (CONFIRMADO_NE_SE iter_0013): pesos analiticos > grid empirico
    - H24 (CONFIRMADO ridge_lr_NE_SE_N iter_0024): inv_mae paradigm com
      champions Ridge/LR ate 4x mais ganho em SE
    - H25 (REFUTADO iter_0035): subir capacidade do ensemble (4-feat
      stacker) PIORA — gargalo NAO e' esquema de ponderacao mas sim
      sinal residual apos LGB. Acrescentar ma7/clim_doy nao destrava.

  Implicacao operacional: ensemble inv_mae H10 continua state-of-art
  para curt D+1 multi-base em inner_val curto. UlFor production
  ensemble (champions Ridge_NE/LR_SE/LR_S/Ridge_N) mantem-se
  inalterada por esta iter.

  H37 derivada? NAO. O risco previsto pelo queue ("30d pequeno")
  concretizou-se; nao ha trial obvio (60d perde fold, NNLS = inv_mae
  reescrito). Marcar H25 done; nao criar derivada.
sanity_checks_passed:
  leak_detection: true            # Ridge fit em Z_val + y_val; coefs aplicadas a Z_te sem retreino; bases test usam full_train (inclui inner_val por design H10) — sem leak de y_te
  permutation_importance: skipped # Ridge 4-feat: coef E' importance; perm n_val=30 ruidoso e redundante
  holdout_temporal_strict: true   # 5 folds walk-forward, test=60d, gap=7d, inner=30d; 60 folds_ok / 0 skipped
  baseline_compare: true          # LGB, persist_d1, ma7, climatologia_doy + H10 inv_mae reproduzido per fold
  distribution_shift: reported    # MAE_test/MAE_inner_val mean 64%-283% por cell (NE/v1 28280% fold extremo)
  zero_count_shift: reported      # Ridge preds clipped >=0 (linear sem garantia)
budget_consumido_iter: 0.7
custo_estimado_usd: 0
ulfor_head_no_inicio: 80620230
novos_requests: 0
follow_ups_created: []
---

# Iter 0035 — H25 Stacker Ridge meta-modelo (4 bases)

## Hipotese

H25 (queue) postulou que um stacker via Ridge regression treinado em
inner_val (30d) sobre 4 base preds — LGB, persist_d1, ma7, climatologia
DOY — capturaria interacoes lineares simples que weighted average
analitico H10 (inv_mae sobre LGB + persist) nao consegue.

Risco explicitado no queue: "inner_val 30d e' pequeno para Ridge robusto
(4 baselines + intercept = 5 parametros). Considerar shrinkage forte ou
inner_val maior ao custo de perder fold."

Threshold de aceitacao: bate H10 best em maioria das cells por **>=2% MAE
reduction** (significancia clinica).

## Como foi rodado

**Script**: `scripts/h25_stacker_ridge_cv.py`
**Comando**:
```bash
cd loops/forecast-mega-loop
python scripts/h25_stacker_ridge_cv.py
```

**Dataset**: `outputs/iter_0002/runs/<sub>/<ver>/features.parquet`
(4 subs × 3 versions = 12 cells; ~440 rows/cell, range 2024-12-15..2026-03-26
em NE; outras subs janelas similares).

**CV walk-forward** (heredado H10 iter_0013):
- 5 folds; test_window = 60d; gap = 7d; inner_val = 30d
- LGB params identicos H10: n_est=300, lr=0.05, num_leaves=31,
  min_child=10, subsample=0.8, colsample=0.9, seed=0

**Bases predictivas (4)**:
| Base | Formula | Dependencia |
|---|---|---|
| `lgb_pred`         | LGBM regressor sobre features completas iter_0002 | full_train (test) / train_inner (val) |
| `persist_d1_pred`  | y[i-1] em test; y_tr_last em test[0]            | apenas valores realizados anteriores |
| `ma7_pred`         | mean(y[i-1..i-7]) (causal, sem leak)            | train_tail + test prior; sem leak |
| `climatologia_doy` | mean(y_train onde doy(dia)==doy)                | lookup do train; fallback ymean |

**Stackers** (5 variantes de regularizacao):
| Variant | alpha | fit_intercept | Notas |
|---|--:|:--:|---|
| `ridge_a01`              | 0.1   | sim | fraco; esperado overfit |
| `ridge_a1`               | 1.0   | sim | padrao sklearn |
| `ridge_a10`              | 10.0  | sim | shrinkage forte |
| `ridge_a100`             | 100.0 | sim | quase media equal |
| `ridge_no_intercept_a10` | 10.0  | nao | suprime grau de liberdade extra |

Implementacao: Ridge closed-form `beta = (X'X + alpha*I)^-1 X'y`,
intercept tratado em coluna separada (nao penalizada).

**Protocolo zero-leak Ridge↔test**:
1. inner_train = train_full[< inner_start]; inner_val = train_full[>= inner_start]
2. LGB fit em inner_train → bases em inner_val
3. Ridge fit em Z_val (col stack 4 bases) + y_val → coefs
4. LGB **retreinado** em full_train (= train_inner + inner_val) → bases em test
5. Aplica coefs do step 3 ao Z_te do step 4 → ensemble_stacker_pred
6. Clip a >=0 (curt nao-negativo); metric_suite

**H10 baseline reproduzida no mesmo fold**:
- pesos = inv_mae normalized sobre (LGB, persist) em inner_val
- aplicados ao test (mesmas bases). Igual `ens_inv_mae` iter_0013.

## Numeros entregues

### Tabela principal (mean MAE across 5 folds per cell)

| Cell | LGB | H10 inv_mae | best Ridge | best variant | pct vs H10 | wins vs H10 |
|---|--:|--:|--:|---|--:|---:|
| NE/v1 | 45959 | 33220 | 33927 | ridge_no_intercept_a10 | **+2.1%** | 2/5 |
| NE/v2 | 32181 | 27993 | 31305 | ridge_no_intercept_a10 | **+11.8%** | 2/5 |
| NE/v3 | 31593 | 27528 | 32049 | ridge_no_intercept_a10 | **+16.4%** | 1/5 |
| SE/v1 |  7752 |  7199 |  8686 | ridge_no_intercept_a10 | **+20.6%** | 0/5 |
| SE/v2 |  7736 |  7211 |  7944 | ridge_no_intercept_a10 | **+10.2%** | 2/5 |
| SE/v3 |  7222 |  6937 |  7748 | ridge_no_intercept_a10 | **+11.7%** | 2/5 |
| S/v1  |  1157 |  1108 |  1523 | ridge_no_intercept_a10 | **+37.4%** | 3/5 |
| S/v2  |  1064 |  1062 |  1528 | ridge_no_intercept_a10 | **+44.0%** | 3/5 |
| S/v3  |   977 |  1006 |  1911 | ridge_no_intercept_a10 | **+90.0%** | 2/5 |
| N/v1  |   578 |   478 |   672 | ridge_no_intercept_a10 | **+40.7%** | 1/5 |
| N/v2  |   578 |   478 |   672 | ridge_no_intercept_a10 | **+40.7%** | 1/5 |
| N/v3  |   553 |   475 |   563 | ridge_a100             | **+18.5%** | 2/5 |

**0/12 cells confirmam >=2% MAE reduction** (threshold majoritario = 7/12).
Ridge stacker NUNCA bate H10 inv_mae — refutacao em escala universal.

### Per-sub summary (mean_pct_delta best_ridge vs H10)

| Sub | n cells | mean pct_delta | best cell | worst cell |
|---|--:|--:|---|---|
| NE | 3 | +10.1% | NE/v1 (+2.1%) | NE/v3 (+16.4%) |
| SE | 3 | +14.2% | SE/v2 (+10.2%) | SE/v1 (+20.6%) |
| S  | 3 | +57.1% | S/v1 (+37.4%)  | S/v3 (+90.0%) |
| N  | 3 | +33.3% | N/v3 (+18.5%)  | N/v1=N/v2 (+40.7%) |

### Variantes com intercept catastroficas (NE/v1 representativo)

Ridge com intercept overfit inner_val a ponto de explodir test:

| Variant | MAE mean NE/v1 | vs LGB |
|---|--:|--:|
| LGB-only           |    45,959 | -    |
| H10 inv_mae        |    33,220 | -28% |
| ridge_no_intercept | **33,927**| -26% (so a melhor variante Ridge) |
| ridge_a01          | 1,709,902 | +3622% (overfit catastrofico) |
| ridge_a1           | 1,709,435 | +3621% |
| ridge_a10          | 1,704,774 | +3611% |
| ridge_a100         | 1,659,553 | +3511% |

O intercept absorve nivel de gen → quando o regime test e' diferente
do regime inner_val, intercept off-by-tens-of-MWh × n_test atira o MAE.

### Distribution shift (sanity check)

MAE_test / MAE_inner_val (% degradacao):

| Cell | mean degradation | max single fold |
|---|--:|--:|
| NE/v1 | **+4545%** | +28280% (extremo) |
| NE/v2 |   +149%   |   +457%           |
| NE/v3 |   +159%   |   +509%           |
| SE/v1 |   +133%   |   +309%           |
| SE/v3 |    +81%   |   +173%           |
| S/v3  |   +283%   |   +969%           |
| N/v1  |    +64%   |   +171%           |

Ridge memoriza inner_val (MAE baixo la); test em outra distribution
**arrasa**. Inv_mae H10, ao usar so escalar MAE para pesar, nao
memoriza inner_val → robusto a shift por construcao.

## Sanity checks (6 default + queue required)

| Check | Status | Detalhe |
|---|---|---|
| leak_detection | PASS | Ridge fit em Z_val+y_val; coefs aplicadas a Z_te sem retreino; bases test usam full_train (inclui inner_val por design H10) — sem leak de y_te. |
| permutation_importance | SKIPPED | Ridge 4-feat: coef E' importance; perm n_val=30 ruidoso e redundante. Reportado em ridge_coefs.csv. |
| holdout_temporal_strict | PASS | 5 folds walk-forward, test=60d, gap=7d, inner=30d. 60 folds_ok / 0 skipped em 12 cells × 5 folds. |
| baseline_compare | PASS | LGB-only, persist_d1, ma7, clim_doy + H10 inv_mae reproduzido per fold. |
| distribution_shift | REPORTED | MAE_test/MAE_inner_val mean 64-283% por cell (extremo NE/v1 28280% num fold). |
| zero_count_shift | REPORTED | Ridge preds clipped >=0 (linear sem garantia). F1_p50 inclui binarizacao. |

**Queue required `[holdout, leak, baseline]`: TODOS PASS.**

## Decisao final

**REFUTADO_NO_GAIN.**

Justificativa formal:
- 0/12 cells atingem aceitacao queue (>=2% MAE reduction vs H10).
- Worst pct_delta = +90.0% (S/v3 ridge_no_intercept_a10).
- Best pct_delta = +2.1% (NE/v1 ridge_no_intercept_a10) — ainda pior que
  H10 e abaixo do threshold clinico negativo (-2%).
- 4/5 variantes Ridge (com intercept) explodem MAE 50x-200x em alguns
  cells, evidenciando overfit catastrofico em n_val=30.
- A unica variante "respeitavel" (ridge_no_intercept_a10) e' o teto
  factivel da capacidade Ridge nesta janela — e mesmo essa perde por
  +10% a +90% mean.

Mecanismo: distribution shift severa entre inner_val(30d) → test(60d)
desfaz pesos Ridge empiricamente otimizados. H10 inv_mae nao depende
de minimizacao empirica de erro em inner_val (apenas escala 1/MAE) →
robusto.

**Implicacao**: o gargalo do ensemble curt D+1 multi-base NAO e' o
esquema de ponderacao mas o sinal residual disponivel apos LGB. Subir
capacidade (4-feat stacker em vez de 2-base inv_mae) **piora**.

## Follow-up

**Nenhuma hipotese derivada criada**.

Possibilidades descartadas:
- H37 = inner_val 60d (perde fold; +30d nao destrava overfit estrutural
  com 4 features colineares + shift de regime do test 60d).
- H38 = NNLS sum-to-1 nao-negativo 4 bases (= inv_mae estendido com
  ma7+clim; ja' testado implicitamente pela variante no_intercept_a10
  que aproxima isso sob shrinkage forte).
- H39 = stacker GBDT no lugar de Ridge (H22 ja' refutou GBDT em janela
  curta sob distribution shift — overfit ainda pior).

O queue ja' tem hipoteses mais promissoras enfileiradas (H26 conformal
prediction P/I bands, H27+, etc). Avancar.

## Artefatos

```
outputs/iter_0035/h25_stacker_ridge/
├── results.json          # per fold completo (12 cells × 5 folds × 5 ridge variants + bases + H10)
├── summary.csv           # aggregate por cell (60 rows campos: mae_<base/ridge/h10>_mean/std, pct, wins)
├── verdict.json          # CONFIRMADO/REFUTADO/INDETERMINADO + per-cell decision
├── ridge_coefs.csv       # coefs por (cell, fold, variant): w_lgb, w_per, w_ma7, w_clima, intercept
└── sanity_checks.json    # 6 default + queue required summary + shift observations
```

**Script**: `scripts/h25_stacker_ridge_cv.py`

## Lessons learned

1. **Inner_val=30d × 4 features colineares + intercept overfit catastrofico**.
   Ridge sem intercept e' a unica variante respeitavel — e ainda perde
   para H10 inv_mae por +10-90%. ERM em janelas curtas com colinearidade
   media e' frutifero so quando n >> p e shift moderado.

2. **H10 inv_mae paradigm continua state-of-art para curt D+1 ensemble
   em janelas curtas**. Esquemas mais simples (sem fit empirico em
   inner_val, escala derivada por 1/MAE escalar) generalizam melhor sob
   distribution shift documentada em iter_0012 (KS<1e-4 NE+SE).

3. **Adicionar bases (ma7, clim_doy) NAO destrava ensemble**. Em todas
   as cells, lgb sozinho ou (lgb + persist) inv_mae bate qualquer combo
   estendida com ma7/clim_doy via Ridge. Sinal residual apos LGB e'
   limitado para estes targets.

4. **Test-time degradation Ridge > 100% por cell** e' o sintoma diagnostico
   chave de overfit ensemble. Comparar MAE_inner vs MAE_test como proxy
   futuro de robustez de qualquer stacker — se >> 30%, abandonar.

5. **H10 + H22 + H25 juntos delimitam o espaco "ensemble curt D+1"**:
   H10 → pesos analiticos > grid empirico (n=30 small).
   H22 → GBDT residual nao supera OLS linear (small feature space).
   H25 → Ridge stacker 4-feat nao supera inv_mae 2-feat (overfit val).
   Conclusao: novo ganho EXIGE novo sinal (features upstream, dado
   exogeno novo), nao novo esquema de combinacao.

## Convergencia com producao UlFor

UlFor production champions (Ridge_NE / LR_SE / LR_S / Ridge_N — endpoint
`/api/forecast/d1`) NAO usam ensemble multi-base. H10 ja' havia
documentado que inv_mae (lgb + persist) ganharia (CONFIRMADO_NE_SE),
mas mesmo assim UlFor optou por champion Ridge/LR puro (motivo: H22
em SE/v3 e iter_0011 baseado em janela 14d real vs 60d CV — UlFor
prefere champion single-model com val 14d real). H25 refuta a opcao
oposta (adicionar capacidade ao ensemble), reforcando que **o
sweet-spot atual e' single-model linear (Ridge/LR) em UlFor + inv_mae
multi-base no loop ML offline**. Nada a mudar em producao.
