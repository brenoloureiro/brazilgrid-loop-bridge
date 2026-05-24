# Loop status — 2026-05-24T04:15:38Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=54759, .STOP=no)

**Iter atual**: 14
**Alvo ativo**: NE_d1_quantile_forecast

**Budget**: 14 iters / 1.0 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0014_h11_quantile_regression_ne.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 866af26 bridge: sync ulfor iter 3 (2 minutes ago)

**UlFor HEAD**: 5dacb5a2 ulfor checkpoint: 07:15Z — Fase 4 step 1 fechada (Dagster+drift+Telegram)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
