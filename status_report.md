# Loop status — 2026-05-24T17:41:33Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 34
**Alvo ativo**: h22_gbdt_vs_ols_gap

**Budget**: 34 iters / 0.8 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0034_h22_gbdt_vs_ols.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 27360fd bridge: sync ulfor iter 89 (61 seconds ago)

**UlFor HEAD**: 3545d8bb ulfor checkpoint: 17:38Z — fase 4, proximo: Breno desbloqueia (9o forcado, SEM substantive, standby puro tunnels DOWN, sinal/ruido 23.5%)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
