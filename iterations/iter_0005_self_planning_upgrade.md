---
alvo: loop_self_planning_infrastructure
layer: meta
iter_num: 0005
type: infrastructure_upgrade
data_utc: 2026-05-24T04:30:00Z
baseline_tipo: n/a (iter de infra)
hypothesis: |
  Bottleneck atual: humano (Breno) precisa solicitar prompt de cada iter.
  Upgrade: planner.py le estado + handoffs + backlog -> gera proprio
  prompt. Quality gate previne drift. UlFor continua paralelo via canal.
result_metric: |
  Self-planning loop implementado e validado em dry-run (3 iters
  simuladas sem chamar claude --print). Planner seleciona H9 (P1) como
  proxima iter; respeita prioridades, deps, ulfor_session_sync, quality
  gate. Esta e a ultima iter manual; loop continuo aguarda autorizacao
  do Breno.
decision: NAO INICIA loop continuo. Aguarda revisao do Breno.
sanity_checks_passed:
  planner_smoke_dry_run: true
  quality_gate_pass: true
  status_sh_renders: true
  heartbeat_with_bridge_sync: true
  trap_EXIT_garante_heartbeat: true
budget_consumido_iter: 1.5
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/hypotheses_queue.md (15 hipoteses populadas)
  - loops/forecast-mega-loop/watchdog/planner.py
  - loops/forecast-mega-loop/watchdog/quality_gate.py
  - loops/forecast-mega-loop/watchdog/status.sh
  - loops/forecast-mega-loop/watchdog/run.sh (reescrito: continuous loop + --dry-run)
  - loops/forecast-mega-loop/watchdog/heartbeat.sh (sync status_report.md + queue ao bridge)
  - loops/forecast-mega-loop/config.yaml (novo: planner, quality_gate, ulfor_sync, run sections)
  - loops/forecast-mega-loop/README.md (operacional completo)
---

# Iter 0005 — Self-planning upgrade

## Upgrade implementado

A partir desta iter, o loop opera em modo **self-planning autonomo**:
o `planner.py` le estado + queue + handoffs + ulfor sync e gera o
prompt da proxima iter sem humano no meio.

Esta foi a ULTIMA ITER MANUAL. O loop continuo nao inicia
automaticamente — aguarda autorizacao do Breno apos revisar este
handoff.

## Componentes novos

### `hypotheses_queue.md` (Phase A)

Backlog auditavel YAML-list com **15 hipoteses** (H1-H15):
- 4 done (H1 UlFor H1 falsificada, H2 off-by-one refutado, H4 B6
  implementado, H6 SE colapso enviado ao UlFor — mas req-0003 agora
  DONE, ver "UlFor delta" abaixo).
- 4 blocked em req-NNNN externos (H5 termico, H12 carga refresh, H14
  v3.3 audit) ou em H9 (H10, H11).
- 7 queued e selecionaveis: H3, H7, H8, H9, H10, H11, H13, H15.

Topo da fila atual (priority + ordem):
1. **H9** (P1) — substituir NMAE por MAE/R²/F1 (Principio 6 PLANO_FINAL)
2. H7 (P2) — XGBoost vs LGBM systematic comparison
3. H3 (P2) — PDP residual vs gen
4. H11 (P2) — quantile regression bandas P10/P50/P90
5. H15 (P2) — S como classificador binario rare-event
6. H10 (P2, dep H9)
7. H8 (P3)
8. H13 (P3)

Editavel a mao depois pelo Breno.

### `planner.py` (Phase B)

3 modos de output:
- **consolidation** — se quality_gate dispara
- **recon_delta** — se UlFor commitou >= 5 commits desde ultima iter
- **hypothesis_test** — selecao normal do top da queue

Logica de selecao:
1. Quality gate primeiro
2. UlFor sync delta segundo (threshold default 5 commits)
3. Top da queue com deps satisfeitas (req-NNNN done E H done)

Override manual: arquivo `_force_next_iter.txt` consome direto sem planner.

Output em `_next_iter_prompt.txt` + JSON com iter_num/type/hypothesis_id
em stdout.

### `quality_gate.py` (Phase C)

4 regras de forca:
- R1 sanity fail recente
- R2 streak REFUTADO 2+ iters consecutivas
- R3 regressao em `state.best_ml.warn`
- R4 layer drift (3 iters em 3+ layers sem ancora meta)

Saida JSON `{status, reason, consolidation_focus?}`. Status "pass"
permite hypothesis_test. Status "force_consolidation" dispara
prompt de revisao em vez de nova hipotese.

### `status.sh` (Phase E)

Renderiza 1 paragrafo curto com: saude (green/yellow/red), pid alive,
iter atual, budget consumido, requests abertos, ultimo handoff +
decision, quality gate proximo, bridge ultimo sync, UlFor HEAD.

Heartbeat copia o output para `status_report.md` na bridge:
https://github.com/brenoloureiro/brazilgrid-loop-bridge/blob/master/status_report.md

### `run.sh` continuous (Phase D)

