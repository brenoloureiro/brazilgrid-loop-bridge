# Leaderboard — forecast-mega-loop

Atualizado em iter_0038 (2026-05-25T15:30Z, **H27 P50 quantile como
point estimate substituto em N+S — CONFIRMADO**: re-analise direta de
H11 iter_0014 (bit-exato; random_state=0 + same X + same split). D1
target N+S **PASS 6/6 cells** (100%, threshold >=4/6=67%): N mean delta
**-17.9% MAE** (15/15 folds wins, queue dizia "-18%"), S mean delta
**-13.8% MAE** (11/15 folds, queue dizia "-14%"). D2 constraint NE+SE
**PASS na sub-mean**: NE +4.5% (just under +5% threshold), SE -1.6%
(empate). Bonus: **delta_R2 mean N+S = +0.287** (LGB-mean tem R²
negativo em varios folds N/v1; P50 puxa para positivo — sample
extremo N/v1 fold5 flipa de R²=-0.676 para R²=0.218, magnitude 0.89);
delta_F1 mean N+S = +0.022. Decision **PROMOVE_NS_FLAG_NE**: P50
canonical em N+S+SE (drop-in superior, custo zero, mesmo treino mesmo
X swap de hiperparametro); NE mantem LGB-mean por cauda densa (NE/v2
+5.6%, NE/v3 +8.4% individuais > 5% threshold — mecanismo: cauda densa
premia mean, KS p<0.0001 iter_0012 confirma shift NE). Sanity B3
holdout passed_embedded (gap=7d, 60 folds), B4 baseline passed_embedded
(12/12 cells P50 >= persist_d1), B5 dist_shift annotated_reuse (KS NE
explica gap), B1/B2 skipped_inherited (PI iter_0010 p=0), B6 n_test
passed (n_test 58-60 > 30). **0 follow-ups criados**: H28 (NGBoost) e
H37 (CQR-asymmetric+Mondrian) ja queued cobrem "cauda densa precisa de
cauda-aware model" gap NE.
Atualizado anteriormente em iter_0037 (2026-05-25T08:00Z, **H26 conformal prediction
post-hoc INDETERMINADO_NE + bonus CONFIRMADO_SE_S**: split conformal
(Lei et al. 2018 + Romano CQR 2019 symmetric) sobre LGBM quantile fica
borderline em NE — cov_cal mean 74.7% (falha [75%,85%] strict por 0.3pp),
width_ratio_cal_vs_inner 2.03x (falha <2.0 por 0.03x), 3/3 NE cells in
[70%,90%] loose. iv=60 nao salva (cov cai p/ 71.6%, train_inner
empobrece). **Bonus deliverable SE+S: 3/3 cells strict, cov 76.6%/79.5%,
width 1.95x/1.51x** — vira bandas P10/P90 entregavel AGORA para
operador SE+S (decisao Breno, fora scope iter). N over-cobre (90.8%,
width 1.68x) — outliers cauda alta dominam q_alpha global. Replicacao
H11 bit-exato (cov_uncal_full == iter_0014 summary.csv: 45.5/43.5/41.8
NE). Mecanismo conformal validado (cov_cal > cov_uncal em 100% dos 60
fold-runs). Limite e' do TARGET (NE distribution shift documentado
iter_0012 KS p<0.0001), nao do metodo. **H37 derivada criada**:
CQR-asymmetric (separa q_low / q_high — corrige N) + Mondrian conformal
por regime (adapta intra-test — corrige NE folds 2-3) rodadas na mesma
iter. Sanity: holdout passed_embedded, baseline passed_embedded
(tripla UNCAL_FULL/UNCAL_INNER/PERSIST + replicacao H11), dist_shift
annotated_reuse, leak+perm skipped (mesmas features iter_0010 PI p=0.0),
zero_count n/a (banda nao point).
Atualizado anteriormente em iter_0036 (2026-05-25T07:00Z, **RECON_DELTA UlFor
`80620230..4f4e21f4`, 5 commits — 0 substantivos + 5 checkpoints
PARAR-E-PERGUNTAR**). Pior janela substantiva do recon-style (iter_0028
25% / iter_0031 25% / iter_0033 40% / iter_0036 **0%**). UlFor entrou em
**standby puro zero-trabalho** — sprints envelope-safe esgotados em
iter_0033 (smoke loader + fix wrap stdout). NOVIDADE: runner UlFor agora
expoe **SNR auto-medido na mensagem de commit** (23.5% no 9o, 22.2% no
10o checkpoint) — confirmacao matematica de standby legitimo (NAO bug).
Tunnels CH+MLflow `exit=28` mantem-se. Promote v3 (Breno iter_0031) NAO
EXECUTADO. Champions, loader.py e MLflow Registry **INTOCADOS**.
Recomendacao planner para iter_0037: **NAO emitir novo recon_delta** se
HEAD == `4f4e21f4` — rodar H30 (P3 ~1h, fecha H3-family) ou H27 (P50
substituto custo zero, ortogonal a UlFor).
Atualizado anteriormente em iter_0035 (H25 stacker Ridge meta-modelo
REFUTADO_NO_GAIN: Ridge sobre 4 bases [lgb, persist_d1, ma7, clim_doy]
treinado em inner_val=30d **perde para H10 inv_mae em 12/12 cells**, pct_delta
+2.1% a +90.0% (NE +10.1%, SE +14.2%, S +57.1%, N +33.3% mean). Best variante
e' `ridge_no_intercept_a10` mas mesmo sem intercept o estimador overfit
inner_val: shift MAE_test/MAE_inner_val medio 64-283% por cell, com fold
extremo NE/v1 28280%. Ridge com intercept (alpha=0.1..100) e' catastrofico
(MAE explode 50x-200x). Lesson: 30d × 4 bases = 5 params nao da' SNR para
empirical risk minimization; weighted average analitico inv_mae H10 vence
porque NAO memoriza inner_val. **H10 inv_mae confirmado como state-of-art
para ensemble curt D+1 multi-base em janelas curtas.** Sanity required pela
queue (holdout, leak, baseline) AMBOS PASS; perm n/a, dist_shift+zero_count
reported. Verdict robusto: nem 1 cell confirma >=2% reducao clinica).
Atualizado anteriormente em iter_0034 (H22 GBDT vs OLS REFUTADO_NO_NONLINEAR_GAIN).
Atualizado anteriormente em iter_0033 (2026-05-25T04:30Z, RECON_DELTA UlFor
`2917289c..80620230`, 5 commits — 2 substantivos + 3 checkpoints
PARAR-E-PERGUNTAR. UlFor preparou **smoke test pos-promote v3**
(`tests/services/test_forecast_loader_smoke.py`, 5 offline passed + 8
online gated em CH/MLflow) que destrava quality_gate automatico assim
que tunnels EC2 subirem. **Bugfix Windows-only** em `bakeoff_d1.py`
isola `sys.stdout = TextIOWrapper(...)` ao `__main__` — destravava 13
smoke tests + 19 importers (incluindo `loader.py` em PROD). Champions,
loader.py defaults e MLflow Registry **INTOCADOS**. Promote v3
(decidido por Breno em iter_0031) **NAO EXECUTADO** — mesmo bloqueio
infra: tunnels CH+MLflow exit=28).
Atualizado anteriormente em iter_0032 (H20 — auto-flag bake-off
n_test<30 CONFIRMADO_DISPLAY: politica `low_confidence_n_test` explicita
e auditada em 32 artefatos do loop + 18 parquets UlFor; 12 LGBM-replay
runs iter_0002 + 8 sanity JSONs marcados LOW, 0 falsos negativos no topo
do leaderboard. Runner `run_bakeoff_replay.py` patcheado para escrever
flag em meta.json + summary_replay.csv).
Atualizado anteriormente em iter_0031 (RECON_DELTA UlFor
`27152e16..2917289c`, 8 commits — 2 substantivos + 6 checkpoints. Breno
**oficializou promote v3 final**: SE opt A `ridge+h22_MA+α=1` (val14d > CV);
H14-G bias correction NE registrado como UlFor H25 para promote_champions.py,
NAO loader.py. Promote NAO executado — CH local feat_termico stale 2024-12-31).
Atualizado anteriormente em iter_0030 (H15 S classifier — CONFIRMADO_PARCIAL_NON_RARE).
Suite canonica MAE/R²/F1/RMSE/skill + NMAE/bias secundarios. Fonte unica:
parquets UlFor commit `6b21ffdf`
(`experiments/bakeoff_curtailment_multisub/outputs/cv_summary_*.parquet`),
+ 7 parquets alpha-sweep iter_0026 (h22_MA × {α=1,100,1000} + h22_per_fold ×
{α=1,100}) + **3 parquets val14d real iter_0028** (`*_val14d_alpha*`).
**Zero retrain neste iter.**

**Protocolo CV** (4 subs × 7 modelos × 5 folds = 140 runs por feature_set):

- Walk-forward 5 folds, test=60d, gap=7d entre train e test
- Train period: 2024-12-01..2026-03-23 (cresce por fold)
- Test period last fold: 2026-03-24..2026-05-21 (60d)
- Target: `curt_d1` (sum em MWh por sub, agregacao Jensen-puro)

**Suite primaria** (PLANO_FINAL UlFor Principio 6 + nosso H9 CONFIRMADO
iter_0008): `MAE`, `R²`, `F1_p50` — F1 binariza target em `> P50(train)`
sem leak. Secundaria: `NMAE`, `RMSE`, `bias`. Flag `nmae_safe=true` quando
`ymean_test >= 1 MWh` (todos os 4 subs passam — S problema era artefato
n=11 do replay loop iter_0002).

**Skill_vs_persist_d1** = `1 - MAE_champion / MAE_persist_d1` no mesmo CV.
Positivo = champion ganha persist em MAE.

**Politica `low_confidence_n_test`** (H20, iter_0032): qualquer bake-off com
`n_test < 30` recebe flag explicito `low_confidence_n_test=true` no meta.json
do run e no `summary_replay.csv`, e e' explicitamente isolado na secao
"Historico iter loop (deprecado)" do leaderboard. Todos os 4 champions
oficiais + 12 baselines abaixo usam **n_test ∈ {59, 60}** (UlFor CV 5x60d /
val14d single-fold) — `low_confidence_n_test = FALSE`. Auditoria full em
`outputs/iter_0032/h20_leaderboard_low_n_test_warning/n_test_audit.{json,md}`.

---

## Champions oficiais UlFor (CV 5 folds × 60d, gap 7d, periodo 2024-12 → 2026-05)

| sub | modelo | feature_set | n_feat | MAE (MWh) | R² | F1_p50 | RMSE (MWh) | NMAE | skill_vs_persist | bias (MWh) | nmae_safe |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| NE | ridge_alpha10 | full | 55 | **27 317 ± 11 920** | **+0.469 ± 0.098** | **0.808 ± 0.170** | 34 516 ± 13 773 | 33.7 ± 8.1% | **+17.7%** | −13 549 ± 14 503 | ✓ |
| SE | lr_sklearn | full | 55 | **6 117 ± 873** | **+0.383 ± 0.094** | **0.785 ± 0.108** | 8 098 ± 1 124 | 46.5 ± 13.5% | **+34.0%** | −839 ± 3 043 | ✓ |
| S | lr_sklearn | full | 55 | **805 ± 441** | **+0.371 ± 0.164** | **NaN** | 1 289 ± 513 | 87.2 ± 22.6% | **+34.5%** | +42 ± 255 | ✓ |
| N | ridge_alpha10 | clean_plus | 31 | **425 ± 149** | **+0.170 ± 0.185** | **0.790 ± 0.048** | 535 ± 134 | 85.3 ± 27.6% | **+16.4%** | +70 ± 236 | ✓ |

Notas:
- **F1 NaN em S** e' esperado: threshold_p50(train) = 0 (sub com muitos
  zeros estruturais; binarizacao P50 nao discrimina). Documentado em
  iter_0017 req-0007 closure.
