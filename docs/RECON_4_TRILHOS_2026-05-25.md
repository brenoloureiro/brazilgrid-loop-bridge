# Recon 4 Trilhos — 2026-05-25

> Recon read-only para localizar trabalho ja feito em 4 trilhos de
> granularidade/agregacao. ZERO modificacao de codigo, ZERO escrita CH.
> Apenas leitura de tabelas, codigo, docs e git history em todos os
> worktrees. Resultado serve de input pra sessao de retomada.

## Resumo executivo

**Os 4 trilhos NAO estao no zero.** Existem 9 worktrees ativos, ~50
arquivos relevantes em `experiments/`, ~100 em `products/bigforecaster/scripts/`,
um plano mestre vivo (`docs/forecast/PLANO_FINAL.md`, 35KB, 2026-05-22) e
um produto `products/cmo_engine/` que e essencialmente o Trilho C executado.

- **Trilho A — area geoeletrica.** Existe forte. CH tem `cod_areacarga`
  em `ons_carga_verificada` (2,05M, 34 areas, 2021-03→2026-05-21) e em
  `ons_carga_programada` (2,22M, mesma cobertura). Codigo de forecast
  per-area de **carga** existe e venceu ONS em 3/4 areas (commits
  `fb04025f`, `ed3d2394`, V4/V5 per-area bigforecaster). **Mapping
  estado→area documentado** em `services/forecasting/area_mapping.py`
  (26 areas + 4 perdas + 4 subsistemas). **Estado: ~80% — carga ok, mas
  area geoeletrica de GERACAO inexiste em CH** (REPORT_UNIFIED §2.1: nivel 3 = `id_estado` como proxy).
- **Trilho B — bottom-up usina/conjunto.** Trabalho mais maduro dos 4.
  4 REPORTs consolidados (`regional_forecaster`, `unified_forecaster`,
  `deep_dive`, `curtailment_por_razao/REPORT_FINAL`); 11 scripts
  numerados em `experiments/curtailment_por_razao/`;
  `agg_eolica_previsao_conjunto` MERGEADO em master via PR#33;
  forecast_d1_conjunto.py produtizado. Vereditos honestos:
  CatBoost 15,8% NMAE eolica NE per-usina; smoothing 2,1× usina→subsistema
  NE; per-S NO-GO. **Estado: ~85% — direcao decidida, falta deploy operacional.**
- **Trilho C — granularidade DESSEM.** CH tem 18 tabelas dessem_*. Granularidade
  fina disponivel: `dessem_dbar` (356k rows, **12.833 barras**),
  `dessem_cmobar` (CMO por barra×IPER), `sintegre_dessem_eolica` (2M rows,
  **3.664 usinas**), `dessem_dger` (geracao por usina+barra). **Forecast em
  granularidade barra/usina-DESSEM existe** em `products/cmo_engine/` (surrogate
  DESSEM ativo: dados 2023-10→2026-03 semi-horario, walk-forward mensal,
  targets CMO+curt+delta+PLD) mas o ALVO e CMO NE/SE agregado, nao per-barra.
  Forecast per-barra do CMO nao foi tentado. **Estado: ~50% — dados ok,
  modelos surrogate existem, granularidade barra subaproveitada.**
- **Trilho D — residuo DESSEM (programada vs verificada).** Trilho MAIS
  ABANDONADO. Codigo existe pontualmente: `scripts/analysis/eda_erro_previsao_carga.py`
  (calcula `erro = val_cargaglobalprogramada - val_cargaglobal` por
  area NE, 7 areas) e `eda_previsao_vs_programacao.py` (prev vs prog
  para geracao). **Mas NAO ha modelo que aprenda o residuo.** Dados estao
  prontos (cobertura completa 2021-03→2026-05 para ambas as series, ambas
  por `cod_areacarga`). REPORT v3 da `fase4_v3` rejeitou `carga_liquida` como
  feature D+1 mas usou DESSEM-despachado **pos-corte**; nao testou
  **residuo programada−verificada D-1** como sinal D+1. **Estado: ~15%
  — EDA feita, modelo nao construido. Maior gap → menor esforco remanescente.**

---

## Trilho A — Area Geoeletrica

### A.1 Tabelas ClickHouse com `cod_areacarga`

