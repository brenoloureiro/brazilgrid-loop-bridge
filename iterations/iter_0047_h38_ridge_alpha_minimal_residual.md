---
alvo: ridge_alpha_minimal_residual
layer: curtailment
iter_num: 0047
type: hypothesis_test (verdict=INDETERMINADO_ALPHA_LOW)
data_utc: 2026-05-26T10:00:00Z
hypothesis: |
  H38 (P4, queued desde iter_0040 — derivada de H30 REFUTADO_RIDGE com nota
  explicita de baixo impacto pratico): "Alpha < 1 salva NE no set minimal
  residual?"

  H30 (iter_0040) testou ALPHAS=[1.0, 10.0] em sets minimais:
    A = [ger_renovavel, pdp_prev_eolica, pdp_prev_solar] (3 feat, basis bruto)
    B = [ger_renovavel, pdp_residual_total] (2 feat, residual collapsed)
    C = [ger_renovavel, pdp_residual_eolica, pdp_residual_solar] (3 feat, split)

  Verdict H30: REFUTADO_RIDGE. NE/v3 alpha=1 set B passou marginal
  (delta_R²=+0.013 vs A, threshold -0.005); alpha=10 destruiu (-0.064).
  SE falhou em ambos alphas. C catastrofico em SE (-0.47 a -0.88).

  Conjectura H38 (mecanistica): residuals tem std reduzido vs basis bruto
  (NE pdp_residual_e std ~10k MWh vs pdp_prev_e std ~40k MWh = 4x menor).
  Ridge shrinkage e' aplicado em escala absoluta pos-StandardScaler, mas
  pequenas variances tem menos sinal para "puxar" do prior zero — alpha
  grande colapsa coef residual em zero, destruindo informacao. Alpha
  pequeno (<1) preserva sinal. Custo trivial: mesmo loop H30, expandir
  ALPHAS=[0.01, 0.1, 1.0, 10.0].

  Aceitacao H38 (a priori):
    CONFIRMADO_ALPHA_LOW   : delta_R²_B >= -0.005 em NE AND SE para algum
                             alpha < 1 (ie. Ridge minimal funciona com tuning)
    INDETERMINADO_ALPHA_LOW: 1 sub salva mas outra falha
    REFUTADO_ALPHA_LOW     : nenhum alpha < 1 salva ambos subs

  Impacto pratico EXPLICITAMENTE BAIXO (P4): mesmo se CONFIRMADO em
  alpha=0.01, ganho marginal nao justifica swap operacional. Hipotese
  existe para FECHAR caminho tecnico "alpha-sensitivity foi a culpa" e
  nao deixar caveat aberto.

baseline_tipo: |
  Replay-only — sem feature engineering nova, sem novo target, sem novo
  split, sem novo modelo. So' expansao do sweep alpha sobre o mesmo
  protocolo CV walk-forward 5x60d gap 7d de H30 iter_0040.
  Champion UlFor Ridge_alpha10 NE (iter_0007 NMAE_CV 33.7%) NAO ESTA
  em questao — esta hipotese roda em features minimais explicitamente
  para isolar mecanismo de shrinkage em set restrito, nao para
  questionar o champion (que usa features completas).

protocolo:
  cells: [NE/v3, SE/v3]
  source: outputs/iter_0002/runs/{NE,SE}/v3/features.parquet
  cv: 5 folds walk-forward, test 60d cada, gap 7d, min_train=60, min_test=5
  feature_sets:
    A: [ger_renovavel_mwh, pdp_prev_eolica_mwh, pdp_prev_solar_mwh]
    B: [ger_renovavel_mwh, pdp_residual_total_mwh]
    C: [ger_renovavel_mwh, pdp_residual_eolica_mwh, pdp_residual_solar_mwh]
  derived_cols:
    pdp_residual_total_mwh: (pdp_prev_eolica + pdp_prev_solar) - (ger_eolica + ger_solar)
    pdp_residual_eolica_mwh: pdp_prev_eolica - ger_eolica
    pdp_residual_solar_mwh: pdp_prev_solar - ger_solar
  model: Ridge(alpha) com StandardScaler + Pipeline + random_state=0
  alphas: [0.01, 0.1, 1.0, 10.0]
    focus: [0.01, 0.1]      # alphas que TESTAM a hipotese
    replay: [1.0, 10.0]     # alphas que REPLICAM H30 (sanity)
  metric: MAE/R²/F1_p50/NMAE/bias via metric_suite v1.0
  paired_deltas: B-A e C-A por fold (R² + MAE)
  perm_test: feature residual target na fold final por alpha x fs (n_perm=30)

