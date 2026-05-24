---
alvo: recon_delta_ulfor_post_4427a718
layer: meta
iter_num: 0024
type: recon_delta
data_utc: 2026-05-24T20:30:00Z
ulfor_head_inicio: 4427a718
ulfor_head_fim: 5d41d063
commits_absorvidos: 2
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.4
---

# Iter 0024 — RECON_DELTA UlFor (4427a718 → 5d41d063)

## Objetivo

Absorver 2 commits novos da sessao UlFor entre `4427a718` (HEAD na entrada
do iter_0023, fechamento sessao 14:15Z com CHAMPION_DECISION_MATRIX + 7
promovieis e "validation gap" explicito) e `5d41d063` (sprint autopilot
~14:25Z, validation gap FECHADO via single-fold 14d real + correcao do
veredito N). Janela ~10 min reais de UlFor (10:20-10:23 BRT). Conteudo
load-bearing: **fecha a validation_gap dos 7 promovieis** que era o
gatilho do req-0008 (P2 data_publish) que o loop ia emitir nesta iter
(H31_emergente). Confirma 4 acoes / refuta 1 / nao-testavel 1 / nova hipotese 1.

**Pre-empcao**: req-0008 NAO sera emitido — UlFor self-actionou. Atratividade
do loop emitir req cai a zero (UlFor self-publica resultados em
`FINDING_14D_REAL_VALIDATION.md` + atualiza CHAMPION_DECISION_MATRIX inline).

**Nota de escopo**: usuario passou range `ec0fd937..5d41d063` (8 commits)
mas iter_0023 ja absorveu 6 deles (ate `4427a718`). Janela real desta iter
e' `4427a718..5d41d063` (2 commits). HEAD UlFor real atual = `7cc3b604`
(2 commits adicionais 756457c7+7cc3b604 fora de escopo desta iter, ficam
para iter_0025).

## PHASE A — Commits inspecionados

Em ordem cronologica:

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `99af14b7` | exp — validacao 14d real refuta promoves S/N de MATRIX | **FECHA validation_gap** dos 7 promovieis. Single-fold test 2026-03-24..2026-05-21. Adiciona `FINDING_14D_REAL_VALIDATION.md` + flag `--out-suffix` em bakeoff_d1.py + 3 parquets val14d. | PHASE B leaderboard + state |
| 2 | `5d41d063` | exp — correcao N (ridge+h22 NAO refutado em 14d real) | **CORRIGE veredito iter anterior**: comparacao N estava vs `lgbm+full` (que melhor 14d mas perde CV), nao vs `lr+full` status quo. Vs status quo, ridge+h22 ganha em ambas dimensoes. | PHASE B leaderboard + state |

### Interpretacao detalhada por commit

**`99af14b7` — Single-fold 14d real dos 7 promovieis**

Sprint autopilot envelope-safe (proxima acao #2 do checkpoint 14:15Z, sem
PARAR-E-PERGUNTAR porque nao toca producao — so leitura via bakeoff
`--cv-folds 1`). Comando canonico:

```
bakeoff_d1 --feature-set {full|h22_per_fold|h22_model_aware} --cv-folds 1 --no-mlflow
```

Test=2026-03-24..2026-05-21 (n=59 = ultimo fold do walk-forward CV
5x60d, mesma janela do iter_0017 NE bias_correction validation).

Resultado por sub × modelo × feature-set (16 cells × 3 fs = 48 runs;
NMAE / R²):

