# Loop status — 2026-05-24T13:44:24Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=545663, .STOP=no)

**Iter atual**: 25
**Alvo ativo**: leaderboard_canonical_suite_h19_materialization

**Budget**: 25 iters / 1.4 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0025_h19_leaderboard_canonical_suite.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 97c4513 bridge: sync ulfor iter 29 (42 seconds ago)

**UlFor HEAD**: 83abab3e docs(forecast): MATRIX coerencia — promote_champions.py JA tem --ridge-alpha

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
