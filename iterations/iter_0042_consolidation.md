---
alvo: consolidation_refuted_streak_2
layer: meta
iter_num: 0042
type: consolidation
data_utc: 2026-05-26T00:00:00Z
trigger: quality_gate.refuted_streak_2 (iter_0040 REFUTADO_RIDGE + iter_0041 CONFIRMADO_LR-metodologico-sem-mudanca-champion)
mode: consolidation
hypothesis: NAO-aplicavel (consolidation reavalia queue, nao testa hipotese)
result_metric: NAO-aplicavel
decision: CONSOLIDA — encerra 5 frentes derivadas em 5 iters; promove H37 como proximo alvo
budget_consumido_iter: 0.3
custo_estimado_usd: 0.0
---

# Iter 0042 — Consolidation (refuted_streak_2)

## Gatilho

`quality_gate` disparou `refuted_streak_2` apos:
- **iter_0040** H30 `pdp_residual_in_ridge_cv` -> **REFUTADO_RIDGE** (apenas NE/v3 alpha=1 marginal pass; SE quebra ambos alphas; criterio AND).
- **iter_0041** H33 `feat_intercambio_joint_drop_se_lr` -> **CONFIRMADO_LR** *metodologico* (insight 3-fold convergence cross-model arquivado, **mas zero mudanca de champion** e zero deliverable operacional — H8 ja tinha decisao Breno fechada em iter_0031).

Ambas iters foram P3 ~0.25-0.4h, custo total combinado <1h, sem mudanca em
producao (Champions UlFor + bias correction inalterados). Gate identifica
"caminhos fechando sem novo signal substantivo" — momento certo para sair
do modo *hypothesis_test* e reavaliar a queue.

## Achados re-lidos das ultimas 3 iters

### iter_0039 (H28 NGBoost vs LGBM quantile, INDETERMINADO_PINBALL_DEGRADA)
- D1 cov calibration **PASS** em ambas dists (Normal 73.5% sub-mean +30pp,
  LogNormal 88.3% +45pp vs LGBM 43.5%).
- D2 pinball preservation **FAIL** (Normal +14.4%, LogNormal +55.5%).
- **Decision NAO_PROMOVE_E_FECHA_CAMINHO_PARAMETRICO_DEFAULT** para NE D+1
  curt. **Caminho NGBoost defaults encerrado** no replay loop.
- 0 follow-ups criados (H37 ja cobre proximo passo = conformal asymmetric +
  Mondrian, ortogonal e custo menor).

### iter_0040 (H30 pdp_residual em Ridge CV, REFUTADO_RIDGE)
- Apenas 1/4 cells passa CONFIRMADO_RIDGE (NE/v3 alpha=1 B +0.013 R²);
  restantes 3/4 violam REFUTADO threshold (-0.032 a -0.064).
- C (residual_split) catastrofico em SE (delta_R² -0.469 a -0.818).
- **Lesson canonica**: B2 perm confirma `sinal_intrinseco` (NE alpha10 +105%
  MAE drop com shuffle, SE +27%); **NAO confirma `feature_engineering_gain`**.
  Span linear (gen, pdp_prev_e, pdp_prev_s) ja contem (gen, residual_*) — 
  transformacao reduz dim sem expandir basis.
- Convergencia H21 (OLS + LGBM) + H30 (Ridge alpha=1, alpha=10) = 
  **ENCERRA H3-family residual no replay loop**.
- Follow-ups criados: **H38 P4** (alpha<1 sensitivity, fecha caveat tecnico,
  baixo impacto pratico) + **H39 P5** (doc-only `B2_interpretation.md`).

### iter_0041 (H33 joint-drop SE em LR, CONFIRMADO_LR)
- LR_SE joint_dnmae_pp = +1.316 (>+0.5pp threshold). 4/5 folds positivos.
- Direcao 4/4 subs consistente cross-model: NE -4.285 / SE +1.316 / S -7.376
  / N -1.816 (LR ~ Ridge alpha=1 limit).
