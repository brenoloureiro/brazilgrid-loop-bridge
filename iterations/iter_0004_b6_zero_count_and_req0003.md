---
alvo: sanity_check_b6_implementation + h6_SE_colapso_request
layer: meta
iter_num: 0004
type: tooling + investigation_handoff
data_utc: 2026-05-24T03:00:00Z
baseline_tipo: n/a (iter de tooling)
hypothesis: |
  H6 (do iter_0002): SE/v3 colapso +19.2pp NMAE com gap 7d e ganho aparente
  por adjacencia train-test. B6 (novo sanity check) deveria detectar
  automaticamente esse padrao via signal_collapse em lag features.
result_metric: |
  B6 implementado e validado. Em snapshot SE/v3 iter_0002 detectou:
  - curt_lag1: corr_train +0.252 -> corr_test -0.273 (sign flip, severity=high)
  - curt_lag7: corr_train +0.196 -> corr_test -0.146 (sign flip, severity=high)
  - curt_lag14: corr_train +0.159 -> corr_test +0.382 (severity=high, sem collapse)
  - ger_eolica_mwh: corr +0.277 -> -0.291 (sign flip, severity=high, collapse)
  Confirma H6 com evidencia mecanistica.
decision: |
  PROMOVE B6 a pipeline default (a partir de iter_0005).
  ENCAMINHA H6 para validacao no UlFor via req-0003 (test n>=60d).
sanity_checks_passed:
  zero_count_shift_self_validation: true   # caso sintetico canonico flagged corretamente
  signal_collapse_real_case_detection: true # SE/v3 lag sign flip detectado
budget_consumido_iter: 1.2
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/sanity_checks/zero_count_shift.py (B6 implementation)
  - loops/forecast-mega-loop/sanity_checks/__init__.py (canonical pipeline registration)
  - loops/forecast-mega-loop/templates/iter_prompt.md (6 checks default)
  - loops/forecast-mega-loop/scripts/b6_validation_synthetic.py
  - loops/forecast-mega-loop/outputs/iter_0004/zero_count_shift_validation.json
  - loops/forecast-mega-loop/outputs/iter_0004/b6_synthetic_validation.json
  - C:/Projetos/brazilgrid-ulfor/coordination/loop_requests.md (req-0003 appended,
      commit 9c1484e2 master ulfor)
---

# Iter 0004 — B6 zero_count_shift + req-0003

## B6 zero_count_shift implementado

Modulo `loops/forecast-mega-loop/sanity_checks/zero_count_shift.py` expoe:

```python
def check(X_train, X_test, y_train=None, y_test=None,
          threshold_ratio=3.0, zero_eps=1e-9) -> dict
```

Para cada coluna numerica computa:
- `train_zero_rate`, `test_zero_rate`: fracao de valores |x| < eps
- `ratio = test_zr / max(train_zr, eps)`
- `flag = ratio > threshold OR ratio < 1/threshold`
- `severity in {high, medium, low, none}`:
  - **high**: ratio > 10x (ou < 1/10x) E max_zr > 0.05
  - **medium**: ratio > threshold E max_zr > 0.01
  - **low**: ratio > threshold E max_zr > 0
- Se y_train/y_test fornecidos E feature flagged: computa `corr_train`,
  `corr_test`. Se sinal flipa OU |corr_test| < |corr_train|/2 com
  |corr_train| > 0.05: `signal_collapse = True`.

Pipeline default a partir de iter_0005 (registrado em
`sanity_checks/__init__.py` + `templates/iter_prompt.md`):
B1 leak → B2 PI → B3 holdout strict → B4 baselines → B5 dist_shift → **B6 zero_count_shift**.

## Validacao caso canonico (sintetico PDP coalesce(0) em test)

Script: `loops/forecast-mega-loop/scripts/b6_validation_synthetic.py`.

Cenario:
- train n=365 dias, pdp_prev = uniforme(5k, 15k), 1% gap_rate (~3 dias zero)
- test n=60 dias, primeiros 30 normais, ULTIMOS 30 = 0 (cron failure simulado)
- target = 0.3*pdp + ruido em train; em test, segundos 30 dias mantem curt real
  embora pdp=0 (= o que acontece com coalesce-0 envenenando)

Resultado B6 sobre `pdp_prev_solar_mwh`:
- train_zero_rate = 0.011
- test_zero_rate = 0.500
- ratio = 45.6
- severity = **high**
- corr_train = +0.545, corr_test = +0.171 (perdeu 69% magnitude)
- **signal_collapse = True (magnitude_collapse)**

Feature de controle (`clean_feat`, sem zeros): severity = none. **VALIDACAO PASSOU.**

## Validacao caso real (snapshots iter_0002 v1/v2/v3)

Script: `loops/forecast-mega-loop/sanity_checks/zero_count_shift.py main()`.
Output: `outputs/iter_0004/zero_count_shift_validation.json`.

Sumario:

| sub/ver | n_high | n_medium | n_signal_collapse | features high |
|---|---|---|---|---|
| NE/v1 | 0 | 4 | 2 | — |
| NE/v2 | 0 | 4 | 2 | — |
| NE/v3 | 0 | 4 | 2 | — |
| **SE/v1** | **4** | 1 | 3 | ger_eolica_mwh, curt_lag1, curt_lag7, curt_lag14 |
| **SE/v2** | **4** | 1 | 3 | (mesmas) |
| **SE/v3** | **4** | 1 | **4** | (mesmas) + pdp_prev_solar collapse |
| S/v1-v3 | 0 | 1 | 0 | — |
| N/v1-v3 | 0 | 3 | 2-3 | — |

