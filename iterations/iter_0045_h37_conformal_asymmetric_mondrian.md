---
alvo: ne_n_d1_quantile_calibrated_v2
layer: curtailment
iter_num: 0045
type: hypothesis_test (verdict=CONFIRMADO_ASYM_ONLY)
data_utc: 2026-05-26T06:00:00Z
hypothesis: |
  H37 (P3, queued desde iter_0037 — derivada H26 INDETERMINADO_NE +
  bonus CONFIRMADO_SE_S + OVER_N): "CQR-asymmetric + Mondrian conformal
  por regime fecham NE+N gap H26".

  H26 mostrou que conformal symmetric (Romano CQR 2019) com q_alpha
  global:
    - resolve SE+S (cov_cal 76.6% / 79.5% strict, 3/3 cells)
    - fica BORDERLINE em NE (cov 74.7%, 0.3pp do limite strict; ratio 2.03x)
    - OVER-COBRE em N (cov 90.8%, banda larga demais)
  Causa raiz: 1 q_alpha global empurra ambos os lados simetricamente; em
  N o score e' dominado por outliers da cauda alta (37% dos dias com
  curt~0) inflando q10 desnecessariamente; em NE a inflacao e'
  insuficiente em folds de regime shift (KS p<0.0001 iter_0012).

  Fix duplo (custo +30 LoC sobre h26_conformal, sem retreinar):

    A) CQR-ASYMMETRIC (Romano variant):
       s_low_i  = max(0, q10_iv_i - y_iv_i)
       s_high_i = max(0, y_iv_i - q90_iv_i)
       q_low  = quantile_finite_sample(s_low,  1 - alpha/2)
       q_high = quantile_finite_sample(s_high, 1 - alpha/2)
       Banda: [q10 - q_low, q90 + q_high]

    B) MONDRIAN CONFORMAL POR REGIME:
       p50_thresh = median(y_train_inner)            # nao usa inner_val
       Bucketing inner_val por y_iv vs p50_thresh; bucketing test por
       q50_te (predicao da mediana do modelo train_inner) -- D-1-safe.
       Em cada bucket: q_bucket = quantile_finite_sample(s_sym, 1-alpha).
       Shrinkage inverse-variance:
         precision_X = n_X / max(var(s_X), eps)
         w_bucket = precision_bucket / (precision_bucket + precision_global)
         q_used = w_bucket * q_bucket + (1 - w_bucket) * q_global
       Aplicacao per-ponto pelo bucket previsto em q50_te.

  ACEITACAO (literal do queue):
    CQR-asymmetric CONFIRMADO se
      cov_band_80_cal NE in [75%, 85%] em >=2/3 NE cells (iv=30)
      AND cov_band_80_cal N in [75%, 90%] (corrige over-coverage 90.8%)
    Mondrian CONFIRMADO se
      cov_band_80_cal NE in [75%, 85%] em >=2/3 NE cells (iv=30)
      AND fold heterogeneity reduzida (std fold-a-fold <= 0.10)

baseline_tipo: |
  Triplo embedded por fold/iv:
    (sym) = H26 iter_0037 (CQR symmetric, mesmo script bit-exato).
    (asym) = H37 variante A.
    (mond) = H37 variante B.
  Persist_d1 implicito por reuso do mesmo split (calculado em H26).

baseline_metric: |
  H26 iter_0037 NE iv=30 cov_cal_mean = 74.7%, ratio_inner = 2.03x (3/3
  loose, 1/3 strict). N cov_cal_mean = 90.8% (over-coverage, 0/3 strict).

result_metric: |
  ITER_0045 H37 NE+N iv=30:
    ASYM:
      NE cells in [75, 85] = 2/3   (cov_mean 75.5%) PASS_NE
      N  cells in [75, 90] = 3/3   (cov_mean 86.4%) PASS_N
      Veredito A: CONFIRMADO.
    MONDRIAN:
      NE cells in [75, 85] = 1/3   (cov_mean 75.2%)
      NE std fold-a-fold mean (per cell) = 20.9% (>> 10pp threshold)
      Veredito B: REFUTADO (cov + heterogeneity ambos falham).

