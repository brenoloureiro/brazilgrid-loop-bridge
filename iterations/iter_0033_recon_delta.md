---
alvo: recon_delta_ulfor_post_2917289c
layer: meta
iter_num: 0033
type: recon_delta
data_utc: 2026-05-25T04:30:00Z
ulfor_head_inicio: 2917289c
ulfor_head_fim: 80620230
commits_absorvidos: 5
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.3
---

# Iter 0033 — RECON_DELTA UlFor (2917289c → 80620230)

## Objetivo

Absorver 5 commits novos da sessao UlFor entre `2917289c` (HEAD ao fim do
iter_0031 — checkpoint marker 16:48Z em standby legitimo apos Breno
oficializar promote v3 mas tunnels CH+MLflow exit=28) e `80620230` (HEAD
17:17Z apos sprint envelope-safe destravar smoke test offline). Janela
~24 min reais UlFor (13:54-14:17 BRT — commit timestamps). Conteudo:
**2 substantivos** (`3971124d` smoke test loader pos-promote v3 +
`80620230` fix wrap stdout import-time bug) + **3 checkpoints
PARAR-E-PERGUNTAR** identicos do runner heartbeat em standby.

UlFor permanece bloqueado em infra (sem rota EC2). Sessao usou janela
para **preparar o terreno**: smoke test que dispara assim que tunnels
subirem + fix Windows-only side-effect em `bakeoff_d1.py` que herdado
por 19 importers (incluindo `loader.py` prod) destravava capture do
pytest. Producao 100% inalterada (loader.py defaults + MLflow Registry
intocados). Nenhuma decisao Breno foi tocada.

## PHASE A — Commits inspecionados

Em ordem cronologica:

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `b72d0e5c` | checkpoint 16:53Z (5o forcado) | runner force iters-sem-commit threshold; tunnels CH+MLflow exit=28; ~26min standby | state (marker) |
| 2 | `c1cf9779` | checkpoint 16:54Z (5o continuado) | refinamento ~1min apos 16:53Z; 25min standby legitimo | state (marker) |
| 3 | `3971124d` | test(forecast) smoke loader pos-promote v3 | **SUBSTANTIVO** — 218 LoC novo `tests/services/test_forecast_loader_smoke.py`; 5 offline passed + 8 online skipped (gated CH/MLflow); destrava validacao imediata pos-tunnels | leaderboard nota + state delta_resumo |
| 4 | `279f7260` | checkpoint 17:08Z (6o forcado) | confirma 1 commit substantivo na janela (`3971124d`); ainda standby | state (marker) |
| 5 | `80620230` | fix(bakeoff_d1) wrap stdout para `__main__` | **SUBSTANTIVO** — bugfix Windows-only `sys.stdout = TextIOWrapper(...)` em import-time fechava tempfile pytest capture; quebrava 13 smoke tests; impacto secundario em 19 importers (incluindo `loader.py` em prod); fix isola wrap ao CLI | leaderboard nota + state delta_resumo |

**Composicao**: 2 substantivos (40%) + 3 checkpoints PARAR-E-PERGUNTAR (60%).
Janela mais densa que iter_0031 (25% substantivo) — UlFor saiu do
standby zero-trabalho para sprints envelope-safe (sem precisar de CH ou
MLflow), mesmo permanecendo bloqueado pelas tunnels.

### Commit 3 — `3971124d` (smoke test loader v3)

Novo arquivo `tests/services/test_forecast_loader_smoke.py` (218 LoC).
2 camadas:

**Offline (5 testes, sempre rodam, sem CH/MLflow)**:

- `test_champions_table_keys_match_subs` — `CHAMPIONS` no loader cobre
  exatamente os 4 subs do bakeoff (`set(CHAMPIONS.keys()) == set(SUBS)`)
- `test_champions_loader_matches_promote_registered_names` — `registered_name`
  + `alias` do loader = `promote_champions`
- `test_bias_correction_defaults_window_cover_all_subs` —
  `BIAS_CORRECTION_DEFAULTS` + `BIAS_CORRECTION_WINDOW_BY_SUB` cobrem todos
  os subs (NE/SE/S/N)
- `test_load_champion_unknown_sub_raises` — `load_champion('XX')` ->
  `ValueError` sem tocar MLflow
- `test_inference_git_sha_resolves_and_caches` — `_inference_git_sha()`
  retorna SHA + cacheia chamada seguinte

**Online (8 testes, skip graceful se CH/MLflow inacessivel)**:

- 4× `test_load_champion_sub_X` (NE/SE/S/N) — estrutura + `feature_set`
  valido
