---
schema_version: 1
last_updated: 2026-05-24T19:30:00Z
notes: |
  Backlog auditavel. Loop le este arquivo antes de planejar cada iter.
  Editavel manualmente — Breno pode adicionar/repriorizar/declinar.
  Loop marca status=active ao selecionar, e status=done|declined|blocked
  ao final da iter.

  Convencoes:
  - id: monotonic crescente, prefixo H (hypothesis)
  - priority: P0 (must) > P1 (should) > P2 (could) > P3 (would)
  - status:
      queued    — disponivel para selecao
      active    — iter atual esta processando
      done      — concluida (iter_handled preenchido)
      declined  — rejeitada (motivo em ulfor_response ou notes)
      blocked   — bloqueada por dependencia externa (ex: req UlFor OPEN)
  - depends_on / blocks: ids de outras hipoteses ou req-NNNN do canal
  - expected_value: free-text justificando por que vale a pena
---

# Hypotheses Queue

hypotheses:
  - id: H1
    summary: Per-conjunto NE bate per-subsistema em granularidade
    detail: |
      Avaliada pela sessao UlFor (autopilot), nao pelo loop. UlFor commit
      c418e4ab marcou "H1 FALSIFICADA" — per-conjunto NE nao supera
      per-subsistema NE em NMAE D+1. Registrada aqui para fechar audit.
    type: methodology
    layer: curtailment
    target: curtailment_d1_NE_per_conjunto
    priority: P1
    status: done
    iter_handled: ulfor_external
    estimated_effort_hours: null
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: testar se desagregar adiciona skill
    created_at: 2026-05-23T20:00:00Z
    completed_at: 2026-05-23T22:30:00Z

  - id: H2
    summary: Off-by-one no JOIN feat_pdp_renovavel
    detail: |
      Iter 0003 refutou empiricamente: corr(PDP_prev[t], gen[t])=0.9118 e
      pico, corr(PDP_prev[t], gen[t+1])=0.8211. dat_programacao E o target
      day. JOIN correto.
    type: data_quality
    layer: curtailment
    target: feat_pdp_renovavel
    priority: P1
    status: done
    iter_handled: 0003
    estimated_effort_hours: 1.0
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: explicar regressao NE/v3
    created_at: 2026-05-24T00:00:00Z
    completed_at: 2026-05-24T02:30:00Z

  - id: H3
    summary: PDP carrega sinal alem de "previsao de geracao" via residual?
    detail: |
      Corr alta gen[t] vs PDP[t] (0.94 prog, 0.91 prev) sugere PDP e proxy
      quase-perfeito de geracao realizada. Investigar se PDP adiciona algo
      alem disso via residuals(curt ~ gen) ~ PDP. Se r2_extra > 0.05,
      PDP traz sinal de saturacao/curtailment alem de gen.

      VEREDITO iter_0010: CONFIRMADO. Teste OLS contemporaneo (curt,gen,PDP
      no mesmo D, n=486 dias, 2024-12-01 -> 2026-05-01) revelou separacao
      clara entre PDP_prev (previsao) e PDP_prog (programado):
        - NE/pdp_prev: r2_extra = +0.308 (perm p=0.0; train/test 0.339/0.340)
        - SE/pdp_prev: r2_extra = +0.251 (perm p=0.0; train/test 0.268/0.270)
        - S/pdp_prev:  r2_extra = +0.157 train mas COLAPSA test (regime change
          curt-S baixo + cobertura PDP-S so 12 usinas) — fragil
        - NE/pdp_prog: r2_extra = +0.115 (significativo mas inferior)
        - SE/pdp_prog: r2_extra = +0.005 (quase redundante com gen)
      Leak check: corr(pdp[t],curt[t]) > corr(pdp[t],curt[t-1]) em 5/6
      casos -> PDP forward-looking, sem leak. Mecanismo: discrepancia
      (pdp_prev - gen) = proxy direta de curtailment. Implicacao: manter
      pdp_prev_* (NE/SE definitivo, S condicional); pdp_prog_* candidato
      a drop por colinearidade ~0.95 com ger_renovavel.
    type: feature
    layer: curtailment
    target: feat_pdp_renovavel_residual
    priority: P2
    status: done
    iter_handled: 0010
    verdict: CONFIRMADO
    estimated_effort_hours: 1.5
    actual_effort_hours: 0.7
    depends_on: []
    blocks: []
    sanity_checks_required: [leak, perm, dist_shift]
    sanity_checks_done: [leak, perm, dist_shift]
    follow_ups_created: [H21, H22]
    expected_value: justificar manter PDP no modelo apos resolver gaps
    created_at: 2026-05-24T03:00:00Z
    completed_at: 2026-05-24T07:30:00Z

  - id: H4
    summary: B6 zero_count_shift sanity check
    detail: |
      Implementado em iter 0004. Detecta features com mudanca de zero-rate
      train->test + signal_collapse. Caught lag sign-flip em SE/v3.
    type: methodology
    layer: meta
    target: sanity_check_suite
    priority: P1
    status: done
    iter_handled: 0004
    estimated_effort_hours: 1.0
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: ferramenta diagnostica generica
    created_at: 2026-05-24T02:30:00Z
    completed_at: 2026-05-24T03:00:00Z

  - id: H5
    summary: feat_termico stale (>=2024-12-31) — quantificar impacto
    detail: |
      No CH local feat_termico nao recebe ingestao desde 2024-12-31. Iter 0002
      dropou termico features. Comparar bakeoff com vs sem feat_termico no
      UlFor (que tem termico fresco) para quantificar perda de skill.
      Se delta > 5pp NMAE, ingestao termico vira blocker.
    type: data_quality
    layer: entradas
    target: feat_termico_freshness_impact
    priority: P2
    status: blocked
    estimated_effort_hours: 1.0
    depends_on: [req-0004]   # request UlFor a criar
    blocks: []
    sanity_checks_required: [baseline]
    expected_value: prioritizar refresh termico se valer
    created_at: 2026-05-24T03:30:00Z

  - id: H6
    summary: SE/v3 colapso strict +19.2pp — sazonalidade ou amostra?
    detail: |
      B6 detectou sign-flip em curt_lag1/lag7 e ger_eolica_mwh. Sem n>=60d
      nao da pra distinguir entre regime change vs ruido amostral.
      Enviado ao UlFor como req-0003. RESPOSTA UlFor (iter_0006 extract):
      sign-flip NAO se confirma com n=60d gap=7d. Correlacoes test
      atenuam mas mantem direcao positiva (|corr_test|~0.1-0.2 vs +0.25
      train, dentro de variabilidade amostral n=60). VEREDITO: SE/v3
      PROMOVIVEL para FASE 4. Lags: ajudam em NE (+2.9pp se removidos),
      atrapalham em N (-2.3pp se removidos, lgbm vence), SE indiferente.
    type: methodology
    layer: curtailment
    target: SE_v3_lag_collapse
    priority: P1
    status: done
    iter_handled: 0006
    estimated_effort_hours: 0.0
    depends_on: [req-0003]
    blocks: []
    sanity_checks_required: []
    expected_value: decidir se SE/v3 promovivel
    created_at: 2026-05-24T03:00:00Z
    completed_at: 2026-05-24T04:50:00Z

  - id: H7
    summary: XGBoost vs LGBM com mesmo split — qual generaliza melhor?
    detail: |
      iter_0002 treinou ambos com defaults. Saidas mostram XGB melhor em
      alguns subs (NE/v3 R²=+0.52 vs LGB R²=-0.00) mas com test n=11.
      Validar com cross-validation se XGB e sistematicamente melhor que
      LGBM com features iter_0002.

      VEREDITO iter_0012: REFUTADO_LGBM_SYSTEMATICALLY_BETTER. CV walk-forward
      5 folds (60d cada, gap 7d) sobre features iter_0002 nas 12 celulas
      (4 subs x 3 vers): LGBM venceu MAE em 10/12 celulas (83%), XGB em
      1/12 (SE/v1 4/5 folds, mas magnitude -210 MWh em MAE ~7.5k = 2.8%
      clinicamente irrelevante), empate em 1/12. NE foi onde XGB perdeu
      pior: NE/v2 deltaMAE = +11.299 MWh, NE/v3 = +7.291 MWh em favor LGBM.
      iter_0002 NE/v3 XGB R²=+0.52 era ruido n=11 — CV mostra R² medio
      XGB=-0.18 (std 0.46) vs LGBM=+0.26 (std 0.30), inversao total. Vetor
      de degradacao: distribution shift (KS p<0.0001 em NE+SE entre fold1
      e foldN); XGB sofre mais shift que LGBM possivelmente por permitir
      splits mais profundos. Lesson reforca iter_0006/req-0003: test n=11
      e' falso positivo.

      Champions UlFor oficiais sao Ridge/LR (iter_0007/commit 4e0fc7b4),
      nao XGB nem LGBM. H7 e' diagnostica do replay loop, nao de producao.
    type: model
    layer: curtailment
    target: model_comparison_lgbm_xgb
    priority: P2
    status: done
    iter_handled: 0012
    verdict: REFUTADO_LGBM_SYSTEMATICALLY_BETTER
    estimated_effort_hours: 1.5
    actual_effort_hours: 0.9
    depends_on: []
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    sanity_checks_done: [holdout, baseline, dist_shift]
    follow_ups_created: []
    expected_value: escolher modelo certo para v4
    created_at: 2026-05-24T03:30:00Z
    completed_at: 2026-05-24T09:30:00Z

  - id: H8
    summary: Validar feat_intercambio importance via permutation
    detail: |
      iter_0002 PI mostrou val_net_mwmed_lag1 no top-5 de N/v1-v3 mas baixo
      em outros subs. Permutation rigorosa para confirmar se intercambio
      e feature critica (em N) ou ruido.

      ATUALIZADO iter_0021: PARCIALMENTE RESPONDIDA por UlFor H22 (VIF per-fold
      + PI per-fold, commit 10fa56d3). Drop candidates listados em
      FEATURE_DROPS_H22_PER_FOLD:
        - NE: val_export_mwmed (4/5 folds), val_import_mwmed (5/5), val_net_mwmed (5/5)
              -- VIF=1e8 (net = import - export, colineares perfeitos)
              + |PI|<0.5pp consistentemente. Drop candidate confirmado.
        - SE: val_export_mwmed (5/5), val_import_mwmed (3/5). Drop candidate.
        - S:  val_net_mwmed_lag1 (3/5, VIF=17.7, |PI|=0.119pp).
              Drop candidate em S (CONTRADIZ iter_0002 top-5 N — esse era
              regime-specific, n=11 inflado por colinearidade).
        - N:  NAO listado em FINDING_H22 N drop list -> val_net_* MANTEM em N.
              Consistente com iter_0002 top-5 N (sub onde intercambio carrega
              sinal economico real).
      Atratividade H8 nosso CAI: tecnica UlFor (VIF+PI conjunto) supera nossa
      PI puro proposta. Manter queued para confirmacao independente via
      CV-PI proprio se necessario.

      ATUALIZADO iter_0023: H22_model_aware (commit `2daa5d40`) preserva os
      mesmos drop candidates val_export/val_import/val_net em NE+SE (VIF=1e8
      colinearidade perfeita). N: val_net_* nao listado -> MANTEM em N. Sem
      mudanca na atratividade H8 nosso.
    type: feature
    layer: curtailment
    target: feat_intercambio_importance
    priority: P3
    status: queued
    estimated_effort_hours: 0.5
    depends_on: []
    blocks: []
    sanity_checks_required: [perm]
    expected_value: drop intercambio se confirmado ruido
    created_at: 2026-05-24T03:30:00Z
    last_external_update_iter: 0023

  - id: H9
    summary: NMAE substituido por MAE/R²/F1 (Principio 6 PLANO_FINAL)
    detail: |
      PLANO_FINAL UlFor Principio 6: "Para curtailment usar MAE/R²/F1,
      nunca NMAE/media". Iter 0002 usou NMAE — viola principio. Refatorar
      leaderboard + sanity checks B3/B4 para reportar MAE+R²+F1 (binarizada
      em "curt > P50") ao inves de NMAE. Manter NMAE como secundario.

      VEREDITO iter_0008: CONFIRMADO. Aplicar metric_suite sobre iter_0002
      LGBM replay revelou ranking conflicts em 3/4 subs (NE,SE,N) e NMAE
      'unsafe' em S (ymean_test<1 MWh => NMAE 109% reportado em iter_0006
      era artefato, nao sinal). NE/v2 vence em MAE mas F1=0 (cego a
      eventos high-curt). H10/H11 desbloqueadas. H19 derivada criada.
    type: metric
    layer: meta
    target: metric_suite
    priority: P1
    status: done
    iter_handled: 0008
    verdict: CONFIRMADO
    estimated_effort_hours: 1.0
    actual_effort_hours: 1.3
    depends_on: []
    blocks: [H10, H11]   # ensemble e quantile precisam de metric reformada
    sanity_checks_required: []
    expected_value: alinhar com PLANO_FINAL UlFor, evitar inferencia ruim
    created_at: 2026-05-24T03:30:00Z
    completed_at: 2026-05-24T06:00:00Z

  - id: H10
    summary: Ensemble v2_LGBM + persistencia_d1 weighted by skill
    detail: |
      Iter 0002 mostra persist_d1 baseline forte em N (vence ML). Ensemble
      simples (peso = skill score em CV) pode dominar v2 puro em subs onde
      persistencia carrega muito sinal. Testar em NE+SE.

      VEREDITO iter_0013: CONFIRMADO_NE_SE (+ bonus N). CV walk-forward
      5x60d gap 7d sobre features iter_0002 em 12 cells (4 subs x 3 vers),
      inner_val 30d para derivar pesos sem leak. 4 esquemas testados
      (ens_equal/inv_mae/inv_mse/opt_alpha). Best ensemble bate LGB-only
      em maioria das folds em:
        NE 3/3 cells (best delta -12.996 MWh em v1; -4.188 em v2/v3)
        SE 3/3 cells (best delta -565 MWh em v1)
        S  2/3 cells (v3 +26 MWh = 2.7% irrelevante)
        N  3/3 cells (best delta -100 MWh, corrige LGB-pior-que-persist)
      Esquemas vencedores por frequencia: ens_inv_mae 6 cells, ens_inv_mse
      4, ens_equal 2, ens_opt_alpha 0 (sobre-otimiza inner_val). Pesos
      analiticos (proportional-to-precision) > grid-search empirico em
      inner_val de 30d. alpha_opt varia 0.0-1.0 entre folds confirmando
      adaptacao a regime (NE/v1 fold 4: alpha=0.00 = pura persist quando
      LGB falha; NE/v3 fold 5: alpha=0.50 quando persist > LGB).
      Magnitude: NE 13-28% MAE reduction (v1 maior por LGB catastrofico);
      SE 4-7%; N 14-17%; S 0-13%. R² melhora em todas confirming cells
      (NE/v2 0.22->0.41, N/v1 -0.33->+0.01).
    type: model
    layer: curtailment
    target: ensemble_v2_persist
    priority: P2
    status: done
    iter_handled: 0013
    verdict: CONFIRMADO_NE_SE
    estimated_effort_hours: 1.0
    actual_effort_hours: 1.1
    depends_on: [H9]
    blocks: []
    sanity_checks_required: [baseline, holdout]
    sanity_checks_done: [baseline, holdout, dist_shift]
    follow_ups_created: [H24, H25]
    expected_value: ganhos baratos sem novo modelo
    created_at: 2026-05-24T03:30:00Z
    completed_at: 2026-05-24T10:30:00Z

  - id: H11
    summary: Quantile regression para incerteza P10/P50/P90 em NE
    detail: |
      LightGBM com objective=quantile (alphas 0.1, 0.5, 0.9). Substituir
      ponto-estimativa por bandas — util para downstream (operador escolhe
      P90 conservador). Bench worktree ja tem NGBoost similar.
    type: model
    layer: curtailment
    target: NE_d1_quantile_forecast
    priority: P2
    status: done
    iter_handled: 0014
    verdict: REFUTADO_NE
    verdict_summary: |
      Bandas LGBM quantile com defaults sistemicamente UNDERCOVERED:
      coverage_band_80 mean NE = 43.6% vs 80% nominal (0/3 NE cells in
      [70%, 90%]). Causa: LGBM nao modela heteroscedasticidade explicita +
      distribution shift documentado (iter_0012 KS p<0.0001 NE+SE). P50
      magnitude OK (delta NE +4.5% vs LGB-mean) mas banda inutil para
      caso de uso "P90 conservador" do operador. Bonus: P50 quantile BATE
      LGB-mean em N (-18%), S (-14%), SE (-1.6%) — mediana mais robusta
      que mean em distribuicoes com cauda longa de zeros.
    estimated_effort_hours: 2.0
    actual_effort_hours: 1.0
    depends_on: [H9]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    sanity_checks_done: [holdout, baseline, dist_shift, zero_count]
    follow_ups_created: [H26, H27, H28]
    expected_value: deliverable para v1.0 com bandas de confianca
    created_at: 2026-05-24T03:30:00Z
    completed_at: 2026-05-24T11:30:00Z

  - id: H12
    summary: feat_carga_history stale (parado em 2026-03-26) — refresh
    detail: |
      Stale local impede test n>=60d em janela 2026-04+. Request UlFor
      para refresh upstream. Loop sozinho nao pode resolver (= dbt run
      em pipeline ingestao).
    type: data_quality
    layer: entradas
    target: feat_carga_history_freshness
    priority: P1
    status: blocked
    estimated_effort_hours: 0.0
    depends_on: [req-0005]   # criar request UlFor proxima iter
    blocks: [H7, H10, H11]   # tudo que precisa test n>=60 local
    sanity_checks_required: []
    expected_value: destrava replays decentes no loop
    created_at: 2026-05-24T03:30:00Z

  - id: H13
    summary: Persist D-7 alem de persist D-1 como baseline secundario
    detail: |
      Iter 0002 ja calcula persist_d7. Promover para leaderboard como
      baseline auxiliar — em alguns regimes (S) persist_d7 vence persist_d1.
      Mudanca de display + interpretacao, nao re-treino.
    type: metric
    layer: meta
    target: leaderboard_baselines
    priority: P3
    status: queued
    estimated_effort_hours: 0.5
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: leaderboard mais honesto
    created_at: 2026-05-24T03:30:00Z

  - id: H14
    summary: Rodar sanity checks B1-B6 sobre v3.3 UlFor (PDP NE off)
    detail: |
      UlFor commit 055b9d03/e283a03a/c418e4ab evoluiu para v3.3 (PDP NE
      desabilitado, indicator pdp_has_data adicionado). Loop ainda nao
      auditou esse estado. Rodar pipeline B1-B6 sobre v3.3 outputs (quando
      UlFor disponibilizar predicoes em arquivo, ou processar req do canal).
    type: methodology
    layer: curtailment
    target: ulfor_v33_audit
    priority: P2
    status: blocked
    estimated_effort_hours: 1.5
    depends_on: [req-0006]   # request UlFor a criar (publicar v3.3 preds)
    blocks: []
    sanity_checks_required: [leak, perm, holdout, baseline, dist_shift, zero_count]
    expected_value: confirmar que v3.3 e promovivel para FASE 4
    created_at: 2026-05-24T03:30:00Z

  - id: H16
    summary: B6 zero_count_shift: ajustar threshold por n_test, evita falso positivo
    detail: |
      Iter_0006 mostrou que B6 deu falso positivo em iter_0004 (SE/v3 lag
      sign-flip) por causa de n_test=11 muito pequeno. UlFor re-rodou com
      n=60 e correlacoes nao flipam — direcao mantida, magnitudes apenas
      atenuam. Adicionar a B6: se n_test < 30, downgrade severity em 1
      nivel (high -> medium, medium -> low). Ou seguir threshold absoluto
      em |corr_train|, |corr_test| >= 0.2 antes de marcar sign_flip.
      Inclui regression test sintetico com n=10 e n=60 mostrando diferenca.

      VEREDITO iter_0009: CONFIRMADO. Ambas mitigations implementadas:
        (1) `n_test < 30` -> downgrade severity 1 nivel (severity_raw +
            flag downgraded_due_to_small_n_test preservados).
        (2) sign_flip exige |corr_train|>=0.2 AND |corr_test|>=0.2 (era
            >0.05 — overconfident em n=11).
      Regression sintetico (DGP fraco-positivo, n_train=365, 20 seeds):
        - n_test=10: 5/20 falsos positivos de sign_flip eliminados, 20/20
          severities downgrade-adas (medium->low ou high->medium).
        - n_test=60: 0/20 sign_flips perdidos (zero impacto em VP), zero
          downgrades aplicados (corretamente — n_test>=30).
      Revalidation iter_0002 runs: SE/v3 lag sign-flip downgrade high->medium,
      curt_lag7 sign_flip bloqueado pelo novo gate de |corr|, 9 features
      em SE/v3 downgrade-adas. NE/v1-3 curt_lag1/lag7 sign_flips persistem
      (|corr_train|=0.66/0.42, gate novo aceita — sao warnings legitimos).
    type: methodology
    layer: meta
    target: sanity_check_b6_robustness
    priority: P1
    status: done
    iter_handled: 0009
    verdict: CONFIRMADO
    estimated_effort_hours: 0.5
    actual_effort_hours: 0.7
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: evitar req desnecessario ao UlFor por falso positivo
    created_at: 2026-05-24T05:00:00Z
    completed_at: 2026-05-24T06:45:00Z

  - id: H17
    summary: Promover SE/v3 + S/v3.3 PDP-fixed para FASE 4 (SUPERSEDED)
    detail: |
      Premissa original: promover SE/v3 + S/v3.3 XGB para FASE 4.
      VEREDITO iter_0007: SUPERSEDED_BY_ULFOR_RIDGE_LR_CV.
      UlFor self-actionou entre iter_0006 e iter_0007 via commits 76732289
      (Ridge baseline-controle) + 4e0fc7b4 (CV walk-forward 5 folds).
      Champions mudaram em 3/4 subs:
        NE: xgb 35.7% -> ridge 33.7%CV  (R² +0.402 -> +0.469)
        SE: xgb 46.0% -> lr 46.6%CV     (R² +0.386 -> +0.380)
        S:  xgb 109%  -> lr 89.6%CV     (R² -0.135 -> +0.447, +0.58 abs!)
        N:  lgbm 72%  -> ridge 86.3%CV  (FRAGIL, std 31.8%, nao promovivel)
      Loop NAO emite req-0004 (redundante — UlFor ja' executando
      @champion registry plan, OOT 2x agendado).
      Follow-up: H18 (sanity B1-B6 sobre champions Ridge/LR).
    type: model
    layer: curtailment
    target: fase_4_promote_SE_S
    priority: P0
    status: done
    iter_handled: 0007
    verdict: SUPERSEDED_BY_ULFOR_RIDGE_LR_CV
    estimated_effort_hours: 0.0
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: deliverable real do UlFor para producao
    created_at: 2026-05-24T05:00:00Z
    completed_at: 2026-05-24T05:30:00Z

  - id: H18
    summary: Auditar champions Ridge/LR pos-OOT via sanity B1-B6
    detail: |
      Derivada de H17 SUPERSEDED. Quando UlFor publicar predictions de
      ridge_NE + lr_SE + lr_S em parquet acessivel (ou via req-0005 a
      criar), loop roda pipeline B1-B6 completo:
        B1 leak_detection — confirmar sem vazamento target em features
        B2 permutation_importance — top features Ridge/LR per sub
        B3 holdout_temporal_strict — gap 7d + janela OOT alternativa
        B4 baseline_compare — vs persist_d1 e ma7 por fold
        B5 distribution_shift — PSI train vs OOT
        B6 zero_count_shift — atualizado para downgrade severity n_test<30 (H16)
      Acceptance: se 5/6 passam clean, loop sinaliza GO para FASE 4
      (model serializer + drift monitor); se >=2 falham, request investigacao.
      Considerar tambem multicolinearidade (VIF) sugerida por UlFor — pode
      ser nova H19.

      ATUALIZADO iter_0011: champions PROMOVIDOS no MLflow Registry
      (commit 83bc79c2) + endpoint /api/forecast/d1 LIVE (commit a7edb1ef).
      VIF (commit 3ac5916a) ja' coberto por UlFor — H19_VIF nao foi
      criada. Bloqueio req-0005 segue: sanity local requer (a) UlFor
      publicar predicoes em parquet ou (b) acesso MLflow tracking URI
      do loop. Loop NAO emite req-0004 nesta iter (auto-pesado vs valor
      incremental: FINDING_MULTICOLINEARITY + FINDING_LR_N_INSTABILITY
      ja' resumem o que B1-B6 reconfirmaria).

      ATUALIZADO iter_0015: urgencia DIMINUI. UlFor fechou Fase 4 inteira
      (validate_d1.py + drift PSI nativo + Telegram alert + Dagster
      schedule daily 07h BRT — commits c0e80193, 9d7652de, 9873e3c8).
      Pipeline da cobertura empirica continua dos champions com 4 gatilhos
      (skill<0, R²<0, psi_recent_max>1.0, n_feat_drift>10), reduzindo
      valor marginal do B1-B6 audit local. Bloqueio req-0005 continua
      mas pressao caiu. Schedule STOPPED ate Breno gerar token Telegram
      + ativar Dagster UI.

      ATUALIZADO iter_0019: status_change blocked -> blocked-acao-Breno-trivializa.
      UlFor exposicao de mlflow.brazilgrid.com via Cloudflare Access (commit
      cd12cf2a): nginx vhost pronto no EC2 (proxy_pass 127.0.0.1:5000),
      MLflow systemd ja localhost-only, smoke local OK. Acao Breno pendente:
      2 passos manuais no dash Cloudflare (DNS A record + Zero Trust Access
      Application replicando policy clickhouse). Quando ativo, loop ganha
      caminho alternativo a req-0005: pode usar
      mlflow.client.MlflowClient(tracking_uri="https://mlflow.brazilgrid.com")
      via CF Access token e baixar predicoes/artefatos direto do MLflow REST.
      Elimina dependencia em UlFor publicar predicoes em parquet -- o req-0005
      vira opcional. Docs em docs/infraestrutura/SUBDOMINIOS.md +
      MLFLOW_CLOUDFLARE_SETUP.md.
    type: methodology
    layer: curtailment
    target: ridge_lr_champion_audit_pre_fase4
    priority: P1
    status: blocked
    estimated_effort_hours: 1.5
    depends_on: [req-0005]  # publicar predicoes ridge_NE+lr_SE+lr_S em parquet
    blocks: []
    sanity_checks_required: [leak, perm, holdout, baseline, dist_shift, zero_count]
    expected_value: garantir que champion novo nao tem vies escondido antes FASE 4
    created_at: 2026-05-24T05:30:00Z

  - id: H19
    summary: Extrair MAE/R²/F1 dos champions UlFor Ridge/LR para leaderboard
    detail: |
      Derivada de H9 iter_0008. Pos-adocao do metric_suite (MAE/R²/F1
      primarios), o leaderboard ainda mostra NMAE como metrica de
      champions (33.7% NE ridge, 46.6% SE lr, 89.6% S lr) porque UlFor
      publicou so' NMAE no checkpoint. MLflow `bakeoff-curtailment-d1`
      tem 140 runs + 28 CV_SUMMARY com mae/rmse/r2 — extrair para
      preencher coluna "best_metric" do leaderboard com MAE/R²/F1
      consistente.

      Implementacao: ou req-0007 ao UlFor pedindo dump do CV_SUMMARY
      em parquet acessivel, ou parser direto do MLflow proxy file que
      UlFor commitou (verificar coordination ou worktree leitura via
      git show). Loop pode ler via git show sem alterar nada.

      Sem isto, leaderboard fica inconsistente: NE champion = "ridge
      NMAE 33.7%" mas iter_0002 replay LGBM em NE/v2 mostra NMAE 28.2%
      sem R²/F1. Conclusao "ridge melhor" depende de comparar metricas
      identicas.

      ATUALIZADO iter_0015: atratividade CRESCE. UlFor adicionou
      validate_d1.py (commit c0e80193) que loga daily summary JSON com
      MAE/RMSE/NMAE/R²/skill_vs_persist + tabela de predicoes por sub
      no MLflow experiment `ulfor-validation-d1`. Quando Breno ativar
      schedule (07h BRT daily) + loop ganhar acesso a summary JSON via
      filesystem ou MLflow URI, custo H19 cai de "parse MLflow proxy
      file" para "parse summary JSON daily". Manter P2; eligible para
      promover quando schedule estiver LIVE.

      VEREDITO iter_0016: CONFIRMADO_PARCIAL. Loop extraiu via parser
      regex sobre FINDING_RIDGE_BEATS_GBDT.md + leitura literal de
      promote_champions.py:CV_METRICS_BY_FS (commit-acessivel sem
      MLflow tunnel). MAE em MWh derivado por
      `NMAE_champion × ymean_test_estimated`, onde
      `ymean_test_estimated = MAE_persist_iter0013_mwh / NMAE_persist_FINDING`.
      Resultado: NE 25.5k MWh / SE 5.6k / S 916 / N 429. Consistency
      check NMAE_persist FINDING vs state.json: 4/4 subs OK
      (0.445/0.688/1.242/1.007). R² direto do CV_METRICS_BY_FS:
      NE +0.469 / SE +0.380 / S +0.447 / N +0.179. Caveat 1a ordem
      (~10-15%) — ymean varia entre folds. **F1_p50 NAO DISPONIVEL**:
      nenhuma fonte UlFor computa. req-0007 emitido para proximo CV
      bake-off logar F1_p50 + dump per-fold MAE em parquet acessivel.
      Leaderboard reescrito em iter_0016 com MAE/R²/F1(N/A)/NMAE
      consistentes. Status reflexivo: closed in spirit (extracao
      bem-sucedida no que era acessivel; F1 atrasado por req externo).
    type: metric
    layer: meta
    target: leaderboard_consistency_post_h9
    priority: P2
    status: done
    iter_handled: 0016
    verdict: CONFIRMADO_PARCIAL
    estimated_effort_hours: 1.0
    actual_effort_hours: 0.8
    depends_on: []
    blocks: []
    sanity_checks_required: []
    sanity_checks_done: [baseline]
    follow_ups_created: []
    related_external_request: req-0007
    external_unblock_note: |
      ATUALIZADO iter_0017 RECON_DELTA: req-0007 DONE (UlFor commit 6b21ffdf).
      F1_p50 + per-fold parquet publicados em
      experiments/bakeoff_curtailment_multisub/outputs/cv_summary_per_fold_{full,clean_plus}.parquet
      (140 rows cada = 4 subs * 7 modelos * 5 folds, schema MAE/RMSE/NMAE/bias/R²/
      F1_p50/ymean_test/threshold_p50/n_train/n_test/janelas).
      EXTRACT realizado em PHASE B do iter_0017_recon_delta.md — parquet lido,
      champions_metrics_consolidated em state.json reescrito com numeros EXATOS:
        NE ridge full:    MAE 27317±11920 / R² +0.469±0.098 / F1 0.808±0.170
        SE lr    full:    MAE  6117±  873 / R² +0.383±0.094 / F1 0.785±0.108
        S  lr    full:    MAE   805±  441 / R² +0.371±0.164 / F1 NaN (esperado, P50=0)
        N  ridge clean_plus: MAE 425± 149 / R² +0.170±0.185 / F1 0.790±0.048
      Slack derivacao iter_0016 confirmado dentro envelope: NE +7%, SE +9%,
      S -12%, N ~0%. state.json `verdict_post_req_0007_iter_0017=COMPLETED`.
      Verdict formal CONFIRMADO_PARCIAL preservado por auditoria (iter_0016
      entregou o acessivel naquele momento). Leaderboard linhas 4 champions
      atualizadas em iter_0017 com MAE_exato + F1_p50.

      MATERIALIZADO em iter_0025 (2026-05-24T20:30Z, manual por pedido Breno):
      leaderboard.md REESCRITO com estrutura canonica (Champions / Baselines /
      Overlay producao / Candidatos sucessores) e suite COMPLETA por sub:
      MAE / R² / F1_p50 / RMSE / NMAE / bias / skill_vs_persist_d1 / nmae_safe.
      Skill_vs_persist (1 - MAE_champion/MAE_persist) computado direto do
      parquet: NE +17.7%, SE +34.0%, S +34.5%, N +16.4% (todos POSITIVOS - 4/4
      subs champion bate persist em MAE no CV puro, independente de bias_corr).
      Historico do replay loop n=11 (iter_0002..0006) movido para secao
      "deprecada" no rodape para nao poluir leaderboard ativo. Pendencias
      catalogadas: (a) bias_corr CV per-fold (req-0008 candidato P3), (b) F1_p50
      em S NaN estrutural (P75 alt em H32 emergente), (c) holdout 14d real
      h22_per_fold (H31 emergente), (d) skill_ens_vs_persist H24 derivavel.
      Verdict permanece CONFIRMADO_PARCIAL (F1 em S e' NaN inerente do P50
      protocolo; iter_0025 nao destrava esse gap teorico).
    expected_value: leaderboard internamente consistente (MAE/R²/F1 em todas linhas)
    created_at: 2026-05-24T06:00:00Z
    completed_at: 2026-05-24T13:30:00Z

  - id: H20
    summary: Auto-flag bake-off com n_test < 30 — warning explicito no leaderboard
    detail: |
      Derivada de H16 iter_0009. Patched B6 evita falsos positivos de
      severity, mas o problema raiz e' bake-offs com janelas curtas
      (n_test=11 do replay iter_0002 vs n_test=60 do UlFor oficial). Tudo
      que entra no leaderboard com test n<30 deveria ter coluna "n_test"
      visivel e flag `low_confidence_n_test`. Hoje a coluna "sanity_ok"
      embute isso opacamente. Mudanca pequena de display + adicao no
      bake-off runner para escrever n_test no meta.json. Sem dep externa.
    type: meta
    layer: meta
    target: leaderboard_low_n_test_warning
    priority: P3
    status: queued
    estimated_effort_hours: 0.5
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: leaderboard auto-documentado para baixa confianca amostral
    created_at: 2026-05-24T06:45:00Z

  - id: H21
    summary: Feature derivada pdp_residual = pdp_prev_total - gen_renov (engineering)
    detail: |
      Derivada de H3 iter_0010. H3 confirmou que pdp_prev_* carrega sinal de
      curtailment alem de gen (r2_extra +0.31 NE, +0.25 SE). Mecanismo
      identificado: discrepancia entre previsao e gerado e' proxy direta de
      curt. Testar feature engineering explicita
      `pdp_residual_mwh = pdp_prev_total_mwh - gen_renov_mwh` no bake-off.

      Hipotese: 1 canal denso (residual) pode substituir os 2 canais brutos
      (pdp_prev_eolica + pdp_prev_solar) com mesma ou melhor performance,
      reduzindo dimensionalidade e colinearidade.

      Sanity: B1 leak (pdp_prev e D-1-safe, gen e D-only; usar como feature
      em D para predizer D+1 -> ok), B2 perm (importancia vs random shuffle),
      B4 baseline_compare.

      Bonus: testar se pdp_prog_* pode ser DROPADO sem perda (corr 0.93-0.95
      com gen -> quase redundante). Reducao de feature space sem dano.

      VEREDITO iter_0020: REFUTADO (OLS). V_residual_plus_gen (2 feats) perde para
      V_brutos (3 feats) em R² OLS full sample: NE -5.1pp (0.654 vs 0.705),
      SE -4.3pp (0.438 vs 0.481). Holdout 80/20 amplifica: NE 0.282 vs 0.550
      test (perda adicional -27pp). V_residual_split (3 feats: gen +
      residual_eolica + residual_solar) tambem perde -3pp NE / -4pp SE,
      confirmando que decomposicao per-fonte importa: agregar
      eolica + solar em 1 canal destroi sinal. Sanity B1+B2+B4 PASS em NE+SE
      (leak forward-looking, perm p=0.0, residual bate gen-only). Protocol
      identity check H3<->H21: R²_gen_only bate exato (|delta|<0.001) os 3 subs.
      Bonus: pdp_prog NAO_CONFIRMADO drop -- adiciona +5.9pp NE / +9.2pp SE
      em cima dos brutos (contradiz interpretacao H3 pairwise "redundante com
      gen"; multivariate prog conditional em pdp_prev ainda informa). Mecanismo:
      OLS sobre (gen, pdp_prev) ja' tem qualquer combinacao linear de (gen,
      residual) no seu span -- engineering nao expande, so' restringe. Para
      GBDT pode ser diferente (interacoes nao-lineares); H30 derivada testa
      em Ridge CV protocolo UlFor; H22 segue como teste para GBDT.
    type: feature
    layer: curtailment
    target: feat_pdp_residual_engineered
    priority: P2
    status: done
    iter_handled: 0020
    verdict: REFUTADO
    estimated_effort_hours: 1.0
    actual_effort_hours: 0.7
    depends_on: [H3]
    blocks: []
    sanity_checks_required: [leak, perm, baseline]
    sanity_checks_done: [leak, perm, baseline, holdout_temporal_strict]
    follow_ups_created: [H30]
    expected_value: feature mais densa + reducao de dim sem perda de skill
    created_at: 2026-05-24T07:30:00Z
    completed_at: 2026-05-24T16:45:00Z

  - id: H22
    summary: GBDT-only curt~gen+pdp para medir gap nao-linear vs OLS
    detail: |
      Derivada de H3 iter_0010. H3 usou OLS linear para r2_extra; GBDT
      (XGB/LGBM) provavelmente extrai sinal adicional via interacoes
      (pdp x gen, sazonalidade x pdp, regime x pdp). Treinar GBDT com
      apenas `gen + pdp_prev_eolica + pdp_prev_solar` (3 features) e
      comparar R² vs OLS no mesmo split temporal.

      Se gap GBDT vs OLS >= 5pp R², valida que ha interacoes nao-lineares
      relevantes -> bake-off completo deve continuar com GBDT (ja faz),
      mas justifica MANTER pdp como features brutas (nao agregadas via
      engineering linear como em H21).

      ATUALIZADO iter_0021: atratividade SUBIU. UlFor H23 (commit
      `ccf53722`, leave-one-in SE LR pos-H22 ulfor) produziu lesson
      teorico transferivel: **PI deve ser medida com o modelo final, nao
      com proxy mais robusto**. Ridge usa L2 para redistribuir importance
      entre colineares -> PI subestima. LR sem shrinkage colapsa quando
      essas features sao removidas. Aplicavel direto ao nosso H22: se
      usarmos PI sobre OLS para escolher features e treinarmos GBDT
      depois, podemos subestimar features que GBDT precisa para
      interacoes nao-lineares. **Sugestao de protocolo atualizado**:
      medir PI tanto em OLS quanto em GBDT, e comparar deltas — gap
      grande indicaria features carregando interacoes nao-lineares
      mascaradas pelo OLS. Custo zero (mesmo dataset, mesmo split,
      adiciona ~30 LoC de PI-com-GBDT).

      ATUALIZADO iter_0023: lesson REFORCADA por UlFor H22_model_aware
      (commit `2daa5d40`). Implementacao empirica do mesmo principio:
      PI medida com `lr` (champion real de SE/S) recupera 9.1pp NMAE +
      0.449 R^2 em SE/lr vs PI medida com Ridge universal -- `ter_verif_rmean7`
      sobrevive no h22_model_aware. **Implicacao para nosso H22**: o lesson
      teorico nao e' soft suggestion, e' efeito mensuravel de magnitude
      ~10pp. Protocolo H22 nosso deve usar PI-com-GBDT (nao OLS), e a
      comparacao OLS-vs-GBDT no R^2 final continua valida como medida do
      gap nao-linear.
    type: model
    layer: curtailment
    target: pdp_gen_gbdt_vs_ols_gap
    priority: P3
    status: queued
    estimated_effort_hours: 1.0
    depends_on: [H3]
    blocks: []
    sanity_checks_required: [holdout, baseline]
    expected_value: validar engineering linear vs deixar GBDT capturar interacoes
    created_at: 2026-05-24T07:30:00Z

  - id: H15
    summary: S 'nao aprendivel' — rare event classifier em vez de regressor?
    detail: |
      ATUALIZADO iter_0006: PARCIALMENTE OBSOLETA. UlFor v3.3 pos-PDP-fix
      (req-0002) S/ML agora bate baseline (109% < persist_d1 113.7%). S
      JA E aprendivel como regressor — classifier nao e mais P2.
      Manter como P3 — pode ainda dar AUC > regressor para alerta
      operacional (recall mais util que MAE neste sub low-signal).
    type: methodology
    layer: curtailment
    target: S_classification
    priority: P3
    status: queued
    estimated_effort_hours: 1.5
    depends_on: []
    blocks: []
    sanity_checks_required: [holdout, baseline]
    expected_value: maybe upside, ja menos urgente que pre-PDP-fix
    created_at: 2026-05-24T03:30:00Z

  - id: H24
    summary: Mesmo ensemble (model + persist) aplicado aos champions Ridge/LR UlFor
    detail: |
      Derivada de H10 iter_0013 (CONFIRMADO LGBM+persist beat LGBM-only
      em NE 13-28%, SE 4-7%, N 14-17%). Champions UlFor em producao sao
      Ridge_NE, LR_SE, LR_S, Ridge_N (iter_0007/iter_0011, endpoint
      /api/forecast/d1 LIVE). H24 testa: o mesmo esquema de ensemble
      (peso analitico inv_mae ou inv_mse) com champion Ridge/LR no lugar
      do LGBM produz ganho similar?

      Mecanismo esperado: Ridge/LR sao baixa-variancia/alto-bias por
      construcao linear; persist_d1 e' alta-variancia/baixo-bias em
      regimes estaveis. Combinacao deve dar ganho parecido ou maior do
      que com LGBM (LGBM ja' captura mais regime change interno).

      Implementacao codavel local sobre features iter_0002 (reimplementar
      Ridge_alpha10 e LR_sklearn no replay loop nao requer dump MLflow).
      Mesmo CV walk-forward 5x60d gap 7d + inner_val 30d para pesos.
      Comparar Ridge-only vs Ridge+persist em todas as 4 subs.

      Se confirmado, FINDING.md pode sugerir ao UlFor adicionar
      post-processing ensemble no endpoint /api/forecast/d1.
    type: model
    layer: curtailment
    target: ensemble_champion_persist
    priority: P2
    status: done
    iter_handled: 0022
    verdict: CONFIRMADO_3SUBS
    verdict_detail: |
      NE+SE+N confirmam ganho clinico do ensemble vs champion-only:
        NE (ridge_alpha10): 3/3 cells, best delta -9089 MWh (24% MAE
            reducao, ens_equal NE/v3), R² +0.08 -> +0.40
        SE (lr):            3/3 cells, best delta -2100 MWh (24% reducao,
            ens_inv_mse SE/v3), R² FLIP NEG->POS -0.78 -> +0.06.
            Ganho 4x maior que H10 LGBM em SE — confirma predicao H24
            (lineares deixam mais variancia residual para persist).
        N  (ridge_alpha10): 3/3 cells, best delta -107 MWh (19% reducao,
            ens_inv_mse N/v1). Empate magnitude com H10.
        S  (lr):            0/3 confirm. Wins 1-2/5 (abaixo limiar 3/5);
            v3 PIORA +50 MWh. Champion lr_s ja' tight R²=+0.45 v3 —
            ensemble adiciona ruido. Consistente com H10 (S/v3 +26 MWh).
      Best schemes globais (12 cells): ens_inv_mse 5x, ens_equal 5x,
      ens_inv_mae 1x, ens_opt_alpha 2x (S only). Pesos analiticos
      simples dominam — consistente com BMA classico.
    estimated_effort_hours: 1.5
    actual_effort_hours: 1.4
    depends_on: [H10]
    blocks: []
    follow_ups_created: []
    sanity_checks_required: [baseline, holdout, dist_shift]
    sanity_checks_done: [B1_inherit, B3_via_cv, B4_integrated, B5_via_weights, B2_NA, B6_NA]
    expected_value: validar se ganho do ensemble se propaga a producao (Ridge/LR)
    created_at: 2026-05-24T10:30:00Z
    completed_at: 2026-05-24T18:30:00Z

  - id: H25
    summary: Stacker meta-modelo (Ridge sobre base preds) supera weighted average?
    detail: |
      Derivada de H10 iter_0013. H10 mostrou que pesos analiticos
      (inv_mae, inv_mse) > grid-search empirico (ens_opt_alpha) em
      inner_val de 30d. H25 sobe um nivel: stacker via Ridge regression
      sobre features = [LGB_pred, persist_d1_pred, ma7_pred,
      climatologia_doy_pred] no inner_val pode capturar interacoes
      lineares simples (ex: peso de persist depende do nivel da
      previsao do LGB).

      Cuidado de leak: stacker treinado no inner_val SO. Pesos do Ridge
      aplicados ao test sem retreino. Comparar contra H10 (peso analitico
      simples) e LGB-only.

      Risco: inner_val 30d e' pequeno para Ridge robusto (4 baselines +
      intercept = 5 parametros). Considerar shrinkage forte (alpha alto)
      ou inner_val maior (60d?) ao custo de perder fold.

      Aceitacao: stacker bate H10 best em maioria das cells por >=2% MAE
      reduction (significancia clinica).
    type: model
    layer: curtailment
    target: ensemble_stacker_ridge
    priority: P3
    status: queued
    estimated_effort_hours: 2.0
    depends_on: [H10]
    blocks: []
    sanity_checks_required: [holdout, leak, baseline]
    expected_value: validar se Ridge captura interacoes que weighted average perde
    created_at: 2026-05-24T10:30:00Z

  - id: H26
    summary: Conformal prediction post-hoc para calibrar bandas P10/P90
    detail: |
      Derivada de H11 iter_0014 (REFUTADO_NE). LGBM quantile com defaults
      teve coverage_band_80 mean 43.6% vs 80% nominal — under-coverage
      sistemico em todas as 4 subs. Causa raiz e' LGBM nao modelar
      heteroscedasticidade explicita + distribution shift (iter_0012 KS
      p<0.0001 NE+SE).

      Fix candidate: split conformal prediction (Lei et al. 2018).
        1. Treina LGBM quantile no train_inner (mesmo split H11).
        2. No inner_val (30-60d), computa nonconformity scores:
             s_i = max(q10_i - y_i, y_i - q90_i)
        3. Calcula quantile empirico q_alpha = quantile(s_i, 1-alpha) (eg 0.8).
        4. Inflar bandas: [q10 - q_alpha, q90 + q_alpha].
      Goal: cov_band_80 in [75%, 85%] com sharpness preservada o maximo
      possivel. Sem retreinar modelo, baixo custo.

      Risco: inner_val 30d e' pequeno para conformal robusto. Considerar
      enlarge para 60d ou usar cross-validation+ conformal.

      Aceitacao: cov_band_80 mean NE in [75%, 85%] em pelo menos 2/3 cells,
      AND sharpness < 2x do baseline (banda nao explode).
    type: model
    layer: curtailment
    target: NE_d1_quantile_calibrated
    priority: P3
    status: queued
    estimated_effort_hours: 2.0
    depends_on: [H11]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    expected_value: torna H11 deliverable se calibracao funcionar
    created_at: 2026-05-24T11:30:00Z

  - id: H27
    summary: P50 quantile como point estimate substituto em N+S
    detail: |
      Derivada de H11 iter_0014 (bonus finding). P50 quantile BATE
      LGB-mean em magnitude:
        N: -18% MAE mean (0.467k vs 0.578k)
        S: -14% MAE mean (0.97k vs 1.07k)
        SE: -1.6% MAE mean (praticamente empate)
        NE: +4.5% MAE mean (P50 nao quebra, mas tampouco ajuda)
      Mecanismo: mediana e' mais robusta que mean para distribuicoes com
      cauda longa de zeros (N tem ~37% dias com curt~0).

      Setup: trocar objective='regression' por objective='quantile',
      alpha=0.5. Mesmo X, mesmo split. Bake-off em 12 cells x 5 folds CV
      (mesma estrutura H7/H10/H11). Comparar metric_suite (MAE/R²/F1).

      Aceitacao: P50 bate LGB-mean em MAE em pelo menos 2/3 cells de
      N+S, e nao perde >5% em NE+SE.

      Custo zero (objective swap), ganho transversal pequeno mas universal.
    type: model
    layer: curtailment
    target: curtailment_d1_point_p50
    priority: P3
    status: queued
    estimated_effort_hours: 1.0
    depends_on: [H11]
    blocks: []
    sanity_checks_required: [holdout, baseline]
    expected_value: ganho barato sem novo modelo, robustez vs outliers
    created_at: 2026-05-24T11:30:00Z

  - id: H28
    summary: NGBoost vs LGBM quantile — distribuicao parametrica resolve under-coverage?
    detail: |
      Derivada de H11 iter_0014 (alt-modelo). NGBoost (Duan et al. 2020)
      modela distribuicao parametrica (Normal/Lognormal) — sigma(x) e'
      funcao explicita de X, lidando com heteroscedasticidade que LGBM
      quantile nao captura.

      Detail do H11 ja menciona: "Bench worktree ja tem NGBoost similar"
      — UlFor pode ter codigo de referencia. Investigar e adaptar para o
      replay loop (features iter_0002, CV walk-forward).

      Aceitacao: NGBoost (Normal ou LogNormal) com defaults atinge
      cov_band_80 in [70%, 90%] em pelo menos 2/3 NE cells, AND pinball
      loss <= LGBM quantile em P50.

      Risco: NGBoost e' lento (boosting de gradient natural). Considerar
      n_estimators reduzido (100 vs 300 LGBM). Lognormal lida melhor com
      curt positiva-skewed mas precisa transformacao log(y+1).
    type: model
    layer: curtailment
    target: NE_d1_ngboost_quantile
    priority: P3
    status: queued
    estimated_effort_hours: 3.0
    depends_on: [H11]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    expected_value: alt-arquitetura para incerteza se conformal nao bastar
    created_at: 2026-05-24T11:30:00Z

  - id: H30
    summary: pdp_residual em Ridge_alpha10 CV 5x60d -- ortogonal ao OLS de H21?
    detail: |
      Derivada de H21 iter_0020 (REFUTADO em OLS puro). H21 mostrou que
      substituir (pdp_prev_eolica + pdp_prev_solar) por pdp_residual_total
      em OLS perde -5pp R² NE / -4pp SE em-sample (e amplia para -27pp em
      holdout 80/20). OLS sobre (gen, pdp_prev) ja' tem qualquer combinacao
      linear de (gen, residual) no seu span -- engineering linear nao
      expande basis.

      NOTA: outra sessao paralela do loop (iter_0019 recon_delta) ja deixou
      pronto (mas NAO rodou) o script `scripts/h21_pdp_residual_cv.py` com
      setup CV 5x60d LGBM walk-forward + 5 feature sets (A baseline,
      B additive, C replacement, D drop prog, E full simplification). H30
      pode trocar LGBM por Ridge_alpha10 ou rodar AMBOS (LGBM + Ridge) no
      mesmo run -- isso responde simultaneamente H30 (Ridge basis) e parte
      de H22 (GBDT vs OLS gap, ja queued P3).

      H30 testa se Ridge_alpha10 (champion UlFor NE/N, regularizacao
      L2 forte) comporta-se diferente. Mecanismo conjecturado: Ridge
      shrinkage redistribui pesos entre features colineares; com basis
      transformado (residual centrado em zero), shrinkage pode preservar
      mais sinal. Se Ridge tambem perde -3pp+, encerra H3-family residual
      no replay loop.

      Protocolo: CV walk-forward 5 folds (60d, gap 7d), mesma metodologia
      H7/H10/H11/UlFor official. Comparar 3 cells em NE+SE (sem S, que H3
      ja flaggou fragil; sem N, que tem cobertura zero PDP):
        - cell A: Ridge_alpha10 com V_brutos (feature_set base 3-feat)
        - cell B: Ridge_alpha10 com V_residual_plus_gen (2 feat)
        - cell C: Ridge_alpha10 com V_residual_split (3 feat)
      Metric: NMAE_mean + R²_mean por (sub, cell). Acceptance:
        - CONFIRMADO_RIDGE se V_residual_plus_gen >= V_brutos - 0.005 R² em
          NE+SE (ie Ridge redistribui sinal apesar de OLS perder)
        - REFUTADO_RIDGE se delta < -0.01 em qualquer de NE/SE (mesma
          conclusao de H21 OLS estende a Ridge)
      Custo: baixo (codavel local replay loop, mesmo CV de H10/H11; reuso
      parquet cache de H3/H21).

      Implicacao se CONFIRMADO_RIDGE: vale revisar feature_set=full do
      UlFor adicionando residual; permite reduzir 55->54 feat sem perda
      em Ridge. Se REFUTADO_RIDGE: encerra residual como avenida de
      engineering (H22 GBDT segue como ultima tentativa para mecanismo
      nao-linear). Sem req externo (zero dep UlFor).
    type: model
    layer: curtailment
    target: pdp_residual_in_ridge_cv
    priority: P3
    status: queued
    estimated_effort_hours: 1.0
    depends_on: [H21]
    blocks: []
    sanity_checks_required: [holdout, baseline]
    expected_value: encerrar H3-family residual no replay loop (Ridge confirma OLS ou nao)
    created_at: 2026-05-24T16:30:00Z