| Sub | Modelo  | full (status quo) | h22_per_fold     | h22_model_aware  |
|-----|---------|-------------------|------------------|------------------|
| NE  | ridge   | **31.1 / +0.489** | **30.9 / +0.478**| **30.9 / +0.478**|
| NE  | lr      | 33.1 / +0.455     | 31.0 / +0.475    | 31.0 / +0.475    |
| NE  | lgbm    | 37.8 / +0.326     | 34.4 / +0.427    | 34.4 / +0.427    |
| NE  | xgb     | 35.1 / +0.374     | 36.0 / +0.375    | 36.0 / +0.375    |
| SE  | ridge   | 44.7 / +0.365     | 45.7 / +0.311    | 43.9 / +0.393    |
| SE  | lr      | 45.2 / +0.306     | 44.5 / +0.346    | 43.2 / +0.378    |
| SE  | lgbm    | 44.4 / +0.382     | 43.9 / +0.399    | **42.7 / +0.428**|
| SE  | xgb     | **44.2 / +0.380** | 46.7 / +0.340    | 42.9 / +0.413    |
| S   | ridge   | 96.6 / +0.095     | 96.8 / +0.098    | 98.1 / +0.097    |
| S   | lr      | **93.6 / +0.206** | **94.4 / +0.194**| **95.2 / +0.205**|
| S   | lgbm    | 118.7 / -0.101    | 120.1 / -0.161   | 120.1 / -0.126   |
| N   | ridge   | 73.8 / +0.102     | 74.0 / +0.105    | 74.1 / +0.103    |
| N   | lr      | 75.2 / +0.062     | 75.2 / +0.049    | 74.9 / +0.089    |
| N   | lgbm    | **68.1 / +0.139** | 73.7 / +0.035    | 70.3 / +0.099    |
| N   | xgb     | 71.1 / +0.022     | **69.4 / +0.053**| **69.2 / +0.066**|

**Veredito autopilot (pre-correcao iter 2):**

| Acao_matriz                              | CV 5x60d         | 14d_real         | Recomendacao   |
|------------------------------------------|------------------|------------------|----------------|
| NE: lr+full → ridge+h22_per_fold         | -8.8pp NMAE      | tie (~-0.2pp)    | **PROMOVER**   |
| NE: bias H14-C → H14-G(w=14,k=1)         | strict Pareto    | nao testavel*    | **PROMOVER**   |
| SE: lr+full → lr+h22_model_aware         | tie              | -2.0pp NMAE      | **PROMOVER**   |
| SE: lr+full → ridge+h22_per_fold         | -1.8pp NMAE/+R²  | -0.5pp NMAE      | promover marginal |
| S: lr+full → lr+h22_per_fold             | -2.9pp NMAE      | **+0.8pp (pior)**| **NAO PROMOVER** |
| N: lr+full → ridge+h22_per_fold          | -23.7pp NMAE     | +1.3pp (vs lgbm) | **NAO PROMOVER** (sera revisado em commit 2) |
| N: bias H14-B → H14-G(w=60,k=1)          | trade-off        | nao testavel*    | decidir Breno  |

`*bias correction nao e' testavel single-fold; aplica POS-prediction baseado em residuo rolling.`

**Achados novos (NAO promovieis — falta CV win):**
- SE em 14d real prefere `lgbm+h22_MA` (42.7%) vs `xgb+full` (44.2%) →
  regime atual mais nao-linear. CV nao testou lgbm × h22_MA.
- N em 14d real prefere `lgbm+full` (68.1%) vs `lr+full` (75.2%) → CV preferia LR.

UlFor explicito: NAO promover sem CV win primeiro. Reabrir CV para lgbm em
SE+N seria proximo sprint envelope-safe.

Artefatos:
- `FINDING_14D_REAL_VALIDATION.md` (novo, 157 lines)
- `bakeoff_d1.py` adiciona `--out-suffix` (de parallel agent, agora committado)
- `outputs/cv_summary_per_fold_{full,h22_per_fold,h22_model_aware}_val14d.parquet`
- `CHAMPION_DECISION_MATRIX.md` secao "Validation gap" reescrita → FECHADO

**Backward-compat**: nenhuma mudanca em loader.py / Registry / champions.

**`5d41d063` — Correcao N (ridge+h22 NAO foi refutado em 14d real)**

Revisao do FINDING_14D_REAL_VALIDATION apos checar parquets CV 5x60d
**completos incluindo LGBM** (antes ausente da matrix). Erro do iter
anterior (99af14b7): classificou N ridge+h22 como REFUTADO porque perdia
para `lgbm+full` em 14d real (+1.3pp). **Comparacao errada**: deve ser vs
status quo (`lr+full`), nao vs alternativa explorada.

Tabela corrigida N:

| Model+set      | NMAE_CV | R²_CV  | NMAE_14d | R²_14d |
|----------------|---------|--------|----------|--------|
| lr+full (atual)| 110.1   | -0.314 | 75.2     | +0.062 |
| ridge+h22      |  86.4   | +0.200 | 74.0     | +0.105 |
| lgbm+full      |  96.4   | +0.003 | **68.1** | +0.139 |
| lgbm+h22_MA    |  93.1   | +0.061 | 70.3     | +0.099 |
| lgbm+h22       |  94.6   | +0.030 | 73.7     | +0.035 |

