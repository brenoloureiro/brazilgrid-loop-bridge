---
alvo: recon_delta_ulfor_post_4e0fc7b4
layer: meta
iter_num: 0011
type: recon_delta
data_utc: 2026-05-24T08:30:00Z
ulfor_head_inicio: 4e0fc7b4
ulfor_head_fim: c8df4077
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

# Iter 0011 — RECON_DELTA UlFor (4e0fc7b4 → c8df4077)

## Objetivo

Absorver 7 commits novos da sessao UlFor entre o checkpoint 02:15Z
(`4e0fc7b4`, HEAD na entrada do iter_0007) e o checkpoint atual
(`c8df4077`, 04:00Z + 3 acoes downstream). Recon-only — nao executa
nova hipotese de modelagem, apenas reconcilia state/queue/leaderboard
com o trabalho que UlFor completou em paralelo.

## PHASE A — Commits inspecionados

Em ordem cronologica (mais antigo primeiro):

| commit | tipo | impacto | absorvido em |
|---|---|---|---|
| `d1fe9777` | feat — CV walk-forward 5 folds | H2 ulfor CONFIRMADA (Ridge nao e overfit acidental); materializa os numeros oficiais ja absorvidos retroativamente no iter_0007 via checkpoint `4e0fc7b4`. MLflow: 140 runs + 28 CV_SUMMARY logged. | state.json novos_commits_durante_iter0011 |
| `83bc79c2` | feat — promote_champions | **MUDANCA DE STATUS**: candidates -> PRODUCAO. 4 modelos registrados no MLflow Registry com aliases: ridge_curt_ne_d1@champion, lr_curt_se_d1@champion, lr_curt_s_d1@champion, ridge_curt_n_d1@staging. In-sample R²: NE 0.830, SE 0.619, S 0.725, N 0.472 (gap CV ~0.24-0.36, overfit modesto esperado de modelos lineares). | leaderboard tabela top + state.json |
| `3ac5916a` | feat — VIF + CLEAN bake-off | H4 ulfor CONFIRMADA estrutural (38/55 features VIF>=10, cond_num matriz X >1e17, padroes obvios: ger_renovavel=eolica+solar corr+1.000, val_net=export-import -0.998, pdp_prog vs pdp_prev +0.995). PARCIAL acionavel: CLEAN ajuda NE (-0.5pp NMAE, stdev MENOR) e N (-17.7pp, stdev halved), mas HURTS SE (+15pp NMAE, R² cai 0.36) e S (+25.7pp, R² cai 0.35). DECISAO UlFor: manter feature_set=full default. CLEAN candidato Staging NE proximo round. | leaderboard nota iter_0011 |
| `4d6dd73a` | chore — checkpoint marker 04:00Z | Sessao autopilot 03:00-04:00Z, 3 commits + checkpoint. Marker. | absorvido via 83bc79c2 |
| `d2bf38e4` | feat — VIF greedy iterativo | H8 ulfor REFUTADA: greedy converge em 16-18 drops/sub mas dropa drivers economicos primarios (carga_mwmed, cmo_mwmed, ger_eolica_mwh). Score final CV 5x60d: CLEAN vence 7 cells, FULL 3, GREEDY 1 (SE ridge). Multicolinearidade estatistica != redundancia preditiva. | leaderboard nota iter_0011 |
| `a7edb1ef` | feat — endpoint /api/forecast/d1 | **PRODUCAO LIVE**. FastAPI router em services/analytics_api/routes/forecast_d1.py carrega champions via MLflow Registry. Smoke test: /api/forecast/d1/NE = 62.3k MWh (persist 59.1k, ma7 36.2k). Cold 6.5s (CH+download model), warm <50ms (cache 1h modelo + 15min predicao). Safe-failing 503 se MLflow inacessivel. | leaderboard linhas curtailment/d1 + state.json |
| `c8df4077` | feat — investigate_lr_N instability | UlFor H9 ulfor RESPONDIDA. Fold 4 (jul-set/2025, train=221d, mais antigo) = blowup: FULL NMAE 208.6%, CLEAN 128.5% (stdev 52.4->22.7pp). Envenenadoras recorrentes FULL (Δ<-0.5pp em >=2 folds): cmo_range, taxa_penetracao, ter_verif_lag1, carga_mwmed_rmean7. CLEAN ja dropa taxa_penetracao; outras 3 candidatos `FEATURE_DROPS_N`. Em CLEAN ainda aparecem curt_lag1/curt_rmean7 instaveis -> N tem muitos zeros estruturais e OLS extrapola mal. DECISAO UlFor: ridge_curt_n_d1@staging continua a defesa, nao promover lr_N. Possivel feature_set=clean_plus_n proximo round. | leaderboard nota iter_0011 |

## PHASE B — Interpretacao consolidada

### Como nossa visao do leaderboard muda

1. **Champions PROMOVIDOS**. Antes do iter_0011, leaderboard tinha as 4 linhas
   `curtailment | d1_ENE_CNF | <sub>` marcadas como "aud B1-B6 pendente (H18)".
   Continuam pendentes de B1-B6 local, mas o status interno mudou: nao sao
   mais "@champion candidate (await OOT 2x)" — sao **@champion (NE/SE/S) e
   @staging (N) no MLflow Registry**, com endpoint /api/forecast/d1 servindo.
   In-sample R² adicionado a cada linha (NE 0.830, SE 0.619, S 0.725, N 0.472).
   last_iter atualizado de 0007 -> 0011 com nota "endpoint /api/forecast/d1 LIVE".