| Tabela | Rows | Range | Areas distintas |
|---|--:|---|--:|
| `ons_carga_verificada` | **2.051.232** | 2021-03-01 → 2026-05-21 | **34** |
| `ons_carga_programada` | **2.220.288** | 2021-03-01T06:30 → 2026-05-22T03:00 | **34** |

**Distribuicao das 34 areas (verificada):**
- 4 subsistemas: SECO, NE, S, N (91k rows cada)
- 11 areas com cobertura completa (BASE, BAOE, ALPE, CE, MA, PBRN, PI etc. — todas NE — 91k rows cada)
- 19 areas com cobertura parcial (AC, AM, AP, DF, ES, GO, MG, MS, MT, PA, PR, RJ, RO, RR, RS, SC, SP, TON, TOCO — 54k rows)
- 4 perdas: PES, PESE, PEN, PENE (13-14k rows; cobertura parcial)
- **TOCO terminou 2022-09-03** (area deprecated, atencao)

### A.2 Codigo encontrado

**Mapping canonico estado→area:**
- `services/forecasting/area_mapping.py` — 88 linhas, 26 areas geoeletricas mapeadas (BASE=BA+SE, ALPE=AL+PE, PBRN=PB+RN, BAOE=Bahia Oeste, TON=Tocantins Norte etc.); inclui caveats (BAOE precisa mapping conjunto-a-conjunto; TON/TOCO precisa refinamento).

**Forecast de CARGA per area (bigforecaster, branch `bigforecaster`):**
- `products/bigforecaster/scripts/forecast_carga_v3_multi_area.py` — V3 multi-area
- `products/bigforecaster/scripts/forecast_carga_v5_per_area.py` — V5: **1 modelo POR area + NWP centroide individual**
- `forecast_carga_v5_sul_specialist.py`, `comparativo_por_area.py`, `feature_importance_por_area.py`
- `compare_bg_vs_ons.py`, `train_carga_mmgd_forecasts.py`
- **Resultados pushados na branch `bigforecaster`:**
  - `0ee1d012` Carga V2 — 3% NMAE vs ONS 8%
  - `fb04025f` Carga V4 vence ONS em 3/4 areas (HDD/CDD + MMGD V2 + cargaglobalcons)
  - `ed3d2394` V5 per-area — 1 modelo POR area + NWP centroide individual
  - `87cebe11`, `cbd42587`, `5746f9fa`, `ac852121`, `febce83c` (V3 multi-area, 32 areas, bandas P10/P90)
  - `0fd2278c` Carga V5 modelo unico + S weighted 2x + holiday features ricas

**ML 32 areas + perdas (commit msgs):**
- "V3 multi-area (32 areas geoeletricas)"
- "UI selector area geoeletrica - inclui PES/PENE/PEN/PESE"

### A.3 Docs

- `docs/forecast/PLANO_FINAL.md` (35KB, 2026-05-22) — plano UlFor mestre que orquestra os 4 trilhos.
- `experiments/regional_forecaster/REPORT_REGIONAL.md`
  - Pivota analise **regional** mas usa `id_estado` como proxy de area
    (e o limite identificado: area geoeletrica de **geracao** inexiste em CH).
- `experiments/unified_forecaster/REPORT_UNIFIED.md` §2.1 — explicita o
  limite: "Area geoeletrica de geracao inexiste em CH → Nivel-3 = id_estado".

### A.4 Conclusao Trilho A

**Estado real: ~80% (parcial-completo).** Para **CARGA** a granularidade
area-geoeletrica esta totalmente operacionalizada — dados, mapping, modelos
V3-V5, comparativo vs ONS. **Para GERACAO** (eolica/solar/curtailment), o
trilho NAO existe ainda — usa-se `id_estado` como proxy. Bottleneck: nao ha
crosswalk "qual area-geoeletrica gera onde" na dim de usinas; o codigo
existe em `area_mapping.py` mas e estado-based (so resolve carga, nao
ativos de geracao que cruzam estados via SIN).

---

## Trilho B — Bottom-up (usina / conjunto)

### B.1 Inventario de tabelas e modelos

**ClickHouse — agg per-conjunto/usina:**
- `agg_eolica_previsao_conjunto` (Refreshable MV) — **MERGEADO em master via PR#33**
- `agg_portfolio_per_ceg` (43.6k rows, base do perf_metrics) — Refreshable MV 30min
- `agg_usina_mensal`, `agg_conjunto_mensal`
- `bridge_usina_conjunto`
- `obt_usina_enriched` — fonte de verdade per-usina (~90M rows)
- `dim_usina` (wide, 9 fontes, PK ceg)

