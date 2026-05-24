---
alvo: h13_persist_d7_baseline_aux
layer: meta
iter_num: 0029
type: hypothesis_test (verdict=CONFIRMADO_DISPLAY_REFUTADO_REGIME_CLAIM)
data_utc: 2026-05-25T00:30:00Z
hypothesis: |
  H13 (P3, queued desde iter_0001, layer=meta, type=metric):
  "Persist D-7 alem de persist D-1 como baseline secundario. Iter 0002 ja
  calcula persist_d7. Promover para leaderboard como baseline auxiliar —
  em alguns regimes (S) persist_d7 vence persist_d1. Mudanca de display +
  interpretacao, nao re-treino."

  Decomposicao em sub-claims:
    C1 (display): persist_d7 deve estar na tabela de baselines do leaderboard
    C2 (regime claim): persist_d7 vence persist_d1 em alguns regimes, S especificamente
baseline_tipo: |
  Tabela ja' existente no leaderboard secao "Baselines" com persist_d1,
  persist_d7, ma7 por sub (NE/SE/S/N). Fonte: UlFor cv_summary_mean_full.parquet
  (CV 5x60d, gap 7d, feature_set=full). H13 e' display + interpretacao,
  nao re-treino, nao novo CV.
baseline_metric: |
  CV 5x60d MAE persist_d7 vs persist_d1 (canonico):
    NE: d7=58 850 vs d1=33 183 (+77.3%)
    SE: d7=11 073 vs d1= 9 273 (+19.4%)
    S:  d7= 1 555 vs d1= 1 230 (+26.4%)
    N:  d7=   575 vs d1=   508 (+13.1%)
result_metric: |
  C1 (display): ATENDIDO trivialmente (persist_d7 presente desde iter_0007).
  C2 (regime claim "S"): REFUTADO — persist_d1 vence em 4/4 subs no agregado
  e em 19/20 per-fold cells. Unica inversao: N fold 0 (d7=441.6 vs d1=592.4,
  −25.4%), regime temporal antigo do walk-forward, nao S.
  Origem provavel da premissa: replay iter_0002 n=11 mostrou d7 vence d1
  em N (provavel erro de transcricao no detail original — autor citou S,
  dado mostra N).
decision: RETÉM (persist_d7 permanece como diagnostico, agora com nota)
sanity_checks_passed:
  permutation_importance: skipped
  holdout_temporal_strict: true
  leak_detection: true
  baseline_compare: true
  distribution_shift: true
  zero_count: true
budget_consumido_iter: 0.3
custo_estimado_usd: 0.0
---

# Iter 0029 — H13 persist_d7 baseline aux

## Hipótese

H13 do queue (P3, layer=meta): "Promover persist_d7 ao leaderboard como
baseline auxiliar — em alguns regimes (S) persist_d7 vence persist_d1.
Mudanca de display + interpretacao, nao re-treino."

**O que esperavamos ver**: persist_d7 ainda nao no leaderboard; ou um caso
concreto onde persist_d7 < persist_d1 que justificasse promover como
candidato.

**O que motivou a hipotese**: iter_0002 baseline_compare.json mostrou
`persist_d7` calculado mas talvez nao exposto no leaderboard. A claim
"S vence" sugeria que a sub mais zerada/sazonal teria padrao semanal.

## Como foi rodado

Sem retrain, sem novo CV. Pura consulta de parquets ja' existentes:

1. **Source 1**: `cv_summary_mean_full.parquet` (UlFor commit 6b21ffdf,
   CV 5x60d gap 7d feature_set=full) — agregado mean over 5 folds.
2. **Source 2**: `cv_summary_per_fold_full.parquet` — per-fold breakdown.
3. **Source 3**: `outputs/iter_0002/baseline_compare.json` — replay loop
   antigo n=11 (deprecated mas relevante para auditar origem da premissa).

Script de analise: inline Python via Bash tool, sem dependencias novas
(pandas + json). Output: `outputs/iter_0029/persist_d7_baseline_aux/`
contendo `analise.md`, `persist_d7_metrics.json`, `sanity_checks.json`.

## Resultado

### C1 — Display: ATENDIDO (no-op)

`leaderboard.md` ja' continha 4 linhas persist_d7 (uma por sub) na secao
`## Baselines` desde iter_0007 (champions UlFor CV). Nada a adicionar.

