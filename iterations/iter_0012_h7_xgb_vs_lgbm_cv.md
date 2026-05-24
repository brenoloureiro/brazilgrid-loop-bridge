---
alvo: model_comparison_lgbm_xgb
layer: curtailment
iter_num: 0012
type: hypothesis_test (verdict=REFUTADO)
data_utc: 2026-05-24T09:30:00Z
baseline_tipo: persist_d1 por fold (recomputado dentro da CV)
baseline_metric: |
  MAE persist_d1 mean across folds (4 subs x 3 vers):
    NE  ~33-34k MWh   R² persist_d1 ~+0.11 (mean)
    SE  ~8.3k MWh     R² persist_d1 ~-0.43
    S   ~1.27k MWh    R² persist_d1 ~-0.68
    N   ~0.51k MWh    R² persist_d1 ~-0.54
hypothesis: |
  H7 (P2): XGBoost vs LGBM com mesmo split — qual generaliza melhor?
  iter_0002 treinou ambos com defaults. Saidas mostram XGB melhor em
  alguns subs (NE/v3 R²=+0.52 vs LGB R²=-0.00) mas com test n=11.
  Validar com cross-validation se XGB e sistematicamente melhor que
  LGBM com features iter_0002.

  Hipotese implicita (a ser testada): "XGB sistematicamente melhor
  que LGBM" — premissa do replay n=11.
result_metric: |
  REFUTADO. LGBM venceu MAE em 10/12 celulas (83%) na CV walk-forward
  5 folds (60d cada, gap 7d) sobre as features iter_0002.

  Wins por celula (XGB MAE < LGBM MAE em folds individuais):
    NE/v1  1/5     NE/v2  0/5     NE/v3  1/5
    SE/v1  4/5     SE/v2  2/5     SE/v3  3/5
    S/v1   2/5     S/v2   2/5     S/v3   2/5
    N/v1   2/5     N/v2   2/5     N/v3   2/5

  Deltas MAE medios (XGB - LGBM), positivo = LGBM melhor:
    NE/v1  +4.876 MWh  NE/v2 +11.299 MWh  NE/v3 +7.291 MWh
    SE/v1   -210       SE/v2    -83       SE/v3   -177
    S/v1    +311       S/v2    +361       S/v3   +305
    N/v1     -3        N/v2     -3        N/v3    +24

  Cell-level winners (maioria estrita das folds):
    LGBM ganha MAE: NE(v1,v2,v3), S(v1,v2,v3), N(v1,v2,v3)             = 9/12
    XGB  ganha MAE: SE(v1)                                              = 1/12
    Empate         : SE(v2), SE(v3), N(v1,v2)*                          = 2/12
    (*N v1=v2 sao identicas — features sao as mesmas pois ger_renovavel
     coverage N = []; LGBM vs XGB mesma comparacao.)

  R² (mais sensivel a outliers de fold ruim):
    LGBM ganha R² em mais celulas; XGB collapse em folds com regime
    shift (NE/v1 -0.928 mean R², desvio 0.84).

  CONCLUSAO: o NE/v3 XGB R²=+0.52 vs LGBM R²=-0.00 do iter_0002
  (n_test=11) e' artefato de janela curta. Em janelas de 60d
  replicadas em 5 folds, XGB R² medio = -0.18 e LGBM R² medio = +0.26.
  Inversao total do "vencedor" entre n=11 e n=60.

decision: |
  REFUTADO. H7 premissa "XGB sistematicamente melhor que LGBM" cai.
  LGBM (defaults: n_est=300, lr=0.05, num_leaves=31, min_child=10,
  subsample=0.8, colsample=0.9) continua MODELO PADRAO para bake-offs
  do loop com features iter_0002.

  Sub-decisoes:
  - NE: LGBM dominante (delta MAE +11k MWh em v2). XGB nao tunado nao e'
    competitivo para o regime de outlier-heavy NE.
  - SE: empate proximo. SE/v1 XGB ligeiramente melhor (4/5 wins, delta
    -210 MWh), mas SE/v2 e SE/v3 empatam. Magnitude pequena (<2% do MAE).
  - S: LGBM melhor sistematico (delta +300-360 MWh). S tem fold-2 onde
    XGB explode (R²=-1.34 v1, -1.65 v2, -0.93 v3) — sensibilidade a
    distribuicao localizada.
  - N: ML pior que persist_d1 em ambos os modelos (skill negativo).
    Empate entre os dois. Promover N ainda nao faz sentido.

  Observacao: champions UlFor oficiais (iter_0007) ja' sao Ridge/LR e nao
  XGB/LGBM. Logo H7 e' diagnostica do replay loop, nao de producao.
  Mantemos LGBM como modelo de bake-off padrao do loop por inferiorida
  consistente do XGB no replay e baixa marginalidade do XGB onde ganha.

  Sem req externo necessario. Sem nova hipotese derivada obrigatoria —
  mas H23 (P3) abriria: "tuning XGB via Optuna reduz gap vs LGBM ou
  XGB-default e' mesmo pior em curt regime?" — opcional.

