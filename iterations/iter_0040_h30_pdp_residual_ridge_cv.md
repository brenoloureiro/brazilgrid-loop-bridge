---
alvo: pdp_residual_in_ridge_cv
layer: curtailment
iter_num: 0040
type: hypothesis_test (verdict=REFUTADO_RIDGE)
data_utc: 2026-05-25T20:00:00Z
hypothesis: |
  H30 (P3, queued iter_0024 — derivada de H21 iter_0020 REFUTADO em
  LGBM + corroboracao analitica OLS): "Ridge (champion UlFor NE/N) comporta-se
  diferente de LGBM/OLS quanto a engineering de pdp_residual? Shrinkage L2
  redistribui pesos entre features colineares; com basis transformado
  (residual centrado em zero), pode preservar mais sinal."

  Per notes_iter0026 do queue: rodar AMBOS alpha=1 (vencedor h22_MA alpha
  sweep UlFor commits 42dc0d7a/a3c742a9) e alpha=10 (default H30) no mesmo
  loop de CV.

  Feature sets minimais (DIFERENTE de H21 que usou v3 full ~47 feat):
    A = V_brutos          [ger_renovavel_mwh, pdp_prev_eolica, pdp_prev_solar]    (3 feat)
    B = V_residual_plus_gen [ger_renovavel_mwh, pdp_residual_total]               (2 feat)
    C = V_residual_split   [ger_renovavel_mwh, pdp_residual_eolica, pdp_residual_solar] (3 feat)

  Cells: NE/v3 e SE/v3 (sem S — H3 fragil, sem N — cobertura PDP zero).
baseline_tipo: |
  A = V_brutos (3-feat) Ridge alpha=1 e alpha=10, CV walk-forward 5x60d
  gap=7d. Tambem reportado: persist_d1 (sanity floor) por fold.
baseline_metric: |
  Per-cell mae_mean (Ridge baseline V_brutos):
    NE/v3 alpha=1:  mae=38511  r2=-0.044  nmae=0.457  f1=0.754  (vs persist 33837 = -13.8% skill)
    NE/v3 alpha=10: mae=38416  r2=-0.033  nmae=0.452  f1=0.757  (vs persist = -13.5% skill)
    SE/v3 alpha=1:  mae= 7726  r2=-0.049  nmae=0.604  f1=0.734  (vs persist 8328 = +7.2% skill)
    SE/v3 alpha=10: mae= 7703  r2=-0.047  nmae=0.602  f1=0.733  (vs persist = +7.5% skill)
  (R² negativo em todas: Ridge minimal 3-feat NAO supera mean-predict;
   ainda assim B nao melhora, e essa e a pergunta de H30.)
result_metric: |
  Paired delta_R² (B-A, C-A) per cell+alpha vs threshold H30 (CONFIRMADO >= -0.005, REFUTADO < -0.01):

    sub alpha=1:    delta_R²_B   delta_R²_C   delta_MAE_B%   delta_MAE_C%
      NE/v3        +0.013        +0.004       -0.6%          +0.9%
      SE/v3        -0.032        -0.818       +1.0%          +27.5%

    sub alpha=10:  delta_R²_B   delta_R²_C   delta_MAE_B%   delta_MAE_C%
      NE/v3        -0.064        -0.062       +3.2%          +4.9%
      SE/v3        -0.043        -0.469       +1.5%          +19.4%

  Apenas 1/4 (NE/v3 alpha=1, B) passa CONFIRMADO_RIDGE (+0.013 >= -0.005).
  Restantes 3/4 violam REFUTADO_RIDGE threshold (-0.032, -0.043, -0.064 todos < -0.01).
  C (residual_split) DESASTROSO em SE (delta_R² -0.469 a -0.818).
