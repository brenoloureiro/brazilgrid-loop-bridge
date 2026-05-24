---
alvo: recon_delta_ulfor_post_83abab3e
layer: meta
iter_num: 0028
type: recon_delta
data_utc: 2026-05-24T23:30:00Z
ulfor_head_inicio: 83abab3e
ulfor_head_fim: 27152e16
commits_absorvidos: 5
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.3
---

# Iter 0028 — RECON_DELTA UlFor (83abab3e → 27152e16)

## Objetivo

Absorver 5 commits novos da sessao UlFor entre `83abab3e` (HEAD ao fim do
iter_0026 — fix de coerencia MATRIX promote_champions.py) e `27152e16`
(checkpoint 16:02Z apos parallel agent fechar pendencias 15:46Z). Janela
~76 min reais UlFor (12:25-13:42 BRT). Conteudo: **val14d real holdout
alpha sweep** (test 2026-03-24..2026-05-21, n≈58d) que fecha o validation
gap parcial reaberto em iter_0026 para `SE ridge+h22_per_fold+α=1`.

## PHASE A — Commits inspecionados

Em ordem cronologica:

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `d9aefdd1` | checkpoint forcado 15:46Z | runner detectou 3 iters sem commit; PARAR-E-PERGUNTAR + parquets val14d nao-committados = falsa estagnacao | state.json + leaderboard |
| 2 | `f7c56c3d` | exp val14d alpha sweep parquets | **ARTEFATO source-of-truth val14d** 3 candidatos × 4 subs | leaderboard nova secao + queue notas |
| 3 | `1bd8638f` | docs FINDING ADDENDUM val14d | persiste tabela ridge val14d + convergencias/divergencias | leaderboard tabela val14d |
| 4 | `ff112a27` | docs FINDING diff features SE | **MECANISMO** 10 features divergentes h22_MA vs h22_pf | leaderboard mecanismo |
| 5 | `27152e16` | checkpoint 16:02Z marker | tree limpo, parallel agent fechou pendencias 15:46Z | state.json |

### Interpretacao por commit

**`d9aefdd1` (checkpoint 15:46Z FORCADO)**. Runner headless detectou
`iters_sem_commit=3 >= 3` apesar de PARAR-E-PERGUNTAR legitimo ativo
desde 14:35Z (Breno decide promote v3). Causa raiz: val14d alpha sweep
gerou 3 parquets nao-committados — runner viu tree limpo como
"estagnacao real". Mensagem documenta resultados: NE α=1 empate h22_MA
= h22_pf (30.88%); **SE α=1 INVERTE veredito CV** — h22_MA (43.03%)
> h22_pf (44.67%) em val14d; N α=100 + h22_pf 71.51% / +0.158 confirma;
S todas configs ridge perdem persist (~95% > 88%). Promote v3 ganha
opt_A vs opt_B em SE.

**Lesson learned UlFor** (operacional): commit-only-of-artifacts mesmo
em PARAR-E-PERGUNTAR para evitar tree-sujo + sessao-parada simultaneos.

**`f7c56c3d` (exp val14d parquets)**. **Source-of-truth**. 3 parquets
`cv_summary_per_fold_*_val14d_alpha*.parquet` (~7.8 KB cada). Holdout
estrito: train 2024-12-01 → 2026-03-23 (cresce), test 2026-03-24 →
2026-05-21 (n≈58d eff). Sem CV — single fold real recente. Tabela
ridge consolidada (replicada no leaderboard).

**`1bd8638f` (FINDING ADDENDUM val14d)**. +69 linhas em
`FINDING_RIDGE_ALPHA_SWEEP.md`. Persiste decisao em aberto Breno:

- opt_A = ridge + h22_MA + α=1 (val14d melhor SE 43.03%)
- opt_B = ridge + h22_pf + α=1 (CV melhor + coerencia multi-sub)

NE: empate (promover h22_pf por coerencia). N: α=100 + h22_pf vence em
ambos (val14d MELHOR que CV — concept drift positivo). S: rejeitado em
ambos, manter `lr+full`.

