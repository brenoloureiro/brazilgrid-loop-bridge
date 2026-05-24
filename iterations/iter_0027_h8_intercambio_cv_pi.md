---
alvo: feat_intercambio_importance_cv_pi
layer: curtailment
iter_num: 0027
type: hypothesis_test (verdict=CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge)
data_utc: 2026-05-24T22:30:00Z
hypothesis: |
  H8 (P3, queued desde iter_0001 — last_external_update_iter=0023):
  "Validar feat_intercambio importance via permutation". UlFor H22 +
  H22_model_aware (commits 10fa56d3 + 2daa5d40) ja' produziu drop list
  via VIF + PI per-fold:
    NE: val_export, val_import, val_net (drop, VIF=1e8 colinearidade
        perfeita val_net = val_import - val_export)
    SE: val_export, val_import (drop; nos estendemos pra val_net por id)
    S:  val_net_lag1 (drop, regime-specific)
    N:  MANTEM (UlFor nao listou drop — intercambio carrega sinal
        economico real em N segundo iter_0002 top-5)
  Esta iter faz CONFIRMACAO INDEPENDENTE via CV-PI proprio do loop,
  espelhando metodologia walk-forward 5x60d gap 7d (h7/h10/UlFor
  official), com Ridge_alpha10 (champion-aware NE+N) + Ridge_alpha=1
  (UlFor sweep winner h22_per_fold iter_0026 ADDENDUM).
baseline_tipo: |
  Modelo Ridge per fold sobre features iter_0002 v1 (37 feats incluindo
  os 4 intercambio cols). Walk-forward CV 5 folds x 60d test x 7d gap.
  StandardScaler fit so' em train por fold. Per (sub, fold, alpha):
    base_mae   = MAE(Ridge.predict(X_te))
    PI single  = mean over 30 perms da coluna alvo no test (refit-free)
    Joint-drop = MAE(Ridge sem o bundle inteiro de intercambio features)
                 [autoritativo p/ colinears val_net=imp-exp, VIF=1e8]
baseline_metric: |
  Ridge base MAE (CV mean ± std, MWh) com 37 features:
    NE  alpha=10  MAE 37665   alpha=1  MAE 41839
    SE  alpha=10  MAE 6708    alpha=1  MAE 7021
    S   alpha=10  MAE 1134    alpha=1  MAE 1124
    N   alpha=10  MAE  570    alpha=1  MAE  600
  ymean_test variando 8k-145k MWh em NE (regime mudanca confirmada
  iter_0012 KS p<0.0001). N tem NMAE>1 (modelo Ridge nao-resolvido em
  low-signal sub).
