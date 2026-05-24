---
alvo: recon_delta_ulfor_post_c8df4077
layer: meta
iter_num: 0015
type: recon_delta
data_utc: 2026-05-24T12:30:00Z
ulfor_head_inicio: c8df4077
ulfor_head_fim: 5dacb5a2
commits_absorvidos: 7
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.4
---

# Iter 0015 — RECON_DELTA UlFor (c8df4077 → 5dacb5a2)

## Objetivo

Absorver 7 commits novos da sessao UlFor entre o checkpoint 04:00Z
(`c8df4077`, HEAD na entrada do iter_0011) e o checkpoint atual
(`5dacb5a2`, 07:15Z). Recon-only — sem testar hipotese de modelagem,
so reconcilia state/queue/leaderboard com o trabalho que UlFor completou
em paralelo enquanto loop processava H7/H10/H11 (iters 0012-0014).

## PHASE A — Commits inspecionados

Em ordem cronologica (mais antigo primeiro):

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `0481a5a6` | chore — checkpoint 05:15Z | Marker. "H8 ulfor refutada, endpoint operacional, H9 ulfor respondida" referem-se a Hs UlFor internos (Fase 3 PLANO_FINAL), nao ao nosso queue. | absorvido via 4 commits seguintes |
| 2 | `c8e27784` | feat — H10 ulfor clean_plus + Fase 3 DONE + endpoint feature-set aware | **MUDANCA DE CHAMPION**: ridge_curt_n_d1 v2 (clean_plus, 31 features) substitui v1 (full). NMAE 86.3±31.8% (v1) -> **84.8±26.2% (v2)** = -1.5pp mean, -5.6pp std (melhora marginal). lr_N NAO promovido. UlFor H10 clean_plus = CLEAN ∪ {cmo_range, ter_verif_lag1, carga_mwmed_rmean7}. `loader.py` agora le `feature_set` do MLflow run params e aplica drops correspondentes em `build_inference_row` (backward-compat com v1 via param `55_feat_*_v3.3 -> full`). | leaderboard linha N + state.json best_ml_oficial_ulfor_cv_5folds.N |
| 3 | `c0e80193` | feat — validate_d1.py foundation | **FASE 4 STEP 1 INICIADA**. `services/forecasting/validate_d1.py` (247 linhas): replay-predict ultimos N dias usando features as-of D, compara vs realized y_d1 + baseline persist D-1. Loga MLflow experiment `ulfor-validation-d1`. Smoke 14d: **NE 49.2% +9% skill, SE 55.4% +20%, S 137% -37% skill (lr_S regredindo), N 60.5% +45%**. | leaderboard nota iter_0015 + observacao H18 |
| 4 | `26617ba5` | chore — checkpoint 06:15Z | Marker "Fase 3 fechada, Fase 4 iniciada". | absorvido via 3 commits seguintes |
| 5 | `9d7652de` | feat — Dagster asset + schedule daily 07h BRT | Asset `forecast/validate_d1` em `dataops/src/dados_sin/defs/forecast_validation/`. Schedule `forecast_validate_d1_daily` (10h UTC, **STOPPED** ate ativacao Breno). Subprocess pro venv raiz (mlflow nao esta em dataops). Smoke 7d: NE 40.5% +29% skill, SE 71.7% +37%, S 184.3% -41%, N 56.6% +59%. | leaderboard nota Fase 4 + state.json |
| 6 | `9873e3c8` | feat — drift PSI + Telegram alert | **FASE 4 STEPS 2+3 (Opcao C closure)**. Evidently ABANDONADO (conflito `evidently>=0.5 depends on plotly<6` vs nosso pin `plotly>=6.5` — CLAUDE.md gotcha confirmado). PSI nativo numpy/scipy: bins adaptativos ~n/3 max 10, Laplace smoothing, dual `psi_long` (train vs inf) + `psi_recent` (split metade da inf), SEASONAL_FEATURES excluidas. Telegram alert standalone (`services/forecasting/alert.py`) com 4 gatilhos: `skill_vs_persist<0`, `R²<0`, `psi_recent_max>1.0`, `n_features_recent_drift>10`. Smoke 60d (window 30 vs 30): NE psi_recent_max=11.74 39/45 features drift; SE/S 2.32 / N 1.66. **Acao Breno pendente**: gerar `BRAZILGRID_TELEGRAM_BOT_TOKEN/CHAT_ID` + ativar schedule Dagster. | leaderboard nota iter_0015 + observacao H18 |
| 7 | `5dacb5a2` | chore — checkpoint 07:15Z | Marker "Fase 4 step 1 fechada (Dagster+drift+Telegram)". | absorvido via 9873e3c8 |

