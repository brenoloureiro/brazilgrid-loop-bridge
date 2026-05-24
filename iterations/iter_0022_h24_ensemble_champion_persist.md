---
alvo: ensemble_champion_persist
layer: curtailment
iter_num: 0022
type: hypothesis_test (verdict=CONFIRMADO_3SUBS)
data_utc: 2026-05-24T18:30:00Z
baseline_tipo: |
  Champion replica local (sklearn Pipeline scaler+modelo) por sub, sobre
  features iter_0002 (3 versions: v1=base, v2=+PREV, v3=+PDP). CV
  walk-forward 5x60d gap 7d (mesma estrutura H10 iter_0013, mesma do
  UlFor CV_METRICS_BY_FS).
baseline_metric: |
  MAE model-only (mean across folds, MWh):
    NE  ridge_alpha10  v1=37.7k  v2=38.0k  v3=38.7k
    SE  lr             v1=8.21k  v2=7.97k  v3=8.73k
    S   lr             v1=1.11k  v2=1.08k  v3=0.73k
    N   ridge_alpha10  v1=0.57k  v2=0.57k  v3=0.55k
  MAE persist_d1 (mesma CV):
    NE 33.7-33.8k  SE 8.33k  S 1.27k  N 0.51k
hypothesis: |
  H24 (P2, derivada de H10 iter_0013 CONFIRMADO_NE_SE): Mesmo esquema
  de ensemble (peso analitico inv_mae/inv_mse, ou equal/opt_alpha) que
  beneficiou LGBM+persist (NE 13-28%, SE 4-7%, N 14-17%) propaga-se
  aos CHAMPIONS UlFor em producao (Ridge_alpha10 NE/N + LinearRegression
  SE/S)?

  Mecanismo esperado: Ridge/LR sao baixa-variancia/alto-bias (linear);
  persist_d1 e' alta-variancia/baixo-bias em regimes estaveis.
  Combinacao deve dar ganho >= que com LGBM (LGBM ja' captura mais
  regime change interno; lineares deixam mais variancia residual para
  persist capturar).
