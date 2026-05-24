# Loop status — 2026-05-24T16:59:20Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 31
**Alvo ativo**: recon_delta_ulfor_post_27152e16

**Budget**: 30 iters / 1.2 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0031_recon_delta.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: fe37919 bridge: sync ulfor iter 62 (2 minutes ago)

**UlFor HEAD**: c1cf9779 ulfor checkpoint: 16:54Z — fase 4, proximo: Breno desbloqueia (5o forcado consecutivo, tunnels CH+MLflow exit=28, 25min standby legitimo)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