- **N champion @staging** (nao @champion). Versao v2 clean_plus (31 feat)
  promovida iter_0015 sobre v1 full (NMAE 86.3±31.8% → 84.8±26.2%, std −5.6pp).
- **Bias NE/SE estruturalmente negativo** (champions subestimam): trigger
  para bias_correction productizado em PROD (ver "Overlay de producao").

## Baselines (mesmo CV, mesmo feature_set, mesmo target)

| sub | baseline | MAE (MWh) | R² | F1_p50 | RMSE (MWh) | NMAE |
|---|---|---:|---:|---:|---:|---:|
| NE | persist_d1 | 33 183 ± 8 233 | +0.102 ± 0.285 | 0.745 ± 0.211 | 42 205 ± 8 258 | 44.5 ± 17.3% |
| NE | persist_d7 | 58 850 ± 24 461 | −1.297 ± 0.333 | 0.568 ± 0.367 | 71 355 ± 26 948 | 76.6 ± 31.1% |
| NE | ma7 | 39 146 ± 14 243 | −0.067 ± 0.085 | 0.623 ± 0.340 | 48 057 ± 15 398 | 51.3 ± 19.2% |
| SE | persist_d1 | 9 273 ± 1 741 | −0.409 ± 0.208 | 0.692 ± 0.166 | 12 292 ± 1 997 | 68.8 ± 13.8% |
| SE | persist_d7 | 11 073 ± 1 668 | −0.912 ± 0.194 | 0.556 ± 0.254 | 14 297 ± 2 040 | 84.4 ± 24.5% |
| SE | ma7 | 8 447 ± 1 506 | −0.027 ± 0.069 | 0.649 ± 0.274 | 10 490 ± 1 483 | 63.8 ± 16.7% |
| S | persist_d1 | 1 230 ± 762 | −0.517 ± 0.302 | NaN | 2 072 ± 990 | 124.2 ± 24.1% |
| S | persist_d7 | 1 555 ± 1 061 | −1.120 ± 0.216 | NaN | 2 482 ± 1 219 | 151.8 ± 17.4% |
| S | ma7 | 1 269 ± 796 | −0.139 ± 0.068 | NaN | 1 814 ± 884 | 130.8 ± 31.2% |
| N | persist_d1 | 508 ± 129 | −0.608 ± 0.335 | 0.715 ± 0.054 | 747 ± 179 | 100.7 ± 9.3% |
| N | persist_d7 | 575 ± 130 | −0.938 ± 0.307 | 0.647 ± 0.081 | 817 ± 176 | 116.2 ± 21.0% |
| N | ma7 | 464 ± 83 | −0.088 ± 0.073 | 0.786 ± 0.072 | 612 ± 112 | 93.5 ± 9.4% |

