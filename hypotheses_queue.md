---
schema_version: 1
last_updated: 2026-05-24T03:30:00Z
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
    type: feature
    layer: curtailment
    target: feat_pdp_renovavel_residual
    priority: P2
    status: queued
    estimated_effort_hours: 1.5
    depends_on: []
    blocks: []
    sanity_checks_required: [leak, perm, dist_shift]
    expected_value: justificar manter PDP no modelo apos resolver gaps
    created_at: 2026-05-24T03:00:00Z

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
      Enviado ao UlFor como req-0003.
    type: methodology
    layer: curtailment
    target: SE_v3_lag_collapse
    priority: P1
    status: blocked
    iter_handled: 0004
    estimated_effort_hours: 0.0
    depends_on: [req-0003]
    blocks: []
    sanity_checks_required: []
    expected_value: decidir se SE/v3 promovivel
    created_at: 2026-05-24T03:00:00Z

  - id: H7
    summary: XGBoost vs LGBM com mesmo split — qual generaliza melhor?
    detail: |
      iter_0002 treinou ambos com defaults. Saidas mostram XGB melhor em
      alguns subs (NE/v3 R²=+0.52 vs LGB R²=-0.00) mas com test n=11.
      Validar com cross-validation se XGB e sistematicamente melhor que
      LGBM com features iter_0002.
    type: model
    layer: curtailment
    target: model_comparison_lgbm_xgb
    priority: P2
    status: queued
    estimated_effort_hours: 1.5
    depends_on: []
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    expected_value: escolher modelo certo para v4
    created_at: 2026-05-24T03:30:00Z

  - id: H8
    summary: Validar feat_intercambio importance via permutation
    detail: |
      iter_0002 PI mostrou val_net_mwmed_lag1 no top-5 de N/v1-v3 mas baixo
      em outros subs. Permutation rigorosa para confirmar se intercambio
      e feature critica (em N) ou ruido.
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

  - id: H9
    summary: NMAE substituido por MAE/R²/F1 (Principio 6 PLANO_FINAL)
    detail: |
      PLANO_FINAL UlFor Principio 6: "Para curtailment usar MAE/R²/F1,
      nunca NMAE/media". Iter 0002 usou NMAE — viola principio. Refatorar
      leaderboard + sanity checks B3/B4 para reportar MAE+R²+F1 (binarizada
      em "curt > P50") ao inves de NMAE. Manter NMAE como secundario.
    type: metric
    layer: meta
    target: metric_suite
    priority: P1
    status: queued
    estimated_effort_hours: 1.0
    depends_on: []
    blocks: [H10, H11]   # ensemble e quantile precisam de metric reformada
    sanity_checks_required: []
    expected_value: alinhar com PLANO_FINAL UlFor, evitar inferencia ruim
    created_at: 2026-05-24T03:30:00Z

  - id: H10
    summary: Ensemble v2_LGBM + persistencia_d1 weighted by skill
    detail: |
      Iter 0002 mostra persist_d1 baseline forte em N (vence ML). Ensemble
      simples (peso = skill score em CV) pode dominar v2 puro em subs onde
      persistencia carrega muito sinal. Testar em NE+SE.
    type: model
    layer: curtailment
    target: ensemble_v2_persist
    priority: P2
    status: queued
    estimated_effort_hours: 1.0
    depends_on: [H9]
    blocks: []
    sanity_checks_required: [baseline, holdout]
    expected_value: ganhos baratos sem novo modelo
    created_at: 2026-05-24T03:30:00Z

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
    status: queued
    estimated_effort_hours: 2.0
    depends_on: [H9]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    expected_value: deliverable para v1.0 com bandas de confianca
    created_at: 2026-05-24T03:30:00Z

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

  - id: H15
    summary: S 'nao aprendivel' — rare event classifier em vez de regressor?
    detail: |
      S tem ymean_test=0 (rare ENE/CNF events). Regressor NMAE NaN.
      Treinar binario "havera curt ENE/CNF em D+1?" com class_weight=
      balanced. AUC > 0.7 = ja util como alerta operacional. Datasets
      desbalanceados sao mais comuns no setor eletrico de transmissao.
    type: methodology
    layer: curtailment
    target: S_classification
    priority: P2
    status: queued
    estimated_effort_hours: 1.5
    depends_on: []
    blocks: []
    sanity_checks_required: [holdout, baseline]
    expected_value: destrava S, hoje sem nenhum modelo
    created_at: 2026-05-24T03:30:00Z
