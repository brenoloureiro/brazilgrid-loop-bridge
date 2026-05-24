---
alvo: h22_gbdt_vs_ols_gap
layer: curtailment
iter_num: 0034
type: hypothesis_test (verdict=REFUTADO_NO_NONLINEAR_GAIN)
data_utc: 2026-05-25T05:30:00Z
hypothesis: |
  H22 (P3, queued desde iter_0010 — derivada H3 follow_up):
  "GBDT-only curt~gen+pdp para medir gap nao-linear vs OLS"
  Premissa: H3 (iter_0010) usou OLS linear; GBDT (XGB/LGBM) provavelmente
  extrai sinal adicional via interacoes (pdp x gen, sazonalidade x pdp,
  regime x pdp). Treinar GBDT com apenas `gen + pdp_prev_eolica +
  pdp_prev_solar` (3 features) e comparar R² vs OLS no mesmo split temporal.
  Se gap GBDT vs OLS >= 5pp R², CONFIRMA interacoes nao-lineares
  relevantes -> mantem pdp como features brutas (nao agregadas via
  engineering linear como em H21).

  Atualizacoes iter_0021 / iter_0023 reforcaram protocolo: medir PI duo
  (OLS + GBDT) por reproducao empirica do lesson model-aware H22_MA UlFor
  (~10pp magnitude). Esta iter implementou PI duo via refit-drop test
  (mais autoritativo que perm sob colinearidade — H8 lesson).
baseline_tipo: |
  3 baselines:
    a) OLS gen-only (1 feat) — sanity: pdp adiciona sinal?
    b) persist_d1 = curt[D-1] -> curt[D] — sanity: 3-feat bate persist?
    c) (implicito) OLS 3-feat — o sucessor que GBDT precisa bater por
       >=5pp R^2 para confirmar.
baseline_metric: |
  Per sub (R^2 test):
    NE: persist=+0.373, OLS_gen=+0.292, OLS_3feat=+0.550, GBDT_3feat=+0.507
    SE: persist=-0.367, OLS_gen=-0.162, OLS_3feat=+0.072, GBDT_3feat=+0.088
    S:  persist=-0.388, OLS_gen=-0.159, OLS_3feat=-0.252, GBDT_3feat=-0.426
result_metric: |
  Gap R^2 test (GBDT − OLS):
    NE: −0.043 (GBDT_WORSE; OLS overfit menos sob shift NE 2025Q4/2026Q1)
    SE: +0.015 (TIE; ambos R^2 muito baixos ~0.07-0.09 em SE)
    S:  −0.174 (GBDT_WORSE; GBDT overfit catastrofico em S, train R^2 0.81
                vs test -0.43)
  max_gap = +0.015 (SE) < +0.05 threshold em todos os subs.
  mean_gap = -0.067 (nao-positivo).
  GBDT train-test delta R^2: NE -0.48 / SE -0.87 / S -1.24 (overfit severo).

  PI duo refit-drop gap_per_feat |OLS - GBDT| chega a ±85pp (NE pdp_eolica
  GBDT 128pp vs OLS 43pp). Reproduz lesson model-aware EMPIRICAMENTE.
decision: |
  REFUTADO_NO_NONLINEAR_GAIN. Premissa central H22 nao se sustenta com
  3 features (gen + pdp_prev_eolica + pdp_prev_solar). GBDT NAO supera OLS
  em nenhum sub no R^2 test; OLS vence ou empata. Engineering linear
  (i.e. somar gen+pdp_prev como combinacao linear) e' suficiente para o
  sinal disponivel no espaco de 3 features. H22 NAO reverte H21
  (que rejeitou agregacao do residual): a decisao continua "manter
  pdp_prev_eolica e pdp_prev_solar como features brutas, modelo linear
  Ridge/LR continua otimo (per UlFor iter_0007)".

  H36 derivada (P3, queued): testar mesma comparacao com features
  completas iter_0002 v3 (37+ feats). Com 3 feats o teste e' limitado
  em capacidade — com 37 feats as interacoes nao-lineares teriam mais
  espaco para emergir. Custo ~30 LoC sobre h22_gbdt_vs_ols.py.