- 4× `test_predict_d1_sub_X` (NE/SE/S/N) — predicoes >= 0 finitas,
  `target_dia > dia_D`, `bias_correction` self-consistent (`raw -
  bias_mw = corrected` quando `applied=True`), `n_features > 0`

**Skip strategy**: `pytest.importorskip` para `mlflow` + socket check
para CH (TCP connect a `localhost:8123` com timeout 2s). Mensagem clara
"`MLflow inacessivel — skip`" / "`ClickHouse inacessivel — skip`".

**Resultado local agora**: 5 passed, 8 skipped (gated). Pos-tunnels:
basta `uv run pytest tests/services/test_forecast_loader_smoke.py -v`
para certificar v3 NE/SE/N do MLflow Registry bate end-to-end com o
loader.

**Implicacao quality_gate**: quando Breno desbloquear EC2 setup ou raw
sync (gate atual do `[[ch_local_feat_termico_stale]]`), o smoke pronto
e' o **primeiro check** apos promote — antes mesmo de val_recent
formal. UlFor cobriu o gap que loop teria que cobrir via req formal.

### Commit 5 — `80620230` (fix wrap stdout)

Diff minimo (3 linhas removidas, 2 adicionadas) em
`experiments/bakeoff_curtailment_multisub/bakeoff_d1.py`:

```diff
-if os.name == "nt":
-    sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding="utf-8", errors="replace")
-
 CH_URL = os.environ.get("CH_URL", "http://localhost:8123/")
 ...
 if __name__ == "__main__":
+    if os.name == "nt":
+        sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding="utf-8", errors="replace")
     main()
```

**Causa**: o wrap em import-time fechava o `tempfile` que o `pytest
capture` usa, quebrando **todos** os 13 smoke tests do loader (5
offline + 8 online). Side-effect Windows-only mas afetava qualquer
desktop dev local.

**Impacto secundario (mais importante)**: 19 importers de
`bakeoff_d1.py` herdavam o wrap em import-time. Lista nao-exaustiva:

- `services/analytics_api/forecast/loader.py` — **codigo de PROD do
  FastAPI** importa `bakeoff_d1.SUBS` + `FEATURE_DROPS_*` para
  construir features (DRY com bakeoff de pesquisa). Toda chamada
  `/api/forecast/d1/*` no Windows dev tocava `sys.stdout`.
- 11 scripts do loop em `loops/forecast-mega-loop/scripts/` (h7, h8,
  h10, h11, h21, h24, etc.) que importam features ou modelo de
  bakeoff.
- 7 scripts do UlFor em `experiments/bakeoff_curtailment_multisub/`
  (alpha sweeps, promote_champions, validate_d1).

**Fix**: wrap so executa quando `bakeoff_d1.py` e' rodado como CLI
(`uv run python -m experiments...bakeoff_d1`). Importadores nao tocam
mais `sys.stdout`.

**Verificacao UlFor**: 5 passed / 8 skipped (gated por MLflow+CH) —
esperado pos-fix. Sem fix, 0 passed / 13 errored.

**Implicacao loop**: scripts do loop em `scripts/` que importam
`bakeoff_d1` (h21_pdp_residual_cv.py, h8_intercambio_cv_pi.py, etc.)
nao precisam de mudanca propria — bug fica resolvido upstream. Loop
nao tem teste que dependia disso, mas testes do UlFor agora rodam
direito no desktop dev (era um caveat mute ate agora).

### Commits 1, 2, 4 — checkpoints heartbeat (markers)

3 checkpoints PARAR-E-PERGUNTAR identicos:
- `b72d0e5c` 16:53Z (5o forcado consecutivo)
- `c1cf9779` 16:54Z (5o continuado, refinamento)
- `279f7260` 17:08Z (6o forcado, com 1 commit substantivo `3971124d`
  na janela)

Padrao **"runner forca checkpoint a cada ~9-10 min em standby
legitimo"** documentado em iter_0028 e iter_0031 mantem-se. UlFor
permanece em PARAR-E-PERGUNTAR aguardando Breno desbloquear infra (EC2
setup ou skip-local-revalidate). 3 sao standby zero-trabalho; entre o
4o (16:54Z) e o 6o (17:08Z) UlFor capturou ~14 min para sprint
envelope-safe (smoke test) e nas ~9 min finais para fix stdout.

## PHASE B — Atualizacoes

### `state.json`

Bloco novo `ulfor_session_sync.iter0033_*` adicionado:

- `iter0033_inicio_head`: `2917289c`
- `iter0033_fim_head`: `80620230`
- `novos_commits_durante_iter0033`: 5 entradas (3 checkpoints + 2
  substantivos)