verdict: INDETERMINADO_ALPHA_LOW

resultados_principais:
  replay_H30_sanity_bit_exato: |
    Todos os 4 valores chave de H30 iter_0040 reproduzidos ate' 5 casas
    decimais — confirma reproducibilidade do protocolo e do split CV:
      NE alpha=1  delta_r2_B = +0.01275 (iter_0040 = +0.012750418...)
      NE alpha=10 delta_r2_B = -0.06419 (iter_0040 = -0.064193647...)
      SE alpha=1  delta_r2_B = -0.03217 (iter_0040 = -0.032170746...)
      SE alpha=10 delta_r2_B = -0.04313 (iter_0040 = -0.043129694...)

  alpha_sweep_set_B_NE:
    alpha=0.01: delta_r2_B = +0.0242  wins_r2 = 5/5  delta_mae_pct = -1.17%
    alpha=0.1 : delta_r2_B = +0.0231  wins_r2 = 4/5  delta_mae_pct = -1.11%
    alpha=1.0 : delta_r2_B = +0.0128  wins_r2 = 2/5  delta_mae_pct = -0.60%
    alpha=10  : delta_r2_B = -0.0642  wins_r2 = 1/5  delta_mae_pct = +3.24%
    leitura  : monotonico decrescente em alpha; alpha<1 PASSA threshold
               -0.005 com folga (NE responde a hipotese mecanistica).

  alpha_sweep_set_B_SE:
    alpha=0.01: delta_r2_B = -0.0309  wins_r2 = 1/5  delta_mae_pct = +0.94%
    alpha=0.1 : delta_r2_B = -0.0310  wins_r2 = 1/5  delta_mae_pct = +0.95%
    alpha=1.0 : delta_r2_B = -0.0322  wins_r2 = 1/5  delta_mae_pct = +1.01%
    alpha=10  : delta_r2_B = -0.0431  wins_r2 = 1/5  delta_mae_pct = +1.52%
    leitura  : pouca sensibilidade a alpha; TODOS abaixo do threshold
               -0.005 por margem ~5x; sinal SE estruturalmente nao
               responde a shrinkage tuning no set residual collapsed.

  alpha_sweep_set_C_split:
    NE alpha=0.01: delta_r2_C = +0.0153   (passa threshold)
    NE alpha=0.1 : delta_r2_C = +0.0142   (passa)
    NE alpha=1.0 : delta_r2_C = +0.0041   (passa marginal)
    NE alpha=10  : delta_r2_C = -0.0618   (refuta)
    SE alpha=0.01: delta_r2_C = -0.8801   (catastrofico)
    SE alpha=0.1 : delta_r2_C = -0.8741
    SE alpha=1.0 : delta_r2_C = -0.8177
    SE alpha=10  : delta_r2_C = -0.4691
    leitura  : C split mostra MESMA estrutura de B em NE; em SE pior.
               Em SE alpha=10 e' "menos catastrofico" que alpha=0.01
               (inversao vs NE) — possivel artefato de coef colineares
               em pdp_residual_e vs pdp_residual_s (cada um carrega
               sinal ~igual em SE; shrinkage forte amortece os 2 juntos).

  decisao_per_queue_spec:
    confirming_low_alphas_B = []   # nenhum alpha<1 passa em AMBOS subs
    NE_pass_only = [ridge_alpha0.01, ridge_alpha0.1]
    SE_pass_only = []
    => INDETERMINADO_ALPHA_LOW (algum alpha<1 salva 1 sub mas falha outra)

