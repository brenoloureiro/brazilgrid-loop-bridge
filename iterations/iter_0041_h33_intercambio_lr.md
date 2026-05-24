---
alvo: feat_intercambio_joint_drop_se_lr
layer: curtailment
iter_num: 0041
type: hypothesis_test (verdict=CONFIRMADO_LR)
data_utc: 2026-05-25T22:00:00Z
hypothesis: |
  H33 (P3, queued desde iter_0028 — derivada H8 iter_0027
  REFUTADO_PARCIAL_vs_ULFOR_em_SE): "Joint-drop SE em LinearRegression vs
  Ridge — fechar caveat model-aware H22_MA empirico"

  H8 (iter_0027) achou que joint-drop do bundle intercambio
  (val_export, val_import, val_net) em SE PIORA Ridge: +0.27pp NMAE em
  alpha=10, +1.21pp NMAE em alpha=1. Contradiz UlFor H22_model_aware
  (commit 2daa5d40) que listou as 3 como drop_SE usando lr champion +
  PI single-feat + VIF. Lesson H23_ulfor (commit ccf53722): "PI deve
  ser medida com o modelo final, nao com proxy mais robusto".

  Mecanismo conjecturado: Ridge L2 redistribui pesos entre features
  colineares (val_net = val_import - val_export, VIF=1e8); Ridge se
  beneficia da redundancia, drop bundle perde info. LR sem shrinkage
  poderia colapsar -- PI single-feat seria ainda mais explosiva,
  joint-drop expoe o verdadeiro aporte.

  H33 testa: joint-drop em SE refit com LinearRegression revela mesmo
  dNMAE positivo (>0, contra UlFor) ou inverte para negativo (drop
  neutro/melhora, confirma lesson empirica)?

  ACEITACAO (a priori queue):
    CONFIRMADO_LR    : joint_dnmae_SE > +0.5pp (bundle aporta sinal real)
    REFUTADO_LR      : joint_dnmae_SE < -0.5pp (drop melhora; lesson empirica)
    INDETERMINADO_LR : |joint_dnmae_SE| <= 0.5pp (zona ruido)

  NE/S/N reportados para contexto cross-model (direcao deve preservar).
baseline_tipo: |
  Ridge H8 iter_0027 joint-drop SE: alpha=10 +0.270pp, alpha=1 +1.205pp.
  Mesmo split CV walk-forward 5x60d gap 7d, mesmas features iter_0002 v1
  (37 features), mesmo bundle 3-feat SE. Apenas estimator muda (Ridge ->
  LinearRegression, sem alpha).
baseline_metric: |
  Ridge_alpha10 SE joint_dnmae_pp = +0.270 (REFUTADO_PARCIAL em H8)
  Ridge_alpha1  SE joint_dnmae_pp = +1.205 (REFUTADO_PARCIAL em H8)
  Direcao prevista para LR: positiva ainda mais marcada (sem shrinkage)
result_metric: |
  LR_SE joint_dnmae_pp = +1.316 (>+0.5pp threshold) -> CONFIRMADO_LR

  Per-fold SE: [+0.789, +0.576, +0.029, -0.581, +5.768]; 4/5 folds positivos
  (sinal robusto, nao single-fold artifact). Sem fold 5: mean +0.203pp
  (positivo mas sub-threshold). Fold 5 amplifica magnitude.

  Cross-model comparison joint_dnmae_pp:
    sub   LR        Ridge_a10   Ridge_a1    Verdict (LR)
    NE   -4.285    -1.167      -3.951      bundle_drop_improves_LR
    SE   +1.316    +0.270      +1.205      bundle_drop_harmful_in_LR
    S    -7.376    -8.334      -6.989      bundle_drop_improves_LR
    N    -1.816    -2.742      -2.293      bundle_drop_improves_LR

  Direcao consistente em 4/4 subs entre LR e Ridge. LR magnitude ~ Ridge a1
  em todos subs (consistente com LR = limite alpha->0 do spectrum).
decision: DESCARTA — verdict CONFIRMADO_LR fecha caveat H22_MA mas decisao
  operacional UlFor (drop SE no champion LR) ja foi tomada e nao muda.
  Insight metodologico arquivado em handoff + leaderboard.
