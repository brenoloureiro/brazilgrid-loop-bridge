---
alvo: recon_delta_ulfor_post_5d41d063
layer: meta
iter_num: 0026
type: recon_delta
data_utc: 2026-05-24T21:30:00Z
ulfor_head_inicio: 5d41d063
ulfor_head_fim: 83abab3e
commits_absorvidos: 8
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.4
---

# Iter 0026 — RECON_DELTA UlFor (5d41d063 → 83abab3e)

## Objetivo

Absorver 8 commits novos da sessao UlFor entre `5d41d063` (HEAD na entrada
do iter_0024 fim, correcao N ridge+h22 PROMOVER) e `83abab3e` (fix coerencia
MATRIX promote_champions.py). Janela ~25 min reais UlFor (10:23-10:43 BRT).
Conteudo denso: **2 ablacoes negativas** (H22 stricter REFUTADO; reabrir CV
LGBM SE/N REFUTADO), **1 sweep linear que destrava SE preservando familia
linear** (alpha=1 substitui alpha=10 default e resolve cond_num 2.5e17), **1
patch operacional** (promote_champions.py com --ridge-alpha + 3 H22 feature_sets),
**1 ADDENDUM** (z-score val14d), **3 checkpoints markers**.

## PHASE A — Commits inspecionados

Em ordem cronologica:

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `7cc3b604` | docs ADDENDUM val14d | refuta reabrir CV LGBM SE/N (ja existe, perde >4pp) | leaderboard + H27 nota |
| 2 | `756457c7` | chore checkpoint 14:25Z | marker convergencia parallel agent | state.json |
| 3 | `42dc0d7a` | exp H22 stricter REFUTADO + alpha sweep parcial | **3 frentes envelope-safe**; ablation negativa + sweep h22_MA + patch +37 linhas | leaderboard secao H22 stricter + sweep |
| 4 | `a3c742a9` | exp FINDING_RIDGE_ALPHA v3 | **DESTRAVA SE com α=1+h22_pf** (-10.6pp NMAE vs lr+h22_pf, resolve R² -0.07); proposta v3 mesmo feature_set 3/4 subs | leaderboard secao alpha sweep v3 |
| 5 | `75e2431e` | exp promote_champions.py +--ridge-alpha | **OPERACIONALIZA promote v3** sem patch manual; backward-compat default α=10 | state.json patch_status |
| 6 | `db3d2cb9` | chore checkpoint 15:00Z | marker; 0 commits proprios (parallel agent absorveu) | state.json |
| 7 | `34abdd49` | chore checkpoint 14:35Z | marker; 4 sprints + promote v3 PRONTO p/ Breno | state.json |
| 8 | `83abab3e` | docs MATRIX coerencia | fix: notas_op nao mencionar --ridge-alpha (75e2431e ja adicionou) + sintaxe --only-sub | leaderboard ack |

**HEAD UlFor final desta iter** = `83abab3e`. Branch 21 ahead origin (era 14
em iter_0024).

### Resumo conceitual (3 frentes dominantes)

#### FRENTE 1 — Ablacao negativa H22 stricter (commit `42dc0d7a`)

Testado `MIN_FOLDS_DROP=5` (drop unanime nos 5 folds) em vez de `3/5` padrao.
Drop-sets ficam ~3-7x menores: NE 22→7, SE 11→3, S 4→1, N 10→2. Resultado CV
5x60d **regride em 3/4 subs vs h22 (3/5 padrao)**:

| sub | h22_str NMAE | vs h22 (3/5) | vs h22_MA |
|---|---:|---:|---:|
| NE | 33.37% | +1.96pp PIOR | +1.96pp PIOR |
| SE | 48.17% | −8.99pp MELHOR vs h22 | +0.11pp ≈ MA |
| S  | 88.08% | +3.74pp PIOR | +2.25pp PIOR |
| N  | 86.73% | +0.33pp ≈ | −0.50pp marginal |