mecanismo_post_hoc: |
  NE confirma a conjectura mecanistica em escala monotonica em alpha
  (delta_r2_B monotonico decrescente de +0.024 em alpha=0.01 ate
  -0.064 em alpha=10). Diagnostico leak (frac_negative=0.975, residual_total
  mean=-99936 MWh) coerente com PDP cobrir subset de usinas << ger total
  subsistema: o "residual" e' DOMINANTEMENTE o gap de cobertura, nao um
  erro de previsao — entao shrinkage forte que zero-iza coef residual
  basicamente joga fora a info de gap de cobertura, que tem variance
  pequena mas correlacao real com curt residual.

  SE rejeita a conjectura: delta_r2_B varia de -0.031 a -0.043 em todo
  o sweep (faixa de 1.2pp), mas SEMPRE abaixo de -0.005 threshold por
  >= 5x. Diagnostico leak SE (frac_negative=0.188, residual_total
  mean=+11877) indica que residual SE tem qualidade diferente — PDP
  SE cobre fracao maior de ger SE, residual realmente captura
  over/under-prediction e nao gap de cobertura. Nesse regime, juntar
  pdp_residual com ger_renovavel no set B colapsa info que A
  ja distinguia (pdp_prev_e e pdp_prev_s separados), e Ridge nao
  consegue "des-colapsar" via tuning de shrinkage — e' perda de
  representacao basis, nao de regularizacao.

  Implicacao mais geral: residual_total como feature LINEAR (sob qualquer
  shrinkage L2) e' boa em NE (porque encoda gap cobertura, info nova)
  e ruim em SE (porque encoda erro pdp_prev, info que A ja tem em forma
  separada melhor). C split (residual por fonte) MANTEM a vantagem NE
  mas amplifica o problema SE (introduz colinearidade entre 2 residuais
  separados sem informacao independente o suficiente para Ridge separar).

sanity_checks:
  B1_leak:
    status: PASS_DIAG
    note: |
      Replicacao do leak_diagnostic H30. Todos residuals computados
      D-1-safe (pdp_prev publicado D-1 manha; ger_e/ger_s observavel
      fim do D; ambos antes de predizer D+1).
    NE_residual_total: {nan: 0, inf: 0, mean: -99936, std: ~, frac_negative: 0.975}
    SE_residual_total: {nan: 0, inf: 0, mean: +11877, std: ~, frac_negative: 0.188}

  B2_perm:
    status: REPORTED
    note: |
      Permutacao da feature residual target (PDP_RES_TOTAL em B,
      PDP_RES_E em C) na fold final por alpha x fs, n_perm=30. Resultados
      em sanity_summary.json bloco "perm". Esperado: importance > 0 em
      NE (sinal real), importance ~0 em SE alpha<1 (sinal nao usado pelo
      modelo na escala baixa), importance > 0 em alpha=10 SE (modelo
      depende mais do residual quando shrinkage homogeneiza).

  B3_holdout:
    status: PASS_EMBEDDED
    note: |
      CV walk-forward 5 folds com gap=7d, test 60d cada, train ate
      train_cutoff = test_start - 7d. Sem leak temporal. Identico H30.

  B4_baseline:
    status: PASS_EMBEDDED
    note: |
      persist_d1 floor reportado por fold em mae_per_vals.
      NE persist MAE mean = 33837; SE persist MAE mean = 8328.
      Skill A vs persist NE = -14% (A piora vs persist -- modelos
      residual em set minimal nem batem persist); SE = +7.2%.
      Mostra que mesmo o BEST H38 (NE alpha=0.01 set B MAE 38102)
      ainda PERDE para persist (33837) por 12.6% em NE. Reforca
      "impacto pratico nulo" do P4.

  B5_dist_shift:
    status: ANNOTATED_REUSE
    source: iter_0012 H7
    note: KS test y_train vs y_test p<0.0001 NE+SE (causa raiz iter_0012)

  B6_zero_count:
    status: REPORTED
    NE: {n_lt_1mwh: ~9 dias, frac: 0.020, y_mean: ~98k MWh, y_median: ~70k}
    SE: {n_lt_1mwh: ~2 dias, frac: 0.005, y_mean: ~14k MWh, y_median: ~8k}

  B6_n_test:
    status: PASS
    note: 5 folds * 60d = 300 obs por cell. Threshold H20 = 30. Passa por 10x.

