---
iter_num: 0001
type: recon_readonly
target: forecaster_inventory
started_at: 2026-05-23T23:58:01Z
finished_at: 2026-05-24T00:03:00Z
sanity_checks_passed: N/A (recon)
---

# Iter 0001 — Recon read-only da sessão de forecaster ativa

Objetivo: inventariar a sessão Claude Code paralela rodando em
`C:/Projetos/brazilgrid-ulfor` e as 4 worktrees forecast adjacentes,
sem nenhuma modificação fora deste loop. Decidir se o loop espera,
absorve, pivota ou retroage.

## Estado da sessão ativa (brazilgrid-ulfor / master)

- HEAD: `319633a9` "feat(forecast): bakeoff v3 — add feat_pdp_renovavel (QW3)"
  (2026-05-23 20:58:57 -03 = ~5min antes desta recon começar).
- Working tree: **CLEAN, alinhado com origin/master** no instante T0+1
  da recon. (Em T0, segundos antes, havia `M experiments/bakeoff_curtailment_multisub/bakeoff_d1.py`
  uncommitted — comitada e pushada como `319633a9` durante a recon.)
- Foco aparente: projeto **UlFor** (Ultimate Forecaster), consolidação
  decidida com o Breno em 2026-05-22 para substituir os ~9 lugares
  espalhados de forecast. Hoje (23/05) a sessão fez ~30 commits em
  ~7 horas, com cadência de sprint:
  - 14:18-14:25 FASE 2 Frente A (delete DEAD dirs, dedup models/, untrack charts/)
  - 15:16-15:36 Sprint 1+2 (feat_intercambio, feat_geracao_history, prevcarga SINtegre G1+G2)
  - 15:52 Sprint 3 (5 feat_ models: carga, mmgd, cmo, pld, hidro)
  - 16:25-16:30 Sprint 4 (top_conjuntos 61→228, feat_termico G8)
  - 17:17-17:40 FASE 3 BAKEOFF D+1 multi-subsistema (NE/SE/S/N)
  - 19:24 UlFor bakeoff v2 — feat_previsao_renovavel (G14 fechado)
  - 20:32-20:58 G12 crosswalk PDP usina (SIGA+dim_usina+BDGD) → feat_pdp_renovavel → bakeoff v3 QW3
- A criação deste worktree-loop (commits 9712b8e2, 4772942a, 3d830bd3,
  8c3d5994 entre 20:49-20:53) é da MESMA sessão — ela montou o
  scaffold do loop entre Sprint 4 e G14, depois voltou pro tronco
  UlFor. Logo o loop nasceu sancionado pela própria sessão ativa.

## Outras worktrees forecast

| Worktree | Branch | HEAD | Última atividade | Foco |
|---|---|---|---|---|
| brazilgrid-ulfor | master | 319633a9 (sync c/ origin) | 2026-05-23 20:58 | **ATIVA** — UlFor FASE 1-3, bakeoff D+1 multisub |
| brazilgrid-bench-forecast | bench/forecast-2026 | f318bc55 | 2026-05-11 06:54 | DORMENTE — V3 hybrid + NGBoost bandas P10/P50/P90, 16-libs bench |
| brazilgrid-bigforecaster | bigforecaster | 1c99354d | 2026-05-22 16:57 | DORMENTE — V72/V75 serializados em joblib; codigo legado consumido pelo UlFor |
| brazilgrid-eolica-serving | fix/carga-ddl-schema | a652718b | 2026-05-22 10:34 | DORMENTE — fix DDL `ons_carga_verificada` |
| brazilgrid-perf-metrics-iec | feat/perf-metrics-iec | 7c0b34bb | 2026-05-09 21:15 | DORMENTE — IEC 61724-3 POC + BigEfi v0.1, sem relação com forecast |

Todas as 4 worktrees não-ulfor têm working tree limpo (não verificado
exaustivamente porque `git -C ... status --porcelain` retornou vazio
nas 4 — alinhadas com sua branch).

Branches forecast adicionais no remote (sem worktree dedicada):
`origin/feat/eolica-previsao-conjunto`, `origin/fix/coff-eolica-backfill`,
`origin/feat/curtometro-react`. Locais legadas: nenhuma fora das 4 acima.

## Documentos de plano/handoff encontrados

Sob `C:/Projetos/brazilgrid-ulfor/`:

- `docs/forecast/PLANO_FINAL.md` — 27 KB, mtime 2026-05-23 15:24.
  Plano mestre UlFor, 6 fases (0-5). Cabeçalho diz: "Substitui o antigo
  PLANO_UNIFICACAO.md". Princípio 5: "Persistencia (D+1 = hoje mesma
  hora) e o baseline a vencer — sempre". Princípio 6: "Metricas
  honestas: para curtailment usar MAE/R2/F1, nunca NMAE/media".