- **Insight metodologico**: PI single-feat em LR para colinears perfeitos
  (val_net = val_import - val_export) EXPLODE 172000-3000000 pp por
  `lstsq` min-norm SVD + cancelamento de pesos; val_net_lag1 nao-colinear
  PI razoavel 0.18-2.5pp. **|PI_single_feat| arbitrariamente grande em
  colinears = METRICA QUEBRADA**; joint-drop refit dissolve artefato.
- **Bundle intercambio CASE FECHADO** (3-fold convergence Ridge a10 +
  Ridge a1 + LR). Caveat H22_MA SE drop_HARMFUL reproduzido empiricamente
  no champion LR — UlFor decisao operacional confirmada por direcao.
- **Sem mudanca de champion** — H33 era curiosidade metodologica
  pos-iter_0031.
- Follow-up criado: **H40 P5** (doc-only `B2_interpretation.md`, caso 2
  apos H30 NE alpha=10 caso 1).

### Padrao agregado (ultimas ~5 iters: 0037, 0038, 0039, 0040, 0041)
| iter | hipo | verdict | mudou champion? | follow-up gerado |
|---|---|---|---|---|
| 0037 | H26 conformal | INDETERMINADO_NE + BONUS_SE_S | nao (bandas deliverable SE/S, fora scope iter) | H37 (P3) |
| 0038 | H27 P50 substituto | CONFIRMADO_PROMOVE_NS_FLAG_NE | nao (recomendacao p/ UlFor; loop nao executa) | 0 |
| 0039 | H28 NGBoost | INDETERMINADO_PINBALL_DEGRADA | nao | 0 |
| 0040 | H30 pdp_residual Ridge | REFUTADO_RIDGE | nao | H38, H39 |
| 0041 | H33 intercambio LR | CONFIRMADO_LR (metodologico) | nao | H40 |

**5 frentes encerradas** em 5 iters:
1. NGBoost defaults para incerteza (H28)
2. H3-family residual engineering (H21 OLS + H30 Ridge + iter_0020 LGBM supplement)
3. Bundle intercambio drop em SE (H8 Ridge + H33 LR; 3-fold convergence)
4. Conformal symmetric para NE (H26 borderline; H37 vai testar asymmetric)
5. Stacker Ridge meta-modelo (H25, iter_0035)

**3 deliverables informativos** no caminho:
- H27 P50 promove em N+S+SE (recomendacao UlFor; custo zero)
- H26 conformal symmetric vira deliverable SE+S (cov 76.6%/79.5% strict)
- H33 metodologico canonico para `B2_interpretation.md`

## Status atual da queue pos-consolidation

### Hipoteses queued executaveis no loop (sem dep externa)
| id | prio | escopo | custo | atratividade pos-consol |
|---|---|---|---|---|
| **H37** | P3 | CQR-asymmetric + Mondrian conformal NE+N | 2h | **ALTA** — deliverable real (bandas P10/P90 NE), fecha gap borderline H26 (0.3pp do limite NE) + corrige N over-coverage (90.8% -> [75,90]%) |
| H35 | P3 | S binary alert endpoint via LogReg dedicado | 2.5h | BAIXA — bloqueada sem pedido formal Breno (alerta binario nao e prioridade vs forecast continuo) |
| H36 | P3 | GBDT vs OLS gap full features (37 feats v3) | 1h | DIMINUIDA — H22 iter_0034 ja refutou em 3 feats; H21+H22 esgotam linear/nao-linear sobre mesmas variaveis; lesson formada "novo sinal exige novas variaveis, nao trocar modelo" |
| H38 | P4 | Ridge alpha<1 minimal-feat residual | 0.3h | BAIXA — fecha caveat tecnico de H30 (NE/v3 alpha=1 +0.013 R²); explicitamente baixo impacto pratico |
| H39 | P5 | doc-only `B2_interpretation.md` caso 1 (H30) | 0.2h | governance — agrupar com H40 |
| H40 | P5 | doc-only `B2_interpretation.md` caso 2 (H33) | 0.5h | governance — agrupar com H39 |

