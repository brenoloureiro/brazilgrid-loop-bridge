# Loop status — 2026-05-24T12:35:18Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=272863, .STOP=no)

**Iter atual**: 18
**Alvo ativo**: recon_delta_ulfor_post_5c7963d4

**Budget**: 18 iters / 0.4 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0018_recon_delta.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: a7b4d7b bridge: sync iter 18 (18 seconds ago)

**UlFor HEAD**: daf80a6a exp(forecast): H14-F window=60d + threshold sigma_bias (Pareto vs H14-B, NAO produtizar)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
