---
alvo: S_binary_alert_endpoint
layer: curtailment
iter_num: 0043
type: hypothesis_test (verdict=INDETERMINADO_BLOCKED_NO_PEDIDO_BRENO)
data_utc: 2026-05-26T02:00:00Z
hypothesis: |
  H35 (P3, queued desde iter_0030 — derivada de H15 CONFIRMADO_PARCIAL_NON_RARE):
  "Produzir um endpoint binario 'vai ter curtailment em S amanha?' via LogReg
  dedicado adicionaria valor operacional vs apenas thresholdar a saida continua
  do champion LR continuo."

  Sub-claim tecnico (H35 detail): em CV walk-forward 5x60d gap 7d sobre features
  S/v1, LogReg(class_weight=balanced) bate LR_reg binarizado por +4.5pp AUC em
  thr_zero (any curt) e +6.8pp AUC em thr_p75 (big curt). PR-AUC tambem +5-8pp.
  thr_p90 (rare event severo) NAO e' viavel — pos absoluto <5 por fold
  inviabiliza calibracao classifier.

  Sub-claim operacional (H35 detail): expor /api/forecast/d1/s_alert {date,
  p_curt, threshold, decision} alimentando dashboard curtometro/historico.

  Bloqueador explicito (H35 detail): "alerta binario nao e' prioridade Breno
  (vs forecast continuo). H35 fica P3 ate alguem pedir o endpoint."

  ACEITACAO (derivada do detail + regras hard-coded do loop):
    CONFIRMADO            : tecnico-CV >= +4.5pp AUC thr_zero E pedido formal
                            Breno/UlFor para produtizar
    INDETERMINADO_BLOCKED : tecnico-CV >= +4.5pp AUC thr_zero E sem pedido
    REFUTADO              : tecnico-CV < +4.5pp AUC thr_zero
baseline_tipo: |
  Pre-existing evidence iter_0030 H15 (CV walk-forward 5x60d gap 7d,
  S/v1 37 features iter_0002, LogReg vs LR_reg + Ridge_alpha10 binarizados +
  LGBMClassifier + persist_d1 binarizado + climat constant).
  Re-rodar bit-exato = duplicacao (mesmo script h15_s_classifier_vs_regressor.py,
  mesmos dados, mesmo split). H35 e' meta-pergunta sobre PRODUTIZAR o achado.
baseline_metric: |
  iter_0030 (H15 verdict.json) numeros exatos:
    thr_zero  : best_cls=logreg AUC 0.784 vs best_reg=lr_reg 0.739 (delta +4.55pp; PR-AUC +5.12pp)
    thr_p75   : best_cls=logreg AUC 0.821 vs best_reg=lr_reg 0.754 (delta +6.77pp; PR-AUC +8.11pp)
    thr_p90   : best_cls=logreg AUC 0.762 vs best_reg=lr_reg 0.778 (delta -1.63pp; PR-AUC -0.57pp)
  persist_d1 baseline binarizado thr_zero: AUC 0.617 (logreg bate por +16.76pp)
  permutation test (50 perms, fold 5 thr_zero): real AUC 0.872 vs perm 0.512+-0.13, p=0.000
result_metric: |
  Sub-claim tecnico: CONFIRMADO_VIA_PRE_EXISTING_EVIDENCE_iter_0030
    +4.55pp AUC thr_zero (matches H35 detail claim "+4.5pp")
    +6.77pp AUC thr_p75 (matches H35 detail claim "+6.8pp")
    -1.63pp AUC thr_p90 (matches H35 detail caveat "rare event NAO viavel")

  Sub-claim operacional: BLOCKED_NO_STAKEHOLDER_REQUEST
    B1 no_pedido_breno (detail H35 explicit)
    B2 hard_rule_loop_scope (zero interacao com api/services/products/)
    B3 consolidation_decision_iter_0042 (planner promoveu H37 explicitamente)

  Verdict global: INDETERMINADO_BLOCKED_NO_PEDIDO_BRENO
decision: DESCARTA — verdict fecha H35 como deferido sem mudanca operacional.
  Insight tecnico (LogReg dedicado supera regressor binarizado em S thr_zero/p75)
  permanece arquivado em leaderboard.md secao "S binary alert (iter_0030 H15)"
  com recomendacao consolidada para implementar se UlFor/Breno pedirem. NAO
  emite req-NNNN. NAO toca producao. NAO cria H derivada (caminho H15-family
  S classifier encerrado em iter_0030 tecnicamente; H35 fecha lado operacional).
