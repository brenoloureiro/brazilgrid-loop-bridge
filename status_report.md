# Loop status — 2026-05-24T15:51:00Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 26
**Alvo ativo**: leaderboard_canonical_suite_h19_materialization

**Budget**: 25 iters / 1.4 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0026_recon_delta.md
  decision: 

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: 4e9e185 bridge: sync ulfor iter 33 (80 seconds ago)

**UlFor HEAD**: d9aefdd1 ulfor checkpoint: 15:46Z — fase 4, proximo: Breno decide promote v3 (val14d real revisado SE para h22_MA)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