## PHASE B — Interpretacao consolidada

### Como nossa visao do leaderboard muda

1. **Champion N atualizado de v1 para v2 clean_plus** (commit `c8e27784`).
   Antes: `ridge_curt_n_d1 @staging v1 (full, 55 features)` NMAE 86.3±31.8%.
   Agora: `ridge_curt_n_d1 @staging v2 (clean_plus, 31 features)` NMAE
   **84.8±26.2%** (-1.5pp mean, -5.6pp std). Continua `@staging` (nao
   promovido a `@champion` — fragilidade fold-4 persiste mas atenua).
   linha `curtailment | d1_ENE_CNF | N` do leaderboard atualizada;
   `state.json.best_ml_oficial_ulfor_cv_5folds.N` reescrita com nova
   versao + decisao.

2. **Pipeline Fase 4 (observabilidade) PRONTO operacionalmente** (commits
   `c0e80193`, `9d7652de`, `9873e3c8`). Champions agora tem **3 camadas
   de validacao continua**:
   - `validate_d1.py`: replay diario (MAE/R²/skill_vs_persist por sub)
   - `compute_drift()`: PSI por feature, dual long/recent
   - `alert.py`: Telegram em 4 gatilhos
   Schedule diario STOPPED ate Breno gerar token + ativar. Quando ativar,
   loop ganha um **feed gratuito de validacao** sem ter que rodar audit
   B1-B6 local (H18 segue blocked, mas pressao diminui).

3. **Smoke validate_d1 confirma fragilidade do lr_S**. Em janelas 7d e 14d,
   skill_vs_persist NEGATIVO (-37% a -41%) e NMAE 137-184% — colapso
   operacional. Nao muda champion (CV 5x60d ainda mostra +0.447 R²) mas
   valida o flag "FRAGIL" e a nota `champion-S candidate (std alta —
   monitor pos OOT)` que ja temos em `state.json.best_ml_oficial_ulfor_cv_5folds.S`.

4. **Drift PSI confirma B5 distribution shift documentado em iter_0012**.
   NE smoke 60d: `psi_recent_max=11.74` (vs industry-std threshold 0.2),
   39/45 features driftando. Top features: rolling-means de carga e CMO.
   Reproduz padrao KS p<0.0001 entre fold1 e foldN que documentamos em
   iter_0012 H7. Threshold UlFor calibrado em `>1.0` (vs std 0.2) — vai
   ser recalibrado em 2 semanas de operacao real.

5. **Loader feature-set aware** (commit `c8e27784`). Endpoint
   `/api/forecast/d1` agora aplica drops corretos por champion mesmo quando
   sub usa feature_set diferente (NE/SE/S `full`, N `clean_plus`).
   Implementacao backward-compat com v1 — endpoint continua LIVE sem
   regressao.

### Como nossa queue muda

- **Nenhuma H do loop foi resolvida** pelos commits ulfor. UlFor numera
  seus proprios H1/H2/H4/H8/H9/H10 no PLANO_FINAL — independentemente da
  nossa numeracao H1-H28. Disambiguacao critica:
  - **UlFor H10 (clean_plus)** = FEATURE_DROPS_N = CLEAN ∪ {cmo_range,
    ter_verif_lag1, carga_mwmed_rmean7}. PARCIALMENTE CONFIRMADA.
  - **Nosso H10** (LGBM+persist ensemble) = CONFIRMADO_NE_SE em iter_0013.
  - Nao confundir. Nota ja em `state.json.ulfor_session_sync.note`.

- **H18 (sanity B1-B6 sobre champions Ridge/LR)**: bloqueio segue, mas
  pressao para resolver diminui — pipeline validate_d1 + drift PSI da
  cobertura empirica continua que reduz urgencia do B1-B6 local. Nao
  emitir req-0004 nesta iter (ja decidido em iter_0011, segue valido).

- **H19 (extrair MAE/R²/F1 dos champions)**: agora MAIS facil — UlFor
  loga validate_d1 no MLflow experiment `ulfor-validation-d1` (run id
  smoke `a7230f80`). Se Breno ativar schedule + loop ganhar acesso ao
  MLflow tracking URI, daily summary JSON tem MAE/R²/skill por sub.
  Custo de H19 caiu de "parsing MLflow proxy file" para "parse summary
  JSON do schedule diario". Manter P2 na fila — atrativo cresceu.

