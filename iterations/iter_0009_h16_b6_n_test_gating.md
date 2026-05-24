---
alvo: h16_b6_zero_count_shift_robustness
layer: meta
iter_num: 0009
type: hypothesis_test (verdict=CONFIRMADO)
data_utc: 2026-05-24T06:45:00Z
baseline_tipo: B6 v1.0 (iter_0004) — sem n_test gating, sign_flip gate >0.05
hypothesis: |
  H16 (P1): Iter_0004 B6 deu falso positivo (SE/v3 lag sign-flip) por
  n_test=11 pequeno. UlFor req-0003 re-rodou com n=60 e correlacoes nao
  flipam. Patch B6: (1) se n_test<30 downgrade severity 1 nivel; (2)
  sign_flip exige |corr_train|>=0.2 AND |corr_test|>=0.2 (vs gate antigo
  >0.05). Regression test sintetico nT={10,60} mostra diferenca.
result_metric: |
  H16 CONFIRMADO. Patch aplicado em sanity_checks/zero_count_shift.py
  com novos params (n_test_downgrade_threshold=30,
  min_abs_corr_for_sign_flip=0.2) e novos campos (severity_raw,
  downgraded_due_to_small_n_test, n_test, sign_flip_blocked_by_min_abs_corr).

  Regression sintetico (DGP fraco-positivo lag1, n_train=365, 20 seeds):
    n_test=10:  old_sign_flips=5/20  new_sign_flips=0/20  new_downgrade=20/20
    n_test=60:  old_sign_flips=0/20  new_sign_flips=0/20  new_downgrade=0/20
  -> 4/4 criterios de aceitacao passam.

  Revalidation iter_0002 runs (test n=11):
    SE/v3 lag sign-flip iter_0004 (causa H16) -> severity high->medium
    SE/v3 curt_lag7 sign_flip -> BLOCKED (ct=0.196 < 0.2)
    SE/v3 9 features downgrade-adas
    NE/v1-v3 curt_lag1/lag7 sign_flips PERSISTEM (|ct|=0.66/0.42 — warning
      legitimo, gate novo nao mascara)
    N/v3 (n=11): 5 downgrade + 3 sign_flips blocked
decision: |
  CONFIRMADO. Patch ADOTADO no codigo canonical sanity_checks/zero_count_shift.py.
  Loop agora roda bake-off em janelas curtas (replay n=11) sem gerar req
  externa por falso positivo amostral. UlFor nao precisa re-investigar
  sign-flips de iter_0004 — eram, conforme req-0003 ulfor_verdict, ruido
  amostral; B6 v1.1 agora reproduz esse julgamento automaticamente.
sanity_checks_passed:
  leak_detection: skipped         # iter nao treina modelo novo, sem features novas
  permutation_importance: skipped # sem modelo novo
  holdout_temporal_strict: skipped # sem novo bake-off
  baseline_compare: skipped       # sem novo treinamento
  distribution_shift: skipped     # sem novo dataset
  zero_count_shift: true          # B6 e o proprio objeto da mudanca; auto-teste
                                  # via regression (sintetico nT={10,60}x20 seeds)
                                  # + revalidation (iter_0002 runs); 4/4 criterios
                                  # passam, 0 verdadeiros positivos perdidos em n=60
  meta_acceptance_criteria_all_pass: true
budget_consumido_iter: 0.7
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/sanity_checks/zero_count_shift.py (patched B6 v1.1)
  - loops/forecast-mega-loop/scripts/b6_regression_n_test_threshold.py (novo)
  - loops/forecast-mega-loop/outputs/iter_0009/h16_b6_n_test_gating/b6_regression_n_test_threshold.json
  - loops/forecast-mega-loop/outputs/iter_0009/h16_b6_n_test_gating/zero_count_shift_revalidation.json
  - loops/forecast-mega-loop/outputs/iter_0009/h16_b6_n_test_gating/sanity_checks_summary.json
  - loops/forecast-mega-loop/state.json (iter_atual=9, alvo_ativo+, b6_robustness_v2, H16 verdict, planner_next_iter)
  - loops/forecast-mega-loop/hypotheses_queue.md (H16 done + H20 nova)
  - loops/forecast-mega-loop/leaderboard.md (cabecalho + secao iter_0009 + linha sanity_check_B6_v1.1)
  - loops/forecast-mega-loop/iterations/iter_0009_h16_b6_n_test_gating.md