### Hipoteses blocked
- **H5** (P2) feat_termico freshness impact — `req-0004` ao UlFor nao emitido.
- **H12** (P1) feat_carga_history stale — `req-0005`.
- **H14** (P2) audit v3.3 — `req-0006`.
- **H18** (P1) audit champions Ridge/LR pos-OOT — `req-0005` (mas atratividade
  caiu com Fase 4 UlFor: `validate_d1.py` daily + drift PSI + Telegram alert da
  cobertura empirica continua; MLflow exposto via CF Access em commit cd12cf2a
  espera 2 passos manuais Breno).

## Rollback de leaderboard? **NAO**

Champions oficiais UlFor (NE ridge_alpha10 full / SE lr_sklearn full /
S lr_sklearn full / N ridge_alpha10 clean_plus) **intactos**. Bias correction
productized NE (28d) + N (60d) intacto. Promote v3 (Breno opt A SE iter_0031)
permanece NAO EXECUTADO por bloqueio infra (CH local stale + MLflow offline
local) — esse bloqueio e' upstream, nao deriva das ultimas 3 iters.

Suite metrica primaria (MAE / R² / F1_p50) + secundaria (NMAE / RMSE / bias /
skill_vs_persist / `nmae_safe` / `low_confidence_n_test`) consolidada em
iter_0025 + iter_0032 permanece autoritativa.

## Re-priorizacao acordada

1. **H37 promove a proximo alvo** (P3, 2h, dependencia: nenhuma externa).
   Mecanismo: CQR-asymmetric separa `q_low / q_high` (corrige N over-cobre
   por cauda dominada em curt~0); Mondrian conformal por regime (split P50
   train_inner) adapta intra-test (corrige NE folds 2-3 sob shift KS<1e-4).
   Aceitacao explicita no queue (cov in [75,85]% NE em >=2/3 cells AND N
   in [75,90]%; OR std<=0.10 entre folds para Mondrian).

2. **H39 + H40 agrupados como governance sprint** (P5, 0.5h combinado, mesmo
   arquivo `sanity_checks/B2_interpretation.md`). Rodar quando proximo
   recon_delta ou iter governance — nao bloqueador, mas valor cumulativo
   alto (2 casos canonicos > 1).

3. **H36 mantida queued mas atratividade DIMINUIDA**. Justificativa:
   convergencia H21+H22 (linear + 3-feat GBDT) cobre o espaco; com 37 feats
   o gap nao-linear ainda existe mas decoupling sinal vs basis ja foi
   teorizado. Rodar so se H37 falhar E queue para a deriva.

4. **H38 mantida P4 queued**. Custo trivial (0.3h) mas resultado nao destrava
   producao mesmo CONFIRMADO. Fica como `nice-to-close` se houver janela
   inativa.

5. **H35 mantida P3 queued mas blocked pratico** — sem pedido Breno para
   alerta binario S, nao escala atratividade.

6. **Blocked H5/H12/H14/H18**: sem mudanca. Loop nao pode resolver
   localmente (req externos OR acao Breno EC2/Cloudflare).

## Decisao

**CONSOLIDA** — proxima iter (0043) e' `hypothesis_test` com alvo
**H37 CQR-asymmetric + Mondrian conformal NE+N**. Acceptance criteria a
priori herdadas do queue (sem mudanca). Roda local sobre features
iter_0002, replicacao bit-exato do split H26 iter_0037 (mesmo CV
walk-forward 5x60d gap 7d).

Se H37 sair REFUTADO ou INDETERMINADO em NE: fechar frente conformal no
replay loop (H26+H37 esgotam o espaco classico de CV-conformal), entrar
em modo recon_delta ate UlFor mover ou Breno desbloquear CH local.

## Proximo passo

`iter_0043` — H37: CQR-asymmetric (q_low/q_high separados, Romano variant)
+ Mondrian conformal por regime (P50_train_inner split, shrinkage com
q_global por inverse-variance). Mesma estrutura `scripts/h26_conformal_prediction.py`
+ 2 variantes adicionais. Output `outputs/iter_0043/h37_*`.

## Sanity checks consolidation

Nao aplicaveis em iter `type=consolidation` — sem treino, sem split, sem
target. Auditoria de queue e' atividade de governance.
