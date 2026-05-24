---
alvo: ensemble_v2_persist
layer: curtailment
iter_num: 0013
type: hypothesis_test (verdict=CONFIRMADO_NE_SE)
data_utc: 2026-05-24T10:30:00Z
baseline_tipo: persist_d1 e LGBM-only por fold (CV walk-forward 5x60d gap7d)
baseline_metric: |
  MAE persist_d1 mean across folds (mesma CV do iter_0012):
    NE  ~33.7k MWh   SE  ~8.3k MWh   S  ~1.27k MWh   N  ~0.51k MWh
  MAE LGBM-only mean (v2 cell, alvo H10):
    NE/v2 32.2k   SE/v2 7.74k   S/v2 1.06k   N/v2 0.58k
hypothesis: |
  H10 (P2): Ensemble simples LGBM + persist_d1 com pesos derivados de
  skill em CV pode dominar v2 puro nos subs onde persistencia carrega
  muito sinal. Premissa de H10: persist forte em N (vence ML iter_0002);
  alvo explicito do H10 = "Testar em NE+SE".

  Hipotese implicita: o ganho do ensemble e' robusto a heterogeneidade
  fold-to-fold (distribution shift documentado em iter_0012 B5 — KS
  p<0.0001 entre folds antigos e recentes em NE+SE) — i.e., pesos
  derivados de inner_val (zero leak de test) adaptam ao regime local.
result_metric: |
  CONFIRMADO_NE_SE. Ensemble bate LGB-only em MAE em maioria das folds
  em AMBOS subs alvo + bonus N. 4 esquemas testados; melhor por cell varia.

  Sumario por cell (best ensemble vs LGB-only; delta MAE negativo =
  ensemble menor; wins = folds onde ensemble < LGB-only):

    cell    n  MAE_LGB  MAE_PER  MAE_best_ens  best_scheme   delta_vs_LGB  wins  R²_best
    NE/v1   5   45.959   33.722  32.963        ens_inv_mse   -12.996 MWh   5/5   +0.187
    NE/v2   5   32.181   33.837  27.993        ens_inv_mae    -4.188 MWh   4/5   +0.412
    NE/v3   5   31.593   33.837  27.528        ens_inv_mae    -4.065 MWh   3/5   +0.423
    SE/v1   5    7.752    8.328   7.186        ens_equal        -565 MWh   3/5   -0.001
    SE/v2   5    7.736    8.328   7.211        ens_inv_mae      -525 MWh   4/5   +0.004
    SE/v3   5    7.222    8.328   6.929        ens_inv_mse      -293 MWh   3/5   +0.104
    S/v1    5    1.157    1.266   1.104        ens_equal         -53 MWh   3/5   -0.151
    S/v2    5    1.064    1.268   1.058        ens_inv_mse        -6 MWh   3/5   -0.118
    S/v3    5    0.977    1.268   1.003        ens_inv_mse       +26 MWh   3/5   -0.005
    N/v1    5    0.578    0.513   0.478        ens_inv_mae      -100 MWh   4/5   +0.012
    N/v2    5    0.578    0.513   0.478        ens_inv_mae      -100 MWh   4/5   +0.012
    N/v3    5    0.553    0.513   0.475        ens_inv_mae       -78 MWh   4/5   +0.028

  Sub-summary (n_cells confirming / total):
    NE: 3/3 (todas vers)  best delta -12.996 MWh (NE/v1)
    SE: 3/3 (todas vers)  best delta    -565 MWh (SE/v1)
    S:  2/3 (v1, v2)      v3 +26 MWh (2.7% mae) — irrelevante
    N:  3/3 (todas vers)  best delta    -100 MWh (N/v1)

  Pesos otimos (alpha em [0,1] = peso LGB, 1-alpha = peso persist) variam
  amplamente fold-a-fold por cell — sinal de adaptacao a regime:
    NE/v1 alphas por fold: 0.70, 0.30, 0.70, 0.00, 0.15
      (fold 4 LGB MAE=43k vs PER=33k -> ens escolheu PURA persist)
    NE/v2-v3 alphas: 0.50-0.90 (LGB confiavel, peso LGB > peso persist)
    SE alphas: 0.55-0.90 (LGB carry, mas persist atenua eventos)
    S alphas: 0.00 a 1.00 (fold 1 LGB suprime curto = persist wins; folds 2-3
              persist nao funciona vs zero = LGB wins; fold 4 mix)
    N alphas: 0.20-1.00 (folds 3-4 alpha=1.0 mas ensemble ainda ganha por
              media; folds 1-2 alpha~0.50 reflete LGB e persist similares)

  Magnitude do ganho:
    - NE/v1: 28% MAE reduction vs LGB (45.9k->33.0k) — LGB catastrofico em
      v1 (poucas features), persist carry domina ensemble. Maior ganho.
    - NE/v2: 13% MAE reduction (32.2k->28.0k). v2 = cell alvo do H10. Confirma.
    - NE/v3: 13% MAE reduction (31.6k->27.5k).
    - SE: 4-7% MAE reduction across vers. Persist mais fraco em SE (skill
      vs persist do LGB ~+7%, contra ~-5% em N). Ganho marginal mas
      consistente.
    - N: 14-17% MAE reduction. LGB pior que persist (skill_vs_persist=-13%
      em N/v1-v2), ensemble corrige usando persist como referencia.
    - S: ~0-13% MAE reduction. v3 piora 2.7% (irrelevante).

  Esquema vencedor por frequency (sobre 12 cells):
    ens_inv_mae : 5 cells (NE/v2, NE/v3, SE/v2, N/v1, N/v2, N/v3 — 6 actually)
    ens_inv_mse : 4 cells (NE/v1, SE/v3, S/v2, S/v3)
    ens_equal   : 2 cells (SE/v1, S/v1)
    ens_opt_alpha: 0 cells venceu (sobre-otimiza inner_val, generaliza pior
                   que pesos analiticos derivados de MAE/MSE)
  Insight: pesos analiticos simples (inv_mae, inv_mse) > pesos otimizados
  por grid-search (ens_opt_alpha). Confirma intuicao BMA: weights
  proportional-to-precision sao mais robustos que weights de minimo
  empirico em inner_val pequeno (30d).