- `docs/forecast/CATALOGO_FONTES.md` — 11 KB, mtime 23/05 15:24.
  7 famílias de fonte (ONS S3, ONS Integra, SINtegre, CCEE, ANEEL,
  Decks DESSEM/NEWAVE/DECOMP, NWP Open-Meteo). Status por dataset.
- `docs/forecast/EDA_FASE1.md` — 10 KB, mtime 22/05 19:55.
  Entregável FASE 1.3: cobertura/missing/sazonalidade/Fourier das
  feat_ tables; baseline de persistência D-1/D-7 logado em MLflow
  experiment `ulfor-fase1-eda`, run `eda_fase1_20260522T225210Z`.
- `coordination/forecast_session_state.md` — 14 KB, mtime 22/05 15:29
  (**desatualizado vs trabalho de hoje**). Última entrada relevante
  é V3 multi-area 2026-05-11. Não foi tocado nos sprints UlFor de hoje.
- `coordination/benchmark_session_state.md` — 3.4 KB, mtime 22/05 15:29.
  Idem, não atualizado.
- `applied_p0_report.md` — 13 KB, raiz, mtime 22/05 15:29. Não inspecionado.
- `docs/forecast/legacy/` — preservados como referência (commit `d20423e7`).

Nenhum `handoff_*.md`, `checkpoint_*.md`, `roadmap_*.md` modificado
nos últimos 7 dias fora dos artigos do portal Insights e dos doc
files do dataops/decks/deploy/data — todos não relacionados ao
escopo desta recon.

## Artefatos ML modificados <7d

Surpresa: **nenhum .joblib novo de hoje**. A sessão UlFor está em
FASE 1-3 (catálogo, features, primeiro bakeoff) — ainda não serializou
modelos pelo MLflow. .joblib presentes no repo são todos pré-existentes:

- `experiments/forecast_curtailment/fase4_v3/models/00_baseline.joblib`,
  `01.joblib`, `03.joblib`, `04.joblib`, `05.joblib`, `08.joblib`,
  `09.joblib`, `10.joblib`, `11.joblib` — frente Fase 4 v3 anterior.
- `products/cmo_engine/models/dessem_surrogate_*_{lgb,ridge,xgb}.joblib`
  (8 arquivos) — surrogate DESSEM CMO/curt NE.

Arquivos `.py` mudados <7d relevantes ao forecast (ulfor, fora `.venv`/`archive`):

- `experiments/bakeoff_curtailment_multisub/bakeoff_d1.py` — 19 KB,
  mtime 23/05 20:58. **Hot path** — script da bakeoff D+1 multisubsistema
  que rodou 3 versões hoje (v1, v2, v3) acrescentando features sucessivas.
- `services/forecasting/curtailment_d1_por_razao.py` — 14.6 KB, mtime 23/05 15:11.
  Cano principal canonicalizado (FASE 2).
- 6 outros .py em `services/forecasting/` (`features.py`, `models.py`,
  `cv.py`, `predict.py`, `ons_realtime.py`, `area_mapping.py`) — todos
  mtime 22/05 15:29 (FASE 2 Frente B).
- 15 modelos dbt em `dataops/dbt/models/features/feat_*.sql` — todos
  com mtime ≥22/05: calendario, carga_history, cmo, curtailment_history,
  geracao_history, gibr, hidrologico, intercambio, mmgd, nwp,
  pdp_renovavel, pld, previsao_renovavel, saturacao, termico,
  + `_feat__models.yml`. Camada feat_ inteira construída em 36 horas.
- Outros: `bigefi/backend/*`, `brazilgrid/analysis/*`, `dataops/scripts/eda_feat_layer.py`,
  `data/weathernext/test_zarr_access.py`. Periféricos.

## Hipóteses sobre o que está sendo feito agora

Ranqueadas por evidência:

1. **(alta)** Execução linear do `PLANO_FINAL.md` Fase 0→5. Hoje fechou
  FASE 1 (camada feat_ inteira), FASE 2 (canonicalização services/forecasting),
  FASE 3 entrou no bakeoff. Próximo é treinar/serializar/registrar
  modelos via MLflow (FASE 3 termina; FASE 4 = modelo champion).
  Evidência: ordem temporal exata dos commits, prefixos "UlFor FASE N"
  /"Sprint N"/"G<id>".
2. **(alta)** Foco específico em **curtailment D+1 multi-subsistema**
  (NE/SE/S/N). bakeoff_d1.py rodou 3 versões hoje, cada adicionando
  uma feature (v1 base → v2 feat_previsao_renovavel → v3 feat_pdp_renovavel).
  Evidência: nome do experimento + 3 commits seguidos do mesmo arquivo.