sanity_checks_passed:
  leak: PASS_INHERITED                 # iter_0030 source, features inalteradas
  perm: PASS_INHERITED                 # iter_0030 fold 5 thr_zero p=0.000
  holdout_temporal_strict: PASS_INHERITED  # CV 5x60d gap 7d, walk-forward
  baseline_compare: PASS_INHERITED     # logreg bate persist por +16.76pp thr_zero
  distribution_shift: WARN_INHERITED   # max_zero_rate_shift_abs=0.37 — propriedade DOS DADOS S, nao do modelo
  zero_count: WARN_INHERITED           # 3/37 curt_lag* shift>0.2 em fold 5 — conhecido iter_0006 req-0002
  operational_gate: FAIL_NEW           # NOVO desta iter — gate decisivo H35; FAIL = sem pedido formal
budget_consumido_iter: 0.3
custo_estimado_usd: 0.0
artefatos:
  - outputs/iter_0043/h35_s_binary_alert_logreg/verdict.json
  - outputs/iter_0043/h35_s_binary_alert_logreg/sanity_checks.json
  - outputs/iter_0043/h35_s_binary_alert_logreg/summary.md
follow_ups_created: []
encerra_caminhos:
  - "H15-family S classifier": exploracao tecnica esgotada em iter_0030 (CV 6
    sanity PASS/WARN). H35 fecha o lado operacional como deferido sem pedido.
    NAO reabrir sem (a) Breno pedir endpoint, (b) UlFor produtizar autonomo,
    OU (c) dado novo (novo regime, nova feature, nova familia de modelo).