Reescrito como while-loop com:
- Budget caps: `BUDGET_MAX_ITER` (20), `BUDGET_MAX_HOURS` (6),
  `BUDGET_MAX_USD` (50) — env override possivel
- Kill switch via `.STOP` file
- `--dry-run` flag para teste sem chamar `claude --print`
- Trap EXIT garante heartbeat final (ja preservado do iter_0003)
- Heartbeat per-iter dentro do loop
- Validacao de handoff escrito (best-effort)
- Sleep `SLEEP_BETWEEN_ITERS` (30s default)

### `heartbeat.sh` (Phase E support)

Sync para bridge agora inclui: leaderboard.md + state.json +
hypotheses_queue.md + status_report.md.

### `config.yaml` (Phase F)

Novas secoes:
- `planner.*` — context window, queue path, force file
- `quality_gate.*` — thresholds
- `ulfor_sync.delta_commits_force_recon` = 5
- `run.sleep_between_iters_sec` = 30
- `coordenacao.exception_paths` = caminho do loop_requests.md no UlFor

## Smoke test results (dry-run de 3 iters)

```
[watchdog] MODE: DRY-RUN (no claude --print, no real execution)
[watchdog] ENTRANDO em loop continuo
==> iter slot 1: planejando -> H9 selected (P1)
==> iter slot 2: planejando -> H9 selected (queue nao mudou em dry-run)
==> iter slot 3: planejando -> H9 selected
[watchdog] BUDGET HIT: max_iter=3 alcancado
[watchdog] LOOP ENCERRADO: status=budget_iter iters=3 elapsed=0.01h cost=0.0
```

Validou:
- planner_exit=0 em todas 3
- prompt gerado em `_next_iter_prompt.txt` corretamente
- state.json bumped iter_atual em cada dry-run (revertido para iter=4 depois)
- heartbeat per-iter pushou ao bridge em cada iter
- bridge mostra noise iter=5,6,7 → revertido para iter=4 com commit `e13360d`

Comportamento esperado em loop real: H9 seria marcada como DONE na
iter 5, planner selecionaria H7 (proximo P2) na iter 6, etc.

## UlFor delta durante esta iter

Sessao UlFor processou **TODOS os 3 requests** entre iter_0004 e iter_0005
(via autopilot patch aplicado no AUTOPILOT_PROMPT.md):

| commit | mensagem |
|---|---|
| `be2c9186` | coord: AUTOPILOT_PROMPT — passo 4.5 leitura de loop_requests.md |
| `b7acfdcd` | feat(forecast): UlFor PDP gap Abr/26 fix — v3.3 sub-level ganhos |
| (loop_requests.md status updates) | req-0001/0002/0003 marcados DONE |

Detalhes:
- Breno aplicou o patch sugerido em iter_0003 `autopilot_patch.md` →
  autopilot agora le `coordination/loop_requests.md` no inicio de cada
  retomada apos `/clear`
- UlFor processou req-0002 (P0): aplicou fix PDP gap Abr/26 (commit
  b7acfdcd, "v3.3 sub-level ganhos"). **Loop iter_0006 deve absorver
  esse fix via recon_delta** (ver "Proxima iter" abaixo)
- req-0001 e req-0003 marcados DONE — `ulfor_response` ainda nao
  preenchido (autopilot pode ter feito processamento parcial). Loop
  iter_0006 deve verificar e extrair respostas

## Como Breno opera daqui em diante

### Iniciar loop continuo

```bash
bash loops/forecast-mega-loop/watchdog/run.sh \
  > loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
echo $! > loops/forecast-mega-loop/watchdog/_loop.pid
```

Loop roda ate budget esgotar OU .STOP detectado OU queue vazia.

### Ver status (do celular)

https://github.com/brenoloureiro/brazilgrid-loop-bridge/blob/master/status_report.md

Atualizado a cada iter via heartbeat.

### Parar

```bash
echo stop > loops/forecast-mega-loop/.STOP    # graceful
kill $(cat loops/forecast-mega-loop/watchdog/_loop.pid)   # forcado
```

### Ajustar prioridades

Editar `hypotheses_queue.md`. Loop le no proximo planner.

### Forcar iter especifica

```bash
cat > loops/forecast-mega-loop/watchdog/_force_next_iter.txt <<EOF
TAREFA: <conteudo>
EOF
```

Arquivo consumido (deletado) ao usar.

## Riscos remanescentes e mitigations

1. **Queue exhaustion**: 7 hipoteses queued + 4 blocked. Se loop rodar
   todas as queued sem destravar blocked, planner retorna exit 1 e
   loop para gracefully. Breno precisa rebastecer queue manualmente
   ou destravar reqs.
   - Mitigation: status_report.md mostra `open_requests` + queue
     remaining count; Breno monitora.

2. **claude --print custo runaway**: hard caps em USD/hours/iter
   protegem. Mas estimativa de custo extraida do JSON pode falhar
   (best-effort).
   - Mitigation: `BUDGET_MAX_USD=50` cap conservador. Breno revisa
     `_last_run.json` apos primeiro real iter para calibrar.