3. **(média)** Pausa atual (~5min sem novos commits no instante da recon).
  Provavelmente: planejando v4 (próxima feature do catálogo a entrar
  no bakeoff), inspecionando outputs do v3, ou comparando bakeoff vs
  baseline de persistência. Tree clean + push recente reforçam.
4. **(média)** Sessão ativa não pretende tocar bigforecaster/bench-forecast
  /eolica-serving/perf-metrics-iec. O UlFor está reaproveitando o
  conhecimento dessas worktrees (V72/V75, V3 hybrid bandas) via
  documentação no `PLANO_FINAL.md §1.1` mas não fazendo checkout
  dessas branches.
5. **(baixa)** Worktree-loop foi criado às 20:49-20:53 como infra
  paralela. Não houve novo commit no loop desde então — sessão
  ativa devolveu controle do loop para "outra coisa" (humano ou
  outra sessão). Esta recon é exatamente esse "outra coisa".

## Conflitos potenciais com o loop

- **Branches:** sessão ativa em `master`; loop em `feat/forecast-mega-loop-scaffold`
  (ramificada de master). Sem risco de conflito git enquanto o loop
  só editar `loops/forecast-mega-loop/`. Risco aparece se o loop quiser
  abrir PR pra master — esperar momento entre sprints UlFor.
- **ClickHouse `dados_sin` (local Docker):** ambas as sessões podem
  consultar concorrentemente. Sessão ativa também faz `dbt run` em
  `feat_*` (writer em `dados_sin`). Loop deve evitar `dbt run` na
  mesma janela; SELECTs read-only seguros.
- **MLflow tracking:** sessão ativa loga em experiment `ulfor-fase1-eda`
  (e provavelmente um `ulfor-fase3-bakeoff` agora). Loop deve usar
  experiment separado (ex.: `forecast-mega-loop`) pra não poluir
  histórico champion-tracking.
- **.joblib serializados:** sessão ativa AINDA NÃO produziu .joblib
  novo hoje. Quando produzir (FASE 4), gravará via MLflow model registry,
  não direto no repo. Sem conflito de arquivos.
- **Outras worktrees dormentes:** sem risco — todas com último commit ≥1 dia
  e working tree limpo.
- **Memória `~/.claude/projects/.../memory/`:** ambas as sessões podem
  escrever. Risco baixo (escritas raras, append a MEMORY.md). Loop
  evitar `/remember` durante sprint ativo da outra sessão.

## Recomendação

**ABSORVER UlFor como tronco oficial; loop atua em paralelo como
camada de sanity-check formal.**

Justificativa baseada em evidência:

- Esperar a sessão ativa terminar não tem critério de parada — UlFor
  é um plano de 6 fases, hoje está na FASE 3, ainda faltam FASE 4
  (champion + serialização + drift monitor) e FASE 5 (produção). A
  sessão ativa vai durar dias.
- Pivotar (loop atacar outro alvo) desperdiça o plano consolidado.
  O Breno decidiu em 22/05 que UlFor substitui PLANO_UNIFICACAO; o
  loop replicar o mesmo trabalho violaria "uma fonte de verdade por
  coisa" (Princípio 1 do PLANO_FINAL).
- Retroagir (loop varrer Fases 0-1-2 já fechadas) é útil só como
  catalogação. O valor real do loop hoje é fazer o que UlFor ainda
  não faz: `sanity_checks/` formais (leak_detection, holdout_temporal_strict,
  permutation_importance, baseline_compare, distribution_shift) sobre
  os bakeoffs v1/v2/v3 que já existem em `experiments/bakeoff_curtailment_multisub/`.

**Próximos passos sugeridos para iter 0002+:**

1. Iter 0002 = "baseline_persistencia D+1 curtailment NE/SE/S/N",
   pegando o output do bakeoff v3 (UlFor) como BASELINE da comparação,
   rodando `sanity_checks/baseline_compare.py` e `holdout_temporal_strict.py`.
2. Iter 0003 = `leak_detection` sobre as 15 feat_ tables (especialmente
   feat_previsao_renovavel e feat_pdp_renovavel — adicionadas hoje em
   sequência rápida, alto risco de vazamento ex-post inadvertido).
3. Iter 0004 = permutation_importance sobre o ranqueamento de features
   do bakeoff v3.
4. Coordenação: criar `coordination/loop_session_state.md` (no worktree
   loop, NÃO no ulfor) e instruir a próxima iter a checar mtime do
   `coordination/forecast_session_state.md` antes de assumir estado da
   sessão ativa.
5. Não tocar `models/`, `experiments/forecast_curtailment/`, `services/forecasting/`
   no worktree loop — são propriedade da sessão ativa por enquanto.
