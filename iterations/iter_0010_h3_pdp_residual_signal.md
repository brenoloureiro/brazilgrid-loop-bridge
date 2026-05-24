---
alvo: feat_pdp_renovavel_residual
layer: curtailment
iter_num: 0010
type: hypothesis_test (verdict=CONFIRMADO)
data_utc: 2026-05-24T07:30:00Z
baseline_tipo: r2_gen_only (curt ~ gen OLS contemporaneo)
baseline_metric: 0.346 (NE), 0.187 (SE), 0.053 (S) — R² do modelo so-gen
hypothesis: |
  H3 (P2): PDP carrega sinal alem de "previsao de geracao" via residual?
  Mesmo com corr alta gen[t] vs PDP[t] (0.94 prog, 0.91 prev) — sugerindo
  PDP ser proxy quase-perfeito de gen — testar se r2(curt ~ gen + PDP) -
  r2(curt ~ gen) > 0.05 em algum (sub, variant). Mecanismo esperado:
  discrepancia PDP_prev - gen = restricao = curtailment.
result_metric: |
  CONFIRMADO em NE+SE com r2_extra robusto e estavel:
    NE/pdp_prev: r2_extra = +0.308 (r2_gen=0.346 -> r2_full=0.654)
      partial_corr(pdp,curt|gen) = +0.686
      perm p=0.0 (null mean 0.0013, p99 0.0072 -> obs 43x acima)
      train/test split 80/20 temporal: 0.339 / 0.340 (delta +0.001, estavel)
    SE/pdp_prev: r2_extra = +0.251
      partial_corr = +0.556
      perm p=0.0
      train/test: 0.268 / 0.270 (estavel)
    S/pdp_prev: r2_extra = +0.157 train mas COLAPSA test (0.001) -> fragil
      regime change (curt-S baixo no Dez/2025-Mai/2026) + cobertura PDP-S
      so 12 usinas, todas eolicas (nenhuma solar local).
    NE/pdp_prog: r2_extra = +0.115 (significativo mas inferior)
    SE/pdp_prog: r2_extra = +0.005 (nao significativo)
  pdp_prog quase REDUNDANTE com gen (corr 0.95 NE, 0.75 SE, 0.93 S);
  pdp_prev carrega o sinal residual real.
decision: |
  CONFIRMADO. pdp_prev_eolica_mwh + pdp_prev_solar_mwh MANTIDOS no
  modelo (NE+SE definitivo, S condicional). pdp_prog_eolica_mwh +
  pdp_prog_solar_mwh CANDIDATOS A DROP (colinearidade 0.95 com ger_*
  e r2_extra <= 0.12). Sem req externo necessario — crosswalk inline
  funciona com cobertura local 208/606 (34%); UlFor ja' tem agregacao
  materializada (feat_pdp_renovavel, 82% cobertura) que pode rodar
  H21/H22 com mais ext.
sanity_checks_passed:
  leak_detection: true            # corr(pdp[t],curt[t]) > corr(pdp[t],curt[t-1]) 5/6 casos -> PDP forward-looking
  permutation_importance: true    # 500 shuffles, p=0.0 em 5/6 (SE/prog p=0.098 mas r2_extra=0.005 irrelevante)
  holdout_temporal_strict: skipped # nao se aplica (decomp OLS, sem modelo trainavel para gap=7d)
  baseline_compare: skipped       # baseline e' o r2_gen_only, ja' computado in-test
  distribution_shift: partial     # NE+SE estaveis (|delta|<0.003); S FAIL (test colapsa)
  zero_count_shift: skipped       # n/a feature-engineering (sem feature derivada nova com coalesce)
budget_consumido_iter: 0.7
custo_estimado_usd: null
---

# Iter 0010 — H3 PDP residual signal

## Hipótese

H3 (P2 feature, layer curtailment): PDP carrega sinal alem de "previsao
de geracao" via residual?

Corr alta gen[t] vs PDP[t] (0.94 prog, 0.91 prev medidos em iter_0003)
sugere PDP ser proxy quase-perfeito de geracao realizada. Investigar se
PDP adiciona algo alem disso via `residuals(curt ~ gen) ~ PDP`. Se
`r2_extra > 0.05`, PDP traz sinal de saturacao/curtailment alem de gen.

Mecanismo conjecturado: gen = previsao - restricao. Portanto
`pdp_prev - gen ≈ restricao ≈ curtailment` -> residual deveria correlacionar.

## Como foi rodado

Script: `scripts/h3_residual_test.py` (180 linhas).

