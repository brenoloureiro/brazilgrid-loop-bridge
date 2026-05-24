---
alvo: recon_delta_ulfor_post_27152e16
layer: meta
iter_num: 0031
type: recon_delta
data_utc: 2026-05-25T02:30:00Z
ulfor_head_inicio: 27152e16
ulfor_head_fim: 2917289c
commits_absorvidos: 8
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.3
---

# Iter 0031 — RECON_DELTA UlFor (27152e16 → 2917289c)

## Objetivo

Absorver 8 commits novos da sessao UlFor entre `27152e16` (HEAD ao fim do
iter_0028 — checkpoint marker 16:02Z apos parallel agent fechar pendencias
val14d) e `2917289c` (checkpoint 16:48Z apos Breno responder PARAR-E-
PERGUNTAR mas standby por bloqueio infra). Janela ~28 min reais UlFor
(13:20-13:49 BRT — commit timestamps). Conteudo: **decisao Breno oficial
promote v3 final** (commit `0971c699`) + **finding CH local stale** (commit
`6111cda4`) + 6 checkpoints PARAR-E-PERGUNTAR identicos do runner heartbeat
em standby.

## PHASE A — Commits inspecionados

Em ordem cronologica:

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `0971c699` | docs(forecast) DECISAO BRENO promote v3 | **DECISAO OFICIAL** — SE opt A (`ridge+h22_MA+α=1`) escolhido; H14-G em promote_champions.py NAO loader.py; matriz final 4 decisoes | leaderboard nova secao + state delta_resumo + queue notas_iter0031 |
| 2 | `6111cda4` | docs(forecast) FINDING_LOCAL_CH_STALE_FEAT_TABLES | Achado infra: feat_termico congelada 2024-12-31 local; promote NAO executado | state + leaderboard nota bloqueio |
| 3 | `655729cd` | checkpoint 16:27Z (heartbeat) | PARAR-E-PERGUNTAR migra para "Breno decide EC2 setup vs skip" | state (marker) |
| 4 | `759d6cfe` | checkpoint 16:29Z (heartbeat) | ~2 min apos 16:27Z, refinamento wording | state (marker) |
| 5 | `d2be5a26` | checkpoint 16:39Z (forced #2) | runner force iters-sem-commit threshold; standby legitimo | state (marker) |
| 6 | `cf3097cd` | checkpoint 16:38Z (3o standby) | tunnels CH+MLflow exit=28 | state (marker) |
| 7 | `91ecfc5b` | checkpoint 16:47Z (4o forcado) | padrao auto-repete a cada ~9 min | state (marker) |
| 8 | `2917289c` | checkpoint 16:48Z (HEAD atual) | confirma autopilot em standby aguardando Breno infra | state (marker) |

**Composicao**: 2 substantivos (25%) + 6 checkpoints PARAR-E-PERGUNTAR (75%).
Padrao "runner forca checkpoint a cada ~9-10 min em standby legitimo"
permanece — mesmo padrao identificado em iter_0028 (d9aefdd1 forcado +
27152e16 marker).

### Commit 1 — `0971c699` (DECISAO BRENO oficializada)

**FINDING_RIDGE_ALPHA_SWEEP.md** (+83 linhas, novo bloco "Sprint 1 (sessao
15:00Z) — DECISAO BRENO promote v3"):

| sub | acao | champion v3 final | fonte primaria |
|---|---|---|---|
| NE | PROMOVER | `ridge + h22_per_fold + α=1` | CV + val14d coincidem |
| SE | **PROMOVER opt A** | **`ridge + h22_model_aware + α=1`** | **val14d** > CV (regime drift) |
| S | MANTER status quo | `lr + full` | val14d refuta h22_* |
| N | PROMOVER | `ridge + h22_per_fold + α=100` | CV + val14d coincidem |

Justificativa Breno opt A SE:
- val14d e' sinal mais recente do regime real (test 2026-03-24 → 2026-05-21):
  CMO subindo + intercambio SE-S invertendo + parque eolico/solar NE em
  expansao.
- As 10 features que `h22_MA` preserva e `h22_pf` dropa (CMO + intercambio +
  taxa_penetracao) carregam sinal nesse regime.
- val14d delta MA-pf = **−1.64pp NMAE** + R² +0.05 a favor de `h22_MA`.
- Para 14d a frente, sinal recente prevalece sobre robustez agregada.
- Aceita perda de coerencia multi-sub (SE usa `h22_MA`, NE/N usam `h22_pf`)
  em troca de fit ao regime atual. Reavaliar Jun/2026.

**CHAMPION_DECISION_MATRIX.md** (+51 linhas, secao "Decisao Breno — promote
v3"): matriz 4-decisoes + secao "H14-G bias correction NE — implementar em
promote_champions.py (futuro)".

**H14-G decisao**: bias correction (w=14, k=1) e' **artefato promovido**
(binding ao modelo). MLflow tag deve carregar bias spec (`bias_window`,
`bias_threshold_k`, `bias_strategy`). `loader.py` deve ser **dumb** (carrega
artefato + aplica `BiasCorrector` parametrizado pelo tag).

UlFor registrou como **H25_ulfor** para sprint envelope-safe proxima.
Implementacao: arg `--bias` em `promote_champions.py` + log spec + tag +
artefato auxiliar + loader instancia `BiasCorrector`. **NAO** confundir com
nosso H25 (Stacker Ridge meta-modelo, P3 queued).

### Commit 2 — `6111cda4` (FINDING_LOCAL_CH_STALE_FEAT_TABLES)

CH local Docker (porta 8123) tem `feat_*` tables criticamente stale:
- `feat_termico`: `max(dia) = 2024-12-31` (zero 2025/2026)
- `feat_carga_history`: `max = 2026-03-26` (2 meses stale)
- `feat_pld / pdp / inter / sat / cmo`: 8-23 dias stale
- `obt_usina_enriched`: `max = 2026-05-15` (regressao ~6 dias)

Sprint envelope-safe (b)+(c) do checkpoint 16:02Z falhou:
- Alpha sweeps N{200,500} e SE{0.1,0.5} produziram "WARN: pos-dropna
  train=16 test=0 — skip ML" em todos 4 subs.
- Reproduzido com α=1: mesma falha → **nao e' bug do sweep, e' data-stale
  local**.
- Sweep α=1/100 commitado em `f7c56c3d` (iter_0028) rodou contra **EC2 CH
  via tunnel SSH** (porta 18123 = SSH forward), nao Docker local.

5 parquets lixo apagados. Bloqueia tambem (a) smoke loader (MLflow tunnel
down) e (d) H23 LOI pares.

**Convergencia 1:1 com auto-memory**:
- [[ch_local_feat_termico_stale]] — Breno Mai/26: "CH local Docker tem
  `feat_termico` congelada 2024-12-31 (e gaps menores em outras feat_*).
  Bloqueia bake-off/promote local."
- [[ch_local_vs_ec2_separados]] — Breno: "MCP brazilgrid aponta pro CH
  local desktop. Pra estado real de PROD usar `docker exec brazilgrid-
  clickhouse` via SSH no EC2."

UlFor reproduziu independentemente o diagnostico. **Nao gera req formal** —
ja' documentado no auto-memory. Acao Breno (EC2 setup ou sync raw).

### Commits 3-8 — 6 checkpoints PARAR-E-PERGUNTAR (heartbeat noise)

Markers identicos do runner headless apos PARAR-E-PERGUNTAR migrar de
"Breno decide promote v3" (resolvido 16:00Z) para "Breno desbloqueia infra"
(EC2 setup ou skip-local-revalidate). Padrao "runner forca checkpoint a
cada ~9-10 min em standby legitimo". Conteudo: tunnels CH+MLflow exit=28
(forwards SSH nao conectam). Nada mudou desde 16:29Z.

## PHASE B — Atualizacoes feitas

### state.json

- `iter_atual: 30 → 31`
- `mode: self_planning → recon_delta`
- `alvo_ativo: h15_s_classifier_vs_regressor → recon_delta_ulfor_post_27152e16`
- `layer_ativa: curtailment → meta`
- `ulfor_session_sync`:
  - `iter0031_inicio_head = 27152e16`
  - `iter0031_fim_head = 2917289c`
  - `novos_commits_durante_iter0031`: 8 commits absorvidos com interpretacao
  - `delta_resumo_iter0031`: champions inalterados em prod, promote v3
    DECIDIDO (Breno) MAS NAO EXECUTADO (infra); migra de "aguarda decisao"
    para "aguarda infra"; H25_ulfor introduzido para bias correction em
    promote_champions.py
  - `h_loop_impactadas`: H8 (3a evidencia independente), H33 (atratividade
    diminui mais), H30 (inalterado), H25 nosso (inalterado), H18 (blocked-
    mais-critica)

### leaderboard.md

- Atualizada nota do topo refletindo iter_0031 RECON_DELTA
- Nova secao "Decisao Breno oficial v3 final" com tabela 4 decisoes,
  justificativa SE opt A, convergencia 3-fold H8, decisao H14-G
- Nova secao "Promote v3 NAO EXECUTADO — bloqueio infra" documentando
  feat_termico 2024-12-31, MLflow offline, sweep α=1/100 rodou via EC2
  tunnel

### hypotheses_queue.md

- `last_updated: 2026-05-25T02:30:00Z`
- `H33.notes_iter0031`: atratividade diminui mais (caveat model-aware
  resolvido empiricamente por 3-fold convergence; nenhuma decisao depende)
- `notas_iter0031` bloco completo: inspected_range, attractiveness_changes,
  promote_v3_decidido_mas_nao_executado, h14g_implementacao_decidida,
  3_fold_convergence_h8_se

### open_requests

Inalterado: `[]`. Nenhum req novo emitido. Nenhum req fechado pelo delta
UlFor (req-0007 etc. ja DONE; novos achados nao geram req formal — todos
ja' em auto-memory ou pre-empcao).

## PHASE C — Hipotese candidata para iter_0032

Estado da queue (P3 candidates ativos, custo baixo):

| H | summary | cost | atratividade pos-iter_0031 |
|---|---|---|---|
| H30 | pdp_residual em Ridge_α10 CV (rodar α=1 E α=10) | ~1.0h | INALTERADO (continua atrativa) |
| H33 | joint-drop SE em LR vs Ridge | ~0.5h | DIMINUI MAIS (caveat resolvido) |
| H25 | Stacker Ridge meta-modelo | ~2.0h | INALTERADO |
| H26 | Conformal prediction post-hoc P10/P90 | ~2.0h | INALTERADO |
| H27 | P50 quantile como point estimate N+S | ~1.0h | INALTERADO |
| H28 | NGBoost vs LGBM quantile | ~3.0h | INALTERADO |
| H35 | S alerta binario via LogReg dedicado | ~2.5h | INALTERADO |
| H20 | (P3 queued, ver linha 689 queue) | — | INALTERADO |

**Recomendacao para iter_0032**: monitor recon (60-90 min) OR **H30**.

Razao para preferir H30:
- Pos-iter_0028 + iter_0031, alpha=1 e' o regime dominante em 3/4 subs
- H30 pode rodar α=1 E α=10 simultaneamente (custo zero adicional)
- Fecha H3-family residual no replay loop (CONFIRMADO_RIDGE ou REFUTADO_RIDGE)
- Sem dep externa, sem req UlFor pendente

Alternativa: continuar monitor recon se Breno comecar a desbloquear infra
EC2 (proximos commits UlFor vao trazer promote v3 executado real).

## Padrao UlFor multi-agente — 12a iter consecutiva

UlFor PARAR-E-PERGUNTAR permanece envelope-respeitado. Padrao confirmado:
1. PARAR-E-PERGUNTAR quando decisao requer Breno
2. Runner forca checkpoints a cada ~9 min em standby legitimo (heartbeat)
3. Loop NAO emite req formal — auto-memory + state.json delta cobre o
   contexto

Branch UlFor agora ~32 ahead origin (iter_0028 era 26+, iter_0031 ainda
sem push). Promote v3 pendente push + execucao em EC2.

## Convergencia 3-fold H8 SE (lema metodologico mais robusto do loop)

Breno escolher opt A em SE = 3a evidencia independente confirmando lesson
H8 iter_0027 (preservar intercambio em SE Ridge α=1 carrega sinal real):

| iter | metodologia | resultado |
|---|---|---|
| 0027 | H8 joint-drop refit (Ridge α=1 SE bundle intercambio) | +1.21pp NMAE HARMFUL |
| 0028 | val14d real (h22_MA vs h22_pf comparison) | −1.64pp NMAE opt_A wins |
| 0031 | Breno escolhe opt A explicitamente | val14d trumps CV+coerencia |

Mesma direcao, magnitudes consistentes (~1.5-1.6pp), contextos independentes
(CV-PI joint-drop, val14d alpha sweep, decisao operacional). **Lesson
permanece o lemma metodologico mais robusto do loop ate' agora**: PI deve
ser medida com o modelo final (model-aware H22_MA / H23_ulfor); features
preservadas por LR-PI MELHORAM val14d em ridge tambem.

## Budget

0.3h alocados, ~0.3h consumidos (read-only inspection + 4 file updates).
Dentro do envelope.

## Sanity checks

N/A — recon_delta nao executa modelo, nao testa hipotese. Audit puro.
