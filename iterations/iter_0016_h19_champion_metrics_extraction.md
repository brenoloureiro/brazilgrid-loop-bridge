---
alvo: leaderboard_consistency_post_h9
layer: meta
iter_num: 0016
type: hypothesis_test (verdict=CONFIRMADO_PARCIAL)
data_utc: 2026-05-24T13:30:00Z
baseline_tipo: |
  Leaderboard pre-iter_0016: champions UlFor publicados so' por NMAE
  (33.7% NE ridge / 46.6% SE lr / 89.6% S lr / 84.8% N ridge_v2) +
  R²_mean±std do FINDING. Coluna "best_metric" violava H9 Principio 6
  (MAE em MWh ausente, F1_p50 ausente). Linhas LGBM replay iter_0002
  ja' tinham metric_suite mas eram comparaveis-apenas-entre-si.
baseline_metric: |
  pre-iter_0016 leaderboard best_metric coverage:
    NE: NMAE 33.7±8.1% / R² +0.469±0.098 / MAE=? / F1=?
    SE: NMAE 46.6±13.4% / R² +0.380±0.139 / MAE=? / F1=?
    S:  NMAE 89.6±31.2% / R² +0.447±0.202 / MAE=? / F1=?
    N:  NMAE 84.8±26.2% / R² +0.179±0.176 / MAE=? / F1=?
  (2/3 metricas primarias H9 ausentes)
hypothesis: |
  H19 (P2 metric, layer=meta): extrair MAE em MWh + R² + F1_p50 dos
  4 champions UlFor Ridge/LR atualmente em producao no MLflow Registry,
  populando coluna "best_metric" do leaderboard com metricas
  consistentes (alinhadas a metric_suite H9 Principio 6 PLANO_FINAL).

  Hipotese implicita: pelo menos R² e MAE-em-MWh sao extraiveis sem
  acesso direto ao MLflow do EC2 (que requer SSH tunnel) — via parser
  de FINDING_RIDGE_BEATS_GBDT.md + leitura literal de
  promote_champions.py:CV_METRICS_BY_FS + derivacao MAE = NMAE × ymean
  usando baseline persist_d1 publicado pelo loop em iter_0013.

  F1_p50 nao sabe-se a priori se e' acessivel; se nao for, fechar gap
  via req-0007 ao UlFor (canal autorizado para data_publish).
result_metric: |
  CONFIRMADO_PARCIAL. Extracao R²+NMAE direta + MAE derivado bem-sucedido
  para 4/4 champions; F1_p50 confirmado ausente em todas fontes UlFor.

  Champions com metric_suite consolidado (iter_0016/h19_champion_metrics):

    sub  champion              MAE_mwh*  R²_mean   NMAE_mean  F1_p50
    NE   ridge_curt_ne_d1@ch     25521   +0.469     33.7%     N/A
    SE   lr_curt_se_d1@ch         5622   +0.380     46.6%     N/A
    S    lr_curt_s_d1@ch           916   +0.447     89.6%     N/A (FRAGIL: validate_d1 7-14d skill -41%)
    N    ridge_curt_n_d1 v2@stg    429   +0.179     84.8%     N/A (clean_plus 31 feat)

  *MAE em MWh DERIVED por NMAE_champion × ymean_test_estimated
   onde ymean_test_estimated = MAE_persist_iter0013 / NMAE_persist_FINDING.
   Caveat 1a ordem ~10-15% slack (NMAE_mean publicado e' mean(NMAE/fold),
   nao mean(MAE)/mean(ymean); ymean varia entre folds — iter_0014 NE
   32k->72k->67k->46k->32k MWh).

  Consistency check NMAE_persist FINDING vs state.json: 4/4 subs OK
  exato (0.445/0.688/1.242/1.007). Confirma protocol identity entre
  iter_0013 baseline (loop) e UlFor CV — derivacao valida em 1a ordem.

  F1_p50: bakeoff_d1.py:metrics() / promote_champions.py / validate_d1.py
  computam apenas mae/rmse/nmae/bias/r2. F1 binarizada nunca foi logada
  no UlFor. Gap real e nao-derivavel. req-0007 emitido (P2 data_publish).
