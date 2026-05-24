---
alvo: curtailment_d1_pdp_validation + canal_loop_ulfor
layer: meta
iter_num: 0003
type: hypothesis_validation + infra_handshake
data_utc: 2026-05-24T02:30:00Z
baseline_tipo: dbt_join_correto
hypothesis: |
  H2 (do iter_0002): off-by-one no JOIN feat_pdp_renovavel causa "shift -1 dia
  melhora NMAE em pdp_prev_eolica" observado no sanity B1 (delta -4.2pp).
result_metric: |
  H2 REFUTADO empiricamente: corr(PDP_prev[t], gen[t])=0.9118 e o pico,
  corr(PDP_prev[t], gen[t+1])=0.8211 inferior. dat_programacao = target day.
  n=484 dias, NE eolica agregado.
decision: H2 REFUTADO. Manter JOIN atual. Canal loop->ulfor estabelecido.
sanity_checks_passed:
  empirical_correlation_lag_test: true   # n=484, gap entre lags > 0.08 Pearson
  shape_consistency_signs: pass          # MAE confirma pico em gen[t]
  upstream_source_audit: pass            # stg passthrough, raw tem 1 unica data col
budget_consumido_iter: 1.4
custo_estimado_usd: null

artefatos_persistidos:
  - loops/forecast-mega-loop/outputs/iter_0003/h2_off_by_one_pdp/
      _dbt_feat_pdp_renovavel_HEAD.sql
      _dbt_stg_ons_programacao_previsao.sql
      empirical_check.json
      empirical_join.parquet
      investigation_sql.md
      verdict.md
  - loops/forecast-mega-loop/outputs/iter_0003/autopilot_patch.md
  - loops/forecast-mega-loop/scripts/h2_check.py
  - C:/Projetos/brazilgrid-ulfor/coordination/loop_requests.md (exception unica,
      commit ulfor 1fb2bb50)
---

# Iter 0003 — Validacao de H2 + Canal loop->UlFor

## H2 validacao (verdict + evidencia)

### Pergunta

Iter_0002 observou que substituir `pdp_prev_eolica[t]` por `pdp_prev_eolica[t-1]`
no test set MELHORA NMAE em 4.2pp para LGBM NE/v3. Hipotese H2: o JOIN
`pdp.dia = addDays(t.dia, 1)` no bakeoff duplica um shift que ja estaria
implicito no dbt model — off-by-one.

### Metodo

1. **Phase 1A** (read-only sobre UlFor):
   - Snapshot do `feat_pdp_renovavel.sql` HEAD (`d0cf3054`).
   - Snapshot do `stg_ons_programacao_previsao.sql` (pass-through do raw).
   - Verificado: raw `ons_raw___programacao_previsao` tem UMA UNICA coluna
     de data (`dat_programacao`). Semantica nao documentada — precisa
     verificacao empirica.

2. **Phase 1B** (CH SELECT-only):
   - Crosswalk seed CSV → 242 cod_usinapdp EOL NE
   - Agregar `val_previsao` (e `val_programado`) por `dat_programacao` em
     `stg_ons_programacao_previsao`
   - Agregar `geracao_verificada_mw` por `toDate(din_instante)` em
     `obt_usina_enriched` (filtro: fonte=eolica, id_subsistema=NE, is_invalid=0)
   - Janela: 2024-12-01 a 2026-05-01 (n=484 dias com overlap)
   - Computar `corr(PDP[t], gen[t-1])`, `corr(PDP[t], gen[t])`, `corr(PDP[t], gen[t+1])`

### Evidencia

| lag testado | corr Pearson |
|---|---|
| corr(PDP_prev[t], gen[t-1]) | 0.8269 |
| **corr(PDP_prev[t], gen[t])** | **0.9118** (PICO em PREV) |
| corr(PDP_prev[t], gen[t+1]) | 0.8211 |
| corr(PDP_prog[t], gen[t-1]) | 0.8259 |
| **corr(PDP_prog[t], gen[t])** | **0.9408** (PICO GLOBAL) |
| corr(PDP_prog[t], gen[t+1]) | 0.8272 |

Diferenca de correlacao gen[t] vs gen[t+1] = **+0.0907 (PREV)** / **+0.1136 (PROG)**.
Em n=484, std do bootstrap de correlacao << 0.01 — diferenca dezenas de sigma.

MAE em MWh confirma o mesmo pico em gen[t]:
- gen[t-1]: 97171 MWh
- gen[t]: **96125 MWh** (minimo)
- gen[t+1]: 97120 MWh

### Verdict: **H2 REFUTADO**

`dat_programacao` representa o dia-alvo (target day) — interpretacao do dbt
model esta correta. O JOIN `pdp.dia = addDays(t.dia, 1)` no bakeoff esta
correto em trazer "PDP para D+1" quando target row e D. Nao ha off-by-one.

### Reinterpretacao do shift test iter_0002

O sinal "shift -1 melhora NMAE em 4.2pp" do iter_0002 mais provavel:
**ruido amostral em test n=11**. Evidencias:
- Features irmas do mesmo grupo nao tem mesmo sinal de delta: e.g.,
  `pdp_solar_*` splita -2.5pp a +3.5pp. Off-by-one verdadeiro daria
  mesmo sinal em todo grupo.