Ridge+h22 N ganha em **ambas dimensoes vs status quo** (-23.7pp CV /
-1.2pp 14d). LGBM+full e' best 14d mas viola "so promover apos CV win"
(perde 10pp para ridge+h22 em CV 5x60d).

**Correcoes aplicadas:**
- N agora marcado **PROMOVER** (era NAO PROMOVER)
- Adicionada linha LGBM+full como "achado novo NAO promovivel"
- Recomendacao final tem **4 acoes** (era 3): NE + SE + N + manter S
- S permanece como unica acao formalmente refutada

Artefatos:
- `FINDING_14D_REAL_VALIDATION.md` (revisado, +64 lines / -24)
- `CHAMPION_DECISION_MATRIX.md` (revisado, +14 lines / -6)

### Sintese final (post-validation)

Promote enxuto recomendado pelo UlFor autopilot (alta-confianca, CV ∧ 14d
real ambos validados):

1. **NE**: `lr+full` → `ridge+h22_per_fold` + bias `H14-C` → `H14-G(w=14,k=1)`. Strict Pareto.
2. **SE**: `lr+full` → `lr+h22_model_aware`. Menos features (-11), ganho 14d (-2.0pp), equivalencia CV. Mitiga cond_num 2.5e17.
3. **N**: `lr+full` → `ridge+h22_per_fold` (-23.7pp CV / -1.2pp 14d vs status quo). Bias H14-B vs H14-G a definir Breno.
4. **S**: **MANTER STATUS QUO** `lr+full`. CV gain -2.9pp refutado em 14d real (+0.8pp pior).

Achados novos (NAO promoviveis — precisam CV win):
- SE `lgbm+h22_MA` (14d 42.7%)
- N `lgbm+full` (14d 68.1%)

## PHASE B — Atualizacoes do loop

### state.json `ulfor_session_sync`

Adicionado bloco `iter0024_inicio_head=4427a718`, `iter0024_fim_head=5d41d063`,
lista de 2 commits absorvidos, `delta_resumo_iter0024` documentando:
- Champions em producao **INALTERADOS** (Registry intocado);
- **3 acoes promovieis CONFIRMADAS em CV ∧ 14d real**: NE ridge+h22+H14-G,
  SE lr+h22_MA, N ridge+h22 (corrigido em 5d41d063);
- **1 refutacao formal**: S lr+h22 (refutado em 14d real apesar de CV gain);
- **1 ainda indecidivel single-fold**: N bias H14-B → H14-G(60,k=1);
- **2 achados novos** com CV-gap aberto: SE lgbm+h22_MA, N lgbm+full;
- **Pre-empcao req-0008**: H31_emergente que loop ia emitir esta iter
  foi self-actionada pelo UlFor → loop NAO emite;
- 0 novos req recebidos / 0 fechados extras.

`alvo_ativo` → `recon_delta_ulfor_post_4427a718`, `iter_atual` → 24.

### hypotheses_queue.md

**Nenhuma H do loop fechada formalmente** (nenhuma colisao direta com
nosso queue ativo — H22/H24/H30 nosso seguem queued). Atualizacoes:

- **H31_emergente** (nao formalizada, candidata em iter_0021/0023):
  status_change candidata → **PRE-EMPTED** (UlFor self-actionou em
  99af14b7). Sem necessidade de queue entry retroativa.
- **H8 (intercambio drop em N)**: tabela 14d real confirma N segue com
  `val_net_*` (drop list H22_MA exclui N), consistente com iter_0021. Sem mudanca.

### leaderboard.md

Atualizado header `**Iter 0024 (RECON_DELTA)**`. Secao "Candidatos
sucessores" reescrita: cada candidato agora rotulado com veredito 14d
real (**CONFIRMADO** / **REFUTADO** / **MARGINAL** / **NOVO ACHADO sem CV**).

Pendencia #3 do leaderboard ("Holdout 14d real para candidatos h22_per_fold
NE+SE") → **FECHADA** por commits 99af14b7+5d41d063.

### open_requests

Continua vazio. **req-0008 (P2 data_publish) que estava sendo cogitado
em iter_0023 NAO sera emitido** — UlFor self-actionou (publicou validation
parquets + reescreveu matrix inline). Loop reduz superficie de
coordenacao 1 req.

