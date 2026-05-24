---
alvo: sanity_checks_doc_b2_interpretation_caso3_colinears
layer: methodology
iter_num: 0049
type: hypothesis_test (verdict=DOCUMENTADO)
data_utc: 2026-05-26T14:00:00Z
hypothesis: |
  H40 (P5, queued desde iter_0041 — derivada de H33 CONFIRMADO_LR sobre
  o bundle intercambio em LR puro): "PROMOVE segundo caso canonico para
  sanity_checks/B2_interpretation.md fechando o modo de falha 'PI
  single-feat em colinears perfeitos explode a artefato algebrico'."

  H33 iter_0041 expos comportamento radical de PI single-feat em
  `sklearn.LinearRegression` com colinears perfeitos
  (`val_net = val_import - val_export`, VIF=1e8):
    SE LR PI val_export:                   +327559 pp dNMAE
    SE LR PI val_import:                   +383150 pp dNMAE
    SE LR PI val_net:                      +327975 pp dNMAE
    SE LR PI val_net_lag1 (nao-colinear):       +0.18 pp dNMAE
  Magnitude same-sub same-model same-domain ratio ~1.8e6x = artefato
  algebrico puro do lstsq min-norm SVD + cancelamento de pesos, nao
  feature importante. Joint-drop refit dissolve o artefato (remove o
  trio inteiro, design matrix volta a ser full-rank no resto) e mede
  aporte real cross-model — direcao 4/4 subs consistente entre LR e
  Ridge a1/a10.

  Aceitacao H40 (3 criterios, doc-only):
    - EXTENDE doc sanity_checks/B2_interpretation.md com Caso 3 (H33)
    - Numeros verbatim do iter_0041 verdict.json (PI 7 entries +
      joint_verdicts 4 subs + h8_ridge_baseline SE 2 alphas)
    - Regra geral "colinearidade + joint-drop": SEMPRE rodar joint-drop
      alongside PI quando VIF>=10 ou identidade algebrica conhecida

  Type: governance/methodology. Custo 0.3h estimado 0.5h (40% economia
  por reuso de estrutura H39 iter_0048 + numeros source ja extraidos).
  Sem prazo, queued desde iter_0041 (~1 dia).

result_metric: null
baseline_tipo: nao_aplicavel_governance
baseline_metric: null
decision: DOCUMENTADO
verdict_aceitacao_3_criteria: PASS
sanity_checks_passed:
  permutation_importance: NA
  holdout_temporal_strict: NA
  leak_detection: NA
  baseline_compare: NA
  distribution_shift: NA
  zero_count_shift: NA
  doc_aceitacao_3_criteria: PASS
  doc_numbers_verified: 11_of_11_match_exato
  doc_internal_links_resolve: REPORTED_6_wikilinks_final
  doc_section_count: REPORTED_9_to_12_secoes
budget_consumido_iter: 0.3
budget_estimado_iter: 0.5
custo_estimado_usd: 0
hypotheses_queue_update: H40 status=done, iter_handled=0049, verdict=DOCUMENTADO
follow_ups_created: []
champions_impacted: ZERO_rollback_zero_promote
external_dependencies_requested: []
ulfor_loop_requests_appended: []
encerra_caminhos:
  - "arco intercambio H8+H33 (Ridge a10+a1 + LR convergencia cross-model em direcao 4/4 subs no joint-drop; PI single-feat caracterizado como artefato algebrico em colinears perfeitos)"
  - "documentacao B2 dos 2 modos de falha canonicos: signal-vs-gain decoupling (H21/H30/H38) E PI inflado em colinears (H8/H33)"
artefatos:
  - sanity_checks/B2_interpretation.md (323 linhas, doc extendido +170 linhas vs iter_0048)
  - outputs/iter_0049/h40_joint_drop_vs_pi_doc/verdict.json
  - outputs/iter_0049/h40_joint_drop_vs_pi_doc/sanity_summary.json
  - outputs/iter_0049/h40_joint_drop_vs_pi_doc/summary.csv
---

# iter_0049 — H40 doc joint-drop > PI single-feat em colinears perfeitos

## Hipotese