Notas:
- `ma7` (moving average 7d) ja vence persist_d1 em MAE em SE+N+S — usar
  como sanity floor. Champion deve bater **ambos**.
- N champion (MAE 425) bate ma7 (MAE 464) por −8% — overlap modesto.
- S champion (MAE 805) bate ma7 (MAE 1269) por −37% — ganho real.
- **persist_d7 vs persist_d1 (iter_0029, H13)**: persist_d1 vence em 4/4 subs no
  CV agregado (NE +77.3%, SE +19.4%, S +26.4%, N +13.1%) e em 19/20
  per-fold cells. Unica inversao: **N fold 0** (d7=441.6 vs d1=592.4,
  −25.4%), regime temporal mais antigo do walk-forward (consistente com
  seca-2025Q3 dominante em N tardio, ja' identificado em iter_0018).
  **Diagnostico estrutural**: curtailment tem persistencia diaria forte e
  ciclo semanal fraco — contra-intuitivo para audiencia operacional. Sub-claim
  "persist_d7 vence em S" da H13 e' REFUTADO (S e' onde d1 vence d7 por
  +26.4%, 2a maior margem).

## Overlay de producao (NAO no CV puro — pos-processamento)

A rota `/api/forecast/d1` aplica **bias_correction rolante causal** sobre
champions em NE+N por default (`predicted_curt_mwh_default`). Modelo
treinado **inalterado** no MLflow Registry — correcao e' wrapper de
inferencia.

| sub | bias_correction default | janela | impacto CV NMAE | 14d real NMAE | observacao |
|---|---:|---:|---:|---:|---|
| NE | **ON** | 28d | −6.9pp (wins 4/0/5) | raw 59.2% → **corrected 49.2%** | bate persist 54.1% por −4.9pp — **primeira vez no projeto** ML > persist em NE 14d real |
| N | **ON** | 60d | −19.6pp (wins 3/1/5) | smoke pred 246 → default 267 | fold-4 (seca-2025Q3) dominante, raw 226% |
| SE | OFF | 28d | −0.3pp (wins 0/1/5) | +1.6pp regime change | janela fixa joga correcao no rumo errado em transicao chuvoso→seco |
| S | OFF | 28d | +1.0pp (wins 2/3/5) | +10.8pp catastrofico | ymean ~32 MWh amplifica ruido bias |

Fonte: iter_0017 (NE) + iter_0018 (N) + iter_0019 (frente formalmente
fechada SE/S declarados intrinsicamente nao-corrigiveis).

## Candidatos sucessores (UlFor CV ∧ 14d real, **NAO promovidos em PROD**)

Aguardam decisao Breno. Validation gap **FECHADA em iter_0024** via
single-fold 14d real (commits `99af14b7` + `5d41d063`). Detalhes em
iter_0021, iter_0022, iter_0023, iter_0024.

### Veredito final post-validation (CV 5x60d ∧ 14d real)

Tabela consolidada das 7 acoes da CHAMPION_DECISION_MATRIX. Test 14d =
2026-03-24..2026-05-21 (n=59 = ultimo fold CV).

| sub | candidato | NMAE_CV | NMAE_14d | delta vs status quo | veredito | iter |
|---|---|---:|---:|---:|---|---|
| NE | ridge + h22_per_fold | 31.4% | 30.9% | CV −8.8pp / 14d tie −0.2pp | **CONFIRMADO — PROMOVER** | 0024 |
| NE | bias H14-G(w=14,k=1) | strict Pareto vs H14-C | nao-testavel single-fold | -0.9pp + 4→5 wins / 0 loses | **CONFIRMADO — PROMOVER** | 0024 |
| SE | lr + h22_model_aware | 48.1% | 43.2% | CV tie / 14d −2.0pp / -11 features | **CONFIRMADO — PROMOVER** (mitiga cond_num 2.5e17) | 0024 |
| SE | ridge + h22_per_fold | 48.3% | 45.7% | CV −1.8pp / 14d −0.5pp (tradeoff NMAE↔estab) | promover marginal | 0024 |
| S | lr + h22_per_fold | 84.3% | 94.4% | CV −2.9pp **mas 14d +0.8pp PIOR** | **REFUTADO — MANTER STATUS QUO** | 0024 |
| N | ridge + h22_per_fold | 86.4% | 74.0% | CV −23.7pp / 14d −1.2pp vs `lr+full` | **CONFIRMADO — PROMOVER** (corrigido em `5d41d063`) | 0024 |
| N | bias H14-G(w=60,k=1) | trade-off vs H14-B | nao-testavel single-fold | +8.4pp / 0→2 less loss | decisao Breno | 0024 |

**SE LR fragilidade** (commit `34478f93` iter_0021): cond_num(X) ~2.5e17.
h22_MA mitiga (-11 features) preservando familia LR. Promovivel sem trocar
familia, vs alternativa ridge+h22 que mudaria modelo.

### Achados novos 14d real (NAO promoviveis — falta CV win)

UlFor sugere reabrir CV 5x60d para lgbm em SE+N como proximo sprint
envelope-safe. Padrao: regime atual mais nao-linear que historico CV.

| sub | candidato | NMAE_14d | gap | interpretacao |
|---|---|---:|---|---|
| SE | lgbm + h22_model_aware | **42.7%** (best 14d SE) | CV 5x60d nao testou lgbm × h22_MA | regime SE mais nao-linear que historico CV |
| N | lgbm + full | **68.1%** (best 14d N) | CV 5x60d 96.4% (perde -10pp para ridge+h22) | mesmo padrao SE; viola "promover so apos CV win" |

### Promote enxuto recomendado (UlFor autopilot post-validation iter_0024)

**4 acoes alta-confianca** (CV ∧ 14d real):