decision: |
  CONFIRMADO_PARCIAL. Leaderboard reescrito para 4 linhas curtailment/
  d1_ENE_CNF: MAE_derived (caveat marcado) + R²_direct + F1=N/A
  (req-0007 referenciado) + NMAE secundario + in-sample_R² para
  cross-check. 2/3 metricas primarias agora consistentes no
  leaderboard; F1_p50 atrasado por dependencia externa.

  Bonus: ymean_test_per_sub agora persistido em
  state.json:hypotheses_verdict.H19.champions_metrics_consolidated.*.
  ymean_test_estimated_mwh — reutilizavel em bake-offs futuros sem
  re-derivar do baseline.
sanity_checks_passed:
  leak_detection: skipped   # meta-acao de extracao; sem treino
  permutation_importance: skipped   # sem modelo
  holdout_temporal_strict: skipped_inherited_via_source   # FINDING e' do CV 5x60d gap7d UlFor por construcao
  baseline_compare: true   # skill_vs_persist inline (NE +24.3%, SE +32.3%, S +27.9%, N +15.8%) + consistency check NMAE_persist FINDING vs state.json 4/4 OK
  distribution_shift: annotated_reuse   # std de NMAE/R² grande em S/N reflete iter_0014 KS p<0.0001 NE+SE; capturado no _std de cada metric
  zero_count_shift: skipped   # sem features novas
budget_consumido_iter: 0.8
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/scripts/h19_extract_champion_metrics.py
  - loops/forecast-mega-loop/outputs/iter_0016/h19_champion_metrics/champion_metrics.json
  - loops/forecast-mega-loop/outputs/iter_0016/h19_champion_metrics/champion_metrics.csv
  - loops/forecast-mega-loop/outputs/iter_0016/h19_champion_metrics/all_models_cv_table.csv
  - loops/forecast-mega-loop/leaderboard.md (cabecalho iter_0016 + 4 linhas curtailment/d1_ENE_CNF reescritas + secao detalhada)
  - loops/forecast-mega-loop/state.json (H19 verdict + champions_metrics_consolidated + req-0007 open)
  - loops/forecast-mega-loop/hypotheses_queue.md (H19 done, iter_handled=0016)
  - C:/Projetos/brazilgrid-ulfor/coordination/loop_requests.md (req-0007 OPEN)
---

# Iter 0016 — H19 extrai MAE/R²/F1 dos champions UlFor Ridge/LR

## Hipotese

H19 (P2 metric, layer=meta): champions UlFor (ridge_curt_ne_d1, lr_curt_se_d1,
lr_curt_s_d1, ridge_curt_n_d1) estao publicados no leaderboard apenas com
NMAE, violando o metric_suite H9 (MAE/R²/F1 primarios, PLANO_FINAL Principio 6).

Extrair MAE em MWh + R² + F1_p50 — sem MLflow tunnel ao EC2 — usando:
1. Parser regex da tabela CV em `FINDING_RIDGE_BEATS_GBDT.md`
2. Leitura literal de `promote_champions.py:CV_METRICS_BY_FS`
3. Derivacao MAE = NMAE × ymean (ymean estimado do baseline persist_d1
   publicado pelo loop em iter_0013)

Se F1_p50 nao for extraivel de nenhuma fonte: fechar gap via req-0007.

## Como foi rodado

### Inspecao de fontes acessiveis

```
C:/Projetos/brazilgrid-ulfor/experiments/bakeoff_curtailment_multisub/
  FINDING_RIDGE_BEATS_GBDT.md   # tabela CV NMAE±std + R²±std × 6 modelos × 4 subs
  promote_champions.py          # CV_METRICS_BY_FS hard-coded por feature_set
  bakeoff_d1.py                 # NAO computa F1; logs em MLflow EC2 (sem acesso)
  validate_d1.py                # NAO computa F1; summary JSON consumido por Dagster
C:/Projetos/brazilgrid-loop/loops/forecast-mega-loop/
  state.json:baselines          # NMAE persist_d1 oficial UlFor (4 subs)
  iterations/iter_0013_*.md     # MAE persist_d1 em MWh (4 subs)
```

MLflow EC2 esta atras de SSH tunnel (porta 5000 -> 15000 local) — fora do
envelope read-only do loop. Mas `promote_champions.py` commitou os CV_METRICS
hard-coded por feature_set — extraivel via leitura literal.

### Script de extracao

`loops/forecast-mega-loop/scripts/h19_extract_champion_metrics.py`:

1. **Parser FINDING** (regex linha `| **NE** | 44.5±17.3% | 51.3±19.2% | ...`):
   extrai NMAE_mean ± NMAE_std de 6 modelos (persist_d1, ma7, lgbm, xgb, lr,
   ridge) × 4 subs + champion + R²_mean ± R²_std.
2. **Leitura literal CV_METRICS_BY_FS** (de promote_champions.py:200-220):
   numeros oficiais NMAE+R² mean/std por (sub, feature_set ∈ {full, clean,
   clean_plus}). Champions sao full em NE/SE/S, clean_plus em N.
3. **Cross-check consistency**: NMAE_persist do FINDING (parsed) vs
   `state.json.baselines.curtailment_d1_<sub>.nmae_cv5fold_mean`.
4. **Derivacao MAE em MWh**:
   ```
   ymean_test_estimated = MAE_persist_iter0013_mwh / NMAE_persist_FINDING
   MAE_champion_mwh    ≈ NMAE_champion_mean × ymean_test_estimated
   ```
5. **F1_p50** = None explicitamente (fonte ausente; req-0007).
6. **Output**: champion_metrics.{json,csv} + all_models_cv_table.csv para
   audit.

### Comando

```bash
cd C:/Projetos/brazilgrid-loop/loops/forecast-mega-loop
python scripts/h19_extract_champion_metrics.py
```

## Resultado

### Consistency check NMAE_persist (FINDING vs state.json)

| sub | FINDING | state.json | OK? |
|---|---:|---:|---|
| NE | 0.445 | 0.445 | OK |
| SE | 0.688 | 0.688 | OK |
| S  | 1.242 | 1.242 | OK |
| N  | 1.007 | 1.007 | OK |

**4/4 subs exato.** Confirma que iter_0013 baseline persist_d1 e UlFor CV
usam o MESMO target y_d1 e MESMA particao temporal — derivacao MAE valida
em 1a ordem.

### Champions com metric_suite consolidado

| sub | champion (registered_name @ alias v.) | MAE_mwh* | R² mean±std | NMAE mean±std | F1_p50 | in-sample R² |
|---|---|---:|---:|---:|---:|---:|
| NE | ridge_curt_ne_d1 @champion v1 (full, 55 feat) | **25521** | +0.469±0.098 | 33.7±8.1% | N/A** | 0.830 |
| SE | lr_curt_se_d1 @champion v1 (full, 55 feat) | **5622** | +0.380±0.139 | 46.6±13.4% | N/A** | 0.619 |
| S  | lr_curt_s_d1 @champion v1 (full, 55 feat) | **916** | +0.447±0.202 | 89.6±31.2% | N/A** | 0.725 |
| N  | ridge_curt_n_d1 @staging v2 (clean_plus, 31 feat) | **429** | +0.179±0.176 | 84.8±26.2% | N/A** | 0.472 |

\* **MAE em MWh DERIVED** via `NMAE_champion × ymean_test_estimated`.
   Caveat 1a ordem ~10-15% slack (ver secao "Caveat MAE derived" abaixo).

\** **F1_p50 N/A** — req-0007 emitido para proximo CV bake-off computar.

### Audit: tabela completa CV (6 modelos × 4 subs)

`outputs/iter_0016/h19_champion_metrics/all_models_cv_table.csv` reproduz a
tabela do FINDING com coluna adicional `mae_mwh_derived` para cada modelo.
Util para comparar champion vs persist vs lgbm vs xgb em MAE-em-MWh
estimado (todos sobre o mesmo ymean per sub).

### Caveat MAE derived

NMAE publicado e' `mean(NMAE_per_fold)`. NMAE_per_fold = MAE_fold / ymean_fold.
Logo `mean(NMAE) = mean(MAE/ymean) != mean(MAE) / mean(ymean)`.

Para a derivacao ser exata seria preciso ymean constante entre folds, o que
nao acontece. Iter_0014 documentou para NE: y_test_mean folds [51k, 72k, 67k,
46k, 32k] MWh — varia 2.3x entre folds.

Slack esperado: ate ~15% se erro correlaciona com ymean; menor se descorrelado.
Para MAE-em-MWh exato precisa-se dump per-fold MAE direto do MLflow ou
re-execucao de bakeoff_d1.py --cv-folds 5. Ambos fora do envelope desta
iter — req-0007 pede o dump no proximo round.