verdict_full: |
  INDETERMINADO_BLOCKED_NO_PEDIDO_BRENO per criterio H35 a priori:
    "CONFIRMADO se tecnico-CV >= +4.5pp AUC thr_zero E pedido formal Breno/UlFor"
    Tecnico-CV: 4.55pp PASSA threshold (matches detail claim exato).
    Pedido formal: AUSENTE (detail H35 explicit "alerta binario nao e prioridade
    Breno"; iter_0042 consolidation reafirma "H35 BAIXA blocked sem pedido Breno").
    -> Verdict: INDETERMINADO_BLOCKED (CONFIRMED-tech, BLOCKED-deploy).

  Por que NAO REFUTADO: REFUTADO exigia tecnico-CV < 4.5pp — nao e o caso.
  Por que NAO CONFIRMADO: CONFIRMADO exigia AMBOS (tecnico + pedido). Pedido falta.
  Por que NAO INDETERMINADO_PURO: a metrica tecnica e' DEFINITIVA via iter_0030;
  apenas o gate operacional bloqueia. INDETERMINADO_BLOCKED distingue do
  INDETERMINADO comum (zona ruido) — aqui sabemos exatamente onde o bloqueio
  esta (governance, nao dados).

  Loop hard rules conflitam diretamente com a implementacao do endpoint:
    "ZERO interacao com producao (api/, services/forecasting/, products/)"
  Mesmo se o tecnico fosse 10x melhor, o loop NAO pode produzir o deliverable
  H35 (endpoint) sem violar essa regra. Esta e' uma hipotese fundamentalmente
  HANDOFF-shaped — o loop entrega o achado, UlFor/Breno decide produtizar.
  iter_0030 ja entregou o achado.

  Custo desta iter: 0.3h (reconhecimento de evidencia pre-existente + 4 arquivos
  + 3 updates governance: queue, state, leaderboard). USD: 0.0 (zero compute).
  Padrao: hipoteses "endpoint downstream" devem ser detectadas no planner antes
  de selecionar — proxima geracao do selector pode usar campo 'deliverable_in_loop_scope'
  para evitar gastar iters em meta-perguntas de governance. Nota arquivada
  (NAO criada como H derivada por nao ter custo testavel — e' melhoria de
  selector, registrado em handoff sem virar item de queue).
---

# Iter 0043 — H35 S binary alert via LogReg dedicado

## Hipotese

H35 (P3 queued iter_0030, derivada de H15 CONFIRMADO_PARCIAL_NON_RARE) testa
se produzir um endpoint binario `/api/forecast/d1/s_alert` via LogReg
dedicado adicionaria valor operacional ao dashboard curtometro/historico vs
apenas thresholdar a saida continua do champion LR continuo do sub S.

O sub-claim tecnico do detail H35 ("LogReg bate LR_reg binarizado por +4.5pp
AUC em thr_zero e +6.8pp AUC em thr_p75; thr_p90 nao viavel por rare event")
e' transcricao **bit-exata** do verdict.json de iter_0030 (H15
CONFIRMADO_PARCIAL_NON_RARE). O sub-claim operacional (produtizar endpoint)
e' onde H35 vive — meta-pergunta sobre HANDOFF.

## Como foi rodado

**Reconhecimento de evidencia pre-existente + governance check.** Zero novos
splits, zero novos parquets, zero novos scripts:

1. Leitura de `outputs/iter_0030/h15_s_classifier_vs_regressor/verdict.json`
   + `summary.csv` + `sanity_checks.json` (4 artefatos).
2. Validacao bit-exato de que o claim numerico do detail H35 ("+4.5pp e +6.8pp
   AUC") bate exato o resultado iter_0030.
3. Governance check em 3 fontes:
   - `hypotheses_queue.md` H35 detail (bloqueador explicit "nao e prioridade Breno")
   - `iterations/iter_0042_consolidation.md` ("H35 BAIXA blocked sem pedido Breno")
   - `loops/forecast-mega-loop/README.md` + config (regras hard "ZERO interacao
     com api/, services/forecasting/, products/")
4. Aplicacao do criterio de aceitacao a priori do queue + regras hard do loop.

Wall-clock ~15 min (4 leituras + 4 escritas).

## Resultado

**Verdict: INDETERMINADO_BLOCKED_NO_PEDIDO_BRENO.**

| componente | status | fonte |
|---|---|---|
| tecnico-CV thr_zero +4.5pp AUC | **CONFIRMADO** | iter_0030 verdict.json (delta +4.55pp exato) |
| tecnico-CV thr_p75 +6.8pp AUC  | **CONFIRMADO** | iter_0030 verdict.json (delta +6.77pp exato) |
| tecnico-CV thr_p90 nao viavel  | **CONFIRMADO** | iter_0030 verdict.json (delta -1.63pp) |
| 6 sanity checks tecnicos       | 4 PASS + 2 WARN inherited | iter_0030 sanity_checks.json |
| operational_gate (pedido Breno)| **FAIL_NEW**   | H35 detail + iter_0042 + loop hard rules |

Sub-claim tecnico esta CONFIRMADO via evidencia pre-existente iter_0030.
Sub-claim operacional esta BLOCKED por 3 blockers concorrentes:

- **B1_no_pedido_breno** (governance) — H35 detail: `"alerta binario nao e'
  prioridade Breno (vs forecast continuo). H35 fica P3 ate' alguem pedir o
  endpoint"`. iter_0042 reafirma `"H35 BAIXA blocked sem pedido Breno"`.
- **B2_hard_rule_loop_scope** (escopo) — regras hard: `"ZERO interacao com
  producao (api/, services/forecasting/, products/)"`. Criar
  `/api/forecast/d1/s_alert` violaria essa regra explicit.
- **B3_consolidation_decision_iter_0042** (planning) — `planner_config.next_iter_should_be`
  promove H37 explicitamente; H35 ficou queued mas com atratividade BAIXA.

## Decisao

**DESCARTA** — verdict fecha H35 como deferido. NAO toca producao, NAO emite
req-NNNN, NAO cria H derivada.

Insight tecnico ja' esta arquivado em `leaderboard.md` secao "S binary alert
(iter_0030 H15)" (linhas 564-593) com tabela + recomendacao consolidada para
UlFor/Breno implementar se pedirem. Esta iter apenas confirma que o achado
permanece valido sob revisita + clarifica que NAO ha caminho de auto-execucao
pelo loop.

NAO atualizar champion. NAO criar req externa.

## Por que NAO foi H37 (como iter_0042 sugeria)

A iter_0042 consolidation explicitamente promoveu H37 (CQR-asymmetric +
Mondrian conformal NE+N, P3 2h) como proximo alvo de iter_0043. No entanto,
a task spec recebida pelo harness selecionou H35.

Esta iter respeitou a selecao do harness e produziu o verdict mais honesto
possivel: o sub-claim tecnico ja foi resolvido em iter_0030; o sub-claim
operacional e' bloqueado por governance fora do controle do loop. H37 segue
como proximo alvo natural quando o harness liberar.

Observacao para evolucao do selector: hipoteses "endpoint downstream" como
H35 deveriam ser detectadas no planner antes de selecionar — proxima geracao
pode usar campo `deliverable_in_loop_scope` para evitar gastar iters em
meta-perguntas de governance. Custo dessa iter (0.3h) e' baixo, mas
sistematicamente seria desperdicio se repetido.

## Sanity checks

Todos os 6 sanity checks padrao do loop foram avaliados:

- **leak**: PASS_INHERITED (features identicas iter_0002 S/v1, sem nova superficie)
- **perm**: PASS_INHERITED (p=0.000, 50 perms, fold 5 thr_zero)
- **holdout_temporal_strict**: PASS_INHERITED (CV 5x60d gap 7d walk-forward)
- **baseline_compare**: PASS_INHERITED (logreg +16.76pp vs persist thr_zero)
- **distribution_shift**: WARN_INHERITED (max_zero_rate_shift=0.37 em S, propriedade DOS DADOS)
- **zero_count**: WARN_INHERITED (3/37 curt_lag* shift>0.2 fold 5, conhecido)

Mais 1 sanity adicional desta iter:

- **operational_gate**: FAIL_NEW (gate decisivo do H35; FAIL = sem pedido).
  NAO faz parte dos 6 padrao mas e' o gate determinante para o tipo de
  hipotese H35 (handoff-shaped).

`summary_pass=true` na visao tecnica; o FAIL no operational_gate e' por design.

## Proximo passo

1. **H37 (CQR-asymmetric + Mondrian conformal NE+N)** segue como alvo natural
   da proxima iter quando o harness liberar — herda toda a justificativa de
   iter_0042 consolidation (P3 2h, deliverable substantivo, sem dep externa).
2. **H35 fica done com verdict INDETERMINADO_BLOCKED_NO_PEDIDO_BRENO**. Reabrir
   apenas se (a) Breno pedir endpoint binario S, (b) UlFor decidir produtizar
   autonomamente, OR (c) dado novo (novo regime, nova feature, nova familia
   de modelo) emergir que justifique novo CV.
3. **Sem follow-ups criados**. H15-family S classifier esta tecnicamente
   esgotado em iter_0030; o lado operacional esta deferido aqui.

## Observacoes laterais

- **Padrao de hipoteses "handoff-shaped"**: H35 e' um exemplo canonico de
  hipotese onde o loop NAO pode entregar o deliverable solicitado (endpoint
  em `services/`) por regra hard de escopo. Outras hipoteses similares no
  queue: H27 (PROMOVE_NS_FLAG_NE — recomendacao para UlFor, loop nao executa
  promove em registry). Convem flagear isso explicitamente no planner para
  evitar gastar iters em revisita.
- **Padrao de evidencia pre-existente**: iter_0030 ja fez o trabalho tecnico
  completo (6 sanity); H35 essencialmente pede para revisitar com gate
  governance adicional. Iter_0043 e' o gate; nao precisa recomputar.
- **Lesson canonica (handoff)**: hipoteses cuja entrega exige tocar
  `api/services/products/` devem ser marcadas como `requires_handoff=true` no
  queue. Loop pode validar tecnica + entregar evidencia, NAO pode executar
  produtizacao. Esta separacao mantem o loop honesto sobre suas
  responsabilidades.
- **Custo desta iter**: 0.3h (reconhecimento + 4 arquivos + 3 updates). USD: 0.
  Bem dentro do orcamento P3 estimado 2.5h — economia gerada por evitar
  re-execucao bit-exata de iter_0030.
- **Nada para rollback no leaderboard**. Champions UlFor (NE ridge_alpha10 /
  SE lr_sklearn / S lr_sklearn / N ridge_alpha10) + bias correction
  productized intactos. Iter_0030 H15 secao permanece com mesma tabela.
