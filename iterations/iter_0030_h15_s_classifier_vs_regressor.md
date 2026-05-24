---
alvo: h15_s_classifier_vs_regressor
layer: curtailment
iter_num: 0030
type: hypothesis_test (verdict=CONFIRMADO_PARCIAL_NON_RARE)
data_utc: 2026-05-25T01:30:00Z
hypothesis: |
  H15 (P3, queued desde iter_0001 — last_updated iter_0006):
  "S 'nao aprendivel' — rare event classifier em vez de regressor?"
  Hipotese original (pre-PDP-fix): S/ML degradado por dominancia de zeros;
  classifier dedicado captura melhor sinal de alerta.
  Atualizacao iter_0006: parcialmente obsoleta (S/ML ja' bate baseline
  como regressor pos-PDP-fix req-0002). Mantida P3: pode ainda dar AUC >
  regressor para alerta operacional onde recall importa mais que MAE.

  Decomposicao em sub-claims testaveis:
    C1 (any-curt alert): classifier > regressor binarizado em threshold=0
        (pos_rate ~40%)
    C2 (moderate-curt alert): classifier > regressor binarizado em
        threshold P75_train (pos_rate ~25%)
    C3 (rare-event alert, claim ORIGINAL): classifier > regressor
        binarizado em threshold P90_train (pos_rate ~10%)
baseline_tipo: |
  persist_d1 binarizado (y_te[i-1] > thr) + climatology proba constante
  (= positive_rate(y_tr)). Ambos calculados per-fold sem leak.
baseline_metric: |
  Per threshold (mean across 5 folds):
    thr_zero  : persist_d1 AUC=0.617, climat AUC=0.500
    thr_p75   : persist_d1 AUC=0.633, climat AUC=0.500
    thr_p90   : persist_d1 AUC=0.551, climat AUC=0.500
result_metric: |
  Per threshold (best CLS vs best REG binarizado):
    thr_zero  : LogReg AUC=0.784  vs LR_reg AUC=0.739  (+4.5pp); PR-AUC +5.1pp
    thr_p75   : LogReg AUC=0.821  vs LR_reg AUC=0.754  (+6.8pp); PR-AUC +8.1pp
    thr_p90   : LogReg AUC=0.762  vs LR_reg AUC=0.778  (-1.6pp); PR-AUC -0.6pp

  Decision rule: |delta_AUC| > 0.02 AND |delta_PR-AUC| > 0.02.
    C1 CONFIRMADO (delta +0.045/+0.051)
    C2 CONFIRMADO (delta +0.068/+0.081)
    C3 REFUTADO   (delta -0.016/-0.006, dentro do ruido)
decision: |
  CONFIRMADO_PARCIAL_NON_RARE. C1+C2 confirmados; C3 (claim original
  rare-event) REFUTADO. Classifier dedicado serve para alerta moderado
  (any-curt ou big-curt) em S mas NAO substitui regressor binarizado para
  rare-event severo. Hipotese derivada H35 criada (S binary alert
  endpoint via LogReg).
sanity_checks_passed:
  permutation_importance: true
  holdout_temporal_strict: true
  leak_detection: true   # inherited iter_0002 PDP-fix req-0002
  baseline_compare: true
  distribution_shift: warn   # severo, documentado
  zero_count_shift: warn     # 3/37 feats curt_lag* shift>0.2 fold 5
budget_consumido_iter: 1.2
custo_estimado_usd: 0
---

# Iter 0030 — H15 S classifier vs regressor binarizado

## Hipotese

S e' a sub-regiao com menor sinal de curtailment (60% zeros, P50=0,
P90=2.5GWh). Hipotese original: regressor sofre degradacao por
dominancia de zeros, classifier dedicado captura melhor sinal de
alerta operacional onde recall importa mais que MAE.

Atualizacao iter_0006 (pos-PDP-fix req-0002): regressor S/ML ja' bate
baseline (109% < persist_d1 113.7%). Claim original "rare event
classifier > regressor" enfraquece mas mantem upside potencial em
operational alerting (AUC > MAE para decisao binaria).

Refinada em sub-claims testaveis (C1/C2/C3) por threshold.

## Como foi rodado

**Script**: `scripts/h15_s_classifier_vs_regressor.py`
**Sanity**: `scripts/h15_sanity_checks.py`
**Comando**:
```bash
cd loops/forecast-mega-loop
uv run python scripts/h15_s_classifier_vs_regressor.py
uv run python scripts/h15_sanity_checks.py
```

**Dataset**: `outputs/iter_0002/runs/S/v1/features.parquet` (37 feats,
n=464d, range 2024-12-15 → 2026-03-26). Identico ao iter_0002 S/v1 —
zero re-trabalho de feature engineering. PDP-fix (req-0002) inherited.

