---
alvo: recon_delta_ulfor_post_daf80a6a
layer: meta
iter_num: 0021
type: recon_delta
data_utc: 2026-05-24T17:30:00Z
ulfor_head_inicio: daf80a6a
ulfor_head_fim: ec0fd937
commits_absorvidos: 6
novos_requests: 0
requests_fechados_extras: 0
novas_hipoteses_loop: 0
hypothesis: null
baseline_tipo: null
sanity_checks_required: []
sanity_checks_done: []
budget_horas: 0.4
---

# Iter 0021 — RECON_DELTA UlFor (daf80a6a → ec0fd937)

## Objetivo

Absorver 6 commits novos da sessao UlFor entre `daf80a6a` (HEAD na
entrada do iter_0020, fechamento da frente bias_correction com H14-F)
e `ec0fd937` (checkpoint 13:25Z, abertura da frente H22 multi-agente).
Janela ~15 min reais de UlFor (13:10-13:35 BRT). Conteudo denso e de
**alta importancia para o leaderboard**: 1 documento de fechamento
H14-F (Pareto marginal, NAO produtizar) + 1 sprint H22 (**champion
candidate NE+SE potencial via VIF per-fold + Permutation Importance
per-fold**) + 1 anomalia LR_SE numericamente patologica + 1 H23
load-bearing (lesson teorico PI-com-Ridge subestima colineares) +
2 checkpoint markers.

## PHASE A — Commits inspecionados

Em ordem cronologica (mais antigo primeiro):

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `1f178801` | chore — checkpoint 13:10Z (H14-F NAO produtizar, Pareto marginal) | Marker referente ao fechamento explicito de H14-F: combo `window=60d + threshold k*sigma_bias`, N k=1 media -19.63pp, 3/5 wins, 0/5 loses, apply 70%. Pareto-domina H14-B no papel (mesmo upside, zero loses vs 1 lose pequeno) **mas custo produtizar (sigma_bias historico no loader, ~30-50 LoC) NAO justifica**. Status quo preservado. **Achado teorico**: sigma_bias_rolling/sigma_resid ~1/7-1/12 consistente em todos subs — sigma_bias e' a metrica correta para threshold (confirma refutacao H14-D). Conteudo ja absorvido em iter_0019 (commit `daf80a6a`). | absorvido (marker pos-experimento, conteudo iter_0019) |
| 2 | `10fa56d3` | **exp(forecast): H22 ulfor VIF per-fold + Permutation Importance — PROMOVIVEL NE/SE** | **Novo wrapper que corrige refutacao de H8 ulfor (greedy puro VIF)**: cruza VIF (TRAIN only, sem leak) com permutation importance per-fold para identificar features estatisticamente redundantes E preditivamente inuteis. Bake-off CV 5x60d full vs h22_per_fold (FEATURE_DROPS_H22_PER_FOLD): **NE ridge 33.7→31.4% NMAE (-2.3pp), R² +0.469→+0.543 (+0.074)**; **NE lr 40.3→30.8% (-9.5pp), R² +0.302→+0.565 (+0.263)**; **SE ridge 50.8→48.3% (-2.5pp), R² +0.196→+0.390 (+0.194)**; **S/N neutral**. "Champion atual SERIA H22 em NE/SE." UlFor **NAO promove automaticamente** (decisao consciente, deferimento Breno). Artefatos: `h22_vif_perm_per_fold.py`, `FINDING_H22_VIF_PERM_PER_FOLD.md`, novo `feature_set=h22_per_fold` no bakeoff_d1.py, `outputs/cv_summary_*_h22_per_fold.parquet`. **Lesson estrutural**: multicolinearidade estatistica (VIF) sozinha **nao** prediz drop (H8 refutado); permutation importance per-fold (nao agregada) e' o wrapper que falta. | absorvido — **PROXIMO CHAMPION CANDIDATE NE/SE; aguarda decisao Breno** |
| 3 | `34478f93` | exp(forecast): H22 anomalia LR_SE patologica | Probe LR_SE/full revelou PI values absurdos (>800.000pp NMAE para carga_mwmed/carga_liquida/val_import). **Causa**: matriz X SE/full com condition number ~2.5e17, coefs LR oscilam massivamente. NMAE 46.6% em FULL e' acidental — qualquer perturbacao explode. **Sucessor recomendado SE: ridge+h22** (48.3±14.2% NMAE, R² +0.390). Marginalmente pior em NMAE que LR-instavel (~2pp) mas R² maior, sem risco numerico. **NAO promove** (decisao Breno). | absorvido — **sinaliza fragilidade numerica do champion SE atual (lr) escondida por sorte amostral; recomendacao implicita: mover SE de lr para ridge+h22** |
| 4 | `ccf53722` | exp(forecast): H23 SE LR load-bearing — ter_verif_rmean7 SOZINHA salva LR | Investigacao da anomalia H22: SE LR FULL R² +0.380 → H22_drop R² -0.448 (delta -0.828), enquanto SE Ridge FULL +0.196 → H22_drop +0.390 (delta +0.194). Metodologia: leave-one-in. **Resultado**: `ter_verif_rmean7` SOZINHA recupera R² para +0.301 (vs H22 baseline -0.080, delta +0.381, -7.21pp NMAE). Quase recupera FULL (+0.345). Secundarias: cmo_mwmed_rmean7 (+0.173), prev_solar_pico_mw (+0.099), prev_solar_n (+0.061). **Insight teorico**: PI medida com Ridge subestima importancia de features colineares. `ter_verif_rmean7` tinha PI -0.043pp em H22 ("inutil"), mas era artefato de Ridge redistribuir sinal via L2 entre `ter_verif_lag1` / `ter_verif` / `ter_prog`. LR sem shrinkage colapsa quando removida. **Lesson**: PI deveria ser medida com o modelo final que sera usado, nao com proxy mais robusto. H22 (que usou Ridge para PI) e' valido para Ridge; para LR precisaria PI-com-LR per fold. SE LR nao esta em prod — se promover SE com H22_drop, **usar Ridge**. Artefatos: `h23_se_lr_load_bearing.py`, `FINDING_H23_SE_LR_LOAD_BEARING.md`, `outputs/cv_summary_h23_se_lr_loi.parquet`. | absorvido — **conclui investigacao SE LR; reforca recomendacao "SE → ridge+h22"** |
| 5 | `9d9a5979` | chore — checkpoint 13:35Z (H22 PROMOVIVEL NE/SE + anomalia LR_SE patologica) | Marker sintetizando H22 + H22 anomalia LR_SE. Conteudo absorvido nos commits 2+3. | absorvido (sinal sessional) |
| 6 | `ec0fd937` | chore — checkpoint 13:25Z (sprint H23 multi-agente, preventivo) | Checkpoint PREVENTIVO apos 1 sprint pessoal (vs minimo 3): "2-3 agentes paralelos ativos, risco duplicar maior que beneficio acumular sprints". Convergencia com agente paralelo: addendum LR_SE patologico (cond number 2.5e17). **Ambos recomendam ridge+h22 como sucessor SE**. Master agora 5+ ahead origin (sem conflito de merge entre commits paralelos). Conteudo absorvido nos commits 2-4. | absorvido (sinal sessional — sinaliza regime multi-agente ativo no UlFor) |

