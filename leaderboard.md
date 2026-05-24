# Leaderboard — forecast-mega-loop

Atualizado em iter_0024 (2026-05-24T20:30Z). Suite canonica MAE/R²/F1/RMSE/skill
+ NMAE/bias secundarios. Fonte unica: parquets UlFor commit `6b21ffdf`
(`experiments/bakeoff_curtailment_multisub/outputs/cv_summary_*.parquet`),
extraidos via parse direto. **Zero retrain neste iter.**

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

## Candidatos sucessores (UlFor CV-only, **NAO promovidos em PROD**)

Aguardam decisao Breno + holdout 14d real. Detalhes em iter_0021 e
iter_0022.

### Sucessores via feature engineering H22_ulfor (commit `10fa56d3`)

| sub | candidato | feature_set | MAE | R² | NMAE | delta NMAE vs champion |
|---|---|---|---:|---:|---:|---:|
| NE | ridge | h22_per_fold | TBD | +0.543 | 31.4% | **−2.3pp** |
| NE | lr | h22_per_fold | TBD | +0.565 | 30.8% | **−9.5pp** (vs lr_full +0.302) |
| SE | ridge | h22_per_fold | TBD | +0.390 | 48.3% | **−2.5pp** (vs ridge_full +0.196) |
| S, N | — | h22_per_fold | TBD | neutral | neutral | sem ganho |

**SE LR fragilidade exposta** (commit `34478f93`): champion atual
`lr_curt_se_d1@v1 full` tem cond_num(X) ~2.5e17 — NMAE 46.5% e' "acidental"
(sorte amostral). Recomendacao implicita UlFor: **SE → ridge+h22** (R²
essencialmente empate vs lr atual sem risco numerico). H31_emergente
candidata (replicar h22_per_fold em holdout 14d real NE+SE).

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
3. **Holdout 14d real para candidatos h22_per_fold NE+SE**: bloqueado em
   H31_emergente (req-0008 ao UlFor, ~30min execucao).
4. **Skill_vs_persist em CV para ensemble (H24)**: replicas locais
   computam champion+persist combinado mas iter_0022 nao reporta
   `skill_ens_vs_persist` explicitamente — derivavel de parquet em
   iter_0023.

## Historico iter loop (deprecado — replays n=11)

Antes do iter_0007 (champions UlFor Ridge/LR CV 5×60d), o loop usou
replays LGBM com n_test=11 e features iter_0002. Numeros foram
superseded mas mantidos para audit. Tabela resumida abaixo; detalhes em
iterations/iter_0002 a iter_0006.

| iter | acao | resultado | superseded_por |
|---|---|---|---|
| 0002 | LGBM v1/v2/v3 replay | NMAE NE 28.2% v2 / SE 36.2% v2 | iter_0007 UlFor CV |
| 0003 | H2 off-by-one PDP | REFUTADO (corr=0.91 t / 0.82 t+1) | — (definitivo) |
| 0004 | B6 sanity check impl | impl + falso positivo SE/v3 lag | iter_0009 v1.1 |
| 0006 | RECON UlFor v3.3 XGB | NE 35.7% / SE 46.0% / S 109% / N 72.2% | iter_0007 Ridge/LR |
| 0008 | H9 metric_suite MAE/R²/F1 | CONFIRMADO — NMAE viesa 3/4 subs | — (definitivo) |
| 0009 | H16 B6 n_test gating | CONFIRMADO — 5/20 FP eliminados n=11 | — (definitivo) |
| 0010 | H3 PDP residual signal | CONFIRMADO NE+SE (perm p=0.0) | — (definitivo) |
| 0012 | H7 XGB vs LGBM CV | REFUTADO_LGBM (10/12 wins CV) | — (replay-only) |
| 0013 | H10 ensemble LGBM+persist | CONFIRMADO_NE_SE+N (bonus) | iter_0022 H24 Ridge/LR |
| 0014 | H11 quantile LGBM | REFUTADO_NE (cov 43.6% vs 80%) | — (definitivo) |
| 0016 | H19 extracao champion metrics | CONFIRMADO_PARCIAL (MAE_derived ~10% slack) | iter_0017 (MAE exato) |
| 0020 | H21 pdp_residual engineered | REFUTADO (OLS + LGBM CV convergem) | — (definitivo) |
| 0022 | H24 ensemble champion+persist | CONFIRMADO_3SUBS (NE/SE/N) | — (vivo, candidato runtime) |

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
