# Loop status — 2026-05-24T13:42:51Z

**Saude**: yellow (STOP file presente)

**Loop continuo**: no (pid=272863, .STOP=yes)

**Iter atual**: 25
**Alvo ativo**: leaderboard_canonical_suite_h19_materialization

**Budget**: 25 iters / 1.4 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0025_h19_leaderboard_canonical_suite.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 76e874b bridge: sync ulfor iter 28 (8 seconds ago)

**UlFor HEAD**: db3d2cb9 ulfor checkpoint: 15:00Z — H22 stricter REFUTADO + alpha sweep (paralelo), 0 commits proprios convergencia plena

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
