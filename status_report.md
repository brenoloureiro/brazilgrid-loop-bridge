# Loop status — 2026-05-24T03:55:18Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=54759, .STOP=no)

**Iter atual**: 13
**Alvo ativo**: ensemble_v2_persist

**Budget**: 13 iters / 1.1 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0013_h10_ensemble_v2_persist.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: fbdb123 bridge: sync ulfor iter 2 (13 minutes ago)

**UlFor HEAD**: 26617ba5 ulfor checkpoint: 06:15Z — Fase 3 fechada (H10 testada), Fase 4 iniciada (validate_d1 foundation)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