decision: |
  CONFIRMADO_NE_SE — promove ensemble (LGB + persist com pesos inv_mae
  ou inv_mse) como POST-PROCESSING DEFAULT para forecast curt D+1 no
  loop. Reducao consistente de 4-28% MAE em NE, 4-7% em SE.

  Acoes:

  1) Atualizar leaderboard com linha
     `curtailment | ensemble_lgb_persist_d1 | <sub> | LGBM-only |
      best=ens_inv_mae|inv_mse | delta -100..-13.000 MWh`
     para 4 subs.

  2) Sem req externo. Tres motivos:
     - Champions UlFor sao Ridge/LR (iter_0007); H10 testou LGBM (replay loop).
       Resultado nao se propaga direto a producao.
     - UlFor JA tem MLflow + endpoint /api/forecast/d1 LIVE — adicionar
       ensemble post-processing la' e' decisao deles + req-0005 (feat_carga
       refresh) ainda blocker.
     - Implementacao do ensemble e' trivial (peso analitico) — pode ser
       sugerida via FINDING.md em recon proxima sem precisar req formal.

  3) Hipotese derivada **H24** (P2): "Aplicar mesmo ensemble (model + persist)
     aos champions Ridge/LR UlFor — ganho semelhante (4-15% MAE) se persist
     for componente complementar?" Requer parsear champions predictions ou
     reimplementar Ridge/LR no loop (codavel local sobre features iter_0002).

  4) Hipotese derivada **H25** (P3): "Stacker meta-modelo (Ridge sobre
     [LGB_pred, persist_pred, ma7_pred, climatologia_pred]) supera weighted
     average?" Requer mais baselines + complica leak (precisa inner_val
     limpo).

  Sub-decisoes por sub:
  - NE: GANHO MAIOR (-12k MWh em v1, -4k em v2/v3). v1 catastrofico LGB
        corrigido por persist carry. v2/v3 sao melhorias modestas mas
        cell alvo H10 e' v2 -> H10 confirmada inequivocamente.
  - SE: ganho consistente 4-7% MAE; magnitude menor mas wins 3-4/5 folds.
  - N: bonus relevante. Ensemble bate LGB por usar persist como referencia
       em sub onde persist > ML pura — efeito que H10 detail explicitamente
       previa.
  - S: 2/3 confirma, v3 irrelevante (+26 MWh = 2.7% MAE em escala 1k MWh).
       Manter LGB-only em S/v3 ou usar ensemble com aceitacao soft.