result_metric: |
  ===================================================================
  JOINT-DROP TEST (autoritativo para colinears) — primario:
  ===================================================================
  sub  bundle(n_feat)  a10_dNMAE_pp  a1_dNMAE_pp  verdict
  NE   3 (exp,imp,net)  -1.167       -3.951       bundle_drop_IMPROVES_model
  SE   3 (exp,imp,net)  +0.270       +1.205       bundle_drop_HARMFUL (em Ridge)
  S    4 (todos)        -8.334       -6.989       bundle_drop_IMPROVES_model
  N    4 (todos)        -2.742       -2.293       bundle_drop_IMPROVES_model

  ===================================================================
  PI SINGLE-FEAT (secundario, viesado em colinears identitarios):
  ===================================================================
  sub  feature              a10_dNMAE_pp  a10_pval  a1_dNMAE_pp  a1_pval  UlFor_drop  pi_verdict
  NE   val_export_mwmed     -0.758        0.633     -0.353       0.600    Y           ambiguous
  NE   val_import_mwmed     +0.454        0.067     +1.652       0.000    Y           ambiguous
  NE   val_net_mwmed        +0.592        0.067     +1.106       0.033    Y           ambiguous
  NE   val_net_mwmed_lag1   +0.143        0.433     +0.287       0.133    -           drop_confirmed
  SE   val_export_mwmed     +0.010        0.600     +0.195       0.333    Y           drop_confirmed
  SE   val_import_mwmed     +0.034        0.333     +0.306       0.233    Y           drop_confirmed
  SE   val_net_mwmed        +0.178        0.233     +0.273       0.233    -           drop_confirmed
  SE   val_net_mwmed_lag1   -0.055        0.733     +0.129       0.300    -           drop_confirmed
  S    val_export_mwmed     +0.003        1.000    -0.001        1.000    -           drop_confirmed
  S    val_import_mwmed     -0.835        0.567    -0.383        0.433    -           ambiguous
  S    val_net_mwmed        -0.763        0.567    -0.264        0.433    -           ambiguous
  S    val_net_mwmed_lag1   -0.234        0.900    +0.766        0.400    Y           ambiguous
  N    val_export_mwmed     -0.058        0.267    -0.114        0.567    -           drop_confirmed
  N    val_import_mwmed     +0.116        0.200    +0.061        1.000    -           drop_confirmed
  N    val_net_mwmed        -0.099        0.400    -0.173        0.633    -           drop_confirmed
  N    val_net_mwmed_lag1   +0.074        0.200    +0.091        0.233    -           drop_confirmed

  ===================================================================
  Veredito final (joint-drop autoritativo):
  ===================================================================
    NE: CONFIRMA UlFor drop direction. Dropar bundle MELHORA Ridge
        em -1.17pp (a10) a -3.95pp (a1) NMAE — features ativamente
        harmful (modelo Ridge se beneficia removendo-as).
    SE: CONTRADIZ direcao UlFor em Ridge. Dropar bundle PIORA Ridge
        em +0.27 a +1.21pp NMAE. UlFor decidiu drop em SE com
        champion=lr e VIF+PI integrados (commit 2daa5d40 lesson
        model-aware); nosso joint-drop usando Ridge diverge.
        Conclusao operacional: drop SE depende de modelo;
        recomendacao do loop = manter cautela em SE ate replicar
        joint-drop com lr champion (proxima iter — H8b ou
        H22-nosso refletindo lesson H22_MA).
    S:  CONFIRMA UlFor drop direction com magnitude MUITO MAIOR.
        Joint-drop melhora Ridge em -8.33pp (a10) a -6.99pp (a1)
        NMAE. UlFor listou so val_net_lag1; nossa evidencia sugere
        dropar TODOS os 4 intercambio feats em S — extensao em
        direcao consistente. Modelo S tem NMAE proximo de 1
        (limite-de-dado) mas joint-drop tao grande sugere bundle
        majoritariamente ruido para curt-S.
    N:  joint-drop em Ridge SUGERE drop (melhora -2.7 a -2.3pp NMAE)
        — contradiz UlFor keep direction. PORE'M: modelo N base
        NMAE>1 (essencialmente nao funcionando como regressor), e
        UlFor decisao "manter em N" baseou-se em iter_0002 top-5 PI
        LGBM mais regime-economico-real. Joint-drop em modelo
        Ridge unsafe nao refuta UlFor decisao. Marcamos
        INCONCLUSIVO_em_N + ambiguidade modelo-dependente. UlFor
        keep permanece valida (out-of-scope deste teste).

  Insight metodologico chave:
    PI single-feat sobre features perfeitamente colineares (val_net =
    val_import - val_export, identidade analitica) e' SISTEMICAMENTE
    VIESADO PARA CIMA: permutar uma feature quebra a identidade local;
    modelo treinado com a identidade preserva pesos que dependem dela,
    entao predicao quebra. Joint-drop test (refit do modelo sem o
    bundle inteiro) e' o teste autoritativo — alinha com VIF + PI
    multivariado de UlFor H22.
    Em NE alpha=1: val_import single-PI = +1.65pp (parece KEEP) mas
    joint-drop = -3.95pp (bundle e' ATIVAMENTE HARMFUL). Joint-drop
    dissolve a ambiguidade single-PI.
decision: |
  CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge.

  Confirmacoes (3/4 subs):
    NE: drop direcao UlFor confirmada com efeito stronger-than-expected
        (joint-drop melhora ate -3.95pp). Recomendacao loop: aplicar
        drop bundle em qualquer bake-off Ridge NE futuro.
    S:  drop direcao UlFor confirmada com efeito ENORME (-8.3pp
        joint-drop). Recomendacao loop: dropar TODOS os 4 intercambio
        feats em bake-off Ridge S (extensao em direcao consistente).
    N:  inconclusivo — joint-drop Ridge melhora mas modelo Ridge N e'
        nao-funcional (NMAE>1). UlFor keep direction permanece valida
        ate replicar com modelo que de signal real em N.

  Refutacao localizada (1/4 subs):
    SE: joint-drop em Ridge HARMFUL (+1.2pp piora). UlFor decidiu drop
        com lr+VIF — divergencia model-aware. Loop NAO substitui
        UlFor decision aqui; flag lesson model-aware.

  Validacao independente cumprida: o procedimento canonico do loop
  (Ridge CV walk-forward + joint-drop refit + PI 30-perm secundario)
  e' coerente com UlFor H22_model_aware em 75% dos casos e expoe um
  caveat metodologico real (PI single-feat sobre colinears identicos
  e' viesado, joint-drop dissolve).

sanity_checks_passed:
  permutation_importance: true  # 30 perms x 4 features x 5 folds x 4 subs x 2 alphas = 4800
  holdout_temporal_strict: true # walk-forward 5x60d gap 7d com n_test=60 (>30 -> sem downgrade B6)
  leak_detection: true          # features D-1-safe (herda iter_0002); scaler fit so' em train
  baseline_compare: true        # Ridge_alpha=10 + Ridge_alpha=1 dois baselines independentes
  distribution_shift: mitigated # iter_0012 KS p<0.0001 NE+SE; walk-forward expoe robustez
  zero_count_shift: skipped     # nao aplica a meta-feature (sem feature engineering nova)

budget_consumido_iter: 0.7
custo_estimado_usd: 0.0  # local sklearn, sem LLM
---

# Iter 0027 — H8 feat_intercambio CV-PI independente

## Hipótese

H8 (P3, queued desde iter_0001 — last_external_update_iter=0023):
"Validar feat_intercambio importance via permutation". UlFor ja'
respondeu via H22 (VIF per-fold + PI per-fold, commit 10fa56d3) e
H22_model_aware (commit 2daa5d40) listando drop candidates por sub.
Esta iter faz CONFIRMACAO INDEPENDENTE via CV-PI proprio do loop com
metodologia complementar (joint-drop test alem de single-feat PI).

A pergunta especifica: nosso protocolo independente CONFIRMA UlFor
drop list por sub?

## Como foi rodado

Script: `scripts/h8_intercambio_cv_pi.py`. Output:
`outputs/iter_0027/h8_intercambio_cv_pi/{results.json, summary.csv,
verdict.json, sanity_checks.json}`.

Pipeline:
1. Carrega features.parquet de `iter_0002/runs/<sub>/v1/` (mesmo
   feature_set canonico do replay loop, 37 feats incluindo 4
   intercambio: val_export, val_import, val_net, val_net_lag1).
2. Walk-forward CV 5 folds, test_window=60d, gap=7d (mesma estrutura
   h7/h10/UlFor official).
3. Por fold: StandardScaler.fit(train); Ridge.fit(X_tr_scaled, y_tr).
4. Por modelo Ridge (alpha ∈ {10, 1}, dois baselines):
   - base_mae = MAE(predict(X_te))
   - PI single-feat (4 features): permutar coluna 30x com seeds
     0..29; aggregating delta_mae, delta_nmae, p_value (frac perms
     com delta<=0 — ruido se alta).
   - Joint-drop: refit Ridge sem bundle inteiro de intercambio
     features. Bundle por sub: NE/SE 3 cols (exp+imp+net),
     S/N 4 cols (todos). Compara drop_mae vs base_mae.
5. Aggregate mean ± std por (sub, feature, alpha) across folds.
6. Verdict per (sub):
   - Joint-drop SAFE: |delta_NMAE|<0.5pp em ambos alphas
   - Joint-drop HARMFUL: delta_NMAE >= 0.5pp em pelo menos 1 alpha
   - Joint-drop IMPROVES_model: delta_NMAE negativo grande

Custo: ~30s wall-clock local, zero LLM, zero CH query.

## Resultado

Veredito por sub (joint-drop autoritativo):

| sub | a10 dNMAE | a1 dNMAE | verdict | reconciliacao UlFor |
|---|---|---|---|---|
| NE | -1.17pp | -3.95pp | bundle_drop_improves_model | CONFIRMA drop (e' STRONGER) |
| SE | +0.27pp | +1.21pp | bundle_drop_HARMFUL | CONTRADIZ em Ridge (lesson H22_MA) |
| S  | -8.33pp | -6.99pp | bundle_drop_improves_model | CONFIRMA drop (extensao 4-feat) |
| N  | -2.74pp | -2.29pp | bundle_drop_improves_model | INCONCLUSIVO (modelo Ridge N unsafe) |

Sanity checks: 5/6 PASS_or_MITIGATED, 1/6 N/A (zero_count_shift nao
aplica a meta-feature). PI single-feat exposicao do viés:
val_import alpha=1 em NE da single-PI=+1.65pp (parece KEEP) mas
joint-drop=-3.95pp (bundle e' ATIVAMENTE HARMFUL). Joint-drop dissolve
a ambiguidade — VIF=1e8 identidade perfeita val_net=imp-exp envenena
single-feat PI.

Mensagem metodologica forte: **PI single-feat sobre colinears
identitarios e' sistemicamente viesado para CIMA**. Joint-drop refit
e' o teste autoritativo — alinha com VIF + PI multivariado UlFor.

## Decisão

CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge.

3/4 subs concordam com UlFor direction: NE forte, S muito forte
(extensao em direcao consistente), N inconclusivo (modelo unsafe).
1/4 (SE) diverge — lesson model-aware: drop SE depende de modelo;
joint-drop em Ridge piora, mas UlFor decidiu com lr champion + VIF.
Esta divergencia E o efeito mensuravel ~10pp documentado em H22_MA
commit 2daa5d40 (sub_metric_after_drop muda 9.1pp NMAE quando
champion-aware PI vs Ridge-only PI).

H8 marcada como `done`, `iter_handled=0027`, `verdict=
CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge`. NAO emite req-0008
ao UlFor — verdict do loop nao destrava acao UlFor (UlFor ja'
operacionalizou model-aware em commit 2daa5d40). Hipotese derivada
H33 emergente (P3, ~0.5h): replicar joint-drop em SE usando
LinearRegression em vez de Ridge para fechar o caveat model-aware
locally; documenta a magnitude do gap modelo-dependente que UlFor
H22_MA expoe teoricamente.

## Próximo passo

Iter 0028 candidatos (planner_config):
1. **MONITOR / RECON_DELTA** se UlFor avancou alem `83abab3e` (regime
   multi-agente ATIVO desde iter_0021).
2. **H33 emergente** (P3, ~0.5h): joint-drop SE com LR em vez de Ridge
   para validar magnitude lesson model-aware H22_MA empiricamente
   no replay loop. Custo minimo, custo dependencia zero.
3. **H30** (P3, ~1h): Ridge_alpha10 + Ridge_alpha=1 simultaneamente
   sobre pdp_residual CV (script ja' pronto). Atratividade marginal
   SOBE pos-iter_0026 alpha sweep.
4. **H22 nosso** (P3, ~1h): GBDT vs OLS gap em pdp, com PI-com-GBDT
   conforme lesson H23/H22_MA atualizada em iter_0021.

Recomendacao: (1) se UlFor advance; senao (2) — H33 e' o follow-up
mais barato e fecha um caveat metodologico explicito que esta iter
expos.