**Dados** (CH local read-only, sem dbt/ALTER/INSERT):
- target: `curt_mwh = sum(val_curtailment WHERE cod_razaorestricao IN
  ('ENE','CNF') AND is_invalid=0 AND fonte IN ('eolica','solar')) / 2`
  por (id_subsistema, dia) em obt_usina_enriched.
- gen: `gen_renov_mwh = ger_eolica_mwh + ger_solar_mwh` em feat_saturacao.
- PDP: agregacao inline de `stg_ons_programacao_previsao` via crosswalk
  `mapeamento_conjunto_pdp.id_ons -> obt_conjunto.(id_subsistema, fonte)`.
  Cobertura: NE 174 usinas (133 eol + 41 sol), SE 22 (1+21), S 12 (eol),
  N 0 -> excluido.
- Periodo: 2024-12-01 -> 2026-05-01 (n_raw=531 dias por sub; n=486 apos
  filtro `gen > 0 AND (pdp_prev > 0 OR pdp_prog > 0)`).

**Procedimento OLS** (numpy lstsq, sem deps externas alem de polars):
1. `r2_gen_only` = OLS(curt ~ 1 + gen).rsquared
2. `r2_gen_plus_pdp` = OLS(curt ~ 1 + gen + pdp).rsquared
3. `r2_extra = r2_gen_plus_pdp - r2_gen_only`
4. `partial_corr(pdp, curt | gen)` via residuais de pdp~gen e curt~gen
5. Repetir para pdp_prev_total_mwh e pdp_prog_total_mwh

**Decisao rule** (registrada in-script): `max r2_extra > 0.05 across (sub,
variant) AND perm_p < 0.05 -> CONFIRMADO`.

**Sanity checks** integrados ao script:
- **perm** (500 shuffles de pdp entre rows, recompute r2_extra para null
  distribution). p-value = `(null >= obs).mean()`.
- **dist_shift** (split temporal 80/20: first 388 rows train, last 98 test).
  Comparar r2_extra train vs test.
- **leak** (corr(pdp[t], curt[t]) vs corr(pdp[t], curt[t-1]); flag se lag1
  mais forte que contemp).

## Resultado

### Tabela master por (sub, variante)

| sub | variante | r2_gen | r2_gen+pdp | **r2_extra** | partial_corr | corr(gen,pdp) | perm p | train→test delta |
|---|---|---|---|---|---|---|---|---|
| **NE** | pdp_prev | 0.346 | 0.654 | **+0.308** | +0.686 | 0.885 | 0.000 | +0.001 |
| NE | pdp_prog | 0.346 | 0.460 | +0.115 | +0.418 | 0.952 | 0.000 | +0.052 |
| **SE** | pdp_prev | 0.187 | 0.438 | **+0.251** | +0.556 | 0.354 | 0.000 | +0.002 |
| SE | pdp_prog | 0.187 | 0.192 | +0.005 | -0.074 | 0.754 | 0.098 | +0.000 |
| S  | pdp_prev | 0.053 | 0.210 | +0.157 | +0.408 | 0.917 | 0.000 | **-0.182** |
| S  | pdp_prog | 0.053 | 0.083 | +0.030 | +0.178 | 0.928 | 0.000 | -0.027 |

### Sanity checks

- **leak** PASS: corr(pdp[t], curt[t]) > corr(pdp[t], curt[t-1]) em 5/6
  combinacoes. Unica excecao: SE/pdp_prog (-0.37 vs -0.02, mas leak_suspect
  computa `|lag1| > |contemp| + 0.05` e fica false). PDP publicado pelo ONS
  em D-1, disponivel ao iniciar o dia D -> sem leak temporal mesmo no
  setup contemporaneo deste teste.
- **perm**: 5/6 com p=0.0 (observed acima do p99 do null, que fica em
  0.005-0.016). Unico p>0.05 = SE/pdp_prog (0.098, mas r2_extra=0.005 nao
  passa o threshold de qualquer forma — entra como "no signal" coerente).
- **dist_shift**:
  - NE+SE/pdp_prev: estabilidade quase perfeita (|delta|<0.003).
  - S/pdp_prev: COLAPSA. r2_extra_train=0.18 -> r2_extra_test=0.001.
    Cobertura PDP-S so 12 usinas eolicas + janela test (Dez/25-Mai/26)
    com curt-S extremamente baixo (mean ~818 MWh/dia, std 1783 -> regime
    de eventos raros). Sinal nao generaliza nessa amostra.

### Interpretacao tecnica

**Separacao PDP_prev vs PDP_prog e' a descoberta principal:**

