---
alvo: curtailment_d1_ENE_CNF_NE_SE_S_N
layer: curtailment
iter_num: 0002
type: sanity_checks_formais
data_utc: 2026-05-24T01:30:00Z
baseline_tipo: persistencia_d1
hypothesis: |
  Bakeoff UlFor v3 (HEAD 18b400e5, add feat_pdp_renovavel) regrediu vs v2
  no NE. Testar 4 questoes formais: (Q1) leak em prev/pdp, (Q2) NE regression
  causa, (Q3) sobrevivencia SE/S em holdout estrito, (Q4) feature importance
  estavel.
result_metric: |
  NE/v2 LGBM NMAE=28.2% R2=+0.66 (best); NE/v3 NMAE=39.9% R2=-0.00 (regrediu).
  v3 NAO sobrevive holdout estrito em SE (+19.2pp NMAE).
decision: NAO-PROMOVE v3 para FASE 4 — voltar para v2 como referencia.
sanity_checks_passed:
  leak_detection: true     # 0 features com |corr|>0.95 vs y_true
  permutation_importance: true  # rodou em todos 12 (sub,ver)
  holdout_temporal_strict: true # rodou; expoe regressao NE+SE
  baseline_compare: true   # 4 baselines (incl climatologia DOY)
  distribution_shift: warning  # NE/v3 tem 47 PSI alerts (>v2: 43)
budget_consumido_iter: 2.4
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/outputs/iter_0002/runs/{NE,SE,S,N}/{v1,v2,v3}/
      (12 dirs, cada um com features.parquet, preds.parquet, meta.json, model_lgbm.txt)
  - loops/forecast-mega-loop/outputs/iter_0002/{leak_detection,permutation_importance,
      holdout_temporal_strict,baseline_compare,distribution_shift}.json
  - loops/forecast-mega-loop/outputs/iter_0002/summary_replay.csv
  - loops/forecast-mega-loop/scripts/run_bakeoff_replay.py
  - loops/forecast-mega-loop/sanity_checks/{leak_detection,permutation_importance,
      holdout_temporal_strict,baseline_compare,distribution_shift}.py  (esqueletos -> implementados)
---

# Iter 0002 — Sanity Checks Formais sobre bakeoff UlFor v1/v2/v3

## Hipotese

A sessao UlFor reportou v3 (feat_pdp_renovavel adicionada) com regressao
NE: NMAE 36.0% → 37.2%, R² +0.375 → +0.334. Quatro perguntas:

- **Q1**: leak em `feat_previsao_renovavel` ou `feat_pdp_renovavel`?
- **Q2**: regressao NE = redundancia multicolinear ou outra causa?
- **Q3**: ganhos SE/S sobrevivem holdout temporal estrito?
- **Q4**: ranking de feature importance estavel (permutation) em v3?

## Como foi rodado

### Constraints encontrados (Phase A)

1. **MLflow @ :5000 OFFLINE** no host local — runs do experiment `bakeoff-curtailment-d1`
   nao recuperaveis via API. Bakeoff_d1.py nao persiste y_pred/X_test em disco.
2. **3 tabelas feat_ ausentes do CH local**: `feat_calendario`, `feat_previsao_renovavel`,
   `feat_pdp_renovavel`. UlFor materializou em ambiente nao-local (provavelmente EC2)
   ou efemero. Rule "ClickHouse: apenas SELECT" impede `dbt run`.
3. **feat_termico stale** no CH local (ultimos dados 2024-12-31, gap de ~17 meses).

### Workaround (read-only, sem simular)

Reproducao via:
- Snapshot read-only dos 3 commits-chave em `outputs/iter_0002/_ulfor_v{1,2,3}_*.py`
  via `git show <sha>` (read-only sobre worktree UlFor)
- Reconstrucao client-side em Polars das 3 tabelas ausentes:
  - `feat_calendario`: derivado de `pl.col(dia).dt.ordinal_day()` (Fourier sem/ano + weekday)
  - `feat_previsao_renovavel`: CTE inline SQL sobre `stg_sintegre_eolica_previsao` +
    `stg_sintegre_previsao_solar` (logica bit-a-bit do dbt model)
  - `feat_pdp_renovavel`: `stg_ons_programacao_previsao` + crosswalk CSV +
    `dim_usina` fallback. Same logic do dbt model
- **Drop feat_termico**: stale demais; impossivel ter test n>11 com termico
- Treinos LGBM com `random_state=0, n_estimators=300, learning_rate=0.05,
  num_leaves=31, min_child_samples=10, subsample=0.8, colsample_bytree=0.9`
  (IDENTICO ao bakeoff_d1.py UlFor)