### C2 — Regime claim: REFUTADO

**Agregado CV 5x60d:** persist_d1 vence persist_d7 em 4/4 subs:

| sub | persist_d1 MAE | persist_d7 MAE | delta_d7_vs_d1 | winner |
|---|---:|---:|---:|---|
| NE | 33 183 | 58 850 | +77.3% | persist_d1 |
| SE |  9 273 | 11 073 | +19.4% | persist_d1 |
| S  |  1 230 |  1 555 | **+26.4%** | persist_d1 |
| N  |    508 |    575 | +13.1% | persist_d1 |

**S e' especificamente refutado**: e' a segunda maior margem onde
persist_d1 vence (atras apenas de NE). Sub-claim direto e' falso.

**Per-fold (20 cells):**
- NE 0/5, SE 0/5, S 0/5, N **1/5 (fold 0)**.
- N fold 0: d7=441.6 vs d1=592.4 (−25.4%) — janela temporal antiga do
  walk-forward. Consistente com regime sazonal antigo em N (seca-2025Q3
  domina folds tardios, ja' identificado em iter_0018 H14 bias correction).

**Replay iter_0002 (n=11, deprecated):** unico contexto onde persist_d7
vence — e' **em N** (d7=248 vs d1=346, −28%), nao em S. Provavel
erro de transcricao no detail original da H13.

### Sanity checks (6/6 endereçados)

| check | status | rationale |
|---|---|---|
| leak | passed | persist_d7 = lag-7 do target, sem cross-fold leak |
| perm | skipped | persist nao tem features para permutar; N/A |
| holdout_strict | passed | walk-forward 5x60d gap=7d (UlFor) |
| baseline | passed | este check **e** a propria analise — passou |
| dist_shift | passed | per-fold variance flagged (NE RSD ~42%) mas sinal estavel; N fold 0 inversao documentada |
| zero_count | passed | S zero-structural propagado por ambos d1 e d7 simetricamente |

5 passed + 1 skipped (N/A justificado). Detalhes em
`outputs/iter_0029/persist_d7_baseline_aux/sanity_checks.json`.

## Decisão

**RETÉM** persist_d7 no leaderboard como **baseline diagnostico**, com
nota explicita destruindo a expectativa H13 original. Justificativa:

1. Display ja' satisfeito; remocao seria regressao (perder diagnostico
   de auto-correlacao semanal vs diaria).
2. **Insight estrutural util**: curtailment tem persistencia diaria forte
   e ciclo semanal fraco em **todos** os subsistemas. Audiencia operacional
   tipicamente assume padroes semanais (carga industrial, fim-de-semana);
   dado diz "nao para curtailment". Vale destacar.
3. N fold 0 anomalia documentada como caso isolado — ja' coberta por H14
   bias correction (iter_0018).

**Hipotese derivada criada**: NENHUMA. Achado de N fold 0 ja' coberto por
hipoteses fechadas (H14 bias correction, iter_0018 N seca-2025Q3 sazonal).

## Próximo passo

H13 fechada com clean verdict. Voltar ao loop normal:

- Proxima hipotese queued: revisar `hypotheses_queue.md` por proxima P0/P1/P2.
  P3+ existem em volume; H16 (P2, status=done iter_handled=0009), H14
  (P2, status=blocked req-0006) sao referenceadas mas nao re-rodaveis no
  loop.
- Monitorar UlFor para novo RECON_DELTA: HEAD UlFor atual = `27152e16`
  (iter_0028 fim). Se ha commits novos, proxima iter pode ser
  RECON_DELTA_ulfor_post_27152e16.

## Artefatos persistidos

- `outputs/iter_0029/persist_d7_baseline_aux/analise.md` (analise narrativa)
- `outputs/iter_0029/persist_d7_baseline_aux/persist_d7_metrics.json`
  (CV agregado + per-fold + replay n=11)
- `outputs/iter_0029/persist_d7_baseline_aux/sanity_checks.json` (6 checks)
- `leaderboard.md` (header + nota persist_d7 + linha historico iter)
- `hypotheses_queue.md` (H13 status=done, iter_handled=0029, closure_summary)
- `state.json` (iter_atual=29, alvo_ativo, hypotheses_verdict.H13)