decision: CONFIRMADO_ASYM_ONLY (RETEM_ASYM_COMO_DELIVERABLE_REPLAY) -- mondrian NAO viavel sem mais dados por bucket
sanity_checks_passed:
  permutation_importance: skipped_inherited  # mesmas features iter_0002 (PI iter_0010 H3)
  holdout_temporal_strict: true              # 5 folds walk-forward, gap=7d, inner_val temporalmente antes do test
  leak_detection: skipped_inherited           # features iter_0002 leak-free iter_0010; bucketing usa q50_te (modelo train_inner), nao y_te
  baseline_compare: true                      # triple baseline embedded (sym H26 + asym + mond)
  distribution_shift: annotated_reuse         # iter_0012 KS p<0.0001 e' a causa raiz que motivou H37 (per-regime)
budget_consumido_iter: 0.7
custo_estimado_usd: 0.0
---

# Iter 0045 — H37 CQR-asymmetric + Mondrian conformal (NE+N gap H26)

## Hipotese

H37 (P3, derivada H26 iter_0037): testar se duas variantes de conformal
prediction com custo desprezivel (sem retreinar) fecham as 2 falhas
conhecidas do CQR symmetric do H26:

1. **NE borderline** (cov_cal 74.7% iv=30, 0.3pp do limite strict 75%,
   width_ratio_cal_vs_inner 2.03x, 1/3 cells strict)
2. **N over-coverage** (cov_cal 90.8% iv=30, 0/3 strict, banda larga
   demais — q_alpha global e' dominado pelos outliers da cauda alta)

As duas variantes:

- **A) CQR-asymmetric** (Romano variant): scores separados por cauda.
  Em N, esperado q_low pequeno (poucos overshoots por baixo, 37% dos
  dias y~0) -> banda nao infla desnecessariamente abaixo de q10. Em NE,
  q_high mantem a inflacao da cauda alta nos folds 2-3.

- **B) Mondrian per-regime**: particiona inner_val por threshold P50 do
  train_inner; q por bucket; shrinkage inverse-variance com global para
  estabilizar buckets pequenos (15d). Bucket no test escolhido por
  q50_te (predicao da mediana, D-1-safe). Esperado: heterogeneidade
  fold-a-fold reduzida em NE (regime shift KS p<0.0001 iter_0012 e' por
  dia, nao por modelo).

## Como foi rodado

Script: `scripts/h37_conformal_asymmetric_mondrian.py`
Source: `outputs/iter_0002/runs/{NE,SE,S,N}/{v1,v2,v3}/features.parquet`
(12 cells, mesma fonte H26).

Protocolo bit-exato H26 + 2 calculos adicionais por fold:
- 5 folds walk-forward, test_window=60d, gap=7d
- LGBM quantile alphas {0.1, 0.5, 0.9}, defaults H26
- inner_val_days = 30 (primario) e 60 (diagnostico)
- ALPHA_TARGET = 0.2 (banda 80% nominal)
- finite-sample correction Lei et al. 2018:
  k = ceil((n+1) * (1-alpha)) / n, method='higher'
- Para asym: 1-alpha/2 = 0.9 (cada cauda 10% nominal)

Mondrian:
- p50_thresh do train_inner (nao inner_val: evita auto-seleciona-vies)
- bucketing test via q50_te (modelo train_inner sobre X_te) — D-1-safe
- shrinkage: `w_bucket = (n_b/var_b) / (n_b/var_b + n_g/var_g)`
- bucket-min = 5 (caso menor, w_bucket=0 = full global)

NAO modifica producao, ulfor, CH; nao instala dependencias.

Comando: `python scripts/h37_conformal_asymmetric_mondrian.py`
Wall clock: ~3 min (12 cells x 5 folds x 2 iv x 3 quantile fits).

## Resultado

### Per-sub iv=30 (medias entre cells, em pp)

| sub | sym cov | asym cov | mond cov | sym sharp | asym sharp | mond sharp |
|---|---|---|---|---|---|---|
| NE | 74.7% | **75.5%** | 75.2% | 113.4k | 123.5k | 112.4k |
| SE | 76.6% | 80.3% | 75.2% | 21.8k | 26.2k | 21.4k |
| S  | 79.5% | 81.8% | 79.5% | 2.5k | 3.8k | 2.5k |
| N  | 90.8% | **86.4%** | 91.3% | 1.3k | 1.8k | 1.4k |

ASYM resolve EXATAMENTE as duas falhas:
- NE 74.7 -> 75.5% (cruza para o lado certo do limite strict)
- N  90.8 -> 86.4% (cai de over-coverage para faixa aceita [75, 90])
- SE +3.7pp e S +2.3pp como bonus (continuam dentro do range)