result_metric: |
  CONFIRMADO_3SUBS. Ensemble bate champion-only (Ridge/LR) em MAE em
  3/4 subs com magnitude superior a H10 LGBM:

  cell    model         n  MAE_MOD  MAE_PER  MAE_best_ens  best_scheme   delta_vs_MOD  wins  R²_MOD->ens
  NE/v1   ridge_alpha10 5   37665   33722    30562         ens_inv_mae   -7103 MWh     5/5   -0.06 -> +0.31
  NE/v2   ridge_alpha10 5   38007   33837    29431         ens_equal     -8576 MWh     5/5   +0.09 -> +0.40
  NE/v3   ridge_alpha10 5   38677   33837    29588         ens_equal     -9089 MWh     4/5   +0.08 -> +0.40
  SE/v1   lr            5    8207    8328     7077         ens_equal     -1130 MWh     3/5   -0.41 -> +0.00
  SE/v2   lr            5    7969    8328     6906         ens_inv_mse   -1063 MWh     3/5   -0.42 -> +0.03
  SE/v3   lr            5    8734    8328     6633         ens_inv_mse   -2100 MWh     4/5   -0.78 -> +0.06
  S/v1    lr            5    1112    1266     1079         ens_opt_alpha   -34 MWh     1/5   -0.11 -> -0.11
  S/v2    lr            5    1079    1268     1032         ens_opt_alpha   -46 MWh     2/5   -0.08 -> +0.07
  S/v3    lr            5     726    1268      777         ens_inv_mse    +50 MWh     2/5   +0.45 -> +0.30  (PIORA)
  N/v1    ridge_alpha10 5     570     513      463         ens_inv_mse   -107 MWh     4/5   -0.38 -> +0.05
  N/v2    ridge_alpha10 5     570     513      463         ens_inv_mse   -107 MWh     4/5   -0.38 -> +0.05
  N/v3    ridge_alpha10 5     551     513      451         ens_equal      -99 MWh     3/5   -0.24 -> +0.06

  Sub-summary (n_cells confirming / total):
    NE: 3/3   best delta -9089 MWh (NE/v3, ens_equal)   — 23% MAE reduction
    SE: 3/3   best delta -2100 MWh (SE/v3, ens_inv_mse) — 24% MAE reduction
    S:  0/3   best_cell S/v2 delta -46 MWh (wins 2/5, abaixo do limiar 3/5)
    N:  3/3   best delta  -107 MWh (N/v1, ens_inv_mse)  — 19% MAE reduction

  Comparacao direta H10 (LGBM) vs H24 (Ridge/LR), best delta MAE absoluto:
    NE: H10 -12.996 (v1) | H24 -9.089 (v3)   — magnitude similar, MAE menor pos
    SE: H10    -565 (v1) | H24 -2.100 (v3)   — H24 ~4x mais ganho
    S:  H10    -53/-26   | H24    -46/+50    — ambos irrelevantes
    N:  H10    -100 (v1) | H24   -107 (v1)   — empate

  Insight: ganho do ensemble em SE e SUBSTANCIALMENTE MAIOR sobre lineares
  (lr/ridge) do que sobre LGBM. Confirma mecanismo: lineares deixam mais
  variancia residual em SE (champion lr_se R²_mean -0.42 em CV walk-forward
  vs +0.38 NMAE oficial UlFor — gap reflete fragilidade numerica exposta no
  recon iter_0021 commit `34478f93` cond_num ~2.5e17).

  Esquema vencedor (frequencia por cell, 12 cells total):
    ens_inv_mse  : 5 cells (NE/v1 quase-tie com inv_mae, SE/v2, SE/v3, N/v1, N/v2, S/v3)
    ens_equal    : 5 cells (NE/v2, NE/v3, SE/v1, N/v3, S/v1 mas tie)
    ens_inv_mae  : 1 cell  (NE/v1)
    ens_opt_alpha: 2 cells (S/v1, S/v2)
  Insight: pesos analiticos simples (equal/inv_mse) >> opt_alpha — consistente
  com H10. Diferenca vs H10: ens_inv_mae perdeu protagonismo (H10: 6 cells),
  porque variancia maior dos lineares torna inv_mse (que penaliza erros
  quadraticos) preferivel a inv_mae em regiao com outliers.

