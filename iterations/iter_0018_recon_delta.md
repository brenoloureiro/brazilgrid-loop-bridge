---
alvo: recon_delta_ulfor_post_5c7963d4
layer: meta
iter_num: 0018
type: recon_delta
data_utc: 2026-05-24T15:30:00Z
ulfor_head_inicio: 5c7963d4
ulfor_head_fim: 515041e1
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

# Iter 0018 — RECON_DELTA UlFor (5c7963d4 → 515041e1)

## Objetivo

Absorver 6 commits novos da sessao UlFor entre o checkpoint 06:25Z
(`5c7963d4`, HEAD na entrada do iter_0017) e o checkpoint 09:25Z
(`515041e1`, ultimo commit de producao H14-B). Recon-only — sem testar
hipotese de modelagem, so reconcilia state/queue/leaderboard com o
trabalho que UlFor completou em paralelo enquanto loop processava o
recon do iter_0017.

## PHASE A — Commits inspecionados

Em ordem cronologica (mais antigo primeiro):

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `0c2f7429` | chore — checkpoint 12:13Z (autopilot idle) | Marker FORCADO pelo runner headless (gatilho: 348 min > 240 min sem checkpoint). Sessao zero-trabalho, HEAD inalterado, tree limpo. Estado: "H21 REFUTADA, H14 PARCIAL (NE produtizado), H22+H14-B+H14-D abertas. 25 commits ahead origin/master aguardam push do Breno (deploy EC2)." | absorvido (sinal sessional, sem impacto) |
| 2 | `2b262f51` | chore — checkpoint 12:11Z (idle 2x) | Marker idle. UlFor estagnado aguardando Breno em deploy EC2 vs proximas Hs. | absorvido |
| 3 | `1804c496` | chore — checkpoint 12:15Z (idle 3x) | 3o marker idle do dia. UlFor aceita: precisa autorizacao para mover (deploy EC2 = PARAR E PERGUNTAR) ou retomar H22/H14-B/H14-D. | absorvido |
| 4 | `4871998c` | exp — H14-D bias correction threshold k*sigma_resid_train (REFUTADA) | UlFor H14-D testou aplicar bias correction so quando `\|bias_window_28d\| > k*sigma_resid_train` (k=1,2,3). Hipotese: bias rolling 28d e' noisy em SE/S/N e adiciona ruido na maioria dos folds, mas seria sinal quando supera ruido residual. Resultado: NAO produtizar. NE@k=1 wins caem 1/5 (vs 3/5 sem threshold) — perde os ganhos H14-C. SE/S/N zero wins (threshold filtra demais, apply_rate ~0-10%). Threshold NAO destrava SE/S/N. Status atual mantido. Bonus: parametrizou bias_window em h14c.fold_one (preparacao para H14-B). | absorvido |
| 5 | `eb9fca05` | exp — H14-B sweep da janela bias correction (28/60/90d) | UlFor H14-B sweep cross-window. Resultado per-sub (melhor janela): NE 28d (4/5 wins, -7.07pp) ja' produtizado. **N 60d (3/5 wins, 1/5 loses, -19.63pp media) → PRODUTIZAR [NOVO]**. NE@60d regride a 2/5 wins (NAO migrar). SE 0 wins em qualquer janela. S 2 wins max contra 3 loses em qualquer janela. Tabela cross-window publicada em `experiments/bakeoff_curtailment_multisub/FINDING_H14B_WINDOW_SWEEP.md`. Fold-4 N (2025-07/09 seca) e' dominante (-78pp em 60d, raw 226% — modelo colapsa, bias compensa). Outros 4 folds N ganho modesto (-1pp a -12pp). | absorvido — gatilho para commit 6 |
| 6 | `515041e1` | feat — **produtizar bias correction em N (window=60d) per H14-B** | **MUDANCA EM PRODUCAO**. `services/analytics_api/forecast/loader.py`: `BIAS_CORRECTION_DEFAULTS["N"]: False → True`; novo `BIAS_CORRECTION_WINDOW_BY_SUB = {NE:28, SE:28, S:28, N:60}`; `predict_d1` usa sub_window dinamico; response.bias_correction.window_days reflete window real per-sub; source agora cita `FINDING_H14C_BIAS_CV.md + FINDING_H14B_WINDOW_SWEEP.md`. Smoke 4 subs: **N pred 246 → default 267, bias -21, applied=True** (era applied=False iter_0017). NE inalterado window=28d apply=True (raw 62878 → default 55926). SE/S default OFF (raw=default). Backward-compat: `predicted_curt_mwh` (raw) inalterado; callers que ja usam `predicted_curt_mwh_default` em N agora recebem corrected. `predicted_curt_mwh_corrected` (sempre exposto) inalterado. analytics_api NAO deployada ainda no EC2. | absorvido |