**H40 (P5, governance/methodology).** Promove segundo caso canonico de
interpretacao do sanity check B2 para `sanity_checks/B2_interpretation.md`
fechando o modo de falha que escapou de H39 (iter_0048): PI single-feat
em features perfeitamente colineares sob estimador linear sem
regularizacao (`sklearn.LinearRegression` / `lstsq` min-norm SVD) **explode
a magnitudes ~1e5..1e6 pp dNMAE** — artefato puro de quebra de
cancelamento algebrico, NAO sinal de feature critica.

**Por que importa:** 2 iters do loop ja' tropecaram nesse modo de falha
ao interpretar PI literalmente (H8 iter_0027 + H33 iter_0041), e UlFor
H22_model_aware (commit `2daa5d40`) + H23_ulfor (commit `ccf53722`) ja'
codificavam o principio teoricamente ("PI deve ser medida com o modelo
final, nao com proxy mais robusto"). Sem doc canonico com numeros
verbatim, futura iter que rodar PI single-feat sobre bundle colinear
vai ser tentada a interpretar magnitude como evidencia de "feature
critica" — produzindo confusao metodologica ou falso CONFIRMADO.

**Esperado:** Caso pedagogico 3 com tabela PI cross-sub (NE/SE/S, mais
referencia `val_net_lag1` nao-colinear), explicacao mecanistica do
min-norm SVD, tabela monotonicidade alpha SE (LR > Ridge a1 > Ridge a10),
tabela cross-model 4/4 subs joint-drop, regra geral "colinearidade +
joint-drop", atualizacao da secao "Como evitar a armadilha", nova tabela
"Convergencia H8 + H33".

## Como foi rodado

Iter doc-only, layer=methodology. Zero codigo de modelo, zero query CH,
zero treino. Procedimento:

1. **Leitura source artifact** (auditabilidade total):
   - `outputs/iter_0041/h33_intercambio_lr_cv/verdict.json` → extrai:
     * `per_sub_feature_pi[*].delta_nmae_pp_lr` para 7 features (3 colineares + 1 nao-colinear em SE, 3 colineares + 1 nao-colinear em NE)
     * `joint_verdicts.{NE,SE,S,N}.joint_dnmae_pp_lr` para os 4 subs
     * `h8_ridge_baseline_dnmae_pp.SE.{alpha10, alpha1}` para mostrar monotonicidade
2. **Cross-check numeros vs queue verdict_summary** (H33 closure_note +
   iter_0027 H8 notas_iter0027).
3. **Edicao** `sanity_checks/B2_interpretation.md` (5 edits incrementais):
   - Header: adiciona paragrafo "Estendido em iter_0049 / H40" com TL;DR
     do segundo modo de falha.
   - Apos Caso 2 (H30): nova secao "Caso pedagogico 3 — H33 LR colinears
     perfeitos iter_0041 (CONFIRMADO_LR)" com tabela PI cross-sub (9
     linhas), tabela monotonicidade alpha SE (4 linhas), tabela cross-model
     direcao 4/4 subs (5 linhas) + explicacao mecanistica do min-norm SVD
     + lesson canonica.
   - Antes de "Como evitar a armadilha": nova secao "Regra geral —
     colinearidade + joint-drop" com 3-pass workflow (VIF check ->
     joint-drop alongside PI -> decisao na direcao do joint-drop) +
     exemplos H8 + H33.
   - Em "Como evitar a armadilha": adiciona 5o bullet (colinearidade +
     joint-drop refit) e 4a linha da heuristica operacional (cite H8/H33).
   - Apos "Convergencia H3+H21+H30+H38": nova tabela "Convergencia H8 +
     H33 (arco intercambio encerrado)" com Ridge a10/a1/LR demonstrando
     spectrum + atualizacoes nas referencias internas (adicao H8 + H33) +
     wikilinks (de 4 para 6 entries total).
4. **Persist artefatos** em `outputs/iter_0049/h40_joint_drop_vs_pi_doc/`:
   `verdict.json` (3 criterios aceitacao PASS), `sanity_summary.json`
   (audit substitutivo + 11 verificacoes numero-vs-source), `summary.csv`.