**`ff112a27` (FINDING diff features SE)**. **EXPLICA MECANISMO** da
divergencia SE. h22_MA dropa 11 features em SE; h22_pf dropa 21 features;
h22_pf = h22_MA ∪ 10 extras. As 10 features divergentes que h22_MA
PRESERVA mas h22_pf DROPA:

| categoria | features | count |
|---|---|---:|
| CMO (preco) | `cmo_mwmed`, `cmo_mwmed_rmean30`, `cmo_mwmed_rmean7` | 3 |
| Intercambio | `val_export_mwmed`, `val_import_mwmed` | 2 |
| Regime/penetr. | `taxa_penetracao`, `ger_eolica_mwh` | 2 |
| Carga | `carga_pico_mw` | 1 |
| Previsao solar | `prev_solar_pico_mw` | 1 |
| Termica | `ter_verif_rmean7` | 1 |

Razao tecnica: `h22_pf` usa **Ridge universal PI** (L2 mascara importance
via shrinkage); `h22_MA` usa **champion-model real** (LR p/ SE preserva
load-bearing). val14d real (Mar-Mai/2026) regime: CMO subindo
(curt economico volta a importar), intercambio SE-S inverte
(`cmo_mwmed` e `val_export_mwmed` correlacionam), `taxa_penetracao` em
alta (parque eolico/solar NE em expansao). Essas 10 features que h22_MA
preserva carregam sinal em regime recente que h22_pf perde — explica
−1.64pp NMAE SE em val14d.

**`27152e16` (checkpoint 16:02Z marker)**. Tree limpo. Parallel agent
fechou pendencias entre 15:46Z e 16:02Z (commits 3+4 acima). PARAR-E-
PERGUNTAR continua ativo aguardando decisao Breno SE opt_A vs opt_B.
Branch 26+ ahead origin.

## PHASE B — Atualizacoes

### state.json

- `iter_atual: 27 → 28`
- `alvo_ativo: feat_intercambio_importance_cv_pi → recon_delta_ulfor_post_83abab3e`
- `layer_ativa: curtailment → meta`
- Bloco novo `ulfor_session_sync.iter0028_*` com 5 commits + delta_resumo
- `planner_config.next_iter_should_be` atualizado para iter_0029 candidatos
  (A) monitor recon, (B) H30, (C) H22 nosso, (D) H33 (atratividade DIMINUIU)
- `notas_iter0028` adicionado
- `budget_consumido_today.iter: 37 → 38`, `horas: 7.635 → 7.95`
- `ultimo_handoff: iter_0028_recon_delta.md`

### hypotheses_queue.md

- `last_updated: 2026-05-24T23:30:00Z`
- Bloco `notes_iter0028` em H33 (atratividade DIMINUI marginalmente —
  val14d ja deu evidencia model-aware indireta SE via h22_MA vs h22_pf)
- Bloco `notas_iter0028` no rodape com inspected_range, resolved (vazio),
  attractiveness_changes, validation_gap_status_fechada,
  convergencia_h8_val14d, promote_v3_decisao_breno_aberta

**Nenhuma H resolvida nesta iter. Nenhuma H nova criada.** H8 iter_0027
verdict permanece — val14d apenas REFORCA empiricamente.

### leaderboard.md

- Header atualizado: iter_0028, source inclui 3 parquets val14d
- Nova secao **"Val14d alpha sweep v3 — holdout real recente"** com:
  - Tabela 4 subs × 3 candidatos (NMAE / R²)
  - Bloco mecanismo da divergencia SE (10 features)
  - Bloco convergencia com H8 iter_0027
  - Tabela promote v3 (decisao Breno aberta opt_A vs opt_B)
- Bloco "Validation gap PARCIALMENTE REABRE" atualizado com FECHADO
- Linha iter_0028 em "Historico iter loop"

### state.open_requests

`[]` — inalterado. Loop NAO emite req-0008 (UlFor self-actionou val14d
em <90min como pre-empcao multi-agente esperada). `closed_requests`
inalterado.

## PHASE C — Resumo executivo

### Highlights operacionais

1. **Validation gap iter_0026 FECHADO** via UlFor self-action <90min.
   Padrao pre-empcao multi-agente confirmado 4a iter consecutiva
   (iter_0023 H31 emergente, iter_0024 H31 pre-empted, iter_0026 H31
   mantem pre-empted, iter_0028 fechamento explicito).