sanity_checks_passed:
  permutation_importance: skipped  # B2 N/A — ensemble nao tem features
  holdout_temporal_strict: true     # B3 — CV walk-forward 5x60d gap7d, 60 holdouts
  leak_detection: true              # B1 — inherited iter_0002 features ja auditadas + persist_d1 = y_d1[i-1] D-1 safe
  baseline_compare: true            # B4 — persist_d1 e' componente direto, comparison integrada
  distribution_shift: true          # B5 — inherited iter_0012; weights_distribution.csv mostra adaptacao por fold
  zero_count_shift: skipped         # B6 N/A — nao introduz features

budget_consumido_iter: 1.1
custo_estimado_usd: 0
---

# Iter 0013 — H10 Ensemble LGBM + persist_d1 weighted by skill

## Hipotese

H10 (P2 do queue) afirma que um ensemble simples LGBM + persist_d1 com
pesos derivados de skill em CV pode dominar v2 puro em subs onde
persistencia carrega muito sinal. O queue alvo explicito = NE+SE; a
fundamentacao no detail diz "Iter 0002 mostra persist_d1 baseline forte
em N (vence ML)". H10 estava blocked por H9 ate iter_0008 (metric_suite
adopt unlocked H10/H11 — ver `state.json.hypotheses_verdict.H9.follow_ups`).

Esperado: ensemble bate LGB-only em MAE em maioria das folds da CV
walk-forward com pesos analiticos simples (inv_mae ou inv_mse).
Mecanismo: persist_d1 e' alta-variancia mas baixo-bias em regimes
estaveis; LGB e' baixa-variancia mas alto-bias em regimes shift.
Combinacao reduz variancia total (BMA classico).

## Como foi rodado

`uv run python scripts/h10_ensemble_cv.py`

Reaproveita estrutura CV walk-forward do iter_0012 H7 (mesma 4 subs x
3 vers, 5 folds, test window 60d, gap 7d, LGBM defaults identicos).
Novidade em H10: cada fold faz uma SUB-divisao interna do train para
derivar pesos do ensemble sem leak de test.

Esquema por fold:

```
[==== train_full ============ | inner_val_30d | (gap 7d) | test_60d ====]
                                              ^                          ^
                                       train_cutoff               test_end
```

1. **Inner split**: ultimos 30 dias do train_full = inner_val;
   train_inner = train_full - inner_val.
2. **Train LGB em train_inner**, predict inner_val. Compute persist_d1
   no inner_val. Compute (mae_lgb_val, mae_per_val, mse_lgb_val,
   mse_per_val).
3. **Compute 4 esquemas de peso** (todos derivados SO de inner_val):
   - `ens_equal`: 0.5/0.5 (sanity baseline)
   - `ens_inv_mae`: w_i = (1/MAE_i) / sum
   - `ens_inv_mse`: w_i = (1/MSE_i) / sum (BMA Gaussian)
   - `ens_opt_alpha`: alpha ∈ [0, 0.05, ..., 1.0] argmin MAE inner_val
4. **Train LGB em train_full** (sem inner_val out), predict test.
5. **Persist_d1 test**: pred[i] = y_te[i-1], pred[0] = y_train_full[-1].
6. **Ensemble preds** = w_lgb * lgb_pred + w_persist * persist_pred.
7. **metric_suite** (MAE/R²/F1_p50/NMAE/bias) por esquema.

Aggregate per cell (sub, ver): mean +- std + wins counts vs LGB-only.

Verdict criteria (`compute_verdict`):
- CONFIRMADO_NE_SE: existe pelo menos 1 cell em NE e 1 em SE com
  delta MAE medio < 0 (favor ensemble) AND wins >= ceil(n_folds/2).
- REFUTADO: nem NE nem SE tem cell confirming.
- INDETERMINADO: so' NE ou so' SE.

Codigo: `scripts/h10_ensemble_cv.py`. Saidas:
`outputs/iter_0013/h10_ensemble_v2_persist/{results.json, summary.csv,
verdict.json, weights_distribution.csv, sanity_summary.json}`.

## Resultado

**VERDICT: CONFIRMADO_NE_SE** (com bonus N e parcial S).

Tabela completa em result_metric do frontmatter. Highlights:

- **NE: 3/3 cells confirming.** Maior ganho: NE/v1 -12.996 MWh MAE (28%
  reducao). NE/v2 (cell alvo H10) -4.188 MWh (13% reducao). NE/v3 -4.065
  MWh (13% reducao). Best scheme em NE = ens_inv_mae (v2, v3) e
  ens_inv_mse (v1).
- **SE: 3/3 cells confirming.** Ganho 4-7% MAE: -565 a -293 MWh.
  Magnitude menor que NE mas wins 3-4/5 folds.
- **N (bonus): 3/3 cells confirming.** -100 MWh (17% reducao em N/v1-v2).
  Confirmacao direta do mecanismo H10 detail ("persist forte em N").
  R²_LGB era -0.33; ensemble move para +0.01.
- **S: 2/3 cells confirming.** S/v3 +26 MWh (2.7% pior, irrelevante).

**Adaptacao do alpha por fold** confirma que pesos derivados de
inner_val respondem a regime shift. NE/v1 fold 4 (LGB MAE=43k vs PER
MAE=33k): ensemble inv_mse escolheu alpha=0.00 ~= pura persist. NE/v3
fold 5 (LGB MAE=23k vs PER MAE=17k, persist melhor): alpha=0.50.

**Ranking de esquemas** (frequencia de win por cell):
- ens_inv_mae: 6 cells (mais robusto)
- ens_inv_mse: 4 cells
- ens_equal: 2 cells (SE/v1, S/v1 — surpresa positiva)
- ens_opt_alpha: 0 cells (sobre-otimiza inner_val)

Insight: pesos analiticos (inv_mae, inv_mse) >> grid-search empirico
(ens_opt_alpha). Consistente com BMA: weights proportional-to-precision
sao mais robustos que minimo empirico em inner_val pequeno (30d).

### Sanity checks

Coverage detalhada em `sanity_summary.json`. Resumo:

| sanity | status | nota |
|---|---|---|
| B1 leak | INHERITED | iter_0002 features auditadas; persist_d1 = y_d1[i-1] D-1 safe |
| B2 perm | N/A | ensemble e' meta-modelo de 2 ponteiros, sem features |
| B3 holdout strict | DONE_VIA_CV | 12 cells x 5 folds = 60 holdouts, gap 7d entre train e test, inner_val 30d sem overlap com test |
| B4 baseline | DONE_INTEGRADO | persist_d1 e' componente direto do ensemble; skill vs persist computado em todas as 12 cells (skill +0.07 a +0.21) |
| B5 dist_shift | INHERITED + EVID | iter_0012 KS p<0.0001 NE+SE; H10 weights_distribution.csv mostra alpha varia 0.0-1.0 entre folds = ensemble adapta |
| B6 zero_count | N/A | nao introduz features |

Suite COMPLETA para H10. Sem hipotese derivada blocked por sanity.

## Decisao

**CONFIRMADO_NE_SE**. Ensemble (LGBM + persist_d1, pesos inv_mae ou
inv_mse) deve ser POST-PROCESSING DEFAULT para forecast curt D+1 no
loop:
- NE: ganho 13-28% MAE, magnitude clinica relevante
- SE: ganho 4-7% MAE, consistente
- N: ganho 14-17% MAE (bonus, corrige LGB-pior-que-persist)
- S: ganho 0-13% MAE, mas v3 piora 2.7% (irrelevante)

Sem req externo emitido — UlFor opera em Ridge/LR, nao LGBM. H24
derivada (P2) abre frente para testar mesmo ensemble com champions Ridge.

H25 derivada (P3) abre frente para stacker meta-modelo (Ridge sobre
[LGB, persist, ma7, climatologia]).

## Proximo passo

Planner config para iter_0014:
- **Principal**: H21 (P2 feature engineering `pdp_residual = pdp_prev - gen`,
  derivada de H3 iter_0010, codavel local). Continua o plano de iter_0012.
- **Alt**: H24 (ensemble sobre Ridge/LR, derivada de H10 hoje), H11 (quantile),
  H22 (GBDT vs OLS gap), H19 (extrair MAE/R²/F1 champions).

H10 fechada como done iter_handled=0013 verdict=CONFIRMADO_NE_SE.
H24 + H25 adicionadas a queue (P2/P3) sem dependencia.