### O que mudou na nossa interpretacao

1. **Champions NE e SE tem candidatos sucessores NOVOS** prontos para
   decisao Breno via `feature_set=h22_per_fold`. Magnitudes
   relevantes:
   - **NE ridge**: -2.3pp NMAE, +0.074 R² — moderado
   - **NE lr**: -9.5pp NMAE, +0.263 R² — **enorme**
   - **SE ridge**: -2.5pp NMAE, +0.194 R² — moderado
   - **SE lr**: anomalo (FULL +0.380 mas patologico; drop colapsa)
   - **S/N**: neutral (H22 nao destrava)

   Numeros 14d real **nao** rodados (so CV 5x60d). UlFor explicitamente
   deferiu promocao ao Breno. Nosso leaderboard mantem champions atuais
   (ridge_NE@v1 full, lr_SE@v1 full, lr_S@v1 full, ridge_N@staging v2
   clean_plus) — adiciona uma **nota de candidato sucessor** nas
   linhas NE e SE.

2. **Fragilidade numerica do champion SE atual (lr) ficou explicita**
   (commit `34478f93`). Cond number ~2.5e17 da matriz X SE/full
   significa que qualquer perturbacao numerica (drop de feature,
   re-treino com seed diferente, mudanca de blas) **pode explodir**.
   O NMAE 46.6% atual e' "acidental" (sorte amostral). Combinado com
   H23 (commit `ccf53722`), a recomendacao implicita de UlFor e':
   **SE → ridge+h22 e' mais robusto que lr_SE@full ou ridge_SE@full**.
   Magnitude vs lr_full: ~+2pp NMAE pior, mas R² +0.39 vs +0.38
   (essencialmente empate) **sem risco numerico**.

   Esto mantem nosso `data_ceiling D+1 SE` confirmado em iter_0017
   (nenhum ML bate persist em 14d real SE) — H22 nao desafia esse
   teto porque foi medido em CV 5x60d (que ja era favoravel para
   FULL), nao em holdout 14d real. Sugestao implicita: rodar H22
   tambem em 14d real antes de promover.