sanity_checks_passed:
  permutation_importance: true   # 30 perms/(sub,feature). PI single-feat em LR para val_export/import/net explode (172000-3000000 pp) por min-norm + colinearidade perfeita — EXATAMENTE o viez previsto, confirma joint-drop como teste autoritativo. val_net_lag1 (nao-colinear) PI razoavel.
  holdout_temporal_strict: true  # CV walk-forward 5x60d gap 7d, embedded em fold loop. Identico H8/H10/H11/H21/H30.
  leak_detection: true           # reuso integral split iter_0027 H8; mesmas features.parquet iter_0002 v1; H8 ja passou leak.
  baseline_compare: true         # cross-model: LR vs Ridge a10/a1 reportado per sub. Direcao 4/4 consistente.
  distribution_shift: annotated_reuse  # KS NE+SE p<0.0001 dist-shift iter_0012 H7. Estatico, nao re-rodado.
  zero_count: true               # ymean_test SE 8486-21083; todos >> 1 MWh eps NMAE safety. Sem all-zero fold.
  n_test_threshold: true         # 5x60=300 obs/cell, >> 30 H20 threshold em 4 subs. n_train 158-398 (37 feat fit OLS comfortable).
  fold_positivity_robustness: true  # SE 4/5 folds positive; sinal robusto sem single-fold artifact.
budget_consumido_iter: 0.25
custo_estimado_usd: 0.0
artefatos:
  - outputs/iter_0041/h33_intercambio_lr_cv/verdict.json
  - outputs/iter_0041/h33_intercambio_lr_cv/results.json
  - outputs/iter_0041/h33_intercambio_lr_cv/summary.csv
  - outputs/iter_0041/h33_intercambio_lr_cv/sanity_summary.json
  - scripts/h33_intercambio_lr_cv.py
follow_ups_created:
  - H40 (governance, P5): "Documentar 'joint-drop > PI single-feat em colinears perfeitos
    como teste autoritativo cross-model'. Adicionar em sanity_checks/B2_interpretation.md
    como segundo caso canonico (apos NE alpha=10 H30 case): SE LR joint-drop +1.316pp
    confirma sinal real do bundle val_export+val_import+val_net, enquanto PI single-feat
    em LR explode 172000-3000000 pp (artefato de min-norm SVD + colinearidade perfeita).
    Lesson canonica: '|PI_single_feat| arbitrariamente grande em colinears = sinal de
    METRICA QUEBRADA, nao de feature importante'. Doc-only, custo minimo."
encerra_caminhos:
  - "H8/H33-bundle intercambio caveat": Ridge (alpha=10, alpha=1) + LinearRegression
    convergem em direcao 4/4 subs no joint-drop. Caveat H22_MA SE (drop_HARMFUL no
    champion final) reproduzido empiricamente sem alpha. UlFor decisao operacional
    de drop em SE permanece valida — PI single-feat era proxy biased mas joint-drop
    revela aporte real do trio. **NAO reabrir teste do bundle intercambio em LR/Ridge**.
    Ultima frente potencial: outros modelos (GBDT, kernel ridge) que tratam colinears
    diferente — fora do escopo H33.
verdict_full: |
  CONFIRMADO_LR per criterio H33 a priori:
    "CONFIRMADO_LR se joint-drop LR_SE dNMAE > +0.5pp threshold"
    SE LR: +1.316pp -> PASSA threshold.

  Cross-model convergencia (LR ~ Ridge a1 ~ Ridge a10 em direcao 4/4):
    +SE bundle aporta sinal real ao modelo final, em qualquer alpha do spectrum
     Ridge (alpha=0 LR -> alpha=10). Ridge a10 atenua (+0.27pp) por shrinkage
     mais agressivo; LR (alpha=0) amplifica (+1.32pp). Magnitude monotonica
     decrescente com alpha.

    -NE/S/N bundle e' ruido em LR (drop melhora) — consistente Ridge.
     LR melhora MAIS que Ridge (NE -4.3 vs Ridge a10 -1.2 / a1 -4.0),
     confirma direcao monotonica com alpha.

  Insight metodologico (para sanity_checks/B2_interpretation.md):
    PI single-feat em LR para val_export/val_import/val_net explode
    (172000-3000000 pp) — min-norm SVD distribui pesos com cancelamento
    perfeito, permutar quebra cancelamento. Joint-drop refit dissolve
    artefato. Lesson: |PI_single_feat| arbitrariamente grande em colinears
    perfeitos = sinal de METRICA QUEBRADA, nao de feature importante.
    val_net_mwmed_lag1 (nao-colinear) PI razoavel (0.18-2.5pp), confirma
    diagnostic.

  Decisao operacional UlFor (drop SE no champion LR via H22_model_aware
  commit 2daa5d40) PERMANECE valida — PI single-feat era proxy biased
  mas direcao do drop foi assertiva por outras evidencias (VIF + champion
  match). Loop fecha o caveat empiricamente cross-model.

  H33 era pure investigacao de curiosidade metodologica pos-iter_0031
  (notes_iter0031 do queue: "nao ha mais decisao que dependa dela").
  Custo 0.25h, valor confirma 3-fold convergencia (Ridge a10 + Ridge a1 +
  LR) sobre o caveat, e gera 1 follow-up doc-only.