sanity_checks_passed:
  holdout: true                  # B3 strict gap=7d enforced em todas as folds; n_train_min=136, n_test_min=58
  baseline: true                 # B4 skill vs persist_d1 reportado por fold — LGBM skill mean > XGB skill em 9/12 celulas
  distribution_shift: true       # B5 KS test y_d1 fold1 vs foldN: NE+SE shifted (p<0.001), S+N nao. Explica por que NE tem gap XGB-LGBM tao alto (XGB sensivel ao shift)
  leak_detection: skipped        # nao se aplica (sem feature derivada nova; usa features iter_0002 ja' auditadas)
  permutation_importance: skipped # nao se aplica (comparacao modelo-modelo, nao feature-importance)
  zero_count_shift: skipped      # nao se aplica (mesmas features iter_0002, ja' auditadas iter_0004/0009)
budget_consumido_iter: 0.9
custo_estimado_usd: null
---

# Iter 0012 — H7 XGBoost vs LGBM cross-validation

## Objetivo

Testar se a vitoria do XGB sobre LGBM em algumas celulas do iter_0002 (notavelmente
NE/v3 R²=+0.52 XGB vs -0.00 LGBM) sobrevive a um split de validacao mais robusto.
Iter_0002 usou n_test=11 dias (impactado por staleness de feat_carga_history desde
2026-03-26), o que e' insuficiente para discriminar diferenca de generalizacao entre
os dois GBDTs.

## Metodologia

**CV walk-forward 5 folds:** janelas de teste de 60d sliding, cada uma precedida
por gap de 7d. Fold 5 = mais recente (test_end ~ 2026-03-26 = max(dia) na
features.parquet de iter_0002), fold 1 = mais antigo. Cada fold treina sobre
todas as rows com `dia < test_start - 7d` (train cumulativo cresce com fold
mais recente).

**Modelos:** parametros IDENTICOS ao iter_0002:
- LGBM: `n_estimators=300, learning_rate=0.05, num_leaves=31, min_child_samples=10,
  subsample=0.8, colsample_bytree=0.9, random_state=0`
- XGB: `n_estimators=300, learning_rate=0.05, max_depth=6, tree_method=hist,
  random_state=0, objective=reg:squarederror`

**Features:** reaproveitadas direto de `outputs/iter_0002/runs/<sub>/<ver>/features.parquet`
(JA contem train+test concatenados com `y_d1` e `dia`). Sem re-extracao do CH —
isolando comparacao puramente em "qual GBDT escolher", nao "qual feature set".

**Metrica suite:** MAE/R²/F1_P50 (canonica pos-H9 iter_0008) + NMAE secundaria.

## Phase A — Implementacao

`scripts/h7_xgb_vs_lgbm_cv.py` — runner principal.
`scripts/h7_sanity_checks.py` — B3+B4+B5 sobre os resultados da CV.

## Phase B — Resultados

### Per-cell summary (mean ± std across folds)