**Forecast scripts per-usina/conjunto:**
- `products/bigforecaster/scripts/`:
  - `curt_v6_per_usina.py`, `curt_v12_per_usina_nwp.py`
  - `forecast_v6_per_usina.py`, `forecast_v10_2_ute_per_usina.py`
  - `forecast_v9_2_uhe_per_usina.py`, `forecast_v9_3_uhe_semihorario.py`
  - `compute_bigfc_v2_usina.py`, `train_lgbm_curt.py`, `train_lgbm_eol.py`, `train_lgbm_ufv.py`
  - Commits: `V53 per-usina dedicado`, `V56 per-conjunto top 50`, `90cd8b2c V54 reconciliacao hierarquica`
- `experiments/curtailment_por_razao/` (branch `exp/eolica-d1-onsprev-20260521`):
  - 11 scripts (01_analise_ene_sobreoferta → 11_estado_transmissao)
  - `10_modelo_per_conjunto.py`, `forecast_d1_conjunto.py` (produtizado)
  - **REPORT_FINAL.md consolidado**
- `experiments/regional_forecaster/` (branch `exp/regional-system-20260519`)
- `experiments/unified_forecaster/` (branch `exp/unified-system-20260518`)
- `experiments/deep_dive/` (branch `exp/deep-dive-20260519`)

### B.2 Vereditos consolidados (dos 4 REPORTs)

| Frente | NMAE per-usina | Smoothing (usina→subsistema) | Veredito |
|---|--:|--:|---|
| Eolica D+1 NE | CatBoost 15,8% | 2,10× normal / **2,99× evento** | **GO agregado subsistema** |
| Eolica D+1 S | CatBoost 24,1% | 1,43× / 1,46× | **NO-GO** (poucos sites + frentes correlacionadas) |
| Solar D+1 NE | CatBoost 8,5% | 2,19× / 1,60× | GO agregado |
| Solar D+1 SE | LightGBM 7,6% | — | GO |
| Curtailment D+1 NE | LGBM **MAE 2,4 MWh, R² 0,23, F1 0,55** | — | NO-GO ponto, OK F1/quantile |

**Achado V0 forense (deep_dive):** a metrica NMAE 104% era artefato de
zero-inflation (66% zeros + media baixa). Metrica correta: **MAE(MWh) +
R² + F1(curt>0)** — curtailment per-usina tem sinal real fraco-mas-mensuravel.

### B.3 Estado das branches

- `feat/eolica-previsao-conjunto` — **MERGEADO PR#33** (eolica per-conjunto)
- `exp/eolica-d1-onsprev-20260521` — REPORT_FINAL escrito, NAO mergeado
- `exp/regional-system-20260519` — REPORT_REGIONAL pronto, NAO pushado (memoria diz "branch local nao pushada")
- `exp/unified-system-20260518` — REPORT_UNIFIED pronto, branch local
- `exp/deep-dive-20260519` — REPORT_DEEP_DIVE pronto

### B.4 Conclusao Trilho B

**Estado: ~85% (mais maduro dos 4).** Direcao operacional decidida com
evidencia robusta. **Bottleneck** unico restante: a maioria dos REPORTs
estao em branches locais NAO PUSHADAS; o produto vendavel
(eolica/solar D+1 agregado subsistema com Cat/LGBM) nao esta deployado
em `services/analytics_api/` nem expoe API.

---

## Trilho C — Granularidade DESSEM

### C.1 Tabelas ClickHouse `dessem_*`

| Tabela | Rows | Range | Granularidade unica |
|---|--:|---|--:|
| `dessem_dbar` | **356.236** | 2026-02-01 → 2026-03-26 | **12.833 barras** (lat/lon/area/kV/pg_mw/pl_mw, por patamar dom15h etc.) |
| `dessem_cmobar` | 1,2M (memoria) | — | barra × IPER (1-48), com subsistema |
| `dessem_dger` | 15k (memoria) | — | usina × barra |
| `dessem_dlin` | 365k (memoria) | — | linha |
| `dessem_renovaveis_previsao` | — | dat_deck 2025-01-01+ | usina (UTE+UHE+EOL+UFV), col `dat_deck` (atencao: nao `dat_produto`), inclui MMGD via SOMBRA_REE |
| `sintegre_dessem_eolica` | **2.079.680** | 2026-02-01 → 2026-03-26 | **3.664 usinas** (val_potencia_mw, val_geracao_mw, val_fatcap, barra, subsistema) |
| `sintegre_dessem_somflux` | 8k (memoria) | — | fluxo |
| `sintegre_dessem_cmosist` | 21.648 | 2025-07-01 → 2026-03-26 | subsistema × IPER (SE/NE/S/N/FC) |