---

# Iter 0009 — H16 B6 zero_count_shift robustez n_test

## Contexto

Iter_0004 implementou B6 e detectou em SE/v3 que `curt_lag1`, `curt_lag7`,
`ger_eolica_mwh` tinham `sign_flip` (corr_train positiva, corr_test negativa).
Loop emitiu req-0003 ao UlFor. UlFor processou e respondeu em iter_0006:
em n=60d a direcao se mantem positiva — os sign-flips do replay n=11 eram
amostrais. H16 nasce para mitigar isso no proprio B6, sem dep externa.

## Hipotese

Duas mitigations ortogonais:

1. **n_test gating**: `n_test < 30 -> downgrade severity 1 nivel`
   (high->medium->low->none). Severity bruta preservada em `severity_raw`
   + flag `downgraded_due_to_small_n_test`.

2. **sign_flip gate de correlacao**: exigir `|corr_train| >= 0.2` E
   `|corr_test| >= 0.2` antes de marcar sign_flip (era `>0.05` em ambos).
   Em test n=11, correlacao amostral em `[0.05, 0.20]` nao discrimina
   sinal de ruido. Flag novo `sign_flip_blocked_by_min_abs_corr` documenta
   o que o gate antigo teria flaggado.

## Como foi rodado

### Patch B6

Edits em `loops/forecast-mega-loop/sanity_checks/zero_count_shift.py`:
- Constantes novas: `N_TEST_DOWNGRADE_THRESHOLD=30`, `MIN_ABS_CORR_FOR_SIGN_FLIP=0.2`
- Helper `_downgrade_severity(sev)` (high->medium->low->none)
- `check()` recebe `n_test_downgrade_threshold` e `min_abs_corr_for_sign_flip`;
  computa `n_test` do `X_test.height`; aplica downgrade quando `small_n` e
  feature `flag_raw=True`; retorna `severity_raw`, `downgraded_due_to_small_n_test`,
  `n_test`, e opcional `sign_flip_blocked_by_min_abs_corr` quando gate novo
  filtra um sign_flip que o antigo aprovaria.
- Standalone runner redireciona output para
  `outputs/iter_0009/h16_b6_n_test_gating/zero_count_shift_revalidation.json`
  e imprime colunas `dwn` (downgrades) e `sfblk` (sign_flips bloqueados).

### Regression test sintetico

Novo arquivo `loops/forecast-mega-loop/scripts/b6_regression_n_test_threshold.py`.
Inclui reimplementacao fiel do B6 pre-H16 (`_legacy_check`) para comparar
side-by-side. DGP: `lag1 = base + ruido` correlacionado fraco positivo com
`y = 0.25 * base + ruido`. Train n=365, 2% zeros aleatorios. Test sem zeros
(zero_rate ratio == 0 -> B6 flag medium ou high pelo ramo "max_zr > 0.01").
20 seeds para distribuicao amostral. Roda 2 cenarios: `n_test=10` e `n_test=60`.

```
$ uv run python loops/forecast-mega-loop/scripts/b6_regression_n_test_threshold.py
=== B6 regression test (H16) ===
n_test=10 (small):  old_sign_flips=5  new_sign_flips=0  new_downgraded=20
n_test=60 (large):  old_sign_flips=0  new_sign_flips=0  new_downgraded=0

Acceptance criteria:
  OK   small_n_old_flagged_at_least_one_sign_flip
  OK   small_n_new_reduces_sign_flips_or_downgrades_majority
  OK   large_n_no_severity_downgrade
  OK   large_n_new_sign_flips_le_old

VERDICT: CONFIRMADO
```

### Revalidation sobre iter_0002 runs

```
$ uv run python -m loops.forecast-mega-loop.sanity_checks.zero_count_shift

sub/ver     n_high   n_med  collapse   dwn  sfblk  high_severity_features
NE/v1            0       0         2     4      0
NE/v2            0       0         2     4      0
NE/v3            0       0         2     8      0
SE/v1            0       4         2     5      1
SE/v2            0       4         2     5      1
SE/v3            0       4         3     9      1
S/v1             0       0         0     2      0
S/v2             0       0         0     2      0
S/v3             0       0         0     6      0
N/v1             0       0         0     3      2
N/v2             0       0         0     3      2
N/v3             0       0         1     5      3
```

## Resultado

