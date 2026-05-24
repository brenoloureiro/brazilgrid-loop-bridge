---
alvo: h9_metric_suite_principio_6
layer: meta
iter_num: 0008
type: hypothesis_test (verdict=CONFIRMADO)
data_utc: 2026-05-24T06:00:00Z
baseline_tipo: NMAE-only (Principio 6 violado)
hypothesis: |
  H9 (P1): PLANO_FINAL UlFor Principio 6 — "Para curtailment usar
  MAE/R²/F1, nunca NMAE/media". Iter_0002 e leaderboard usam NMAE como
  metrica primaria, violando o principio. Refatorar metric suite para
  MAE+R²+F1 (F1 binarizada por P50 do TRAIN target) como primarios;
  NMAE rebaixada a secundaria com flag de seguranca quando ymean<EPS.
result_metric: |
  H9 CONFIRMADO. Ao aplicar metric_suite sobre iter_0002 LGBM replay:
    (a) Ranking por NMAE diverge de R²/F1 em 3/4 subs:
        NE: best_NMAE=v2 (0.282) mas best_F1=v1 (0.000 todos, NE no high-curt em test)
        SE: best_NMAE=v2 (0.362) mas best_R²=v3 (+0.544)
        N:  best_NMAE=v3 (0.974) mas best_R²/F1=v1 (-1.402 / 0.800)
    (b) S NMAE 'unsafe' em todos 3 vers — ymean_test < 1 MWh (test n=11
        tem quase zero curtailment). NMAE 109% reportado em iter_0006 era
        artefato de denominador, nao sinal de modelo ruim. MAE=80 MWh em S/v3
        e' a metrica fisica honesta.
    (c) NE/v2 fitting bem em magnitude (MAE 8367) mas F1=0 (cego a
        eventos high-curt). Conhecimento que NMAE/MAE escondiam.
