---
alvo: h17_fase4_promote_SE_S
layer: curtailment
iter_num: 0007
type: hypothesis_test (verdict=SUPERSEDED) + recon_delta
data_utc: 2026-05-24T05:30:00Z
baseline_tipo: persist_d1 (CV 5 folds 60d, official_source UlFor)
hypothesis: |
  H17 (P0): promover SE/v3 + S/v3.3 XGB para FASE 4 (model serializer +
  drift monitor). Premissa: numeros iter_0006 (SE 46.0%, S 109% XGB v3.3
  pos-PDP-fix) sao definitivos; loop deve emitir req-0004 pedindo a
  UlFor a operacionalizacao.
result_metric: |
  H17 SUPERSEDED. UlFor self-actionou entre iter_0006 (be2c9186) e
  iter_0007 (4e0fc7b4) via 2 commits novos:
    - 76732289: Ridge baseline-controle BATE XGB (-4.4pp NMAE em NE, -8.6pp em S)
    - 4e0fc7b4: CV walk-forward 5 folds CONFIRMA — champions per-sub:
        NE: ridge_alpha10 33.7±8.1% / R²+0.469±0.098 (5/5 folds)
        SE: lr_sklearn 46.6±13.4% / R²+0.380±0.139 (4/5 folds)
        S:  lr_sklearn 89.6±31.2% / R²+0.447±0.202 (4/5 folds) [CORTA xgb -19.4pp]
        N:  ridge_alpha10 86.3±31.8% / R²+0.196±0.289 FRAGIL (nao promovivel)
  Champion XGB de iter_0006 nao e mais champion em NE/SE/S. req-0004 nao
  emitido — UlFor ja' tem @champion registry plan + OOT 2x agendado.
decision: |
  H17 -> status=done, verdict=SUPERSEDED_BY_ULFOR_RIDGE_LR_CV.
  H18 nova (P1 methodology, blocked req-0005): auditar Ridge/LR via B1-B6
  antes de FASE 4 final. Loop continua queue (proximo planner select: H16 ou H9).
sanity_checks_passed:
  ulfor_session_sync_iter0007: true        # commits 76732289 + 4e0fc7b4 absorved
  champion_shift_detection: true           # mudanca em 3/4 subs capturada
  no_redundant_req_emission: true          # req-0004 nao emitido (UlFor ja' actiona)
  cv_evidence_robustness: true             # Ridge_NE std 8.1% < xgb 20.8% — robust
  leak_detection: skipped                  # meta-acao, nao treina modelo
  permutation_importance: skipped          # idem
  holdout_temporal_strict: skipped         # idem
  baseline_compare: skipped                # idem (baseline updated indiretamente)
  distribution_shift: skipped              # idem
  zero_count_shift: skipped                # idem
budget_consumido_iter: 1.0
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/outputs/iter_0007/fase_4_promote_SE_S/verdict.md
  - loops/forecast-mega-loop/outputs/iter_0007/fase_4_promote_SE_S/champion_shift_evidence.json
  - loops/forecast-mega-loop/state.json (best_ml_oficial_ulfor_cv_5folds + DEPRECATED xgb section)
  - loops/forecast-mega-loop/hypotheses_queue.md (H17 done + H18 nova)
  - loops/forecast-mega-loop/leaderboard.md (4 linhas champions + 4 linhas DEPRECATED)
  - loops/forecast-mega-loop/iterations/iter_0007_h17_fase4_promote_superseded.md
---

# Iter 0007 — H17 superseded por Ridge/LR CV (recon-style)

## Contexto

H17 entrou na queue como P0 em iter_0006, baseada nos numeros XGB v3.3
absorbidos do commit UlFor `b7acfdcd` (SE/v3 XGB 46.0% PROMOVIVEL,
S/v3.3 XGB 109% quebra teto persist). Esperava-se que iter_0007 emitisse
req-0004 ao UlFor para FASE 4: model serializer + drift monitor.

Durante a janela de ~5h entre iter_0006_fim (be2c9186) e iter_0007_inicio
(4e0fc7b4), UlFor (sessao paralela) executou autonomamente um achado
disruptivo: baseline-controle Linear/Ridge BATE GBDT em 3/4 subs.

## UlFor commits absorved

### `76732289` — Ridge baseline-controle BATE XGB em NE/S