## PHASE C — Handoff

### Hipotese candidata para iter_0025

Pelo planner_config + atualizacao desta iter:

- **(A-recomendado) H27 (P3 ~1h)** — P50 quantile como point estimate
  substituto via LGBM objective swap. iter_0014 bonus mostrou P50 BATE
  LGB-mean em magnitude em N (-18%), S (-14%), SE (-1.6%). Custo zero,
  ganho transversal garantido N+S. Ortogonal a frente H22_MA/H14-G UlFor.
  **Sem dep externa**. Atratividade SUBE pos-iter_0024 porque o achado
  novo "SE+N preferem LGBM em 14d real" sugere objective swap pode ser
  benefico transversal a sub.

- **(B) H30 (P3 ~1h)** — pdp_residual em Ridge_alpha10 CV; script base
  `scripts/h21_pdp_residual_cv.py` ja existe (criado em recon iter_0019).
  Resposta natural ao "H21 REFUTADO em OLS" — testa se Ridge L2 redistribui
  pesos diferente.

- **(C) H22 nosso (P3 ~1h)** — GBDT vs OLS gap; carrega lesson H23 ulfor
  (PI com modelo final). Atratividade marginalmente subiu pos-iter_0024
  (achado novo "SE+N preferem LGBM"). PI medida com LGBM podia revelar
  features carregando interacoes nao-lineares mascaradas em LR/Ridge.

- **(D) Acompanhar UlFor para sprint `lgbm × h22_MA × CV 5x60d`** —
  ortogonal mas baixa-acao pro loop. Aguardar UlFor publicar e absorver
  em proxima recon. **Nao gera iter ativa**.

### Sequencia recomendada

1. **iter_0025**: H27 (LGBM quantile P50 substituto) — custo zero, ganho
   transversal, ortogonal a UlFor multi-agente. Sem dep externa, sem
   colisao prevista. Cobre N+S que o validation gap mostrou serem
   resistentes a feature engineering.
2. **iter_0026**: recon_delta UlFor (capturar 756457c7+7cc3b604 + qualquer
   sprint novo) OU H30 (Ridge pdp residual CV) dependendo de UlFor.

### Riscos detectados

- **HEAD UlFor real (7cc3b604) > escopo desta iter (5d41d063)**: 2 commits
  novos ja' aguardam absorcao em iter_0025 RECON_DELTA. Sao
  `756457c7` (checkpoint 14:25Z) + `7cc3b604` (ADDENDUM val14d z-score +
  lgbm CV ja existe). Provavelmente formaliza investigacao do
  "lgbm bate em 14d real" sugestionada nesta iter.
- **Regime UlFor multi-agente continua ativo**: freq recon loop deve
  permanecer 30-60min quando UlFor "online" (vs 2-4h offline).
- **CHAMPION_DECISION_MATRIX discrepancia "Champion ATUAL"** flaggada em
  iter_0023 (NE+N listados como `lr+full` na matriz, vs ridge no Registry):
  NAO investigada nesta iter; segue para iter_0025+. Hipotese provavel (B)
  bug de documentacao continua mais plausivel — esta iter nao mexeu em
  loader.py.
- **Push origin UlFor pendente** (master 14+ ahead origin agora). Sem
  impacto direto no loop (read-only via git show).

### Status H_loop pos-iter_0024

| H | status | mudanca esta iter |
|---|--------|---|
| H27 | queued (P3) | atratividade SUBIU — promovida a (A) recomendado iter_0025 |
| H30 | queued (P3) | inalterado — fica (B) |
| H22 nosso | queued (P3) | atratividade marginalmente subiu (LGBM ganha 14d real) |
| H25, H26, H28 | queued (P3) | inalterado |
| H8 nosso | queued (P3) | inalterado |
| H18 | blocked-acao-Breno-trivializa | inalterado (MLflow CF Access pendente Breno) |
| H31_emergente | nao-formalizada → PRE-EMPTED | UlFor self-actionou em 99af14b7 |

## Sanity checks

`type=recon_delta` — sanity_checks_required = []. NAO aplicavel.

## Budget

- Estimado: 0.4h
- Real: ~0.4h (leitura 2 commits + 2 docs + sintese + corretividade do veredito N)
- Acumulado iter (24): ~0.4h / cap 2.5h por iter / cap 16h diario