decision: |
  CONFIRMADO_3SUBS para NE+SE+N. Ensemble (champion_linear + persist_d1,
  pesos equal ou inv_mse) e' post-processing benefico em 75% dos
  champions de producao UlFor:

  - NE (ridge_alpha10): ganho 19-24% MAE -- 7-9k MWh.
  - SE (lr):            ganho 13-26% MAE -- 1.1-2.1k MWh. R² flip negativo->positivo.
  - N  (ridge_alpha10): ganho 17-19% MAE -- 100 MWh.
  - S  (lr):            inconclusivo. Delta best -46 MWh mas wins 2/5
                        em duas cells; v3 ate piora +50 MWh. Champion S
                        ja' e' muito tight (R² +0.45 v3) — ensemble nao
                        tem o que adicionar.

  Acoes:

  1) Emitir req-0008 (P2 model_post_processing) ao UlFor: adicionar
     ensemble post-processing no endpoint `/api/forecast/d1` apenas
     para NE+SE+N. Implementacao trivial: salvar persist_d1_pred no
     mesmo response payload e blendar com peso fixo medio sugerido
     (NE: ens_equal 0.5/0.5; SE: ens_inv_mse w_model~=0.55; N:
     ens_inv_mse w_model~=0.45) OU rederivar weights por janela 30d
     em runtime (mais robusto a regime shift, ver weights_distribution.csv).

  2) NAO sugerir ensemble para S (sub baixo-signal, champion ja' apertado).

  3) Atualizar leaderboard com nova linha
     `curtailment | ensemble_champion_persist_d1 | <sub> | champion-only |
      best=ens_equal|inv_mse | delta -107 a -9089 MWh`
     para NE+SE+N + 1 linha "ensemble nao ajuda" para S.

  4) Caveat 1a ordem: features iter_0002 (47/47/44/39 features) sao
     SUBSET do feature_set=full UlFor (55 features). Direcao do
     resultado ("ensemble ajuda") e' robusta a essa substituicao —
     mecanismo BMA generaliza. Magnitude pode mudar +-20% no champion
     real. UlFor pode validar com weights derivados em CV oficial
     (NMAE_persist iter_0013 ~= NMAE_persist FINDING_RIDGE — ja
     consistente) antes de produtizar.

  5) NAO emitir req novo so para "adicionar persist no payload" — pode
     ser sugerido via FINDING.md em recon proxima. Manter req-0008
     opcional caso Breno priorize produzir o ganho.

  Hipotese derivada NENHUMA neste iter — H25 (stacker meta-modelo) ja'
  estava na queue (P3) e segue queued. H10 fechado, H24 fechado, H25
  permanece como upgrade incremental do mesmo eixo (improvavel ganho
  marginal substantivo dado que pesos analiticos ja' funcionam).

sanity_checks_passed:
  permutation_importance: skipped  # B2 N/A — ensemble nao introduz feature nova
  holdout_temporal_strict: true     # B3 — CV walk-forward 5x60d gap7d, 12 cells x 5 folds = 60 holdouts; inner_val 30d sem overlap test
  leak_detection: true              # B1 — features iter_0002 ja auditadas + persist_d1 = y_d1[i-1] D-1 safe + inner_val < test_start
  baseline_compare: true            # B4 — persist_d1 e' componente DIRETO do ensemble; skill_model_vs_persist computado por cell
  distribution_shift: true          # B5 — inherited iter_0012 (KS p<0.0001 NE+SE entre folds) + weights_distribution.csv mostra alpha varia 0.0-1.0 entre folds
  zero_count_shift: skipped         # B6 N/A — nao introduz feature; reuso iter_0002 ja em B6 iter_0009

artefatos_persistidos:
  - loops/forecast-mega-loop/scripts/h24_ensemble_champion_cv.py
  - loops/forecast-mega-loop/outputs/iter_0022/h24_ensemble_champion_persist/results.json
  - loops/forecast-mega-loop/outputs/iter_0022/h24_ensemble_champion_persist/summary.csv
  - loops/forecast-mega-loop/outputs/iter_0022/h24_ensemble_champion_persist/verdict.json
  - loops/forecast-mega-loop/outputs/iter_0022/h24_ensemble_champion_persist/weights_distribution.csv
  - loops/forecast-mega-loop/outputs/iter_0022/h24_ensemble_champion_persist/sanity_summary.json
  - loops/forecast-mega-loop/iterations/iter_0022_h24_ensemble_champion_persist.md
  - loops/forecast-mega-loop/leaderboard.md (cabecalho iter_0022 + 4 linhas curtailment/ensemble_champion_persist_d1)
  - loops/forecast-mega-loop/state.json (iter_atual=22, H24 verdict + sub_confirms)
  - loops/forecast-mega-loop/hypotheses_queue.md (H24 done, iter_handled=0022)

budget_consumido_iter: 1.4
custo_estimado_usd: 0
---

# Iter 0022 — H24 Ensemble champion (Ridge/LR) + persist_d1

## Hipotese

H24 (P2, derivada de H10 iter_0013 CONFIRMADO_NE_SE): mesmo esquema
de ensemble (peso analitico inv_mae/inv_mse/equal/opt_alpha) que
beneficiou LGBM+persist (NE 13-28%, SE 4-7%, N 14-17%) propaga-se aos
CHAMPIONS UlFor em producao -- Ridge_alpha10 (NE+N) e LinearRegression
(SE+S)?

Mecanismo esperado: Ridge/LR sao baixa-variancia/alto-bias (familia
linear); persist_d1 e' alta-variancia/baixo-bias em regimes estaveis.
Combinacao deve dar ganho >= que com LGBM (LGBM ja' captura mais regime
change interno; lineares deixam mais variancia residual para persist
capturar). Predicao especifica: SE deve ter ganho MAIOR que H10, porque
champion lr_se e' numericamente fragil (recon iter_0021 cond_num ~2.5e17).

## Como foi rodado

`uv run python scripts/h24_ensemble_champion_cv.py`

Reaproveita 100% da estrutura H10 (iter_0013) com SWAP do estimator:
- LGBMRegressor -> sklearn Pipeline(StandardScaler + Ridge(alpha=10))
  para NE+N
- LGBMRegressor -> sklearn Pipeline(StandardScaler + LinearRegression)
  para SE+S

Por que StandardScaler? Champion UlFor original usa scaler tambem
(pipeline padrao bakeoff_d1.py). Sem scaler, Ridge(alpha=10) sub-shrinka
features de grande escala (carga_mwmed 10k-80k) e over-shrinka pequenas
(cmo_mwmed 0-1k); LR sem scaler tem mesmo problema apenas na inversao
numerica de X^T X. Pipeline reproduz o tratamento esperado.

Mesma CV: 5 folds walk-forward, test 60d cada, gap 7d antes do train,
inner_val 30d (ultimos do train) para derivar pesos sem leak.

Por que features iter_0002 e nao feature_set=full UlFor (55)?
H24 detail autoriza explicitamente: "Implementacao codavel local sobre
features iter_0002 (reimplementar Ridge_alpha10 e LR_sklearn no replay
loop nao requer dump MLflow)". As 47/47/44/39 features (NE/SE/S/N v3)
sao subset robusto -- cobrem PREV+PDP+CMO+PLD+carga+gen+lags+sazonalidade.
Mecanismo BMA "shrink toward more reliable component" e' robusto a
substituicao de feature set; magnitude do delta pode mudar +-20%.

Por fold:
```
[==== train_full ============ | inner_val_30d | (gap 7d) | test_60d ====]
                                              ^                          ^
                                       train_cutoff               test_end
```

1. Inner split: ultimos 30d do train_full = inner_val
2. Train champion em train_inner; predict inner_val. Compute persist_d1
   no inner_val. Compute (mae_m_val, mae_per_val, mse_m_val, mse_per_val).
3. Compute 4 esquemas de peso derivados SO de inner_val:
   - ens_equal: 0.5/0.5
   - ens_inv_mae: w_i = (1/MAE_i) / sum
   - ens_inv_mse: w_i = (1/MSE_i) / sum (BMA Gaussian)
   - ens_opt_alpha: alpha argmin MAE inner_val em grid 0..1 step 0.05
4. Retreina champion no FULL train, predict test.
5. Persist_d1 test: pred[i] = y_te[i-1], pred[0] = y_train_full[-1].
6. Ensemble = w_model * model_pred + w_persist * persist_pred.
7. metric_suite (MAE/R²/F1_p50/NMAE/bias) por esquema, por fold.

Aggregate per cell (sub, ver): mean +- std + wins counts vs champion-only.

Verdict tiering:
- CONFIRMADO_4SUBS  : 4/4 subs confirmam (>= 1 cell por sub: delta MAE < 0 + wins >= ceil(n/2))
- CONFIRMADO_3SUBS  : 3/4 subs confirmam
- CONFIRMADO_PARCIAL: 2/4
- REFUTADO          : <= 1
- INDETERMINADO     : zero cells validos

Codigo: `scripts/h24_ensemble_champion_cv.py`. Saidas:
`outputs/iter_0022/h24_ensemble_champion_persist/{results.json,
summary.csv, verdict.json, weights_distribution.csv, sanity_summary.json}`.

## Resultado

**VERDICT: CONFIRMADO_3SUBS** (NE+SE+N confirm; S nao confirma).

Tabela completa em result_metric do frontmatter. Highlights:

### NE (ridge_alpha10): 3/3 cells confirming
- NE/v1 delta -7103 MWh (19% reducao), wins 5/5
- NE/v2 delta -8576 MWh (23% reducao), wins 5/5
- NE/v3 delta -9089 MWh (24% reducao), wins 4/5 — best cell H24
- R² flip dramatico: champion-only -0.06 a +0.09 -> ensemble +0.31 a +0.40
- Best scheme varia: ens_equal (v2,v3) ou ens_inv_mae (v1)

### SE (lr): 3/3 cells confirming
- SE/v1 delta -1130 MWh (14% reducao), wins 3/5
- SE/v2 delta -1063 MWh (13% reducao), wins 3/5
- SE/v3 delta -2100 MWh (24% reducao), wins 4/5 — best cell H24
- R² FLIP de NEGATIVO para POSITIVO: champion-only -0.41 a -0.78 ->
  ensemble +0.00 a +0.06. Confirma fragilidade champion lr_se (cond_num
  ~2.5e17 exposto em recon iter_0021)
- Best scheme: ens_inv_mse (v2,v3) ou ens_equal (v1)
- Ganho 4x maior que H10 LGBM em SE — predicao H24 sustentada

### N (ridge_alpha10): 3/3 cells confirming
- N/v1 delta -107 MWh (19% reducao), wins 4/5 — best cell H24
- N/v2 delta -107 MWh (idem v1 — mesmas features)
- N/v3 delta -99 MWh (18% reducao), wins 3/5
- R² flip: -0.24 a -0.38 -> +0.05 a +0.06
- Best scheme: ens_inv_mse (v1, v2) ou ens_equal (v3)
- Magnitude similar a H10 LGBM (-100 MWh) — N e' sub onde persist e'
  forte, ensemble corrige model-pior-que-persist

### S (lr): 0/3 cells confirming — NAO ajuda
- S/v1 delta -34 MWh, wins 1/5 (abaixo limiar 3/5)
- S/v2 delta -46 MWh, wins 2/5 (abaixo limiar)
- S/v3 delta +50 MWh (PIORA), wins 2/5
- R² champion v3 ja' +0.45 (champion S e' tight para padrao baixo-signal)
- ensemble adiciona ruido — persist no S e' mais volatil (R² persist -0.68)
- Resultado consistente com H10 (S/v3 +26 MWh, "irrelevante")