1. **NE**: `lr+full` → `ridge+h22_per_fold` + bias `H14-C` → `H14-G(w=14,k=1)`
2. **SE**: `lr+full` → `lr+h22_model_aware`
3. **N**: `lr+full` → `ridge+h22_per_fold` (bias H14-B vs H14-G decisao Breno)
4. **S**: **MANTER STATUS QUO** `lr+full` (unica refutacao formal post-14d)

### Alpha sweep v3 (commits `42dc0d7a` + `a3c742a9` + `75e2431e`, iter_0026)

UlFor estendeu sweep para `ridge × {h22_MA, h22_per_fold} × {α=1, 10, 100, 1000}`.
**α=1 vence 3/4 subs** em ambos feature_sets H22; α=100 dedicado a N
(mais ruido/colinearidade justificam shrinkage); α=1000 degenerado em todos
subs. h22_MA ≡ h22_per_fold com α=1 (delta <0.5pp). **SURPRESA SE**:
`lr+h22_per_fold` QUEBRA (R² −0.07!) mas `ridge+h22_per_fold+α=1` entrega
R² +0.43 — regularizacao minima resolve `cond_num 2.5e17` sem trocar familia.

| sub | champion v2 atual (matriz iter_0023) | **champion v3 (alpha-aware)** | NMAE CV | R² CV | Ganho v2→v3 |
|---|---|---|---:|---:|---|
| NE | ridge + h22_per_fold (α=10) | **ridge + h22_per_fold (α=1)** | **30.81%** | **+0.563** | −0.6pp NMAE (~tie) |
| SE | lr + h22_model_aware | **ridge + h22_per_fold (α=1)** | **46.60%** | **+0.430** | **−10.6pp NMAE** + R² +0.50 vs lr+h22_pf |
| S | lr + full (status quo) | lr + full (status quo) | 87.2% | +0.371 | n/a (val refuta h22 em S) |
| N | ridge + h22_per_fold (α=10) | **ridge + h22_per_fold (α=100)** | **84.43%** | **+0.216** | −2.0pp NMAE / R² +0.02 |

**Vantagem operacional v3**: NE+SE+N todos rodam `h22_per_fold`, so o `alpha`
muda. `promote_champions.py` ja patcheado (commit `75e2431e`) com
`--ridge-alpha` + feature_sets `h22_per_fold` / `h22_model_aware` /
`h22_stricter` (back-compat preservada, default α=10 legado).

**Validation gap PARCIALMENTE REABRE para v3**: SE `ridge+h22_pf+α=1` NAO
foi testado em 14d real (iter_0024 testou apenas α=10 default). Principio 5
(CV win first) NAO basta — UlFor explicito: "nao promovivel sem val_recent".
Loop NAO emite req-0008 nesta iter (padrao pre-empcao UlFor self-actiona).
**→ FECHADO em iter_0028** (UlFor self-actionou em <90min; ver bloco a seguir).

### Val14d alpha sweep v3 — holdout real recente (commits `f7c56c3d` + `1bd8638f` + `ff112a27`, iter_0028)

Holdout estrito: train **2024-12-01 → 2026-03-23**, test **2026-03-24 →
2026-05-21** (n≈58d). 3 candidatos ridge × 4 subs. Pre-empta `req-0008` que
loop ia emitir.

| sub | h22_MA + α=1 | h22_pf + α=1 | h22_pf + α=100 | veredito val14d |
|---|---:|---:|---:|---|
| NE | **30.88% / +0.476** | **30.88% / +0.476** | 34.59% / +0.380 | empate h22_MA = h22_pf — promover h22_pf por coerencia |
| SE | **43.03% / +0.392** ✅ | 44.67% / +0.344 | 47.83% / +0.227 | **INVERSAO vs CV**: h22_MA vence val14d por −1.64pp |
| S | 95.03% / +0.176 | 93.20% / +0.177 | 100.5% / +0.071 | TODAS perdem persist (~93%) — manter `lr+full` |
| N | 75.31% / +0.057 | 75.32% / +0.056 | **71.51% / +0.158** ✅ | confirma α=100 + h22_pf (val MELHOR que CV — concept drift positivo) |

**Mecanismo da divergencia SE** (commit `ff112a27` diff features): `h22_MA`
preserva 10 features que `h22_pf` dropa: CMO (×3), Intercambio
(`val_export_mwmed`, `val_import_mwmed`), regime (`taxa_penetracao`,
`ger_eolica_mwh`), `carga_pico_mw`, `prev_solar_pico_mw`, `ter_verif_rmean7`.
Causa: `h22_pf` usa Ridge universal PI (L2 mascara importance); `h22_MA`
usa champion-model real (LR p/ SE preserva load-bearing). Regime val14d
(Mar-Mai/2026) favorece CMO+intercambio (curt economico em alta) — features
marginais que CV agregado dilui voltam a contribuir.

**Convergencia com H8 iter_0027**: `val_export_mwmed` + `val_import_mwmed`
estao entre as 10 features divergentes. iter_0027 H8 ja' provou via
joint-drop refit que dropar intercambio em SE Ridge α=1 e' HARMFUL
(+1.21pp NMAE). iter_0028 val14d empiricamente reforca: opt_A (preserva
intercambio + outras 8) BATE opt_B (dropa intercambio + outras 8) por
−1.64pp NMAE. Mesma direcao, magnitudes consistentes, contextos
independentes.

**Promote v3 atualizado (decisao Breno aberta)**:

| sub | Acao | Champion v3 | Fonte da decisao |
|---|---|---|---|
| NE | PROMOVER | `ridge + h22_per_fold + α=1` | CV + val14d coincidem (30.88%) |
| SE | opt_A | `ridge + h22_model_aware + α=1` | val14d 43.03% / +0.392 (Breno aposta regime recente persistir) |
| SE | opt_B | `ridge + h22_per_fold + α=1` | CV 46.60% / +0.430 + coerencia multi-sub + menor overfit (recomendacao tecnica UlFor) |
| S | MANTER | `lr + full` (status quo) | val14d confirma rejeicao ridge em ambos |
| N | PROMOVER | `ridge + h22_per_fold + α=100` | CV+val14d coincidem; val14d 71.51% MELHOR que CV |

Decisao SE defensavel em ambos sentidos. Diff <2pp NMAE = margem amostral
val14d (n≈58d). Loop NAO tem voto. `promote_champions.py` ja' patcheado
(iter_0026 commit `75e2431e`). Branch 26+ ahead origin — Breno + push.

### Decisao Breno oficial v3 final (commit `0971c699`, iter_0031)

Breno respondeu ao PARAR-E-PERGUNTAR escolhendo **opt A em SE** (val14d > CV):

| sub | acao | champion v3 final | fonte primaria |
|---|---|---|---|
| NE | PROMOVER | `ridge + h22_per_fold + α=1` | CV + val14d coincidem (30.88%) |
| SE | **PROMOVER opt A** | **`ridge + h22_model_aware + α=1`** | **val14d** > CV+coerencia (regime drift) |
| S | MANTER | `lr + full` (status quo) | val14d confirma rejeicao |
| N | PROMOVER | `ridge + h22_per_fold + α=100` | CV + val14d coincidem |