- **H12 (feat_carga_history stale)**: ortogonal ao trabalho ulfor.
  Continua blocked por req-0005.

### Reqs

- Sem novos requests pendentes (open_requests=[]).
- Sem novos requests fechados em iter_0015 (todos os 3 antigos seguem DONE
  desde iter_0006; verificado contra `git show 5dacb5a2:coordination/loop_requests.md`).
- Decisao explicita de NAO emitir req-0004/0005 nesta iter (segue
  motivacao iter_0011: evitar duplicar trabalho ja resumido nos
  FINDINGs UlFor + Fase 4 operacional cobre observabilidade).

## PHASE C — Sanity checks

`type=recon_delta` nao requer suite B1-B6 (hard-coded rule: "6 sanity
checks default no final SE for hypothesis_test"). Recon absorve estado
externo, nao testa hipotese.

Auditoria do absorption:
- state.json `ulfor_session_sync` atualizado com `iter0015_inicio_head`,
  `iter0015_fim_head`, `novos_commits_durante_iter0015` (7 entries),
  `delta_resumo_iter0015`.
- state.json `iter_atual=15`, `ultimo_handoff=iter_0015_recon_delta.md`.
- state.json `best_ml_oficial_ulfor_cv_5folds.N` reescrita (v2 clean_plus
  substitui v1 full).
- state.json `open_requests=[]` (verificado contra
  `git show 5dacb5a2:coordination/loop_requests.md` — 3 reqs DONE).
- leaderboard.md: nota Iter 0015 adicionada; linha N champion atualizada
  para v2 clean_plus; seccao "Iter 0015 — RECON_DELTA UlFor" com tabela
  dos 7 commits + interpretacao.
- hypotheses_queue.md: nenhuma H mudou de status (todas as alteracoes
  foram em Hs UlFor internos, nao no nosso queue). H18 detail atualizado
  com nota Fase 4 operacional + H19 detail atualizado com nota
  "summary JSON acessivel via validate_d1".

## Verdict / proxima iter

**Recon limpo**. Sem hipoteses do loop resolvidas, sem novas geradas, sem
novos requests emitidos. Champion N teve upgrade marginal (v1 -> v2
clean_plus). Pipeline Fase 4 inteira (Dagster + drift PSI + Telegram)
fechada operacionalmente pelo UlFor em paralelo aos iters 0012-0014.

`planner_config.next_iter_should_be` mantido: **H21** (P2 feature
engineering `pdp_residual_mwh = pdp_prev - gen`). Razoes inalteradas
desde iter_0014:
- Codavel localmente, sem dep externa.
- Mecanismo identificado em iter_0010 (residual = proxy curt).
- Sanity required: B1 leak (pdp_prev D-1 safe + gen D-only), B2 perm,
  B4 baseline.
- Custo estimado: 1.0h.
- Considerar usar ensemble post-processing (H10 nosso) + LGBM (iter_0012
  confirmado GBDT padrao) para baseline final do bake-off H21.

Alt next: H27 (P50 quantile substituto, ganho universal custo zero,
derivada iter_0014), H24 (ensemble sobre champions Ridge/LR, derivada
iter_0013), H19 (extrair MAE/R²/F1 — atratividade cresceu pos-Fase 4
UlFor, summary JSON disponivel).

## Lessons learned

- **Recon iter continua barato e protege**. UlFor entregou Fase 4 inteira
  (Opcao C completa) em ~5h paralelos aos nossos iters 0012-0014 (H7+H10+H11
  ~3.0h). Sem o recon, iter_0016 poderia tentar "loop implementa validate_d1
  proprio" — UlFor ja' fez.
- **Champion N evolui silenciosamente em CV**. iter_0011 marcou
  ridge_N@staging v1 (full); 1 commit depois, v2 clean_plus (-5.6pp std).
  Tracking versao do champion na linha do leaderboard e' obrigatorio
  (nao so' modelo + NMAE — incluir feature_set + versao MLflow).
- **Pipeline observabilidade UlFor diminui urgencia do H18**. validate_d1
  + drift PSI + Telegram alert dao cobertura continua que reduz valor
  marginal do B1-B6 audit local. Loop pode investir tempo em hipoteses
  novas (H21+ feature engineering) em vez de tentar replicar audit.
- **Disambig nomenclatura H persistente**. Comecando a haver muita colisao:
  nosso H10 (ensemble) != UlFor H10 (clean_plus). Convencao proposta:
  prefixar Hs externos como `Hxx_ulfor` em referencias internas. Aplicar
  no proximo recon.