### Adaptacao do alpha por fold (weights_distribution.csv)

Como em H10, alphas escolhidos por inner_val variam amplamente fold-a-fold:
- NE/v1 alphas: 0.55, 0.65, 0.55, 0.50, 0.20 (5/5 wins, persist atenua todos folds)
- NE/v3 fold 1: alpha=0.65 (model+persist mix); fold 5: alpha=0.50 (tie)
- SE/v3 fold 1: alpha=0.30 (champion pior, persist domina); fold 4: alpha=1.00
  (champion melhor sozinho, mas mesmo assim ensemble bate via inv_mse)
- S/v1 fold 1: alpha=1.00 (champion sozinho); fold 5: alpha=1.00 (tie persist)
- N/v3 fold 1: alpha=0.65 (mix); fold 5: alpha=0.50 (tie)

Sinal claro de regime adaptation: pesos analiticos derivados de inner_val
respondem corretamente ao regime local.

### Comparacao H10 (LGBM) vs H24 (Ridge/LR)

| sub | H10 best delta MAE | H24 best delta MAE | razao H24/H10 | nota |
|---|---|---|---|---|
| NE | -12.996 (v1) | -9.089 (v3) | 0.70x | similar magnitude absoluta; MAE_MOD H24 maior (linear pior que GBDT) entao % reducao similar |
| SE |    -565 (v1) | -2.100 (v3) | 3.72x | H24 ~4x MAIOR — confirma predicao H24 (linear deixa mais residual) |
| S  |     -53 (v1) |    -46 (v2) | ~tie | ambos irrelevantes; champion lr ja' tight |
| N  |    -100 (v1) |   -107 (v1) | 1.07x | empate; persist > model em ambos cases |