3. **Path Windows / Git Bash**: scripts usam `cygpath -m` fallback
   para converter `/c/...` → `C:/...` para Python. Testado e
   funcionando, mas frageil.
   - Mitigation: `_winpath()` helper centralizado em status.sh +
     `LOOP_ROOT_WIN` em run.sh.

4. **Heartbeat per-iter push noise**: cada iter gera 1 commit no bridge.
   3 iters → 3 commits. Em loop de 20 iters: 20 commits ao bridge em
   poucas horas.
   - Mitigation: heartbeat e idempotente (so commita se ha diff),
     entao iters sem mudanca real nao commitam. Consolidations e
     recon_deltas tipicamente mudam state.json/leaderboard.md.

5. **UlFor concorrencia**: UlFor commitando em `coordination/
   loop_requests.md` no momento exato em que loop iter tambem esta
   editando = race condition rara. UlFor processa em retomada
   pos-`/clear`, loop processa no fim de cada iter — janelas geralmente
   nao se sobrepoem.
   - Mitigation: se ocorrer conflito, loop iter falha em `git push`
     do canal, e retoma na proxima iter com pull rebase.

6. **Quality gate falsos positivos**: regra R4 (drift) pode disparar
   em loop legitimo que pula entre layers de proposito.
   - Mitigation: facil ajustar `quality_gate.drift_max_layers` no
     config.yaml. Default 3 e conservador.

7. **Self-planning recursive nullification**: se planner.py tem bug e
   sempre seleciona mesma hipotese (como em dry-run), loop nao
   progride.
   - Mitigation: hypothesis_test prompt instrui CC a marcar H como
     `done` no queue. Se CC nao fizer, planner re-seleciona — sintoma
     visivel em status_report.md (mesmo H em "ultima iter" repetidamente).

## Primeira iter real planejada (sem executar): qual hipotese, por que essa

**NAO H9** (proxima do queue normalmente). Em vez disso, **iter_0006 deve
ser RECON_DELTA** porque UlFor commitou 2 commits relevantes
(b7acfdcd PDP fix, be2c9186 autopilot patch) + processou 3 requests.

Planner deve detectar via `_ulfor_sync_delta()` — current HEAD
`be2c9186` vs last known `9c1484e2` = 2 commits, ABAIXO do threshold
5. Logo nao dispara recon_delta automatico.

**Acao recomendada para Breno antes de iniciar loop continuo**:

Opcao A — preferida — usar override manual para forcar iter_0006
como recon_delta:

```bash
cat > loops/forecast-mega-loop/watchdog/_force_next_iter.txt <<'EOF'
TAREFA: Iter 0006 do forecast-mega-loop (modo: RECON_DELTA manual).

UlFor processou TODOS os 3 requests do loop entre iter_0004 e _0005:
- be2c9186 aplicou patch AUTOPILOT_PROMPT (passo 4.5 lendo loop_requests)
- b7acfdcd PDP gap Abr/26 fix — v3.3 sub-level ganhos
- coordination/loop_requests.md status req-0001/0002/0003 = DONE

PHASE A: inspecionar commits UlFor read-only:
  git -C C:/Projetos/brazilgrid-ulfor show b7acfdcd
  git -C C:/Projetos/brazilgrid-ulfor show be2c9186
  ler coordination/loop_requests.md atualizado (3 reqs DONE)

PHASE B: extrair ulfor_response e atualizar:
  - state.json: open_requests => mover DONE para closed_requests
  - hypotheses_queue.md: H6 ja done (req-0003 fechado), atualizar
    iter_handled e completed_at
  - leaderboard.md: se UlFor reportou metricas v3.3 sub-level, atualizar
  - state.ulfor_session_sync: novo fim_head + lista de commits

PHASE C: handoff iter_0006_recon_delta_post_reqs.md
  com resumo de mudancas, nova prioridade da queue (H9 ainda topo),
  recomendacao sobre re-rodar B6 sobre v3.3 outputs (= H14).

Commit + heartbeat. Loop continua normalmente apos.
EOF
```

Opcao B — alternativa — atualizar `state.json.ulfor_session_sync.
iter0005_fim_head` para algo antigo (ex.: `1fb2bb50`) para forcar
delta >= 5, o que dispara recon_delta automatico do planner.

Apos iter_0006, planner volta a selecao normal. **H9 sera proxima**
(P1, NMAE -> MAE/R²/F1).

## Estado da queue ao fim de iter_0005

Atualizado para refletir UlFor:

| id | status novo | iter_handled |
|---|---|---|
| H1 | done | ulfor_external |
| H2 | done | 0003 |
| H4 | done | 0004 |
| H6 | done | (req-0003 fechado, marcar 0004→fechamento via 0006) |

Demais 11 hipoteses (H3, H5, H7-H15) permanecem como na queue inicial.

## Commit + heartbeat

Iter 0005 commita: queue, planner, gate, status, run.sh, heartbeat,
config, README, este handoff. Heartbeat sync ao bridge. NAO inicia
loop continuo.

Breno autoriza inicio manual depois de revisar.