### Comparativo iter_0004 vs iter_0009 (SE/v3, target H16)

Quatro features em SE/v3 (test n=11) que iter_0004 marcou `severity=medium`
+ `signal_collapse=sign_flip`:

| feature | iter_0004 severity | iter_0004 sf | iter_0009 severity | iter_0009 sf | nota |
|---|---|---|---|---|---|
| `ger_eolica_mwh` | medium (raw=high) | yes | **medium** (downgrade) | yes | corr 0.277->-0.291 ambos >0.2 |
| `curt_lag1` | medium (raw=high) | yes | **medium** (downgrade) | yes | corr 0.252->-0.273 ambos >0.2 |
| `curt_lag7` | medium (raw=high) | yes | **medium** (downgrade) | **no** (blocked) | corr 0.196<-0.146 — `|ct|<0.2` |
| `val_import_mwmed` | medium (raw=high) | no | medium (downgrade) | no | — |

Ganho: 1/4 sign_flips bloqueado pelo novo gate de correlacao; 4/4 severities
downgrade-adas de high para medium pelo gating de n_test. Resultado alinha
com req-0003 ulfor_verdict que estes flips nao confirmam em n=60.

### NE/v1-v3 (controle — sinal forte)

`curt_lag1` e `curt_lag7` em NE tem `|corr_train|` = 0.66/0.42 em iter_0004.
Gate novo |corr|>=0.2 aprova ambos -> sign_flip PERSISTE. Validacao de que
o patch nao mascara warnings legitimos quando o sinal de train e forte.

### Verdict

H16 **CONFIRMADO** em ambos vetores:
- Regression sintetico: 4/4 criterios de aceitacao passam.
- Revalidation iter_0002: SE/v3 false-positive corretamente atenuado;
  N/v3 com mais downgrades+blocks (sub mais frio); NE preserva warning legitimo.

## Sanity checks (defaultlist)

| check | status | motivo |
|---|---|---|
| leak_detection | skipped | sem features novas, sem modelo novo |
| permutation_importance | skipped | sem modelo novo |
| holdout_temporal_strict | skipped | sem novo bake-off |
| baseline_compare | skipped | sem novo treinamento |
| distribution_shift | skipped | sem novo dataset |
| zero_count_shift | **passed** | objeto da mudanca, auto-testado por regression sintetico + revalidation |

Resumo no JSON: `outputs/iter_0009/h16_b6_n_test_gating/sanity_checks_summary.json`.

Sanity checks de modelo nao se aplicam (methodology hypothesis sobre rotina
diagnostica). B6 e o proprio escopo da mudanca e foi auto-testado por
duas vias independentes.

## Decisao

**ADOTADO**. Patch entra na rotina canonical `sanity_checks/zero_count_shift.py`.

Consequencias:
- Loop pode rodar B6 em replay n=11 sem ruido false-positive.
- Nao havera mais req externa motivada por sign-flip de baixa-confianca amostral.
- Audit preservado via `severity_raw` (severidade antes do downgrade) +
  `sign_flip_blocked_by_min_abs_corr` (o que o gate antigo flaggaria).
- H18 (auditar champions Ridge/LR via B1-B6 pos-OOT) ganha B6 mais robusto.

## Proximo passo

Planner sugere **H10** (P2 ensemble v2_LGBM + persist_d1 weighted by skill,
unblocked desde iter_0008). Alt: H19 (P2 extrair MAE/R²/F1 dos champions
UlFor via parse MLflow proxy) ou H7 (P2 XGB vs LGBM via CV). H20 (P3
derivada — auto-flag n_test<30 no leaderboard) entrou na queue para
seguimento natural de H16.

## Notas tecnicas

- `_downgrade_severity()` e idempotente entre niveis adjacentes; severidade
  `"none"` retorna `"none"` (clamping no max(0, idx-1)).
- `n_test` e derivado uma vez por `check()` (chamada por subset, nao por feature)
  — comportamento determinista para todas features daquele bake-off.
- `signal_collapse` agora pode marcar `True` por `magnitude_collapse` mesmo se
  `sign_flip` foi bloqueado (logica preservada — magnitude colapso e
  diagnostico ortogonal a sign).
- Back-compat com iter_0004 JSONs: campos antigos (`flag`, `severity`,
  `signal_collapse`, `signal_collapse_reason`) preservados; campos novos
  adicionados sem renomear nada.
