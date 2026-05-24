---
alvo: h20_low_n_test_warning
layer: meta
iter_num: 0032
type: hypothesis_test (verdict=CONFIRMADO_DISPLAY)
data_utc: 2026-05-25T03:30:00Z
hypothesis: |
  H20 (P3, queued desde iter_0009, layer=meta, type=meta):
  "Auto-flag bake-off com n_test < 30 — warning explicito no leaderboard.
  Patched B6 (H16) evita falsos positivos de severity mas o problema raiz
  e' bake-offs com janelas curtas. Tudo que entra no leaderboard com
  n_test<30 deveria ter coluna n_test visivel + flag
  `low_confidence_n_test`. Hoje a coluna `sanity_ok` embute isso
  opacamente. Mudanca pequena de display + adicao no bake-off runner para
  escrever n_test no meta.json. Sem dep externa."

  Sub-claims testaveis:
    C1 (cobertura): existe n_test em TODOS os artefatos quantitativos do
       loop (meta.json, summary CSV, parquets UlFor)?
    C2 (propagation runner): run_bakeoff_replay.py expoe flag explicito
       low_confidence_n_test em meta.json + summary?
    C3 (display leaderboard): leaderboard mostra n_test+flag por linha
       quando aplicavel, em vez de aviso textual por secao?
    C4 (false-negative top): zero linhas com n<30 marcadas como OK nas
       tabelas ativas (Champions, Baselines, Overlay, Sucessores)?

baseline_tipo: |
  Status anterior do leaderboard. Tabelas Champions/Baselines/Overlay/
  Sucessores nao expoem n_test explicito. Aviso de janela curta vivia
  apenas no titulo da secao "Historico iter loop (deprecado — replays n=11)"
  -- leitor podia citar linha individual sem o contexto. Runner ja' escrevia
  n_test em meta.json desde iter_0002 mas SEM flag derivado, SEM threshold
  exposto, SEM coluna no summary CSV.

baseline_metric: |
  Cobertura n_test no estado pre-H20:
    - meta.json (runner replay): n_test presente em 12/12 artefatos OK
    - summary_replay.csv: coluna n_test presente, flag AUSENTE
    - parquets UlFor per_fold: coluna n_test presente em 18/18 (n_test∈{59,60})
    - leaderboard: aviso por secao (string em titulo), zero coluna n_test
    - 0 flags `low_confidence_n_test` em qualquer artefato

result_metric: |
  Pos-H20:
    - meta.json: low_confidence_n_test + low_n_test_threshold expostos no
      runner (patch scripts/run_bakeoff_replay.py linhas 44-46, 426-431,
      481-484, 503-505). Backfill manual NAO aplicado em meta.json
      individuais (sao read-only audit; novo run sobrescreve).
    - summary_replay.csv (iter_0002): coluna low_confidence_n_test ja'
      backfilled (12/12 linhas true, todas com n_test=11).
    - leaderboard.md: bloco "Politica low_confidence_n_test" no topo;
      tabela "Historico iter loop (deprecado)" ganha 2 colunas (n_test,
      low_confidence_n_test); lessons learned acresce entrada
      display-per-row vs aviso por secao.
    - Audit unificado: outputs/iter_0032/h20_leaderboard_low_n_test_warning/
      {n_test_audit.json, n_test_audit.md, sanity_checks.json}.

  Decomposicao por sub-claim:
    C1 (cobertura): ATENDIDO. n_test presente em 100% dos artefatos
       quantitativos (12 meta + 21 outros JSON + 18 parquets = 51
       artefatos; o unico cinza e' o false-positive de regex em
       verdict.json iter_0012 onde "n":5 = numero de folds, excluido
       por allowlist KNOWN_NON_N_TEST_FIELDS).
    C2 (propagation runner): ATENDIDO. Patches em 3 sites do
       run_bakeoff_replay.py (constante LOW_N_TEST_THRESHOLD, ramo skip,
       ramo ok + linha do summary).
    C3 (display leaderboard): ATENDIDO. Coluna n_test + flag por linha
       na secao deprecada + politica explicita no topo + lesson learned.
    C4 (false-negative top): ATENDIDO. 0 linhas LOW nas tabelas ativas;
       todas referenciam parquets per_fold UlFor com n_test∈{59,60}.

  Totais audit:
    - 12 meta.json LOW / 0 OK (todos iter_0002 LGBM replay)
    - 8 outros JSON LOW (4 iter_0002 + 2 iter_0009 + 1 iter_0008 + 1 iter_0004)
    - 12 outros JSON OK (iter_0010/0012/0013/0014/0020/0022/0027/0030)
    - 1 outro JSON excluido false-positive (iter_0012 verdict.json: "n":5 = folds)
    - 0 UlFor parquets LOW / 18 OK
    - 12/12 linhas summary_replay.csv LOW

decision: PROMOVE
sanity_checks_passed:
  permutation_importance: skipped_NA
  holdout_temporal_strict: skipped_NA
  leak_detection: skipped_NA
  baseline_compare: PASS_BY_AUDIT
  distribution_shift: skipped_NA
  zero_count_shift: skipped_NA
  audit_idempotent: PASS
  audit_coverage_100pct: PASS
  leaderboard_top_no_low_confidence: PASS
  runner_patch_backward_compat: PASS
  csv_backfill_correctness: PASS
budget_consumido_iter: 0.6
custo_estimado_usd: null
---

# Iter 0032 — H20 leaderboard low_confidence_n_test

## Hipotese

