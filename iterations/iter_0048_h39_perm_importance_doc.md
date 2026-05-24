---
alvo: sanity_checks_doc_b2_interpretation
layer: meta
iter_num: 0048
type: hypothesis_test (verdict=DOCUMENTADO)
data_utc: 2026-05-26T12:00:00Z
hypothesis: |
  H39 (P5, queued desde iter_0040 — derivada de H30 REFUTADO_RIDGE com
  perm_importance massivo + R^2 piora): "PROMOVE doc canonico
  sanity_checks/B2_interpretation.md fechando o decoupling 'perm
  importance confirma sinal intrinseco != adicionar feature melhora
  modelo'." Caso pedagogico canonico: NE/v3 Ridge_alpha10 set B na fold
  final mostrou pdp_residual_total perm_importance +104.6% (base_mae
  17400 -> perm_mae 35600, std 2489) — sinal real, NAO ruido (delta 7.3x
  std). Ainda assim mean R^2 do mesmo set B perdeu -0.064 vs baseline A
  com 3 features brutas. Mesmo padrao em SE alpha=10 (+27.4% perm,
  -0.043 R^2). H21 LGBM iter_0020 ja' havia mostrado o mesmo decoupling
  em modelo de arvore (NE set C +6.6% perm / SE +10.8% perm; mean MAE
  bake-off NE +2.0pp / SE +3.1pp pior que A) — convergencia OLS + LGBM
  + Ridge cobre todas as familias de modelo.

  Aceitacao H39 (3 criterios, doc-only):
    - PROMOVE doc para sanity_checks/B2_interpretation.md
    - Inclui exemplo H21 LGBM + H30 Ridge com numeros verbatim
    - Adiciona paragrafo "what perm_importance means" + "what it does NOT mean"

  Type: governance/meta. Custo 0.2h, zero codigo de modelo/sanity, zero
  query CH. Sem prazo, queued desde iter_0040 (~6 dias).

result_metric: null
baseline_tipo: nao_aplicavel_governance
baseline_metric: null
decision: DOCUMENTADO
verdict_aceitacao_3_criteria: PASS
sanity_checks_passed:
  permutation_importance: NA
  holdout_temporal_strict: NA
  leak_detection: NA
  baseline_compare: NA
  distribution_shift: NA
  zero_count_shift: NA
  doc_aceitacao_3_criteria: PASS
  doc_numbers_verified: 6_of_6_match_exato
  doc_internal_links_resolve: REPORTED_4_wikilinks
budget_consumido_iter: 0.2
custo_estimado_usd: 0
hypotheses_queue_update: H39 status=done, iter_handled=0048, verdict=DOCUMENTADO
follow_ups_created: []
champions_impacted: ZERO_rollback_zero_promote
external_dependencies_requested: []
ulfor_loop_requests_appended: []
artefatos:
  - sanity_checks/B2_interpretation.md (153 linhas, doc canonico)
  - outputs/iter_0048/h39_perm_importance_doc/verdict.json
  - outputs/iter_0048/h39_perm_importance_doc/sanity_summary.json
  - outputs/iter_0048/h39_perm_importance_doc/summary.csv
---

# iter_0048 — H39 doc perm_importance signal != feature_engineering gain

## Hipotese

**H39 (P5, governance/meta).** Promove documento canonico de interpretacao
do sanity check B2 (`permutation_importance`) para fechar o caveat
metodologico que aparece sistematicamente em iters do arco residual:
perm_importance confirma que feature CARREGA sinal intrinseco, **mas
NAO** que adicionar feature MELHORA o modelo treinado. Decoupling
`sinal_intrinseco` vs `ganho_incremental`.

**Por que importa**: 3 iters do loop ja' tropecaram nesse decoupling como
ponto de discussao no verdict (H21 iter_0020, H30 iter_0040, H38
iter_0047). Sem doc canonico, futura iter que rodar B2 e ver perm gigante
vai ser tentada a tratar como evidencia de "feature serve" — produzindo
re-trabalho ou pior, falso CONFIRMADO. Doc estabelece heuristica
operacional (B2 diagnostico, B4 decisorio) e exemplos citaveis.

**Esperado:** doc de ~150 linhas com 2 casos pedagogicos numericos +
secao "o que B2 confirma" / "o que B2 NAO confirma" + heuristica gate +
tabela convergencia + referencias internas auditaveis.

## Como foi rodado

Iter doc-only, layer=meta. Zero codigo de modelo, zero query CH, zero
treino. Procedimento:

1. **Leitura source artifacts** (auditabilidade total):
   - `outputs/iter_0020/h21_pdp_residual_engineered/sanity_summary.json` →
     extrai `perm.{NE,SE}/v3.C.importance_pct` (LGBM defaults, CV 5x60d
     gap7d, 30 shuffles, n=440-442 dias).
   - `outputs/iter_0040/h30_pdp_residual_ridge_cv/sanity_summary.json` →
     extrai `perm.{NE,SE}/v3/ridge_alpha10.B.importance_pct` + base_mae +
     perm_mae_mean + perm_mae_std.
   - `outputs/iter_0040/h30_pdp_residual_ridge_cv/verdict.json` → extrai
     `alpha_verdicts.ridge_alpha10.delta_r2_{NE,SE}_B`.
