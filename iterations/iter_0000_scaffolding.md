---
alvo: scaffolding
layer: meta
iter_num: 0
data_utc: 2026-05-23T23:52:32Z
baseline_tipo: nenhum
baseline_metric: null
hypothesis: a estrutura do loop é exercitável ponta-a-ponta antes de rodar qualquer modelo
result_metric: null
decision: PROMOVE
sanity_checks_passed:
  permutation_importance: skipped
  holdout_temporal_strict: skipped
  leak_detection: skipped
  baseline_compare: skipped
  distribution_shift: skipped
budget_consumido_iter: 0.0
custo_estimado_usd: 0.0
---

# Iter 0 — scaffolding

## O que foi criado

Worktree isolado em `C:/Projetos/brazilgrid-loop`, branch
`feat/forecast-mega-loop-scaffold` baseada em `origin/master` (HEAD
`1c55af9f`). A árvore do scaffold dentro do worktree:

```
loops/forecast-mega-loop/
├── config.yaml                          # 4 layers DAG, métricas, budget 20 iter/6h/$50
├── state.json                           # iter_atual=0, alvo_ativo=null
├── leaderboard.md                       # tabela vazia c/ headers
├── README.md                            # como rodar, parar, política
├── iterations/
│   ├── iter_0000_scaffolding_smoke.md   # marker do smoke (Phase 3)
│   └── iter_0000_scaffolding.md         # este handoff (Phase 4)
├── sanity_checks/                       # 5 esqueletos
│   ├── baseline_compare.py
│   ├── distribution_shift.py
│   ├── holdout_temporal_strict.py
│   ├── leak_detection.py
│   └── permutation_importance.py
├── templates/
│   ├── handoff.md                       # frontmatter YAML + corpo
│   ├── iter_prompt.md                   # prompt p/ claude --print
│   └── leaderboard_row.md
└── watchdog/
    ├── heartbeat.sh                     # push p/ /tmp/bridge_simulation
    ├── kill_check.sh                    # .STOP detector
    └── run.sh                           # loop principal (smoke + skeleton real)
```

## Resultado do smoke test (Phase 3)

```
[watchdog 2026-05-23T23:52:31Z] modo smoke: roda 1 iter dummy e sai
[watchdog 2026-05-23T23:52:31Z] smoke file escrito: iter_0000_scaffolding_smoke.md
heartbeat ok -> /tmp/bridge_simulation/iter_0000_scaffolding_smoke.md
[watchdog 2026-05-23T23:52:32Z] smoke ok
exit=0
```

Kill switch validado: `.STOP` aciona abort em **71 ms** (exit 5),
muito abaixo do orçamento de 60 s.

## Pendências conhecidas

- **Bridge URL real**: `config.bridge.url_real` está `null`. Hoje
  o heartbeat só toca `/tmp/bridge_simulation` (git init local sem
  remote). Quando a infra do bridge externo (outra máquina/repo/HTTP
  endpoint) estiver definida, preencher `url_real` e estender
  `watchdog/heartbeat.sh` com o push correspondente.
- **Hooks com a outra sessão de Claude Code**: outra sessão está ativa
  agora no monorepo principal trabalhando em forecast. Este loop **não
  pode iniciar iter 1** enquanto ela estiver rodando — risco de escrita
  concorrente em `.joblib`/ClickHouse/tabelas dbt. Convenção textual
  vive em `config.yaml -> coordenacao.arquivos_que_nao_posso_tocar`,
  mas não há guard automático ainda. TODO: hook PreToolUse global que
  bloqueie escrita fora de `loops/` quando este worktree estiver ativo.
- **Primeiro alvo a atacar**: por ordem do DAG, a primeira layer é
  `entradas`, primeiro alvo `nwp_subdiario` (RMSE normalizado vs
  persistência 24h, threshold 5%). Mas depende de qual baseline já
  está disponível em CH e qual feature manifest a iter 1 vai usar —
  decisão fica para o handoff da iter 1.
- **Loop real ainda não implementado**: `watchdog/run.sh` só conhece
  modo `smoke`. O modo real (que despacha `claude --print` por iter,
  parseia handoff, atualiza state.json/leaderboard.md, contabiliza
  budget) é trabalho da próxima sessão de scaffold (iter "infra-1"
  fora do DAG, antes da iter 1 real).
- **3 arquivos WIP de analytics no worktree principal**: ficaram
  intactos em `C:/Projetos/brazilgrid` na branch
  `feat/relatorio-independente-v2` (compute_events_alerts/analytics.conf/
  analytics.py). Não são deste loop e seguem sob responsabilidade da
  sessão que os criou.

## Próximos passos

1. **Aguardar a sessão grande de forecast terminar** antes de iniciar
   qualquer trabalho de modelo. Confirmar via `git worktree list` que
   nenhum worktree de forecast (`brazilgrid-bench-forecast`,
   `brazilgrid-bigforecaster`, etc.) tem commit recente em andamento.
2. **Iter infra-1**: implementar modo `real` do watchdog (parser de
   handoff, atualização de state/leaderboard, accounting de budget).
   Esta iter ainda não toca dados.
3. **Iter 1 real**: primeiro alvo, primeiro modelo, primeiro sanity
   check rodado de verdade. Provavelmente `nwp_subdiario` ou outro
   alvo da layer `entradas`, a ser decidido pelo state quando a iter
   infra-1 estabilizar.

## Decisão

**PROMOVE** — scaffold completo, smoke passou, kill switch validado.
Loop pronto para receber iter infra-1 quando a coordenação com a outra
sessão permitir.