- Versoes via ABLACAO de features sobre v3 (NAO checkout, NAO modify):
  - v1 = sem PREV nem PDP
  - v2 = +PREV (SINtegre Combinada)
  - v3 = +PDP (PDP NE habilitado, estado pre-v3.1)

### Limitacoes do replay

- **Test n=11 dias** (nao 60): janela limitada pela staleness de `feat_intercambio`
  (max 2026-05-14) e `feat_carga_history` (max 2026-03-26) no CH local.
  Train n≈430-453 dias, OK.
- Numeros absolutos diferem do UlFor (NE/v2 ours=28.2% NMAE vs UlFor=36.0%) mas
  **direcao bate**: v3 regrediu vs v2 no NE em todas as 3 metricas (MAE, NMAE, R2).
- Bakeoff UlFor durante esta iter: sessao paralela commitou 3 novos commits
  (5e1bc70a v3.2 indicator, e283a03a v3.3 PDP NE off final, b7fa795c checkpoint).
  Nao refeito — entra em iter_0003.

## Resultado — Respostas Q1-Q4

### Q1: Ha leak em prev_renovavel ou pdp_renovavel?

**Resposta: NAO ha leak forte. SIM ha sinal de mis-alignment em PREV_EOLICA.**

Leak detection (correlacao Pearson/Spearman test set, threshold |r|>0.95):
- **0 features** com correlacao acima do threshold em qualquer (sub, ver) — 486 pares
  testados, 0 leak flags.

Teste comportamental — shift -1 dia (substitui valor t por t-1) em test, mede
delta NMAE no booster v3 ja treinado (`prev_/pdp_` features):

| Feature (NE/v3) | base NMAE | shift-1 NMAE | delta |
|---|---|---|---|
| prev_eolica_ger_mwh | 0.399 | 0.343 | **-5.5pp** ⚠ |
| pdp_prev_eolica_mwh | 0.399 | 0.356 | -4.2pp |
| pdp_prog_eolica_mwh | 0.399 | 0.366 | -3.3pp |
| pdp_prev_solar_mwh | 0.399 | 0.374 | -2.5pp |
| prev_solar_ger_mwh | 0.399 | 0.412 | +1.3pp |
| pdp_prog_solar_mwh | 0.399 | 0.434 | **+3.5pp** |

**Interpretacao**: `prev_eolica_ger_mwh`, `pdp_prev_eolica_mwh` melhoram NMAE
quando usam o dia D-1 em vez de D. NAO e leak (correlacoes <0.95) mas sugere
**mis-alignment**: o sinal "previsao para D+1 emitida em D" pode estar mal-alinhado
com como o modelo usa esses features no NE. Em test n=11 esse delta pode ser
ruido — em iter_0003 reverificar com test maior.

Em SE/v3 o sinal e mais claro: `pdp_prog_solar_mwh` shift +9.5pp piora — feature
funciona bem; `prev_solar_ger_mwh` shift -4.4pp melhora (mesmo padrao do NE).

### Q2: NE regression v3 = redundancia ou outra causa?

**Resposta: NAO e redundancia. PDP esta DOMINANTE em NE/v3. Causa real e
distribution shift train→test nas features PDP, somado a gaps Apr/2026
coalesced-a-zero.**

Permutation importance NE/v3 top-5 (delta MAE quando feature embaralhada):

| # | feature | importance (Δmae) |
|---|---|---|
| 1 | **pdp_prev_solar_mwh** | **5268.7** |
| 2 | is_weekend_d1 | 2651.8 |
| 3 | **pdp_prog_solar_mwh** | **1663.0** |
| 4 | prev_solar_ger_mwh | 450.0 |
| 5 | carga_mwmed | 400.7 |

PDP esta no top-1 e top-3. Multicolinearidade NAO drenou a importancia — o
modelo USA pesadamente as features PDP. Comparacao com NE/v2 top-5
(sem PDP): `is_weekend_d1=3242.2, prev_eolica_ger_mwh=1572.8, semana_sin_d1=974.6,
prev_solar_ger_mwh=194.1, curt_lag1=136.8`. Sem PDP, o modelo distribui peso entre
`is_weekend_d1` + `prev_eolica`.

**Por que entao a regressao?** Distribution shift (B5):
- NE/v2: 37 features com KS p<0.01 (de 43 testadas), 43 PSI alerts
- **NE/v3: 39 features com KS p<0.01 (de 47), 47 PSI alerts (+4)** — as 4 PDP cols
  adicionaram shift train→test
- Causa imediata: PDP tem **gaps de ingestao em Apr/2026** (cron falhou, ver
  CLAUDE.md UlFor + commit 18b400e5). UlFor coalesced a 0 — mas isso cria modo
  bimodal artificial (valores normais ~10k MWh + spike em 0). Treino aprende o
  shape pre-gap; test inclui dias coalesced.