**Lesson**: features qualificadas em 3-4/5 folds (ambiguas, nao unanimes)
carregam sinal residual util para generalizacao. Manter `MIN_FOLDS_DROP=3`.
Linha de pesquisa encerrada.

#### FRENTE 2 — Ridge alpha sweep (commits `42dc0d7a` + `a3c742a9` + `75e2431e`)

Sweep `ridge × {h22_MA, h22_per_fold} × {α=1, 10, 100, 1000}`. Resultados:

**h22_MA (NMAE_mean %)**:

| sub | α=1     | α=10 atual | α=100  | α=1000 | best     |
|-----|---------|------------|--------|--------|----------|
| NE  | **30.81** | 31.41    | 35.85  | 48.33  | α=1      |
| SE  | **46.23** | 48.40    | 52.99  | 60.63  | α=1      |
| S   | **88.54** | 95.49    | 102.69 | 114.18 | α=1      |
| N   | 91.23   | 87.23      | **84.35** | 88.99  | α=100  |

**h22_per_fold (NMAE_mean %)**:

| sub | α=1     | α=10 atual | α=100   | best     |
|-----|---------|------------|---------|----------|
| NE  | **30.81** | 31.41    | 35.85   | α=1      |
| SE  | **46.60** | 48.27    | 52.71   | α=1      |
| S   | **88.16** | 94.42    | 103.93  | α=1      |
| N   | 91.08   | 86.41      | **84.43** | α=100  |

**Conclusao 1**: α=1 vence 3/4 subs em ambos feature_sets H22; α=100
dedicado a N (justificada por mais ruido/colinearidade); α=1000 degenerado.

**Conclusao 2**: h22_MA ≡ h22_per_fold com α=1 (delta <0.5pp todos subs).

**Conclusao 3 (SURPRESA SE)**: `lr+h22_per_fold` QUEBRA em SE (R² −0.07!)
mas `ridge+h22_per_fold+α=1` entrega R² +0.43 — regularizacao minima
resolve `cond_num 2.5e17` (instabilidade exposta em iter_0021 commit
`34478f93`) **sem trocar familia para algo nao-linear**.

**Proposta v3 emergente (mesmo feature_set 3/4 subs, so alpha muda)**:

| sub | champion v3 (alpha-aware)    | ganho vs matriz iter_0023 |
|-----|------------------------------|---------------------------|
| NE  | ridge + h22_per_fold + α=1   | −0.6pp NMAE (~tie vs α=10) |
| SE  | **ridge + h22_per_fold + α=1** | **−10.6pp NMAE / R² +0.50 vs lr+h22_pf** |
| S   | lr + full (status quo)       | n/a (val refutou h22)     |
| N   | ridge + h22_per_fold + α=100 | −2.0pp NMAE / +0.02 R²    |

**Operacionalizacao** (`75e2431e`): `promote_champions.py` agora aceita
`--ridge-alpha float` (default=10.0 legado) + feature_sets
`h22_per_fold` / `h22_model_aware` / `h22_stricter`. Back-compat preservada.
Dry-run validado para NE+h22_per_fold+α=1: rows 2152, features 28 (de 55,
−22 drops + recalc prev coverage), in-sample NMAE 27.3% / R² +0.823
(consistente com CV +0.563 — overfit modesto esperado de LR-like).

#### FRENTE 3 — ADDENDUM val14d z-score (commit `7cc3b604`)

Pos-convergencia com parallel agent. Complementa FINDING_14D_REAL_VALIDATION
(iter_0024 `99af14b7`) com 2 angulos:

1. **Z-score val_recent vs envelope CV**: TODOS pontos `|z| ≤ 1σ` → sem
   regime shift detectavel; diferencas relativas em val_recent (e.g. LGBM
   ganha SE por +0.46pp) plausivelmente amostrais.
