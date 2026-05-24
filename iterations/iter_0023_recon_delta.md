---
alvo: recon_delta_ulfor_post_ec0fd937
layer: meta
iter_num: 0023
type: recon_delta
data_utc: 2026-05-24T19:30:00Z
ulfor_head_inicio: ec0fd937
ulfor_head_fim: 4427a718
commits_absorvidos: 6
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.4
---

# Iter 0023 — RECON_DELTA UlFor (ec0fd937 → 4427a718)

## Objetivo

Absorver 6 commits novos da sessao UlFor entre `ec0fd937` (HEAD na entrada
do iter_0022, fim do bloco H22 ulfor ABERTA-EM-CV) e `4427a718` (checkpoint
14:15Z, fechamento de 3 sprints + decisao matrix pendente Breno). Janela
~75 min reais de UlFor (13:00-14:15Z = 09:57-10:13 BRT). Conteudo
**MUITO denso e load-bearing**: 1 patch H22 model-aware que DESBLOQUEIA SE
preservando familia LR, 1 sweep H14-G window×k que SUPERA H14-B em N (mas
sub-aplicado), 1 documento de sintese CHAMPION_DECISION_MATRIX (7 acoes
promovieis), 1 profile S, 3 checkpoints markers.

## PHASE A — Commits inspecionados

Em ordem cronologica:

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `2daa5d40` | feat — H22 model-aware (PI com champion-real, fix SE/lr) | **DESBLOQUEIA SE preservando familia LR** | PHASE B leaderboard + state |
| 2 | `593a1093` | chore — checkpoint 13:45Z (H22_MA fechado, PROMOVIVEL SE/lr) | Marker absorvido em commit 1 | state ulfor_session_sync |
| 3 | `1ba9cb39` | exp — H14-G window×k sweep N+NE | **N w=14d k=1.5 SUPERA H14-B (-31.87pp vs -19.63pp)**, mas apply 41% vs 100% e 1L tradeoff vs 1L | PHASE B leaderboard linha N |
| 4 | `de190f9d` | chore — checkpoint 14:05Z (frente bias correction H14..H14-G fechada) | Marker | state |
| 5 | `c5004bac` | docs — CHAMPION_DECISION_MATRIX (24 cells + 7 acoes Breno) | **Sintese decision-aid load-bearing** — consolida H22/H22_MA/H14-G | PHASE B leaderboard + state |
| 6 | `4427a718` | chore — checkpoint 14:15Z + FINDING_S_PROFILE.md | Profile S (curt-zero estrutural, low signal) + marker fechamento sessao | state |

### Interpretacao detalhada por commit

**`2daa5d40` — H22 model-aware (FIX da anomalia LR_SE/H22)**

Patch que **substitui o PI universal (Ridge) do `feature_set=h22_per_fold`
por PI medida com o champion-model REAL de cada sub**:
- NE/N (ridge) → PI com Ridge (= mesmo que h22_per_fold original)
- SE/S (lr) → PI com LR (NOVO)

**Resolve o bug teorico de H23_ulfor** (recon iter_0021): "Ridge mascara
importance via shrinkage L2 — features load-bearing pra LR ficam com PI ~ 0
quando medidas com Ridge". `ter_verif_rmean7` em SE/lr era esse caso —
H22 universal Ridge-PI dropava-a, LR colapsava (-0.828 R²); H22_MA preserva.

Resultado CV 5x60d (4 subs × {ridge, lr} × full+h22+h22_MA = 24 cells):
- **NE ridge/lr**: H22 ≡ H22_MA (mesmo champion). ridge_NE/h22 = 33.6%/+0.470; lr_NE/h22 = ainda PROMOVIVEL com LR mas champion atual e ridge.
- **SE/lr**: 57.2%/-0.068 (H22) → **48.1%/+0.381** (H22_MA). Recupera 9.1pp NMAE + 0.449 R². Champion alternativo PROMOVIVEL preservando familia LR.
- **SE/ridge**: 48.3%/+0.390 (H22) ≡ 48.4%/+0.369 (H22_MA) — Ridge nao precisa do model-aware fix.
- **S/lr**: 87.2%/+0.371 (full) → **84.3%/+0.387** (H22_MA com 4 drops). PROMOVIVEL marginal.
- **N**: NE ridge/lr h22 ≡ h22_MA (champion atual N e ridge).

Artefatos: `h22_model_aware_per_fold.py` (script paralelo), 
`FINDING_H22_MODEL_AWARE.md`, novo `--feature-set h22_model_aware` em 
`bakeoff_d1.py`, `outputs/cv_summary_*_h22_model_aware.parquet`.