- `delta_resumo_iter0033`:
  - `champions_status_change`: **INALTERADO** em producao
    (loader.py + MLflow Registry intocados). Decisao Breno promote v3
    (iter_0031) NAO executada — mesmo bloqueio infra.
  - `infra_smoke_test_pronto`: novo `tests/services/test_forecast_loader_smoke.py`
    com 5 offline passed + 8 online skipped (gated CH/MLflow).
    Pos-tunnels: 1 comando para certificar end-to-end.
  - `bug_fix_stdout_wrap`: `bakeoff_d1.py` wrap stdout Windows-only
    movido de import-time para `__main__`. Impacto: destrava 13 smoke
    tests + 19 importers (incluindo `loader.py` prod).
  - `novos_requests`: []
  - `requests_fechados_extras`: []
  - `novas_hipoteses_loop_geradas`: []
- `iter_atual`: 32 -> 33

### `hypotheses_queue.md`

**Sem mudancas.** Nenhuma H do loop fica afetada por commits de
infra-de-teste/bug-fix. Reverificadas:

- H8/H22/H30/H27/H33 nosso (analytics-related): inalteradas — nao
  tocam loader.py, MLflow Registry, ou champions.
- H18 (MLflow read-only via tunnel, P1 blocked-acao-Breno-trivializa):
  inalterada — bug fix nao destrava tunnel CF Access.

### `leaderboard.md`

Header atualizado para mencionar iter_0033 recon. Nenhuma tabela
muda. Note explicita que UlFor preparou smoke test que destrava
quality_gate automatico pos-promote.

## PHASE C — Handoff

### Estado proximo iter

- **iter_atual** = 33
- **HEAD UlFor real** = `80620230` (capturado).
- **Promote v3** = decidido por Breno em iter_0031, NAO executado
  (mesmo bloqueio EC2 setup vs skip-local-revalidate).
- **Quality_gate cobertura** = SOBE com smoke test pos-promote pronto
  (UlFor pre-emptou potencial req-0008 de loop "exigir smoke test
  pos-promote v3").

### Recomendacao planner para iter_0034

- **(A) MONITOR / RECON_DELTA** se HEAD UlFor avancou alem de
  `80620230` — janela densa (2 substantivos em ~24 min) sugere UlFor
  pode ter saido de standby completo para sprints envelope-safe se
  multi-agente reativar.
- **(B) H30** (P3 ~1h, Ridge_alpha=10 + alpha=1 sobre `pdp_residual`
  CV): zero dependencia externa, fecha frente H3-family residual,
  responde alpha sweep simultaneamente. Atratividade reforcada por
  iter_0026/0028 (alpha=1 vence alpha=10 em h22_MA/h22_pf NE+SE).
- **(C) H22 nosso** (P3 ~1h, GBDT vs OLS gap em pdp_residual com
  PI-com-GBDT lesson H23_ulfor/H22_MA): aplicar PI medida com modelo
  final (LGBM) em vez de proxy mais robusto (OLS).
- **(D) H33** (P3 ~0.3h, joint-drop SE em LR): atratividade
  CONTINUA DIMINUIDA (val14d iter_0028 + decisao Breno opt A iter_0031
  ja deram 3 evidencias independentes de model-aware). So vale como
  curiosidade metodologica.

**RECOMENDACAO**: (A) se HEAD UlFor advance; senao (B) — H30 fecha
H3-family e da' resposta limpa ao alpha sweep dentro da mesma janela.

### Lessons learned (potenciais, registradas como observacao)

- **UlFor multi-agente em standby usa janela para infra-de-teste**:
  padrao iter_0028/iter_0031/iter_0033 mostra que mesmo quando
  bloqueado por tunnels (EC2 setup pendente), UlFor pode entregar
  sprints envelope-safe (sem CH/MLflow): testes, fix de bugs,
  documentacao. Loop NAO precisa emitir req formal para esses gaps —
  pre-empcao recorrente.
- **bug Windows-only import-time** (stdout TextIOWrapper) em
  `bakeoff_d1.py` herdado por 19 importers (incluindo PROD): lembrete
  que side-effects em import-time sao **transitivos** mesmo no Python
  com `__name__ == "__main__"` idiom. UlFor refatorou para isolar ao
  CLI mode.

### Convencao nomenclatura

Convencao `Hxx_ulfor` mantida (13a iter consecutiva recon-style). Esta
iter NAO introduz novas `H_ulfor` formais — sprints sao infra-de-teste
+ bugfix, nao hipoteses experimentais.

### Sanity checks

Iter type=recon_delta — sanity checks default NAO aplicaveis (nenhum
modelo treinado, nenhum dataset gerado). PHASE D dispensavel.

### Budget

- Estimado: 0.3h
- Atual: ~0.3h (read-only inspect + escrita handoff)