### Q3: Ganhos SE/S em v3 sobrevivem holdout temporal estrito?

**Resposta: NAO para SE. S e ambiguo (NMAE NaN — ymean=0).**

Holdout estrito = train_end + 7d gap + test:

| sub/ver | NMAE original | NMAE strict | delta (pp) | R² strict |
|---|---|---|---|---|
| NE/v2 | 28.2% | 31.7% | +3.5 | +0.46 |
| NE/v3 | 39.9% | 46.5% | **+6.7** | -0.20 |
| SE/v2 | 36.2% | 42.6% | +6.4 | +0.28 |
| **SE/v3** | **38.8%** | **58.0%** | **+19.2** | **-0.08** |
| S/v1-v3 | nan | nan | nan | nan |
| N/v3 | 97.4% | 103.9% | +6.5 | -2.02 |

SE/v3 colapsa com gap de 7 dias: +19.2pp NMAE, R² vira negativo. Isso confirma
que o ganho aparente do SE/v3 vinha de adjacencia temporal train-test (modelo
estava decorando lag1/rmean7 da janela imediatamente anterior).

S e N permanecem nao-aprendiveis nessa janela (n_test=11, ymean=0 ou alvo
muito esparso).

### Q4: Ranking de feature importance estavel em v3 (permutation)

**Top-5 por subsistema (permutation importance LGBM v3, n_repeats=10, scoring=neg_MAE):**

```
NE: pdp_prev_solar_mwh (5269), is_weekend_d1 (2652), pdp_prog_solar_mwh (1663),
    prev_solar_ger_mwh (450), carga_mwmed (401)

SE: is_weekend_d1 (1293), pdp_prog_solar_mwh (247), curt_lag7 (196),
    taxa_penetracao_rmean7 (171), ger_eolica_mwh (155)

S:  cmo_range (12.7), pdp_prog_solar_mwh (10.4), cmo_mwmed_desvio_30d (9.3),
    semana_sin_d1 (7.9), carga_vale_mw (5.3)

N:  pdp_prev_solar_mwh (72.9), pdp_prog_solar_mwh (35.2), carga_mwmed_lag1 (24.3),
    val_net_mwmed_lag1 (16.1), is_weekend_d1 (12.8)
```

**Padrao 1**: `is_weekend_d1` aparece no top-5 de 3 subsistemas. Captura padrao de
demanda de fim-de-semana — efeito real.

**Padrao 2**: PDP solar (`pdp_prog_solar_mwh`, `pdp_prev_solar_mwh`) e dominante
em NE, SE e N. PDP eolica nao aparece — em parte porque PREV_EOLICA (SINtegre)
ja cobre NE/S e absorve esse sinal.

**Padrao 3**: lags de curt (`curt_lag1`, `curt_lag7`, `curt_rmean7`) aparecem so em
SE e S — modelo usa autocorrelacao quando ha sinal estavel.

### Comparacao oficial UlFor vs nosso replay vs holdout estrito (NMAE %)

| sub/ver | UlFor oficial | replay (ours) | holdout estrito | n_test |
|---|---|---|---|---|
| NE/v2 | 36.0 | 28.2 | 31.7 | 11 |
| NE/v3 | 37.2 (regrediu) | 39.9 (regrediu) | 46.5 | 11 |
| SE/v2 | 48.4 | 36.2 | 42.6 | 11 |
| SE/v3 | 46.2 (melhor) | 38.8 (melhor) | 58.0 | 11 |
| S/v2-v3 | 121→117 | nan | nan | 11 |
| N/v2-v3 | 72.3→72.2 | 97.4→97.4 | 89.8→103.9 | 11 |

Diferenca absoluta (UlFor maior) e esperada: nosso replay usa janela menor + sem
feat_termico + test n=11 vs 60. **Direcao bate** em NE (v3 regrediu) e SE
(replay diz "v3 melhor" mas com -19pp em holdout estrito o ganho some).

### Skill score vs 4 baselines (B4)

NE/v3 vs persist_d1 = **-0.04** (PERDE para repetir o dia). v2/v3 nao adiciona
informacao acima de persistencia robusta no NE com n=11.

NE/v2 vs persist_d1 = +0.27 (ganha 27%). v2 e o unico modelo que ganha vs
persist_d1 em NE.

Climatologia DOY = baseline mais fraco (ymean por dia-do-ano). NE/v2 ganha
+0.68; NE/v3 ganha +0.55.

## Decisao

**v3 NAO PROMOVIVEL para FASE 4 do UlFor — recomendar volta para v2 como
referencia + investigar PDP antes de tentar incorporar novamente.**