decision: DESCARTA — REFUTADO_RIDGE consistente; encerra H3-family residual no replay loop
sanity_checks_passed:
  permutation_importance: true   # B2 perm: residual_total CARREGA sinal real (NE +105%, SE +27% MAE-drop quando shuffle), mas e duplicata do span ja contido em A
  holdout_temporal_strict: true  # embedded — CV walk-forward 5 folds, gap 7d obrigatorio entre train e test
  leak_detection: true           # residual_total/eolica/solar sem NaN/inf; frac_negative NE 97.5% (PDP < ger esperado) SE 18.8% (PDP > ger maior parte do tempo SE)
  baseline_compare: true         # persist_d1 reportado por fold; NE Ridge base ABAIXO persist (-13.5%) revela modelo minimal insuficiente; SE Ridge ACIMA persist (+7.5%)
  distribution_shift: annotated_reuse  # KS p<0.0001 NE+SE ja confirmado iter_0012 H7; estatica estrutural, nao re-rodada
  zero_count: true               # NE 8/440 zero (1.8%), SE 57/442 zero (12.9%); ambos abaixo de risco "all zeros" — fold ok
  n_test_threshold: true         # 5 folds x 60d = 300 obs per cell, >> 30 threshold H20 em todos cells
  perm_importance_residual: true # confirmado nas folds finais B (NE alpha10 +105%, SE alpha10 +27%)
budget_consumido_iter: 0.4
custo_estimado_usd: 0.0
artefatos:
  - outputs/iter_0040/h30_pdp_residual_ridge_cv/verdict.json
  - outputs/iter_0040/h30_pdp_residual_ridge_cv/results.json
  - outputs/iter_0040/h30_pdp_residual_ridge_cv/sanity_summary.json
  - outputs/iter_0040/h30_pdp_residual_ridge_cv/summary.csv
  - scripts/h30_pdp_residual_ridge_cv.py
follow_ups_created:
  - H38 (curtailment, P4): "Alpha-sensitivity Ridge minimal-feat — NE/v3 alpha=1 B bateu A
    (+0.013 R²) mas alpha=10 perdeu (-0.064). Em-set minimal 2-3 feat, Ridge shrinkage
    excessivo (alpha grande) destroi sinal residual ja escasso. Hipotese: alpha optimo
    para basis-residual em set minimal e' alpha<1 ou Ridge-CV. Custo baixo (mesmo loop CV)."
  - H39 (governance, P5): "Documentar achado 'B2 perm confirms signal but H21+H30 reject':
    importance metrica nao implica feature engineering melhora. Adicionar ao playbook
    sanity_checks/B2_interpretation.md com NE/alpha10 case (perm +105% mas mean R² -0.064)
    como exemplo cannonico. Custo minimo (doc-only)."
encerra_caminhos:
  - "H3-family residual": OLS (H21 analitico) + LGBM (H21 empirico) + Ridge (H30 empirico)
    convergem em REFUTADO. Nenhum modelo linear/L2/GBDT extrai vantagem do basis residual
    explicito sobre o baseline raw (gen + pdp_prev_*). Fechado para o replay loop.
verdict_full: |
  REFUTADO_RIDGE per criterio H30 a priori:
    "REFUTADO_RIDGE se delta_R²(B-A) < -0.01 em qualquer NE/SE em AMBOS alphas"
  alpha=1:  NE B +0.013 (PASSA), SE B -0.032 (FAIL: < -0.01)
  alpha=10: NE B -0.064 (FAIL), SE B -0.043 (FAIL)
  -> AMBOS alphas violam em SE. NE alpha=1 nao salva (criterio AND, nao OR).
  C (residual_split) catastrofico em SE (delta_R² -0.47 a -0.82) em ambos alphas.

  Convergencia com H21 OLS+LGBM (mesmo verdict REFUTADO) FECHA H3-family residual
  no replay loop como avenida de feature engineering. Mecanismo unificado:
  span linear de (gen, pdp_prev_e, pdp_prev_s) ja contem qualquer combinacao
  (gen, residual_*) — transformacao explicita reduz dim sem expandir basis. Shrinkage
  L2 nao salva (mesmo padrao em alpha=1 e alpha=10).

  H22 (GBDT vs OLS para mecanismo nao-linear) segue como ultima frente residual no
  queue; se tambem refutar, encerra residual completamente.