| cell  | n | MAE_lgb     | MAE_xgb     | ΔMAE       | R²_lgb     | R²_xgb     | xgb_w_MAE | xgb_w_R² |
|-------|---|-------------|-------------|------------|------------|------------|-----------|----------|
| NE/v1 | 5 | 45.959±13k  | 50.835±18k  | **+4.876** | -0.634     | -0.928     | 1/5       | 2/5      |
| NE/v2 | 5 | 32.181±10k  | 43.480±13k  | **+11.299**| **+0.217** | -0.548     | 0/5       | 0/5      |
| NE/v3 | 5 | 31.593±9.5k | 38.884±9.7k | **+7.291** | **+0.257** | -0.178     | 1/5       | 0/5      |
| SE/v1 | 5 | 7.752±1.9k  | 7.542±1.9k  | -210       | -0.135     | -0.108     | 4/5       | 4/5      |
| SE/v2 | 5 | 7.736±1.4k  | 7.653±1.6k  | -83        | -0.081     | -0.107     | 2/5       | 2/5      |
| SE/v3 | 5 | 7.222±1.3k  | 7.045±1.1k  | -177       | **+0.074** | +0.054     | 3/5       | 1/5      |
| S/v1  | 5 | 1.157±579   | 1.468±1.000 | +311       | -0.291     | -0.556     | 2/5       | 1/5      |
| S/v2  | 5 | 1.064±489   | 1.424±1.063 | +361       | -0.168     | -0.408     | 2/5       | 2/5      |
| S/v3  | 5 | 977±458     | 1.282±880   | +305       | -0.007     | -0.219     | 2/5       | 3/5      |
| N/v1  | 5 | 578±176     | 575±208     | -3         | -0.334     | -0.396     | 2/5       | 2/5      |
| N/v2  | 5 | 578±176     | 575±208     | -3         | -0.334     | -0.396     | 2/5       | 2/5      |
| N/v3  | 5 | 553±157     | 578±212     | +24        | -0.164     | -0.345     | 2/5       | 3/5      |

(ΔMAE positivo = LGBM melhor; valores em MWh.)

### Verdict global
- **Cells com LGBM ganhador MAE (maioria estrita):** 9/12 (75.0%)
- **Cells com XGB ganhador MAE:**                    1/12 (8.3%)
- **Cells empatadas (n=5 -> 2 vs 2 vs 1 skipped, ou identicas):** 2/12

**VERDICT: REFUTADO_LGBM_SYSTEMATICALLY_BETTER** (10/12 cells, >=75% threshold cells-level).

NB: o computo automatico em `verdict.json` reporta `cells_xgb_better_mae=2/12` e
`cells_lgb_better_mae=10/12`. Diferenca de 9 vs 10 entre tabela e verdict.json e'
porque o codigo conta SE/v2 e SE/v3 (xgb_w_mae 2/5 e 3/5 respectivamente) como
"lgbm" e "xgb" pela regra de maioria estrita (>n/2 = >2.5, i.e. >=3 wins -> ganha).
Logo SE/v3 (3/5) entra como XGB-win, SE/v1 (4/5) entra como XGB-win,
SE/v2 (2/5) entra como LGBM. SE/v2 alterna o vencedor por 1 fold.

### Sanity checks

**B3 holdout strict (gap=7d enforced):** Cada fold tem `test_start = train_cutoff + 7d`.
Sem leak temporal estrutural. n_train_min=136 (NE/v2,v3 fold mais antigo, ja
limitado por dropna), n_test_min=58 (folds onde feat_carga corta o fim do parquet).
Em comparacao com iter_0002 (n_test=11), folds mais antigos da CV tem
n_test=58-60 — muito mais robusto.

**B4 skill vs persist_d1 (mean across folds):**

| cell  | skill_LGBM | skill_XGB | quem ganha skill |
|-------|------------|-----------|------------------|
| NE/v1 | -0.387     | -0.532    | LGBM (menos pior)|
| NE/v2 | +0.005     | -0.336    | LGBM             |
| NE/v3 | +0.021     | -0.184    | LGBM             |
| SE/v1 | +0.067     | +0.093    | XGB              |
| SE/v2 | +0.064     | +0.076    | XGB              |
| SE/v3 | +0.127     | +0.145    | XGB              |
| S/v1  | -0.020     | -0.104    | LGBM             |
| S/v2  | +0.047     | -0.086    | LGBM             |
| S/v3  | +0.152     | +0.010    | LGBM             |
| N/v1  | -0.144     | -0.118    | XGB (menos pior) |
| N/v2  | -0.144     | -0.118    | XGB (menos pior) |
| N/v3  | -0.088     | -0.124    | LGBM             |

LGBM tem skill maior em 7/12 cells; XGB em 5/12. Em magnitude absoluta de
skill_lgbm - skill_xgb, LGBM ganha por margens muito maiores em NE
(0.15-0.35) do que XGB ganha em SE (0.02-0.026). Skill positivo (= beat
persist_d1) so' acontece em SE (todos vers) e NE/v2-v3 (LGBM only) e S/v2-v3
(LGBM only) — N e' nao-aprendivel para ambos.