Justificativa baseada nas 4 perguntas:

- Q1 (leak): negativo em correlacao, mas mis-alignment em PREV_EOLICA sugere
  o modelo nao aproveita o feature como D+1-prev — entra como ruido em NE.
- Q2 (causa NE): PDP dominante na importancia mas com distribution shift +
  gaps Apr/26 coalesced-a-zero envenenam train. Removendo PDP do NE (que e
  o que a v3.1/v3.3 da sessao paralela faz) deve recuperar performance de v2.
- Q3 (SE strict): ganho SE/v3 colapsa com gap 7d (+19.2pp NMAE). Pseudo-ganho
  por adjacencia.
- Q4 (importance): estavel, mas dominancia PDP nao se traduz em ganho real
  (overfitting).

### Recomendacao para a sessao UlFor (nao executar, so recomendar)

1. **`feat_previsao_renovavel` (SINtegre Combinada): MANTER**
   - Ganho v1→v2 e robusto em NE (NMAE 47.0%→31.7% holdout estrito, R² -0.12→+0.46)
   - Sem PSI alerts catastroficos
   - Sinal real de previsao oficial D+1

2. **`feat_pdp_renovavel`: NAO USAR como esta. Antes de re-incorporar:**
   - Re-imputar gaps Apr/2026 em vez de coalesce(0) — usar rolling-7d-mean ou
     forward-fill dentro do (sub, fonte). Coalesce-zero cria modo bimodal toxico.
   - Verificar shift -1 dia em PDP_PREV_EOLICA — esta melhorando NMAE com
     shift, indicador de off-by-one no JOIN `addDays(t.dia, 1)`.
   - Re-treinar so apos fix (a) e (b), comparar com v2.

3. **v3.1/v3.3 (PDP NE off) da sessao paralela**: e patch valido — equivalente
   a so-NE descartar PDP. Manter mas reconhecer que e workaround, nao fix raiz.

4. **Modelo v3 NAO promovivel para FASE 4** ate (1) e (2) resolvidos.

5. **Para futuras iters**: usar `holdout_temporal_strict` (GAP 7d) como
   gate-by-default antes de declarar ganho — pseudo-ganhos por adjacencia
   sao comuns em problemas curtos.

## Proximo passo

### Iter 0003 (recomendado)

Frente B (descobertas novas durante iter_0002 + sessao paralela):

1. **Re-rodar replay com PDP no estado v3.3 (PDP NE off, pdp_has_data indicator)**.
   Sessao paralela commitou 3 novos HEADs (5e1bc70a, e283a03a, b7fa795c) durante
   esta iter. Snapshot esses e rodar mesmo pipeline.
2. **Testar fix PDP gaps Apr/2026** — substituir coalesce(0) por
   rolling_mean(7).forward_fill(). Comparar v3 (current) vs v3-fixed.
3. **Investigar off-by-one no JOIN PDP** — shift -1 dia em pdp_prev_eolica
   melhorar NMAE pode indicar erro de 1 dia no `addDays(t.dia, 1)`.

### Hipoteses novas (lista para iter_0003+)

- H1: **PDP coalesce(0) cria modo bimodal toxico**. Eliminar gaps via rolling
  mean recupera ganho esperado de PDP em todos subs.
- H2: **off-by-one PDP**. O JOIN `t.dia + 1` esta correto para PREV (que ja tem
  `dat_alvo = dat_produto + 1`), mas PDP ja tem `dat_programacao` = dia-alvo —
  entao o shift +1 esta DUPLICADO em pdp.
- H3: **N nao e aprendivel com n_train<500**. N tem ymean ~250 MWh/dia mas
  variancia gigante. Provavelmente precisa muito mais train.
- H4: **S e domain-shift puro**. NMAE NaN porque ymean test = 0, mas variance
  nao-zero (sub tem dias raros de curt ENE/CNF). Pode ser caso de classificador
  (rare events) antes de regressor.

### Pendencias operacionais

- `feat_calendario`, `feat_previsao_renovavel`, `feat_pdp_renovavel`,
  `feat_curtailment_history`, `feat_nwp`, `feat_gibr` NAO estao no CH local.
  Se UlFor pretende rodar replays/sanity locais no futuro, materializar
  via `dbt run --select feat_*` (essa janela e ulfor-only, nao do loop).
- `feat_termico` tem gap de ingestao desde 2024-12-31 (~17 meses) no CH local.
  Reativar fonte ou aceitar exclusao.
- `feat_carga_history` para em 2026-03-26 (~2 meses atraso). Precisa
  re-ingestao para ter janelas de teste razoaveis.