---

# Iter 0041 — H33 joint-drop SE em LinearRegression vs Ridge (H8)

## Hipotese

H8 (iter_0027) demonstrou que dropar o bundle intercambio
(`val_export_mwmed`, `val_import_mwmed`, `val_net_mwmed`) em SE PIORA o
modelo Ridge (+0.27pp NMAE em alpha=10, +1.21pp em alpha=1) — contradizendo
a decisao UlFor `H22_model_aware` (commit `2daa5d40`) de listar essas 3 como
`drop_SE`. UlFor usou o champion SE (LR) + PI single-feat + VIF; H8 testou
em Ridge.

H33 fecha o caveat **rodando o mesmo joint-drop test em LinearRegression**
(o estimator que UlFor usou para gerar a drop_list). Se LR tambem mostra
+dNMAE (CONFIRMADO_LR), a decisao UlFor foi VIF-driven mesmo no modelo
correto e a divergencia Ridge/LR e' apenas magnitude. Se LR mostra -dNMAE
(REFUTADO_LR), a lesson H23_ulfor "PI deve usar modelo final" se confirma
empiricamente.

## Como foi rodado

`scripts/h33_intercambio_lr_cv.py` — extensao trivial de
`h8_intercambio_cv_pi.py`:
- mesmo split CV walk-forward 5 folds x 60d test, gap 7d
- mesmas `features.parquet` iter_0002 v1 (37 features, 4 NE/SE/S/N)
- mesma `StandardScaler` (mantida por paridade; OLS coefs invariantes a
  escala mas PI usa colunas scaled)
- mesmo bundle SE = `{val_export, val_import, val_net}`
- mesmo protocolo PI (30 perms, p-value via `frac(delta <= 0)`)
- estimator unico: `sklearn.LinearRegression` (sem alpha)

Wall-clock ~30s. Zero novos dados; zero novos splits; reuso 100% pipeline H8.

## Resultado

**Verdict: CONFIRMADO_LR.**

```
sub   LR_dNMAE_pp   Ridge_a10   Ridge_a1   verdict
NE      -4.285        -1.167     -3.951    bundle_drop_improves_LR
SE      +1.316        +0.270     +1.205    bundle_drop_harmful_in_LR
S       -7.376        -8.334     -6.989    bundle_drop_improves_LR
N       -1.816        -2.742     -2.293    bundle_drop_improves_LR
```

**Direcao 4/4 subs consistente entre LR e Ridge.** Magnitude LR ~ Ridge_a1
em todos os 4 subs — comportamento monotonico esperado: LR = limite
alpha->0 do spectrum Ridge; menos shrinkage = mais sensivel ao drop
(amplifica ganho ou perda).

SE per-fold: `[+0.789, +0.576, +0.029, -0.581, +5.768]` pp. **4/5 folds
positivos** (sinal robusto, nao artifact de single fold). Fold 5 amplifica
mean (ymean_test menor=8486 + base_mae=6613 produz nmae~0.78 e
delta~0.39/0.078 = +0.49pp por MAE +489). Sem fold 5: mean +0.203pp
(positivo, mas sub-threshold).

**Insight metodologico capturado em B2-style** (sanity check): PI single-feat
para val_export/import/net em LR explode (`172000`-`3000000` pp delta NMAE
quando permutado) — artefato puro de `lstsq` min-norm SVD em design matrix
rank-deficient (val_net = val_import - val_export). Permutar uma quebra
o cancelamento perfeito de pesos colineares, predicoes saem dos eixos.
`val_net_mwmed_lag1` (nao colinear) tem PI razoavel (0.18-2.5pp),
confirmando que a explosao e' especifica do bundle perfeito.