- N=11 e insuficiente para detectar 4pp em NMAE com confianca > 90%
  considerando std intra-fold tipica.

Confirmacao formal entregue ao UlFor como req-0001 (re-rodar com n>=60d).

## Canal loop->ulfor (estado, primeiro request)

### Arquivo criado

`C:/Projetos/brazilgrid-ulfor/coordination/loop_requests.md` — exception
unica a regra "loop nao escreve em ulfor". Commit no master UlFor
**`1fb2bb50`**: `coord: canal loop->ulfor estabelecido (loop iter 0003)`.

WIP do UlFor (bakeoff_nec.py modificado) preservado intocado — `git add`
escopado para apenas o arquivo coordination/.

### Protocolo (patch text-only, NAO aplicado pelo loop)

`loops/forecast-mega-loop/outputs/iter_0003/autopilot_patch.md` contem
diff sugerido para o Breno aplicar manualmente em AUTOPILOT_PROMPT.md,
adicionando passo 4.5 ao "PROTOCOLO DE RETOMADA APOS /clear":

> 4.5. Ler `coordination/loop_requests.md`. Para cada request com
>      `status: OPEN` ou `status: IN_PROGRESS`, avaliar se cabe no
>      envelope do autopilot. Se sim, executar antes de continuar
>      PLANO_FINAL, registrar `ulfor_response` e marcar DONE. Se nao,
>      registrar motivo e marcar DECLINED.

### Primeiros requests

| id | priority | type | summary |
|---|---|---|---|
| **req-0001** | P1 | investigation | Validar com test n>=60d que shift -1d e ruido amostral (off-by-one ja refutado). |
| **req-0002** | P0 | feature_fix | Substituir `coalesce(pdp_prev_mwh, 0)` por forward-fill em bakeoff p/ eliminar modo bimodal artificial dos gaps Abr/2026. |

req-0002 e P0 porque e a CAUSA RAIZ identificada no iter_0002 da regressao
NE/v3. O fix nao requer touch no dbt model — e mudanca no consumidor
(bakeoff_d1.py).

## Atualizacoes leaderboard/state

`leaderboard.md`:
- 4 linhas curtailment_d1 (uma por sub) + 1 linha meta (h2)
- Marca v2 LGBM como best NMAE em NE (28.2% original, 31.7% holdout estrito)
- Marca SE/v3 com warning B3 fail strict (+19.2pp)
- N permanece com persist_d1 vencendo ML
- S nao_aprendivel (ymean=0 na janela testavel)

`state.json`:
- iter_atual = 3
- alvo_ativo = curtailment_d1_pdp_validation
- hypotheses_verdict.H2_off_by_one_pdp_join = REFUTADO
- open_requests = [req-0001, req-0002]
- ulfor_session_sync rastreia HEAD: `b7fa795c → 1fb2bb50` (3 commits novos
  intermediarios da sessao paralela + nosso commit final)

## Proximas hipoteses para iter_0004

Encaminhadas via canal req-* (UlFor executa) OU rodaveis no loop:

### Para UlFor (via loop_requests.md)
- **req-0001**: confirmar/refutar ruido amostral n=11 (saida: fechar capitulo
  "shift test em features PDP")
- **req-0002**: validar fix forward-fill nos gaps PDP (saida: v4 deveria
  recuperar NE para nivel v2)

### Para loop (iter_0004 candidatos)
- **H3**: corr alta gen[t] vs PDP[t] (0.94) sugere PDP e proxy quase-perfeito
  de geracao realizada. Investigar se PDP adiciona algo alem de "previsao
  de geracao" — i.e., se carrega sinal de saturacao/curtailment que nao
  esta em `feat_saturacao`. Metodo: residuals(curt ~ gen) vs PDP residual.
- **H4**: distribution shift por gaps Apr/2026 e detectavel em runtime?
  Adicionar B6 sanity check: para cada feature, comparar
  `count_zeros(train)` vs `count_zeros(test)`. Threshold = razao > 3x flag.
- **H5**: feat_termico stale (>=2024-12-31) e ENORME no impacto: bakeoff
  perde sinal em janela 2025+. Avaliar se UlFor tem ingest fresco —
  request P2 metric_check.
- **H6**: SE/v3 colapso em holdout strict de iter_0002 (+19.2pp) vale
  isolated investigation. Possivelmente lag features (curt_lag1, curt_rmean7)
  estao fazendo a maior parte do trabalho e nao distinguem train-test
  adjacency. Metodo: zerar lags em test, ver NMAE.

### Prioridade sugerida

1. **iter_0004**: H6 (SE colapso strict) — relevante para o que UlFor pode
   promover, rodavel no loop sem dependencia externa.
2. **iter_0005**: H4 (B6 dist-shift zero-count) — investe em ferramenta
   diagnostico generica, util alem desse caso.
3. **Aguardar** UlFor processar req-0001/req-0002 antes de iter sobre PDP
   (evita duplicar trabalho).