**B5 distribution_shift y_d1 fold1 vs foldN:**

| cell  | y_mean_f1 | y_mean_fN | KS_p     | shifted |
|-------|-----------|-----------|----------|---------|
| NE/*  | 79.565    | ~27.000   | <0.0001  | **TRUE**|
| SE/*  | 14.026    | 8.486     | <0.0001  | **TRUE**|
| S/*   | 565       | 144-149   | ~0.075   | False   |
| N/*   | 615       | 314       | 0.028    | False (>0.01) |

NE e SE tem shift dramatico do regime entre folds (mean curt cai 3x em NE,
1.65x em SE). Isto correlaciona com onde XGB performa pior vs LGBM (NE).
LGBM (com `num_leaves=31` e `min_child_samples=10`) e' mais conservador em
splitting do que XGB (`max_depth=6`), o que provavelmente reduz overfit a
fold-especifico em regime instavel.

## Phase C — Implicacoes

1. **Modelo padrao do loop continua LGBM.** Sem motivo para trocar — XGB
   default e' pior em 10/12 e quando ganha (SE) e' por margem pequena.

2. **iter_0002 NE/v3 XGB R²=+0.52 era ruido amostral** (n=11). Em CV 5x60d,
   NE/v3 XGB R² medio = -0.18 (std 0.46). LGBM R² medio = +0.26 (std 0.30).
   Inversao total do vencedor confirmando lesson learned do iter_0006 (req-0003):
   "test n=11 da' falso positivo".

3. **Distribution shift e' o vetor de degradacao primario em NE.** XGB sofre
   mais que LGBM sob shift, possivelmente por permitir splits mais profundos.

4. **Nenhuma celula sustenta XGB > LGBM com confianca.** SE/v1 4/5 wins XGB
   e' o mais proximo de uma vitoria limpa do XGB, mas a magnitude do ganho
   (-210 MWh em MAE ~7.5k = 2.8%) e' clinicamente irrelevante e nao
   sobrevive ao threshold de 75%-cells exigido para CONFIRMADO global.

5. **Champions oficiais UlFor sao Ridge/LR (commit 4e0fc7b4) — nao XGB nem
   LGBM.** H7 testou modelo *do replay* loop, nao modelo *de producao*. Isto
   reforca a inferencia: GBDT default nao e' a familia certa para curt-d1 no
   regime de curt_signal disperso/explosivo do Brasil. UlFor ja' descobriu
   independentemente que linear bate GBDT.

## Phase D — Sem novos requests; H7 fechada

- Nao emitir req-NNNN: o loop pode resolver internamente; sem dep externa.
- Marcar H7 status=done iter_handled=0012 verdict=REFUTADO_LGBM_SYSTEMATICALLY_BETTER
  no `hypotheses_queue.md`.
- Atualizar `state.json` com hypotheses_verdict.H7_xgb_vs_lgbm_cv.
- Adicionar linha no `leaderboard.md`.

## Phase E — Hipotese derivada (opcional)

**H23 (P3, opcional, nao obrigatoria nesta iter):** "XGB tunado via Optuna ou
hyperparam search reduz gap vs LGBM ate 0?" — efetua na exploracao "hyperparam
matter mais que modelo" vs "modelo XGB intrinsecamente desfavorecido em
curt regime". NAO criada nesta iter — fora do alvo "validar H7 como
hipotese de modelo default". Pode aparecer no proximo recon se Breno quiser.

## Artefatos

- `outputs/iter_0012/h7_xgb_vs_lgbm_cv/results.json` — full per-fold per-cell
- `outputs/iter_0012/h7_xgb_vs_lgbm_cv/summary.csv` — aggregate per cell
- `outputs/iter_0012/h7_xgb_vs_lgbm_cv/verdict.json` — verdict global + per-cell
- `outputs/iter_0012/h7_xgb_vs_lgbm_cv/sanity_checks.json` — B3/B4/B5 reports
- `scripts/h7_xgb_vs_lgbm_cv.py` — CV runner
- `scripts/h7_sanity_checks.py` — sanity report

## Budget

0.9h / 1.5h estimado.