requests_externos: []
  # NAO houve necessidade de pedir dado novo a UlFor. Hipotese roda 100%
  # local sobre iter_0002/runs/{NE,SE}/v3 ja existentes.

champions_status:
  ridge_alpha10_NE: INTACTO  # H38 nao testa champion (champion usa 47 feats)
  lr_SE: INTACTO              # H38 testa NE/SE em features minimais isoladas
  lr_S: INTACTO               # S nao testado em H38 (segue exclusao H30/H21)
  ridge_clean_plus_N: INTACTO # N nao testado em H38 (cobertura PDP zero)
  bias_correction_productized: INTACTO
  rollback: NONE

artefatos:
  - outputs/iter_0047/h38_ridge_alpha_minimal_residual/results.json
  - outputs/iter_0047/h38_ridge_alpha_minimal_residual/summary.csv
  - outputs/iter_0047/h38_ridge_alpha_minimal_residual/sanity_summary.json
  - outputs/iter_0047/h38_ridge_alpha_minimal_residual/verdict.json
  - scripts/h38_ridge_alpha_minimal_residual.py

follow_ups_created: []
  # Arco residual ENCERRADO. H21 (OLS REFUTADO iter_0020) + H22 (3-feat
  # GBDT-vs-OLS TIE/WORSE iter_0034) + H30 (Ridge alpha=1,10 REFUTADO
  # iter_0040) + H38 (Ridge alpha [0.01..10] INDET NE-only iter_0047)
  # esgotam frente residual em curt D+1 no envelope iter_0002 + CV-5x60d.
  # "Per-sub tuning poderia salvar SE" e' epistemicamente possivel mas
  # P4 explicito nao reabre (impacto marginal + complexidade ops + arco
  # ja tem 4 hipoteses convergindo no mesmo veredito qualitativo).

custo:
  estimado_horas: 0.3
  real_horas: 0.3

leitura_curta: |
  H38 INDETERMINADO_ALPHA_LOW. Alpha sweep expandido para [0.01, 0.1, 1.0, 10.0]
  bate iter_0040 H30 BIT-EXATO em alpha=1+alpha=10 (replay sanity total).
  NE responde monotonicamente a alpha (delta_r2_B vai de +0.024 em alpha=0.01
  ate -0.064 em alpha=10, melhor que H30 +0.013 alpha=1); confirma mecanismo
  conjecturado (residuals tem std baixo, shrinkage forte zera-os). MAS SE
  nao responde a alpha (faixa estreita -0.031 a -0.043, todos abaixo do
  threshold -0.005 por >= 5x): residual SE encoda info ja capturada por A,
  perda de representacao basis nao corrigivel por tuning. FECHA caveat
  alpha-sensitivity de H30 com nuance NE-pass-marginal / SE-fail-estrutural.
  Impacto pratico nulo (P4): mesmo melhor NE alpha=0.01 ainda perde para
  persist por 12.6% MAE. Champions UlFor INTACTOS, 0 follow-ups, arco
  residual em curt D+1 (H21+H22+H30+H38) ENCERRADO.
---

# iter_0047 — H38 Ridge alpha-sensitivity em set minimal residual

## TL;DR

Verdict: **INDETERMINADO_ALPHA_LOW**.

Sweep `ALPHAS=[0.01, 0.1, 1.0, 10.0]` em set minimal residual sobre
o protocolo H30 (CV 5x60d gap7d, NE+SE v3, sets A/B/C). Resultados
chave:

| sub | alpha  | delta_r2_B | wins_r2 | passa_-0.005? |
|-----|--------|------------|---------|---------------|
| NE  | 0.01   | **+0.0242** | 5/5    | **SIM**       |
| NE  | 0.1    | +0.0231    | 4/5    | SIM           |
| NE  | 1.0    | +0.0128    | 2/5    | SIM           |
| NE  | 10     | -0.0642    | 1/5    | nao           |
| SE  | 0.01   | -0.0309    | 1/5    | nao           |
| SE  | 0.1    | -0.0310    | 1/5    | nao           |
| SE  | 1.0    | -0.0322    | 1/5    | nao           |
| SE  | 10     | -0.0431    | 1/5    | nao           |