Justificativa SE opt A: val14d e' sinal recente do regime real (CMO subindo,
intercambio SE-S invertendo, NE em expansao). As 10 features que `h22_MA`
preserva e `h22_pf` dropa (CMO + intercambio + taxa_penetracao) carregam
sinal nesse regime. Aceita perda de coerencia multi-sub. Reavaliar Jun/2026.

**Convergencia 3-fold**: H8 iter_0027 (joint-drop refit em SE Ridge harmful)
+ iter_0028 val14d (h22_MA vence h22_pf por −1.64pp) + iter_0031 (Breno
escolhe explicitamente opt A) = 3 evidencias independentes confirmando que
preservar intercambio + CMO em SE Ridge α=1 carrega sinal real.

**H14-G implementacao** (decisao Breno): em `promote_champions.py`, NAO em
`loader.py`. Bias correction e' parte do artefato promovido (binding ao
modelo). MLflow tag carrega bias spec (`bias_window`, `bias_threshold_k`,
`bias_strategy`). Loader.py deve ser dumb (carrega artefato + aplica bias
parametrizado). Registrado por UlFor como **H25_ulfor** para sprint
envelope-safe proxima. NAO confundir com nosso H25 (Stacker Ridge meta-modelo,
P3 queued).

### Promote v3 NAO EXECUTADO (commit `6111cda4`, iter_0031) — bloqueio infra

CH local Docker (porta 8123) tem `feat_termico` congelada em **2024-12-31**
(upstream raw `ons_raw___geracao_termica_despacho_ho` sem ingestao Dagster
recente). `feat_carga_history` 2 meses stale; `feat_pld/pdp/inter/sat/cmo`
8-23 dias stale. Dataset colapsa a 16 dias finais Dez/24 vs ~535 esperados.
MLflow tambem offline local. Sweep alpha=1/100 (iter_0028 `f7c56c3d`) rodou
contra **EC2 CH via tunnel SSH** (porta 18123 = SSH forward), nao Docker
local. Convergente com auto-memory [[ch_local_feat_termico_stale]] +
[[ch_local_vs_ec2_separados]] (Mai/26 Breno). Aguarda EC2 setup ou sync raw.
Comandos prontos: ver `FINDING_RIDGE_ALPHA_SWEEP.md` secao "Comandos prontos
para retomar (ambiente correto)".

### Ablation negativa H22 stricter (commit `42dc0d7a`, iter_0026) — REFUTADO

Testado `MIN_FOLDS_DROP=5` (drops unanimes 5/5 folds em vez de 3/5). Drop-sets
ficam muito menores (NE 22→7, SE 11→3, S 4→1, N 10→2), mas CV regride em
3/4 subs vs `h22` (3/5):

| sub | h22_stricter NMAE | vs h22 (3/5) | vs h22_MA (3/5) |
|---|---:|---:|---:|
| NE | 33.37% | +1.96pp PIOR | +1.96pp PIOR |
| SE | 48.17% | −8.99pp vs h22 (MELHOR) | +0.11pp (≈ MA) |
| S | 88.08% | +3.74pp PIOR | +2.25pp PIOR |
| N | 86.73% | +0.33pp (≈) | −0.50pp marginal |

**Lesson**: features qualificadas em 3-4/5 folds (ambiguas) carregam sinal
residual util para generalizacao. Manter `MIN_FOLDS_DROP=3`. Encerra essa
linha de pesquisa.

### ADDENDUM val14d z-score (commit `7cc3b604`, iter_0026)

Complementa `FINDING_14D_REAL_VALIDATION` (iter_0024 `99af14b7`) com 2
angulos: (a) z-score `val_recent` vs envelope CV: **TODOS** pontos
`|z| ≤ 1σ` → sem regime shift detectavel; (b) refuta sugestao de "reabrir
CV para LGBM SE/N" — dados ja existem em `cv_summary_mean_*.parquet` e
`lgbm+h22_MA` em SE perde CV por −4.6pp apesar de ganhar val por +0.46pp.
Princípio 5 (CV win first) bloqueia promote LGBM. **Implicacao loop**: H27
(P50 quantile) atratividade INALTERADA — janela atual nao destrava LGBM.

### Sucessores via H22_model_aware (commit `2daa5d40`, iter_0023)

Patch que substitui PI universal (Ridge) por PI medida com **champion-model
real de cada sub** (NE/N=ridge, SE/S=lr). Resolve bug H23_ulfor (Ridge L2
mascara importance de colineares — caso canonico `ter_verif_rmean7` em SE/lr).

| sub | candidato | feature_set | NMAE | R² | n_drops | delta vs H22 universal |
|---|---|---|---:|---:|---:|---|
| NE/N | ridge | h22_model_aware | ≡ H22 | ≡ H22 | ≡ H22 | sem mudanca (mesmo champion) |
| SE | **lr** | **h22_model_aware** | **48.1%** | **+0.381** | 11 (−10 vs H22=21) | **+9.1pp NMAE / +0.449 R²** vs H22 universal |
| S | lr | h22_model_aware | 84.3% | +0.387 | 4 (−3 vs H22=7) | +2.9pp NMAE / +0.387 R² vs full |

**Implicacao operacional**: SE agora tem candidato sucessor **SEM trocar
familia LR** (era unica opcao "ridge+h22" pos-iter_0021). Recomendacao
matriz: SE lr+h22_MA preserva NMAE (+1.6pp vs full / R² ≈ empate) e
**mitiga cond_num 2.5e17 com −11 features**. PROMOVIVEL.

### Bias correction alternativa H14-G (commit `1ba9cb39`, iter_0023)

Sweep `window × k` para threshold sigma_bias (reusa fold_one H14-E). Subs
N+NE (SE/S nao-corrigiveis per H14-F). Criterio strict: wins >= −5pp.

| sub | atual em prod | candidato H14-G | mean dNMAE | wins/loses | apply_rate | decisao UlFor |
|---|---|---|---:|---:|---:|---|
| **N** | H14-B w=60 always-on (−19.63pp 3W/1L 100%) | **w=14d k=1.5** | **−31.87pp** | **3W/1L** | **41%** | **PROMOVIVEL** (SUPERA por −12.24pp) |
| NE | H14-C w=28 always-on (−6.90pp 4W/0L 100%) | w=14d k=0.5 (best) | −8.29pp | 3W/0L | 79% | MANTER status quo (−1.39pp marginal) |
| NE | H14-C w=28 always-on (−6.90pp 4W/0L) | w=14d k=1.0 | −7.79pp | 5W/0L | 66% | Pareto-strict (Wins↑) — opcao secundaria |