**Implicacao operacional**: agora a UlFor pode promover SE com `lr+h22_MA`
**sem trocar familia de modelo** (lr→ridge), mantendo continuidade da serie
de validacao. Antes (recon iter_0021): unica opcao para SE era `ridge+h22`
(troca familia).

**`1ba9cb39` — H14-G window×k sweep N+NE**

Sprint co-autorado com agente paralelo. Sweep grosso 
`window ∈ {14,28,60,90} × k ∈ {0.5,1.0,1.5}` para bias correction com 
threshold sigma_bias (reusa fold_one de H14-E). Subs N+NE (SE/S 
intrinsicamente nao-corrigiveis per H14-F).

Veredito por sub (criterio strict wins >= -5pp):

| Sub | Champion atual | Best H14-G | Decisao UlFor |
|-----|---|---|---|
| N (atual H14-B w=60 always-on -19.63pp 3W/1L) | best=w=14 k=1.5: **-31.87pp 3W/1L apply=41%** (SUPERA por -12.24pp) | **PROMOVIVEL** |
| NE (atual H14-C w=28 always-on -6.90pp 4W/0L) | best=w=14 k=0.5: -8.29pp 3W/0L apply=79% (MARGINAL -1.39pp) | MANTER status quo |

H14-G N (w=14, k=1.5) Pareto-supera H14-B no papel: mesma 3W mas reducao 
**-31.87pp vs -19.63pp** e apply_rate 41% (vs 100%) — bias correction so 
quando `|bias_rolling_14d| > 1.5·sigma_bias_rolling_train`. Mais conservadora 
e mais efetiva nos folds onde dispara.

Custo produtizar: medio (cache `sigma_bias_rolling_train` em runtime, similar 
H14-F). UlFor explicito: **PARAR E PERGUNTAR Breno antes de promover** 
(mudar loader.py = tocar prod).

Artefatos: `h14g_window_k_sweep.py`, `FINDING_H14G_WINDOW_K_SWEEP.md`.

**`c5004bac` — CHAMPION_DECISION_MATRIX**

**Documento decision-aid LOAD-BEARING** consolidando todas as alternativas
side-by-side. 4 secoes:

1. **Champion ATUAL em prod (status quo, loader.py)** — tabela 4 subs com 
   modelo + feat-set + bias correction + CV NMAE + R².
2. **Matriz completa CV 5x60d (24 cells)**: `ridge+lr × {full, h22, h22_MA}` 
   × 4 subs. Negrito marca champion potencial por sub.
3. **Bias correction alternativas (H14-B/C/G)** — 6 cells NE+N com 
   delta_NMAE, wins/loses, apply_rate.
4. **Recomendacoes por sub** com tradeoff explicito + 5. Risk/value table 
   (7 acoes) + 6. Validation gap.

**7 acoes promovieis listadas:**

| Acao | Esforco | Beneficio CV | Risco |
|---|---|---|---|
| Promover NE ridge+h22 | Baixo | -8.8pp NMAE / +0.07 R² | Baixo |
| Promover SE ridge+h22 | Baixo | +1.8pp / +0.007 R² | Baixo (trade-off NMAE↔estabilidade) |
| Promover SE lr+h22_MA | Baixo | +1.6pp / -0.002 R² | Mesma ordem do full |
| Promover S lr+h22 | Baixo | -2.9pp / +0.016 R² | Baixo |
| Promover N ridge+h22 | Baixo | -23.7pp / +0.51 R² | Baixo |
| Trocar NE bias H14-C→H14-G(w=14,k=1) | Baixo | -0.9pp / 0→1 less loss | Baixo (Pareto strict) |
| Trocar N bias H14-B→H14-G(w=60,k=1) | Baixo | +8.4pp / 0→2 less loss | Trade-off, nao Pareto |

**Validation gap explicito**: tudo CV, sem 14d real (so H21 ulfor foi 
validado em 14d real). **UlFor sugere**: rodar bake-off em 14d real para 
candidatos PROMOVIVEIS antes de produtizar. Comando:
```
CH_URL=http://localhost:18123/ uv run python -m \
  experiments.bakeoff_curtailment_multisub.bakeoff_d1 \
  --feature-set h22_per_fold --cv-folds 1 --no-mlflow
```