**Limitacao crucial (REPORT_v3 §4):** `sintegre_dessem_eolica` tem
`val_fatcap≡1.0` no periodo curto observado → inutilizavel como
"renovavel-potencial-prevista" para D+1 de curtailment.

### C.2 Codigo encontrado

**Produto `products/cmo_engine/` — Surrogate DESSEM ATIVO:**
- `src/build_dataset_dessem.py`, `src/train_dessem_surrogate.py`, `src/data_loader.py`, `src/inference.py`, `src/benchmark_vs_ons.py`
- Docs: `arquitetura.md`, `benchmark_vs_ons.md`, `inventario_dados.md`, `report_camada1.md`
- **Camada 1 ATIVA**: targets CMO NE/SE, Curtailment NE, Delta CMO, PLD estimado; ~29 meses (2023-10→2026-03), walk-forward mensal
- **Camadas 2 e 3 sao PLACEHOLDERs** (1 dia parseado de DECOMP/NEWAVE — inviavel treinar)
- **NOTA: granularidade do surrogate e SUBSISTEMA (CMO NE/SE), nao barra.**

**Experiments contra DESSEM:**
- `experiments/curtailment_por_razao/07_validar_dessem_previsao.py` — validacao DESSEM previsao
- `experiments/curtailment_por_razao/08_walkforward_dessem.py` — walk-forward dessem
- `experiments/forecast_curtailment/build_dataset_ne.py` (v3 GIBR + DESSEM programado)

**Parser de decks DESSEM:**
- `scripts/analysis/parse_decks_fev2026.py`
- `scripts/analysis/eda_balanco_dessem.py`

**Forecast usando DESSEM como feature/ancora:**
- `products/bigforecaster/scripts/forecast_v14_4_dessem_anchor.py` (commit `e3b4a52e`: "DESSEM oficial e ground truth — NMAE_log 1.9%")
- `forecast_v14_5_offer_based.py`
- `forecast_v14_cmo_pld.py`, `forecast_v14_2_cmo_pld_semanal.py`

### C.3 Veredito existente sobre DESSEM como feature D+1 curt

`forecast_curtailment/REPORT_v3.md` §4 — **REPROVOU** DESSEM-despachado como
feature D+1 de curtailment:
> "stg_ons_balanco_dessem_detalhe sinal DESPACHADO pos-corte → corr ρ≈−0.05, armadilha"

Razao: DESSEM tem o despacho **apos** o curtailment do ONS, entao a relacao
e circular.

### C.4 Conclusao Trilho C

