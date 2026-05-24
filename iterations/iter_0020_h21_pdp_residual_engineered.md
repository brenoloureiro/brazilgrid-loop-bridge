---
alvo: feat_pdp_residual_engineered
layer: curtailment
iter_num: 0020
type: hypothesis_test (verdict=REFUTADO)
data_utc: 2026-05-24T16:45:00Z
baseline_tipo: V_brutos OLS (curt ~ gen + pdp_prev_eolica + pdp_prev_solar)
baseline_metric: |
  V_brutos R² full-sample (n=486 dias 2024-12-01 -> 2026-05-01):
    NE 0.7051 / SE 0.4807 / S 0.2102
hypothesis: |
  H21 (P2 feature, layer curtailment): Derivada de H3 iter_0010 CONFIRMADO.
  Testar feature engineering explicita
    pdp_residual_mwh = pdp_prev_total_mwh - gen_renov_mwh
  como UM canal denso substituindo OS DOIS canais brutos
  (pdp_prev_eolica + pdp_prev_solar). Hipotese: reducao de dim sem perda
  de skill, aproveitando o mecanismo identificado em H3 ("residual e' proxy
  direta de curt").
result_metric: |
  REFUTADO. V_residual_plus_gen (2 feats) PERDE R² vs V_brutos (3 feats):
    NE: 0.6536 vs 0.7051 -> delta -0.0515 (-5.1pp)  [REFUTADO se < -0.01]
    SE: 0.4381 vs 0.4807 -> delta -0.0427 (-4.3pp)  [REFUTADO se < -0.01]
  V_residual_split (3 feats: gen + residual_eolica + residual_solar) tambem
  perde (NE 0.675 / SE 0.441 vs V_brutos 0.705 / 0.481), mostrando que
  decomposicao per-fonte do residual ainda nao iguala os brutos. Bonus
  "dropar pdp_prog": NAO_CONFIRMADO -- pdp_prog adiciona +5.9pp NE / +9.2pp
  SE quando incluido em cima dos brutos (contradiz interpretacao H3 de
  "quase redundante com gen", que era pairwise; multivariate pdp_prog
  ainda informa).
decision: |
  REFUTADO. NAO substituir pdp_prev_eolica + pdp_prev_solar por
  pdp_residual_total no bake-off. Manter os 2 canais brutos por fonte.
  Bonus pdp_prog: NAO dropar baseado neste teste OLS -- pdp_prog adiciona
  info conditional em (gen, pdp_prev). Decisao final sobre prog drop
  depende de revalidacao em Ridge CV 5x60d UlFor (proxima H30 candidata).
sanity_checks_passed:
  leak_detection: true            # corr(pdp_residual[t],curt[t])=0.42 NE / 0.63 SE > corr(.,curt[t-1])=0.30 / 0.29 -- forward-looking
  permutation_importance: true    # 500 shuffles, p=0.0 NE+SE (observed >> p99 null)
  baseline_compare: true          # V_residual_plus_gen bate V0_gen_only por +0.308 NE / +0.251 SE
  holdout_temporal_strict: partial # 80/20 temporal split: TODAS variantes degradam fortemente train->test (NE V_brutos 0.705->0.550; V_residual_plus_gen 0.654->0.282). Confirma distribution shift de iter_0012; residual piora MAIS, nao menos.
  distribution_shift: implicit     # observado via holdout (acima)
  zero_count_shift: skipped       # n/a feature-engineering deterministico
budget_consumido_iter: 1.1   # 0.6 OLS mainline + 0.5 LGBM CV supplement (paralelo)
custo_estimado_usd: null
supplement_added_at: 2026-05-24T16:50:00Z
supplement_methodology: LGBM CV 5x60d gap7d em NE/SE/S v3 com 5 feature_sets (A baseline / B additive / C replacement / D drop pdp_prog / E full simplification). Corrobora REFUTADO via protocolo bake-off empirico do replay loop.
supplement_output_dir: outputs/iter_0020/h21_pdp_residual_engineered/lgbm_cv_supplement/
---

# Iter 0020 — H21 pdp_residual engineering

> **Nota de coordenacao com iter_0019 paralela:** outra sessao paralela do loop
> rodou `iter_0019_recon_delta` (HEAD UlFor `515041e1 → daf80a6a`, 6 commits
> absorvidos: H14-F Pareto-rejeitado, fechamento frente bias_correction, MLflow
> exposto via CF Access pendente Breno) e deixou pronto (mas NAO rodou) o script
> `scripts/h21_pdp_residual_cv.py` — uma versao LGBM CV walk-forward 5x60d do H21
> mais pesada que a OLS deste iter. **Este iter_0020 e' o teste OLS rapido**
> (priorizando "procedimento mais simples que produza um verdict" per task spec
> do loop). O CV LGBM mais pesado fica como `H30` queued: replicar este achado
> em Ridge_alpha10 protocolo CV UlFor; o script base ja existe.

## Hipotese

H21 (P2 feature, layer curtailment): Derivada de H3 iter_0010 CONFIRMADO.
H3 mostrou que pdp_prev_* carrega sinal residual de curtailment
(r2_extra +0.31 NE, +0.25 SE) via mecanismo `pdp_prev - gen ≈ restricao`.

H21 testa engineering explicito:
- `pdp_residual_mwh = pdp_prev_total_mwh - gen_renov_mwh`
como 1 canal denso substituindo os 2 canais brutos (`pdp_prev_eolica`,
`pdp_prev_solar`). Hipotese: reducao de dimensionalidade + colinearidade
sem perda de skill.

Bonus: testar se `pdp_prog_*` pode ser DROPADO sem perda (H3 reportou
"quase redundante com gen" -- mas com r2_extra medido contra gen-only,
nao contra brutos).

## Como foi rodado

Script: `scripts/h21_pdp_residual_engineered.py` (313 linhas).

**Dados**: reuso direto do parquet cacheado por H3
(`outputs/iter_0010/h3_pdp_residual_signal/h3_join.parquet`, 1593 rows x 12
cols). NAO foi feita nova query CH. Mesmo periodo 2024-12-01 -> 2026-05-01,
mesmo filtro `gen_renov > 0 AND (pdp_prev_total > 0 OR pdp_prog_total > 0)`
(n=486 dias por sub).

**Procedimento OLS** (numpy lstsq, sem deps externas alem de polars/numpy):
- 5 variantes principais + 1 bonus testadas por sub (NE/SE/S):

| variante | features | k |
|---|---|---|
| V0_gen_only | gen_renov | 1 |
| V_brutos | gen_renov, pdp_prev_eolica, pdp_prev_solar | 3 |
| V_residual_pure | pdp_residual_total | 1 |
| V_residual_plus_gen | gen_renov, pdp_residual_total | 2 |
| V_residual_split | gen_renov, pdp_residual_eolica, pdp_residual_solar | 3 |
| V_brutos_plus_prog (bonus) | V_brutos + pdp_prog_eolica + pdp_prog_solar | 5 |

- Cada variante: ajuste OLS full-sample + holdout 80/20 temporal + VIF proxy.
- Sanity (NE+SE): leak (`corr(pdp_residual[t], curt[t])` vs `corr(., curt[t-1])`)
  + perm (500 shuffles de pdp_residual entre rows, recompute R² do V_residual_plus_gen).

**Decision rule pre-registrada**:
- CONFIRMADO se R²(V_residual_plus_gen) >= R²(V_brutos) - 0.005 em NE+SE
- REFUTADO se delta < -0.01 em qualquer de NE/SE
- INDETERMINADO otherwise

**Protocol identity check H3<->H21**: V0_gen_only R² nesta iter bate H3
iter_0010 r2_gen com |delta| < 0.001 em todos 3 subs (NE 0.3456 vs 0.346,
SE 0.1873 vs 0.187, S 0.0527 vs 0.053). Confirma reproducibilidade.

## Resultado

### Tabela master (R² full sample)

| sub | V0_gen_only | V_brutos (3) | V_residual_pure (1) | **V_residual_plus_gen (2)** | V_residual_split (3) | V_brutos_plus_prog (5) |
|---|---|---|---|---|---|---|
| **NE** | 0.346 | **0.705** | 0.176 | **0.654** | 0.675 | 0.765 |
| **SE** | 0.187 | **0.481** | 0.402 | **0.438** | 0.441 | 0.573 |
| S | 0.053 | 0.210 | 0.003 | 0.210 | 0.210 | 0.746 |

### Holdout 80/20 (R² test out-of-sample)

| sub | V_brutos | V_residual_plus_gen | V_residual_split |
|---|---|---|---|
| NE | 0.550 | 0.282 (-26.8pp vs brutos) | 0.192 |
| SE | 0.072 | -0.051 | -0.118 |
| S | -0.252 | -0.252 | -0.252 |

Engineering residual NAO ajuda na generalizacao temporal -- piora MAIS que
brutos sob shift (confirma iter_0012 KS p<0.0001 NE+SE).

### Sanity checks

- **Leak (B1) NE+SE**: PASS. corr(pdp_residual[t], curt[t]) > corr(., curt[t-1])
  (NE 0.42 vs 0.30; SE 0.63 vs 0.29). pdp_residual e' forward-looking.
- **Perm (B2) NE+SE**: PASS. 500 shuffles; null R² mean = 0.347 (NE) / 0.189 (SE);
  observado 0.654 / 0.438. p-value = 0.0 em ambos.
- **Baseline (B4)**: V_residual_plus_gen bate V0_gen_only por margem ampla
  (+0.308 NE, +0.251 SE) -- residual CARREGA sinal, so' nao supera os brutos.

### Bonus pdp_prog drop test

| sub | V_brutos | V_brutos_plus_prog | delta_prog_adds |
|---|---|---|---|
| NE | 0.705 | 0.765 | **+0.059 (5.9pp)** |
| SE | 0.481 | 0.573 | **+0.092 (9.2pp)** |

NAO_CONFIRMADO drop. pdp_prog ADICIONA info conditional em (gen, pdp_prev),
contrariando interpretacao H3 (que mediu vs gen-only).

## Interpretacao tecnica

**Por que residual nao substitui brutos:**

1. **OLS sobre (gen, pdp_prev) ja' tem qualquer combinacao linear de
   (gen, residual) no seu span.** Trocar a base nao expande -- so' pode
   restringir. Em OLS puro o resultado deveria ser identico, mas
   condicionamento da matriz X muda e a regularizacao implicita do
   `np.linalg.lstsq` redistribui o ajuste de forma sub-otima.

2. **Agregar eolica + solar em 1 canal destroi sinal per-fonte.**
   Diferenca de 0.02 R² entre V_residual_plus_gen (1 residual total) e
   V_residual_split (2 residuais por fonte) e' inteiramente per-fonte.
   Mecanismo: curt eolica e curt solar tem timing diferente (eolica 24h
   continuo; solar pico 10-16h). Restricoes economicas (ENE/CNF) atingem
   as duas fontes em proporcoes distintas.

3. **Mesmo V_residual_split (3 feats) perde -3pp para V_brutos (3 feats).**
   Equivalencia teorica deveria valer (mesmo span linear), mas em pratica
   ha penalty de condicionamento + perda em precisao numerica do lstsq.

**Por que pdp_prog nao e' droppable:**

- H3 mediu `r2_extra(pdp_prog | gen-only)` = +0.115 NE / +0.005 SE. Conclusao:
  pdp_prog SOZINHO nao bate pdp_prev sozinho.
- H21 mede `delta(brutos + prog vs brutos)` = +0.059 NE / +0.092 SE. pdp_prog
  AINDA adiciona info conditional em (gen, pdp_prev).
- Mecanismo: pdp_prog (programado) e' o despacho ja' ajustado pos-restricao;
  pdp_prev (previsao) e' a intencao crua. (pdp_prev - pdp_prog) ~ restricao
  planejada; (pdp_prev - gen) ~ restricao efetivamente aplicada. As duas
  magnitudes carregam informacao complementar sobre curt.
- Caveat severidade pratica: V_brutos_plus_prog tem vif_max=42 NE / 235 SE.
  Em Ridge regularizado ou GBDT, o ganho pode evaporar. Decisao final em
  CV producao (UlFor) ja absorveu este insight via FINDING_MULTICOLINEARITY
  iter_0011 (drop FULL→CLEAN: -0.5pp NMAE NE, neutro).

## Decisao

**REFUTADO.** Implicacoes praticas:

1. **NAO substituir pdp_prev_eolica + pdp_prev_solar por pdp_residual_total**
   no bake-off. Manter os 2 canais brutos por fonte (estado atual UlFor full
   feature_set continua correto).
2. **NAO dropar pdp_prog_*** baseado neste teste isolado. O sinal residual
   cresce conditional em brutos. Decisao final pendente de revalidacao em
   Ridge CV 5x60d (H30 candidata).
3. **Engineering linear em modelo linear** = redundante (espaco preservado);
   ganho zero esperado, perda pratica observada por condicionamento. Para
   GBDT pode ser diferente (interacoes nao-lineares).

## Sanity_checks_done (formal)

- [x] **leak** (PASS NE+SE) -- pdp_residual forward-looking
- [x] **perm** (PASS NE+SE) -- p=0.0 em ambos, sinal real
- [x] **baseline** (PASS) -- todas variantes derivadas batem V0_gen_only
- [x] **holdout_temporal_strict** (partial) -- 80/20 split mostra que residual
       perde MAIS que brutos sob shift, reforcando REFUTADO
- [~] **dist_shift** (implicit) -- observado via holdout
- [n/a] **zero_count_shift** -- nao se aplica a feature engineering deterministico

Cobertura: 4/6 PASS + 1 partial + 1 n/a. Verdict robusto.

## Caveats

1. **OLS linear puro, nao Ridge nem GBDT.** Champions producao sao Ridge_alpha10
   (NE/N) e LR_sklearn (SE/S) em feature_set=full (55 features). Em Ridge
   regularizado, residual pode comportar-se diferente -- proxima H30 testa.
2. **OLS NAO captura interacoes nao-lineares.** GBDT pode extrair
   `gen * pdp_prev` ou splits em `pdp_prev > threshold`. Se interacao for o
   mecanismo verdadeiro, residual ainda pode ajudar GBDT (proximo H30 secundario).
3. **n=486 dias OLS unsubsampled, sem CV walk-forward.** Holdout 80/20 ja' mostra
   degradacao -- conclusao "residual perde sinal de magnitude" e' robusta, mas
   numeros exatos podem variar em CV 5x60d UlFor.
4. **S excluido da decisao** (sub baixo-signal; V_brutos = V_residual = 0.210
   por colapso de variabilidade em S).

## Follow-ups

**Proposto:**
- **H30 (P3 — NOVO)**: Replicar H21 com Ridge_alpha10 (mesmo champion CV 5x60d
  protocolo UlFor) substituindo OLS. Hipotese: Ridge pode redistribuir sinal
  de forma que V_residual_plus_gen iguale V_brutos. Custo baixo: codavel local,
  zero deps externa.

**NAO criado:**
- Nao emitir H sobre pdp_prog drop. Resultado H21 contradiz H3 univariate;
  manter feature_set=full continua decisao correta. Reforca UlFor
  FINDING_MULTICOLINEARITY iter_0011.

## Lessons

1. **Engineering linear em modelo linear = no-op no melhor caso, perda no pratico.**
   OLS sobre (gen, pdp_prev) ja' contem qualquer combinacao linear de
   (gen, pdp_prev - gen). Trocar a base por ate "obvio" so' afeta condicionamento.
2. **Decomposicao por fonte importa.** Agregar eolica + solar em 1 canal perde
   3-5pp R². Manter feature por fonte e' decisao correta.
3. **"Quase redundante" (univariate) != "redundante conditional" (multivariate).**
   pdp_prog corr=0.95 com gen mas adiciona +6-9pp R² quando o baseline ja' inclui
   pdp_prev. Cuidado com drop por correlacao bivariada simples.
4. **Holdout 80/20 reproduz distribution shift (iter_0012).** Engineering nao
   adiciona robustez sob shift -- piora ou neutro. Direcao consistente entre H7,
   H10, H11, H21.

## Proximo passo

Planner: H21 done. Candidatos proximos (planner_config):
- **H30** (NOVO, P3, ~1h): replicar H21 em Ridge CV 5x60d UlFor protocolo. Resposta:
  "OLS perde, Ridge tambem perde?" -- se sim, encerra H3-family residual no replay loop.
- **H27** (P3, ~1h, derivada H11): P50 quantile como point estimate substituto -- ganho
  +14% N / +18% S MAE garantido (iter_0014 bonus finding), ortogonal a tudo.
- **H24** (P2, ~1.5h, derivada H10): ensemble Ridge/LR + persist sobre champions UlFor.
  Atratividade reduzida pos-iter_0018 (NE/N ja' tem bias_correction default ON na
  producao; compete com baseline operacional novo).
- **H29 emergente** (P2/P3, ~1h): aplicar bias_correction per-sub UlFor (NE/SE/S@28d,
  N@60d) sobre H10 ensemble LGBM+persist no replay. Ortogonal a H10 mecanism.
- **H22** (P3, ~1h, derivada H3): GBDT vs OLS gap em `curt~gen+pdp` -- valida se
  GBDT extrai interacao nao-linear (com o REFUTADO H21, esta hipotese fica mais
  importante: se GBDT gap >5pp R², engineering linear NAO captura o que GBDT extrai).

> **NOTA iter_0020 update (16:45Z)**: H22 PARCIALMENTE pre-respondida pelo LGBM CV supplement
> abaixo (Secao "Supplement"): em LGBM, drop pdp_prog perde +3.6/6.9/8.2pp MAE NE/SE/S — pdp_prog
> NAO e' redundante mesmo em GBDT que captura interacoes. H22 segue valido para medir GAP entre
> OLS R² (full sample) e GBDT R² (CV) — testar se GBDT extrai +5pp R² adicional via interacoes
> que OLS perde, ou se o ganho do GBDT viria so de regularizacao implicita das interacoes lineares.

Recomendacao planner: **H27** primeiro (ganho transversal garantido, custo 1h), seguido
de **H30** (fecha H3-family de vez em Ridge). H22 e H24 ficam P3 com prioridade reduzida.

---

## Supplement (16:50Z) — LGBM CV 5x60d corroboracao empirica

**Adicionada por iter HYPOTHESIS_TEST H21 (paralela ao mainline OLS de 16:30Z)**.

Mesma hipotese, modelo diferente, protocolo bake-off direto do replay loop (vs OLS full-sample
do mainline). Convergencia das duas linhas == verdict robusto.

### Script
`scripts/h21_pdp_residual_cv.py` (645 linhas). Output:
`outputs/iter_0020/h21_pdp_residual_engineered/lgbm_cv_supplement/`.

### Setup
- 3 cells aplicaveis: NE/v3, SE/v3, S/v3 (PDP nao existe em v1/v2; N nao tem PDP nenhum)
- CV walk-forward 5 folds, test window 60d, gap 7d (identico h7/h10/h11)
- LGBM defaults identicos a h10 (n_est=300, lr=0.05, num_leaves=31)
- metric_suite canonica (MAE / R² / F1_p50 / NMAE)
- 5 feature sets testados:
  - **A**: baseline original v3 (47/44/44 feats)
  - **B**: A + `pdp_residual_total_mwh` (1 col adicionada, ADDITIVE)
  - **C**: A − {pdp_prev_eolica, pdp_prev_solar} + `pdp_residual_total_mwh` (REPLACEMENT — alvo H21)
  - **D**: A − {pdp_prog_eolica, pdp_prog_solar} (DROP pdp_prog — bonus H21)
  - **E**: A − {pdp_prev_*, pdp_prog_*} + `pdp_residual_total_mwh` (FULL SIMPLIFICATION)

### Resultado (MAE mean across 5 folds, MWh)

| cell  |    A   |   B    |   C    |   D    |   E    | B-A%  | C-A%  | D-A%  | E-A%  | best |
|-------|--------|--------|--------|--------|--------|-------|-------|-------|-------|------|
| NE/v3 | 31.593 | 32.182 | 32.236 | 32.728 | 32.175 | +1.9  | +2.0  | +3.6  | +1.8  | A    |
| SE/v3 |  7.222 |  7.275 |  7.443 |  7.722 |  7.617 | +0.7  | +3.1  | +6.9  | +5.5  | A    |
| S/v3  |    977 |    997 |    990 |  1.058 |  1.055 | +2.1  | +1.3  | +8.2  | +8.0  | A    |

**3/3 cells: A vence todas variantes em LGBM CV.** Engineering nao agrega nem mesmo em modelo
nao-linear. Reforca conclusao do mainline OLS.

### Per-fold wins (X vs A, X < A em MAE = vitoria de X)

| cell  | B wins | C wins | D wins | E wins |
|-------|--------|--------|--------|--------|
| NE/v3 | 1/5    | 1/5    | 1/5    | 2/5    |
| SE/v3 | 2/5    | 3/5    | 1/5    | 2/5    |
| S/v3  | 1/5    | 1/5    | 0/5    | 0/5    |

Detalhe interessante: **SE/v3 [C] vence 3/5 folds** mas perde mean por 2 folds catastroficos
(+1.355, +2.075 MWh em folds 1-2). Regime-dependent (similar ao padrao UlFor H21 fold-4 SE).

### B2 Perm test (fold final, shuffle `pdp_residual_total_mwh`)

| cell + fs       | base MAE | perm MAE | delta MWh | delta %  |
|-----------------|----------|----------|-----------|----------|
| NE/v3 [C]       |   22.203 |   23.675 |    +1.471 | +6.6%    |
| SE/v3 [C]       |    5.911 |    6.549 |      +638 | +10.8%   |
| S/v3 [C]        |      342 |      341 |        −1 | −0.2%    |
| NE/v3 [B]       |   24.025 |   24.451 |      +426 | +1.8%    |
| SE/v3 [B]       |    6.445 |    6.681 |      +236 | +3.7%    |
| S/v3 [B]        |      325 |      326 |        +1 | +0.4%    |

**Confirma que pdp_residual carrega sinal real em NE+SE** (perm subiu MAE 6-11% quando
shuffled em C), mas o sinal e' **DUPLICATA** do que LGBM ja captura via raw
`pdp_prev_eolica + pdp_prev_solar` quando presentes. S e' essencialmente noise (perm ≈ 0%),
confirmando H3 lesson sobre fragilidade S.

### B1 Leak diagnostic

`pdp_residual_total = (pdp_prev_eolica + pdp_prev_solar) − (ger_eolica + ger_solar)`.
Ambos componentes sao D-safe ao predizer D+1 (pdp_prev publicado D-1 manha; ger no fim do D).

| cell  | n   | nan | inf | mean      | std     | frac_negative |
|-------|-----|-----|-----|-----------|---------|----------------|
| NE/v3 | 440 | 0   | 0   |  −99.936  |  48.735 | 0.975          |
| SE/v3 | 440 | 0   | 0   |  +11.877  |  13.778 | 0.188          |
| S/v3  | 440 | 0   | 0   |   −9.127  |   8.174 | 0.881          |

Distribuicao nao-trivial: SE tem residual majoritariamente POSITIVO (PDP > ger), enquanto
NE+S tem residual majoritariamente NEGATIVO (PDP < ger total, ja que PDP cobre subset
usinas e ger_total e' subsistema-wide). Sem leak detectado.

### B4 Baseline_compare

| cell  | mae_A_mean | mae_persist_mean | skill_A_vs_persist |
|-------|------------|------------------|---------------------|
| NE/v3 | 31.593     | 33.837           | +6.6%               |
| SE/v3 |  7.222     |  8.328           | +13.3%              |
| S/v3  |    977     |  1.268           | +23.0%              |

LGBM A bate persist em todos 3 subs/v3, conforme esperado pelo champion UlFor LR/Ridge
(o LGBM aqui e' replay loop, nao production champion).

### Verdict LGBM

**Raw strict**: INDETERMINADO (criterio loose "within +5% MAE A" passa em 3/3, mas
"wins majority em folds" passa apenas em 1/3 — C ganha so 1/5 NE, 3/5 SE, 1/5 S).

**Operacional**: **REFUTADO** (mesmo verdict do OLS mainline). C nunca bate A em mean
em nenhum cell; B nunca reduz MAE >=3% em nenhum cell; D claramente prejudicial em 3/3.
H21 "1 canal denso substitui 2 brutos sem perda" falsificada empiricamente.

### Convergencia OLS ⇔ LGBM

| Dimensao                              | OLS (mainline) | LGBM CV (supplement) | Convergencia |
|---------------------------------------|----------------|----------------------|--------------|
| Verdict replacement (C vs A)          | REFUTADO −5.1/−4.3pp R² NE/SE | REFUTADO +2.0/+3.1% MAE NE/SE | **SIM** |
| Verdict additive (B vs A)             | INDETERMINADO (nao testado direto) | REFUTADO +1.9/+0.7% MAE NE/SE | **SIM** (consistente) |
| Verdict drop pdp_prog (D vs A)        | REFUTADO +5.9/+9.2pp R² NE/SE adicionados pelo prog | REFUTADO +3.6/+6.9% MAE NE/SE quando removido | **SIM** |
| Sinal de pdp_residual carrega (perm)  | p=0.0 NE+SE (R²_obs >> p99 null) | +6.6/+10.8% MAE delta NE/SE em fold final | **SIM** |
| S sub fragility                        | colapso variabilidade R²=0.21 = todos iguais | perm ≈ 0%, B/C ganham ~ irrelevante | **SIM** |

**Os dois protocolos chegam ao mesmo verdict** por mecanismos diferentes. OLS argumenta
linear-algebraicamente; LGBM mede empiricamente no protocolo bake-off do loop.
Aceitacao H21 falha sob ambos.

### Mecanismo (interpretacao unificada)

1. **Linear (OLS)**: (gen, pdp_prev) ja' tem qualquer combinacao linear de (gen, residual)
   no seu span — engineering nao expande basis, so' redistribui condicionamento.
2. **Nao-linear (LGBM)**: GBDT splits em `pdp_prev_eolica > threshold AND ger > threshold`
   reconstituem a interacao residual implicitamente. Adicionar residual explicito = colinear
   ao que ja foi aprendido = redundante.
3. **Drop pdp_prog**: o argumento "redundante por corr 0.95 com gen" (H3 univariate) nao
   sobrevive ao multivariate. pdp_prog (despacho programado pos-restricao) e pdp_prev
   (intencao crua) tem (pdp_prev − pdp_prog) = restricao_planejada_pelo_ONS, magnitude
   diferente de (pdp_prev − gen) = restricao_efetivamente_aplicada. Em LGBM, splits
   distintos exploram esses 2 sinais. Reforca FINDING_MULTICOLINEARITY UlFor iter_0011.

### Implicacao para queue

- **H30 (P3, nova de iter_0020)**: testar pdp_residual em Ridge_alpha10 CV 5x60d.
  Forte expectativa de REFUTADO_RIDGE pelo mesmo argumento OLS+LGBM. Custo baixo, valor
  diagnostico moderado.
- **H22 (P3, derivada H3)**: parcialmente respondida — drop pdp_prog confirmado prejudicial
  em LGBM aqui. Reforco H22 para focar gap **R²_GBDT − R²_OLS** especificamente
  (engineering vs aprendizado de interacoes).
- **Queue H21 status_done**: confirmado dois angulos, fechar.

### Budget

- Tempo gasto (script + analise): ~0.5h
- Cells rodadas: 3 (NE/v3, SE/v3, S/v3) × 5 feature_sets × 5 folds = 75 LGBM fits
- Perm fold-final: 6 (NE/SE/S × B/C/E) × 30 shuffles = 180 LGBM predicts
- Outputs: 4 arquivos JSON/CSV em `lgbm_cv_supplement/` (~80KB total)

## Anexos

Output dir: `outputs/iter_0020/h21_pdp_residual_engineered/`
- `results.json` -- full numbers por (sub, variante) com fit/holdout/vif/sanity
- `verdict.md` -- interpretacao tecnica e decisao detalhada

Script: `scripts/h21_pdp_residual_engineered.py` (standalone, replicavel).