### O que mudou na nossa interpretacao

1. **Champion N efetivo MUDA em PRODUCAO sem mudar modelo treinado** (analogo
   ao NE em iter_0017). Wrapper de inferencia agora aplica
   bias_correction_60d default ON em N. Marco: segunda sub a ganhar
   bias correction productized. Modelo treinado (`ridge_curt_n_d1@staging
   v2 clean_plus 31 feat`) INALTERADO no MLflow Registry — bias
   correction e' post-processing layer no inference time. Bias absoluto
   N e' MUITO menor que NE (~21 MWh vs ~7.3k MWh), mas o ymean tambem
   e' menor (~506 MWh vs ~75.7k MWh) → o ganho relativo CV e' maior em
   N (-19.6pp media) do que em NE (-7.1pp media). Fold-4 N catastrofico
   (-78pp) puxa a media, mas wins 3/1 dos outros folds sao reais e robustos.

2. **`BIAS_CORRECTION_WINDOW_BY_SUB` agora explicito** (era global 28d
   implicito iter_0017). Indica que UlFor adotou a convencao de
   **janela per-sub** — abre porta para futuros ajustes (e.g., NE migrar
   para janela maior se regime estabilizar, ou S/SE com janela diferente).
   Implementacao codavel local: nosso H14-equivalente teria que
   contemplar essa flexibilidade ao planejar H29 (bias correction sobre
   nosso H10 ensemble).

3. **H14-D (threshold k*sigma) cobriu mais um espaco do search-space
   bias correction e veio negativa**. Combinado com H14-B (sweep de
   janela), o espaco de variacoes simples do bias correction esta'
   praticamente esgotado: janela curta/media/longa testada (H14-B),
   threshold do sigma testado (H14-D). UlFor proximos passos sao
   provavelmente **H22 (sazonalidade do bias)** ou deploy EC2 da
   producao atual. **Implicacao loop**: bias correction nao vai
   destravar SE/S sem **dado novo** (regime detection, weather
   nowcast, ou target alternativo) — confirma data ceiling iter_0017.

4. **3 checkpoints idle em sequencia** sinalizam que UlFor esta em
   pausa aguardando decisao Breno (deploy EC2 ou retomar Hs abertas).
   25 commits ahead origin/master. Nao afeta nosso plano — loop
   continua independente. Sinal para nos prepararmos para um possivel
   ramo de **mais commits acumulados** quando Breno autorizar deploy
   ou nova frente.

### Reqs / hipoteses

- **Nenhum novo request emitido**. open_requests=[] continua.
- **Nenhum request UlFor fechado nesta iter**. req-0001/2/3/7 ja' DONE.
- **Nenhuma H do loop resolvida** pelos commits. H14-B/H14-D sao
  UlFor internos (extensoes da UlFor H14 ja absorvida iter_0017).
  Disambig nomenclatura aplicada: `Hxx_ulfor` em comments.
- **H29 emergente (NAO criada formalmente nesta iter)**: aplicar
  bias_correction com janela per-sub sobre nosso H10 ensemble
  (LGBM+persist). Bias_correction e' ortogonal a ensemble (post-processing
  layer diferente), pode compor. H10 e' replay loop sobre LGBM iter_0002;
  bias_correction UlFor e' sobre champion Ridge/LR. Sequencia natural:
  primeiro H21 (pdp_residual feature) → H24 (ensemble sobre Ridge/LR) →
  H29 (bias_correction sobre H24).

