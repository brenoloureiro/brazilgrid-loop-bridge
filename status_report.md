# Loop status — 2026-05-24T02:30:04Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=54759, .STOP=no)

**Iter atual**: 7
**Alvo ativo**: fase_4_champions_ridge_lr_post_cv

**Budget**: 7 iters / 1.0 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0007_h17_fase4_promote_superseded.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: b0d5fb6 bridge: sync iter 7 (24 seconds ago)

**UlFor HEAD**: 83bc79c2 feat(forecast): promote champions D+1 — ridge_NE/lr_SE/lr_S Champion, ridge_N Staging

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