**Achado-chave SE/v3** (signal_collapse com sign flip da correlacao):

| feature | train_zero_rate | test_zero_rate | corr_train | corr_test | collapse |
|---|---|---|---|---|---|
| ger_eolica_mwh | 0.390 | 0.000 | **+0.277** | **−0.291** | sign_flip |
| curt_lag1 | 0.137 | 0.000 | **+0.252** | **−0.273** | sign_flip |
| curt_lag7 | 0.139 | 0.000 | **+0.196** | **−0.146** | sign_flip |
| curt_lag14 | 0.139 | 0.000 | +0.159 | +0.382 | (sem flip) |
| pdp_prev_solar_mwh | 0.005 | 0.000 | +0.375 | −0.302 | sign_flip |

Mesmo padrao em **NE/v3** (com magnitude menor, sub menos catastrofico):
- curt_lag1: +0.664 → −0.229 (sign flip)
- curt_lag7: +0.415 → −0.341 (sign flip)

**Interpretacao**: a correlacao curt_lag(D-N) vs curt(D+1) e POSITIVA na maioria
da janela de treino (regimes persistentes de curtailment, "ontem teve, hoje vai
ter"). Na janela 2026-03-16 a 03-26 (test do replay), a correlacao FLIPA: dias
com lag alto sao seguidos de lag baixo. Pode ser:
1. Regime de transicao sazonal (fim de verao no SE → menor curtailment esperado
   apos picos de marco)
2. Janela muito curta (n=11) onde 1-2 outliers dominam
3. Operacao ONS especifica nesse periodo

Sem n>=60d nao da pra distinguir entre (1), (2), (3). Por isso req-0003 ao UlFor.

## req-0003 criado (H6 SE colapso)

Anexado a `C:/Projetos/brazilgrid-ulfor/coordination/loop_requests.md`
commit master ulfor `9c1484e2`. Conteudo:

- **type**: investigation
- **priority**: P1
- **summary**: SE/v3 +19.2pp NMAE strict; B6 do iter_0004 detectou sign-flip
  em curt_lag1/lag7 e ger_eolica. Investigar se e sazonalidade ou amostra.
- **acceptance_criteria**:
  - Re-rodar SE/v3 com test_n >= 60d E gap 7d
  - Reportar corr_train e corr_test das 7 features candidatas
  - Variante **v3b** sem curt_lag1/lag7/lag14 (so curt_rmean7)
  - Decisao: SE/v3 promovivel ou nao
- **expected_effort**: 30min-1h

Estado dos requests open agora:

| req | from_iter | prio | type | status |
|---|---|---|---|---|
| req-0001 | 0003 | P1 | investigation | OPEN |
| req-0002 | 0003 | P0 | feature_fix | OPEN |
| **req-0003** | **0004** | **P1** | **investigation** | **OPEN** |

UlFor ainda nao processou nenhum. O patch `autopilot_patch.md` (iter_0003)
adiciona passo 4.5 ao protocolo de retomada e ainda nao foi aplicado pelo
Breno. Sem o patch, UlFor nao le `coordination/loop_requests.md` automaticamente.

## Atualizacoes leaderboard + state

**leaderboard.md**: adicionada linha `meta sanity_check_B6` registrando
implementacao + validacao + integracao.

**state.json**:
- `iter_atual` = 4
- `alvo_ativo` = sanity_check_b6_implementation
- `hypotheses_verdict.H6_SE_colapso_strict` = EVIDENCIA_FORTE (B6 caught
  sign-flip), aguarda confirmacao n>=60d
- `sanity_checks_disponiveis` agora inclui zero_count_shift
- `sanity_checks_default_pipeline` = 6 checks (B1-B6)
- `open_requests` += req-0003
- `ulfor_session_sync.iter0004_fim_head` = 9c1484e2

## Proximas iters candidatas

### Aguardar UlFor processar (gating)

req-0001/0002/0003 estao OPEN. Apos o Breno aplicar autopilot_patch e o
proximo "retomar ulfor", UlFor le e processa em ordem (P0 → P1).
Resultado de req-0003 fecha capitulo H6.

### Rodaveis no loop sem dependencia

- **iter_0005 (recomendado)**: H7 — explorar **N como classificacao** em
  vez de regressao. N tem ymean baixo, variancia alta, rare events. Treinar
  classificador binario "haverah curt ENE/CNF em D+1?" pode ser mais util
  que regressor.
- **iter_0006**: H3 — investigar se PDP adiciona algo alem de "previsao de
  geracao" via `residuals(curt ~ gen) ~ PDP`. Loop pode rodar com dados
  existentes (n=11 e suficiente para correlacao residual).
- **iter_0007**: H5 — quantificar impacto de feat_termico stale. Comparar
  bakeoff com vs sem feat_termico no UlFor (request P2 metric_check).

### Hipoteses derivadas da iter_0004

- **H8 (nova)**: signal_collapse via sign-flip e prevalente alem de SE/v3?
  Aplicar B6 sistematicamente em mais combinacoes (sub, ver, ate em
  outras layers como geracao_d1, congestao_d1). Construir taxonomia de
  features-toxicas-em-regime-X.
- **H9 (nova)**: substituir curt_lag1/lag7 por features regime-aware
  (curt_rmean30, indicador_sazonalidade, fase_seca/umida) elimina o
  collapse sem perder skill?