Nenhum alpha < 1 salva ambos NE+SE => INDETERMINADO_ALPHA_LOW por
queue spec.

## Replay H30 bit-exato

Os 4 valores de iter_0040 reproduzidos ate' 5 casas decimais:

- NE alpha=1  delta_r2_B = +0.01275 (iter_0040 idem)
- NE alpha=10 delta_r2_B = -0.06419 (iter_0040 idem)
- SE alpha=1  delta_r2_B = -0.03217 (iter_0040 idem)
- SE alpha=10 delta_r2_B = -0.04313 (iter_0040 idem)

Reproducibilidade total — confirma split CV identico e Pipeline
estavel (mesmo random_state, mesmo Ridge solver, mesma ordem de cols).

## Mecanismo

**NE confirma**: delta_r2_B monotonico decrescente em alpha. Conjectura
"residuals com std baixo (~10k vs 40k MWh basis bruto) + shrinkage
absoluto pos-StandardScaler = alpha grande zera coef residual" VALIDA.
Diagnostico leak NE: frac_negative=0.975, mean residual_total=-99936
MWh — residual encoda DOMINANTEMENTE o gap de cobertura PDP (subset
usinas) vs ger total subsistema, info real com variance pequena.

**SE rejeita**: delta_r2_B varia apenas 1.2pp em todo o sweep e
SEMPRE fica abaixo do threshold por >= 5x. Diagnostico leak SE:
frac_negative=0.188, mean=+11877 MWh — residual SE captura
over/under-prediction PDP real, info QUALITATIVAMENTE diferente
do NE. Set B (residual collapsed) DEGRADA representacao que A ja
distinguia (pdp_prev_e + pdp_prev_s separados); Ridge nao
des-colapsa via tuning. Set C (split residual) introduz
colinearidade artificial entre 2 residuais com info nao
independente — Ridge amplifica o problema vs B.

## Impacto pratico nulo (P4 explicito desde queueing)

Best caso: NE alpha=0.01 set B MAE = 38102 MWh. Persist_d1 MAE = 33837.
**Modelo ainda PERDE para persist por 12.6%**. R² = -0.023 (sub-zero).
Champion UlFor Ridge_alpha10 NE com features completas (iter_0007)
entrega NMAE_CV 33.7% — set minimal residual nem chega perto. Esta
hipotese nao move agulha operacional alguma; existia para fechar
caveat tecnico "alpha-sensitivity foi a culpa de H30" — fechou com
nuance.

## Sanity checks (6 default)

- B1 leak: PASS_DIAG (residual D-1-safe, frac_negative coerente com
  cobertura PDP esperada)
- B2 perm: REPORTED (importance feature residual fold-final por
  alpha/fs em sanity_summary.json)
- B3 holdout: PASS_EMBEDDED (CV walk-forward 5 folds gap=7d)
- B4 baseline: PASS_EMBEDDED (persist_d1 floor reportado; best H38
  NE alpha=0.01 ainda perde para persist por 12.6%)
- B5 dist_shift: ANNOTATED_REUSE iter_0012 H7 (KS p<0.0001 NE+SE)
- B6 zero_count: REPORTED (NE 2.0%/SE 0.5% frac<1MWh) + n_test 5x60=300 > 30 PASS

## Conclusao do arco residual

H21 (OLS analitico iter_0020 REFUTADO) + H22 (3-feat GBDT-vs-OLS
iter_0034 TIE/WORSE) + H30 (Ridge alpha=1/10 iter_0040 REFUTADO) +
H38 (Ridge alpha [0.01..10] iter_0047 INDET NE-only) **convergem em
4 hipoteses** com mesmo veredito qualitativo: residual como feature
em set minimal NAO destrava ganho operacional vs basis bruto em
curt D+1 no envelope iter_0002 + CV-5x60d. Frente residual
**ENCERRADA**. 0 follow-ups.

## Proximo alvo

A definir pelo planner/consolidation. Queue ativo: H37 ja done,
H35/H39/H40/H5/H12/H14/H18 com restricoes (P5 doc ou bloqueado por
acao externa). Consolidacao ou rotacao para hipotese aberta P3 sao
caminhos possiveis.
