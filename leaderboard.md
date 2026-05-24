# Leaderboard — forecast-mega-loop

Estado de cada alvo atravessando o DAG. Linha por (layer, alvo, sub).
Atualizado pelo watchdog ao final de cada iteração com ganho promovido.

**Iter 0008 (H9):** metricas primarias agora **MAE/R²/F1** (PLANO_FINAL Principio 6).
NMAE mantida como secundaria — flaggada `unsafe` quando ymean<1 MWh.

**Iter 0021 (RECON_DELTA):** 6 commits UlFor `daf80a6a..ec0fd937` absorvidos em
~15 min reais (13:10-13:35 BRT). **Producao 100% inalterada** (loader.py +
MLflow Registry intocados). **(1) NOVA FRENTE H22_ulfor ABERTA — champion
candidate NE+SE via `feature_set=h22_per_fold`** (commit `10fa56d3`): VIF
(TRAIN only) x Permutation Importance per-fold corrige refutacao H8_ulfor
(greedy puro). CV 5x60d: **NE ridge 33.7->31.4% NMAE (-2.3pp), R² +0.469->
+0.543**; **NE lr 40.3->30.8% (-9.5pp), R² +0.302->+0.565 (ENORME)**; **SE
ridge 50.8->48.3% (-2.5pp), R² +0.196->+0.390**; **S/N neutral**. UlFor
**NAO promove automaticamente** (deferimento Breno, mesmo pattern H14-F
iter_0019). Artefatos: `h22_vif_perm_per_fold.py`,
`FINDING_H22_VIF_PERM_PER_FOLD.md`, novo `feature_set=h22_per_fold` em
`bakeoff_d1.py`, `outputs/cv_summary_*_h22_per_fold.parquet`. **Lesson
estrutural**: multicolinearidade estatistica != redundancia preditiva; PI
per-fold (nao agregada) e' o wrapper que faltava. **(2) FRAGILIDADE NUMERICA
CHAMPION SE EXPOSTA** (commit `34478f93`): matriz X SE/full tem cond_num
~2.5e17. NMAE 46.6% atual e' "acidental" (sorte amostral). PI absurdos
(>800.000pp NMAE em carga_mwmed/carga_liquida/val_import) sao oscilacao
numerica. **Recomendacao implicita UlFor: SE -> ridge+h22** (~+2pp NMAE
pior que lr_full mas R² empate +0.39 vs +0.38, sem risco numerico).
**(3) ACHADO TEORICO TRANSFERIVEL (H23_ulfor)** (commit `ccf53722`):
leave-one-in mostrou `ter_verif_rmean7` SOZINHA recupera SE LR de R²
-0.448 (H22_drop) para +0.301 (delta +0.381). Mecanismo: **PI medida
com Ridge subestima importance de features colineares** porque L2
redistribui sinal entre `ter_verif_lag1`/`ter_verif`/`ter_prog`; LR sem
shrinkage colapsa quando removidas. **Lesson**: PI deve ser medida com o
modelo final, nao com proxy mais robusto. Aplicavel ao nosso H22 queued
(GBDT vs OLS gap): se PI medida em OLS, gap GBDT pode ficar oculto.
**(4) REGIME ULFOR MUDOU PARA MULTI-AGENTE PARALELO** (checkpoint
`ec0fd937`): "2-3 agentes paralelos ativos, risco duplicar maior que
beneficio acumular sprints" — checkpoints preventivos apos 1 sprint (vs
minimo 3 antes). Padrao: freq recon loop sobe (max 30-60 min vs 2-4h
antes). 6 commits em 15 min reais. **Nenhuma H do loop fechada; nenhuma
nova gerada formalmente** (H31_emergente candidata P3 ~0.5h: replicar
h22_per_fold em holdout 14d real NE+SE antes de Breno decidir promover).
H22_nosso (queued) atratividade SUBIU por convergencia de lesson com
H23_ulfor. Detalhe em `iterations/iter_0021_recon_delta.md`.