sanity_checks_passed:
  leak_detection: true            # corr_pdp_t/curt_t > corr_pdp_t/curt_t-1; heredado H3
  permutation_importance: true    # GBDT perm PI > 0 com magnitude clara (NE eol 0.62, SE sol 0.30)
  holdout_temporal_strict: true   # 80/20 temporal, sem leak (features mesmo dia)
  baseline_compare: true          # OLS_gen_only + persist_d1 baselines reportados
  distribution_shift: reported    # delta R^2 train-test severo confirma B5 iter_0012
  zero_count_shift: reported      # NE 2->0%, SE 15->9%, S 35->52% shift documentado
budget_consumido_iter: 0.8
custo_estimado_usd: 0
ulfor_head_no_inicio: 80620230
novos_requests: 0
follow_ups_created: [H36]
---

# Iter 0034 — H22 GBDT vs OLS gap em curt ~ gen + pdp_prev

## Hipotese

H22 (queue) postulou que GBDT (LightGBM defaults) extrairia sinal
nao-linear adicional sobre OLS no mesmo dataset minimal de H3 (3 features:
gen_renov + pdp_prev_eolica + pdp_prev_solar). Threshold de confirmacao:
gap R^2 test >= 5pp em algum sub.

Atualizacao iter_0021/0023 reforcou: o protocolo deveria incluir PI duo
(medir PI em OLS E em GBDT), porque H22_MA UlFor (commit `2daa5d40`)
mostrou empiricamente que PI eh model-dependent com magnitude ~10pp em
SE/lr vs Ridge.

## Como foi rodado

**Script**: `scripts/h22_gbdt_vs_ols.py`
**Comando**:
```bash
cd loops/forecast-mega-loop
uv run python scripts/h22_gbdt_vs_ols.py
```

**Dataset**: `outputs/iter_0010/h3_pdp_residual_signal/h3_join.parquet`
(heredado de H3, 1593 rows, 3 subs NE/SE/S, range 2024-12-01..2026-05-15).
Filtro identico H3 nz: `gen_renov > 0 AND pdp_prev_total > 0` → 1.458
dias uteis.

**Split** (espelhando H3 dist_shift_split):
- Temporal 80/20 chronological
- train: 2024-12-01..2025-12-25 (n=388 per sub)
- test: 2025-12-26..2026-05-01 (n=98 per sub)
- gap=0 (features mesmo dia D; nao ha exogenos com lag)

**Modelos** (2 + 3 baselines):
- OLS: `numpy.linalg.lstsq` com intercept (sem regularizacao, 3 feats bem
  condicionados — col_num pequeno)
- GBDT: LightGBM defaults H10 iter_0013 (n_est=300, lr=0.05, num_leaves=31,
  min_child=10, subsample=0.8, colsample=0.9, seed=13)
- Baseline V0: OLS gen-only (1 feat) → mede contribuicao incremental do pdp
- Baseline persist_d1: y_te[i-1] → y_te[i] (sem leak por construcao)
- Baseline GBDT_gen_only: GBDT 1 feat → mede capacidade do GBDT sem pdp

**PI duo** (refit-drop test, mais autoritativo que perm sob colinearidade
— H8/H22_MA lesson):
- Para cada feature, refita modelo sem essa feature, mede R^2_test_drop.
- `abs_drop_pp = |R^2_full - R^2_drop| × 100`
- Reporta per (sub, feature, modelo) e gap `gbdt_pp - ols_pp`

**Permutation PI** (sanity, n_perm=30 GBDT pdp_eolica/pdp_solar):
- Shuffle X_test[:, feat], recompute R^2 → mean_drop_r2 do GBDT.

## Numeros entregues

### Tabela principal (R^2 test)

