# Loop status — 2026-05-24T16:21:28Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 28
**Alvo ativo**: recon_delta_ulfor_post_83abab3e

**Budget**: 25 iters / 1.4 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0028_recon_delta.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: bec6314 bridge: sync iter 28 (25 seconds ago)

**UlFor HEAD**: 6111cda4 docs(forecast): FINDING_LOCAL_CH_STALE_FEAT_TABLES — CH local Docker tem feat_termico 2024-12-31, bloqueia bakeoff val14d

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