2. **H4 ulfor (VIF) NAO destrava drop universal**. Apesar de multicolinearidade
   massiva confirmada (38/55 features VIF>=10), FEATURE_DROPS_CLEAN ajuda so
   NE/N. UlFor mantem feature_set=full default. Implicacao para o loop:
   nao adianta pushar uma hipotese "loop testa CLEAN replay iter_0002" porque
   UlFor ja' rodou isso em janela melhor (5 folds × 60d vs nosso n=11). H21
   permanece o next.

3. **H8 ulfor (VIF greedy) REFUTADA**. Greedy puro nao bate CLEAN curado.
   Confirma intuicao do loop iter_0010: o que importa e' a SEMANTICA do drop
   (pdp_prog vs pdp_prev e' redundante porque um copia o real e o outro e'
   previsao independente, NAO porque correlacao e' alta).

4. **lr_N continua nao promovivel**. Fold 4 jul-set/2025 e' o blowup;
   `feature_set=clean_plus_n` (CLEAN + drop 3 features adicionais)
   sugerido pelo UlFor como proximo round.

### Como nossa queue muda

- **Nenhuma H do loop foi resolvida** pelos commits ulfor. UlFor numera
  seus proprios H1/H2/H4/H8/H9 no PLANO_FINAL — independente da nossa
  numeracao H1-H22. **Particular cuidado**: commit `c8df4077` diz "H9
  RESPONDIDA" referindo-se ao H9 ulfor (lr_N investigation), **nao** ao
  nosso H9 (metric_suite MAE/R²/F1, iter_0008 CONFIRMADO). Disambig
  registrada em state.json `nota_nomenclatura`.

- **H18 (sanity B1-B6 sobre champions Ridge/LR)**: bloqueio segue. Champions
  agora em MLflow Registry, mas loop nao tem acesso direto ao MLflow
  tracking URI. Bloqueio req-0005 (pedir predicoes em parquet) ou criar
  req-0004 (dump CV_SUMMARY MLflow). Decisao: **nao emitir req-0004
  nesta iter** — auto-pesado vs valor incremental, ja' temos
  FINDING_MULTICOLINEARITY + FINDING_LR_N_INSTABILITY resumindo o que B1-B6
  reconfirmaria.

- **H17 (SUPERSEDED iter_0007)**: continua done. Adicionar nota em proxima
  inspecao ao queue de que "SE/S champions estao agora em PRODUCAO" e' opcional.

### Reqs

- Sem novos requests pendentes (open_requests=[]).
- Sem novos requests fechados em iter_0011 (todos os 3 antigos ja DONE
  desde iter_0006).
- Nenhum req-0004/0005 emitido (decisao explicita: evitar duplicar
  trabalho ja resumido nos FINDINGs UlFor).

## PHASE C — Sanity checks

`type=recon_delta` nao requer suite B1-B6 (ver hard-coded rule:
"6 sanity checks default no final SE for hypothesis_test"). Recon
absorve estado externo, nao testa hipotese.

Auditoria do absorption:
- ✅ state.json `ulfor_session_sync` atualizado com `iter0011_inicio_head`,
  `iter0011_fim_head`, `novos_commits_durante_iter0011` (7 entries),
  `delta_resumo_iter0011`.
- ✅ state.json `iter_atual=11`, `alvo_ativo=recon_delta_ulfor_post_4e0fc7b4`,
  `ultimo_handoff=iter_0011_recon_delta.md`.
- ✅ state.json `open_requests=[]` (verificado contra
  `git show c8df4077:coordination/loop_requests.md` — 3 reqs DONE).
- ✅ leaderboard.md: nota Iter 0011 adicionada, 4 linhas champion
  atualizadas com status PROMOVIDO + R² in-sample, seccao "Iter 0011 —
  RECON_DELTA UlFor" com tabela de 7 commits + interpretacao.
- ✅ hypotheses_queue.md: H18 detail atualizado com nota
  iter_0011 (status MLflow Registry + endpoint LIVE).

## Verdict / proxima iter

**Recon limpo**. Sem ajuste de hipoteses ja done, sem novas hipoteses
geradas, sem novos requests emitidos. Champions promovidos no MLflow +
endpoint LIVE = a "FASE 4" prometida em H17 efetivamente entrou em
producao via autopilot UlFor.

`planner_config.next_iter_should_be` mantido: **H21** (P2 feature engineering
`pdp_residual_mwh = pdp_prev - gen`). Razoes:
- Codavel localmente, sem dep externa.
- Mecanismo identificado em iter_0010 (residual = proxy curt) sugere
  reducao de 2 features para 1 canal denso sem perda de skill.
- Sanity required: B1 leak (pdp_prev D-1 safe + gen D-only), B2 perm,
  B4 baseline.
- Custo estimado: 1.0h.

Alt next: H10 (P2, ensemble v2_LGBM + persist_d1), H22 (P3 GBDT vs OLS
gap no curt~gen+pdp), H19 (P2 extrair MAE/R²/F1 dos champions — agora
talvez parseavel via commit 3ac5916a CV outputs em
`experiments/forecast/curtailment_d1/FINDING_MULTICOLINEARITY.md`).

## Lessons learned

- **Recon iter e' barato e protege contra duplicar trabalho**. UlFor
  resolveu 4 acoes em ~5h em paralelo enquanto loop processava H3 no
  iter_0010. Sem o recon, iter_0012 poderia tentar uma hipotese ja'
  superseded (ex: "testar drop CLEAN no NE" — UlFor ja' fez).
- **Cuidado com colisao de nomenclatura H<n>**. UlFor numera seus
  proprios H's no PLANO_FINAL. Sempre verificar contexto antes de
  marcar como done. Disambig em `state.json.nota_nomenclatura`.
- **Champions promovidos NAO automaticamente destravam sanity B1-B6**.
  MLflow Registry e' utilidade do **deploy**; loop precisa de acesso
  a predictions/features serializadas (parquet) para auditar
  independentemente. H18 segue blocked ate req-0005.