UlFor explicito: **PARAR E PERGUNTAR Breno antes de promover** (mudar
loader.py = tocar prod). Custo produtizar similar H14-F (cache
`sigma_bias_rolling_train` em runtime).

### CHAMPION_DECISION_MATRIX (commit `c5004bac`, iter_0023)

Documento decision-aid consolidando **7 acoes PROMOVIVEIS** (5 modelo+feature,
2 bias correction). Validation gap explicito: **tudo CV, sem 14d real**.
UlFor sugere `--feature-set h22_per_fold --cv-folds 1 --no-mlflow` antes
de produtizar. Ranking risk/value:

| Acao | Esforco | Beneficio CV | Risco |
|---|---|---|---|
| Promover NE ridge+h22 | Baixo | −8.8pp NMAE / +0.07 R² | Baixo |
| Promover N ridge+h22 | Baixo | **−23.7pp / +0.51 R²** | Baixo |
| Promover SE lr+h22_MA | Baixo | +1.6pp / −0.002 R² | Mesma ordem do full + −11 feat |
| Promover SE ridge+h22 | Baixo | +1.8pp / +0.007 R² | Trade-off NMAE↔estabilidade |
| Promover S lr+h22 | Baixo | −2.9pp / +0.016 R² | Baixo |
| NE bias H14-C→H14-G(w=14,k=1) | Baixo | −0.9pp / Wins↑ | Pareto strict |
| N bias H14-B→H14-G(w=60,k=1) | Baixo | +8.4pp / 2 loses evitadas | Trade-off, nao Pareto |

**CAVEAT detectado pelo loop**: matriz "Champion ATUAL em prod" lista
TODOS os 4 subs como `lr + full`, contradiz state.json (NE=ridge, N=ridge).
Provavel bug de documentacao da matriz (outras secoes "substituir X por Y"
sao consistentes com state.json). FLAGGADO para proxima recon; leaderboard
**mantem ridge_NE / lr_SE / lr_S / ridge_N** por seguranca.

### Sucessores via ensemble (H24_loop CONFIRMADO_3SUBS iter_0022)

Replicas locais champion (sklearn) + persist_d1 com peso analitico
inv_mse/equal em CV 5×60d (mesma estrutura H10):

| sub | champion_only MAE | best_ens MAE | delta | best_scheme | wins |
|---|---:|---:|---:|---|---:|
| NE/v3 | 38 677 | 29 588 | **−9 089 (−24%)** | ens_equal | 4/5 |
| SE/v3 | 8 734 | 6 633 | **−2 100 (−24%)** | ens_inv_mse | 4/5 |
| N/v1 | 570 | 463 | **−107 (−19%)** | ens_inv_mse | 4/5 |
| S/v3 | 726 | 777 | +50 (PIORA) | — | 1/5 |

R² flip dramatico em SE/v3: −0.78 → +0.06. Ganho 4x maior que H10 LGBM
(confirma predicao H24: lineares deixam mais variancia residual para
persist absorver). NAO produtizado — req opcional UlFor: blend
champion+persist em runtime para NE+SE+N.

---

## Conflitos de ranking observados

Best-per-sub diverge entre metricas em 0/4 subs no nivel de **champion vs
baseline** — todas as 4 subs concordam que champion bate persist em
MAE+R²+F1+RMSE+NMAE+skill simultaneamente. **Ranking unico** este iter.

Conflitos PERSISTEM no nivel de **escolha de modelo intra-sub** (ridge vs
lr, full vs clean_plus) — documentados nos handoffs iter_0017
(req-0007) e iter_0021 (H22_ulfor). Ex: NE lr+clean_plus tem MAE
ligeiramente menor que ridge+clean_plus (28 512 vs 27 021) mas F1
ligeiramente pior (0.802 vs 0.799 — empate); ridge venceu por NMAE (33.2%
vs 34.7%) e por estabilidade std.

## Pendencias (metricas nao extraidas neste iter)

1. **bias_correction overlay metrics em CV**: parquet UlFor reporta CV
   sobre champion *raw*; valores pos-correction so' existem em logs 14d
   real (iter_0017/0018). Para fechar — req-0008 ao UlFor (P3): rodar
   validate_d1 em janela CV-completa e logar `mae_corrected_per_fold`.
2. **F1_p50 em S**: NaN estrutural, nao computavel sob protocolo P50.
   Alternativa: threshold P75 ou P90 — H32 emergente (P3) candidata.
3. ~~**Holdout 14d real para candidatos h22_per_fold NE+SE**~~ → **FECHADA
   em iter_0024** por commits `99af14b7` + `5d41d063` (UlFor self-actionou
   single-fold 14d real para os 7 promovieis). H31_emergente PRE-EMPTED.
   **REABRE PARCIAL em iter_0026** para alpha sweep v3 (SE `ridge+h22_pf+α=1`
   nao testado em 14d real — so CV). Loop NAO emite req formal (padrao
   pre-empcao UlFor multi-agente).
4. **Skill_vs_persist em CV para ensemble (H24)**: replicas locais
   computam champion+persist combinado mas iter_0022 nao reporta
   `skill_ens_vs_persist` explicitamente — derivavel de parquet em iter_0025+.
5. **CV 5x60d para `lgbm × {full, h22_MA}` em SE+N**: achados novos 14d real
   (iter_0024) sugerem regime nao-linear atual; promote bloqueado por
   ausencia de CV. UlFor candidato a executar em proximo sprint
   envelope-safe.

## S binary alert (iter_0030 H15) — classifier viavel para alerta operacional

CV 5×60d gap 7d, S/v1 (37 feats iter_0002), LogReg(class_weight=balanced)
vs LGBMClassifier vs LR_sklearn (champion S) vs Ridge_α10 binarizados.
Persist_d1 binario + climatology como baselines. P50_thresholds derivados
do y_tr (sem leak).

| threshold | pos_rate_te | best CLS AUC | best REG AUC | persist AUC | ΔAUC CLS−REG | best PR-AUC CLS | best PR-AUC REG | ΔPR-AUC |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| thr_zero  (any curt)   | 40% | **0.784 (logreg)** | 0.739 (lr_reg)   | 0.617 | **+0.045** | **0.784** | 0.733 | **+0.051** |
| thr_p75   (big curt)   | 25% | **0.821 (logreg)** | 0.754 (lr_reg)   | 0.633 | **+0.068** | **0.716** | 0.635 | **+0.081** |
| thr_p90   (rare event) | 10% | 0.762 (logreg)     | **0.778 (lr_reg)** | 0.551 | −0.016     | 0.451     | **0.456** | −0.006 |

- **Confirma classifier > regressor binarizado em alerta moderado** (thr_zero
  + thr_p75) com gain 4.5-6.8pp AUC e 5.1-8.1pp PR-AUC.
- **Refuta classifier > regressor em rare event** (thr_p90): regressor
  binarizado empata classifier. Mecanismo: pos absoluto baixo no test (fold
  5: 1 positivo em 60d) inviabiliza calibracao LogReg.