**CV protocolo** (espelhando H7/H10/H24 padrao loop):
- 5 folds walk-forward, test window = 60 dias contiguos
- gap = 7 dias entre train_cutoff e test_start (D+1 horizon safe)
- StandardScaler fit so' em train por fold (sem leak)
- Per fold treina TODOS os modelos do mesmo X_tr → predicoes em mesmo X_te

**Modelos** (4 + 2 baselines):
- Regressors (output continuo → threshold → bin):
  - LR_sklearn (champion oficial S)
  - Ridge α=10 (uniformidade com NE/N champion family)
- Classifiers (output proba positivo):
  - LogReg C=1.0, max_iter=1000, class_weight="balanced" (rare-event aware)
  - LGBMClassifier n_est=300, lr=0.05, num_leaves=31, objective="binary"
- Baselines:
  - persist_d1 binarizado: score = y_te[i-1] continuo, bin = score > thr
  - climat: proba constante = positive_rate(y_tr_bin)

**Thresholds** (derivados de y_tr per fold — sem leak):
- thr_zero = 0.0           (any curt; pos_rate train ~30-43%)
- thr_p75  = P75(y_tr)     (big curt; pos_rate test 5-65%, varia por fold)
- thr_p90  = P90(y_tr)     (rare event; pos_rate test 2-42%, varia por fold)

**Metricas**:
- AUC-ROC (Mann-Whitney U com tie-handling avg-rank)
- PR-AUC (average precision, monotone interpolation)
- F1, Precision, Recall, Specificity em threshold operacional
  (score > thr para regressors/persist; proba > 0.5 para classifiers/climat)
- Brier score para classifiers (calibracao)

## Resultado

### Per-threshold aggregate (mean across 5 folds)

| threshold | pos_rate_te | best CLS AUC | best REG AUC | persist AUC | ΔAUC CLS−REG | best CLS PR-AUC | best REG PR-AUC | ΔPR-AUC | C confirm? |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| thr_zero  (any curt)   | 0.40 | **0.784 (logreg)** | 0.739 (lr_reg)    | 0.617 | **+0.045** | **0.784** | 0.733 | **+0.051** | C1 YES |
| thr_p75   (big curt)   | 0.25 | **0.821 (logreg)** | 0.754 (lr_reg)    | 0.633 | **+0.068** | **0.716** | 0.635 | **+0.081** | C2 YES |
| thr_p90   (rare event) | 0.10 | 0.762 (logreg)     | **0.778 (lr_reg)** | 0.551 | −0.016     | 0.451     | **0.456** | −0.006 | C3 NO  |

### F1 / Precision / Recall (mean across folds)

| threshold | model | F1 | Precision | Recall |
|---|---|---:|---:|---:|
| thr_zero | logreg_cls | 0.718 | 0.707 | 0.737 |
| thr_zero | lgbm_cls   | 0.451 | 0.685 | 0.393 |
| thr_zero | lr_reg     | 0.682 | 0.559 | 0.900 |
| thr_zero | ridge_reg  | 0.675 | 0.539 | 0.926 |
| thr_zero | persist    | 0.603 | 0.605 | 0.601 |
| thr_p75  | logreg_cls | 0.625 | 0.588 | 0.681 |
| thr_p75  | lr_reg     | 0.612 | 0.490 | 0.911 |
| thr_p90  | logreg_cls | 0.311 | 0.530 | 0.397 |
| thr_p90  | lr_reg     | 0.330 | 0.409 | 0.381 |

**Observacao crucial**: regressor binarizado tem RECALL muito mais alto
(~0.90-0.93) mas PRECISION muito mais baixa (~0.54-0.56). LogReg
otimiza F1 melhor (0.72 vs 0.68) com balanço precision/recall mais util
operacionalmente (~0.71/0.74). Regressor classifica "qualquer
predicao > 0" como positivo, gerando muitos falsos positivos.

### Sanity checks (6 default)

| check | status | nota |
|---|---|---|
| leak | PASS_INHERITED | features identicas iter_0002 S/v1, PDP-fix req-0002 ja aplicado, sem nova superficie de leak |
| perm | PASS | real LogReg AUC=0.872 (fold 5, thr_zero) vs perm 0.512±0.13 (n=50, p=0.000); PR-AUC real 0.598 vs perm 0.220 (p=0.000) |
| holdout_strict | PASS | walk-forward 5×60d gap 7d, fold mais recente termina 2026-03-26, zero overlap |
| baseline | PASS | persist_d1 binarizado AUC 0.617 + climat AUC 0.500 incluidos; LogReg thr_zero +16.8pp vs persist |
| dist_shift | WARN | y_te zero_rate varia 0.30-0.83 entre folds vs y_tr 0.57-0.75; max shift 0.37 (fold 2); regime change real, explica volatilidade thr_p90 |
| zero_count | WARN | 3/37 features com |shift|>0.2 em fold 5: curt_lag1/lag7/lag14 (auto-correlacao do target zero-dominado); herdado iter_0002 features, ja revisado |

