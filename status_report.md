# Loop status — 2026-05-24T17:28:02Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 33
**Alvo ativo**: recon_delta_ulfor_post_2917289c

**Budget**: 32 iters / 1.8 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0033_recon_delta.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 75e6352 bridge: sync iter 33 (21 seconds ago)

**UlFor HEAD**: 7b2f1297 ulfor checkpoint: 17:24Z — fase 4, proximo: Breno desbloqueia (7o forcado, mas com 1 commit substantivo 80620230 fix bakeoff_d1 stdout wrap destrava 5 smoke offline)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