3. **Lesson teorico transferivel (H23 ulfor)**: **PI deve ser medida
   com o modelo final, nao com proxy mais robusto**. Ridge usa L2
   para redistribuir importancia entre features colineares; PI
   medida em Ridge subestima features que Ridge "carregou" em
   features-vizinhas. LR sem shrinkage colapsa quando essas
   features sao removidas porque nao tem o mesmo mecanismo de
   redistribuicao.

   Aplicavel ao nosso H22 (GBDT vs OLS gap para curt~gen+pdp,
   ainda queued): se rodarmos PI sobre OLS para escolher features
   e treinarmos GBDT depois, podemos subestimar features que GBDT
   precisa para interacoes nao-lineares. Mesma classe de erro.

4. **Regime UlFor mudou para multi-agente paralelo** (checkpoint
   `ec0fd937`): "2-3 agentes paralelos ativos, risco duplicar maior
   que beneficio acumular sprints" → checkpoints preventivos apos
   1 sprint pessoal (vs minimo 3 antes). Sinal de aceleracao da
   sessao UlFor. Para nos:
   - **freq de recon precisa aumentar** (talvez 1 recon a cada
     2 hypothesis_test em vez de 1 a cada 5)
   - **probability de colisao Hxx_ulfor vs nosso Hxx aumenta**
     (UlFor agora abre H22+H23 em paralelo; ja tinha colisao com
     nosso H21 em iter_0017). Convencao Hxx_ulfor mantida.

5. **Frente bias_correction continua FECHADA** (iter_0019). Nenhum
   commit desta janela toca bias_correction (so referencia
   teorica em H22 sobre fragilidade colinear).

### Reqs / hipoteses

- **Nenhum novo request emitido**. Promocao H22 para NE/SE e' decisao
  Breno (UlFor explicitamente deferiu). Loop nao tem autoridade para
  pedir UlFor "promover" — pode propor follow-up de auditoria via req
  mas custo/beneficio ruim agora (UlFor ja' tem todos os numeros e o
  parquet publicado).
- **Nenhum request UlFor fechado nesta iter**. req-0001/2/3/7 ja DONE;
  req-0004/5/6 nunca emitidos.
- **Nenhuma H do loop resolvida**. H22 UlFor (VIF per-fold + PI per-fold)
  e' **independente** do nosso H22 (GBDT vs OLS gap para curt~gen+pdp,
  queued). Colisao de numeracao Hxx_ulfor vs nosso Hxx **continua
  pattern recorrente** — convencao mantida.
- **Nenhuma nova H do loop gerada formalmente**. Candidato emergente
  H31: **auditar feature_set=h22_per_fold em 14d real (holdout
  pos-2026-05-08)** antes de Breno promover. Custo baixo (parquet
  publicado), mas barreira: H22 e' UlFor-side, parquet `cv_summary_*`
  e' CV nao holdout. Precisaria rodar bakeoff_d1.py com
  `--feature-set h22_per_fold` em holdout NE+SE — codavel, mas requer
  UlFor venv (nao loop venv). Deixar como candidato P3 para hypotheses_queue.

### Status quo da producao apos iter_0021