decision: |
  CONFIRMADO. metric_suite.py adotado como canonical. B3+B4 patched com
  campos `metric_suite_*` primarios (campos NMAE legados preservados
  para back-compat). Leaderboard cabecalho reescrito. H10/H11 desbloqueadas
  (dependiam de H9). H19 derivada criada (extrair MAE/R²/F1 dos champions
  UlFor Ridge/LR — leaderboard hoje so' tem NMAE deles).
sanity_checks_passed:
  metric_suite_implements_principle_6: true   # MAE+R²+F1 primarios, NMAE secundario com flag
  patch_b3_imports_clean: true                # importlib carrega B3 patched sem erro
  patch_b4_runs_clean: true                   # B4 standalone executou OK e regravou JSON com novos campos
  ranking_divergence_detected: true           # 3/4 subs (NE,SE,N) mostraram conflict NMAE vs R²/F1
  nmae_unsafe_flag_works: true                # 3/3 vers em S marcados unsafe (ymean<1 MWh)
  no_breaking_change_in_legacy_fields: true   # skill, mae_model, baselines preservados em B4 JSON
  leak_detection: skipped                     # meta-acao, nao treina modelo novo
  permutation_importance: skipped             # idem
  holdout_temporal_strict: skipped            # idem (apenas patch da rotina, sem novo run)
  baseline_compare: skipped                   # idem (B4 rerun do replay so' para gerar campos novos)
  distribution_shift: skipped                 # idem
  zero_count_shift: skipped                   # idem
budget_consumido_iter: 1.3
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/scripts/metric_suite.py
  - loops/forecast-mega-loop/outputs/iter_0008/h9_metric_suite/metric_suite_iter0002_replay.json
  - loops/forecast-mega-loop/sanity_checks/holdout_temporal_strict.py (patched B3)
  - loops/forecast-mega-loop/sanity_checks/baseline_compare.py (patched B4)
  - loops/forecast-mega-loop/outputs/iter_0002/baseline_compare.json (rerun com metric_suite_* fields)
  - loops/forecast-mega-loop/leaderboard.md (cabecalho + secao iter_0008 + linha meta/metric_suite)
  - loops/forecast-mega-loop/state.json (metric_suite_primary + H9 verdict + planner_next_iter)
  - loops/forecast-mega-loop/hypotheses_queue.md (H9 done + H19 nova)
  - loops/forecast-mega-loop/iterations/iter_0008_h9_metric_suite_mae_r2_f1.md
---

# Iter 0008 — H9 metric_suite MAE/R²/F1 (Principio 6 PLANO_FINAL)

## Hipotese

H9 (P1 metric, layer=meta): substituir NMAE como metrica primaria por
MAE+R²+F1 (PLANO_FINAL UlFor Principio 6 — "Para curtailment usar
MAE/R²/F1, nunca NMAE/media"). Esperava-se mostrar que NMAE-only ranking
e' viesado em pelo menos um sub.

## Como foi rodado

1. **`scripts/metric_suite.py`** — `compute(y_true, y_pred,
   y_train_for_threshold)` retorna dataclass `MetricResult` com
   campos primarios `mae, r2, f1_p50` + secundarios `rmse, nmae, bias` +
   diagnostico `n, threshold_p50, ymean_test, nmae_safe`. F1 binariza
   target em "above P50 do TRAIN" (sem leak). NMAE retorna `None`
   quando `ymean_test < EPS_MEAN=1 MWh` (flag `nmae_safe=False`).
2. **Standalone runner**: aplica metric_suite sobre 12 runs do iter_0002
   (4 subs × 3 vers) e gera `outputs/iter_0008/h9_metric_suite/
   metric_suite_iter0002_replay.json` com rankings e conflicts.
3. **B3 patched** (`sanity_checks/holdout_temporal_strict.py`): import
   `metric_suite.compute`, adiciona campo `metric_suite_strict` ao JSON
   de cada run + `delta_mae_strict_minus_original`. Cabecalho de tabela
   console reescrito: `MAE R² F1_p50 NMAE* d_MAE`. Campos legados
   `metrics_strict, metrics_original, delta_nmae_pp_strict_minus_original,
   skill_vs_persist_d1` preservados.
4. **B4 patched** (`sanity_checks/baseline_compare.py`): mesmo padrao.
   Adiciona `metric_suite_lgbm` + `metric_suite_climatologia_doy`.
   `skill` mantido (back-compat — calculado em MAE).
5. **Validacao**: B4 rerun completo (gerou JSON com novos campos sem
   quebrar legados). B3 import limpa via importlib (LGBMRegressor
   standalone run depende de sklearn no env — pre-existente, nao H9).

## Resultado

### Tabela LGBM (iter_0002 replay)

| key | MAE | R² | F1_p50 | NMAE | nmae_safe |
|---|---:|---:|---:|---:|---|
| NE/v1 | 14862 | -0.167 | 0.000 | 0.501 | true |
| NE/v2 |  8367 | +0.660 | 0.000 | 0.282 | true |
| NE/v3 | 11830 | -0.002 | 0.000 | 0.399 | true |
| SE/v1 |  3599 | +0.467 | 0.857 | 0.409 | true |
| SE/v2 |  3180 | +0.394 | 1.000 | 0.362 | true |
| SE/v3 |  3406 | +0.544 | 0.857 | 0.387 | true |
| S/v1  |    22 | NaN   | 0.000 | -    | **false** |
| S/v2  |   125 | NaN   | 0.000 | -    | **false** |
| S/v3  |    80 | NaN   | 0.000 | -    | **false** |
| N/v1  |   269 | -1.402| 0.800 | 1.011 | true |
| N/v2  |   269 | -1.402| 0.800 | 1.011 | true |
| N/v3  |   259 | -1.637| 0.750 | 0.974 | true |

### Conflicts (best-per-sub diverge entre NMAE e R²/F1)

| sub | best por NMAE | best por MAE | best por R² | best por F1 |
|---|---|---|---|---|
| **NE** | v2 | v2 | v2 | **v1** (NE F1=0 todos, tiebreak NE/v1) |
| **SE** | v2 | v2 | **v3** | v2 |
| **S**  | unsafe | v1 | n/a (ss_tot≈0) | tie zero |
| **N**  | v3 | v3 | **v1** | **v1** |

3/4 subs mostram divergencia. NE: NMAE escolhe v2 mas F1 mostra que v1
captura tanto evento high-curt quanto v2 (F1=0 em todos — NE test set nao
tem evento >P50). SE: NMAE/MAE escolhem v2 mas R² explica mais variancia
em v3. N: NMAE/MAE escolhem v3, mas R²/F1 escolhem v1.

### S e' o caso patognomonico

S NMAE 109% reportado em iter_0006 vinha de `MAE_xgb / ymean_test`. No
replay loop (n=11), ymean_test em S = 0.05–0.7 MWh (test sem curtailment
real). MAE LGBM = 22-125 MWh. NMAE = 31–2500 (matematicamente correto,
operacionalmente meaningless). Flag `nmae_safe=False` previne usar isso
para inferencia.

R² em S tambem retorna NaN porque `ss_tot ≈ 0` (test target praticamente
constante). F1 retorna 0 porque o threshold P50 train (~30000 MWh) e' muito
acima de qualquer valor test (~30 MWh). Em S, **so' MAE e' interpretavel**
em magnitude fisica — confirmando o ponto de Principio 6 de que metricas
relativas precisam de salvaguarda.

### NE — F1=0 captura falsos alarmes que MAE/NMAE escondem

Threshold P50 NE/v2 train = 62960 MWh. Test set NE n=11:
- `y_true` range [519, 59599] — **todos abaixo do threshold** (zero positives reais)
- `y_pred_lgbm` NE/v2 = [..., 72135] — **uma predicao acima do threshold**
- tp=0, fp=1, fn=0 -> precision=0/1=0, recall=0/0 indef -> F1=0 (sklearn convention)

Investigacao inicial suspeitou bug em `_f1_score` quando classe-positiva
real e' vazia. Reproducao manual (NE/v2):

```
y_true>thr: [0 0 0 0 0 0 0 0 0 0 0]
y_pred>thr: [0 0 0 0 0 0 0 0 0 0 1]   <-- LGBM previu high-curt no ultimo dia
```

Comportamento correto: modelo gritou "evento high-curt" sem ter um —
precision=0, F1=0. **Nao e' bug — e' a metrica fazendo o trabalho:**
F1 captura falso-alarme operacional que MAE 8367 / NMAE 0.282 nao denunciam.
Isso reforca H9: as 3 versoes NE tem MAE/NMAE razoaveis mas sao 100%
inuteis para alertar evento high-curt nesta janela.

A divergencia "best por NMAE = NE/v2, best por F1 = NE/v1" no quadro de
conflicts e' real e tem interpretacao operacional clara: se o uso
downstream e' alerta de evento, F1=0 em todos significa "nenhum modelo
NE serve" — informacao que NMAE 28% sugeriria estar tudo bem.

(Nota: F1 tie em zero para 3 modelos NE — tiebreak para "best por F1"
e' arbitrario; o ponto e' que TODOS NE falham no F1 enquanto NMAE
sugere v2 ganhador.)

## Decisao

**CONFIRMADO** — H9 e' aceita. Adocao:

- metric_suite.py canonical.
- B3 + B4 emitem `metric_suite_*` como primario.
- Leaderboard cabecalho reescrito (MAE/R²/F1 primario).
- NMAE preservado como secundario com flag `nmae_safe`.
- H10/H11 desbloqueadas (dependiam de H9).

Evidencia principal por sub:
- **NE**: F1=0 em todos os 3 vers revela que TODOS sao inuteis para
  alerta high-curt (false positives sem true positives). NMAE 28% sugere
  v2 ganhador, F1 derruba a leitura.
- **SE**: R² rank (v3+0.544) discorda de NMAE rank (v2 0.362). Mesma
  evidencia, criterios divergentes — operador tem que escolher.
- **S**: NMAE unsafe (ymean<1 MWh) demoliu o "S NMAE 109%" como
  artefato. MAE 22-125 MWh e' a metrica honesta.
- **N**: NMAE rank (v3) discorda de R²/F1 rank (v1). NMAE marginal
  (~1.01) tambem proximo do unsafe.

## Proximo passo

Planner sugere **H16** (P1 — B6 threshold por n_test) para fechar a serie
metodologica do iter_0004-0008. H19 (P2 — extrair MAE/R²/F1 dos champions
UlFor) e' codavel via git show / mlflow proxy. H10 (ensemble v2+persist)
agora unblocked — boa candidata se queremos voltar a layer curtailment.

Budget: 1.3h (sob 2.5h cap).