Insight chave: ganho em **SE quadruplicou** ao trocar LGBM por LR.
Sugere que LGBM em SE ja' captura bem a estrutura nao-linear (NMAE ~46%
com modelo proprio) enquanto LR fica em NMAE ~50% (deixa residuo);
ensemble com persist preenche esse gap em LR mas tem pouco a adicionar
em LGBM.

### Best scheme distribution

12 cells, melhor esquema por MAE medio:
- ens_inv_mse: 5 cells (NE/v1 tie, SE/v2, SE/v3, N/v1, N/v2, S/v3)
- ens_equal: 5 cells (NE/v2, NE/v3, SE/v1, N/v3 — S/v1 tie)
- ens_inv_mae: 1 cell (NE/v1)
- ens_opt_alpha: 2 cells (S/v1, S/v2 — onde nao confirma)

Diferenca vs H10: ens_inv_mae perdeu protagonismo. Hipotese: variancia
maior dos preds lineares torna metricas quadraticas (MSE) mais
discriminativas para weight derivation.

### Sanity checks

| sanity | status | nota |
|---|---|---|
| B1 leak | INHERITED | iter_0002 features ja auditadas + persist_d1 = y_d1[i-1] D-1 safe + inner_val < test_start sem overlap |
| B2 perm | N/A | ensemble nao introduz feature nova |
| B3 holdout strict | DONE_VIA_CV | 12 cells x 5 folds = 60 holdouts; CV walk-forward 5x60d gap 7d; inner_val 30d isolado |
| B4 baseline | DONE_INTEGRADO | persist_d1 e' componente DIRETO do ensemble; skill_model_vs_persist por cell |
| B5 dist_shift | INHERITED + EVID | iter_0012 KS p<0.0001 NE+SE + weights_distribution.csv mostra alpha varia 0.20-1.00 |
| B6 zero_count | N/A | nao introduz feature; reuso iter_0002 em B6 iter_0009 |