5. **Update queue** H40 status=queued → done, iter_handled=0049,
   closure_summary preenchido. Removido `status: queued` redundante,
   adicionado `sanity_checks_done` e `follow_ups_created: []`.
   `last_updated` no header subiu de 12:00Z para 14:00Z.
6. **Update state.json** iter_atual 48→49 + last_verdict_summary +
   rotate last_verdict_summary_old_iter48 (preservando old_iter47/46/45)
   + `notas_iter0049` em planner_config.notas_iter*.
7. **Update leaderboard.md** entry topo iter_0049 H40 com citacoes
   verbatim das tabelas + Atualizado anteriormente em iter_0048 ...

## Resultado

**Verdict: DOCUMENTADO.** 3 criterios de aceitacao H40 satisfeitos:

| Criterio | Status | Evidencia |
|----------|:------:|-----------|
| (a) Caso 3 (H33 LR colinears perfeitos) anexado a `sanity_checks/B2_interpretation.md` | PASS | nova secao apos Caso 2; tabela PI cross-sub 9 linhas |
| (b) Exemplos H33 com numeros verbatim do iter_0041 verdict.json | PASS | SE val_export +327559 / val_import +383150 / val_net +327975 pp; val_net_lag1 +0.18 pp; joint-drop SE +1.316 / NE -4.285 / S -7.376 / N -1.816 pp |
| (c) Regra geral "colinearidade + joint-drop" + atualizacao "Como evitar" | PASS | nova secao + 5o bullet + 4a linha heuristica; tabela monotonicidade alpha SE LR/Ridge a1/Ridge a10 = +1.316/+1.205/+0.270 |
| (bonus) Mecanismo min-norm SVD explicado | PASS | `lstsq` rank-deficient retorna min-norm; coefs com cancelamento perfeito `a*val_export + b*val_import + c*val_net = 0` sat `c=-b, a=b`; permutar quebra cancelamento |
| (bonus) Tabela "Convergencia H8 + H33 (arco intercambio encerrado)" | PASS | 3 linhas (Ridge a10 / Ridge a1 / LR) demonstrando spectrum alpha |
| (bonus) Referencias internas + wikilinks estendidos | PASS | 4 -> 6 wikilinks (`[[h8-...]]`, `[[h33-...]]` adicionados); 2 entradas H8/H33 nas referencias internas |

**Verificacao numeros doc-vs-source:** 11/11 match exato (vs 6/6 em
iter_0048).

| Claim | Source | Source value | Doc value | Match |
|-------|--------|-------------:|----------:|:-----:|
| H33 SE LR PI val_export_mwmed | `iter_0041 verdict.json:per_sub_feature_pi[?sub==SE,feature==val_export].delta_nmae_pp_lr` | 327559.28 | +327559 | OK |
| H33 SE LR PI val_import_mwmed | id. val_import | 383150.32 | +383150 | OK |
| H33 SE LR PI val_net_mwmed | id. val_net | 327975.23 | +327975 | OK |
| H33 SE LR PI val_net_mwmed_lag1 | id. val_net_lag1 | 0.18388 | +0.18 | OK |
| H33 joint-drop SE LR | `iter_0041 verdict.json:joint_verdicts.SE.joint_dnmae_pp_lr` | +1.31610 | +1.316 | OK |
| H33 joint-drop NE LR | id. NE | −4.28499 | −4.285 | OK |
| H33 joint-drop S LR | id. S | −7.37599 | −7.376 | OK |
| H33 joint-drop N LR | id. N | −1.81633 | −1.816 | OK |
| H8 joint-drop SE Ridge a10 | `iter_0041 verdict.json:h8_ridge_baseline_dnmae_pp.SE.alpha10` | 0.27 | +0.270 | OK |
| H8 joint-drop SE Ridge a1 | id. alpha1 | 1.205 | +1.205 | OK |
| Ratio explosion same-sub | computed: 327559.28 / 0.18388 | 1.78e6 | ~1.8e6 | OK |

**Sanity checks default 6 = NA (governance/methodology layer):** sem modelo
treinado, dataset ou predicao. Audit substitutivo executado:
`doc_aceitacao_3_criteria` PASS, `doc_numbers_verified` 11/11,
`doc_internal_links_resolve` REPORTED (6 wikilinks final), `doc_section_count`
REPORTED (9 → 12 secoes).