---

# Iter 0040 — H30 pdp_residual em Ridge_alpha10/alpha1 CV

## Hipotese

Replicar a investigacao de H21 (REFUTADO em OLS analitico + LGBM empirico) com
Ridge — champion atual da UlFor em NE/N — para validar se a regularizacao L2
muda o verdicto. Mecanismo conjecturado: shrinkage L2 pode redistribuir pesos
entre features colineares de tal modo que basis residual (centrado em zero)
preserve mais sinal. Per notes_iter0026 do queue, testar alpha=1 (vencedor
h22_MA alpha sweep UlFor) e alpha=10 (default H30) simultaneamente.

## Como foi rodado

`scripts/h30_pdp_residual_ridge_cv.py`. CV walk-forward 5 folds (60d cada,
gap 7d), identico H7/H10/H11/H21. Modelo: `sklearn.Ridge(alpha)` com
StandardScaler embedded (features tem escalas O(10^4) MWh muito diferentes).
Cells: NE/v3 (n=440) e SE/v3 (n=442); features.parquet de iter_0002. 3
feature sets minimais por cell:

| set | features | dim |
|---|---|---|
| A | `ger_renovavel_mwh`, `pdp_prev_eolica_mwh`, `pdp_prev_solar_mwh` | 3 |
| B | `ger_renovavel_mwh`, `pdp_residual_total_mwh` | 2 |
| C | `ger_renovavel_mwh`, `pdp_residual_eolica_mwh`, `pdp_residual_solar_mwh` | 3 |

Onde `pdp_residual_* = pdp_prev_* - ger_*`. Cada cell rodada 2x (alpha=1, 10).
Total 12 (sub x alpha x fs) bake-offs em 5 folds = 60 fits. Wall-clock ~12s.

Sanity checks aplicados: leak diagnostic (B1), perm test em residual feature
nas folds finais B/C (B2, 30 perms), holdout embedded via walk-forward gap 7d
(B3), persist_d1 comparison por fold (B4), KS dist-shift annotated_reuse
iter_0012 (B5), n_test 300 obs/cell >> 30 H20 threshold (B6), zero_count
diagnostico do target.

## Resultado

**Verdict: REFUTADO_RIDGE.**

Paired delta R² (B-A) e (C-A) per cell+alpha:

```
sub        alpha=1                          alpha=10
       dR²_B    dR²_C    dMAE_B%      dR²_B    dR²_C    dMAE_B%
NE/v3  +0.013   +0.004   -0.6%        -0.064   -0.062   +3.2%
SE/v3  -0.032   -0.818   +1.0%        -0.043   -0.469   +1.5%
```

Threshold H30: CONFIRMADO se delta_R²_B >= -0.005 em AMBOS NE+SE; REFUTADO
se delta_R²_B < -0.01 em qualquer cell em AMBOS alphas. Resultado: 3/4 das
combinacoes (sub x alpha) violam threshold REFUTADO. Apenas NE alpha=1 B bate
A (+0.013), mas SE quebra em ambos alphas. **Criterio CONFIRMADO_RIDGE pede AND
(NE AND SE)** — falha.

**C (residual_split) catastrofico em SE** (delta_R² -0.469 a -0.818, MAE
+19-28%): split residual por fonte (eolica+solar separados) introduz
multicolinearidade severa que Ridge minimal nao resolve mesmo com alpha=10.