- **Permutation test FORTE**: real LogReg AUC=0.872 (fold 5, thr_zero) vs
  perm 0.512±0.13 (n=50, p=0.000). Sinal nao e' artefato amostral.
- **Distribution shift severo** (status WARN): zero_rate y_te varia
  0.30-0.83 entre folds vs y_tr 0.57-0.75. Explica volatilidade thr_p90.
- **Implicacao produtizavel** (sugestao para UlFor, NAO requisitada): se o
  endpoint /api/forecast/d1 quiser expor "vai ter curtailment em S amanha?"
  como flag binaria, LogReg dedicado a esse problema bate a binarizacao do
  champion LR continuo por 4.5pp AUC sem custo de re-treino do regressor.
  F1 thr_zero: LogReg 0.718 vs LR_reg 0.682 (close), recall LogReg 0.737
  vs LR_reg 0.900 (regressor recupera mais mas com 14pp menos precisao).

Artefatos: `outputs/iter_0030/h15_s_classifier_vs_regressor/`.

## Historico iter loop (deprecado — replays n=11)

Antes do iter_0007 (champions UlFor Ridge/LR CV 5×60d), o loop usou
replays LGBM com n_test=11 e features iter_0002. Numeros foram
superseded mas mantidos para audit. Coluna `n_test` + flag
`low_confidence_n_test` (H20, iter_0032) tornam o aviso explicito por linha
em vez de embutido no titulo da secao. Tabela resumida abaixo; detalhes em
iterations/iter_0002 a iter_0006.

| iter | acao | resultado | n_test | low_confidence_n_test | superseded_por |
|---|---|---|---:|:---:|---|
| 0002 | LGBM v1/v2/v3 replay | NMAE NE 28.2% v2 / SE 36.2% v2 | 11 | **⚠ TRUE** | iter_0007 UlFor CV |
| 0003 | H2 off-by-one PDP | REFUTADO (corr=0.91 t / 0.82 t+1) | — | n/a | — (definitivo) |
| 0004 | B6 sanity check impl | impl + falso positivo SE/v3 lag | 11 | **⚠ TRUE** | iter_0009 v1.1 |
| 0006 | RECON UlFor v3.3 XGB | NE 35.7% / SE 46.0% / S 109% / N 72.2% | 11 | **⚠ TRUE** | iter_0007 Ridge/LR |
| 0008 | H9 metric_suite MAE/R²/F1 | CONFIRMADO — NMAE viesa 3/4 subs | 11 | **⚠ TRUE** (n=11 do replay) | — (definitivo) |
| 0009 | H16 B6 n_test gating | CONFIRMADO — 5/20 FP eliminados n=11 | 10 e 60 (sintetico) | n/a | — (definitivo) |
| 0010 | H3 PDP residual signal | CONFIRMADO NE+SE (perm p=0.0) | 98 (holdout OLS) | FALSE | — (definitivo) |
| 0012 | H7 XGB vs LGBM CV | REFUTADO_LGBM (10/12 wins CV) | 58-60 (CV 5 folds) | FALSE | — (replay-only) |
| 0013 | H10 ensemble LGBM+persist | CONFIRMADO_NE_SE+N (bonus) | 58-60 (CV 5 folds) | FALSE | iter_0022 H24 Ridge/LR |
| 0014 | H11 quantile LGBM | REFUTADO_NE (cov 43.6% vs 80%) | 58-60 (CV 5 folds) | FALSE | — (definitivo) |
| 0016 | H19 extracao champion metrics | CONFIRMADO_PARCIAL (MAE_derived ~10% slack) | n/a (derivacao) | n/a | iter_0017 (MAE exato) |
| 0020 | H21 pdp_residual engineered | REFUTADO (OLS + LGBM CV convergem) | 98 OLS / 58-60 CV | FALSE | — (definitivo) |
| 0022 | H24 ensemble champion+persist | CONFIRMADO_3SUBS (NE/SE/N) | 58-60 (CV 5 folds) | FALSE | — (vivo, candidato runtime) |
| 0024 | RECON_DELTA UlFor 4427a718..5d41d063 | validation_gap dos 7 promovieis FECHADA (4 PROMOVER, 1 REFUTADO, 1 decisao Breno, 1 marginal) + 2 achados novos (lgbm em SE+N regime atual) | 59 (val14d single-fold) + 59-60 CV | FALSE | — (handoff) |
| 0026 | RECON_DELTA UlFor 5d41d063..83abab3e | alpha sweep v3 (NE+SE+N ridge+h22_pf, α-aware) + H22 stricter REFUTADO + promote_champions.py patched + ADDENDUM val14d (LGBM refuted-CV) | 59-60 (CV) | FALSE | — (handoff) |
| 0027 | H8 feat_intercambio CV-PI independente | CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge — joint-drop primario + 30-perm single-feat reproduz UlFor H22 drop direction em 3/4 subs (NE -3.95pp joint a1; S -8.3pp gigante; N inconclusivo unsafe) e diverge em SE (+1.21pp joint a1, lesson model-aware H22_MA empirico). H33 derivada. | 59-60 (CV) | FALSE | — (definitivo) |
| 0028 | RECON_DELTA UlFor 83abab3e..27152e16 | val14d alpha sweep (3 candidatos ridge × 4 subs, n≈58d) FECHA validation gap parcial iter_0026; promote v3 consolidado (NE/N coincidem CV+val14d, S rejeita ambos, SE INVERTE — opt_A h22_MA val14d vs opt_B h22_pf CV); diff features SE elucida mecanismo (10 features carregam regime recente CMO+intercambio); convergencia empirica com H8 iter_0027 | 59 (val14d) | FALSE | — (handoff) |
| 0029 | H13 persist_d7 baseline aux | CONFIRMADO_DISPLAY_REFUTADO_REGIME_CLAIM — persist_d7 ja' presente no leaderboard desde iter_0007 (display OK); sub-claim "vence persist_d1 em S" REFUTADO em CV canonico (persist_d1 vence 4/4 subs no agregado, 19/20 per-fold cells). Unica inversao: N fold 0 (regime sazonal antigo, nao S). Origem da premissa: replay iter_0002 n=11 onde d7 venceu d1 EM N (nao S, erro de transcricao do detail). Mantido como diagnostico auto-correlacao | 59-60 (CV) | FALSE | — (definitivo) |
| 0030 | H15 S classifier vs regressor binarizado | CONFIRMADO_PARCIAL_NON_RARE — em thr_zero (any curt, pos_rate 40%) e thr_p75 (big curt, pos_rate 25%) LogReg(class_weight=balanced) bate LR_reg+Ridge_reg binarizados em **+4.5pp/+6.8pp AUC** e **+5.1pp/+8.1pp PR-AUC**; em thr_p90 (rare event, pos_rate 10%) regressor binarizado EMPATA classifier (ΔAUC −1.6pp, ΔPR-AUC −0.6pp, dentro do ruido). Mecanismo: rare events com test_pos absoluto baixo (1-5 positivos em fold 5) inviabilizam calibracao do LogReg. Perm test FORTE: real AUC=0.872 vs perm 0.512±0.13 (p=0.000). Hipotese original ("classifier > regressor em rare-event") REFUTADA, mas H15 derivada: classifier e' o caminho para alerta binario "vai ter curt em S?" (LogReg AUC 0.78 vs persist 0.62, +16.8pp) | 59-60 (CV) | FALSE | H35 (alerta operacional moderado S) |
| 0032 | H20 auto-flag low_confidence_n_test | CONFIRMADO_DISPLAY — politica `low_confidence_n_test=(n_test<30)` propagada para meta.json/summary_replay.csv/leaderboard; audit cobre 12 meta runs + 21 outros JSON + 18 parquets UlFor; 12 LGBM-replay runs + 8 sanity-JSONs marcados LOW, 0 marcados LOW na secao Champions/Baselines do topo. Runner `run_bakeoff_replay.py` patcheado. | n/a (display) | n/a | — (definitivo) |
| 0034 | H22 GBDT vs OLS gap (3 feats gen+pdp_prev) | REFUTADO_NO_NONLINEAR_GAIN — holdout temporal 80/20 (n_train=388/n_test=98 per sub). NE: OLS R²=+0.550 vs GBDT +0.507 (gap −4.3pp); SE: OLS +0.072 vs GBDT +0.088 (gap +1.5pp TIE); S: OLS −0.252 vs GBDT −0.426 (gap −17.4pp). max_gap=+0.015 < +5pp threshold em todos; mean_gap=−0.067. GBDT overfit train R²=0.81-0.99 vs test 0.51/0.09/−0.43 sob distribution shift (B5 iter_0012 KS<1e-4). PI duo refit-drop reconfirma EMPIRICAMENTE lesson H22_MA UlFor: gap_per_feat ±85pp (NE pdp_eolica OLS 43pp vs GBDT 128pp) — PI eh model-dependent, mas isso NAO traduz em R²_test melhor para GBDT. OLS bate persist_d1 em NE+SE; GBDT empata. H36 derivada (testar com 37 feats v3). | 98 (holdout 80/20) | FALSE | — (definitivo, escopo 3 feats) |