**CAVEAT detectado pelo loop**: a secao "Champion ATUAL" lista TODOS os 4 
subs como `lr + full`, mas state.json + iter_0017 + iter_0018 indicam 
**NE = ridge_alpha10**, **N = ridge_alpha10@staging v2 clean_plus** 
(state.json `best_ml_oficial_ulfor_cv_5folds`). Possiveis explicacoes 
(NAO investigadas nesta iter):
- (A) UlFor reverteu silentemente loader.py para lr+full em todos os subs 
  (improvavel — nao tem commit `loader.py` na janela ec0fd937..4427a718);
- (B) A secao "Champion ATUAL" da matriz tem bug de documentacao 
  (provavel — outras secoes mencionam `lr+full` apenas em recomendacoes 
  "substituir X por Y", consistente com state.json);
- (C) UlFor mediu CV usando lr+full como referencia padrao mas Registry 
  segue ridge_curt_ne_d1 + ridge_curt_n_d1 (consistente com (B)).

**Acao**: flaggar discrepancia em state.json `ulfor_session_sync` para 
proxima recon investigar via `git show 4427a718:services/.../loader.py` 
ou /api/forecast/d1 smoke test. Por seguranca, **leaderboard nao reescreve 
champion atual baseado nesta matriz** — mantem ridge_NE / lr_SE / lr_S / 
ridge_N do state.json.

**`4427a718` — checkpoint 14:15Z + FINDING_S_PROFILE.md**

Marker de fechamento sessao + profile S detalhado (`s_sub_profile.py`). 
Bullet do checkpoint: "Status: champion atual em prod **INTACTO**. 7 acoes 
promovieis consolidadas em CHAMPION_DECISION_MATRIX.md — PARAR E PERGUNTAR 
Breno." Concorrencia coordenada: 3 commits paralelos + 2 do UlFor leader + 
1 colaborativo, sem merge conflict. Tunneis CH+MLflow OK. loop_requests 
4 DONE / 0 OPEN. Master 13 ahead origin.

`FINDING_S_PROFILE.md` (94 linhas) + `s_sub_profile.py` (261 linhas): 
profile do sub S — n_train pequeno (205-461 dias dependendo do fold), 
ymean ~32-1023 MWh fold-dependente, curt-zero estrutural ~37% dias 
(consistente com F1_p50 NaN no parquet). Nao muda interpretacao do 
champion S mas documenta o "porque" da fragilidade conhecida.

**`593a1093`, `de190f9d`** — markers (absorvidos no descrito acima).

## PHASE B — Atualizacoes do loop

### state.json `ulfor_session_sync`

Adicionado bloco `iter0023_inicio_head=ec0fd937`, 
`iter0023_fim_head=4427a718`, lista de 6 commits absorvidos, 
`delta_resumo_iter0023` documentando:
- Champions em producao **INALTERADOS** (Registry intocado);
- **DOIS candidatos sucessores novos** alem dos de iter_0021:
  - **SE lr+h22_MA** (48.1% NMAE / R² +0.381) — preserva familia LR, 
    fix da fragilidade exposta em iter_0021 (cond_num 2.5e17 do lr+full);
  - **N bias H14-G (w=14, k=1.5)** (CV -31.87pp vs H14-B -19.63pp);
- **Discrepancia detectada** entre CHAMPION_DECISION_MATRIX "Champion 
  ATUAL" (lista lr+full para todos) e state.json/`/api/forecast/d1` 
  oficial (ridge_NE + ridge_N) — flaggada para investigacao;
- Regime UlFor multi-agente PERMANECE ativo (3 sprints coordenados em 
  ~75min com agente paralelo);
- 0 novos `req` recebidos / fechados extras.

### hypotheses_queue.md

**Nenhuma H do loop fechada por estes commits** (nenhuma colisao direta 
com nosso queue). Pequena atualizacao em H22 (nosso GBDT vs OLS gap) 
para registrar que H22_MA reforca o lesson H23 (PI com modelo final) — 
ja documentado em iter_0021. H8 (intercambio) sem atualizacao 
(`val_export_mwmed` / `val_import_mwmed` continuam drop candidates em 
NE+SE via H22_MA, identicos a H22 — UlFor confirma).

### leaderboard.md

Adicionado header `**Iter 0023 (RECON_DELTA)**` no topo. Linhas dos 4 
champions atualizadas:

- **NE** — adicionado pointer para sucessor `ridge+h22_per_fold` 
  (H22 = H22_MA, mesmo champion ridge). H14-G (w=14, k=1) Pareto-supera 
  H14-C marginal.