**Sanity B2 perm carrega contradica aparente**: residual_total nas folds finais
B mostra importance massiva (NE alpha=10 +105% MAE drop com shuffle, SE
alpha=10 +27%) — sinal e REAL. Mas mean CV ainda perde. Lesson:
*permutation importance confirma signal, nao confirma feature engineering ganho.*
O sinal e' **duplicata** do que (pdp_prev_eolica + pdp_prev_solar) ja contem
em A; transformar para residual nao adiciona basis, apenas reduz dim.

**Sanity B4 baseline reveal**: Ridge minimal 3-feat NE perde para
persist_d1 (-13.5% MAE worse), enquanto SE Ridge ganha (+7.5%). Sugere que
em NE o set 3-feat e' insuficiente — modelo full v3 (47 feat) seria
necessario para superar persist. Mas isso e' tangente ao verdict H30: a
pergunta era *relativa entre A/B/C*, nao absoluta vs persist.

Convergencia H21 (OLS analitico + LGBM empirico) + H30 (Ridge alpha=1, alpha=10)
== mesmo verdict REFUTADO. Mecanismo unificado: span linear de
(gen, pdp_prev_e, pdp_prev_s) ja contem (gen, residual_total) e
(gen, residual_e, residual_s) como combinacoes lineares. Engineering
explicito reduz dim sem expandir basis. Shrinkage L2 nao salva o que
algebra linear ja dizia.

## Decisao

**DESCARTA** — H30 REFUTADO confirma e estende H21. **Encerra H3-family
residual no replay loop**. Nao usar pdp_residual_* como feature engineering
em nenhum dos 3 modelos testados (OLS, LGBM, Ridge). Manter pdp_prev_* brutos
no baseline (decisao operacional ja vigente).

NAO atualizar champion (NE/N permanecem Ridge_alpha10 do UlFor com feature_set
full 55 feat — H30 nao tocou nessa avaliacao). NAO criar req externa (zero dep
UlFor, conforme spec H30).

## Proximo passo

1. **H22 (GBDT vs OLS) segue como ultima frente residual** (P3 no queue desde
   iter_0024). Mecanismo nao-linear (LGBM/XGBoost interaction terms) e' a
   ultima janela teorica para residual destravar ganho. Se H22 tambem refutar,
   fecha residual completo (linear + L2 + GBDT cobertos).
2. **H38 (P4) criada como follow-up**: Ridge alpha-sensitivity em set minimal.
   NE/v3 alpha=1 marginalmente confirmou (delta_R²_B +0.013), alpha=10 destruiu
   (-0.064). Investigar se alpha<1 (Ridge-CV) muda o veredito apenas para
   in-sample minimal. **Baixa prioridade** — mesmo se confirmar, ganho marginal
   em 1 sub nao justifica swap de feature engineering operacional.
3. **H39 (P5, doc-only) criada**: documentar achado "perm_importance confirms
   signal mas H21+H30 reject feature_engineering" como entry canonica em
   `sanity_checks/B2_interpretation.md`. NE alpha=10 case (perm +105% mas mean
   R² -0.064) e' o exemplo perfeito de "signal != marginal_gain".

## Observacoes laterais

- **SE PDP > ger frac 81%**: notavel que em SE 81% dos dias `pdp_prev_total >
  ger_total`, oposto de NE (97% PDP < ger). Estrutura de cobertura PDP-vs-ger
  diferente entre subs explica por que C (split) explode em SE — multicolinear
  com sinais opostos por fonte. **Possivel H derivada futura**: por que PDP_SE
  super-estima ger consistentemente vs PDP_NE sub-estima?
- **Ridge minimal 3-feat NE abaixo de persist_d1** sugere que o problema em NE
  D+1 nao e' "encontrar a feature certa" mas "ter features suficientes". Champion
  UlFor opera com 55 feat por razao — minimal sets sao educacionais mas nao
  produzem D+1 production-grade. Ja era assumido; sanity B4 confirma.
- **Convergencia metodologica**: H30 conclusao identica a H21 via 3 mecanismos
  ortogonais (algebra OLS + LGBM bake-off + Ridge bake-off). Robusto.