| Sub | n_test | OLS R²_test | GBDT R²_test | **Gap (pp)** | Verdict local |
|---|--:|--:|--:|--:|---|
| NE | 98 | +0.550 | +0.507 | **−4.3** | GBDT_WORSE |
| SE | 98 | +0.072 | +0.088 | **+1.5** | TIE |
| S  | 98 | −0.252 | −0.426 | **−17.4** | GBDT_WORSE |

- max_gap = +0.015 (SE) → abaixo do threshold +5pp em todos.
- mean_gap = −0.067 (nao-positivo).

### Overfit GBDT (train-test delta R^2)

| Sub | R²_train OLS | R²_train GBDT | R²_test OLS | R²_test GBDT | Δ_test−train GBDT |
|---|--:|--:|--:|--:|--:|
| NE | 0.703 | 0.989 | 0.550 | 0.507 | **−0.482** |
| SE | 0.538 | 0.959 | 0.072 | 0.088 | **−0.871** |
| S  | 0.240 | 0.814 | −0.252 | −0.426 | **−1.240** |

GBDT memoriza train (R^2 0.81-0.99), test colapsa proporcional a magnitude
do distribution shift documentado em B5 iter_0012 (KS p<1e-4 NE+SE).

### PI duo refit-drop (lesson H22_MA empirico)

| Sub | Feature | OLS abs_drop pp | GBDT abs_drop pp | Gap pp |
|---|---|--:|--:|--:|
| NE | gen_renov           | 14.8 | 0.7   | −14.1 |
| NE | pdp_prev_eolica     | 43.2 | 128.5 | **+85.3** |
| NE | pdp_prev_solar      | 37.1 | 24.2  | −12.9 |
| SE | gen_renov           | 21.5 | 64.6  | +43.1 |
| SE | pdp_prev_eolica     | 21.7 | 1.6   | **−20.1** |
| SE | pdp_prev_solar      | 13.3 | 39.8  | +26.5 |
| S  | gen_renov           |  6.1 | 29.1  | +23.0 |
| S  | pdp_prev_eolica     |  9.2 | 51.0  | +41.7 |
| S  | pdp_prev_solar      |  0.0 | 0.0   |  0.0  |

PI per-feat diverge entre OLS e GBDT em ate ±85pp (NE pdp_eolica). Reproduz
empiricamente o lesson model-aware H22_MA UlFor (magnitude maior que os
~10pp do lr vs Ridge em SE/v3 features completas, porque feature space
aqui e' pequeno e GBDT depende heavily da unica feature com signal
nao-linear: NE pdp_eolica).

**Conclusao da PI duo**: PI EH model-dependent. Mas isso NAO se traduz
em melhor R^2 test do GBDT. GBDT usa as features (perm PI > 0) — apenas
ineficientemente.

### Permutation PI (sanity GBDT)

| Sub | Feature | mean drop R^2 | std drop R^2 |
|---|---|--:|--:|
| NE | pdp_prev_eolica | 0.618 | 0.125 |
| NE | pdp_prev_solar  | 0.216 | 0.064 |
| SE | pdp_prev_eolica | 0.260 | 0.095 |
| SE | pdp_prev_solar  | 0.301 | 0.138 |

GBDT esta usando as features pdp. Importance forte e estavel.

## Sanity checks (6 default)

| Check | Status | Detalhe |
|---|---|---|
| leak_detection | PASS | corr_pdp_t/curt_t > corr_pdp_t/curt_t−1 mantido (heredado H3) |
| permutation_importance | PASS | GBDT perm PI pdp eol/sol > 0 em NE+SE, magnitude clara |
| holdout_temporal_strict | PASS | 80/20 temporal, sem leak; gap=0 OK pois features mesmo dia |
| baseline_compare | PASS | OLS_gen_only + persist_d1 reportados; OLS_3feat bate ambos em NE+SE |
| distribution_shift | REPORTED | delta R² test−train severo (GBDT NE −0.48 / SE −0.87 / S −1.24) |
| zero_count_shift | REPORTED | NE 2%/0%, SE 15%/9%, S 35%/52% (S notavelmente maior em test) |

Sanity_required_pela_queue (`[holdout, baseline]`): AMBOS PASS.