2. **Refuta "reabrir CV para LGBM SE/N"**: dados ja existem em
   `cv_summary_mean_*.parquet`. LGBM+h22_MA em SE perde CV por **−4.6pp**
   apesar de ganhar val por +0.46pp. Princípio 5 (CV win first) bloqueia
   promote LGBM.

**Implicacao loop**: H27 (P50 quantile LGBM substituto) atratividade
INALTERADA — esta janela nao destrava LGBM. Atratividade nao sobe nem desce.

## PHASE B — Atualizacoes

### state.json (commits + delta_resumo + planner)

- `iter0026_inicio_head` = `5d41d063`, `iter0026_fim_head` = `83abab3e`
- `novos_commits_durante_iter0026` populado com 8 entradas (interpretacao
  por commit)
- `delta_resumo_iter0026` com 11 campos cobrindo todos os angulos
  (champions, novos requests, validation_gap, alpha sweep v3, h22 stricter,
  achados nao-promoviveis, H_loop impactadas, regime UlFor, push origin,
  discrepancia documentacao, promote patch status)
- `iter_atual` = 26, `ultimo_handoff` = `iter_0026_recon_delta.md`
- `planner_config.notas_iter0026` resumindo o delta
- `planner_config.next_iter_should_be` propondo candidatos iter_0027

### leaderboard.md

- Cabecalho atualizado para refletir parquets alpha-sweep (7 novos arquivos)
- Nova secao **"Alpha sweep v3 (commits 42dc0d7a + a3c742a9 + 75e2431e,
  iter_0026)"** com matriz v2→v3 + vantagem operacional + nota validation gap
- Nova secao **"Ablation negativa H22 stricter (commit 42dc0d7a, iter_0026)
  — REFUTADO"**
- Nova secao **"ADDENDUM val14d z-score (commit 7cc3b604, iter_0026)"** com
  conclusao "H27 atratividade INALTERADA"
- Pendencia #3 atualizada: "REABRE PARCIAL em iter_0026 para SE
  ridge+h22_pf+α=1"
- Tabela "Historico iter loop" adiciona linha 0026

### hypotheses_queue.md

- `last_updated: 2026-05-24T21:30:00Z`
- H30 ganha `notes_iter0026` documentando atratividade marginal subindo
  pos-alpha-sweep (recomenda rodar α=1 E α=10 simultaneamente em iter_0027)
- Footer `notas_iter0026` documentando:
  - `inspected_range: 5d41d063..83abab3e`
  - `resolved: []`, `newly_blocked: []`, `newly_queued: []`
  - `pre_empted: [H31_emergente]`
  - `attractiveness_changes`: H30 SOBE marginal, H22/H27/H31 INALTERADO
  - `validation_gap_partial_reopening` detalhe SE ridge+h22_pf+α=1

### coordination/loop_requests.md

- **NAO modificado**. Nenhum req-NNNN do loop foi processado/respondido
  nesta janela UlFor (req-0001..0003, req-0007 ja estavam DONE em
  iter_0017/iter_0024). Nenhum req-0008 emitido (padrao pre-empcao).

## PHASE C — Handoff

### Sumario maximo: 10 commits relevantes + interpretacao

Todos os 8 commits sao "relevantes" (nenhum drop). Ordem por impacto loop:

1. `a3c742a9` — **DESTRAVA SE** com ridge+h22_pf+α=1 (proposta v3) pela
   primeira vez sem trocar modelo familia
2. `42dc0d7a` — 3 frentes envelope-safe simultaneas; H22 stricter REFUTADO
   confirma `MIN_FOLDS_DROP=3` como ponto correto
3. `75e2431e` — operacionaliza promote v3 (script aceita --ridge-alpha +
   3 H22 feature_sets); destrava decisao Breno
4. `7cc3b604` — ADDENDUM val14d refuta LGBM como avenida (H27 atratividade
   nao sobe)
