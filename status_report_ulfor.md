# UlFor headless runner — status — 2026-05-24T13:46:45Z

**Saude**: red (pid registrado mas processo nao esta vivo)

**Runner**: alive=no pid=273001 .STOP=no
**Loop status**: budget_iter_daily
**Ultimo heartbeat**: 2026-05-24T13:46:44Z

## Iter / decisao

- **Iter atual UlFor (runner)**: 32
- **Ultimo plan_step modo**: MODO_1_CONTINUACAO
- **Ultimo plan_step razao**: sem loop_requests abertos, sem trigger de checkpoint, sem PLANO_FINAL_COMPLETE
- **err_streak**: 3
- **iters_sem_commit**: 3

## UlFor master (estado observado)

- **HEAD**: 83abab3e docs(forecast): MATRIX coerencia — promote_champions.py JA tem --ridge-alpha
- **Branch**: master
- **Ultimo commit observado (state)**: 83abab3e
- **Ultimo checkpoint**: ulfor_2026-05-24T14-35.md
  - **Idade**: 3 min
- **Fase PLANO_FINAL**: 4
- **Loop requests pendentes**: 0

## Budget (reset diario 00:00 UTC)

- **Data UTC**: 2026-05-24
- **Iters hoje**: 32 / 30
- **Horas hoje**: 6.7866 / 12
- **Custo hoje**: $113.7667 / $50

---

Para parar o runner:
  echo stop > /c/Projetos/brazilgrid-ulfor-runner/runners/ulfor-headless/.STOP

Para forcar prompt especifico na proxima iter:
  echo "<prompt content>" > /c/Projetos/brazilgrid-ulfor-runner/runners/ulfor-headless/runner/_force_next.txt

Para retomar manual apos pausa:
  bash /c/Projetos/brazilgrid-ulfor-runner/runners/ulfor-headless/runner/run.sh > /c/Projetos/brazilgrid-ulfor-runner/runners/ulfor-headless/_runner_stdout.log 2>&1 &
  echo $! > /c/Projetos/brazilgrid-ulfor-runner/runners/ulfor-headless/_runner.pid
