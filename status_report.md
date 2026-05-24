# Loop status — 2026-05-24T17:54:59Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 35
**Alvo ativo**: h25_stacker_ridge

**Budget**: 34 iters / 0.8 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0035_h25_stacker_ridge.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: ee3125a bridge: sync ulfor iter 93 (7 minutes ago)

**UlFor HEAD**: 4f4e21f4 ulfor checkpoint: 17:44Z — fase 4, proximo: Breno desbloqueia (10o forcado, SEM substantive, standby puro tunnels DOWN, sinal/ruido 22.2%)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
