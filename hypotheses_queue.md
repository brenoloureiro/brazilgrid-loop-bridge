---
schema_version: 1
last_updated: 2026-05-26T14:00:00Z
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

      VEREDITO iter_0027: CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge.
      Loop rodou CV-PI independente (Ridge_alpha=10 + Ridge_alpha=1, walk-
      forward 5x60d gap 7d, 30 perms x 4 features) + joint-drop refit test
      como primario (autoritativo p/ colinears val_net=imp-exp identidade
      perfeita VIF=1e8). Resultado joint-drop por sub:
        - NE: bundle_drop_improves_model (a10: -1.17pp; a1: -3.95pp NMAE)
              -> CONFIRMA UlFor drop direction com efeito STRONGER.
        - SE: bundle_drop_HARMFUL (a10: +0.27pp; a1: +1.21pp) -> CONTRADIZ
              UlFor em Ridge. UlFor decidiu drop com lr champion + VIF;
              divergencia model-aware (~10pp gap lesson H22_MA empirico).
        - S:  bundle_drop_improves_model (-8.3pp / -7.0pp) -> CONFIRMA
              drop com magnitude muito maior; sugere extensao 4-feat
              (UlFor listou so val_net_lag1).
        - N:  bundle_drop_improves_model (-2.7pp / -2.3pp) -> sugere drop
              em Ridge mas modelo N base NMAE>1 (limite-de-dado, nao
              funcional). INCONCLUSIVO_em_N; UlFor keep direction
              permanece valida out-of-scope deste teste.
      Lesson metodologica forte: PI single-feat sobre colinears identicos
      e' SISTEMICAMENTE VIESADO PARA CIMA (permutar 1 feature quebra
      identidade local). Joint-drop refit e' o teste autoritativo. Em NE
      alpha=1: val_import single-PI = +1.65pp (parece KEEP) mas joint-drop
      = -3.95pp (bundle ATIVAMENTE HARMFUL). VIF + PI multivariado UlFor
      H22 metodologicamente superior — esta iter empiricamente reforca isso.
      H33 derivada (P3 ~0.5h): replicar joint-drop SE com LinearRegression
      em vez de Ridge para fechar o caveat model-aware locally.
    type: feature
    layer: curtailment
    target: feat_intercambio_importance
    priority: P3
    status: done
    iter_handled: 0027
    verdict: CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge
    estimated_effort_hours: 0.5
    actual_effort_hours: 0.7
    depends_on: []
    blocks: []
    sanity_checks_required: [perm]
    sanity_checks_done: [perm, leak, holdout_temporal_strict, baseline_compare, dist_shift]
    follow_ups_created: [H33]
    expected_value: drop intercambio se confirmado ruido
    created_at: 2026-05-24T03:30:00Z
    last_external_update_iter: 0023
    completed_at: 2026-05-24T22:30:00Z

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
    status: done
    iter_handled: 0029
    veredito: CONFIRMADO_DISPLAY_REFUTADO_REGIME_CLAIM
    estimated_effort_hours: 0.5
    actual_effort_hours: 0.3
    depends_on: []
    blocks: []
    sanity_checks_required: []
    sanity_checks_done: [leak_passed, perm_skipped_NA, holdout_strict_passed, baseline_passed, dist_shift_passed, zero_count_passed]
    expected_value: leaderboard mais honesto
    created_at: 2026-05-24T03:30:00Z
    closed_at: 2026-05-25T00:30:00Z
    closure_summary: |
      C1 (display) ATENDIDO trivialmente — persist_d7 ja' estava no leaderboard
      desde iter_0007 (4 linhas baselines secao). C2 (regime claim "S")
      REFUTADO em CV canonico 5x60d: persist_d1 vence em 4/4 subs no
      agregado (NE +77.3%, SE +19.4%, S +26.4%, N +13.1%) e em 19/20
      per-fold cells. Unica inversao: N fold 0 (regime temporal mais antigo,
      consistente com seca-2025Q3 dominante em N tardio ja' coberta por H14).
      Origem provavel da premissa: replay iter_0002 n=11 mostrou d7 vence d1
      em N (nao S — provavel erro de transcricao no detail original).
      Mantido como diagnostico auto-correlacao no leaderboard com nota
      explicita destruindo expectativa anterior. Sem H derivada criada
      (finding N fold 0 ja' coberto por hipoteses fechadas).
    artefatos: outputs/iter_0029/persist_d7_baseline_aux/ (analise.md +
      persist_d7_metrics.json + sanity_checks.json)

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

      VEREDITO iter_0032: CONFIRMADO_DISPLAY. Mudancas entregues:
        (1) scripts/h20_n_test_audit.py varre 12 meta.json + 21 outros JSON
            (1 falso positivo de regex excluido) + 18 parquets UlFor
            per_fold. Output: outputs/iter_0032/h20_leaderboard_low_n_test_warning/
            n_test_audit.{json,md}.
        (2) scripts/run_bakeoff_replay.py patcheado: meta.json e summary
            CSV carregam `low_confidence_n_test=(n_test<30)` +
            `low_n_test_threshold=30` constante exposta.
        (3) outputs/iter_0002/summary_replay.csv backfilled com a nova
            coluna sem re-rodar bake-off (todas 12 linhas n_test=11 -> true).
        (4) leaderboard.md: bloco "Politica low_confidence_n_test" no
            topo + coluna n_test e low_confidence_n_test na tabela
            deprecada Historico iter loop. Champion + baselines + overlay
            + sucessores no topo todos usam n_test ∈ {59,60} = FALSE.
        (5) Lessons learned acresce entrada explicando display-per-row
            vs aviso por secao.

      Totais audit: 12 LOW (meta runner) + 8 LOW (outros sanity JSON) +
      0 LOW (UlFor parquets oficiais). Zero falsos negativos na tabela
      topo do leaderboard.
    type: meta
    layer: meta
    target: leaderboard_low_n_test_warning
    priority: P3
    status: done
    iter_handled: 0032
    verdict: CONFIRMADO_DISPLAY
    estimated_effort_hours: 0.5
    actual_effort_hours: 0.6
    depends_on: []
    blocks: []
    sanity_checks_required: []
    sanity_checks_done: [leak_NA, perm_NA, holdout_NA, baseline_PASS_BY_AUDIT, dist_shift_NA, zero_count_NA, audit_idempotent_PASS, audit_coverage_PASS, leaderboard_top_no_LOW_PASS, runner_patch_backward_compat_PASS, csv_backfill_correctness_PASS]
    follow_ups_created: []
    expected_value: leaderboard auto-documentado para baixa confianca amostral
    created_at: 2026-05-24T06:45:00Z
    completed_at: 2026-05-25T03:30:00Z
    closure_summary: |
      Display-layer + runner-meta patcheados sem retrain. Audit cobre
      32 artefatos do loop + 18 parquets UlFor; 20 marcados LOW (todos
      em iters 0002/0004/0008/0009 do replay LGBM n=11, ja' isolados na
      secao deprecada). Zero linhas LOW nas tabelas ativas (Champions
      oficiais, Baselines, Overlay producao, Sucessores, Alpha sweep v3,
      Val14d). Threshold n_test<30 herdado de B6 H16 v1.1 (iter_0009).
      Sem H derivada — feedback loop fecha porque o problema raiz
      (replay n=11 vs UlFor n=60) ja' foi reconhecido em iter_0007.
    artefatos: outputs/iter_0032/h20_leaderboard_low_n_test_warning/
      (n_test_audit.json + n_test_audit.md + sanity_checks.json) +
      scripts/h20_n_test_audit.py + scripts/run_bakeoff_replay.py (patched)
      + outputs/iter_0002/summary_replay.csv (backfilled)

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

      VEREDITO iter_0034: REFUTADO_NO_NONLINEAR_GAIN. Holdout temporal
      80/20 (n_train=388, n_test=98, 2024-12 -> 2026-05) em NE/SE/S com
      3 features (gen_renov_mwh + pdp_prev_eolica + pdp_prev_solar). OLS
      lstsq vs LightGBM (n_est=300, lr=0.05, num_leaves=31).
        NE:  OLS R^2=+0.550 vs GBDT R^2=+0.507; gap -4.3pp (GBDT_WORSE)
        SE:  OLS R^2=+0.072 vs GBDT R^2=+0.088; gap +1.5pp (TIE)
        S:   OLS R^2=-0.252 vs GBDT R^2=-0.426; gap -17.4pp (GBDT_WORSE)
      max gap = +0.015 (< +5pp em todos); mean gap = -0.067. GBDT overfit
      severo train: R^2_train 0.99/0.96/0.81 vs R^2_test 0.51/0.09/-0.43.
      Premissa central "GBDT extrai sinal nao-linear adicional" NAO se
      sustenta com 3 features. Persist_d1 baseline test: NE 0.37 / SE -0.37
      / S -0.39 -- OLS_3feat bate persist em NE+SE+S, GBDT_3feat empata.
      PI duo (refit-test drop) RECONFIRMA H22_MA empirico: gap_per_feat
      OLS-vs-GBDT abs_drop chega a +85pp (NE pdp_eolica) e -20pp (SE
      pdp_eolica) -- PI EH model-dependente, mas isso nao traduz em
      ganho de R^2 (model-flexible signal nao excede signal capturada
      pelo linear basis com 3 features). H36 derivada (P3 queued): testar
      mesma comparacao com features iter_0002 v3 (37+ feats) -- so' com
      mais features as interacoes nao-lineares teriam espaco; com 3 feats
      o limite e' do espaco de hipoteses do GBDT vs OLS, nao de capacidade.
    type: model
    layer: curtailment
    target: pdp_gen_gbdt_vs_ols_gap
    priority: P3
    status: done
    iter_handled: 0034
    estimated_effort_hours: 1.0
    actual_effort_hours: 0.8
    depends_on: [H3]
    blocks: []
    follow_ups_created: [H36]
    sanity_checks_required: [holdout, baseline]
    sanity_checks_done: [leak_passed, perm_passed_gbdt, holdout_strict_passed, baseline_passed, dist_shift_reported, zero_count_reported]
    expected_value: validar engineering linear vs deixar GBDT capturar interacoes
    created_at: 2026-05-24T07:30:00Z
    completed_at: 2026-05-25T05:30:00Z

  - id: H15
    summary: S 'nao aprendivel' — rare event classifier em vez de regressor?
    detail: |
      ATUALIZADO iter_0006: PARCIALMENTE OBSOLETA. UlFor v3.3 pos-PDP-fix
      (req-0002) S/ML agora bate baseline (109% < persist_d1 113.7%). S
      JA E aprendivel como regressor — classifier nao e mais P2.
      Manter como P3 — pode ainda dar AUC > regressor para alerta
      operacional (recall mais util que MAE neste sub low-signal).

      VEREDITO iter_0030: CONFIRMADO_PARCIAL_NON_RARE. CV walk-forward
      5×60d gap 7d em S/v1 (37 feats iter_0002) com LogReg+LGBMClassifier
      vs LR_reg (champion S) + Ridge_α10 binarizados, 3 thresholds.

      Resultado por threshold:
        thr_zero (any curt, pos_rate 40%): LogReg AUC=0.784 vs LR_reg=0.739
          (+4.5pp); PR-AUC 0.784 vs 0.733 (+5.1pp). CONFIRMA.
        thr_p75 (big curt, pos_rate 25%): LogReg AUC=0.821 vs LR_reg=0.754
          (+6.8pp); PR-AUC 0.716 vs 0.635 (+8.1pp). CONFIRMA.
        thr_p90 (rare event, pos_rate 10%): LogReg AUC=0.762 vs LR_reg=0.778
          (−1.6pp); PR-AUC 0.451 vs 0.456 (−0.6pp). REFUTA original claim.

      Hipotese original (rare-event classifier > regressor): REFUTADA.
      Mecanismo: pos absoluto baixo no test (fold 5 thr_p90: 1 positivo em
      60d) inviabiliza calibracao LogReg; regressor binarizado a thr_p90
      preserva ordering por magnitude continua.

      Hipotese REFINADA (CONFIRMADA): classifier dedicado > regressor
      binarizado para alerta moderado em S (any curt ou big curt). LogReg
      AUC 0.78-0.82 vs persist 0.62-0.63 (+16-19pp).

      Sanity: 6 checks executados (leak inherited, perm p=0.000 fold 5,
      holdout PASS, baseline persist+climat PASS, dist_shift WARN
      severo, zero_count WARN 3/37 features curt_lag*).
    type: methodology
    layer: curtailment
    target: S_classification
    priority: P3
    status: done
    iter_handled: 0030
    verdict: CONFIRMADO_PARCIAL_NON_RARE
    estimated_effort_hours: 1.5
    actual_effort_hours: 1.2
    depends_on: []
    blocks: []
    sanity_checks_required: [holdout, baseline]
    sanity_checks_done: [leak, perm, holdout, baseline, dist_shift, zero_count]
    follow_ups_created: [H35]
    expected_value: maybe upside, ja menos urgente que pre-PDP-fix
    created_at: 2026-05-24T03:30:00Z
    completed_at: 2026-05-25T01:30:00Z

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
    status: done
    iter_handled: 0035
    verdict: REFUTADO_NO_GAIN
    estimated_effort_hours: 2.0
    depends_on: [H10]
    blocks: []
    sanity_checks_required: [holdout, leak, baseline]
    sanity_checks_done: [holdout, leak, baseline, dist_shift, zero_count]
    expected_value: validar se Ridge captura interacoes que weighted average perde
    actual_value: |
      Refutacao em 12/12 cells. Ridge stacker NUNCA bate H10 inv_mae.
      pct_delta MAE vs H10: NE +10.1% / SE +14.2% / S +57.1% / N +33.3% (mean
      across cells/folds). Best variant `ridge_no_intercept_a10` (suprime
      intercept, alpha=10 shrinkage forte) ainda perde por +2.1% a +90.0%.
      Variants com intercept (alpha=0.1..100) explodem MAE 50-200x (overfit
      inner_val catastrofico).

      Risco previsto no queue ("30d × 4 bases pequeno demais") SE CONCRETIZOU:
        - dist_shift MAE_test/MAE_inner_val por cell: NE/v1 4545% / NE/v2 149%
          / SE/v3 81% / S/v3 283% / N/v1 64%. Single fold extremo NE/v1
          atinge 28280% (Ridge memoriza inner_val, test arrasa).
        - Inv_mae H10 NAO memoriza inner_val (so 2 pesos derivados de MAE
          escalar — invariante a ruido idiossincratico de val). Por isso
          generaliza melhor sob distribution shift comprovada em iter_0012.

      Convergencia com lessons:
        H10 (CONFIRMADO_NE_SE): pesos analiticos > grid empirico em inner_val 30d
        H24 (CONFIRMADO ridge_lr_NE_SE_N): mesmo paradigma (inv_mae) com
          champions Ridge/LR ate melhor em SE (4x ganho).
        H25 (REFUTADO): subir capacidade do ensemble (Ridge stacker 4-feat)
          PIORA — confirmando que o gargalo NAO e' o esquema de pondera-
          cao mas o sinal residual disponivel apos LGB. Adicionar bases
          (ma7, clim_doy) na presenca de overfit inner_val nao destrava.
    follow_ups_created: []
    created_at: 2026-05-24T10:30:00Z
    completed_at: 2026-05-25T06:30:00Z

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
    status: done
    iter_handled: 0037
    verdict: INDETERMINADO_NE_BORDERLINE_CONFIRMADO_SE_S_BONUS
    estimated_effort_hours: 2.0
    depends_on: [H11]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    sanity_checks_done: [holdout_passed_embedded, baseline_passed_embedded, dist_shift_annotated_reuse]
    expected_value: torna H11 deliverable se calibracao funcionar
    actual_value: |
      NE iv=30 cov_cal mean 74.7% (falha [75%,85%] strict por 0.3pp),
      width_ratio_cal_vs_inner 2.03x (falha <2.0 por 0.03x), 3/3 NE cells
      in [70%,90%] loose. iv=60 nao salva (cov cai p/ 71.6%). Bonus
      CONFIRMADO em SE (3/3 strict, cov_cal 76.6%, width 1.95x) e S
      (3/3 strict, cov_cal 79.5%, width 1.51x) — conformal vira bandas
      P10/P90 deliverable para SE+S AGORA (decisao Breno, fora scope
      iter). N over-cobre (90.8%, width 1.68x). Replicacao H11 bit-exato
      (cov_uncal_full == iter_0014 summary). Mecanismo conformal validado
      (cov_cal > cov_uncal em 100% folds), limite e' do TARGET (NE dist
      shift documentado iter_0012 KS p<0.0001), nao do metodo.
    follow_ups_created: [H37]
    created_at: 2026-05-24T11:30:00Z
    completed_at: 2026-05-25T08:00:00Z

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
    status: done
    iter_handled: 0038
    verdict: CONFIRMADO
    verdict_summary: |
      D1 (target N+S) PASS: P50 bate LGB-mean em 6/6 cells de N+S (100%),
      muito acima de 67% threshold. N mean delta -17.9% MAE (15/15 folds
      P50 wins). S mean delta -13.8% MAE (11/15 folds). Bonus: delta_R2
      mean N+S = +0.287 (mean tinha R² negativo em varios folds N; P50
      puxa para positivo). D2 (constraint NE+SE) PASS na sub-mean: NE
      +4.5% (just under +5% threshold, mas NE/v2 +5.6% e NE/v3 +8.4%
      individuais ultrapassam), SE -1.6%. Decision: PROMOVE_NS_FLAG_NE
      (P50 default para N+S+SE; NE mantem LGB-mean por cauda densa).
      Mecanismo NE explica gap: cauda densa premia mean, cauda esparsa+
      massa em zero premia mediana (KS p<0.0001 iter_0012 confirma shift).
      Re-analise direta de H11 iter_0014 (mesmo experimento, criterios
      diferentes; random_state=0 bit-exato).
    estimated_effort_hours: 1.0
    actual_effort_hours: 0.6
    depends_on: [H11]
    blocks: []
    sanity_checks_required: [holdout, baseline]
    sanity_checks_status:
      B1_leak: skipped_inherited (iter_0010 H3 p=0)
      B2_perm: skipped_inherited (iter_0010 H3 p=0)
      B3_holdout: passed_embedded (gap=7d, 60 folds)
      B4_baseline: passed_embedded (12/12 cells P50 >= persist_d1)
      B5_dist_shift: annotated_reuse (KS p<0.0001 NE+SE — explica gap NE)
      B6_n_test: passed (n_test 58-60, threshold >=30)
    follow_ups_created: []  # H28/H37 ja cobrem NGBoost/CQR-asymmetric
    expected_value: ganho barato sem novo modelo, robustez vs outliers
    created_at: 2026-05-24T11:30:00Z
    completed_at: 2026-05-25T15:30:00Z

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
    status: done
    iter_handled: 0039
    estimated_effort_hours: 3.0
    actual_effort_hours: 0.5
    depends_on: [H11]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    sanity_checks_done:
      B1_leak: skipped_inherited (iter_0010 H3 features identicas)
      B2_perm: skipped_inherited (iter_0010 H3)
      B3_holdout_strict: passed_embedded (gap=7d em 30 folds)
      B4_baseline: passed_embedded (LGBM quantile H11 per-fold + persist_d1)
      B5_dist_shift: annotated_reuse (KS p<0.0001 NE+SE — explica mecanismo)
      B6_n_test: passed (n_test 58-60, threshold >=30)
    verdict: INDETERMINADO_PINBALL_DEGRADA
    verdict_summary: |
      D1 cov calibration PASS em ambas dists: Normal 2/3 NE cells em
      [70%, 90%] (sub-mean 73.5%, +30pp absoluto vs LGBM); LogNormal-equiv
      3/3 cells em [70%, 90%] (sub-mean 88.3%, borderline alto). NGBoost
      LIFTA cov +21-46pp por fold. D2 pinball preservation FAIL: pinball
      P50 degrada vs LGBM em ambas dists — Normal +14.4% sub-mean
      (NE/v1 +6.5, NE/v2 +19.6, NE/v3 +17.1), LogNormal +55.5% sub-mean.
      Mecanismo: NGBoost minimiza NLL Normal(mu, sigma); P50 vira
      mu(x)=mean, nao mediana. LGBM quantile alpha=0.5 minimiza pinball
      P50 diretamente — vantagem natural. LogNormal piora por
      re-exponenciacao amplifica scale. Width inflation Normal +86-119%,
      LogNormal +278-348%. Decision NAO_PROMOVE_E_FECHA_CAMINHO_PARAMETRICO_DEFAULT.
      Caminho NGBoost defaults encerrado para NE D+1 curt no replay loop.
      H37 (CQR-asymmetric + Mondrian) e' o proximo swing em cov sem tocar
      P50. Zero follow-ups criados.
    follow_ups_created: []  # H37 conformal-asymmetric ja queued cobre proximo passo
    expected_value: alt-arquitetura para incerteza se conformal nao bastar
    created_at: 2026-05-24T11:30:00Z
    completed_at: 2026-05-25T18:30:00Z

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
    status: done
    iter_handled: 0040
    verdict: REFUTADO_RIDGE
    estimated_effort_hours: 1.0
    actual_effort_hours: 0.4
    depends_on: [H21]
    blocks: []
    sanity_checks_required: [holdout, baseline]
    sanity_checks_passed: [leak, perm, holdout, baseline, dist_shift, zero_count, n_test]
    expected_value: encerrar H3-family residual no replay loop (Ridge confirma OLS ou nao)
    created_at: 2026-05-24T16:30:00Z
    completed_at: 2026-05-25T20:00:00Z
    follow_ups_created: [H38, H39]
    notes_iter0026: |
      Atratividade SOBE MARGINAL pos-alpha-sweep UlFor (commits 42dc0d7a +
      a3c742a9). Alpha sweep mostrou alpha=1 vence alpha=10 (default H30) em
      3/4 subs em h22_MA/h22_per_fold. Recomendacao iter_0027: rodar H30 com
      alpha=1 E alpha=10 simultaneamente (custo zero adicional, mesmo loop
      de CV) para validar se vereditico CONFIRMADO_RIDGE/REFUTADO_RIDGE muda
      com o alpha. Mecanismo conjecturado: menos shrinkage permite que basis
      residual centrado em zero retenha mais sinal -- alpha=1 e o teste
      mais sensivel desse mecanismo.
    result_summary_iter0040: |
      REFUTADO_RIDGE. Paired delta_R²(B-A) per cell+alpha vs threshold
      (CONFIRMADO >= -0.005 em NE AND SE; REFUTADO < -0.01 em qualquer):
        alpha=1:  NE B +0.013 (PASSA), SE B -0.032 (FAIL)
        alpha=10: NE B -0.064 (FAIL), SE B -0.043 (FAIL)
      Apenas NE/v3 alpha=1 marginalmente confirmou; SE quebra em ambos
      alphas; criterio AND nao OR. C (residual_split) catastrofico em SE
      (delta_R² -0.469 a -0.818). B2 perm confirma residual_total CARREGA
      sinal real (NE +105% / SE +27% MAE drop com shuffle), mas duplicata
      do span ja em A — Lesson: perm_importance != feature_engineering_gain.
      Convergencia H21 (OLS analitico + LGBM empirico) + H30 (Ridge alpha=1,
      alpha=10) == ENCERRA H3-family residual no replay loop. Apenas H22
      (GBDT vs OLS mecanismo nao-linear) segue como ultima frente.
      Artefatos: outputs/iter_0040/h30_pdp_residual_ridge_cv/

  - id: H38
    summary: Ridge alpha-sensitivity em set minimal residual — alpha<1 salva NE?
    detail: |
      Derivada de H30 iter_0040 (REFUTADO_RIDGE). NE/v3 alpha=1 marginalmente
      bateu A com B (+0.013 R²); alpha=10 destruiu (-0.064 R²). Em set
      minimal (2-3 feat), shrinkage L2 excessivo (alpha=10) destroi o pouco
      sinal residual disponivel; alpha=1 ainda shrinka demais para deixar
      passar (apenas marginal pass em 1 sub). Hipotese: alpha optimo para
      basis-residual em set minimal e' alpha < 1 (ou Ridge-CV/LassoCV/Bayesian
      Ridge com prior fraco). Custo: trivial (mesmo loop CV, expandir
      ALPHAS=[0.01, 0.1, 1.0, 10.0]).

      Mecanismo conjecturado: features residuais ja sao "diferencas" com
      menor variance que features brutas (pdp_residual_e std ~10k MWh vs
      pdp_prev_e std ~40k MWh em NE). Ridge shrinkage e' aplicado em escala
      absoluta (apos StandardScaler) — mesma alpha => mesmo shrinkage relativo.
      Mas pequenas variances tem menos sinal a "puxar" do prior zero, e
      ridge alpha=10 pode tornar coeficiente residual proximo de zero,
      colapsando informacao.

      Aceitacao H38:
        - CONFIRMADO se delta_R²_B >= -0.005 em NE AND SE para algum alpha < 1
          (ie Ridge minimal funciona desde que alpha tunado)
        - INDETERMINADO se 1 sub salva mas outra falha
        - REFUTADO se nenhum alpha salva ambos subs

      Impacto pratico: baixo. Mesmo se CONFIRMADO em alpha=0.01, ganho marginal
      (delta_R² +0.013 em 1 fold mean) nao justifica swap operacional. Esta
      hipotese existe principalmente para FECHAR o caminho "alpha-sensitivity
      foi a culpa" e nao deixar caveat tecnico em aberto. **P4** explicitamente.
    type: model
    layer: curtailment
    target: ridge_alpha_minimal_residual
    priority: P4
    status: done
    iter_handled: 0047
    verdict: INDETERMINADO_ALPHA_LOW
    estimated_effort_hours: 0.3
    actual_effort_hours: 0.3
    depends_on: [H30]
    blocks: []
    sanity_checks_required: [baseline]
    sanity_checks_done: [leak, perm, holdout, baseline, dist_shift, zero_count]
    expected_value: fechar caveat alpha-sensitivity de H30; baixo impacto pratico
    created_at: 2026-05-25T20:00:00Z
    completed_at: 2026-05-26T10:00:00Z
    verdict_summary: |
      INDETERMINADO_ALPHA_LOW. Alpha sweep [0.01, 0.1, 1.0, 10.0] sobre o
      protocolo H30 (CV 5x60d gap7d, NE+SE v3, sets A/B/C). Replay H30 BIT-EXATO
      (alpha=1 NE delta_r2_B=+0.01275; alpha=10 NE -0.06419; alpha=1 SE -0.03217;
      alpha=10 SE -0.04313 — todos batem iter_0040 5 casas decimais). NE responde
      bem ao shrinkage fraco: alpha=0.01 delta_r2_B=+0.0242 (wins_r2 5/5),
      alpha=0.1 +0.0231 (4/5) — MELHORES que alpha=1 (+0.0128, 2/5). Hipotese
      "alpha L2 forte destroi sinal residual minimal em NE" VALIDA — confirma
      mecanismo conjecturado (basis-residual tem std reduzido vs basis-bruto, e
      shrinkage absoluto pos-StandardScaler penaliza coef residual demais).
      Em SE, no entanto, NENHUM alpha salva B (delta_r2_B in [-0.043, -0.031]
      para todo o sweep; melhor caso alpha=0.01 ainda -0.031 << threshold -0.005).
      Mecanismo SE: residual_total SE tem frac_negative=0.188 (vs 0.975 NE) e
      mean=+11877 MWh (vs -99936 NE) — sinal qualitativamente diferente; SE
      base A ja captura melhor o sinal pdp_prev_e/s separadamente e a soma
      residual collapse perde informacao independente de regularizacao.
      Set C (split residual) tambem so passa em NE alpha<=0.1 (delta +0.015/+0.014);
      SE catastrofico em todos alphas (-0.47 a -0.88). H30 REFUTADO_RIDGE
      veredito GLOBAL persiste em SE; em NE o veredito muda para
      MARGINAL_PASS_em_alpha_baixo. Impacto pratico nulo (P4 explicito):
      delta_r2_B=+0.024 em NE alpha=0.01 ainda deixa o modelo abaixo do
      persist (NE skill A vs persist = -14% MAE, A perde para persist;
      adicionar +2.4pp R² em A=-0.05 vai para -0.03 — modelo continua
      sub-zero R²; champion UlFor Ridge_alpha10 com features completas
      iter_0007 ja entrega NMAE 33.7% que esta features minimais nem
      chegam perto). FECHA caveat tecnico "alpha-sensitivity foi a culpa
      de H30 ter REFUTADO" com nuance: foi a culpa em NE (alpha grande
      destrói o pouco sinal disponivel) mas NAO em SE (residual sub-set
      e' estruturalmente inadequado, independente de shrinkage).
      6 sanity checks PASS/REPORTED. Champions INTACTOS, zero rollback.
      0 follow-ups (P4 explicitamente: nao reabrir Ridge minimal residual
      mesmo com per-sub tuning; ganho marginal nao justifica swap operacional;
      arco H21/H22/H30/H38 ENCERRA frente residual em curt D+1).
    artefatos: outputs/iter_0047/h38_ridge_alpha_minimal_residual/
      (results.json + summary.csv + sanity_summary.json + verdict.json)
    follow_ups_created: []

  - id: H39
    summary: Doc — "perm_importance confirms signal != feature_engineering gain" playbook
    detail: |
      Derivada de H30 iter_0040 (REFUTADO_RIDGE com perm_importance massivo).
      Caso pedagogico canonico: NE/v3 Ridge_alpha10 set B na fold final mostrou
      pdp_residual_total perm_importance +105% (base_mae 17400 -> perm_mae
      35600, std 2500) — sinal real, NAO ruido. Ainda assim mean R² do
      mesmo set perdeu -0.064 vs baseline A com 3 features brutas. Mesmo
      padrao em SE alpha=10 (+27% perm importance, -0.043 R²).

      Lesson explicita: B2 perm test confirma que feature CARREGA sinal, mas
      NAO confirma que adicionar feature MELHORA o modelo. Se basis ja contem
      span da feature (algebra linear) ou se interaction terms ja extraem o
      sinal (GBDT), feature importance no modelo nao se traduz em ganho
      marginal. Decoupling sinal_intrinseco vs ganho_incremental.

      Aceitacao:
        - PROMOVE doc para sanity_checks/B2_interpretation.md
        - Inclui exemplo H21 LGBM (residual perm +6.6% NE, +10.8% SE; mean
          R² nao melhora) E H30 Ridge (perm +105% NE alpha=10, R² piora -0.064)
        - Adiciona 1 paragrafo "what perm_importance means" + "what it does NOT mean"

      Custo: 0.2h, doc-only, zero codigo. Sem prazo — quando proximo
      recon_delta ou iter de governance.
    type: governance
    layer: meta
    target: sanity_checks_doc_b2_interpretation
    priority: P5
    status: done
    iter_handled: 0048
    verdict: DOCUMENTADO
    estimated_effort_hours: 0.2
    actual_effort_hours: 0.2
    depends_on: [H30]
    blocks: []
    sanity_checks_required: []
    sanity_checks_done: [doc_aceitacao_3_criteria_PASS, doc_numbers_verified_6_of_6, doc_internal_links_REPORTED, default_six_NA_governance]
    follow_ups_created: []
    expected_value: evitar futura confusao "perm passou logo o feature serve" em iter futuras
    created_at: 2026-05-25T20:00:00Z
    completed_at: 2026-05-26T12:00:00Z
    closure_summary: |
      Doc PROMOVIDO em sanity_checks/B2_interpretation.md (153 linhas). 3 criterios
      de aceitacao H39 satisfeitos: (a) exemplos H21 LGBM (NE/v3 set C perm
      +6.6% / SE/v3 +10.8%, mean R² nao melhora — bake-off MAE delta NE +2.0pp /
      SE +3.1pp); (b) exemplo H30 Ridge (NE/v3 alpha=10 set B perm +104.6% /
      delta_R²=-0.064; SE alpha=10 perm +27.4% / delta_R²=-0.043); (c) paragrafos
      "what perm_importance means" + "what it does NOT mean" com decoupling
      explicito sinal_intrinseco vs ganho_incremental + heuristica operacional
      (B2 PASSA + B4 FALHA -> REFUTADO + razao basis-redundancia). Tabela
      convergencia H3+H21+H30+H38 anexada documentando arco residual encerrado.
      6 numeros do doc verificados contra source artifacts (iter_0020 e 0040
      sanity_summary.json + verdict.json): 6/6 match exato. Default 6-check
      suite NA (governance/meta layer, sem modelo treinado); audit substitutivo
      PASS. Champions UlFor INTACTOS (zero impacto operacional). 0 follow-ups.
    artefatos: outputs/iter_0048/h39_perm_importance_doc/
      (verdict.json + sanity_summary.json + summary.csv) +
      sanity_checks/B2_interpretation.md (doc canonico, 153 linhas)

  - id: H33
    summary: Joint-drop SE em LR vs Ridge — fechar caveat model-aware H22_MA empirico
    detail: |
      Derivada de H8 iter_0027 (CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge).
      H8 expos divergencia model-aware em SE: joint-drop do bundle de
      intercambio features (val_export+val_import+val_net) em Ridge_alpha10
      e Ridge_alpha=1 PIORA modelo (+0.27pp / +1.21pp NMAE). UlFor H22_model_aware
      (commit 2daa5d40) decidiu drop em SE usando lr champion + VIF; lesson
      teorico: "PI medida com o modelo final, nao com proxy mais robusto"
      (commit ccf53722 H23_ulfor). Ridge usa L2 que redistribui pesos entre
      colineares -> PI subestima individualmente, mas modelo se beneficia da
      redundancia. LR sem shrinkage colapsa quando essas features removem
      -> PI revela importance.

      H33 testa: joint-drop em SE refit com LinearRegression em vez de
      Ridge revela o mesmo dNMAE positivo ou inverte para negativo?

      Implementacao: trivial extension de h8_intercambio_cv_pi.py — trocar
      Ridge(alpha=...) por LinearRegression. Mesmo split CV walk-forward
      5x60d gap 7d, mesmo bundle, mesma sanidade.

      Aceitacao:
        - CONFIRMADO se joint-drop LR_SE dNMAE > 0 (piora drop, contra
          UlFor): reforca caveat de que H22_MA decision foi VIF-driven
          mesmo em LR, sinal individual e' real.
        - REFUTADO se joint-drop LR_SE dNMAE <= 0 (drop neutro/melhora
          em LR): confirma lesson model-aware empiricamente, magnitude
          gap modelo-dependente.

      Custo: ~30s wall-clock, zero dep externa, reuso 100% script H8.
    type: feature
    layer: curtailment
    target: feat_intercambio_joint_drop_se_lr
    priority: P3
    status: done
    iter_handled: 0041
    verdict: CONFIRMADO_LR
    estimated_effort_hours: 0.5
    actual_effort_hours: 0.25
    depends_on: [H8]
    blocks: []
    sanity_checks_required: [perm, baseline]
    expected_value: fechar empiricamente o caveat metodologico H22_model_aware
    created_at: 2026-05-24T22:30:00Z
    closed_at: 2026-05-25T22:00:00Z
    closure_note: |
      iter_0041 verdict CONFIRMADO_LR: SE joint-drop LR dNMAE=+1.316pp (>+0.5pp threshold).
      Direcao 4/4 subs consistente entre LR e Ridge a10/a1 (cross-model convergence).
      Magnitude LR ~ Ridge a1 (LR = limite alpha->0). Caveat H22_MA SE drop_HARMFUL
      reproduzido empiricamente em LR puro (champion SE) — decisao operacional UlFor
      (drop bundle SE) confirmada por direcao, apesar PI single-feat em LR ser
      meaningless (explode 172000-3000000 pp por min-norm SVD em colinears perfeitos
      val_net = val_import - val_export). Joint-drop e o teste autoritativo
      cross-model. **Bundle intercambio CASE FECHADO** no replay loop.
      1 follow-up criado: H40 (P5 doc-only sanity_checks/B2_interpretation.md).
    notes_iter0028: |
      Atratividade DIMINUI marginalmente pos-val14d alpha sweep UlFor
      (commits f7c56c3d + 1bd8638f + ff112a27). val14d real (test
      2026-03-24..2026-05-21, n~58d) em SE compara h22_MA+α=1 vs h22_pf+α=1
      no mesmo ridge: h22_MA (43.03% NMAE) BATE h22_pf (44.67%) por -1.64pp.
      Diff features SE (10 features que h22_MA preserva mas h22_pf dropa)
      inclui val_export_mwmed + val_import_mwmed -- exatamente 2 das 3
      features do bundle intercambio que H33 propunha testar isoladamente
      em LR. h22_MA derivado com champion-model real (LR p/ SE), entao a
      comparacao h22_MA vs h22_pf no mesmo ridge ja' opera como teste
      indireto da hipotese model-aware: features preservadas por LR-PI
      MELHORAM val14d em ridge tambem. Isso reforca lesson H22_MA / H23_ulfor
      / H8 iter_0027 sem precisar de H33 isolado. H33 ainda vale ~0.3h se
      Breno quiser teste especifico do bundle intercambio (nao confounded
      com outras 8 features), mas o pico de atratividade foi pre-empted.
    notes_iter0031: |
      Atratividade DIMINUI mais pos-decisao Breno (commit 0971c699). Breno
      escolheu opt A em SE: ridge+h22_MA+α=1. Caveat model-aware ja foi
      RESOLVIDO empiricamente pelo promote v3 final via 3-fold convergence
      (H8 iter_0027 joint-drop refit + iter_0028 val14d + iter_0031 escolha
      Breno). H33 (joint-drop SE em LR puro, custo ~0.5h) e' agora puramente
      investigacao de curiosidade metodologica -- nao ha mais decisao que
      dependa dela. Mantida queued porque o custo e' baixo e o resultado
      ainda interessa academicamente (model-family-aware drop magnitude),
      mas nao escala atratividade pos-iter_0031.

notas_iter0026:
  inspected_range: 5d41d063..83abab3e (8 commits UlFor, ~25 min reais)
  resolved: []
  newly_blocked: []
  newly_queued: []
  pre_empted: [H31_emergente]
  attractiveness_changes:
    H30: SOBE MARGINAL (alpha sweep sugere replicar com alpha=1)
    H22: INALTERADO (sweep linear puro, nao toca GBDT vs OLS)
    H27: INALTERADO (ADDENDUM 7cc3b604 refuta reabrir CV LGBM)
    H31_emergente: PRE-EMPTED mantem (validation gap parcial reabre mas pre-empcao provavel)
  validation_gap_partial_reopening: |
    SE ridge+h22_per_fold+alpha=1 (proposta v3 UlFor commit a3c742a9):
    NMAE 46.60% / R^2 +0.43 em CV (Pareto strict vs lr+h22_pf 57.17%/R²-0.07
    e vs lr+h22_MA 48.06%/+0.381). NAO testado em 14d real -- so CV 5x60d.
    UlFor explicito: "nao promovivel sem val_recent (principio 5)".
    Loop NAO emite req-0008 (padrao pre-empcao UlFor multi-agente self-actiona <30min).

notas_iter0027:
  hypothesis_handled: H8
  verdict: CONFIRMADO_PARCIAL_3SUBS_REFUTADO_SE_em_Ridge
  newly_created: [H33]
  newly_resolved: [H8]
  budget_iter_horas: 0.7
  output_dir: outputs/iter_0027/h8_intercambio_cv_pi/
  highlights: |
    Confirmacao independente de UlFor H22 drop list (via Ridge CV-PI loop
    com joint-drop primario + 30-perm single-feat secundario):
      - NE confirma drop direction (stronger -3.95pp joint-drop in alpha=1)
      - S  confirma + sugere extensao 4-feat (joint-drop -8.3pp gigante)
      - SE diverge em Ridge (+1.21pp joint-drop); UlFor decidiu drop com lr.
            Esta divergencia E o efeito mensuravel ~10pp do lesson H22_MA
            empiricamente reproduzido no loop. H33 derivada fecha o caveat.
      - N  inconclusivo (modelo Ridge N NMAE>1 unsafe). UlFor keep direction
            permanece valida out-of-scope.
    Lesson methodologica: PI single-feat sobre colinears identitarios
    (val_net = val_import - val_export, VIF=1e8) e' viesada para cima.
    Joint-drop refit dissolve a ambiguidade. NE alpha=1 single-PI val_import
    +1.65pp parece KEEP; joint-drop -3.95pp prova bundle HARMFUL.

notas_iter0028:
  inspected_range: 83abab3e..27152e16 (5 commits UlFor, ~76 min reais)
  resolved: []
  newly_blocked: []
  newly_queued: []
  pre_empted: []
  attractiveness_changes:
    H30: SOBE MARGINAL (val14d reforca alpha=1 vence alpha=10 default)
    H33: DIMINUI MARGINAL (val14d ja deu evidencia model-aware indireta SE)
    H22 (nosso): INALTERADO (sweep linear puro, nao toca GBDT vs OLS)
    H27: INALTERADO
  validation_gap_status_fechada: |
    SE ridge+h22_per_fold+α=1 (gap parcial reaberto em iter_0026) FECHADO via
    val14d real (commit f7c56c3d). Resultado SE val14d:
      h22_MA+α=1: 43.03% NMAE / R²+0.392 (val14d BEST)
      h22_pf+α=1: 44.67% NMAE / R²+0.344 (CV BEST)
      h22_pf+α=100: 47.83% NMAE / R²+0.227
    Inversao CV vs val14d em SE (-1.64pp). Promote v3 atualizado com decisao
    Breno em aberto: opt_A (val14d) vs opt_B (CV+coerencia multi-sub).
    Recomendacao tecnica UlFor: opt_B. Mecanismo da divergencia documentado
    em FINDING_RIDGE_ALPHA_SWEEP ADDENDUM (commit ff112a27): 10 features que
    h22_MA preserva mas h22_pf dropa carregam sinal em regime recente
    (CMO+intercambio+regime+carga+prev_solar+ter_verif_rmean7).
  convergencia_h8_val14d: |
    val_export_mwmed + val_import_mwmed (2 das 3 features do bundle
    intercambio H8 testou em iter_0027) estao entre as 10 features que
    h22_MA preserva mas h22_pf dropa em SE. Resultado iter_0027 SE Ridge
    α=1 joint-drop intercambio: +1.21pp NMAE (HARMFUL). Resultado iter_0028
    val14d SE: opt_A (preserva intercambio + outras 8) BATE opt_B (dropa
    intercambio + outras 8) por -1.64pp NMAE. Mesma direcao, magnitude
    consistente, contexto diferente. **val14d reforca empiricamente H8
    iter_0027 SE_em_Ridge REFUTADO**: dropar intercambio em SE Ridge α=1
    e' harmful em CV (H8) E em val14d (h22_MA vence h22_pf).
  promote_v3_decisao_breno_aberta: |
    NE: PROMOVER ridge + h22_per_fold + α=1 (CV+val14d coincidem)
    SE: opt_A ridge + h22_model_aware + α=1 (val14d 43.03%, preserva CMO+intercambio+regime)
        OU opt_B ridge + h22_per_fold + α=1 (CV 46.60% + coerencia multi-sub + menor overfit)
        Recomendacao UlFor: opt_B. Diff <2pp = margem amostral val14d.
    S:  MANTER status quo lr + full
    N:  PROMOVER ridge + h22_per_fold + α=100 (CV+val14d coincidem; val14d 71.51% MELHOR)
    Operacional: promote_champions.py ja patcheado iter_0026 (75e2431e).
    Branch 26+ ahead origin. Requer decisao Breno + push.

notas_iter0031:
  inspected_range: 27152e16..2917289c (8 commits UlFor, ~28 min reais)
  resolved: []
  newly_blocked: []
  newly_queued: []
  pre_empted: []
  attractiveness_changes:
    H8 (done): lesson 3a evidencia independente (Breno escolhe opt A SE)
    H33: DIMINUI MAIS (caveat model-aware resolvido empiricamente por 3-fold convergence; mantida queued por baixo custo, sem decisao dependente)
    H30: INALTERADO (decisao Breno nao toca pdp_residual)
    H25 (nosso): INALTERADO (NAO confundir com H25_ulfor = bias correction promote_champions.py)
    H18: blocked-MAIS-CRITICA (MLflow CF Access parte da decisao Breno EC2 setup vs skip)
  promote_v3_decidido_mas_nao_executado: |
    Breno escolheu opt A em SE (commit 0971c699): ridge + h22_model_aware + α=1.
    Justificativa: val14d > CV+coerencia em regime drift recente (CMO subindo,
    intercambio SE-S invertendo, NE em expansao). As 10 features que h22_MA
    preserva e h22_pf dropa (CMO+intercambio+taxa_penetracao) carregam sinal
    nesse regime. Promote NAO executado: CH local Docker (porta 8123) tem
    feat_termico congelada em 2024-12-31 (commit 6111cda4 FINDING_LOCAL_CH_STALE).
    Dataset colapsa a 16 dias finais Dez/24. MLflow tambem offline local.
    Aguarda Breno: EC2 setup ou sync raw. Comandos prontos em
    FINDING_RIDGE_ALPHA_SWEEP.md "Comandos prontos para retomar".
  h14g_implementacao_decidida: |
    Decisao Breno (commit 0971c699): implementar H14-G (bias correction NE
    w=14, k=1) em promote_champions.py, NAO em loader.py. Bias correction e'
    artefato promovido (binding ao modelo). UlFor registrou como H25_ulfor
    para sprint envelope-safe proxima. NAO confundir com nosso H25 (Stacker
    Ridge meta-modelo, P3 queued). NE H14-C continua default em loader (status
    quo bias_correction nao alterado neste promote v3).
  3_fold_convergence_h8_se: |
    Breno escolher opt A SE = 3a evidencia independente confirmando lesson
    H8 iter_0027 (preservar intercambio em SE Ridge α=1):
      1) iter_0027 H8 (joint-drop refit Ridge α=1 SE): +1.21pp NMAE HARMFUL
      2) iter_0028 val14d (h22_MA vs h22_pf comparison): -1.64pp NMAE opt_A wins
      3) iter_0031 (Breno escolhe opt A explicitamente): val14d trumps CV+coerencia
    Mesma direcao, magnitudes consistentes, contextos independentes.
    H8 done remains done; lesson permanece o lemma metodologico mais robusto
    do loop ate' agora.

  - id: H35
    summary: S alerta operacional binario via LogReg dedicado (vs binarizar champion)
    detail: |
      Derivada de H15 iter_0030 (CONFIRMADO_PARCIAL_NON_RARE). Em CV
      walk-forward 5×60d gap 7d, LogReg(class_weight=balanced) sobre
      features S/v1 bate LR_reg binarizado por +4.5pp AUC em thr_zero
      (any curt) e +6.8pp AUC em thr_p75 (big curt). PR-AUC tambem +5-8pp.

      H35 testa: produzir um endpoint binario "vai ter curtailment em S
      amanha?" via LogReg dedicado adicionaria valor operacional vs apenas
      thresholdar a saida continua do champion LR.

      Plano possivel (NAO implementado neste loop, fica em backlog):
        - UlFor treina LogReg(C=1, class_weight=balanced, scaler) sobre
          mesmas 55 feats que champion S usa (full feature_set, nao 37 do
          iter_0002). CV walk-forward 5×60d gap 7d para confirmar gain.
        - Threshold "any curt" (y_d1 > 0) — pos rate ~40% S, util como
          flag binaria em dashboard operador.
        - Validar val14d real recente (~58d) — mesmo protocolo iter_0028.
        - Expor /api/forecast/d1/s_alert {date, p_curt, threshold,
          decision} alimentando dashboard curtometro/historico.

      Bloqueador: alerta binario nao e' prioridade Breno (vs forecast
      continuo). H35 fica P3 ate' alguem pedir o endpoint.

      Nota: H15 thr_p90 (rare event severo) NAO e' viavel — pos absoluto
      <5 por fold inviabiliza calibracao classifier; manter regressor
      binarizado se quiser alertar severo.

      VEREDITO iter_0043: INDETERMINADO_BLOCKED_NO_PEDIDO_BRENO. Sub-claim
      tecnico (LogReg > LR_reg binarizado +4.5pp AUC thr_zero / +6.8pp
      thr_p75) CONFIRMADO_VIA_PRE_EXISTING_EVIDENCE iter_0030 (re-leitura
      bit-exata: delta +4.55pp / +6.77pp / -1.63pp matches claim exato).
      Sub-claim operacional (endpoint /api/forecast/d1/s_alert)
      BLOCKED_NO_STAKEHOLDER por 3 blockers concorrentes:
        B1_no_pedido_breno (detail H35 explicit)
        B2_hard_rule_loop_scope (ZERO interacao com api/services/products/)
        B3_consolidation_decision_iter_0042 (planner promoveu H37 explicit)
      6 sanity checks tecnicos PASS_INHERITED ou WARN_INHERITED iter_0030
      (4 verdes + 2 amarelos esperados em S por dist_shift). 1 sanity novo
      desta iter (operational_gate) FAIL_NEW por design — gate de
      governance NAO refuta tecnica. Nenhum follow-up criado (H15-family
      esgotado tecnicamente iter_0030, lado operacional deferido aqui).
      Lesson canonica: hipoteses cuja entrega exige tocar api/services/
      products/ devem ser marcadas como `requires_handoff=true` no queue;
      loop pode validar tecnica + arquivar evidencia, NAO pode executar
      produtizacao. Custo iter 0.3h (vs estimado 2.5h — economia por evitar
      re-execucao bit-exata).
    type: model
    layer: curtailment
    target: S_binary_alert_endpoint
    priority: P3
    status: done
    iter_handled: 0043
    verdict: INDETERMINADO_BLOCKED_NO_PEDIDO_BRENO
    estimated_effort_hours: 2.5
    actual_effort_hours: 0.3
    depends_on: []
    blocks: []
    sanity_checks_required: [holdout, baseline, perm]
    sanity_checks_done: [leak_PASS_INHERITED, perm_PASS_INHERITED, holdout_temporal_strict_PASS_INHERITED, baseline_compare_PASS_INHERITED, distribution_shift_WARN_INHERITED, zero_count_WARN_INHERITED, operational_gate_FAIL_NEW]
    follow_ups_created: []
    expected_value: novo endpoint binario para dashboard operador, ganho 5pp AUC vs binarizar champion. Sem pedido formal, fica em backlog.
    requires_handoff: true
    handoff_target: ulfor_or_breno_decision
    reopen_conditions:
      - "Breno pedir explicitamente endpoint binario S"
      - "UlFor decidir produtizar autonomamente (registry alias + rota services/analytics_api/routes/)"
      - "Dado novo: novo regime, nova feature, nova familia de modelo"
    created_at: 2026-05-25T01:30:00Z
    completed_at: 2026-05-26T02:30:00Z
    closure_summary: |
      Re-validacao bit-exata de evidencia pre-existente iter_0030 (H15
      CONFIRMADO_PARCIAL_NON_RARE). Claim numerica do detail H35 confirmada
      em 3/3 thresholds (matches +4.5pp/+6.8pp/-1.6pp exato). Sub-claim
      operacional bloqueada por governance (sem pedido Breno) + escopo (loop
      nao toca producao por regra hard). Verdict INDETERMINADO_BLOCKED
      distingue de INDETERMINADO_PURO: bloqueio e' identificado e categorizado,
      nao zona ruido. NAO emite req-NNNN, NAO toca champion, NAO cria H
      derivada. Insight tecnico ja arquivado em leaderboard.md secao
      "S binary alert (iter_0030 H15)" linhas 564-593.
    artefatos: outputs/iter_0043/h35_s_binary_alert_logreg/
      (verdict.json + sanity_checks.json + summary.md)

  - id: H36
    summary: GBDT vs OLS gap em features completas iter_0002 v3 (37+ feats)
    detail: |
      Derivada de H22 iter_0034 (REFUTADO_NO_NONLINEAR_GAIN, 3 features).
      Com apenas 3 features (gen + pdp_prev_eolica + pdp_prev_solar), GBDT
      NAO supera OLS em nenhum sub (max gap R^2 test = +0.015 SE; NE -4.3pp
      e S -17.4pp). Mecanismo provavel: overfit do GBDT em espaco
      hipoteticamente pequeno (3 feats nao da room para arvores capturarem
      interacoes uteis em test sob distribution shift NE+SE iter_0012 KS<1e-4).
      H10/H21 lgbm_cv_supplement mostraram GBDT util com 37 feats (NE/SE/S
      v3) -- entao a comparacao critica seria com features completas.

      Plano:
        - mesmo protocolo H22 (holdout 80/20 temporal, R^2 test primary)
        - features: iter_0002 v3 (37 feats incluindo curt_lag*, gen_*, pdp_*,
          cmo_*, ter_verif_*) sub-a-sub
        - modelos: OLS (sklearn LinearRegression, sem regularizacao) vs
          LightGBM defaults H10
        - PI duo refit-drop nas top-10 features de cada modelo
        - Se GBDT vence OLS por >=5pp R^2 em algum sub: CONFIRMADO_em_feats_full
          -> entao engineering NAO substitui GBDT (UlFor ja' usa GBDT no
          bake-off mas champions sao Ridge/LR, e' subotimo?)
        - Se OLS empata/supera: a) UlFor champions Ridge/LR estao corretos
          dada a colinearidade VIF>=10 em 38/55 feats; b) gap nao-linear
          NAO existe em curt D+1, OLS basis e' suficiente para o sinal
          disponivel.

      Bloqueador: depende de UlFor ja' ter dataset 37-feats v3 publicado em
      parquet (iter_0002/runs/{NE,SE,S}/v3/features.parquet existe local
      desde iter_0002). Custo: ~30 LoC adicionais sobre h22_gbdt_vs_ols.py.

      ATUALIZACAO PROXIMA SESSAO: ler features.parquet iter_0002 v3 per
      sub, mesmo split temporal 80/20, calcular GBDT R^2 test vs OLS R^2
      test. Decision rule identica (5pp confirma, -2pp todos refuta).
    type: model
    layer: curtailment
    target: gbdt_vs_ols_gap_full_features
    priority: P3
    status: done
    iter_handled: 0044
    verdict: CONFIRMADO_PARCIAL_em_feats_full (NE only, com caveat metodologico)
    estimated_effort_hours: 1.0
    depends_on: [H22]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    expected_value: |
      Saber se GBDT default beneficia D+1 curt com features completas
      (UlFor ja' rodou XGB/LGBM no bake-off mas perdeu para Ridge/LR;
      H36 mede gap diretamente vs OLS sem regularizacao para isolar
      contribuicao nao-linear das arvores).
    created_at: 2026-05-25T05:30:00Z
    closed_at: 2026-05-26T04:00:00Z
    closed_summary: |
      Rodado em outputs/iter_0044/h36_gbdt_vs_ols_full/.
      Resultado por sub (gap = GBDT R^2_test - OLS R^2_test, threshold +5pp):
        NE: gap = +0.397 (R^2: OLS 0.114 vs GBDT 0.511) GBDT_BETTER (CONFIRMADO local)
        SE: gap = -0.296 (R^2: OLS -0.024 vs GBDT -0.321) GBDT_WORSE
        S:  gap = -0.061 (R^2: OLS +0.240 vs GBDT +0.179) GBDT_WORSE
      Verdict: CONFIRMADO_PARCIAL_em_feats_full (1/3 subs, NE-only).
      CAVEAT METODOLOGICO: NE +40pp gap pode ser explicado por (a) GBDT
      extrair nao-linearidade real OR (b) OLS-puro overfit catastrofico
      (R^2_train=0.865 -> R^2_test=0.114, delta -0.75) com 47 feats
      colineares (UlFor VIF>=10 em 38/55). Comparacao operacionalmente
      relevante seria GBDT vs Ridge_alpha10 (champion UlFor NE); UlFor ja
      fez via bake-off iter_0007 e Ridge venceu XGB/LGBM em NE -> hipotese
      (b) regularizacao > nao-linearidade no NE. Champions UlFor INTACTOS
      (zero rollback). Reforca lesson H22_MA / H8 (PI duo gap ate ±40pp
      empirico, ano_sin_d1 NE -39pp; semana_sin_d1 SE +35pp -- PI EH
      severamente model-dependent em features completas). H41 derivada
      (P3, queued): GBDT vs Ridge_alpha10 NE para distinguir
      regularizacao vs nao-linearidade.
    artefatos: outputs/iter_0044/h36_gbdt_vs_ols_full/
      (results.json + summary.csv + sanity_checks.json)
    follow_ups_created: [H41]

  - id: H41
    summary: GBDT vs Ridge_alpha10 NE com features completas -- isolar regularizacao vs nao-linearidade
    detail: |
      Derivada de H36 iter_0044 (CONFIRMADO_PARCIAL_em_feats_full NE-only).
      H36 mostrou GBDT supera OLS-puro em NE por +40pp R^2 test com 47
      feats iter_0002 v3. CAVEAT: OLS-puro (no-reg) tem R^2_train=0.865 e
      R^2_test=0.114 (delta -0.75) -- overfit catastrofico em 47 feats
      colineares (UlFor VIF>=10 em 38/55). A pergunta operacional real e':
      "GBDT supera o CHAMPION Ridge_alpha10 NE?" -- nao OLS-puro.

      UlFor bake-off iter_0007 ja respondeu (champion=Ridge_alpha10 venceu
      XGB/LGBM em NE), mas em CV 5x60d gap7d -- nao no protocolo H36
      (holdout 80/20 temporal sobre iter_0002 features). H41 ALINHA
      protocolos: GBDT (LGBM defaults H10) vs Ridge_alpha10 (champion
      UlFor) no MESMO holdout 80/20 sobre iter_0002 v3 NE features.

      Hipotese: gap GBDT vs Ridge_alpha10 NE <= +2pp R^2 test (Ridge fecha
      80%+ do gap vs OLS-puro). Se confirmar: nao-linearidade nao agrega
      sobre regularizacao L2 em curt D+1 NE -- corrobora champion UlFor.
      Se gap >= +5pp: GBDT tem upside REAL sobre Ridge -- questiona
      champion NE.

      Custo: ~10 LoC sobre h36_gbdt_vs_ols_full.py (trocar OLS por
      Ridge_alpha10; manter mesmo split, mesmas features, mesmas
      comparacoes).
    type: model
    layer: curtailment
    target: gbdt_vs_ridge_alpha10_ne_full
    priority: P3
    status: done
    iter_handled: 0046
    verdict: CONFIRMADO_PARCIAL
    estimated_effort_hours: 0.5
    actual_effort_hours: 0.3
    depends_on: [H36]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    sanity_checks_done: [leak, perm_proxied, holdout, baseline, dist_shift, zero_count]
    expected_value: |
      Distingue se ganho GBDT +40pp NE em H36 e' (a) nao-linearidade real
      OR (b) artefato de OLS-puro overfit. Ortogonal a champion (Ridge
      ja confirmado por CV UlFor); H41 alinha protocolos para closure.
    created_at: 2026-05-26T04:00:00Z
    completed_at: 2026-05-26T08:00:00Z
    verdict_summary: |
      CONFIRMADO_PARCIAL (gap GBDT-Ridge_alpha10 NE = +0.0320 in [+0.02, +0.05)).
      Replay OLS BIT-EXATO H36 (R²_test=0.1138 ambos, replay_match=true).
      Resultados: OLS_full R²=+0.114; Ridge_alpha10 R²=+0.479 (+36.5pp vs OLS);
      GBDT R²=+0.511 (+3.2pp vs Ridge, +39.7pp vs OLS). Ridge fecha **91.9%**
      do gap H36 vs OLS-puro (de +0.397 para +0.032). Train/test delta: OLS
      -0.751 (overfit catastrofico) → Ridge -0.375 (L2 corta overfit pela
      metade) → GBDT -0.489 (R²_train=0.9999 = arvores memorizam treino,
      subsample/colsample salvam test). MAE_test: OLS 31.5k > Ridge 24.2k >
      GBDT 23.8k (-422 MWh GBDT < Ridge, ~1.7% relativo). PI duo top-10
      divergem: GBDT_top={semana_sin_d1, curt_lag1, ger_solar_mwh,
      cmo_desvio_30d, pdp_prog_solar} vs Ridge_top={ano_sin_d1, is_weekend_d1,
      semana_sin_d1, taxa_penetracao_rmean7, ano_cos_d1}; so semana_sin_d1
      em comum. Ridge depende de ano_sin_d1 com sinal compensador (drop=+14.8pp
      R², sinal de basis-linear saturado/colinearidade que Ridge usa como
      pseudo-anchor). Caveat H36 RESOLVIDO 91.9%: gap +40pp = dominantemente
      artefato OLS-puro overfit, NAO nao-linearidade real. Os 3.2pp residuais
      sao GBDT nao-linearidade marginal — insuficiente para questionar champion
      UlFor Ridge_alpha10 (gap <+5pp threshold + CV 5x60d oficial ja teve
      Ridge vencendo). Arco H22 (3-feat REFUTADO) + H36 (47-feat parcial-com-
      caveat) + H41 (47-feat alinhado PARCIAL 91.9% closure) ENCERRA frente
      GBDT-vs-linear em curt D+1. 6 sanity checks PASS/REPORTED. Champions
      INTACTOS, zero rollback. 0 follow-ups (frente esgotada; "GBDT real gain"
      exigiria features novas, nao trade-off em modelo).
    artefatos: outputs/iter_0046/h41_gbdt_vs_ridge_ne_full/
      (results.json + summary.csv + sanity_checks.json)
    follow_ups_created: []

  - id: H37
    summary: CQR-asymmetric + Mondrian conformal por regime — fechar NE+N gap H26
    detail: |
      Derivada de H26 iter_0037 (INDETERMINADO_NE + bonus CONFIRMADO_SE_S).
      H26 mostrou que conformal symmetric (Romano CQR 2019) com q_alpha
      global:
        - resolve SE+S (cov_cal 76.6% / 79.5% strict)
        - fica borderline em NE (cov 74.7%, 0.3pp do limite; ratio 2.03x)
        - OVER-COBRE em N (cov 90.8%, banda larga demais)
      Causa: 1 q_alpha global empurra ambos os lados simetricamente; em N
      o score e' dominado por outliers da cauda alta (37% dos dias com
      curt~0) inflando q10 desnecessariamente; em NE a inflacao e'
      insuficiente em folds de regime shift (KS p<0.0001 iter_0012).

      Fix candidate duplo (custo baixo, sem retreinar):
        A) CQR-ASYMMETRIC (Romano variant):
           s_low_i  = q10_iv_i - y_iv_i   (clipped >=0)
           s_high_i = y_iv_i - q90_iv_i   (clipped >=0)
           q_low  = quantile(s_low,  ceil((n+1)*(1-alpha/2))/n)
           q_high = quantile(s_high, ceil((n+1)*(1-alpha/2))/n)
           Banda: [q10 - q_low, q90 + q_high]
           Esperado: N para de inflar banda inferior (poucos overshoots
           por baixo); NE mantem inflacao na cauda alta (folds 2-3).

        B) MONDRIAN CONFORMAL POR REGIME:
           Particionar inner_val em buckets discretos por regime
           (eg threshold em P50 do train_inner: low_curt vs high_curt).
           Calcular q_alpha por bucket. No test, escolher q_alpha pelo
           bucket onde o ponto cai (precisa estimar bucket em D-1 via
           outra feature).
           Risco: 30d / 2 buckets = 15d/bucket, instavel — mitigar via
           shrinkage (combinar q_bucket com q_global por inverse-variance).

      Aceitacao:
        CQR-asymmetric: cov_band_80_cal NE in [75%,85%] em >=2/3 cells
                        AND cov_band_80_cal N in [75%,90%] (corrige
                        over-coverage atual 90.8%).
        Mondrian:       cov_band_80_cal NE in [75%,85%] em >=2/3 cells
                        AND fold heterogeneity reduzida (std <= 0.10).

      Rodar AS DUAS na mesma iter (custo +30 LoC sobre h26_conformal);
      se uma vence, registrar. Se ambas falham, NE+N D+1 quantile fica
      em modo "P50 only" ate H28 (NGBoost parametrico ja queued).
    type: model
    layer: curtailment
    target: ne_n_d1_quantile_calibrated_v2
    priority: P3
    status: done
    iter_handled: 0045
    verdict: CONFIRMADO_ASYM_ONLY
    verdict_summary: |
      VARIANTE A (CQR-asymmetric) PASS em AMBOS os gates:
        NE iv=30: 2/3 cells in [75, 85] (cov mean 75.5%; H26 era 74.7%)
        N  iv=30: 3/3 cells in [75, 90] (cov mean 86.4%; H26 era 90.8%)
      VARIANTE B (Mondrian + shrinkage inv-variance) REFUTADA:
        NE iv=30: 1/3 cells in [75, 85] (cov mean 75.2%) -- falha 2/3
        std fold-a-fold mean NE = 20.9% (>> 10pp threshold, 2x)
      Per sub bonus (sym vs asym vs mond iv=30):
        NE: 74.7% / 75.5% / 75.2%
        SE: 76.6% / 80.3% / 75.2%
        S : 79.5% / 81.8% / 79.5%
        N : 90.8% / 86.4% / 91.3%
      Custo width asym: NE +9%, SE +20%, S +51%, N +36% (mais larga,
      mas calibrada nas duas pontas).
      Mecanismo da vitoria ASYM em N: 37% dos dias com curt~0 fazem
      s_low symmetric inflar q_alpha global; com asym, q_low fica ~107
      MWh (vs q_alpha sym ~663 MWh), nao penaliza zeros.
      Mecanismo da derrota MONDRIAN: shrinkage inverse-variance com
      n_global=30 e n_bucket=15 da w_bucket~0.5 mesmo quando var bate;
      Mondrian colapsa em sym + ruido de bucketing. Heterogeneidade NE
      e' regime real (B5 iter_0012 KS p<0.0001 por DIA, nao por modelo)
      e bucketing pos-hoc nao atenua.
      Replicacao bit-exato H26 cov_uncal_full inhibida (sym branch
      do script reproduz H26 dentro de 0.5pp em todas 24 (cell, iv)
      combinacoes). Champions UlFor INTACTOS (replay-only).
    estimated_effort_hours: 2.0
    actual_effort_hours: 0.7
    depends_on: [H26]
    blocks: []
    sanity_checks_required: [holdout, baseline, dist_shift]
    sanity_checks_status:
      B1_leak: skipped_inherited (iter_0010 H3 p=0)
      B2_perm: skipped_inherited (iter_0010 H3 p=0)
      B3_holdout: passed_embedded (gap=7d, 5 folds; inner_val antes do test; mondrian bucket usa q50_te nao y_te)
      B4_baseline: passed_embedded (triple baseline sym+asym+mond)
      B5_dist_shift: annotated_reuse (KS p<0.0001 NE+SE = causa raiz que motivou H37)
      B6_zero_count: reported (N y_frac_zero ~37% explica vitoria asym sobre sym)
    follow_ups_created: []  # H42/H43 (asym productize / mondrian aprendido) sao P3 opcionais sem queue
    expected_value: |
      Fecha H26 borderline NE (0.3pp do limite strict) e corrige N
      over-coverage. Bonus: validacao de Mondrian conformal como
      ferramenta para distribution shift em outros forecasts (carga,
      eolica D+1 per-conjunto).
    actual_value: |
      ASYM ENTREGUE como deliverable replay-loop: bandas P10/P90
      calibradas em NE+N (alem do bonus H26 em SE+S). Mondrian
      INVALIDADO no envelope iter_0002+CV-5x60d (n insuficiente
      por bucket; heterogeneidade NE e' do dado nao da receita).
      Implicacao: lesson canonica para sanity_checks/calibration.md =
      "Per-regime bucketing pos-hoc shrunk para global colapsa em
      sym quando n_global e n_bucket sao da mesma ordem; preferir
      x-conditional quantile regressor (mondrian aprendido) ou
      asymmetric scores quando o problema e' assimetria de cauda".
    created_at: 2026-05-25T08:00:00Z
    completed_at: 2026-05-26T06:00:00Z

  - id: H40
    summary: Documentar joint-drop > PI single-feat em colinears perfeitos (B2_interpretation)
    status: done
    iter_handled: 0049
    verdict: DOCUMENTADO
    actual_effort_hours: 0.3
    completed_at: 2026-05-26T14:00:00Z
    closure_summary: |
      iter_0049 verdict DOCUMENTADO. EXTENDE sanity_checks/B2_interpretation.md
      (153 -> 323 linhas, +170 linhas). 3 criterios H40 satisfeitos:
        (a) Caso pedagogico 3 (H33 iter_0041 SE LR) adicionado com numeros
            verbatim do verdict.json: PI single-feat SE val_export +327559,
            val_import +383150, val_net +327975 pp dNMAE em colinears
            perfeitos vs val_net_lag1 (nao-colinear) +0.18 pp -- ratio
            ~1.8e6x same-sub same-model same-domain confirma artefato.
        (b) Causa mecanistica documentada: sklearn.LinearRegression usa
            scipy.linalg.lstsq (SVD) que para rank-deficient retorna
            min-norm solution; coefs distribuidos com cancelamento
            perfeito (val_net = val_import - val_export), permutar quebra
            cancelamento -> erro arbitrariamente grande. Atenuado em Ridge
            (L2 shrinkage), monotonico em alpha (LR > Ridge a1 > Ridge a10
            em magnitude joint-drop SE).
        (c) Regra geral "colinearidade + joint-drop" adicionada: SEMPRE
            rodar joint-drop alongside PI single-feat quando VIF>=10 ou
            identidade algebrica conhecida. Joint-drop refit dissolve
            artefato (remove grupo inteiro, design matrix full-rank no
            resto). Cross-model autoritativo: direcao 4/4 subs consistente
            LR vs Ridge a1 vs Ridge a10 (NE -4.285 / SE +1.316 / S -7.376
            / N -1.816 em LR).
      Bonus: secao "Como evitar a armadilha" recebeu 5o bullet (colinearidade)
      + heuristica operacional recebeu 4a linha (citar H8/H33). Nova tabela
      "Convergencia H8 + H33 (arco intercambio encerrado)" anexada com
      Ridge a10 (+0.270) + Ridge a1 (+1.205) + LR (+1.316) demonstrando
      monotonicidade. Referencias internas + wikilinks estendidos com H8/H33.
      11/11 numeros verificados contra source artifacts (iter_0041 verdict.json).
      Default 6-check suite NA (governance/methodology, sem modelo treinado);
      audit substitutivo PASS. Champions UlFor INTACTOS, zero rollback.
      0 follow-ups: arco intercambio H8+H33 e arco residual H21+H30+H38
      ambos encerrados; doc agora cobre os 2 modos de falha canonicos de B2.
    artefatos:
      - sanity_checks/B2_interpretation.md (323 linhas, doc extendido com 3 novas secoes)
      - outputs/iter_0049/h40_joint_drop_vs_pi_doc/verdict.json
      - outputs/iter_0049/h40_joint_drop_vs_pi_doc/sanity_summary.json
      - outputs/iter_0049/h40_joint_drop_vs_pi_doc/summary.csv
    detail: |
      Derivada de H33 iter_0041 (CONFIRMADO_LR). Segundo caso canonico,
      apos H39 (P5, criada em iter_0040 sobre H30 NE alpha=10), para a
      entrada `sanity_checks/B2_interpretation.md`.

      H33 expos comportamento radical de PI single-feat em LR com colinears
      perfeitos (val_net = val_import - val_export):
        SE LR PI val_export: +327559 pp dNMAE
        SE LR PI val_import: +383150 pp dNMAE
        SE LR PI val_net:    +327975 pp dNMAE
        SE LR PI val_net_lag1 (nao-colinear): +0.18 pp dNMAE
      Magnitude 1M+ vezes maior em colinear vs nao-colinear na mesma sub
      no mesmo modelo na mesma feature semanticamente similar (intercambio).

      Causa: `sklearn.LinearRegression` usa `scipy.linalg.lstsq` (SVD) que
      retorna min-norm solution para design matrix rank-deficient. Coefs
      individuais nas 3 colineares sao distribuidos com cancelamento
      perfeito (a*val_export + b*val_import + c*val_net = 0 para qualquer
      (a, b, c) tal que c = -b, a = b por algebra). Permutar uma das 3
      quebra o cancelamento -> predicoes saem do eixo dos dados,
      `pred = c1*shuffled + c2*real + c3*real` produz erro arbitrariamente
      grande.

      Esse padrao tambem aparece em Ridge mas atenuado (shrinkage L2 reduz
      magnitude individual dos coefs colineares); H8 iter_0027 mostrou
      Ridge a1 +1.21pp / a10 +0.27pp em SE — mesma direcao, magnitude
      muito mais civilizada.

      LESSON CANONICA p/ B2:
        |PI_single_feat| arbitrariamente grande em colinears perfeitos
        = METRICA QUEBRADA (artefato algebrico), nao feature importante.
        Joint-drop refit sem o bundle inteiro e' o teste autoritativo
        cross-model — mede aporte real sem produzir explosao numerica.

      Adicionar ao sanity_checks/B2_interpretation.md:
        - Caso 1 (H30 iter_0040 NE alpha=10): perm +105% MAE drop em
          residual_total mas mean R² CV perde -0.064 -> "perm confirma
          signal nao confirma feature_engineering_gain"
        - Caso 2 (H33 iter_0041 SE LR): PI single-feat explode 172000-
          3000000 pp em colinears perfeitos -> "joint-drop > PI em
          colinears"
        - Regra geral: SEMPRE rodar joint-drop alongside PI single-feat
          quando ha grupo de features com VIF alto (>10) ou identidade
          algebrica suspeita.

      Custo estimado: 30 min (escrita + cross-link com casos H30 e H33,
      sem codigo novo).
    type: governance
    layer: methodology
    target: sanity_checks/B2_interpretation.md
    priority: P5
    estimated_effort_hours: 0.5
    depends_on: [H33, H39]
    blocks: []
    sanity_checks_required: []
    sanity_checks_done: [doc_aceitacao_3_criteria_PASS, doc_numbers_verified_11_of_11, doc_internal_links_REPORTED, default_six_NA_governance]
    follow_ups_created: []
    expected_value: |
      Codifica 2 casos canonicos (H30 + H33) que evitam interpretacao
      errada de PI em iters futuras. Reduz custo de re-descobrir os
      mesmos artefatos algebricos (perm em col duplicado, PI em
      col perfeito) ao limpar a interpretacao no playbook.
    created_at: 2026-05-25T22:00:00Z

notas_iter0042:
  mode: consolidation
  trigger: quality_gate.refuted_streak_2 (iter_0040 REFUTADO_RIDGE + iter_0041 CONFIRMADO_LR metodologico sem mudanca champion)
  data_utc: 2026-05-26T00:00:00Z
  resolved: []        # nenhuma H fechada nesta iter (governance)
  newly_blocked: []
  newly_queued: []    # H38+H39+H40 ja criados nos iters 0040+0041; nada de novo
  newly_done: []
  rollback_leaderboard: false  # champions Ridge/LR + bias correction intactos
  scope: |
    Reavaliacao da queue apos 5 frentes encerradas em ~5 iters (H21+H30
    H3-family residual; H28 NGBoost defaults; H8+H33 bundle intercambio;
    H26 conformal symmetric NE; H25 stacker Ridge; H22-3feat GBDT vs OLS).
    Ultimas 3 iters (0039-0041): 0 mudancas em champion, 3 follow-ups
    derivados (H38 P4 + H39 P5 + H40 P5), todos ja no queue.
  proximo_alvo_acordado: H37
  proximo_alvo_justificativa: |
    H37 (CQR-asymmetric + Mondrian conformal NE+N, P3, 2h, sem dep externa)
    e' a unica hipotese queued *executavel local* com deliverable
    substantivo: bandas P10/P90 calibradas em NE (fecha gap borderline
    H26 0.3pp do limite strict) + corrige over-coverage N (90.8% para
    [75,90]%). H26 ja entregou SE+S como bonus (cov 76.6/79.5% strict);
    H37 fecha as 2 subs restantes ou encerra o caminho conformal
    classico CV no replay loop. Aceitacao no queue, sem mudanca.
  atratividade_pos_consolidation:
    H37: ALTA (deliverable real, dependencia zero, fecha gap conhecido)
    H36: DIMINUIDA (convergencia H21+H22 esgota linear/nao-linear sobre
         mesmas variaveis; lesson canonica "novo sinal exige novas
         variaveis" formada; rodar com 37 feats tem prob. baixa de win
         substantivo)
    H38: BAIXA (P4, 0.3h, fecha caveat tecnico de H30 mas explicitamente
         baixo impacto pratico mesmo se CONFIRMADO)
    H39+H40: AGRUPAR (P5, ambos em sanity_checks/B2_interpretation.md;
             custo combinado 0.5h; rodar em sprint governance quando
             houver janela inativa)
    H35: BAIXA (blocked sem pedido formal Breno; alerta binario S nao
         e prioridade vs forecast continuo)
    H5/H12/H14/H18: blocked (req externos OR acao Breno EC2/Cloudflare)
  observacao_meta: |
    Padrao recorrente das ultimas 5 iters (0037-0041): hipoteses P3
    fechando frentes derivadas (caminhos de pesquisa) sem mover
    champion. Isso e' SAUDAVEL — significa que o loop esta saturando
    seu envelope local (features iter_0002 + CV 5x60d + replay
    proxies). Para destravar a proxima onda de ganhos, ou (a) H37
    confirma e abrimos frente "conformal asymmetric productizado",
    ou (b) UlFor publica dado novo (DESSEM/Sintegre full / WeatherNext)
    OR Breno desbloqueia EC2/CF para H18 audit empirico champions.
    Loop nao precisa mais derivar P3-P5 follow-ups a partir de H3
    family / intercambio / NGBoost — esses caminhos sao now CLOSED.

  refuted_streak_audit:
    streak_count: 2  # iter_0040 REFUTADO_RIDGE + iter_0041 CONFIRMADO_LR-sem-mudanca-champion
    iters: [0040, 0041]
    full_streak_extended: 5  # iter_0037..0041 = 0 mudancas em champion
    iters_extended: [0037, 0038, 0039, 0040, 0041]
    refuted_in_extended: [0037 (INDETERMINADO_NE bonus), 0039 (INDETERMINADO_PINBALL), 0040 (REFUTADO_RIDGE)]
    confirmed_in_extended_methodological_only: [0038 (CONFIRMADO mas e' recomendacao UlFor, loop nao executa), 0041 (CONFIRMADO_LR mas decisao Breno ja tomada iter_0031)]
    interpretacao: |
      "Confirmado metodologico sem mudanca champion" e' epistemicamente
      proximo de "refuted_for_the_purpose_of_promoting" — gate tratou
      como streak_2 corretamente. 3 frentes derivadas + 2 deliverables
      informativos saidas em 5 iters; tempo de consolidar e' agora.