- **SE** — **DOIS candidatos sucessores agora**: (a) `ridge+h22_per_fold` 
  (recon iter_0021, troca familia) ou (b) NOVO `lr+h22_model_aware` 
  (recon iter_0023, preserva familia LR, R² 0.381 ≈ champion atual 0.383, 
  -11 features = mitiga cond_num).
- **S** — NOVO candidato sucessor `lr+h22_per_fold` (84.3% / +0.387, 
  -2.9pp NMAE).
- **N** — alternativa bias H14-G (w=14, k=1.5): -31.87pp CV vs H14-B 
  -19.63pp; mais conservador (apply 41% vs 100%). PROMOVIVEL.

### open_requests

Continua vazio. **req-0008 (P2 data_publish)** ainda nao emitido formalmente 
mas atratividade SUBE: agora ha 7 candidatos PROMOVIVEIS aguardando holdout 
14d real para decisao Breno informada (era ~3 em recon iter_0021). 
**Recomendacao**: emitir req-0008 na proxima iter pedindo bake-off 14d real 
para o subset PROMOVIVEL (NE ridge+h22, SE lr+h22_MA, SE ridge+h22, 
S lr+h22, N ridge+h22, N bias H14-G). Custo UlFor: ~30-60min execucao. 
Comando ja' documentado no CHAMPION_DECISION_MATRIX (validation gap).

## PHASE C — Handoff

### Hipotese candidata para iter_0024

Pelo planner_config existente (notas_iter0022 do planner), opcoes ranqueadas:

- **(A-recomendado) H27 (P3 ~1h)** — P50 quantile como point estimate 
  substituto via LGBM objective swap; iter_0014 bonus mostrou P50 BATE 
  LGB-mean em magnitude em N (-18%), S (-14%), SE (-1.6%). Custo zero, 
  ganho transversal garantido em N+S. Ortogonal a frente H22_MA/H14-G 
  do UlFor (evita colisao com agentes paralelos). **Sem dep externa.**
- **(B-NOVA pos-iter_0023) H31 emergente (P2 ~0.5h)** — emitir req-0008 
  ao UlFor pedindo holdout 14d real para o conjunto PROMOVIVEL 
  consolidado em CHAMPION_DECISION_MATRIX. **Custo loop**: ~30min escrita 
  + sync coordination/loop_requests.md. **Custo UlFor**: ~30-60min execucao 
  (comando ja definido). **Ganho**: destrava decisao Breno informada com 
  evidencia 14d real para 6 candidatos. Atratividade SUBIU significativamente 
  pos-iter_0023 (3 candidatos extras absorvidos). **Sem dep externa.**
- **(C) H30 (P3 ~1h)** — replicar H21 em Ridge_alpha10 CV; script base 
  `scripts/h21_pdp_residual_cv.py` ja existe.
- **(D) H22 nosso (P3 ~1h, atratividade alta pos-iter_0021)** — GBDT vs 
  OLS gap; carrega lesson H23 ulfor (PI com modelo final).
- **(E) H29 emergente (P2 ~1h)** — aplicar bias_correction per-sub UlFor 
  sobre H10 ensemble LGBM+persist no replay.

### Sequencia recomendada

1. **iter_0024**: H31 emergente (escrita req-0008, low-touch UlFor) — 
   alavanca decisao Breno informada em 7 acoes consolidadas. Sem 
   colisao com agentes paralelos UlFor.
2. **iter_0025**: H27 (P50 substituto, custo zero) OU H30 (Ridge pdp 
   residual) dependendo de progresso UlFor.

### Riscos detectados

- **Discrepancia "Champion ATUAL" no decision matrix** vs state.json — 
  flaggada para investigacao mas NAO bloqueia decisao Breno (matriz 
  inteira ja oferece alternativas, e ridge_NE / lr_SE atuais aparecem 
  como linhas nas comparacoes).
- **Regime UlFor multi-agente continua ativo** — freq de recon loop deve 
  ser max 30-60 min quando UlFor estiver "online" (vs 2-4h offline).
- **Push origin pendente** (UlFor: master 13 ahead origin) — Breno precisa 
  rodar `git push origin master` quando autorizar deploy EC2. Sem 
  impacto direto no loop (read-only via git show).

## Sanity checks

`type=recon_delta` — sanity_checks_required = []. NAO aplicavel.

## Budget

- Estimado: 0.4h
- Real: ~0.4h (leitura 6 commits + 3 docs + sintese)
- Acumulado iter (29-30): ~5.6h / cap 16h diario