**Estado: ~50% (intermediario, com surrogate consolidado).** Dados
disponiveis em **granularidade fina** (barra para topologia, usina para
eolica DESSEM), mas exploracao em granularidade-barra subaproveitada.
**O que existe:** surrogate CMO/curt agregado subsistema (`cmo_engine`).
**O que NAO existe:** forecast per-barra; analise de quais barras tem CMO
descolado do subsistema; uso de `dessem_dbar` como **mapa topologico**
para curtailment per-corredor (gap apontado em fase4_v3: "GIBR so tem
valor em granularidade por conjunto/corredor"). **Worktree `revisions-tracking`**
(branch dela) adicionou Fase A/B/C de revisions-tracking — verificar se
inclui granularidade DESSEM antes de retomar.

---

## Trilho D — Residuo do DESSEM

### D.1 Dados disponiveis (prontos)

| Tabela | Rows | Range | Granularidade |
|---|--:|---|--:|
| `ons_carga_programada` | 2.220.288 | 2021-03-01 → 2026-05-22 | cod_areacarga (34) |
| `ons_carga_verificada` | 2.051.232 | 2021-03-01 → 2026-05-21 | cod_areacarga (34) |
| `sintegre_prevcarga_dessem` | 12.288 | 2026-02-19 → 2026-04-03 | id_subsistema (4) — esparso |

**Cobertura completa para Trilho D em 5 anos por area geoeletrica.**
Diferenca direta `val_cargaglobalprogramada − val_cargaglobal` calculavel
por linha (mesmo `din_referenciautc` e `cod_areacarga`).

### D.2 Codigo encontrado

**EDA explicita sobre o residuo:**
- `scripts/analysis/eda_erro_previsao_carga.py` (80 linhas iniciais lidas)
  > Formula: `erro = val_cargaglobalprogramada − val_cargaglobal`
  >  - erro > 0 → ONS superestimou (potencial excesso de geracao)
  >  - erro < 0 → ONS subestimou
  > Hipotese: ONS superestima carga → despacha mais geracao → excesso → curtailment.
  > MMGD cresce mais rapido que a previsao → carga liquida real < programada.
  - AREAS_GEO_NE = ["CE", "PBRN", "ALPE", "BASE", "MA", "PI", "BAOE"]
- `scripts/analysis/eda_previsao_vs_programacao.py` (181 charts em `charts/`)
  > Investiga `ons_programacao_previsao`: val_geracaoprogramada vs val_previsao
  > **Achado:** "EDA de 'erro de previsao' pode ter medido curtailment, nao
  > erro do modelo meteorologico"
- `scripts/analysis/eda_previsao_vs_programacao_v2.py`
- `scripts/analysis/eda_erro_previsao_corrigida.py`
- `scripts/analysis/eda_erro_previsao_geracao.py`

**Forecasts tangenciais (NAO residuo direto):**
- `products/bigforecaster/scripts/forecast_v10_ute_residual.py` — UTE residual
  top-down (commit `82494cdd` V10.1: "UTE residual top-down — NMAE 9.4%")
- `analise_carga_nwp_erro.py`, `analise_carga_vies_clima.py`

**Fase 4 v3 do curtailment usou erro_prev_carga como feature mas REJEITOU:**
- `forecast_curtailment/REPORT_v3.md`: `erro_prev_carga_mw` = ex-post, leak
  → so vale pra nowcast (R² 0,965), nao pra D+1.

### D.3 Lacuna identificada

NAO existe modelo que tente **prever o residuo D+1 como alvo proprio.**
Todas as referencias usam o residuo como:
1. **feature** num modelo de curtailment (rejeitado por leak em REPORT_v3) — Fase4
2. **diagnostico EDA** (eda_erro_previsao_carga.py — qualitativo)
3. **ground truth para validar DESSEM** (cmo_engine benchmark_vs_ons.md)

O alvo "prever amanha o erro que o DESSEM vai cometer" — que abre tres
produtos derivados (1) sinal de curtailment preventivo, (2) hedge MMGD para
operador, (3) ajuste de bias para previsao operacional — esta **vazio**.

### D.4 Conclusao Trilho D

**Estado: ~15% (mais abandonado).** Dados completos, EDA inicial feita,
mas **modelo nao construido**. Eh o trilho com **maior gap entre o que
existe e o que poderia existir.** Por outro lado: features candidatas
(MMGD verificada, area-geo, dia da semana, feriado, NWP, calor) ja foram
todas operacionalizadas no V4/V5 de carga do bigforecaster — montar o
modelo do residuo deveria ser quase-trivial em cima daquele dataset.

---

## Recomendacao de priorizacao

**Ordem sugerida:** D → A → C → B.

| # | Trilho | Esforco remanescente | Retorno esperado | Justificativa |
|---|---|---|---|---|
| 1 | **D — residuo DESSEM** | **MENOR** | ALTO | Dados 100% prontos (5 anos), EDA feita, features ja existem em V4/V5 carga, **modelo proprio nao existe**. Reusa dataset bigforecaster. Saida = sinal D+1 acionavel (curtailment preventivo + hedge MMGD + ajuste bias). |
| 2 | A — area geoeletrica (geracao) | MEDIO | MEDIO | Para CARGA ja resolvido (A.4). Falta o crosswalk geracao→area (depende de `dim_usina` + mapping topologico que `area_mapping.py` nao tem). Reduz heterogeneidade de modelos per-estado. |
| 3 | C — granularidade barra DESSEM | MEDIO-ALTO | MEDIO | Surrogate subsistema ja existe (cmo_engine). Granularidade barra resolve corredor-curtailment (gap apontado em fase4_v3 e regional). Mas exige descida nao-trivial do alvo. |
| 4 | B — bottom-up | BAIXO (proximo) | ALTO ja capturado | 85% pronto. Falta deploy operacional dos REPORTs, nao mais pesquisa. Tratar como **producao**, nao recon. |

**Por que D primeiro:**
- Dataset pronto: 2,05M × 2,22M com chave `(din_referenciautc, cod_areacarga)`.
- Feature engineering ja resolvida em outro lugar (bigforecaster V4/V5 carga).
- Hipotese forte e nao testada: MMGD cresce mais rapido que previsao ONS →
  residuo positivo deve predizer curtailment ENE D+1 (mesma area-geo,
  mesma semi-hora). Hoje a fase4_v3 reprovou usar residuo **ex-post** —
  ninguem testou residuo **lag-D-1**.
- Saida operacional clara: "alerta vermelho NE-MA: ONS deve superestimar 800 MW
  amanha 14h → curt ENE provavel". Vendavel para o operador.

**Por que B nao e prioridade nova:** ja resolvido em pesquisa.
A acao remanescente nao e "fazer recon", e "subir API". Diferente.

---

## Worktrees varridos (resumo)

| Worktree | Branch | Conteudo relevante aos 4 trilhos |
|---|---|---|
| `brazilgrid-ulfor` (this) | `master` | `services/forecasting/area_mapping.py`, `experiments/`, `scripts/analysis/eda_*`, `docs/forecast/PLANO_FINAL.md`, `products/cmo_engine/` |
| `brazilgrid-bigforecaster` | `bigforecaster` | 100+ scripts: V3 multi-area, V4 Carga (vence ONS 3/4), V5 per-area, V10 UTE residual, V14.4 DESSEM anchor, V53 per-usina |
| `brazilgrid-bench-forecast` | `bench/forecast-2026` | Benchmark V3 hybrid + NGBoost (P10/P50/P90), 16 libs forecasting comparadas |
| `brazilgrid-eolica-serving` | `fix/carga-ddl-schema` | Inclui PR#33 mergeado (agg_eolica_previsao_conjunto), curtometro DDL fixes, monitoring |
| `brazilgrid-loop` | `feat/forecast-mega-loop-scaffold` | 49+ iteracoes documentadas — H1-H41 hipoteses sobre curtailment NE/SE; champions UlFor v3.3+CV5 oficial |
| `brazilgrid-perf-metrics-iec` | `feat/perf-metrics-iec` | EPI v3 batch 68 conjuntos + bigefi v0.1 backend (FastAPI 8 endpoints, port 8800) — adjacente |
| `brazilgrid-revisions` | `revisions-tracking` | Fase A/B/C revisions-tracking — capture + log dbt model + views razao_restricao |
| `brazilgrid-relatorio` | detached | Relatorio independente (irrelevante 4 trilhos) |
| `brazilgrid-ulfor-runner` | `feat/ulfor-headless-runner` | Runner headless — infra, nao recon |
| `brazilgrid` (legacy) | `feat/relatorio-independente-v2` | Frontend relatorio — irrelevante 4 trilhos |
| `bg-merge` | detached | Staging de merges — vazio para recon |

---

## Arquivos-chave para retomada (8 leituras prioritarias)

1. **`docs/forecast/PLANO_FINAL.md`** (35KB, 2026-05-22) — plano UlFor mestre (Fase 0-5, principios MLOps).
2. **`products/cmo_engine/docs/arquitetura.md`** — Camadas 1/2/3 DESSEM/DECOMP/NEWAVE surrogate.
3. **`experiments/curtailment_por_razao/REPORT_FINAL.md`** — D+1 ENE+CNF, ENE bate persistencia com ONS prev.
4. **`experiments/forecast_curtailment/REPORT_v3.md`** — fase4_v3, teto-de-dados D+1.
5. **`experiments/regional_forecaster/REPORT_REGIONAL.md`** — smoothing per-regiao, CatBoost campeao.
6. **`experiments/unified_forecaster/REPORT_UNIFIED.md`** — T1 smoothing / T2 MMGD→curt / T3 net load.
7. **`experiments/deep_dive/REPORT_DEEP_DIVE.md`** — V0 forense, val_lim quase-circular, WeatherNext.
8. **`scripts/analysis/eda_erro_previsao_carga.py`** — semente direta de Trilho D.

---

**Recon executado em ~1h, ZERO modificacao em CH ou codigo. 9 worktrees,
~50 arquivos amostrados, ~15 tabelas CH inspecionadas.**
