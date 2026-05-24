# Loop status — 2026-05-24T12:25:08Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=272863, .STOP=no)

**Iter atual**: 17
**Alvo ativo**: recon_delta_ulfor_post_5dacb5a2

**Budget**: 17 iters / 0.5 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0017_recon_delta.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: b2d29dc bridge: sync ulfor iter 10 (9 minutes ago)

**UlFor HEAD**: eb9fca05 exp(forecast): H14-B sweep da janela bias correction (28/60/90d)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