## Lessons learned (transferiveis)

- **NMAE-only ranking enganou inferencia em 3/4 subs** (iter_0008). Suite
  MAE+R²+F1 primaria + NMAE secundaria com flag `nmae_safe` quando
  `ymean<1 MWh` agora padrao do loop.
- **B6 v1.1 downgrade severity quando n_test<30** (iter_0009): replay
  loop em janela curta nao gera req desnecessario ao UlFor.
- **CV 5×60d e' protocolo oficial** mas **holdout 14d real e' ground-truth
  final** acima do CV-mean (iter_0017 UlFor H21 evidenciou: ganho CV
  +11.27pp colapsa para regressao +15.31pp em 14d real porque ganho era
  fold-4-only sazonal).
- **Engineering linear em modelo linear = no-op no melhor caso, perda
  pratica por condicionamento numerico** (iter_0020 H21 REFUTADO).
- **PI deve ser medida com o modelo final, nao com proxy mais robusto**
  (iter_0021 H23_ulfor lesson): Ridge usa L2 para redistribuir importance
  entre colineares → PI subestima. LR sem shrinkage colapsa quando essas
  features sao removidas.
- **Multicolinearidade estatistica != redundancia preditiva** (iter_0011
  H8_ulfor REFUTADA + iter_0021 H22_ulfor PROMOVIVEL): VIF/corr sozinho
  nao guia drops; precisa cruzar com PI per-fold.
- **PI single-feat sobre colinears identitarios e' viesada para CIMA**
  (iter_0027 H8): permutar 1 feature do conjunto colinear val_net =
  val_import - val_export (VIF=1e8) quebra a identidade local; modelo
  treinado com a identidade preserva pesos que dependem dela; predicao
  quebra; aparenta importance alta. **Joint-drop refit do bundle inteiro
  e' o teste autoritativo** — em NE alpha=1, val_import single-PI =
  +1.65pp (parece KEEP) mas joint-drop = -3.95pp (bundle ATIVAMENTE
  HARMFUL). Reforça empiricamente o lesson teorico H22_model_aware
  UlFor commit `2daa5d40` (PI deve ser medida com o modelo final).
- **Display de confianca amostral deve ser per-row, nao por secao**
  (iter_0032 H20): antes desta iter, aviso de janela curta vivia no
  titulo da secao "deprecado" — leitor podia citar uma linha individual
  sem o contexto. H20 promove o flag `low_confidence_n_test` para coluna
  visivel em meta.json + summary_replay.csv + leaderboard, mais auditoria
  unificada em `outputs/iter_0032/h20_leaderboard_low_n_test_warning/`.
  Threshold `n_test < 30` herdado de B6 H16 v1.1 (iter_0009).
- **Capacidade nao-linear sem features novas != ganho** (iter_0034 H22):
  com apenas 3 features (gen + pdp_prev_eolica + pdp_prev_solar), GBDT
  defaults overfit (R²_train 0.81-0.99 → R²_test 0.07-0.55) e NAO supera
  OLS em nenhum sub. max_gap = +0.015 (SE) abaixo do threshold +5pp em
  todos. Mecanismo: distribution shift severo (B5 iter_0012) penaliza
  modelos com variance alta. PI duo OLS-vs-GBDT diverge ±85pp empiricamente
  (reproducao do lesson H22_MA UlFor com magnitude ampliada por feat-space
  pequeno) mas isso NAO traduz em R²_test melhor. **Conclusao operacional**:
  novo sinal em curt D+1 EXIGE novas variaveis, nao apenas trocar familia
  de modelo sobre as mesmas variaveis (H21+H22 esgotam o espaco de
  "transformar (gen, pdp_prev)" — linear redundante OK, nao-linear tambem).

## Como atualizar

Quando UlFor publicar novo CV bake-off:

1. Ler parquets `cv_summary_mean_{full,clean_plus}.parquet` em
   `experiments/bakeoff_curtailment_multisub/outputs/`.
2. Para champion atual de cada sub: extrair `mae_mean`, `r2_mean`,
   `f1_p50_mean`, `nmae_mean`, `rmse_mean` (computar de per-fold se
   ausente), `bias_mean` + respectivos std.
3. Para persist_d1: extrair mesmo schema.
4. Recomputar `skill = 1 - mae_champion / mae_persist`.
5. Atualizar tabela top + `state.json.leaderboard_oficial_v33_champions`.
6. Quality gate R3 detecta regressao automatica se `best_ml.warn` cair
   abaixo dos thresholds.
