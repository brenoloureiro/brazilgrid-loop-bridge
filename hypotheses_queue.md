---
schema_version: 1
last_updated: 2026-05-24T07:30:00Z
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
    type: metric
    layer: meta
    target: leaderboard_consistency_post_h9
    priority: P2
    status: queued
    estimated_effort_hours: 1.0
    depends_on: []
    blocks: []
    sanity_checks_required: []
    expected_value: leaderboard internamente consistente (MAE/R²/F1 em todas linhas)
    created_at: 2026-05-24T06:00:00Z

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
    type: feature
    layer: curtailment
    target: feat_pdp_residual_engineered
    priority: P2
    status: queued
    estimated_effort_hours: 1.0
    depends_on: [H3]
    blocks: []
    sanity_checks_required: [leak, perm, baseline]
    expected_value: feature mais densa + reducao de dim sem perda de skill
    created_at: 2026-05-24T07:30:00Z

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