2. **Promote v3 consolidado em 3/4 subs**, divergente em SE:
   - NE: PROMOVER `ridge + h22_per_fold + α=1` (CV + val14d coincidem)
   - N: PROMOVER `ridge + h22_per_fold + α=100` (CV + val14d coincidem)
   - S: MANTER `lr + full` (val14d confirma rejeicao ridge)
   - SE: DECISAO BRENO entre opt_A (val14d −1.64pp) vs opt_B
     (CV + coerencia + menor overfit). Recomendacao tecnica UlFor: opt_B.
     Diff <2pp NMAE = margem amostral.

3. **Mecanismo da divergencia SE documentado**: 10 features divergentes
   (CMO + intercambio + regime + carga + prev_solar + ter_verif_rmean7).
   Ridge universal PI mascara via L2; LR sem shrinkage usa esses canais.
   val14d regime Mar-Mai/2026 favorece preservacao (curt economico em alta).

4. **Convergencia empirica com H8 iter_0027 (REFUTADO_SE_em_Ridge)**:
   `val_export_mwmed` + `val_import_mwmed` (2 das 10 features divergentes)
   sao o bundle intercambio que H8 testou em joint-drop. iter_0027 H8
   prova joint-drop intercambio em SE Ridge α=1 = +1.21pp HARMFUL.
   iter_0028 val14d prova preservar intercambio em SE Ridge α=1 BATE
   dropar por −1.64pp. Mesma direcao, magnitude consistente, contextos
   independentes. **Lesson PI-com-modelo-final reforcado empiricamente
   por 2 protocolos diferentes**.

5. **Branch UlFor 26+ ahead origin**. Acao Breno pendente: (1) decidir
   SE opt_A vs opt_B; (2) executar `promote_champions.py` ja patcheado
   iter_0026 commit `75e2431e`; (3) push deploy EC2. Loop NAO toca prod.

### Hipotese ficou na fila para iter_0029

- **H30** (P3, ~1h, atratividade SOBE marginal pela 2a vez consecutiva):
  Ridge_alpha=10 + Ridge_alpha=1 sobre pdp_residual CV. Script pronto
  (`scripts/h21_pdp_residual_cv.py` deixado em iter_0019). val14d
  iter_0028 reforca que α=1 vence α=10 default em NE+SE em ridge —
  H30 e o teste mais sensivel desse mecanismo em basis residual.
  **Zero dep externa**.

- **H22 nosso** (P3, ~1h, atratividade INALTERADA): GBDT vs OLS gap em
  pdp, com PI-com-GBDT lesson H23/H22_MA aplicada. UlFor val14d e' sweep
  linear puro — nao toca a divergencia GBDT vs OLS.

- **H33** (P3, ~0.3h, atratividade DIMINUI marginal): joint-drop SE em
  LR. val14d ja deu evidencia model-aware indireta via h22_MA (LR-based
  PI) vs h22_pf (Ridge-based PI). Teste especifico do bundle intercambio
  ainda nao foi feito mas pico de atratividade pre-empted.

- **Monitor recon** (A): se UlFor HEAD avancar alem `27152e16` ate
  inicio do iter_0029 — provavel se Breno decide promote v3 entre
  agora e proxima sessao.

**RECOMENDACAO**: (A) se HEAD UlFor advance; senao (B) H30 — fecha
H3-family residual e valida alpha sweep simultaneamente.

### Sanity checks

N/A para RECON_DELTA (nao e hypothesis_test). Tipos default 6-checks
nao aplicam.

### Budget

~0.3h (recon read-only, sem execucao Python). Acumulado dia 2026-05-24:
7.95h / iter 38.

### Discrepancia ainda nao investigada

`CHAMPION_DECISION_MATRIX.md` iter_0023 lista NE+N como "lr+full" no
"Champion ATUAL em prod" — contradiz state.json/Registry com
ridge_NE+ridge_N. iter_0028 NAO toca matrix nem `loader.py`. Hipotese
(B) bug documentacao matriz permanece mais plausivel. Loop NAO flagga
req formal (cosmetico). Leaderboard mantem state.json por seguranca.
