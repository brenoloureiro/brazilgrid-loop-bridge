# Loop status — 2026-05-24T13:09:03Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=272863, .STOP=no)

**Iter atual**: 21
**Alvo ativo**: recon_delta_ulfor_post_daf80a6a

**Budget**: 21 iters / 0.4 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0021_recon_delta.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 97d3e7d bridge: sync ulfor iter 21 (22 seconds ago)

**UlFor HEAD**: de190f9d ulfor checkpoint: 14:05Z — frente bias correction fechada (H14 a H14-G)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