MONDRIAN deslocamento pequeno (deslocamento medio < 1pp em todas as 4
subs) — shrinkage inverse-variance puxa q_used quase para q_global na
maioria dos folds (w_bucket low na cauda alta em ~0.4, w_bucket high
~0.4 — calibra metade do peso para o global).

### Verdict por criterio (iv=30, alvo NE+N)

| variante | NE in [75,85] | NE cov mean | N in [75,90] | N cov mean | std fold mean | PASS? |
|---|---|---|---|---|---|---|
| ASYM | 2/3 | 75.5% | 3/3 | 86.4% | n/a | **PASS** (ambos gates) |
| MOND | 1/3 | 75.2% | n/a | n/a | 20.9% | FAIL (cov + heterogeneity) |

ASYM passa AMBOS os gates da aceitacao:
- NE: 2/3 cells in [75, 85] (limiar de aceitacao = 2/3) — PASS
- N: 3/3 cells in [75, 90] (corrige over-coverage 90.8% -> 86.4%) — PASS

MONDRIAN falha em ambos os gates:
- NE: 1/3 cells in [75, 85] (limiar 2/3) — FAIL
- std fold-a-fold mean = 20.9% (limiar 10pp) — FAIL por 2x

### Por que ASYM venceu N

N tem 37% dias com curt~0 (outliers de zero). Em CQR symmetric, o score
`max(q10_iv - y, y - q90_iv)` retorna `q10_iv - y` quando y=0 e q10_iv>0
— um overshoot por BAIXO grande, ainda que conceitualmente y nao pode
ser negativo. Esse score contamina q_alpha global, que infla tanto
q10 quanto q90.

Em CQR-asymmetric:
- s_low_i = max(0, q10_iv - y) = max(0, q10_iv) na maioria dos
  zero-overshoots, mas tipicamente q10_iv ~ 0 em folds com sub-N (LGBM
  quantile aprende a posicionar q10 perto de zero). Resultado: q_low
  tipicamente ~100 MWh nos folds N (vs q_alpha symmetric ~660 MWh).
- s_high_i = max(0, y - q90_iv) e' onde o sinal real esta'.

Numericamente (medias dos 5 folds N/v1):
  ASYM q_low ~107, q_high ~909
  Symmetric q_alpha ~663

Banda asymmetric mais "puxada para cima" — perde alguns zeros que sym
captura pelo lado de baixo, mas ainda fica dentro de [75, 90].

### Por que MONDRIAN falhou

Hipotese inicial: regime shift no test (KS p<0.0001 iter_0012) e' por
dia, particionar por regime estabilizaria. Resultado empirico:

(1) Bucketing NAO transferiu bem do train_inner para o test em NE:
    fold 3 NE/v1 tem cov_band 0.52 nas 3 variantes — falha simultanea.
    Causa: a divisao por P50(train_inner) e' arbitraria quando o
    test inteiro vem de regime novo (low_curt residual concentrado em
    novo nivel). w_bucket fica baixo (~0.40-0.45 cauda alta, 0 cauda
    baixa quando inner_val nao tem dias com y<p50) e q_used ~ q_global.

(2) Shrinkage inverse-variance e' conservador demais para CV 30d/60d:
    n_global = 30 e n_bucket = 15 da' w_bucket ~ 0.5 mesmo quando
    var_bucket << var_global. Mondrian colapsa em sym + ruido de
    bucketing — pior dos dois mundos.

(3) std fold-a-fold mean = 20.9% em NE (limiar 10pp). NE/v2 std = 22%,
    NE/v3 std = 22% — fold 3 colapsa (cov 0.38) em todas as variantes
    inclusive sym (0.38). Mondrian NAO atenuou heterogeneidade — apenas
    refletiu o regime shift verdadeiro do dado.