| sub | champion (MLflow) | bias correction default | window | bate persist 14d real? | h22_per_fold candidato? | mudou em iter_0021? |
|---|---|---|---|---|---|---|
| NE | ridge_curt_ne_d1@v1 (full, 55 feat) | **ON** | 28d | **SIM** (49.2% vs 54.1% persist, -4.9pp) | **SIM** (-2.3pp NMAE CV; lr seria -9.5pp, mas champion atual e' ridge) | NAO |
| SE | lr_curt_se_d1@v1 (full, 55 feat) | OFF | 28d | NAO | **SIM mas trocar para ridge+h22** (lr_SE patologico cond=2.5e17; ridge_SE+h22 +0.194 R² CV) | NAO |
| S | lr_curt_s_d1@v1 (full, 55 feat) | OFF | 28d | NAO | NAO (H22 neutral em S) | NAO |
| N | ridge_curt_n_d1@staging v2 (clean_plus, 31 feat) | **ON** | **60d** | n/a (raw bate persist) | NAO (H22 neutral em N) | NAO |

**Producao 100% inalterada** nesta iter. Loader.py nao tocado. Modelos
MLflow nao tocados. **Mas o "campo de batalha" mudou**: NE e SE tem
sucessor pronto + recomendacao implicita para SE trocar familia
(lr → ridge). Acao Breno pendente acumula com 2 anteriores (deploy EC2
+ MLflow CF Access).

### Lessons learned (incrementais)

- **PI mede-se com o modelo final**, nao com proxy mais robusto. Ridge
  redistribui sinal via L2 entre colineares — PI subestima. Validar
  importancia em duas familias antes de fazer drop universal
  (H22 UlFor passou nesse teste para Ridge; falhou para LR — por isso
  H23 surgiu).
- **Cond number como warning silencioso**: cond_num >1e16 em qualquer
  matriz X de modelo linear (LR sem shrinkage) e' bomba relogio.
  Champion SE atual (lr_SE@full, cond=2.5e17) so' nao explodiu por
  sorte. Ridge L2 atenua mas nao elimina (Ridge ainda tem cond_num
  alto em FULL, mas redistribui em vez de oscilar). **Lesson para
  nosso bake-off**: reportar cond_num da matriz X junto com NMAE/R²
  no leaderboard (proximo H32 candidato a queue, P3 metric).
- **Multi-agente UlFor → freq recon ↑**: 1 sprint produziu H22+H23
  em paralelo com 1 anomalia. Recon iter_0021 quase nao foi acionado
  a tempo (HEAD pulou 6 commits em 15 min). **Padrao**: se UlFor
  expoe que esta multi-agente (checkpoint `ec0fd937`), reduzir
  intervalo de recon para max 30-60 min em vez de 2-4h.

### Proxima iter

`iter_0022` retoma planner_config. **Recomendacao atualizada**:

**(opcao A — recomendada nesta iter pos-recon)** **H27** (P3 P50
quantile como point estimate substituto, ~1h). Custo zero (objective
swap LGBM regression → quantile alpha=0.5), ganho transversal
garantido em N (-18% MAE iter_0014 bonus) + S (-14%). Ortogonal a
tudo que UlFor fez na frente NE/SE. Nao colide com nenhum H22/H23
ulfor.

**(opcao B)** **H30** (P3 ~1h, derivada H21 iter_0020) — pdp_residual
em Ridge_alpha10 CV 5x60d. Fecha H3-family residual no replay loop.
Script base ja existe (`scripts/h21_pdp_residual_cv.py`).

**(opcao C — atratividade SUBIU pos-iter_0021)** **H22 (NOSSO)**
(P3 GBDT vs OLS gap para curt~gen+pdp, ~1h). UlFor H22 mostrou que
PI-com-Ridge subestima features colineares — esse mecanismo pode
explicar parte do gap entre OLS e GBDT (GBDT extrai interacao
nao-linear, mas se PI-com-Ridge usado para escolher features, gap
fica oculto). Convergente com H23 ulfor lesson.

**(opcao D)** **H29 emergente** (P2/P3, ~1.5h): aplicar
bias_correction per-sub (NE=28d, N=60d, SE/S=OFF) sobre H10 ensemble
LGBM+persist_d1 no replay loop.

**(opcao E — agora MAIS valiosa)** **NOVA H31 candidata** (P3 ~0.5h):
**replicar h22_per_fold em holdout 14d real NE+SE** (codavel via
bakeoff_d1.py --feature-set h22_per_fold, mas no UlFor venv).
**Alavanca decisao Breno** de promover/nao promover H22 sucessor —
informa se H22 segue a mesma armadilha de H21_ulfor (ganho CV → perda
holdout 14d). Pode ser proposta como req-0008 ao UlFor (P2,
data_publish: rodar holdout 14d real para feature_set=h22_per_fold em
NE+SE, dump parquet). Custo loop: 0.5h escrita req. Custo UlFor:
~30min execucao. Ganho: destrava decisao Breno informada.

**(opcao F — Breno-trigger)** **H18** (sanity B1-B6 via MLflow REST
direto). Trigger: Breno completa DNS A + ZT Access para
mlflow.brazilgrid.com.

Alt: H26 (P3 conformal post-hoc), H25 (P3 stacker), H8 (P3 perm
intercambio), H20 (P3 auto-flag n_test<30), H15 (P3 rare-event
classifier S).