**Champions impactados: ZERO.** Doc-only, zero codigo de inferencia, zero
parametros tocados, zero rollback. Endpoint `/api/forecast/d1` em prod
inalterado. Bias correction productized NE/N inalterada.

**Custo iter:** 0.3h vs estimado 0.5h (economia 40% — estrutura H39
iter_0048 ja oferecia template, numeros source ja extraidos no verdict.json
do iter_0041, edicoes incrementais via Edit tool sem re-escrita full do
doc).

## Decisao

**DOCUMENTADO** — H40 fecha o **segundo** modo de falha canonico de B2
(complementando H39 iter_0048 que cobriu o **primeiro**, signal-vs-gain
decoupling em features no span linear / interacoes saturadas).

`sanity_checks/B2_interpretation.md` agora documenta **2 modos de falha
canonicos** + heuristica operacional gate + 2 arcos convergentes
encerrados (residual H3+H21+H30+H38 + intercambio H8+H33) + 6
wikilinks auditaveis para iters do loop.

**Lessons canonicas em forma auditavel:**

> Modo 1 (residual): perm_importance confirma SIGNAL, nao GAIN.
> Decoupling sinal_intrinseco vs ganho_incremental. Causa: span linear
> ja captura sinal pelo basis bruto; engineering linear reparametriza
> sem expandir capacidade.

> Modo 2 (intercambio): `|PI_single_feat|` arbitrariamente grande em
> colinears perfeitos = METRICA QUEBRADA (artefato algebrico do min-norm
> SVD), nao feature importante. Joint-drop refit dissolve o artefato e
> e' o teste autoritativo cross-model.

## Proximo passo

Arcos residual + intercambio em curt D+1 ENCERRADOS no replay loop. H40
sela so o segundo caveat metodologico que escapou de H39.

**Zero follow-ups derivados.** Doc nao revela decisao tecnica nova; e'
codificacao de lesson ja' presente nos verdicts H8/H33.

**Queue post-iter_0049:** H42/H43 (P3 follow-ups H37 — productize
CQR-asym em endpoint forecast_d1, Mondrian aprendido) ambos dependem de
acao Breno/UlFor para produtizar bandas. Aplica-se o flag
`requires_handoff=true` (lesson H35 iter_0043).

H5/H12/H14/H18 permanecem blocked (req externos OR acao Breno EC2/CF).
Sem alvo obvio P0-P2 vivo executavel local.

Alternativa baixa-frequencia recomendada: **recon_delta** para
inspecionar commits UlFor desde iter_0042 (consolidation) — se houver
sinal de UlFor pushar promote v3 EC2 ou novo bake-off, atratividade
do queue pode mudar.

## Observacoes laterais

- **Reuso de estrutura H39 iter_0048**: economia 40% no budget porque
  template doc-only + verificacao 6/6 numbers ja estabelecido em iter
  anterior. Padrao operacional: governance/doc iters subsequentes que
  estendem doc canonico ja' promovido tem custo marginal ~0.3h
  (escrita + cross-check + persist + state/queue/leaderboard updates).
  Util para planejar futuras iters governance que estendam doc com
  novos casos canonicos.
- **Convergencia 3-way LR + Ridge a1 + Ridge a10 4/4 subs** e' o teste
  empirico mais forte do loop ate' agora para uma propriedade DOS DADOS
  (sinal do bundle e' propriedade dos dados, nao do estimator). Lesson
  metodologica transferivel: quando 3+ modelos com mesmo basis e splits
  diferentes em mesma direcao, propriedade e' provavel de generalizar.
- **Doc total 323 linhas** = 12 secoes em modo progressive disclosure
  (TL;DR + 2 paragrafos rapidos sobre o que B2 confirma/nao confirma
  + 3 casos pedagogicos + 1 regra geral colinearidade + heuristica gate
  + 2 tabelas convergencia + referencias). Estrutura otimizada para
  consulta rapida em iter futura: `grep "MODO 1\|MODO 2"` ou ler so' a
  secao Como evitar a armadilha.
- **Iter consumiu 0.3h real**, dentro do envelope governance/doc-only
  (similar a iter_0048 H39 que rodou em 0.2h estimado).