### Status quo da producao apos iter_0018

| sub | champion (MLflow) | bias correction default | window | bate persist 14d real? |
|---|---|---|---|---|
| NE | ridge_curt_ne_d1@v1 (full, 55 feat) | **ON** | 28d | **SIM** (49.2% vs 54.1% persist, -4.9pp) |
| SE | lr_curt_se_d1@v1 (full, 55 feat) | OFF | 28d | NAO (77.4% vs 69.6% persist, +7.8pp) |
| S  | lr_curt_s_d1@v1 (full, 55 feat) | OFF | 28d | NAO (159.6% vs 99.9% persist, +59.7pp) |
| N  | ridge_curt_n_d1@staging v2 (clean_plus, 31 feat) | **ON (NOVO iter_0018)** | **60d** | n/a (raw ja bate persist em 14d real per iter_0017 smoke) |

### Lessons learned (incrementais)

- **Per-sub hyperparams sao a regra, nao excecao**. Bias_correction
  window=28d ON em NE, window=60d ON em N, OFF em SE/S e a primeira
  evidencia clara que diferentes janelas serao a norma. NE foi 4/5
  wins @28d; N foi 1/5 → 3/5 wins quando alargou para 60d. Razao
  fisica: fold-4 N seca-2025Q3 e' anomalia que precisa de janela
  longa para nao contaminar mean. Padrao a importar pro loop: **toda
  hipotese com hiperparam global deve testar variacao per-sub**.
- **Search-space de bias correction simples esgotando**. H14 (janela
  fixa 28d), H14-B (sweep 28/60/90d), H14-C (CV producao 28d), H14-D
  (threshold sigma). Proximos passos uteis precisam ou
  (a) regime detection (apply OFF em transicao chuvoso→seco), ou
  (b) sub-stratificacao (per-conjunto), ou (c) target alternativo
  (log1p, classifier).
- **Recon iter continua sendo barato e absolutamente necessario**.
  6 commits em ~3h reais de UlFor desde nosso checkpoint anterior
  (iter_0017) — N ganhou bias_correction default ON sem que loop
  soubesse. Custo recon ~0.4h.

### Proxima iter

`iter_0019` retoma planner_config com pequena variacao:
**(opcao A — recomendada)** **H21** (P2 pdp_residual = pdp_prev - gen,
codavel local, 1.5h). Razoes inalteradas desde iter_0014/15/16/17.
Considerar ensemble post-processing (H10) + LGBM (H7) como baseline
final do bake-off H21. Bias_correction iter_0018 nao afeta H21
diretamente (H21 e' feature engineering, ortogonal).

**(opcao B — nova)** H29 (P2/P3, ~1.5h): aplicar bias_correction
rolante per-sub (NE=28d, N=60d, SE=28d, S=28d) sobre nosso H10
ensemble (LGBM+persist_d1) no replay loop. Confirmar se ganhos
bias_correction se propagam quando o modelo base e' ensemble em
vez de Ridge/LR. Se sim, abre via FINDING ao UlFor.

**(opcao C)** **H24** (P2 ensemble Ridge/LR vs persist, ~1.5h).
Baseline mudou de novo (NE ja tinha bias_correction iter_0017;
N ganhou iter_0018). Comparacao agora deve ser:
ridge_curt_ne_d1@bias_28d_ON vs ridge_curt_ne_d1@bias_28d_ON+persist_ensemble.

Alt: H27 (P3 P50 quantile substituto custo zero), H26 (P3 conformal
post-hoc), H22 (P3 GBDT vs OLS gap pdp), H25 (P3 stacker), H8 (P3
perm intercambio), H20 (P3 auto-flag n_test<30).

### Sanity checks

Recon iter — nenhum requerido. 6 defaults nao aplicaveis (sem
treino/feature novo). NAO computado.
