# Loop status — 2026-05-24T17:17:24Z

**Saude**: green (loop saudavel)

**Loop continuo**: yes (pid=554874, .STOP=no)

**Iter atual**: 32
**Alvo ativo**: h20_low_n_test_warning

**Budget**: 32 iters / 1.8 horas consumidas

**Open requests ao UlFor**: 0

**Ultimo handoff**: iter_0032_h20_low_n_test_warning.md
  decision: PROMOVE

**Quality gate proximo**: pass — all_rules_passed

**Bridge ultimo sync**: af6b14a bridge: sync ulfor iter 72 (10 seconds ago)

**UlFor HEAD**: 80620230 fix(bakeoff_d1): mover wrap stdout para __main__ (destrava smoke offline)

---

Para parar o loop:
  echo stop > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/.STOP

Para forcar uma iter especifica:
  echo "<prompt content>" > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_force_next_iter.txt

Para retomar manual:
  bash /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/run.sh > /c/Projetos/brazilgrid-loop/loops/forecast-mega-loop/watchdog/_loop_stdout.log 2>&1 &
