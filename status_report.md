# Loop status — 2026-05-24T03:15:48Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=54759, .STOP=no)

**Iter atual**: 10
**Alvo ativo**: feat_pdp_renovavel_residual

**Budget**: 10 iters / 0.7 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0010_h3_pdp_residual_signal.md
  decision: |

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 6e82dc6 bridge: sync iter 9 (18 minutes ago)

**UlFor HEAD**: c8df4077 feat(forecast): investigar lr_N instabilidade — fold 4 e o blowup, CLEAN derruba stdev 57%

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