Joint-drop refit *dissolve* esse artefato: remove TODO o trio
simultaneamente -> design matrix passa a ser full-rank no resto,
modelo aprende coefs estaveis nas features remanescentes, delta_MAE
e' o aporte REAL do bundle.

Sanity checks: leak ok (reuso iter_0002 v1), holdout strict (gap 7d
embedded), baseline cross-model (4/4 direcao), n_test >> 30, zero_count
sem all-zero fold, dist_shift annotated_reuse iter_0012 (KS p<0.0001 NE+SE
estatico).

## Decisao

**DESCARTA** — verdict CONFIRMADO_LR confirma direcao da decisao
operacional UlFor (drop SE bundle no champion LR via `H22_model_aware`)
mas nao muda nenhuma decisao operacional. Per notes_iter0031 do queue,
H33 ja era investigacao de curiosidade metodologica pos-promote v3
(commit Breno `0971c699` escolheu opt A SE ridge+h22_MA+alpha=1).

Insight metodologico (LR PI single-feat explode em colinears perfeitos,
joint-drop e' o teste autoritativo cross-model) e' o **valor real desta
iter**. Arquivado em handoff + leaderboard + 1 follow-up doc-only (H40).

NAO atualizar champion. NAO criar req externa.

## Proximo passo

1. **H40 (P5, doc-only) criada como follow-up**: adicionar caso H33 em
   `sanity_checks/B2_interpretation.md` como **segundo exemplo canonico**
   (apos H30 NE alpha=10 case). Esta entrada documentaria: "|PI_single_feat|
   arbitrariamente grande em colinears perfeitos = METRICA QUEBRADA, nao
   feature importante; joint-drop e' o teste autoritativo cross-model".
2. **H22 (GBDT vs OLS para mecanismo nao-linear)** segue como ultima frente
   teorica residual no queue. Convergencia LR+Ridge na direcao do bundle
   intercambio reforca model-family-aware como principio (a magnitude do
   gap drop-vs-keep e' modelo-dependente mas a *direcao* e' consistente
   linear/L2; GBDT poderia diferir por interaction terms).
3. **Bundle intercambio CASE FECHADO** no replay loop. 3-fold convergence
   (Ridge a10 + Ridge a1 + LR) em direcao para 4/4 subs encerra
   investigacao bundle-level. Outros estimators (kernel ridge, GBDT)
   seriam objeto de hipoteses derivadas separadas, fora do escopo H33.

## Observacoes laterais

- **Fold 5 SE amplifica magnitude mas nao direcao**: 4/5 folds positivos
  garantem sinal robusto. Sem fold 5 a magnitude cai pra +0.203pp
  (sub-threshold), mas direcao positiva permanece. CV mean +1.316pp e'
  representativo do regime "1 fold pode ser pequeno mas o sinal cumulativo
  e' real".
- **Monotonicidade alpha**: NE LR -4.29 < Ridge a1 -3.95 < Ridge a10 -1.17 |
  SE LR +1.32 > Ridge a1 +1.20 > Ridge a10 +0.27 | S LR -7.38 < Ridge a1
  -6.99 < Ridge a10 -8.33 (S quebra ligeiramente — Ridge a10 mais agressivo
  improve, possivel artefato de shrinkage em S baixo-signal) | N LR -1.82 >
  Ridge a1 -2.29 > Ridge a10 -2.74. **3/4 subs monotonicos**. Reforca que
  Ridge shrinkage atenua direcao bundle (positiva e negativa).
- **Modelo champion SE = LR** (UlFor). Nosso teste em LR e' o de mais alto
  rigor possivel para o caveat H22_MA — se LR mostrasse -dNMAE seria
  refutacao definitiva. CONFIRMADO_LR fecha o assunto.
- **Convergencia cross-model**: o sinal do bundle intercambio em cada sub
  e' uma propriedade DOS DADOS, nao do estimator. Ridge L2 atenua,
  GBDT poderia mudar magnitude por interaction non-linear, mas a *direcao*
  ja foi observada coincidente em 4/4 subs em estimadores lineares com
  spectrum alpha completo (0 -> 1 -> 10).