**Iter 0020 (H21):** Feature engineering `pdp_residual = pdp_prev_total - gen_renov`
testada como substituto dos 2 canais brutos (pdp_prev_eolica + pdp_prev_solar) — **REFUTADO**.
(iter_0019 paralela fez recon_delta UlFor `515041e1..daf80a6a`; este iter_0020 e' o teste H21.)
OLS full-sample n=486 dias (parquet cacheado de H3 iter_0010, mesmo periodo
2024-12-01..2026-05-01): V_residual_plus_gen (2 feats) PERDE para V_brutos (3 feats) em R²
em **NE -5.1pp** (0.654 vs 0.705), **SE -4.3pp** (0.438 vs 0.481). V_residual_split (gen +
residual_eolica + residual_solar, 3 feats) tambem perde -3pp NE / -4pp SE -- decomposicao
per-fonte importa (curt eolica/solar tem timing diferente). Holdout 80/20 amplifica:
NE V_residual_plus_gen R²_test=0.282 vs V_brutos 0.550 (-27pp adicional). Sanity B1+B2+B4
PASS NE+SE (leak forward-looking, perm p=0.0, residual bate gen-only por +0.31/+0.25).
Protocol identity H3<->H21: R²_gen_only bate exato (|delta|<0.001) nos 3 subs.
**Bonus pdp_prog drop test: NAO_CONFIRMADO** -- pdp_prog adiciona **+5.9pp NE / +9.2pp SE**
em cima dos brutos (vif_max=42/235; contradiz interpretacao H3 univariate "redundante
com gen"; conditional em pdp_prev ainda informa). Mecanismo do REFUTADO H21: OLS sobre
(gen, pdp_prev) ja' tem qualquer combinacao linear de (gen, residual) no seu span --
engineering nao expande basis. **H30 derivada** (P3): replicar H21 em Ridge_alpha10
CV 5x60d UlFor protocolo. **H22** (GBDT vs OLS gap) sobe importancia: se GBDT extrair
interacao nao-linear `gen × pdp_prev`, residual pode ainda valer no champion-class certa.
**LGBM CV SUPPLEMENT 16:50Z** (sessao paralela HYPOTHESIS_TEST): `scripts/h21_pdp_residual_cv.py`
RODADO. CV 5x60d/gap7d em NE/SE/S v3 com 5 feature_sets (A baseline / B additive / C replacement /
D drop pdp_prog / E full simplification). A vence todas em mean MAE em 3/3 cells -- C +1.3 a +3.1%,
B +0.7 a +2.1%, D +3.6 a +8.2%, E +1.8 a +8.0%. B2 perm fold-final: NE [C] +6.6%, SE [C] +10.8%,
S [C] -0.2% -- residual carrega sinal real NE+SE mas e' redundante quando LGBM tem pdp_prev_eolica
+ pdp_prev_solar raw. **OLS + LGBM CV CONVERGEM no verdict REFUTADO**. H22 PARCIALMENTE
pre-respondida (drop pdp_prog prejudicial em GBDT tambem, +3.6/6.9/8.2pp MAE NE/SE/S quando removido).
Outputs em `outputs/iter_0020/h21_pdp_residual_engineered/lgbm_cv_supplement/`. Detalhe completo
(OLS mainline + LGBM supplement) em `iterations/iter_0020_h21_pdp_residual_engineered.md`.

**Iter 0019 (RECON_DELTA):** 6 commits UlFor `515041e1..daf80a6a` absorvidos em
~8 min reais (09:27-09:34 BRT). **Producao 100% inalterada** (loader.py +
MLflow Registry intocados). **(1) FRENTE BIAS_CORRECTION FORMALMENTE FECHADA**:
H14-F (combo `window=60d + threshold k*sigma_bias`, commit `daf80a6a`) **domina
H14-B no papel** em N (mesma media -19.63pp, **zero loses vs 1 lose +0.84pp**),
mas custo de produtizar alto (`sigma_bias_rolling_train` em runtime, ~30-50 LoC
+ cache) acima do ganho marginal (lose evitado abaixo do ruido CV). **NAO
produtizar**. SE/S declarados **intrinsicamente nao-corrigiveis** pela frente
bias (bias bidirecional, oscila em torno de 0 — `FINDING_H14E` commit
`be529e93`). **(2) ACHADO TEORICO TRANSFERIVEL**: razao
`sigma_bias_rolling / sigma_resid` (~1/7 a 1/12 em todos subs) classifica
regimes — **NE bias-dominated** (correcao destrava), **SE/S noise-dominated**
(correcao adiciona ruido), **N intermediario**. Aplicavel ao planejamento H29
emergente (bias_correction sobre H10 ensemble — diagnostico previo per-sub
antes do experimento). **(3) INFRA MAJOR — `mlflow.brazilgrid.com` exposto
via Cloudflare Access** (commit `cd12cf2a`): nginx vhost pronto no EC2
(`proxy_pass 127.0.0.1:5000`), MLflow systemd ja localhost-only, smoke local
OK. **Acao Breno pendente**: 2 passos manuais no dash Cloudflare (DNS A record
+ Zero Trust Access Application). Quando ativo, **DESTRAVA H18** (sanity B1-B6
via MLflow REST direto, elimina dep req-0005 parquet). Docs
`docs/infraestrutura/SUBDOMINIOS.md` + `MLFLOW_CLOUDFLARE_SETUP.md`. Nenhuma
H do loop resolvida; nenhuma nova gerada. Detalhe em
`iterations/iter_0019_recon_delta.md`.

**Iter 0018 (RECON_DELTA):** 6 commits UlFor `5c7963d4..515041e1` absorvidos.
**(1) MUDANCA DE COMPORTAMENTO DE PRODUCAO N** (segundo sub a ganhar bias correction
default ON): commit `515041e1` produtizou bias_correction com **window=60d** em N
per UlFor H14-B (commit `eb9fca05` sweep cross-window). CV 5x60d N@60d: **-19.63pp
NMAE media, wins 3/1/5** vs N@28d MAYBE (3/5 wins, 2/5 loses). Fold-4 N (2025-07/09
seca) dominante (-78pp em 60d, raw 226% — modelo colapsa, bias compensa). Smoke
e2e: N pred 246 → default 267 (bias -21, applied=True; era applied=False iter_0017).
Modelo treinado (`ridge_curt_n_d1@staging v2 clean_plus 31 feat`) **INALTERADO** no
MLflow Registry — wrapper de inferencia agora subtrai bias mean(pred-actual) dos
ultimos 60d causais. **(2) Padrao per-sub adotado**: `BIAS_CORRECTION_WINDOW_BY_SUB
= {NE:28, SE:28, S:28, N:60}` substitui global 28d. **(3) UlFor H14-D
(threshold k*sigma_resid_train) REFUTADA** (commit `4871998c`): NE@k=1 perde
2/5 wins (passa a 1/5); SE/S/N zero wins (threshold filtra demais, apply_rate
~0-10%). Status atual mantido. **(4) Search-space bias correction esgotando**:
janela curta/media/longa (H14-B), threshold sigma (H14-D), CV producao (H14-C)
todos testados. SE/S nao destravam sem **dado novo** — confirma data ceiling
iter_0017. **(5) 3 checkpoints idle** (`0c2f7429/2b262f51/1804c496`) sinalizam
UlFor em pausa aguardando deploy EC2 ou nova frente Breno (25 commits ahead
origin/master). **Nenhuma H do loop fechada; nenhuma nova gerada formalmente
(H29 emergente: bias_correction per-sub sobre H10 ensemble — candidata ALT
proxima iter)**. Detalhe em `iterations/iter_0018_recon_delta.md`.

**Iter 0017 (RECON_DELTA):** 15 commits UlFor `5dacb5a2..5c7963d4` absorvidos.
**(1) MUDANCA DE COMPORTAMENTO DE PRODUCAO NE**: bias_correction rolante-28d
default ON na route `/api/forecast/d1` (commits `41d8d952`+`586eeae6`). 14d real
(2026-05-08..21) NE NMAE **49.2% bate persist 54.1% por -4.9pp** — **primeira
vez no projeto** que ML supera persist em NE. Champion treinado (ridge_curt_ne_d1@v1)
INALTERADO no MLflow Registry; wrapper de inferencia agora subtrai bias mean(pred-actual)
dos ultimos 28d causais. CV 5x60d confirma: -6.90pp NMAE / wins 4/0/5. SE/S/N
default OFF (regime change na janela 14d joga correcao no rumo errado em SE+S;
N raw ja bate persist 36.9pp, bias estrutural negligivel). **(2) req-0007 DONE**
(commit `6b21ffdf`): F1_p50 + per-fold MAE parquet publicados em
`experiments/bakeoff_curtailment_multisub/outputs/cv_summary_per_fold_{full,clean_plus}.parquet`
(140 rows cada). F1_p50 clean_plus headline ridge NE=0.80 vs persist 0.75; lr SE=
TBD (parser p/ proximo iter); ridge N=0.79; S NaN (P50=0 esperado). **(3) Fase 4
100% FECHADA** (commits `13f4a4de`+`8a30a29e`): dashboard MLflow validation D+1
em `vitrine.brazilgrid.com/forecast.html` + rastreabilidade end-to-end no endpoint
(mlflow_url + training_git_sha + inference_git_sha + training_ran_at). **(4) UlFor
internal H verdicts** (NAO confundir com nosso queue): H13 REFUTADA (ridge_S+clean_plus
regride CV+14d); H19 root-cause SE/fold4 = `ter_verif_rmean7` single-feature
restaurou 99% gap; H20 CONFIRMADA clean_plus_v2 SE -11.27pp (mas champions
mantidos em `full`); H18-A REFUTADA (drop universal envenenadoras ridge_S
piora +5.55pp); H21 REFUTADA (lr_SE clean_plus_v2 regride +15.31pp em 14d real).
**Nenhuma H do loop fechada**; nenhuma nova gerada. Detalhe em
`iterations/iter_0017_recon_delta.md`.

**Iter 0016 (H19):** MAE+R²+NMAE dos champions UlFor Ridge/LR extraidos para
leaderboard — **CONFIRMADO_PARCIAL**. Fontes acessiveis sem MLflow tunnel:
(1) `FINDING_RIDGE_BEATS_GBDT.md` parser regex (NMAE+R² mean±std 6 modelos × 4 subs);
(2) `promote_champions.py:CV_METRICS_BY_FS` hard-coded por feature_set (full/clean/clean_plus);
(3) state.json baselines + iter_0013 baseline persist_d1 MAE em MWh.
**MAE em MWh derivado** por `NMAE_champ × (MAE_persist_mwh / NMAE_persist)` →
NE 25.5k / SE 5.6k / S 916 / N 429 MWh. Caveat 1a ordem (~10-15%): ymean varia
entre folds (iter_0014: NE 32k-72k). Consistency check `NMAE_persist FINDING vs
state.json` = OK 4/4 subs (0.445/0.688/1.242/1.007). **F1_p50 NAO DISPONIVEL** —
gap real: bakeoff_d1/promote_champions/validate_d1 nao computam F1 binarizada.
**req-0007 emitido** ao UlFor pedindo (a) F1_p50 logado por fold, (b) dump
parquet do CV_SUMMARY com per-fold MAE em MWh. Sem isto, leaderboard inconsistente
viraria permanente. Detalhe + CSVs em
`outputs/iter_0016/h19_champion_metrics/`.

**Iter 0009 (H16):** B6 zero_count_shift v1.1 — `n_test<30 -> downgrade severity 1 nivel`
e sign_flip exige `|corr_train| >= 0.2 AND |corr_test| >= 0.2` (era >0.05). Falso positivo
SE/v3 lag (iter_0004 com n_test=11) automaticamente atenuado: curt_lag7 sign_flip bloqueado,
severities raw=high downgrade para medium. Replay loop pode rodar em janelas curtas sem
gerar req desnecessario ao UlFor.

**Iter 0010 (H3):** PDP carrega sinal alem de gen via residual — **CONFIRMADO**. OLS decomp
`curt ~ gen + pdp` em n=486 dias contemporaneos: r2_extra(pdp_prev) = +0.31 NE, +0.25 SE,
+0.16 S(train) mas COLAPSA test. perm p=0.0, train/test estavel em NE+SE. pdp_prog quase
redundante com gen (corr 0.95, r2_extra <=0.12). Mecanismo: residual(pdp_prev - gen) e' proxy
de curtailment. Implicacao: manter pdp_prev_*; pdp_prog_* candidato a drop. H21+H22 derivadas.

**Iter 0011 (RECON_DELTA):** 7 commits UlFor `4e0fc7b4..c8df4077` absorvidos. Champions
**PROMOVIDOS** no MLflow Registry (ridge_NE / lr_SE / lr_S @champion; ridge_N @staging) +
**endpoint `/api/forecast/d1` LIVE** servindo D+1 (cold 6.5s, warm <50ms). VIF analise
confirma multicolinearidade massiva (38/55 features VIF>=10, cond_num >1e17), mas drop
universal CLEAN ajuda so NE/N — feature_set=full continua default. VIF greedy iterativo
**REFUTADO** (dropa drivers economicos primarios). lr_N instability isolada em fold 4
(jul-set/2025 = blowup 208%) → ridge_curt_n_d1@staging continua a defesa. Sem novos
requests; sem novas hipoteses do loop geradas. Detalhe em `iterations/iter_0011_recon_delta.md`.

**Iter 0012 (H7):** XGB vs LGBM CV walk-forward (5 folds 60d, gap 7d) sobre features
iter_0002 — **REFUTADO_LGBM_SYSTEMATICALLY_BETTER**. LGBM venceu MAE em **10/12 celulas
(83%)**. iter_0002 NE/v3 XGB R²=+0.52 (n=11) era ruido: em CV NE/v3 XGB R² medio=-0.18
vs LGBM +0.26 — inversao total. Maior gap: NE/v2 ΔMAE=+11.299 MWh em favor LGBM. Vetor
de degradacao: distribution shift (KS p<0.0001 NE+SE entre fold1 e foldN; y_mean cai 3x
em NE), XGB sofre mais por splits mais profundos. LGBM continua modelo padrao do loop.
Champions UlFor sao Ridge/LR (iter_0007) — H7 e' diagnostica do replay loop, nao de
producao. Lesson reforca iter_0006/req-0003 (n=11 falsos positivos). Detalhe em
`iterations/iter_0012_h7_xgb_vs_lgbm_cv.md`.

**Iter 0015 (RECON_DELTA):** 7 commits UlFor `c8df4077..5dacb5a2` absorvidos.
**Champion N atualizado**: ridge_curt_n_d1 v1 (full, 55 feat) → **v2 (clean_plus, 31
feat)** @staging, NMAE 86.3±31.8% → **84.8±26.2%** (-1.5pp mean, -5.6pp std). UlFor
H10 (FEATURE_DROPS_N = CLEAN ∪ {cmo_range, ter_verif_lag1, carga_mwmed_rmean7}) **PARCIALMENTE
CONFIRMADA** (nao confundir com nosso H10 ensemble iter_0013). Endpoint `/api/forecast/d1`
agora feature-set aware (loader le `feature_set` do MLflow run params, backward-compat v1).
**Fase 4 observabilidade FECHADA** pelo UlFor: `validate_d1.py` (replay daily MAE/R²/skill)
+ drift PSI nativo numpy/scipy (dual long/recent, Evidently abandonado por conflito plotly 5/6)
+ Telegram alert 4 gatilhos (`skill<0`, `R²<0`, `psi_recent_max>1.0`, `n_feat_drift>10`) +
Dagster asset `forecast/validate_d1` + schedule 07h BRT **STOPPED** ate Breno gerar
`BRAZILGRID_TELEGRAM_BOT_TOKEN/CHAT_ID`. Smoke confirma `lr_S` em colapso operacional
(NMAE 137-184% em 7-14d, skill -37 a -41%) e drift NE alto (psi_recent_max=11.74,
39/45 features) — reproduz B5 distribution shift documentado iter_0012. Sem novos
requests; sem H do loop resolvida. Detalhe em `iterations/iter_0015_recon_delta.md`.

**Iter 0014 (H11):** LGBM quantile regression para incerteza P10/P50/P90 — **REFUTADO_NE**.
CV walk-forward 5x60d gap 7d sobre features iter_0002 em 12 cells; verdict julgado em NE.
Coverage_band_80 mean: NE **43.6%** (0/3 cells in [70%, 90%]) vs 80% nominal —
under-coverage sistemico em todas as 4 subs (SE 45.4%, S 52.1%, N 47.1%). Causa raiz:
LGBM nao modela heteroscedasticidade + distribution shift (iter_0012 KS p<0.0001).
P50 magnitude OK em NE (delta +4.5% vs LGB-mean) mas banda inutil para "P90 conservador"
do operador. **Bonus inesperado**: P50 quantile BATE LGB-mean em magnitude em N (-18%),
S (-14%), SE (-1.6%) — mediana mais robusta que mean em distribuicoes com cauda longa
de zeros. H26 (conformal post-hoc), H27 (P50 substituto, custo zero) e H28 (NGBoost)
derivadas. Detalhe em `iterations/iter_0014_h11_quantile_regression_ne.md`.

**Iter 0013 (H10):** Ensemble LGBM + persist_d1 com pesos analiticos derivados de
inner_val 30d (zero leak) **CONFIRMADO_NE_SE + bonus N**. CV walk-forward 5x60d gap 7d,
4 esquemas (equal / inv_mae / inv_mse / opt_alpha). Best ensemble bate LGB-only em
maioria das folds em: **NE 3/3 cells** (-12.996 MWh em v1 = 28%, -4.188 em v2 = 13%,
-4.065 em v3 = 13%), **SE 3/3 cells** (-565..-293 MWh = 4-7%), **N 3/3 cells** (-100
MWh = 17% — bonus, persist forte em N como H10 previa), S 2/3 (v3 +26 MWh irrelevante).
Esquemas vencedores: ens_inv_mae 6 cells, ens_inv_mse 4, ens_equal 2, ens_opt_alpha 0
(sobre-otimiza). Pesos analiticos > grid-search empirico. alpha_opt varia 0.0-1.0
entre folds confirmando adaptacao a regime (NE/v1 fold 4 alpha=0.00 = pura persist
quando LGB MAE=43k vs PER=33k). R² NE/v2 sobe 0.22->0.41; N/v1 -0.33->+0.01. H24
(ensemble sobre champions Ridge/LR) + H25 (stacker Ridge meta-modelo) derivadas a
queue. Detalhe em `iterations/iter_0013_h10_ensemble_v2_persist.md`.

| layer | alvo | sub | baseline (MAE_mwh, CV) | best_metric (MAE/R²/F1, modelo) | NMAE secundario | last_iter | sanity_ok | data_utc |
|---|---|---|---|---|---|---|---|---|
| curtailment | d1_ENE_CNF | NE | persist_d1 MAE≈33.7k MWh (CV 5x60d, NMAE 44.5%) | **ridge_curt_ne_d1 @champion + bias_corr_28d (PROD default ON desde iter_0017) — MAE 27.3k±11.9k MWh (parquet) / R² +0.469±0.098 / F1_p50 0.808±0.170** (req-0007 closed iter_0017); in-sample R²=0.830; **14d real corrected NMAE 49.2% bate persist 54.1% por -4.9pp — PRIMEIRA VEZ no projeto**; **iter_0021: candidato sucessor `ridge+h22_per_fold` (UlFor commit `10fa56d3`) CV 31.4% NMAE (-2.3pp) / R² +0.543 (+0.074) — aguarda decisao Breno + holdout 14d real (H31 emergente)** | NMAE 33.7±8.1% CV; raw 14d 59.2% / corrected 49.2%; sucessor h22 NMAE CV 31.4±?% | 0021 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE c/ bias_correction_mw exposto** | 2026-05-24T17:30Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 MAE≈8.3k MWh (CV 5x60d, NMAE 68.8%) | **lr_curt_se_d1 @champion — MAE 6.1k±0.9k MWh (parquet) / R² +0.383±0.094 / F1_p50 0.785±0.108** (req-0007 closed iter_0017); in-sample R²=0.619; **UlFor H14-C decidiu NAO produtizar bias_correction** (regime change Mai/26 chuvoso->seco joga bias no rumo errado, +1.61pp 14d real); **UlFor H21 REFUTADA** (clean_plus_v2 regride +15.31pp em 14d real); **teto-de-dados D+1 estendido NE->SE: NENHUM ML bate persist em 14d real**; **iter_0021: FRAGILIDADE NUMERICA EXPOSTA** (commit `34478f93` — matriz X SE/full cond_num ~2.5e17, NMAE 46.6% e "acidental"); **candidato sucessor `ridge+h22_per_fold` (commit `10fa56d3`) CV 48.3% NMAE / R² +0.390 — recomendacao implicita UlFor: trocar familia lr→ridge** (essencialmente empate R² sem risco numerico). Aguarda decisao Breno + holdout 14d real (H31 emergente) | NMAE 46.6±13.4% CV (lr fragil); sucessor ridge+h22 NMAE CV 48.3±14.2% | 0021 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE; champion atual numericamente fragil** | 2026-05-24T17:30Z |
| curtailment | d1_ENE_CNF | S | persist_d1 MAE≈1.27k MWh (CV 5x60d, NMAE 124.2%) | **lr_curt_s_d1 @champion — MAE 805±441 MWh (parquet) / R² +0.371±0.164 / F1_p50 NaN** (P50_train=0 — sub com muitos zeros, esperado per spec req-0007); in-sample R²=0.725 FRAGIL (validate_d1 7-14d skill -37 a -41%); **UlFor H13 REFUTADA** (ridge_S+clean_plus regride CV+14d); **UlFor H18 ABERTA** (S underperforma persist estruturalmente em 2026-05); **UlFor H14-C NAO produtizou bias_correction** (+10.81pp 14d real, ymean ~32 MWh amplifica ruido) | NMAE 89.6±31.2% (CV ymean≈1k MWh > EPS=1 → safe; iter_0008 unsafe era replay n=11) | 0017 | aud B1-B6 pendente (H18) — **endpoint /api/forecast/d1 LIVE** | 2026-05-24T14:30Z |
| curtailment | d1_ENE_CNF | N | persist_d1 MAE≈0.51k MWh (CV 5x60d, NMAE 100.7%) | **ridge_curt_n_d1 v2 @staging + bias_corr_60d (PROD default ON desde iter_0018)** — MAE 425±149 MWh (parquet) / R² +0.170±0.185 / F1_p50 0.790±0.048 (req-0007 closed iter_0017; vs persist 0.72) (clean_plus, 31 feat; in-sample R²=0.472); FRAGIL atenuado vs v1 (era MAE≈440 MWh / R² +0.196±0.289); **UlFor H14-B PROMOVEU bias_correction com window=60d** (CV 5x60d: -19.63pp NMAE media, wins 3/1/5; fold-4 seca-2025Q3 dominante, -78pp em 60d raw 226%); smoke e2e: pred 246 → default 267 (bias -21, applied=True) | NMAE 84.8±26.2% | 0018 | nao promovivel ainda (champion @staging) — **endpoint /api/forecast/d1 LIVE c/ bias_correction_mw exposto + applied_in_default=True** | 2026-05-24T15:30Z |
| meta | metric_suite | — | NMAE (Principio 6 violado) | **MAE/R²/F1 primario + NMAE secundario com flag** | 3/4 subs (NE,SE,N) conflict NMAE↔R²/F1 em iter_0002 replay; S NMAE unsafe | 0008 | H9 CONFIRMADO | 2026-05-24T06:00Z |
| curtailment | feat_pdp_residual | NE+SE | V_brutos OLS (gen + pdp_prev_e + pdp_prev_s) | **V_residual_plus_gen REFUTADO** -- R² OLS in-sample NE -5.1pp (0.654 vs 0.705) / SE -4.3pp (0.438 vs 0.481); holdout 80/20 amplifica NE -27pp adicional; V_residual_split (per-fonte) tambem perde -3/-4pp. **Bonus pdp_prog drop: NAO_CONFIRMADO** (prog adds +5.9pp NE / +9.2pp SE conditional em brutos). H30 derivada para Ridge CV protocolo UlFor. | n/a (R² metric, sem MAE-pp comparavel a champions) | 0020 | leak/perm/baseline PASS; holdout 80/20 PASS (mostra REFUTADO ainda mais forte test) | 2026-05-24T16:45Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | NE | persist_d1 | NMAE 35.7% xgb UlFor v3.3 (superseded por ridge_alpha10) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | SE | persist_d1 | NMAE 46.0% xgb UlFor v3.3 (superseded por lr) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | S | persist_d1 | NMAE 109% xgb UlFor v3.3 (superseded por lr -19.4pp) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF (DEPRECATED) | N | persist_d1 | NMAE 72.2% lgbm UlFor v3.3 single fold (CV mostra 99.5±37%) | superseded iter_0007 | 0006 | — | 2026-05-24T05:00Z |
| curtailment | d1_ENE_CNF | NE | persist_d1 (loop replay n=11) | NMAE 28.2% v2 LGBM original / 31.7% holdout strict | superado por UlFor v3.3 | 0002 | [4/5] | 2026-05-24T01:30Z |
| curtailment | d1_ENE_CNF | SE | persist_d1 (loop replay n=11) | NMAE 36.2% v2 LGBM | superado por UlFor v3.3 | 0002 | [3/5] B3 fail | 2026-05-24T01:30Z |
| meta | h2_off_by_one_pdp | — | dbt_join_correto | corr(PDP[t], gen[t])=0.9118 / corr(PDP[t], gen[t+1])=0.8211 | H2 REFUTADO | 0003 | n=484 dias | 2026-05-24T02:30Z |
| meta | sanity_check_B6 | — | n/a | zero_count_shift + signal_collapse implementado e validado (sintetico high+collapse, real SE/v3 lag sign-flip) | adicionado a default pipeline | 0004 | passou | 2026-05-24T03:00Z |
| meta | sanity_check_B6_v1.1 | — | B6 v1.0 (overconfident em n_test=11) | **B6 + n_test<30 downgrade + sign_flip gate \|corr\|>=0.2** | 5/20 FP sint n=10 / 0 perdas n=60 / SE/v3 lag iter_0002 downgrade high->medium | 0009 | regression + revalidation OK | 2026-05-24T06:45Z |
| meta | h3_pdp_residual_signal | NE | r2_gen_only=0.346 | **r2_gen+pdp_prev=0.654 (r2_extra +0.308; partial_corr +0.69)** | perm p=0.0; test/train delta=+0.001 (estavel); leak ok | 0010 | H3 CONFIRMADO | 2026-05-24T07:30Z |
| meta | h3_pdp_residual_signal | SE | r2_gen_only=0.187 | r2_gen+pdp_prev=0.438 (r2_extra +0.251; partial_corr +0.56) | perm p=0.0; test/train delta=+0.002; leak ok | 0010 | H3 CONFIRMADO | 2026-05-24T07:30Z |
| meta | h3_pdp_residual_signal | S | r2_gen_only=0.053 | r2_gen+pdp_prev=0.210 (r2_extra +0.157 train; +0.001 test!) | perm p=0.0; **dist_shift FAIL** (test colapsa, cobertura 12 usinas) | 0010 | H3 fragil em S | 2026-05-24T07:30Z |
| meta | h7_xgb_vs_lgbm_cv | — | persist_d1 por fold | **LGBM > XGB em 10/12 cells (83%) CV 5x60d gap7d**; NE/v3 R² LGBM +0.257 vs XGB -0.178 (inverteu iter_0002 n=11) | NE/v2 ΔMAE+11.299 MWh, SE/v1 wins XGB so 4/5 mag -210 MWh = 2.8% (irrelevante) | 0012 | H7 REFUTADO; [3/3 sanity B3+B4+B5] | 2026-05-24T09:30Z |
| curtailment | ensemble_lgb_persist_d1 | NE | LGBM-only (CV 5x60d) MAE 32.2k v2 | **best=ens_inv_mae MAE 28.0k v2 (-13%), -28% em v1, -13% em v3**; R² NE/v2 0.22->0.41 | NE/v1 LGB catastrofico 46k corrigido por persist (alpha=0.00 fold 4) | 0013 | H10 CONFIRMADO; [B3+B4+B5 done; B1 inherit; B2/B6 N/A] | 2026-05-24T10:30Z |
| curtailment | ensemble_lgb_persist_d1 | SE | LGBM-only (CV 5x60d) MAE 7.74k v2 | best=ens_inv_mae MAE 7.21k v2 (-7%); -7% v1, -4% v3 | wins 3-4/5 folds; magnitude clinica modesta mas consistente | 0013 | H10 CONFIRMADO | 2026-05-24T10:30Z |
| curtailment | ensemble_lgb_persist_d1 | S  | LGBM-only (CV 5x60d) MAE 1.06k v2 | best=ens_inv_mse MAE 1.06k (~0%); v1 -5%, v3 +3% (irrelevante) | regime instavel; alpha varia 0.00-1.00 entre folds | 0013 | H10 PARCIAL (2/3 confirming) | 2026-05-24T10:30Z |
| curtailment | ensemble_lgb_persist_d1 | N  | LGBM-only (CV 5x60d) MAE 0.58k v1-v2 | **best=ens_inv_mae MAE 0.48k (-17%)**; R² LGB -0.33 -> ens +0.01 | bonus H10 — persist forte em N (skill_LGB_vs_persist negativo); ensemble corrige | 0013 | H10 CONFIRMADO (bonus) | 2026-05-24T10:30Z |

---

## Legenda

- **baseline**: tipo do baseline + tamanho do test set onde foi medido
- **best_metric**: melhor metrica conhecida (modelo + janela)
- **delta_vs_baseline**: skill score `1 - MAE_model/MAE_baseline`. Positivo = melhor que baseline.
- **last_iter**: iter que produziu o numero atual
- **sanity_ok**: `[n/5]` = quantos sanity checks (B1 leak, B2 PI, B3 holdout strict, B4 baseline, B5 dist shift) passaram limpo. WARN = parcial.
- **data_utc**: ISO 8601 UTC do registro

## Notas

- **Test n=11 dias** no replay (limite imposto pela staleness do CH local em feat_intercambio + feat_carga + feat_termico). UlFor roda em ambiente com test n=60d.
- **NE/v3 nao-promovivel** ainda (iter_0002): regressao vs v2 por distribution shift PDP, NAO por off-by-one (confirmado iter_0003).
- **SE/v3 ganho aparente NAO sobrevive holdout estrito** (+19.2pp NMAE com gap 7d).
- **N permanece nao-aprendivel** com janela atual: ML NMAE 97% > persist_d1 34%.

## Iter 0003 — H2 validation

Iter 0003 nao produziu novo modelo. Validou H2 (off-by-one PDP) empiricamente:
correlacao(PDP_prev[t], gen[t]) = 0.9118 > correlacao(PDP_prev[t], gen[t+1]) =
0.8211. **H2 REFUTADO.** dbt model + bakeoff JOINs estao corretos.

Causa real do "shift -1 melhora NMAE" do iter_0002: provavelmente ruido
amostral (n=11) — request req-0001 enviado ao UlFor para confirmar com
n>=60d.

## Iter 0003 — Canal loop->UlFor

Estabelecido em commit UlFor `1fb2bb50`. Primeiros requests:
- req-0001 (P1 investigation): validar "shift -1 e ruido" com n>=60d
- req-0002 (P0 feature_fix): forward-fill em vez de coalesce(0) para gaps PDP Apr/2026

## Iter 0004 — B6 zero_count_shift + req-0003

- B6 implementado em sanity_checks/zero_count_shift.py + integrado a __init__.py
- Validacao sintetica: caso canonico coalesce(0) flagged severity=high + signal_collapse=magnitude_collapse
- Validacao real (snapshot iter_0002 v3): caught lag-feature sign-flip em SE/v3 e NE/v3:
  - SE/v3: curt_lag1 +0.252->-0.273, curt_lag7 +0.196->-0.146, ger_eolica +0.277->-0.291 (TODOS sign flip)
  - NE/v3: curt_lag1 +0.664->-0.229, curt_lag7 +0.415->-0.341 (sign flip)
- B6 e diagnostico da CAUSA RAIZ do "SE colapso strict +19.2pp" do iter_0002
- req-0003 enviado ao UlFor (P1 investigation, evidencia B6): commit `9c1484e2`

## Iter 0005 — Self-planning upgrade

- Loop opera em modo self-planning a partir desta iter
- 4 componentes novos: hypotheses_queue.md (15 H), planner.py, quality_gate.py, status.sh
- run.sh reescrito como continuous loop com budget caps + --dry-run
- Smoke test (3 iters dry-run): planner seleciona H9 (P1), bridge sync funcionou
- UlFor processou TODAS 3 requests durante esta iter (be2c9186 patch + b7acfdcd PDP fix)
- Loop continuo NAO INICIADO — aguarda autorizacao manual do Breno apos revisao

## Iter 0006 — Recon delta UlFor (numbers oficiais v3.3 absorved)

UlFor v3.3 sub-level oficial pos PDP gap fix (test 2026-03-23 -> 2026-05-21, n=60d, gap 7d):

| sub | NMAE pre-fix | NMAE pos-fix | R² pre | R² pos | observacao |
|---|---|---|---|---|---|
| NE | 36.0% | **35.7%** | +0.375 | **+0.402** | +2.7pp var explicada |
| SE | 46.2% | **46.0%** | +0.373 | **+0.386** | bias -2017 -> -1896 |
| S | 117% | **109%** | -0.133 | -0.135 | **ML quebra teto persist=113.7%!** |
| N | 72.2% | 72.2% | +0.022 | +0.041 | lgbm slight melhor |

Decisoes oficiais UlFor:
- **SE/v3 PROMOVIVEL para FASE 4** (req-0003 verdict)
- **S agora aprendivel como regressor** (era nao_aprendivel)
- Lags por sub: NE=KEEP, S=KEEP (ajudam), N=REMOVE (atrapalham), SE=indiferente

B6 lessons learned (req-0001 + req-0003 responses):
- B6 com test n=11 deu falso positivo (sign-flip em SE) que n=60d nao confirma
- Causa raiz iter_0002 NE delta -4.2pp era amostragem (14/16 deltas <2pp com n=60)
- H16 nova (P1): ajustar B6 para downgrade severity quando n_test < 30

Hipoteses fechadas nesta iter: H6 (B6 sign-flip refutado por UlFor)
Hipoteses adicionadas: H16 (B6 threshold-by-n), H17 (P0 promover SE/S v3.3 a FASE 4)

## Iter 0008 — H9 metric_suite MAE/R²/F1 substitui NMAE como primaria

PLANO_FINAL UlFor Principio 6 adotado: NMAE rebaixada a secundaria.

Resultados ao aplicar metric_suite sobre iter_0002 LGBM replay (n_test=11):

| sub/ver | MAE | R² | F1_p50 | NMAE | NMAE_safe |
|---|---|---|---|---|---|
| NE/v1 | 14862 | -0.167 | 0.000 | 0.501 | true |
| NE/v2 |  8367 | +0.660 | 0.000 | 0.282 | true |
| NE/v3 | 11830 | -0.002 | 0.000 | 0.399 | true |
| SE/v1 |  3599 | +0.467 | 0.857 | 0.409 | true |
| SE/v2 |  3180 | +0.394 | 1.000 | 0.362 | true |
| SE/v3 |  3406 | +0.544 | 0.857 | 0.387 | true |
| S/v1  |    22 |   NaN  | 0.000 |  -    | **false** (ymean<1) |
| S/v2  |   125 |   NaN  | 0.000 |  -    | **false** |
| S/v3  |    80 |   NaN  | 0.000 |  -    | **false** |
| N/v1  |   269 | -1.402 | 0.800 | 1.011 | true |
| N/v2  |   269 | -1.402 | 0.800 | 1.011 | true |
| N/v3  |   259 | -1.637 | 0.750 | 0.974 | true |

**Conflito de ranking detectado em 3/4 subs** (best-per-sub diverge entre NMAE e R²/F1):

| sub | best por NMAE | best por MAE | best por R² | best por F1 |
|---|---|---|---|---|
| NE | v2 | v2 | v2 | **v1** (sobe!) |
| SE | v2 | v2 | **v3** | v2 |
| S  | unsafe | v1 | n/a | tie zero |
| N  | v3 | v3 | **v1** | **v1** |

Decisao: H9 CONFIRMADO. Ranking unico-criterio (NMAE) viesa inferencia.
Suite MAE+R²+F1 primaria (e.g., NE/v2 vence em magnitude mas NE/v1 captura
melhor eventos high-curt; SE/v3 melhor em variancia explicada mas SE/v2 em
event detection). NMAE secundaria com flag `unsafe` quando ymean<1 MWh —
elimina ruido reportado de "S NMAE 109%" que era artefato de denominador
baixo (test n=11 ymean<1).

Patches:
- `scripts/metric_suite.py` — canonical `compute(y_true, y_pred, y_train)` +
  `format_table()` + standalone runner sobre iter_0002.
- `sanity_checks/baseline_compare.py` (B4) — emite `metric_suite_lgbm` +
  `metric_suite_climatologia_doy`. Skill score mantido (back-compat).
- `sanity_checks/holdout_temporal_strict.py` (B3) — emite
  `metric_suite_strict` + `delta_mae_strict_minus_original` ao lado dos
  campos legados. Import limpa; LGBMRegressor run depende de sklearn no env.

Hipotese derivada criada: **H19** (P2) — extrair MAE+R²+F1 do bakeoff
oficial UlFor (mlflow tabela) para refletir na linha "best_metric"
do leaderboard. Hoje so' NMAE oficial e' conhecida — MAE/R²/F1 dos
champions Ridge/LR estao no MLflow mas nao no checkpoint do loop.
Requer req-0007 ao UlFor ou parse direto do MLflow proxy file.

H10 e H11 estavam bloqueadas em H9 — agora unblocked.

## Iter 0007 — H17 SUPERSEDED + champions Ridge/LR absorved

UlFor self-actionou entre iter_0006 e iter_0007 (~5h, sem novo req do loop):

- commit `76732289` (2026-05-23 22:59): Ridge baseline-controle BATE XGB em NE/S
- commit `4e0fc7b4` (2026-05-24 02:15Z): CV walk-forward 5 folds confirma

Champions per-sub mudaram em 3/4 subs (mean ± std, CV 5 folds 60d):

| sub | champion antes (iter_0006) | champion agora (iter_0007 CV) | delta NMAE | delta R² |
|---|---|---|---|---|
| NE | xgb 35.7% / +0.402 | **ridge_alpha10 33.7±8.1% / +0.469±0.098** | -2.0pp | +0.067 |
| SE | xgb 46.0% / +0.386 | **lr_sklearn 46.6±13.4% / +0.380±0.139** | +0.6pp (tied) | -0.006 |
| S  | xgb 109% / -0.135 | **lr_sklearn 89.6±31.2% / +0.447±0.202** | **-19.4pp** | **+0.58 absoluto!** |
| N  | lgbm 72.2% / +0.041 | ridge_alpha10 86.3±31.8% / +0.196±0.289 FRAGIL | +14.1pp | +0.155 |

H17 (P0 promover XGB SE/S a FASE 4) -> **SUPERSEDED_BY_ULFOR_RIDGE_LR_CV**:
premissa XGB invalidada, mas a INTENCAO (promover algo a FASE 4) e' valida —
champions Ridge/LR sao os candidatos reais. Loop NAO emite req-0004 pois
UlFor JA executa @champion registry plan + OOT 2x agendado (autopilot).

Lessons:
- Queue e snapshot temporal. UlFor pode resolver em sessao paralela entre iters.
- Recon-style absorve em vez de duplicar trabalho.
- Single fold n=60d (iter_0006) overestima xgb e subestima lr — CV 5 folds
  e ground truth oficial pos-iter_0007.

H18 nova (P1 methodology, blocked por req-0005): auditar champions Ridge/LR
via B1-B6 antes de FASE 4 final.

## Iter 0009 — H16 B6 robustez n_test (CONFIRMADO)

Patch sanity_checks/zero_count_shift.py: dois mitigations ortogonais.

**(1) `n_test < 30 -> downgrade severity 1 nivel`** (high->medium->low->none).
Severity original preservada em `severity_raw`; flag novo
`downgraded_due_to_small_n_test` marca downgrades para audit.

**(2) `sign_flip` exige `|corr_train| >= 0.2 AND |corr_test| >= 0.2`**.
Gate antigo era `>0.05` em ambos — pega ruido amostral em n=11. `corr_test`
em [0.05, 0.20] com n=11 nao discrimina sinal real de ruido. Flag novo
`sign_flip_blocked_by_min_abs_corr` documenta o que o gate antigo teria
flaggado e o novo bloqueia.

**Regression test sintetico** (`scripts/b6_regression_n_test_threshold.py`):
DGP fraco-positivo (lag1 ~ AR(1), corr alvo ~ +0.25), n_train=365, 20 seeds.

| cenario | old_sign_flip | new_sign_flip | new_downgraded | criterio aceite |
|---|---|---|---|---|
| n_test=10 (small) | 5/20 | 0/20 | 20/20 | 5 FP eliminados |
| n_test=60 (large) | 0/20 | 0/20 | 0/20 | zero regressao em VP |

Todos 4 criterios de aceitacao passam -> verdict **CONFIRMADO**.

**Revalidation iter_0002 runs** (snapshot da causa pratica que motivou H16):

| sub/ver (n_test=11) | n_high | n_med | n_collapse | n_downgrade | sf_blocked |
|---|---|---|---|---|---|
| NE/v1 | 0 (era 0) | 0 (era 4) | 2 | 4 | 0 |
| NE/v3 | 0 (era 0) | 0 (era ~4) | 2 | 8 | 0 |
| SE/v3 | 0 (era 0) | 4 (era 4) | 3 (era 4) | 9 | 1 (curt_lag7) |
| N/v3  | 0 | 0 | 1 | 5 | 3 |

- SE/v3: features raw=high downgrade para medium (preserva sinal mas atenua alarme);
  curt_lag7 sign_flip bloqueado (`ct=0.196 < 0.2`). UlFor req-0003 ja' tinha
  confirmado que esses flips eram amostrais em n=60 — patch agora alinha o
  diagnostico do loop com aquela verdade.
- NE/v1-v3: lag sign_flips PERSISTEM (`|ct|=0.66/0.42`, ambos >>0.2) — gate
  novo nao mascara warnings legitimos onde a correlacao train e' forte.
- N/v3: 3 sign_flips bloqueados — sub com mais falso positivo amostral.

H20 derivada (P3): expor n_test no leaderboard + flag `low_confidence_n_test` no
bake-off runner.

### Lessons learned (cumulativas)

- B6 v1.0 (iter_0004): util mas overconfident em janelas curtas. Gera req
  externa de baixa-confianca.
- B6 v1.1 (iter_0009): downgrade calibrado + gate de sign_flip alinhado com
  corr de UlFor (n>=60). Loop agora pode rodar B6 em replay n=11 sem ruido.
- Padrao geral: sanity checks devem expor `n_test` e ajustar severities por
  potencia estatistica — replica-se em B5 (PSI), B1 (leak corr).

## Iter 0010 — H3 PDP residual signal (CONFIRMADO)

Hipotese H3 testada com OLS contemporaneo `curt ~ gen + PDP` em n=486 dias
(2024-12-01 -> 2026-05-01, sub NE/SE/S; N excluido por cobertura PDP local
zero). Crosswalk inline via mapeamento_conjunto_pdp -> obt_conjunto (~208/606
usinas mapeadas, vs UlFor 82% de 606).

### Resultados por (sub, variant)

| sub | variante | r2_gen | r2_gen+pdp | **r2_extra** | partial_corr | perm p | train→test delta |
|---|---|---|---|---|---|---|---|
| **NE** | pdp_prev | 0.346 | 0.654 | **+0.308** | +0.686 | 0.000 | +0.001 (estavel) |
| NE | pdp_prog | 0.346 | 0.460 | +0.115 | +0.418 | 0.000 | +0.052 |
| **SE** | pdp_prev | 0.187 | 0.438 | **+0.251** | +0.556 | 0.000 | +0.002 (estavel) |
| SE | pdp_prog | 0.187 | 0.192 | +0.005 | -0.074 | 0.098 | +0.000 |
| S  | pdp_prev | 0.053 | 0.210 | +0.157 | +0.408 | 0.000 | **-0.182** (fragil!) |
| S  | pdp_prog | 0.053 | 0.083 | +0.030 | +0.178 | 0.000 | -0.027 |

### Sanity checks (queue requeridos: leak, perm, dist_shift)

- **leak**: PASS. corr(pdp[t], curt[t]) > corr(pdp[t], curt[t-1]) em 5/6
  combinacoes. PDP e' forward-looking, nao back-cast (publicado D-1 -> safe).
- **perm** (500 shuffles): 5/6 com p=0.0 (observado MUITO acima do p99 null
  ~0.005-0.016). Apenas SE/pdp_prog com p=0.098 (mas r2_extra so 0.005,
  irrelevante).
- **dist_shift** (split 80/20 temporal): NE+SE estaveis (|delta|<0.003).
  **S falha** (test r2_extra colapsa de 0.18 para 0.001) — regime change
  curt-S baixo no Dez/2025-Mai/2026 + cobertura PDP-S so 12 usinas (todas
  eolicas, nenhuma solar).

### Interpretacao tecnica

Separacao PDP_prev vs PDP_prog e' a chave:

- **PDP_prog tracking-very-tight de gen** (corr 0.95 NE, 0.75 SE, 0.93 S).
  Sinal redundante com geracao realizada. Programado D-1 ja' incorpora
  dispatch real; mudou pouco apos a operacao.
- **PDP_prev e previsao independente** (publicada D-1), recurso esperado.
  A discrepancia `pdp_prev - gen` correlaciona com curtailment porque
  gen = previsao - restricao. residual e' proxy direta de curt.

### Decisao para o modelo

- **Manter `pdp_prev_eolica_mwh`, `pdp_prev_solar_mwh`** (NE+SE definitivo,
  S condicional). Valor agregado robusto.
- **Considerar dropar `pdp_prog_eolica_mwh`, `pdp_prog_solar_mwh`** —
  colinearidade ~0.95 com `ger_*_mwh`, sinal residual <=0.12.

### Follow-ups

- **H21 (P2 feature)**: engineering `pdp_residual_mwh = pdp_prev - gen` como
  1 canal denso (vs 2 brutos), sanity B1 leak + B2 perm + B4 baseline.
- **H22 (P3 model)**: GBDT-only `curt ~ gen + pdp_prev` vs OLS para medir
  gap nao-linear. Se GBDT gap >= 5pp R², ha interacoes que justificam
  manter features brutas em vez de engineering.

Sem req externo necessario. Crosswalk + parser inline funciona; tabela
`feat_pdp_renovavel` no UlFor (cobertura 82%) ja' faz a agregacao
materializada — H21/H22 podem rodar la' diretamente com mais cobertura.

## Iter 0011 — RECON_DELTA UlFor (4e0fc7b4 -> c8df4077)

7 commits absorvidos. Champions de candidates do iter_0007 viraram
producao real:

| commit | acao | impacto |
|---|---|---|
| `d1fe9777` | CV walk-forward 5x60d (impl) | numeros oficiais iter_0007 materializados; MLflow 140 runs + 28 CV_SUMMARY |
| `83bc79c2` | promote_champions MLflow | ridge_NE/lr_SE/lr_S @champion, ridge_N @staging |
| `3ac5916a` | VIF + CLEAN bake-off | H4 ulfor CONFIRMADA estrutural; CLEAN ajuda NE/N, HURTS SE/S (R² S cai 0.35); FULL default |
| `4d6dd73a` | checkpoint marker 04:00Z | — |
| `d2bf38e4` | VIF greedy iterativo | H8 ulfor REFUTADA — multicolin estat != redundancia preditiva |
| `a7edb1ef` | endpoint /api/forecast/d1 | **PRODUCAO LIVE** — cold 6.5s, warm <50ms, cache 1h modelo + 15min predicao |
| `c8df4077` | investigate_lr_N instability | fold 4 (jul-set/2025) = blowup 208% FULL -> 128% CLEAN; envenenadoras: cmo_range, taxa_penetracao, ter_verif_lag1, carga_mwmed_rmean7; ridge_curt_n_d1@staging continua a defesa |

### O que mudou na nossa interpretacao

- **Champions agora estao em PRODUCAO** (nao mais "candidate aguardando OOT 2x").
  MLflow Registry com aliases setados. Endpoint /api/forecast/d1 servindo
  os 4 subs. linhas `curtailment | d1_ENE_CNF | <sub>` da tabela top
  atualizadas para refletir status PROMOVIDO + in-sample R² + nota LIVE.
- **VIF nao destrava drop universal**. Multicolinearidade massiva confirmada
  (38/55 features VIF>=10, cond_num >1e17), mas drop padrao CLEAN ajuda
  so NE+N. feature_set=full continua default. CLEAN candidato Staging
  NE proximo round (potencial robustez +1pp stdev / NMAE -0.5pp).
- **lr_N instabilidade isolada por fold**: fold 4 (jul-set/2025, train=221d,
  mais antigo) puxa stdev=65.4%. CLEAN reduz stdev 52.4->22.7pp.
  Envenenadoras especificas N: cmo_range, taxa_penetracao, ter_verif_lag1,
  carga_mwmed_rmean7. Em CLEAN ainda aparecem curt_lag1/curt_rmean7
  instaveis (N tem muitos zeros estruturais -> OLS extrapola mal).
  ridge_curt_n_d1@staging continua. Possivel `feature_set=clean_plus_n`
  proximo round.

### Reqs / hipoteses

- Sem novos requests pendentes. Os 3 antigos (req-0001/2/3) continuam DONE.
- Sem novas hipoteses do **loop** geradas. UlFor numera seus proprios H1/H2/H4/H8/H9
  no PLANO_FINAL — **nao confundir**. Em particular: commit `c8df4077` diz
  "H9 RESPONDIDA" referindo-se ao H9 ulfor (lr_N), **nao** ao nosso H9
  (metric_suite MAE/R²/F1, iter_0008 CONFIRMADO).
- Status H18 (sanity B1-B6 sobre champions Ridge/LR) ainda blocked por
  req-0005 — sanity local requer ou (a) UlFor publicar predicoes em
  parquet acessivel ou (b) loop ganhar acesso ao MLflow tracking URI.
  Candidato a req-0004 (dump MLflow CV_SUMMARY) **nao** emitido nesta
  iter — auto-pesado, evitar duplicar trabalho ja resumido em FINDING.

### Proxima iter

`iter_0012` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`). Derivada de H3 iter_0010, codavel
localmente, sem dep externa. Alt: H10, H22, H19.

## Iter 0013 — H10 Ensemble LGBM + persist_d1 (CONFIRMADO_NE_SE)

Hipotese H10 (P2): ensemble simples LGBM + persist_d1 com pesos derivados de
skill em CV pode dominar LGBM puro em subs onde persistencia carrega muito
sinal. Alvo explicito do queue = NE+SE; detail H10 destaca persist forte em N.

CV walk-forward 5 folds (60d cada, gap 7d) sobre features iter_0002 nas 12
cells (4 subs x 3 vers). Inner split adicional: ultimos 30d do train viram
inner_val para derivar pesos do ensemble SEM leak de test. 4 esquemas de peso:
`ens_equal`, `ens_inv_mae` (w∝1/MAE), `ens_inv_mse` (BMA Gaussian),
`ens_opt_alpha` (grid 0..1 step 0.05 argmin MAE inner_val).

### Resultados (best ensemble vs LGB-only, deltas MAE em MWh; negativo=ganho)

| cell  | LGB MAE | PER MAE | best ens MAE | best_scheme    | delta vs LGB | wins  | R² LGB | R² best |
|-------|--------:|--------:|-------------:|----------------|-------------:|-------|-------:|--------:|
| NE/v1 |  45.959 |  33.722 |       32.963 | ens_inv_mse    |     -12.996  | 5/5   | -0.634 | +0.187  |
| NE/v2 |  32.181 |  33.837 |       27.993 | ens_inv_mae    |      -4.188  | 4/5   | +0.217 | +0.412  |
| NE/v3 |  31.593 |  33.837 |       27.528 | ens_inv_mae    |      -4.065  | 3/5   | +0.257 | +0.423  |
| SE/v1 |   7.752 |   8.328 |        7.186 | ens_equal      |        -565  | 3/5   | -0.135 | -0.001  |
| SE/v2 |   7.736 |   8.328 |        7.211 | ens_inv_mae    |        -525  | 4/5   | -0.081 | +0.004  |
| SE/v3 |   7.222 |   8.328 |        6.929 | ens_inv_mse    |        -293  | 3/5   | +0.074 | +0.104  |
| S/v1  |   1.157 |   1.266 |        1.104 | ens_equal      |         -53  | 3/5   | -0.291 | -0.151  |
| S/v2  |   1.064 |   1.268 |        1.058 | ens_inv_mse    |          -6  | 3/5   | -0.168 | -0.118  |
| S/v3  |   0.977 |   1.268 |        1.003 | ens_inv_mse    |         +26  | 3/5   | -0.007 | -0.005  |
| N/v1  |   0.578 |   0.513 |        0.478 | ens_inv_mae    |        -100  | 4/5   | -0.334 | +0.012  |
| N/v2  |   0.578 |   0.513 |        0.478 | ens_inv_mae    |        -100  | 4/5   | -0.334 | +0.012  |
| N/v3  |   0.553 |   0.513 |        0.475 | ens_inv_mae    |         -78  | 4/5   | -0.164 | +0.028  |

Sub-summary (cells confirming / total):
- **NE 3/3** — best delta -12.996 MWh em v1 (LGB catastrofico 46k corrigido
  por persist via fold 4 alpha=0.00 = pura persist). v2 (cell alvo H10)
  -4.188 MWh = 13% reduction.
- **SE 3/3** — magnitude menor mas wins 3-4/5 folds.
- **N 3/3 (bonus)** — confirmacao do mecanismo H10 detail: persist forte
  em N corrige LGB-pior-que-persist. R² LGB -0.33 -> ensemble +0.01.
- **S 2/3** — v3 +26 MWh (2.7%, irrelevante em escala 1k MWh).

### Esquemas de peso — ranking por frequencia de win

| esquema         | n_cells_won | nota                                                     |
|-----------------|------------:|----------------------------------------------------------|
| ens_inv_mae     |           6 | mais robusto (NE/v2-v3, SE/v2, N/v1-v2-v3)              |
| ens_inv_mse     |           4 | BMA Gaussian (NE/v1, SE/v3, S/v2-v3)                    |
| ens_equal       |           2 | surpresa positiva (SE/v1, S/v1)                          |
| ens_opt_alpha   |           0 | grid-search sobre-otimiza inner_val 30d, generaliza pior |

**Insight**: pesos analiticos proportional-to-precision >> minimo empirico
em inner_val pequeno. Consistente com BMA classico.

### Adaptacao do alpha por fold

alpha_opt varia 0.0-1.0 entre folds da mesma cell — confirma que pesos
respondem a regime shift documentado em iter_0012 B5 (KS p<0.0001 NE+SE).

| cell  | alphas por fold (1..5)                | interpretacao                              |
|-------|---------------------------------------|--------------------------------------------|
| NE/v1 | 0.70, 0.30, 0.70, 0.00, 0.15          | fold 4 LGB MAE=43k vs PER=33k -> pura persist |
| NE/v2 | 0.80, 0.70, 0.75, 0.75, 0.50          | LGB confiavel, peso LGB > peso persist     |
| NE/v3 | 0.75, 0.75, 0.90, 0.80, 0.50          | LGB carry, mix em fold 5 (PER>LGB)         |
| SE    | 0.55-0.90                             | LGB carry com pequeno hedge persist        |
| S/v*  | 0.00 a 1.00                           | regime instavel; fold 1 LGB suprime curto  |
| N     | 0.20-1.00                             | LGB e persist similares em folds 1-2       |

### Sanity checks (queue requeridos: baseline, holdout)

- **B1 leak**: INHERITED iter_0002 features ja auditadas; persist_d1 =
  y_d1[i-1] D-1 safe por construcao.
- **B2 perm**: N/A — ensemble e' meta-modelo de 2 ponteiros, sem features.
- **B3 holdout strict**: DONE_VIA_CV — 12 cells x 5 folds = 60 holdouts,
  gap 7d entre train e test, inner_val 30d sem overlap com test.
- **B4 baseline_compare**: DONE_INTEGRADO — persist_d1 e' componente direto
  do ensemble; skill vs persist computado em todas as 12 cells (skill +0.07
  a +0.21).
- **B5 dist_shift**: INHERITED iter_0012 + evidencia adicional
  `weights_distribution.csv` (alpha varia 0.0-1.0 entre folds = ensemble
  responde a shift).
- **B6 zero_count**: N/A — nao introduz features.

Coverage total documentada em
`outputs/iter_0013/h10_ensemble_v2_persist/sanity_summary.json`.

### Decisao

CONFIRMADO_NE_SE. Ensemble (LGBM + persist_d1, pesos inv_mae ou inv_mse)
deve ser POST-PROCESSING DEFAULT para forecast curt D+1 no loop. Sem req
externo emitido — champions UlFor sao Ridge/LR; H10 testou LGBM (replay
loop). H24 deriva: aplicar mesmo esquema sobre champions Ridge/LR.

### Hipoteses derivadas

- **H24** (P2): mesmo ensemble aplicado a champions Ridge/LR UlFor — ganho
  similar? Implementacao codavel local (Ridge_alpha10 + LR_sklearn no
  replay sobre features iter_0002).
- **H25** (P3): stacker Ridge meta-modelo sobre [LGB, persist, ma7,
  climatologia] supera weighted average analitico?

### Proxima iter

`iter_0014` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`). Alt: H24 (ensemble champions, derivada
de hoje), H11 (quantile, unblocked por H9), H22 (GBDT vs OLS gap), H19
(extrair MAE/R²/F1 champions).


## Iter 0014 — H11 LGBM Quantile Regression NE (REFUTADO_NE)

Hipotese H11 (P2): LightGBM com objective='quantile' (alphas 0.1, 0.5, 0.9).
Substituir ponto-estimativa por bandas — util para downstream (operador
escolhe P90 conservador). Alvo explicito do queue = NE.

CV walk-forward 5 folds (60d cada, gap 7d) sobre features iter_0002 nas 12
cells (4 subs x 3 vers; verdict julgado APENAS em NE; SE/S/N viram bonus).
Cada fold treina 4 modelos no mesmo train: mdl_mean (objective='regression'
= baseline point) + mdl_q01 + mdl_q05 + mdl_q09 (quantile per alpha).

### Resultados (mean across 5 folds, por cell — 12 cells)

| cell  |  PB10  |  PB50  |  PB90  | cov80 | cov10 | cov90 | width  | cross | P50/mean Δ |
|-------|-------:|-------:|-------:|------:|------:|------:|-------:|------:|-----------:|
| NE/v1 |   7937 |  23198 |  18589 | 45.5% | 15.1% | 60.6% |  66276 |  1.7% |   -0.6%    |
| NE/v2 |   6960 |  17091 |  13870 | 43.5% | 14.0% | 57.5% |  57228 |  6.7% |   +5.6%    |
| NE/v3 |   7046 |  17002 |  14114 | 41.8% | 16.4% | 58.2% |  53463 |  6.8% |   +8.4%    |
| SE/v1 |   1725 |   3784 |   2569 | 48.2% | 25.1% | 73.3% |  11319 |  4.0% |   -1.1%    |
| SE/v2 |   1757 |   3853 |   2444 | 47.5% | 24.4% | 71.9% |  11957 |  3.0% |   -0.7%    |
| SE/v3 |   2034 |   3484 |   2363 | 40.5% | 30.2% | 70.6% |   9579 | 13.4% |   -2.9%    |
| S/v1  |    122 |    507 |    459 | 53.1% | 28.8% | 79.3% |   1985 |  9.7% |  -16.5%    |
| S/v2  |    127 |    482 |    415 | 51.6% | 28.9% | 78.5% |   1674 | 13.0% |  -15.6%    |
| S/v3  |    128 |    472 |    355 | 51.5% | 29.3% | 78.1% |   1753 | 14.5% |   -9.2%    |
| N/v1  |     75 |    233 |    122 | 47.5% | 37.2% | 84.6% |   1011 |  2.0% |  -18.2%    |
| N/v2  |     75 |    233 |    122 | 47.5% | 37.2% | 84.6% |   1011 |  2.0% |  -18.2%    |
| N/v3  |     83 |    229 |    109 | 46.5% | 39.2% | 85.6% |    996 |  2.0% |  -17.1%    |

cov80 nominal = 80%; cov10 nominal = 10%; cov90 nominal = 90%.
Δ = (MAE_P50 - MAE_LGB_mean) / MAE_LGB_mean (negativo = P50 melhor).

### Coverage por sub (mean)

| sub | cov_band_80 | cov_10 | cov_90 | folds in [70%, 90%] | diagnostico                        |
|-----|------------:|-------:|-------:|--------------------:|------------------------------------|
| NE  |      43.6%  | 15.2%  | 58.8%  |  0/15               | severe under-coverage              |
| SE  |      45.4%  | 26.6%  | 71.9%  |  1/15               | severe under-coverage              |
| S   |      52.1%  | 29.0%  | 78.6%  |  1/15               | under-coverage (medio P10 alto)    |
| N   |      47.1%  | 37.9%  | 84.9%  |  3/15               | melhor cov_90, mas cov_10 explode  |

### Fold heterogeneity (cov_band por fold, NE/v1 exemplo)

| fold | window           | cov_band | LGB mean MAE | y_te mean | comentario                |
|------|------------------|---------:|-------------:|----------:|---------------------------|
|   1  | 2025-09→2025-11  |    62%   | 39383        |       51k | regime mais estavel       |
|   2  | 2025-11→2026-01  |    27%   | 62065        |       72k | transicao curt-up         |
|   3  | 2026-01→2026-03  |    40%   | 58850        |       67k | transicao continua        |
|   4  | 2026-03→2026-05  |    47%   | 43185        |       46k | novo regime, estabiliza   |
|   5  | 2026-05→2026-07  |    52%   | 26311        |       32k | regime estavel novo       |

Reproduz B5 distribution shift documentado em iter_0012 (NE+SE KS p<0.0001).

### Sanity checks (queue requeridos: holdout, baseline, dist_shift)

- **B1 leak**: SKIPPED — features identicas iter_0002, ja auditadas em
  iter_0010 (PDP_prev forward-looking, p_perm=0.0).
- **B2 perm**: SKIPPED — PI iter_0010 cobre 5/6 cells com p=0.0.
- **B3 holdout strict**: PASSED_EMBEDDED — gap=7d em todas folds; n_test
  58-60 em todas as folds, zero overlap.
- **B4 baseline_compare**: PASSED_EMBEDDED — P50 comparada a persist_d1 E
  LGB-mean por fold. delta_mae_p50_vs_mean_pct mean NE = +4.5% (passa).
  Bonus: P50 BATE LGB-mean em N/S/SE (-18% a -1%).
- **B5 dist_shift**: ANNOTATED_REUSE — evidencia iter_0012 (KS p<0.0001).
  Padrao fold-a-fold do under-coverage confirma diagnostico.
- **B6 zero_count**: PASSED_EMBEDDED — n_test >= 58 (threshold downgrade=30),
  sem zero-only folds.

Coverage total em `outputs/iter_0014/h11_quantile_regression_ne/sanity_summary.json`.

### Decisao

**REFUTADO_NE.** Bandas LGBM quantile com defaults sistemicamente
UNDERCOVERED em todas as 4 subs (cov_band 41-53% vs 80% nominal).
Operador escolhendo "P90 conservador" estaria errado em ~30% dos casos
criticos. Nao virar deliverable v1.0 nesta forma.

Causa raiz: LGBM nao modela heteroscedasticidade explicita (variancia
vem de variancia em-amostra), e' insuficiente sob distribution shift
fold-a-fold documentado em iter_0012. Sem req externo — UlFor nao
desbloqueia mudanca de arquitetura.

P50 magnitude OK em NE (delta +4.5% vs LGB-mean, passa B4); bonus em
N/S/SE: P50 BATE LGB-mean (delta -1% a -18%) — mediana mais robusta
que mean em distribuicoes com cauda longa de zeros.

### Hipoteses derivadas

- **H26** (P3): conformal prediction post-hoc — calibra banda LGBM via
  nonconformity score do inner_val. Goal cov_band in [75%, 85%] sem
  retreinar modelo. Likely-fix se atacarmos H11 de novo.
- **H27** (P3): P50 quantile como POINT ESTIMATE substituto em N+S.
  P50 BATE LGB-mean em magnitude com custo zero (objective swap). Ganho
  transversal pequeno mas universal nos subs com cauda longa.
- **H28** (P3): NGBoost vs LGBM quantile. Distribuicao parametrica
  (Normal/Lognormal) lida com heteroscedasticidade que LGBM quantile
  nao captura. Bench worktree ja tem NGBoost similar (per H11 detail).

### Proxima iter

`iter_0015` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`, derivada H3 confirmado). Alt: H24
(ensemble champions Ridge/LR), H27 (P50 substituto, derivada hoje,
ganho baixo custo), H26 (conformal, fix de H11).

## Iter 0015 — RECON_DELTA UlFor (c8df4077 -> 5dacb5a2)

7 commits absorvidos. Champion N teve upgrade marginal de versao (v1 full ->
v2 clean_plus). Fase 4 observabilidade (validate_d1 + drift PSI + Telegram
alert) FECHADA pelo UlFor em paralelo aos iters 0012-0014 do loop:

| commit | acao | impacto |
|---|---|---|
| `0481a5a6` | checkpoint marker 05:15Z | "H8 ulfor refutada, endpoint operacional, H9 ulfor respondida" — todos Hs UlFor internos (Fase 3), nao do loop |
| `c8e27784` | H10 ulfor clean_plus + endpoint feature-set aware | **MUDANCA DE CHAMPION N**: ridge_curt_n_d1 v2 (clean_plus, 31 feat) @staging substitui v1 (full, 55 feat). NMAE 86.3±31.8% -> **84.8±26.2%** (-1.5pp mean, **-5.6pp std**). lr_N nao promovido. UlFor H10 = CLEAN ∪ {cmo_range, ter_verif_lag1, carga_mwmed_rmean7} PARCIALMENTE CONFIRMADA. `loader.py` agora le `feature_set` do MLflow run params e aplica drops em `build_inference_row` (backward-compat v1 via param `55_feat_*_v3.3 -> full`) |
| `c0e80193` | validate_d1.py foundation | **FASE 4 STEP 1**. Replay-predict ultimos N dias usando features as-of D + comparacao vs realized y_d1 e baseline persist D-1. MLflow experiment `ulfor-validation-d1`. Smoke 14d: NE 49.2% +9% skill, SE 55.4% +20%, **S 137% -37% (alerta)**, N 60.5% +45% |
| `26617ba5` | checkpoint marker 06:15Z | "Fase 3 fechada, Fase 4 iniciada" |
| `9d7652de` | Dagster asset + schedule 07h BRT | Asset `forecast/validate_d1` + schedule `forecast_validate_d1_daily` (10h UTC, **STOPPED**). Subprocess pro venv raiz (mlflow nao em dataops). Smoke 7d: NE 40.5% +29%, SE 71.7% +37%, **S 184.3% -41% (alerta)**, N 56.6% +59% |
| `9873e3c8` | drift PSI + Telegram alert | **FASE 4 STEPS 2+3 (Opcao C closure)**. Evidently ABANDONADO (conflito `evidently>=0.5 depends on plotly<6` vs nosso pin `plotly>=6.5`). PSI nativo numpy/scipy: bins adaptativos n/3 max 10, Laplace smoothing, dual `psi_long`+`psi_recent`, SEASONAL_FEATURES excluidas. Telegram standalone com 4 gatilhos. Window default 60d. Smoke: **NE psi_recent_max=11.74 39/45 features** drift |
| `5dacb5a2` | checkpoint marker 07:15Z | "Fase 4 step 1 fechada (Dagster+drift+Telegram)" |

### O que mudou na nossa interpretacao

1. **Champion N evoluiu silenciosamente**. Iter_0011 marcou ridge_N@staging
   v1 (full); 1 commit depois, v2 clean_plus. Tracking versao do champion
   no leaderboard agora obrigatorio (nao so' modelo + NMAE — incluir
   `feature_set` + `versao MLflow`). Linha N atualizada com `v2 (clean_plus, 31 feat)`.

2. **Pipeline observabilidade Fase 4 inteira PRONTA**. Champions tem 3
   camadas de validacao continua quando schedule for ativado: replay
   diario (`validate_d1`), drift PSI (`compute_drift`), Telegram alert
   (`alert.py`). **Acao Breno**: gerar `BRAZILGRID_TELEGRAM_BOT_TOKEN`
   + `BRAZILGRID_TELEGRAM_CHAT_ID` (mesma convencao `sintegre_freshness_alert`)
   + ativar `forecast_validate_d1_daily` no Dagster UI.

3. **Smoke validate_d1 confirma fragilidade lr_S**. Janelas 7d e 14d:
   skill_vs_persist NEGATIVO (-37 a -41%), NMAE 137-184%. CV 5x60d
   ainda mostra +0.447 R² mas operacionalmente o modelo colapsa em
   janelas curtas. Valida flag "FRAGIL" no leaderboard.

4. **Drift PSI confirma B5 distribution shift iter_0012**. NE smoke 60d:
   psi_recent_max=11.74 (vs industry-std threshold 0.2), 39/45 features
   driftando. Top: rolling-means de carga e CMO. Threshold calibrado em
   `>1.0` (vs std 0.2) sera recalibrado em 2 semanas de operacao real.

### Reqs / hipoteses

- Sem novos requests pendentes. Os 3 antigos (req-0001/2/3) continuam DONE
  (verificado contra `git show 5dacb5a2:coordination/loop_requests.md`).
- Sem novas hipoteses do **loop** geradas. Mudancas absorvidas referem-se
  a Hs UlFor internos (PLANO_FINAL). **Critico**: UlFor H10 (clean_plus
  FEATURE_DROPS_N) != nosso H10 (LGBM+persist ensemble, iter_0013).
  Disambig persistente em `state.json.nota_nomenclatura`.
- **H18 (sanity B1-B6 sobre champions)**: bloqueio segue, mas urgencia
  diminui — pipeline validate_d1 + drift PSI da cobertura empirica
  continua que reduz valor marginal do B1-B6 audit local.
- **H19 (extrair MAE/R²/F1 dos champions)**: atratividade cresce — daily
  summary JSON do validate_d1 tem MAE/R²/skill por sub. Quando schedule
  ativar + loop ganhar acesso MLflow URI, custo H19 cai significativamente.

### Proxima iter

`iter_0016` retoma planner_config: **H21** (P2 feature engineering
`pdp_residual = pdp_prev - gen`). Derivada de H3 iter_0010, codavel
localmente, sem dep externa. Razoes inalteradas desde iter_0014/0015.
Considerar usar ensemble post-processing (H10 nosso, iter_0013 confirmado)
+ LGBM (iter_0012 confirmado GBDT padrao) para baseline final do bake-off
H21. Alt: H27 (P50 substituto custo zero), H24 (ensemble Ridge/LR), H19
(MAE/R²/F1 dos champions — agora parseavel via summary JSON UlFor).

## Iter 0016 — H19 extrai MAE/R²/F1 dos champions (CONFIRMADO_PARCIAL)

Hipotese H19 (P2 metric, layer=meta): champions UlFor (ridge_curt_ne_d1 etc)
estao publicados so' por NMAE no leaderboard. Pos-adocao do metric_suite H9
(MAE/R²/F1 primarios), preciso extrair as 3 metricas para cada champion para
o leaderboard ser internamente consistente.

### Fontes acessiveis ao loop (sem MLflow tunnel ao EC2)

| fonte | dado | localizacao |
|---|---|---|
| FINDING_RIDGE_BEATS_GBDT.md | tabela CV 5x60d: NMAE+R² mean±std de 6 modelos × 4 subs | `C:/Projetos/brazilgrid-ulfor/experiments/bakeoff_curtailment_multisub/FINDING_RIDGE_BEATS_GBDT.md` |
| promote_champions.py | `CV_METRICS_BY_FS = {"full": ..., "clean": ..., "clean_plus": ...}` por sub | `C:/Projetos/brazilgrid-ulfor/experiments/bakeoff_curtailment_multisub/promote_champions.py:200-220` |
| state.json baselines + iter_0013 baseline_metric | MAE persist_d1 em MWh por sub (CV 5x60d gap7d) + NMAE persist | `state.json` + `iterations/iter_0013_h10_ensemble_v2_persist.md` |

### Derivacao MAE em MWh

```
ymean_test_estimated = MAE_persist_mwh / NMAE_persist
MAE_champion_mwh     ≈ NMAE_champion_mean × ymean_test_estimated
```

| sub | ymean_test (MWh) | NMAE_champion | **MAE_champion derived** |
|---|---:|---:|---:|
| NE | 75730 | 0.337 | **25521 MWh** |
| SE | 12064 | 0.466 | **5622 MWh** |
| S  | 1023  | 0.896 | **916 MWh** |
| N  | 506   | 0.848 | **429 MWh** |

**Consistency check** (NMAE_persist do FINDING vs state.json):
- NE 0.445 vs 0.445 OK
- SE 0.688 vs 0.688 OK
- S  1.242 vs 1.242 OK
- N  1.007 vs 1.007 OK

Confirma que `iter_0013` MAE_persist + `FINDING` NMAE_persist usam o MESMO
target y_d1 e MESMA particao temporal — derivacao MAE valida em 1a ordem.

### Caveat MAE derived

NMAE publicado e' `mean(NMAE_per_fold)`, nao `mean(MAE_per_fold) / mean(ymean_per_fold)`.
ymean varia entre folds (iter_0014: NE folds 32k → 72k → 67k → 46k → 32k MWh).
Logo MAE_champion_derived ≠ MAE_champion_mean(folds) exatamente. Estimativa
1a ordem; slack provavel ~10-15% se erro correlaciona com ymean.

Para MAE-em-MWh exato precisa-se ou (a) dump per-fold MAE do MLflow, ou (b)
re-rodar bakeoff_d1.py --cv-folds 5 com CH local — ambos fora do envelope
desta iter (a requer MLflow tunnel, b requer execucao de codigo UlFor com
features full em CH local).

### Gap F1_p50

`bakeoff_d1.py`, `promote_champions.py`, `validate_d1.py` — nenhum computa F1
binarizada por P50 train. Source unica de F1 hoje e' `scripts/metric_suite.py`
do loop (H9 iter_0008), aplicado so' a iter_0002 replay (LGBM, n_test=11,
nao Ridge/LR). Para fechar o gap definitivamente -> **req-0007 emitido** ao
UlFor pedindo (a) F1_p50 por fold no proximo CV bake-off; (b) dump
CV_SUMMARY parquet acessivel ao loop.

### Sanity checks (queue requeridos: []. Mas rodar 6 defaults)

- **B1 leak**: N/A — meta-acao de extracao de metricas, sem treino.
- **B2 perm**: N/A — sem modelo treinado.
- **B3 holdout strict**: N/A_inherited_via_source — FINDING numbers vem
  do CV 5x60d gap7d UlFor, que e' holdout estrito por construcao.
- **B4 baseline_compare**: PASSED_DOUBLE — (i) skill_vs_persist computado
  inline para cada champion (NE +24.3%, SE +32.3%, S +27.9%, N +15.8%);
  (ii) consistency check NMAE_persist FINDING vs state.json em 4/4 subs.
- **B5 dist_shift**: ANNOTATED_REUSE — std de NMAE/R² grande em S/N
  (iter_0014 KS p<0.0001 NE+SE; iter_0012 ymean cai 3x em NE entre folds).
  Capturado nos `_std` ao lado de cada metric.
- **B6 zero_count**: N/A — sem features novas.

### Decisao

**CONFIRMADO_PARCIAL.** Leaderboard agora consistente para 2/3 metricas
primarias (MAE_derived + R²) + NMAE secundaria. F1_p50 gap fechado via
req-0007 (proximo round CV do UlFor). Bonus: ymean_test_per_sub agora
disponivel para reusar em proximos bake-offs sem repetir derivacao.

### Hipoteses derivadas

Nenhuma. Plano natural pos-iter_0016 segue: **H21** (P2 feature
engineering `pdp_residual = pdp_prev - gen`, derivada H3 iter_0010).

## Iter 0017 — RECON_DELTA UlFor (5dacb5a2 -> 5c7963d4)

15 commits absorvidos. **Maior delta do projeto desde iter_0007**: bias
correction NE EM PRODUCAO bate persist em 14d real pela primeira vez.

| commit | acao | impacto |
|---|---|---|
| `13f4a4de` | feat(vitrine) — dashboard MLflow validation D+1 (Fase 4 step 2) | Pagina forecast.html na vitrine com NMAE/R²/skill/PSI + timelines + tabela de runs (CSS-puro, le `/api/mlflow/validation-runs`). Link nav global. |
| `8a30a29e` | feat(forecast) — rastreabilidade end-to-end no /api/forecast/d1 (Fase 4 step 3) | mlflow_url clicavel + training_git_sha + experiment_id/name + training_ran_at + inference_git_sha por response. Divergencia training/inference git_sha = alerta de modelo desatualizado. Smoke 4 subs OK. |
| `5ea5a412` | checkpoint marker 08:15Z | Fase 4 STEPS 2+3 fechadas. |
| `f64cfbb7` | coord — ACK req-0007 | UlFor le request do canal loop_requests.md. |
| `f697449d` | feat(forecast) — UlFor **H13 closure S** REFUTADA | ridge_S+clean_plus piora CV (-0.28 R² lr / -0.14 ridge vs full); ridge bate lr em clean_plus mas nao revolucao. Em 14d real (2026-05-08..21) **TODOS ML perdem persist em S**: persist_d1 99.9% / lr+full champion 159.6% R² -1.915 / ridge+clean_plus 137.4%. Champion lr_curt_s_d1@v1 (full) MANTIDO. **Nova UlFor H18 aberta** (S underperforma persist estruturalmente — investigar regime shift, target alt, persist como fallback first-class). |
| `6b21ffdf` | feat(forecast) — **req-0007 IMPLEMENTADO** | helper `_f1_p50()` portado de `scripts/metric_suite.py` (positive-class F1, threshold P50(y_train), NaN se thr<=0). `metrics()` aceita y_train + retorna f1_p50/threshold_p50/ymean_test. Per-fold parquet em `experiments/bakeoff_curtailment_multisub/outputs/cv_summary_per_fold_{full,clean_plus}.parquet` (~13KB cada, 140 rows = 4 subs x 7 modelos x 5 folds). F1_p50 clean_plus: **NE ridge/lr 0.80** (persist 0.75), SE xgb 0.71 (persist 0.69), N ridge/ma7 0.79 (persist 0.72), S NaN (P50=0 esperado). MAE per-fold MWh direto: NE ridge mean 27.0K, SE lgbm 7.0K, S ridge 1.05K, N ridge 425. **req-0007 CLOSED.** |
| `d93100ae` | feat(forecast) — UlFor H19 root-cause SE/fold4 R²-2.5 | Ablation single-feature add-back: **`ter_verif_rmean7`** e UNICA das 17 dropadas que sozinha restaura 99% gap (clean R² -2.53 -> +1 feat -0.48 ≈ full -0.45). Outras 16 ficam em R² -1.83 a -2.75. Dropada por corr +0.92 com `ter_verif_lag1` — mas em seca-2025Q3 termico sobe e rolling-7d captura trend que lag1 nao captura. VIF/corr NAO BASTA para guiar drops em sub com regimes sazonais fortes. |
| `8708c329` | checkpoint marker 05:00Z | — |
| `ea8ca325` | feat(forecast) — UlFor **H20 CONFIRMADA** | clean_plus_v2 = clean_plus SEM dropar ter_verif_rmean7 (39 feat). Re-CV 5x60d: **SE lr -11.27pp NMAE / +0.62 R²** major win; NE/N/S within bound (≤+0.35pp). Fold 4 SE blowup curado: R² lr -3.12 -> -0.07. CHAMPIONS NAO ALTERADOS (todos em `full` por consistencia MLflow). clean_plus_v2 = alternativa analitica + base UlFor H21. |
| `e56aa5f3` | feat(forecast) — UlFor **H18-A REFUTADA** | Drop universal 4 envenenadoras ridge_S+FULL (curt_rmean7, pdp_prev_solar_mwh, val_net_mwmed, curt_lag14): NMAE 97.9% -> 103.5% (+5.55pp), R² +0.230 -> +0.146, stdev 18.7% -> 29.2%. Per-fold: -2 a -7pp em 2/5, +0.15pp em 1/5, +13 a +24pp em 2/5. `pdp_prev_solar_mwh`: Top-3 IMPORTANTE fold 1 / Top-3 ENVENENADORA fold 0 — papel inverso por regime. lr_S+full MANTIDO. Follow-ups (NAO autopilot): H18-B per-fold FS / H18-C target log1p ou binario / H18-D mixture-of-experts. |
| `108772a5` | checkpoint marker 05:30Z | H20 + H18-A. |
| `4223aa4c` | feat(forecast) — UlFor **H21 REFUTADA** | Treino lr_SE ate 2026-05-07 + holdout 14d (2026-05-08..21): persist 69.6% / lr+full champion 77.4% / **lr+clean_plus_v2 92.7% (+15.31pp regride)**. Ganho UlFor H20 estava CONCENTRADO em fold 4 (seca-2025Q3); regime Mai/26 (transicao chuvoso->seco, 3 dias zero curt) favorece FULL. Champion lr_curt_se_d1@v1 (full) PERMANECE. **NENHUM ML bate persist em 14d SE** (teto-de-dados D+1 estendido do NE ao SE no regime atual). Bias estrutural +5500-7100 MWh em todos ML abre UlFor H14. |
| `41d8d952` | feat(forecast) — **UlFor H14 + H14-C PRODUTIZAR** (NE only) | Bias correction rolante 28d: `corrected_D = max(0, pred_D - mean(pred-actual)[ult 28d causais])`. **14d real NE: -9.94pp NMAE (59.2->49.2%), R² -0.348 -> -0.135. BATE persist (54.1%) por 4.9pp — PRIMEIRA VEZ NO PROJETO.** SE +1.61pp (regime change joga correcao no rumo errado). S +10.81pp catastrofico (ymean ~32 MWh, bias absoluto -139 amplifica ruido). N +2.15pp (raw ja bate persist 36.9pp). CV walk-forward 5x60d (cobre 2025-07..2026-05): **NE -6.90pp media / wins 4/0/5 -> PRODUTIZAR**. SE -0.33 / 0/1/5 -> NAO. S +0.95 / 2/3/5 -> NAO. N -28.63 / 3/2/5 -> MAYBE (fold 4 -120pp puxa mean; folds 0/1 regridem +2pp). |
| `586eeae6` | feat(forecast) — **route /api/forecast/d1 expoe bias_correction_mw per-sub** | `services/analytics_api/forecast/loader.py`: BIAS_CORRECTION_DEFAULTS={NE:True, SE:False, S:False, N:False}. compute_bias_correction(sub, model, feature_set, window_days=28) replay-pred ultimos 28d com y_d1 conhecido (falha-silenciosa). Response: predicted_curt_mwh (raw, back-compat), predicted_curt_mwh_corrected (sempre exposto), predicted_curt_mwh_default (per-sub default), bias_correction.{bias_mw, applied_in_default, recommended_apply, window_days, n_used, window_start, window_end, applicable, source}. Smoke e2e: **NE bias=+7319 MWh applied=True raw=62329 -> corrected=55010 (perto de persist=59149)**. SE bias=-284 applied=False raw=5484. S bias=-167 applied=False raw=303. N bias=-29 applied=False raw=246. analytics_api NAO deployada (mudanca local). |
| `5c7963d4` | checkpoint marker 06:25Z | H21 REFUTADA + H14 analitica + H14 produtizado. |

### O que mudou na nossa interpretacao

1. **Champion NE EFETIVO mudou em PRODUCAO sem mudar modelo treinado**. Wrapper
   de inferencia agora aplica bias_correction_28d default ON em NE. Marco: 14d
   real NMAE 49.2% bate persist 54.1% por -4.9pp — **primeira vez no projeto
   que ML supera persist em NE**. Trazendo o NE para uma posicao em que o ML
   passa a ter valor operacional direto (vs ate iter_0011 onde era "bate persist
   em CV mas perde em 14d real"). Champion treinado (`ridge_curt_ne_d1@v1`)
   INALTERADO no MLflow Registry — bias correction e' post-processing.

2. **Teto-de-dados D+1 estendido do NE ao SE no regime Mai/26**. UlFor H21
   REFUTADA mostrou que NENHUM ML bate persist_d1 em NMAE 14d real em SE
   (persist 69.6% < lr+full 77.4% < lr+clean_plus_v2 92.7%). Padrao identico
   ao que observamos em NE pre-bias_correction. **Sugestao implicita ao loop**:
   testar bias_correction_28d sobre `lr_curt_se_d1` com sub-janela
   estendida ou correcao adaptativa por regime (UlFor abriu UlFor H14 mas
   decidiu NAO produtizar SE porque janela fixa 28d joga correcao no rumo
   errado em transicao chuvoso->seco).

3. **req-0007 fechado dentro do envelope esperado**. F1_p50 + per-fold MAE
   parquet acessiveis via git em `experiments/bakeoff_curtailment_multisub/outputs/`.
   Coluna `f1_p50` do bloco `champions_metrics_consolidated` atualizada para 4
   subs (NE 0.80, N 0.79 de ridge no clean_plus; SE pending parser do parquet
   full; S NaN esperado). MAE-em-MWh AGORA exato (sem caveat 10-15% slack do
   iter_0016 H19 derivation).

4. **Pipeline observabilidade 100% pronto**. Fase 4 fechada nas 3 steps (1:
   `validate_d1.py` iter_0015 + 2: dashboard MLflow vitrine + 3: rastreabilidade
   endpoint). Champions ridge/lr em PRODUCAO com (a) replay diario `validate_d1`,
   (b) drift PSI dual long/recent, (c) Telegram alert 4 gatilhos, (d) dashboard
   visivel `vitrine.brazilgrid.com/forecast.html`, (e) git_sha tracking
   training-vs-inference. Schedule Dagster STOPPED ate Breno gerar
   `BRAZILGRID_TELEGRAM_BOT_TOKEN/CHAT_ID`.

5. **Insights metodologicos consolidados** (3 commits UlFor sobre o mesmo tema):
   - VIF/corr NAO BASTA para guiar drops em sub com regimes sazonais (H19 ulfor:
     `ter_verif_rmean7` corr +0.92 com lag1 mas load-bearing em seca-2025Q3).
   - Drop GLOBAL por feature 'envenenadora em >=2 folds' INVALIDO (H18-A ulfor:
     `pdp_prev_solar` inverte papel por regime — Top-3 IMPORTANTE fold 1,
     Top-3 ENVENENADORA fold 0). Per-fold FS pode resolver.
   - Ganhos CV CONCENTRADOS em fold-especifico NAO sobrevivem 14d real
     (H21 ulfor: clean_plus_v2 ganho era fold-4-only seca-2025Q3, regime
     Mai/26 chuvoso->seco joga +15.31pp). **Holdout temporal 14d real e' o
     test ground-truth, NAO o CV mean** — aplica tambem ao loop.

### Como nossa queue muda

- **Nenhuma H do loop foi resolvida** pelos commits. Todos os Hs resolvidos
  sao **UlFor internos** (PLANO_FINAL Fase 3+4) — sem relacao com nosso
  queue. Disambiguacao critica continua valida: nosso H13/H14/H18/H19/H20/H21
  ≠ UlFor H13/H14/H18/H19/H20/H21. **4 colisoes ativas** agora; convencao
  proposta: prefixar Hs externos `Hxx_ulfor` em comments e iter handoffs.

- **H19 (loop, status=done)**: gap F1_p50=N/A do iter_0016 fechado via
  req-0007 implementado. Parser do parquet (~0.5h) pode preencher 4 linhas
  champion sem rodar bake-off. Candidato a micro-iter 0018a.

- **H24 (loop, status=queued)**: BASELINE DE COMPARACAO MUDOU. Bias_correction
  em NE entrega -9.94pp NMAE em 14d real (ordem similar do ensemble H10
  +13% MAE NE/v2 em CV). H24 (ensemble Ridge+persist) agora precisa comparar
  contra Ridge+bias_corrected (nova baseline operacional NE), nao so contra
  Ridge-only. Possivel coexistencia: bias correction (anti-drift estrutural) +
  ensemble (mistura com persist quando regime instavel). Se ensemble incluir
  persist_d1, ja captura parte do bias por outra via — testar nao-trivialmente
  sobreposicao.

- **H18 (loop, status=blocked)**: urgencia continua diminuindo. Fase 4 100%
  fechada (steps 1+2+3) somam mais camadas de observabilidade aos champions.
  Valor marginal do B1-B6 audit local mais baixo ainda.

### Reqs

- **req-0007 CLOSED** (UlFor commit `6b21ffdf`, ulfor_verdict: "AMBAS as
  lacunas fechadas em uma sprint"). open_requests=[] novamente.
- Sem novos requests emitidos.

### Proxima iter

`iter_0018` retoma planner_config com pequena variacao:
**(opcao A — recomendada)** Micro-iter 0018a: parser do parquet
`cv_summary_per_fold_full.parquet` para preencher F1_p50 dos 4 champions
no leaderboard (ridge_NE+full, lr_SE+full, lr_S+full, ridge_N+clean_plus).
Custo <0.5h. Fecha gap conceitual deixado em iter_0016. **(opcao B)**
**H21** original (P2 pdp_residual = pdp_prev - gen, codavel local, 1.5h).
Razoes inalteradas desde iter_0014/0015/0016.

Alt: H24 (ensemble Ridge/LR vs Ridge+bias_correction — baseline mudou; 1.5h),
H27 (P50 quantile substituto custo zero), H26 (conformal post-hoc para H11
calibration).

### Lessons learned

- **Bias correction rolante e' upgrade gratuito quando bias estrutural existe**.
  UlFor H14: erro sistematico mean(pred-actual) em janela causal ≠ 0 e bias
  CORRIGIVEL post-hoc sem retreinar. Pre-requisito: erro estatisticamente
  estavel na janela de calibracao (28d). REGIME CHANGE no holdout joga a
  correcao no rumo errado (UlFor H14 evidence: SE +1.61pp; S +10.81pp).
  Decisao por-sub conservadora (default ON so quando wins 4/0/5 em CV) e' o
  certo.
- **Holdout 14d real continua sendo ground-truth final** acima do CV-mean.
  UlFor H21 mostrou ganho CV+11.27pp colapsa para regressao +15.31pp em 14d
  real porque ganho era fold-4-only sazonal. **Padrao a importar pro loop**:
  toda hipotese promovida via CV deve passar tambem por holdout 14d real
  antes de virar champion permanente (UlFor adotou; loop ainda nao tem
  pipeline equivalente porque CH local stale > 30d em features).
- **Recon iter continua sendo barato e absolutamente necessario**. 15 commits
  em ~10h reais de UlFor desde nosso checkpoint anterior (iter_0015) — sem
  recon, loop tentaria iter_0017=H21 sobre features iter_0002 ignorando que
  UlFor ja produtizou bias correction em NE (modelo operacional mudou). Custo
  recon ~0.5h vs ~2.5h de hipotese; ROI altissimo.
- **Disambig nomenclatura H critica**. 4 colisoes ativas (H13/H14/H18/H19/H20/H21
  duplicadas entre loop e UlFor). Convencao escolhida: `Hxx_ulfor` em comments
  e iter handoffs do loop; commits UlFor seguem usando `Hxx` nu (contexto
  resolve no proprio repo). Aplicar daqui em diante.