5. `83abab3e` — fix coerencia MATRIX nota_operacional; cosmetico
6-8. `756457c7`, `34abdd49`, `db3d2cb9` — checkpoints markers

### Mudancas no leaderboard/queue/state

- **Leaderboard**: 3 secoes novas (alpha sweep v3 + H22 stricter REFUTADO
  + ADDENDUM val14d). Tabela historico +1 linha (0026).
- **Queue**: H30 ganha note SOBE atratividade marginal; footer
  `notas_iter0026` documenta absorption read-only.
- **State**: HEAD bump 5d41d063→83abab3e, 8 commits absorvidos,
  delta_resumo com 11 campos, planner_config.next_iter_should_be
  re-priorizado (A-MONITOR / B-H30 / C-H27 / D-H22 nosso / E-req-0008
  desencorajado por pre-empcao).

### Qual hipotese fica na fila para iter_0027

**Recomendacao primaria (planner)**: **(A) MONITOR (15min) RECON_DELTA**
se UlFor avancar HEAD alem `83abab3e`. Probabilidade alta dado regime
multi-agente ATIVO (8 commits em 25min = ~3min/commit medio na janela).

**Fallback se UlFor dormente** (mais provavel iter_0027): **(B) H30**
(P3 ~1h, atratividade marginal SOBE pos-alpha-sweep). Custo zero adicional
rodar α=1 E α=10 simultaneamente; valida se vereditico CONFIRMADO_RIDGE/
REFUTADO_RIDGE muda com o alpha. Mecanismo conjecturado: menos shrinkage
permite que basis residual centrado em zero retenha mais sinal.

**Alternativas**:
- (C) H27 (P3 ~1h) — P50 quantile LGBM. Atratividade INALTERADA por
  ADDENDUM val14d; mantem como fallback baixo-risco.
- (D) H22 nosso (P3 ~1h) — GBDT vs OLS gap pdp. Lesson H22_MA +
  H23_ulfor + alpha sweep convergem em "PI deve ser medida com modelo
  final"; ainda nao testado.
- (E) req-0008 emergente (P2 ~0.5h) — pedir val_recent 14d real SE
  ridge+h22_pf+α=1. **DESENCORAJADO** — pattern pre-empcao UlFor
  multi-agente sugere auto-action <30min.

## Lessons (transferiveis)

1. **alpha=1 supera alpha=10 default em 3/4 subs com h22_MA/h22_pf**:
   sugere reavaliar defaults de outras hipoteses Ridge (H30 especificamente)
   e considerar α=1 como ponto operacional padrao quando ja houve drop H22.
2. **Regularizacao minima resolve cond_num explosivo sem trocar familia**:
   SE Ridge+α=1 elimina `cond_num 2.5e17` que travava LR em iter_0021,
   preservando feature_set h22_per_fold (apenas diferenca operacional vs
   NE/N e o alpha). Lesson generalizavel: ridge α=1 e o "intermediario
   barato" entre LR/OLS e ridge α=10 quando ha colinearidade residual
   pos-VIF/PI.
3. **Ablations negativas precisam ser publicadas**: H22 stricter foi
   testado COM expectativa de melhora (drop unanime ≡ menos ruido) e
   REFUTADO. Documentar ablacoes negativas pos-FINDING (commit `42dc0d7a`
   linha 1) protege loop de re-testar a mesma ideia.
4. **Pattern pre-empcao UlFor**: 2 iters consecutivas (0024 + 0026) com
   gap detectado pelo loop e fechado por UlFor multi-agente em <30min
   antes de req formal. Atratividade de emitir req-0008 cai a zero quando
   regime multi-agente ATIVO. Recon e' o protocolo correto.

## Budget consumido

~0.4h leitura + escrita. Bem dentro de 2.5h max por iter.

## Heartbeat

Trap EXIT no run.sh garante heartbeat. Manual aqui: `state.json.iter_atual`
agora = 26.