PLANO_FINAL FASE 3 item 5 ("Se o GBDT nao bate o DLinear com folga,
o problema e FEATURE nao ARQUITETURA") — CONFIRMADO.

Adicionado sklearn LinearRegression + Ridge(alpha=10) ao bakeoff_d1.py.
Resultado single fold 60d:

| Sub | xgb NMAE/R² | ridge NMAE/R² | lr NMAE/R² |
|---|---|---|---|
| NE | 35.7% / +0.402 | **31.3% / +0.493** | 33.6% / +0.457 |
| SE | 46.0% / +0.386 | 46.1% / +0.343 | 47.4% / +0.251 |
| S  | 109% / -0.135 | 100.4% / +0.097 | **92.2% / +0.316** |
| N  | 75.9% / -0.070 | 77.9% / +0.086 | 78.9% / +0.052 |

### `4e0fc7b4` — CV walk-forward 5 folds CONFIRMA Ridge/LR

5 folds × 60d test cada, train expanding. 140 runs + 28 CV_SUMMARY no MLflow
`bakeoff-curtailment-d1`. Resultado mean ± std:

| Sub | persist | xgb | ridge | lr | Champion CV |
|---|---|---|---|---|---|
| NE | 44.5±17.3% | 44.8±20.8% | **33.7±8.1%** R²+0.469 | 40.3±14.1% | **ridge** 5/5 folds |
| SE | 68.8±13.8% | 56.5±13.9% | 50.8±7.0% | **46.6±13.4%** R²+0.380 | **lr** 4/5 folds |
| S  | 124.2±24.1% | 115.8±30.0% | 100.1±18.5% | **89.6±31.2%** R²+0.447 | **lr** 4/5 folds |
| N  | 100.7±9.3% | 94.2±28.2% | **86.3±31.8%** R²+0.196 | 106.1±65.4% | **ridge** 2/5 FRAGIL |

Robustez: Ridge_NE std 8.1% e' MENOR que xgb 20.8%, lgbm 21.5%, lr 14.1%.
Sinal real, nao overfit acidental.

## Por que H17 vira SUPERSEDED (nao CONFIRMADO)

H17 propunha promover XGB SE/v3 + S/v3.3 a FASE 4. Premissa parcial
invalidada por:

- **SE/XGB no point estimate (single fold n=60d) era 46.0%**, mas CV mostra
  **xgb 56.5±13.9%**. LR vence em 4/5 folds com mean 46.6%. XGB e estavel
  no single fold mas instavel atraves de regimes.
- **S/XGB 109% (single fold)** vira **115.8±30.0% em CV**. LR domina com
  89.6±31.2% (R² +0.447 vs xgb -0.135 = +0.58 absoluto). LR e o champion
  real, nao XGB.
- **NE: xgb 35.7%** ja' nao era o melhor mesmo no single fold — Ridge bate
  a 31.3%. Em CV, Ridge 33.7±8.1% > xgb 44.8±20.8%.
- **N: lgbm 72.2% (single fold)** vira **99.5±37% em CV**. CV mostra que N
  permanece intratavel (ma7 e ridge competem; ridge marginal). H17 nao
  cobria N de qualquer forma.

A INTENCAO de H17 (promover a FASE 4) e' valida e necessaria. Mas o ARTEFATO
a promover mudou — Ridge/LR substituem XGB.

## Por que NAO emiti req-0004

UlFor JA executa a acao P0 que H17 pedia. Verificado no checkpoint
`ulfor_2026-05-24T02-15.md` (commit `4e0fc7b4`) e em
`FINDING_RIDGE_BEATS_GBDT.md`:

- Ridge entra no @champion candidate set (28 runs logged MLflow)
- Promover ridge_NE como @champion apos OOT 2x test windows (agendado)
- Investigar multicolinearidade (VIF) das 55 features sub-level

Loop emitir req-0004 ("favor promover SE/v3 + S/v3.3 XGB") seria
duplicacao + obsoleto. UlFor proxima retomada (autopilot envelope)
fecha @champion registry + OOT.

## H18 — Follow-up planejado

Criada nesta iter (P1 methodology, blocked):

```yaml
- id: H18
  summary: Auditar champions Ridge/LR pos-OOT via sanity B1-B6
  type: methodology
  layer: curtailment
  target: ridge_lr_champion_audit_pre_fase4
  priority: P1
  status: blocked
  depends_on: [req-0005]   # request UlFor a publicar predicoes
  sanity_checks_required: [leak, perm, holdout, baseline, dist_shift, zero_count]
  acceptance: 5/6 passam clean = GO FASE 4; >=2 falham = investigacao
```

req-0005 ainda nao escrito — aguarda iter posterior (loop respeita
"NAO emite req desnecessario" — UlFor pode publicar predicoes a parquet
naturalmente no flow autopilot, ou loop emite na proxima iter).

Considerar tambem H19 (VIF multicolinearidade) sugerida pelo
FINDING_RIDGE_BEATS_GBDT.md hipotese 1 — Ridge ganha porque features
sub-level sao multicolineares e L2 reg captura estrutura conjunta.

## Estado da queue pos-iter_0007

- DONE: H1, H2, H4, H6, H17 (este)
- QUEUED: H3, H7, H8, H9, H10, H11, H13, H15, H16, H18 (este)
- BLOCKED: H5 (req-0004 ja' atrasou planejado, mas e termico — separar),
  H12 (req-0005 carga refresh), H14 (req-0006 v3.3 preds), H18 (req-0005)

Proximo planner select (sugerido): **H16** (P1 — B6 robustness ajustar
threshold por n_test) porque e' codavel no loop sem dep UlFor + serve
para evitar falsos positivos como o iter_0004. Alt: **H9** (P1 metric
suite MAE/R²/F1 substituindo NMAE — alinhado com PLANO_FINAL UlFor).

## Sanity checks aplicaveis

Os 6 default (leak, perm, holdout, baseline, dist_shift, zero_count) sao
SKIPPED porque esta iter nao treina modelo nem avalia metricas novas
(meta-acao + recon, padrao iter_0006).

Sanity checks meta-acao aplicaveis e aprovados:

| check | resultado |
|---|---|
| ulfor_session_sync_iter0007 | OK (2 commits absorved em state.json) |
| champion_shift_detection | OK (3/4 subs marcados) |
| no_redundant_req_emission | OK (req-0004 NAO emitido) |
| cv_evidence_robustness | OK (CV 5 folds + std comparison) |

## Decisao final

**SUPERSEDED** — `H17` original (promover XGB) e' obsoleta. A INTENCAO
(promover algo a FASE 4) prossegue via `H18` quando UlFor publicar
predicoes Ridge/LR para auditoria B1-B6. Sem req-0004.

## Proximo passo

iter_0008 — planner deve selecionar **H16** (codavel) ou **H9** (codavel,
P1 alinhado PLANO_FINAL). H18 fica blocked ate predicoes Ridge/LR
estarem em parquet acessivel.

Budget: 1.0h (sob 2.5h cap).
