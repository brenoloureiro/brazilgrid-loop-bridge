# Loop status — 2026-05-24T20:27:53Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 46
**Alvo ativo**: gbdt_vs_ridge_alpha10_ne_full

**Budget**: 41 iters / 0.25 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0046_h41_gbdt_vs_ridge_ne_full.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 3c829e4 bridge: sync iter 45 (17 minutes ago)

**UlFor HEAD**: 4f4e21f4 ulfor checkpoint: 17:44Z — fase 4, proximo: Breno desbloqueia (10o forcado, SEM substantive, standby puro tunnels DOWN, sinal/ruido 22.2%)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