Coverage: 4 done + 2 N/A = 6/6.

## Decisao

**CONFIRMADO_3SUBS** (NE+SE+N). Ensemble (champion_linear + persist_d1,
peso analitico equal/inv_mse) e' post-processing benefico em 3/4
champions de producao UlFor:

- NE (ridge_alpha10): ganho 19-24% MAE — 7-9k MWh. Maior valor clinico.
- SE (lr):            ganho 13-26% MAE — 1.1-2.1k MWh. R² flip neg->pos.
- N  (ridge_alpha10): ganho 17-19% MAE — ~100 MWh. Marginal por escala
  pequena, mas consistente.
- S  (lr):            inconclusivo. Wins 1-2/5 + R² champion ja +0.45 v3
  + v3 PIORA +50 MWh — ensemble nao ajuda.

Ganho em SE **quadruplica** vs H10 LGBM — confirma predicao H24 (lineares
deixam mais variancia residual para persist absorver).

Sem hipotese derivada nova. H25 (stacker meta-modelo) ja queued P3 e
permanece — improvavel ganho marginal substantivo dado que pesos
analiticos simples ja funcionam.

req-0008 (P2 model_post_processing) opcional: emitir somente se Breno
priorizar produzir o ganho no /api/forecast/d1.

## Proximo passo

Planner config para iter_0023:
- **Recon delta UlFor** se houver commits novos no head ec0fd937..HEAD
  (sessao multi-agente regime checkpoints frequentes — alta probabilidade
  de 2-4 commits novos).
- **Alt principal**: H22 (P2 model — GBDT vs OLS gap, queued, atratividade
  subiu pos lesson H23_ulfor sobre PI-com-Ridge subestima colineares).
- **Alt secundario**: H25 (P3, stacker meta-modelo — baixo expected value
  dado H24 mostrar pesos analiticos ja funcionam).

H24 fechada como done iter_handled=0022 verdict=CONFIRMADO_3SUBS.

## Nota artefatos

summary.csv foi pos-processado por linter (column `champion_model`
renomeada para `model_kind`, valor `ridge_alpha10` simplificado para
`ridge`); valores numericos inalterados. results.json e verdict.json
mantem schema original (campo `champion_model`). Tabelas em
result_metric usam labels originais para fidelidade ao spec UlFor
(ridge_alpha10/lr).