CONCLUSAO MONDRIAN: per-regime bucketing com 15d/bucket + shrinkage
inverse-variance e' instavel demais; para esse alvo, o regime e' melhor
modelado pelo proprio quantile regressor (que e' x-conditional)
do que por bucket discreto pos-hoc. Caminho fechado dentro do envelope
iter_0002/CV-5x60d.

### Bonus per sub (replicacao H26)

ASYM em SE+S (subs que H26 ja' confirmou):
- SE: sym 76.6% -> asym 80.3% (sobe para o meio da banda, ainda dentro
  [75, 85]); width +20% (deal aceitavel)
- S:  sym 79.5% -> asym 81.8% (margem ainda maior); width +51% (S tem
  cauda mais densa, asymmetric infla mais)

Trade-off operacional: para SE+S, ASYM nao adiciona valor sobre H26
symmetric (ambos passam); decisao Breno: usar symmetric (mais simples)
para SE+S em prod, ASYM apenas para NE+N onde resolve gap conhecido.

### Sanity checks (B1-B6)

- **B1 leak**: SKIPPED_INHERITED. Features iter_0002 validadas iter_0010
  H3 (PDP leak-free, corr[t]/curt[t] > corr[t]/curt[t-1] em 5/6). Mesma
  fonte que H26 iter_0037 (heranca).
- **B2 perm**: SKIPPED_INHERITED. PI p=0.0 em 5/6 cells iter_0010 cobre
  features iter_0002.
- **B3 holdout_strict**: PASSED_EMBEDDED. 5 folds walk-forward, gap=7d,
  inner_val sempre temporalmente antes do test. Mondrian bucketing
  usa q50_te (modelo train_inner sobre X_te), NUNCA y_te — D-1-safe.
- **B4 baseline**: PASSED_EMBEDDED. Triple baseline reportado em todas
  as 24 (cell, iv) combinacoes: sym (= H26 bit-exato), asym, mond.
  Replicacao sym confirma identidade numerica vs results.json iter_0037.
- **B5 dist_shift**: ANNOTATED_REUSE. iter_0012 KS p<0.0001 NE+SE em
  y_d1 e' a causa raiz que motivou H37 (per-regime). Mondrian foi a
  resposta hipotetizada; achado empirico = a heterogeneidade nao reduz
  por bucketing pos-hoc nesse n.
- **B6 zero_count**: REPORTED. N tem 37% dias com curt<1 MWh (causa do
  over-coverage symmetric). ASYM corrige por nao penalizar a cauda
  baixa quando ela e' degenerada (q_low ~ 100 MWh em N vs q_alpha
  symmetric ~ 660). y_te_frac_zero por fold em results.json.

## Decisao

**CONFIRMADO_ASYM_ONLY** — variante A (CQR-asymmetric) atinge a
aceitacao formal H37 em ambos os alvos primarios (NE 2/3 in [75, 85],
N 3/3 in [75, 90]). Variante B (Mondrian) REFUTADA por falha em
heterogeneity (std 20.9% > 10pp) E cobertura (NE 1/3 < 2/3).

Acao de loop (replay-only, sem deploy no champion):
1. Atualiza H37 status=done no queue (verdict + actual_value).
2. Adiciona row no leaderboard com numeros NE+N before/after.
3. Atualiza state.json (iter_atual=45, last_verdict=CONFIRMADO_ASYM_ONLY).
4. Sem req externo — ASYM nao muda o modelo do champion (Ridge/LR
   continuam point estimates oficiais); calibracao quantile e' camada
   post-hoc separada que UlFor pode adotar se decidir produtizar bandas.

Champions UlFor (Ridge NE / LR SE+S / Ridge_clean_plus N) INTACTOS.

## Proximo passo

H37 e' o ultimo gap operacional do arco conformal classico no envelope
iter_0002/CV-5x60d. Caminhos derivaveis a partir daqui:

- **(opcional) H42**: produtizar ASYM como camada conformal-asym no
  endpoint forecast_d1_conjunto quando UlFor publicar quantile models
  com mesma estrutura. P3, depende de decisao Breno + UlFor side
  (banda P10/P90 e' deliverable de produto, fora do scope loop).
- **(opcional) H43**: Mondrian com bucketing aprendido (LGBM stacker
  sobre features para escolher q_bucket) — destrava por reduzir o
  problema de "qual bucket usar". P3, custo 1.5h. Mas atratividade
  agora baixa: heterogeneidade NE e' do dado (regime real), nao da
  receita de calibracao.

Sem follow-ups criados imediatamente. O loop esta saturando o envelope
features iter_0002 + CV 5x60d. Proxima onda de ganhos exige (a)
UlFor publicar quantile models para que conformal-asym seja productized
externally, OR (b) novos dados (DESSEM/SINtegre/WeatherNext) que entrem
em iter_0002+1.