### Verdict por sub-claim

- **C1 (any-curt alert)**: CONFIRMADO. LogReg bate LR_reg por +4.5pp AUC,
  +5.1pp PR-AUC. F1 melhor (0.718 vs 0.682) com balanço precision/recall
  mais util. Persist binarizado 16.8pp atras.
- **C2 (moderate-curt alert)**: CONFIRMADO. Maior margem absoluta (+6.8pp
  AUC, +8.1pp PR-AUC). LogReg AUC=0.821 e' o pico geral do estudo.
- **C3 (rare-event alert, claim ORIGINAL H15)**: REFUTADO. ΔAUC −0.016,
  ΔPR-AUC −0.006 dentro do ruido. Mecanismo: pos absoluto baixo no test
  (fold 5 thr_p90 = 1 positivo em 60d) inviabiliza calibracao LogReg;
  regressor binarizado preserva ordering por magnitude continua que
  beneficia de informacao sub-threshold.

## Decisao

**RETÉM como diagnostico + cria H35 (P3) para producao**.

Hipotese original (rare-event classifier > regressor) **REFUTADA**.
Hipotese refinada (classifier > regressor para alerta moderado em S)
**CONFIRMADA**.

Nao ha' acao imediata em producao porque:
1. Champion S oficial e' LR_sklearn regressor — nao ha' endpoint binario
   no /api/forecast/d1.
2. Alerta binario nao foi requisitado por Breno (prioridades 2026-05
   sao `usinas` > `bigsin` > `curtometro` continuo).
3. Custo: novo endpoint + dashboard wiring + monitoramento.

H35 captura a oportunidade no backlog. Implementacao codavel pelo UlFor
se/quando algum produto pedir alerta binario "vai ter curtailment em S
amanha?". Plano em H35 detalha protocolo.

**Nenhum req-NNNN emitido ao UlFor** — descoberta nao bloqueia nada
ativo e nao requer dados frescos.

## Proximo passo

Loop continua self-planning. Proxima iter pode atacar:
- H25/H26/H27/H28: hipoteses queued nao ainda processadas
- H30: outra queued
- H33: derivada iter_0027 (H8 single-PI vs joint-drop methodology)
- recon UlFor se houver novos commits desde `27152e16` (iter_0028 HEAD)

H35 fica em backlog P3 ate' alguem pedir alerta binario S em algum
produto.

## Artefatos produzidos

- `outputs/iter_0030/h15_s_classifier_vs_regressor/results.json` — full
  per-fold per-threshold per-model metrics
- `outputs/iter_0030/h15_s_classifier_vs_regressor/summary.csv` — long
  format aggregated (4 models × 6 models × 5 metrics)
- `outputs/iter_0030/h15_s_classifier_vs_regressor/verdict.json` — verdict
  rule + per-threshold confirms
- `outputs/iter_0030/h15_s_classifier_vs_regressor/sanity_checks.json` —
  6 checks status + rationale
- `scripts/h15_s_classifier_vs_regressor.py` — CV runner (300 lines)
- `scripts/h15_sanity_checks.py` — sanity check runner (280 lines)
- `leaderboard.md` — nova secao "S binary alert (iter_0030 H15)" +
  timeline iter 0030 + header atualizado
- `hypotheses_queue.md` — H15 status=done + verdict + H35 derivada queued

## Lessons learned (transferiveis)

1. **Threshold escolhe o modelo, nao o contrario**: em S, classifier
   ganha em pos_rate moderado (10-40%); regressor ganha em rare event
   (<10% test absoluto). Decisao de "classifier vs regressor" e'
   threshold-dependent, nao universal.
2. **class_weight=balanced e' essencial para sub low-signal**: LogReg
   sem balanced colapsa em baseline (todos predizem 0). Com balanced,
   AUC 0.78 em thr_zero. Detalhe critico para qualquer rare-event
   classifier no loop.
3. **Permutation test on labels classify-side**: rodando 50 perms do
   y_tr_bin com retreino, AUC real=0.872 vs perm 0.512±0.13. p=0.000.
   Padrao reutilizavel para qualquer hypothesis_test classificador.
4. **Sub S regime shift e' estrutural**: distribution shift de zero_rate
   varia 0.30-0.83 entre folds — qualquer modelo S sofre dessa
   instabilidade. Lesson: reportar shift WARN ao inves de tentar
   "consertar" com features adicionais (nao havera' fix sem novo regime
   data, e shift e' real e nao um bug).
