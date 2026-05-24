# Loop status — 2026-05-24T13:17:55Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=272863, .STOP=no)

**Iter atual**: 22
**Alvo ativo**: ensemble_champion_persist

**Budget**: 22 iters / 1.4 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0022_h24_ensemble_champion_persist.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 98adcf8 bridge: sync ulfor iter 24 (4 minutes ago)

**UlFor HEAD**: 4427a718 ulfor checkpoint: 14:15Z — 3 sprints (H22_MA validado, H14-G, matrix) + decisao pendente Breno

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