## Decisao final

**REFUTADO_NO_NONLINEAR_GAIN**

Justificativa formal:
- max_gap_R^2_test = +0.015 (SE) < +0.05 threshold em **todos** os 3 subs.
- mean_gap = −0.067 (nao-positivo).
- 2/3 subs apresentam GBDT pior por mais de 4pp (NE −4.3pp, S −17.4pp).
- 1/3 sub (SE) tie dentro da margem (+1.5pp, abaixo do threshold).
- Mecanismo: GBDT overfit severo (train 0.81-0.99 → test 0.07-0.55) sob
  distribution shift NE+SE (B5 iter_0012 KS<1e-4 + S shift de zeros 35→52%).
- OLS sem regularizacao com 3 features bem condicionadas (col_num pequeno)
  NAO tem capacidade para overfit; em distribution shift, "menos
  capacidade" venceu "mais capacidade".

**Implicacao para decisoes upstream**:
- H21 (REFUTADO_engineering_linear): mantida — agregar pdp_eol+pdp_sol via residual destruia sinal; H22 confirma a outra direcao (trocar OLS por GBDT tambem nao adiciona).
- Decisao UlFor iter_0007 (champions Ridge/LR > XGB/LGBM em curt D+1): CORROBORADA pelo replay loop. H22 empirico em pequeno feat-space converge com bake-off completo UlFor em feature_set full (55 feats).
- pdp_prev_eolica e pdp_prev_solar continuam features brutas no modelo (UlFor ja' faz isso).

## Follow-up

**H36 derivada** (P3, queued, ~1h):
- Mesma comparacao GBDT vs OLS, mas com features completas iter_0002 v3 (37+ feats).
- Hipotese: com mais features, GBDT teria espaco para capturar interacoes que nao caberiam em 3D.
- Se H36 tambem REFUTAR, GBDT nao tem upside em curt D+1 vs OLS — explica empiricamente porque UlFor champions sao Ridge/LR.
- Custo: ~30 LoC sobre `scripts/h22_gbdt_vs_ols.py` (apenas trocar source parquet + lista de features).

## Artefatos

```
outputs/iter_0034/h22_gbdt_vs_ols_gap/
├── results.json          # full per-sub metrics, PI duo, perm PI, sanity
├── summary.csv           # 3 rows NE/SE/S, R^2 + gap + verdict_local
├── sanity_checks.json    # 6 default checks status
└── verdict.md            # detalhado por sub + lessons
```

**Script**: `scripts/h22_gbdt_vs_ols.py`

## Lessons learned

1. **OLS 3 feats bem-escolhidas > GBDT defaults sob distribution shift**. Em curt D+1 com gen + 2 pdp features, modelo linear vence. Converge com lesson iter_0007 UlFor (Ridge/LR > GBDT no full feature_set).

2. **PI duo OLS vs GBDT diverge em ±85pp empiricamente**. Magnitude maior que os ~10pp do lesson H22_MA UlFor porque feature space e' pequeno aqui. Confirma necessidade de PI-with-final-model como protocolo.

3. **GBDT capacidade > generalizacao em distribuicao nao-estacionaria**. Train R^2 0.99 NE / 0.96 SE / 0.81 S; test colapsa proporcional a magnitude do shift. Classic bias-variance: OLS (bias alto) > GBDT (variance alto) sob shift severo.

4. **Refit-drop > perm single-feat sob colinearidade leve**. Mesmo em apenas 3 feats com colinearidade fraca, refit-drop deu sinal limpo e e' mais barato (~3 fits vs 30 perm shuffles per feat).

5. **H21 + H22 juntos esgotam o espaco de "transformar (gen, pdp_prev)"**: H21 disse que combinacao linear (residual) nao ajuda; H22 disse que GBDT nonlinear-combos com mesmas 3 vars tambem nao ajudam. Conclusao operacional: novo sinal em curt D+1 EXIGE novas variaveis (ex: features iter_0002 cmo_*, carga_*, ter_verif_*) — H36 testa isso.