2. **Cross-check numeros vs queue verdict_summary** (H21/H30/H38).
3. **Escrita** `sanity_checks/B2_interpretation.md` (153 linhas):
   - TL;DR com decoupling explicito.
   - "O que B2 confirma" (3 bullets) vs "O que B2 NAO confirma" (3 bullets).
   - Caso pedagogico 1 — H21 LGBM iter_0020 (tabela perm NE+SE + verdict).
   - Caso pedagogico 2 — H30 Ridge iter_0040 (tabela perm + tabela delta_R²
     alpha=1/10 NE+SE).
   - Secao "como evitar a armadilha" com heuristica operacional gate.
   - Tabela convergencia H3+H21+H30+H38 (modelo / B2 / B4 / verdict).
   - Referencias internas (4 wikilinks).
4. **Persist artefatos** em `outputs/iter_0048/h39_perm_importance_doc/`:
   `verdict.json` (3 criterios aceitacao PASS), `sanity_summary.json`
   (audit substitutivo + 6 verificacoes numero-vs-source), `summary.csv`.
5. **Update queue** H39 status=queued → done, iter_handled=0048,
   closure_summary preenchido.
6. **Update state.json** iter_atual 47→48 + last_verdict_summary + rotate
   last_verdict_summary_old_iter47.
7. **Update leaderboard.md** entry topo iter_0048 H39.

## Resultado

**Verdict: DOCUMENTADO.** 3 criterios de aceitacao H39 satisfeitos:

| Criterio | Status | Evidencia |
|----------|:------:|-----------|
| (a) Doc promovido a `sanity_checks/B2_interpretation.md` | PASS | 153 linhas, novo file |
| (b) Exemplo H21 LGBM com numeros verbatim | PASS | NE/v3 set C +6.6% / SE/v3 +10.8% perm citados |
| (c) Exemplo H30 Ridge com numeros verbatim | PASS | NE alpha=10 set B +104.6% perm / Δ R²=−0.064; SE +27.4% / Δ R²=−0.043 |
| (bonus) Paragrafos "what perm means" / "what perm does NOT mean" | PASS | Decoupling sinal_intrinseco vs ganho_incremental explicito |
| (bonus) Heuristica operacional gate | PASS | "B2 PASSA + B4 FALHA → verdict REFUTADO + razao basis-redundancia" |
| (bonus) Tabela convergencia H3+H21+H30+H38 | PASS | 4 linhas, arco residual encerrado |

**Verificacao numeros doc-vs-source:** 6/6 match exato.

| Claim | Source | Source value | Doc value | Match |
|-------|--------|-------------:|----------:|:-----:|
| H21 LGBM NE/v3 set C perm | `iter_0020 sanity_summary.json:perm.NE/v3.C.importance_pct` | 6.6256 | +6.6% | OK |
| H21 LGBM SE/v3 set C perm | `iter_0020 sanity_summary.json:perm.SE/v3.C.importance_pct` | 10.8001 | +10.8% | OK |
| H30 Ridge NE alpha=10 set B perm | `iter_0040 sanity_summary.json:perm.NE/v3/ridge_alpha10.B.importance_pct` | 104.6241 | +104.6% | OK |
| H30 Ridge SE alpha=10 set B perm | `iter_0040 sanity_summary.json:perm.SE/v3/ridge_alpha10.B.importance_pct` | 27.3512 | +27.4% | OK |
| H30 NE alpha=10 Δ R²_B | `iter_0040 verdict.json:alpha_verdicts.ridge_alpha10.delta_r2_NE_B` | −0.06419 | −0.064 | OK |
| H30 SE alpha=10 Δ R²_B | `iter_0040 verdict.json:alpha_verdicts.ridge_alpha10.delta_r2_SE_B` | −0.04313 | −0.043 | OK |

**Sanity checks default 6 = NA (governance/meta layer):** sem modelo
treinado, dataset ou predicao. Audit substitutivo executado:
`doc_aceitacao_3_criteria` PASS, `doc_numbers_verified` PASS, `doc_internal_links` REPORTED.

**Champions impactados: ZERO.** Doc-only, zero codigo de inferencia, zero
parametros tocados, zero rollback. Endpoint /api/forecast/d1 em prod
inalterado.

**Custo iter:** 0.2h = estimado (zero variance).

## Decisao

**DOCUMENTADO** — H39 fecha o caveat metodologico de H30/H21/H38.
Lesson canonica em forma auditavel ja' citavel por iters futuras.

Heuristica operacional do loop (agora em `sanity_checks/B2_interpretation.md`):

> Em hypothesis_test que propoe feature nova derivada:
> - B2 PASSA + B4 FALHA → verdict REFUTADO + razao "perm signal real
>   mas duplicata do basis/interacoes existentes" + cite H21/H30.
> - B2 PASSA + B4 PASSA → verdict CONFIRMADO (ortogonal real).
> - B2 FALHA → feature nao carrega sinal sob esse modelo; B4 secundaria.

## Proximo passo

Arco residual em curt D+1 ENCERRADO em iter_0046/0047 (H22+H30+H36+H38+H41
convergem). H39 sela so' o caveat metodologico que escapou daquelas iters.

**Zero follow-ups derivados:** o doc nao revela decisao tecnica nova; e'
codificacao de lesson ja' presente nos verdicts de H21/H30/H38.

Proxima iter: selecionar do queue (P3 candidates: H42 productize CQR-asym
em endpoint forecast_d1, H43 Mondrian aprendido) — ambas dependem de
acao Breno/UlFor (productize bandas) nao do loop. Alternativa: recon_delta
para inspecionar commits UlFor desde iter_0042 (consolidation) +
preempt/promote remanescentes. Sem alvo obvio P0/P1 vivo no queue.