- `pdp_prog` (programado) tracking-very-tight de `gen` (corr 0.95 NE, 0.93 S).
  Sinal redundante com geracao realizada. Faz sentido: o "programado" para
  D-1 ja' incorpora dispatch real apos restricao -> 1 dia depois pouco
  difere do que efetivamente gerou. Resultado: r2_extra apenas 0.005-0.12.
- `pdp_prev` (previsao independente, publicada D-1) sustenta r2_extra
  0.25-0.31 com partial_corr 0.56-0.69. **A discrepancia entre previsao
  e geracao realizada ESTA correlacionada com curtailment**. Mecanismo
  identificado: `pdp_prev - gen ≈ restricao_aplicada`. residual e' proxy
  direta do curtailment.

Coerente com a logica oficial ONS de curtailment (val_geracaoreferenciafinal
ou val_geracaolimitada) — que e' essencialmente `previsao - despacho_restrito`.

## Decisão

**CONFIRMADO.** PDP_prev deve permanecer no modelo (NE+SE definitivo,
S condicional). Implicacoes praticas:

- **MANTER** `pdp_prev_eolica_mwh` + `pdp_prev_solar_mwh` no bake-off
  v3.3+/v4 (ja estao la').
- **CONSIDERAR DROPAR** `pdp_prog_eolica_mwh` + `pdp_prog_solar_mwh`
  (colinearidade 0.95 com ger_renovavel, sinal residual <= 0.12). Reducao
  de feature space sem dano. Validar via H21 antes de aplicar.

Sem req externo necessario:
- Crosswalk inline (mapeamento_conjunto_pdp + obt_conjunto) funciona
  com cobertura local 208/606 (~34%).
- UlFor ja tem `feat_pdp_renovavel` materializado com cobertura 82% via
  seed_crosswalk_pdp_usina -> rodar H21/H22 la' diretamente da mais ext.

## Caveats

1. **Teste contemporaneo (mesmo D)**, nao D+1 forecast. Replica a estrutura
   da hipotese ("PDP[t] vs gen[t]") mas nao mede skill em D+1 diretamente.
   Para D+1: feature PDP[D+1] joina com gen[D] na predicao de curt[D+1] —
   e o uso real do bakeoff. O sinal residual demonstrado aqui justifica
   manter PDP_prev no bake-off D+1.
2. **S fragil** (test r2_extra colapsa). Pode indicar regime change em
   2026 (curt-S baixo) ou cobertura PDP-S insuficiente (12 usinas).
   Verdict S e' INDETERMINADO localmente; UlFor com cobertura maior
   provavelmente resolve.
3. **Cobertura PDP local 34% vs UlFor 82%**. Replicar em UlFor pode aumentar
   r2_extra e cobrir sub N (que aqui foi excluido por zero cobertura).
4. **OLS linear** — captura so a parte linear. GBDT (XGB/LGBM) provavelmente
   extrai mais via interacoes pdp x gen, sazonalidade. Quantificar via H22.

## Próximo passo

Planner_config aponta para **H21 (P2 feature engineering)**: testar
`pdp_residual_mwh = pdp_prev - gen` como 1 canal denso vs 2 brutos no
bake-off replay. Se sustentar mesmo skill, reduz dim sem perda. Sanity:
B1 leak (pdp_prev D-1-safe, gen D-only -> ok como feature em D para
predizer D+1), B2 perm, B4 baseline.

Alternativos: H10 (ensemble v2+persist, unblocked H9), H22 (GBDT vs OLS
gap), H19 (extrair MAE/R²/F1 dos champions UlFor), H7 (XGB vs LGBM CV),
H20 (auto-flag n_test<30 leaderboard).

## Anexos

Output dir: `outputs/iter_0010/h3_pdp_residual_signal/`
- `h3_join.parquet` (1593 rows, 11 cols) — dataset CH-extracted
- `results.json` — full numbers por (sub, variante) com fit/perm/shift/leak
- `verdict.md` — interpretacao e decisao detalhada

Script: `scripts/h3_residual_test.py` (standalone, replicavel).

## Bug encontrado durante a iter (auto-debug)

Primeira execucao reportou "CONFIRMADO -> INDETERMINADO" porque:

```python
(best.get("perm_p") or 1.0) < 0.05
```

interpretava p=0.0 como missing (Python `0.0 or x => x`). Patch:

```python
perm_p = best.get("perm_p") if best is not None else None
perm_p_significant = perm_p is not None and perm_p < 0.05
```

Lesson: nunca usar `or default` para floats que podem legitimamente ser 0.