### Por que F1 nao e derivavel

F1_p50 (threshold = P50 do TRAIN y) requer:
1. y_train por fold (para computar P50_train sem leak)
2. y_pred binarizada por fold
3. y_true binarizada por fold

UlFor logs MLflow contem (1) e (2) mas (3) nao e' computado em lugar nenhum
(metrics() so' retorna metricas continuas). Para derivar localmente teria
que rodar o bake-off — fora do envelope. Logo: req externo unico caminho.

### Skill score vs baseline (inline B4)

| sub | NMAE champion | NMAE persist | skill_pp | skill_pct |
|---|---:|---:|---:|---:|
| NE | 33.7% | 44.5% | -10.8pp | **+24.3%** |
| SE | 46.6% | 68.8% | -22.2pp | **+32.3%** |
| S  | 89.6% | 124.2% | -34.6pp | **+27.9%** |
| N  | 84.8% | 100.7% | -15.9pp | **+15.8%** |

Todos os 4 champions batem persist_d1 — confirma promocao @champion / @staging
do MLflow Registry.

### Sanity checks (queue requeridos: []. Mas rodar 6 defaults)

- **B1 leak_detection**: SKIPPED — meta-acao de extracao, sem treino de
  modelo novo nem feature nova.
- **B2 permutation_importance**: SKIPPED — sem modelo treinado nesta iter.
- **B3 holdout_temporal_strict**: SKIPPED_INHERITED_VIA_SOURCE — FINDING
  numbers vem do CV walk-forward 5x60d gap 7d UlFor, que e' holdout estrito
  por construcao.
- **B4 baseline_compare**: PASSED — (i) skill_vs_persist computado inline
  para cada champion (acima); (ii) consistency check NMAE_persist FINDING
  vs state.json em 4/4 subs OK exato.
- **B5 distribution_shift**: ANNOTATED_REUSE — std de NMAE/R² grande em
  S/N (iter_0014 KS p<0.0001 NE+SE; iter_0012 ymean cai 3x em NE entre
  folds) ja' aparece no `_std` ao lado de cada metric. lr_S fragility
  no leaderboard tambem reflete dist_shift via validate_d1 7-14d.
- **B6 zero_count_shift**: SKIPPED — sem features novas introduzidas.

`sanity_summary` em `outputs/iter_0016/h19_champion_metrics/champion_metrics.json`
campo `champions_metric_suite_primary_status`.

## Decisao

**CONFIRMADO_PARCIAL.**

- Leaderboard reescrito para 4 linhas curtailment/d1_ENE_CNF com:
  - **MAE em MWh** (derived, caveat marcado)
  - **R²** mean ± std (direct do CV_METRICS_BY_FS)
  - **F1_p50** = N/A explicito + ref req-0007
  - NMAE secundario com std
  - in-sample R² mantido como sanity cross-check
- `last_iter` = 0016 nas 4 linhas; `data_utc` atualizado.
- Champions_metrics_consolidated agora em `state.json` para reuso futuro.
- `req-0007` (P2) emitido ao UlFor: logar F1_p50 + dump per-fold MAE em
  parquet no proximo CV bake-off.

**Por que PARCIAL e nao CONFIRMADO total**: a meta declarada de H19 era
"leaderboard internamente consistente (MAE/R²/F1 em todas linhas)". F1
ficou como N/A — consistencia 2/3 metricas primarias. Quando UlFor
responder req-0007, derivado iter pode promover para CONFIRMADO total
sem nova H.

## Proximo passo

Planner inalterado desde iter_0014/0015: proxima iter retoma **H21**
(P2 feature engineering `pdp_residual = pdp_prev - gen`, derivada H3
iter_0010). Codavel localmente, zero dep externa, sanity bem definido
(B1 leak + B2 perm + B4 baseline). Considerar baseline final com:
- LGBM (iter_0012 confirmou como GBDT padrao)
- Ensemble post-processing inv_mae (iter_0013 H10 confirmado)
- Comparar com MAE-em-MWh derived dos champions Ridge/LR (agora
  disponivel via state.H19.champions_metrics_consolidated)

Quando UlFor fechar req-0007: derivar mini-iter (anexar F1_p50 ao
leaderboard, remover caveat MAE, promover H19 a CONFIRMADO total).
Promovel para iter de "mini-recon" da resposta UlFor, custo <0.5h.