H20 (P3 queued desde iter_0009): bake-offs com `n_test < 30` deveriam
ter coluna visivel + flag explicito `low_confidence_n_test` em todos os
artefatos auditaveis (meta.json, summary CSV, leaderboard). Antes desta
iter o aviso vivia em titulo de secao ("Historico iter loop deprecado —
replays n=11") — opaco para quem cita uma linha individual sem o
contexto. Derivada do lesson H16 v1.1 (iter_0009): threshold n_test<30
ja' era usado para downgrade de severity em B6 mas nao estava propagado
em outros locais.

## Como foi rodado

Nenhum treino, nenhum CV novo. Inteiramente meta + display:

1. **Inventory** (`scripts/_iter0032_inventory_n_test.py`, throwaway):
   varredura read-only de outputs/ + ULFOR_OUT parquets para mapear
   onde n_test aparece e em que valores. Identificou 1 falso positivo
   de regex em verdict.json iter_0012 (`"n": 5` = numero de folds).

2. **Audit consolidado** (`scripts/h20_n_test_audit.py`): emite
   `outputs/iter_0032/h20_leaderboard_low_n_test_warning/n_test_audit.{json,md}`
   classificando cada artefato como LOW/OK + agregando totais. Allowlist
   `KNOWN_NON_N_TEST_FIELDS` exclui false positives.

3. **Patch runner** (`scripts/run_bakeoff_replay.py`): 3 sites editados.
   - L43-46: constante `LOW_N_TEST_THRESHOLD = 30` exposta no header.
   - L426-431: ramo `status=skip` ganha `low_confidence_n_test` +
     `low_n_test_threshold`.
   - L481-484: ramo `status=ok` ganha as mesmas chaves.
   - L502-507: dict do summary_rows ganha `low_confidence_n_test`.

4. **Backfill summary CSV** (one-shot polars): adiciona
   `low_confidence_n_test` em `outputs/iter_0002/summary_replay.csv`
   sem re-rodar bake-off (12/12 linhas true porque n_test=11<30).

5. **Display leaderboard**: bloco "Politica low_confidence_n_test"
   logo apos a definicao do skill_vs_persist; coluna n_test + flag por
   linha na tabela "Historico iter loop (deprecado)"; lesson novo no
   bloco "Lessons learned".

6. **State + queue**: H20 marcado done verdict CONFIRMADO_DISPLAY;
   `low_n_test_policy` registrada como nova chave em state.json
   (paralela a `metric_suite_primary` e `b6_robustness_v2` existentes);
   `hypotheses_verdict.H20_low_confidence_n_test_flag` adicionado.

Comando de retomada/replay:
```
cd C:/Projetos/brazilgrid-loop/loops/forecast-mega-loop
uv run python scripts/h20_n_test_audit.py
```
(read-only; nao depende de CH nem MLflow.)

## Resultado

**Verdict: CONFIRMADO_DISPLAY**. Todos os 4 sub-claims atendidos:

| sub-claim | status | evidencia |
|---|---|---|
| C1 cobertura n_test | OK | 12 meta + 21 outros + 18 parquets = 51 artefatos; 1 falso positivo de regex excluido |
| C2 propagation runner | OK | run_bakeoff_replay.py L43-46/426-431/481-484/502-507 |
| C3 display leaderboard | OK | bloco politica topo + coluna n_test+flag tabela deprecada + lesson |
| C4 zero falso-negativo topo | OK | Todas tabelas ativas usam parquets per_fold UlFor n_test∈{59,60} |

**Audit final** (outputs/iter_0032/h20_leaderboard_low_n_test_warning/):
- 12 LOW em meta.json (iter_0002 LGBM replay, todos n_test=11)
- 8 LOW em outros JSON (iter_0002 + iter_0004 + iter_0008 + iter_0009 — todos derivados do mesmo replay n=11)
- 0 LOW em parquets UlFor (todos 59-60)
- 12/12 linhas summary_replay.csv LOW

Sanity checks: 6 default sao NA (H20 nao treina, nao computa CV, nao
introduz feature). B4 baseline_compare passa "por auditoria" (todas
baselines ativas usam parquets n_test∈{59,60}). 5 secondary
consistency checks passam (audit idempotent, coverage 100%, top sem LOW,
runner backward compat, csv backfill correto).

## Decisao

PROMOVE — politica documentada (`low_n_test_policy` em state.json),
runner patcheado, display atualizado. Nenhuma H derivada criada porque
o problema raiz (replay n=11 vs UlFor n=60) ja' foi reconhecido em
iter_0007 e o lesson de threshold ja' estava em H16 v1.1. H20 fecha o
gap residual: tornar essa politica visivel por linha em vez de embutida
em titulo de secao.

## Proximo passo

Backlog imediato apos H20:
- Queue ainda tem H22 (gbdt-only curt~gen+pdp gap nao-linear, P3,
  queued, ~1h) — atratividade subiu apos H22_model_aware lesson iter_0023.
- H33 (replicar joint-drop SE com LinearRegression, P3, derivada
  iter_0027 H8) tambem queued.
- H35 (alerta operacional moderado S via LogReg, derivada iter_0030 H15)
  queued — depende de UlFor adicionar endpoint /api/forecast/d1 expondo
  alerta binario (mudanca de produto, nao do loop).

H14, H18, H5, H12 seguem blocked aguardando UlFor ou Breno (req-0006,
req-0005, req-0004, req-0005 respectivamente).

Nada a propagar para UlFor — H20 e' interno ao loop (display + runner
do replay). Sem req novo emitido nesta iter.
