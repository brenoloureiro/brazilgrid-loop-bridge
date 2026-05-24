---
alvo: recon_delta_ulfor_post_515041e1
layer: meta
iter_num: 0019
type: recon_delta
data_utc: 2026-05-24T16:30:00Z
ulfor_head_inicio: 515041e1
ulfor_head_fim: daf80a6a
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

# Iter 0019 — RECON_DELTA UlFor (515041e1 → daf80a6a)

## Objetivo

Absorver 6 commits novos da sessao UlFor entre o ultimo commit de
producao H14-B (`515041e1`, HEAD na entrada do iter_0018) e o commit
mais recente (`daf80a6a`, fechamento experimental da frente
bias_correction com H14-F). Janela curta de ~8 min reais de UlFor
(09:27-09:34 BRT), conteudo denso: 1 infra (MLflow publico) + 1
documento de fechamento de frente + 1 experimento Pareto + 2
checkpoint markers + 1 prep code.

## PHASE A — Commits inspecionados

Em ordem cronologica (mais antigo primeiro):

| # | commit | tipo | impacto | absorvido em |
|---|--------|------|---------|---|
| 1 | `be529e93` | docs — comparacao H14-B vs H14-E + nota sobre SE/S | Atualiza `FINDING_H14E_BIAS_THRESHOLD_SIGMA_BIAS.md`: (a) tabela direta H14-B (window=60d always-on, -19.63pp, 3/5 wins, apply 100%) vs H14-E (window=28d + threshold 1*sigma_bias, -27.74pp, 3/5 wins, apply 57%) em N; (b) decisao autopilot deferida ao Breno; (c) **fechamento explicito** da frente bias_correction para SE/S: "SE/S permanecem **intrinsicamente** dificeis: bias bidirecional (mean rolling oscila em torno de 0), threshold mais apertado nao gera wins". Nao toca producao. | absorvido (doc reforca conclusao iter_0018 lessons) |
| 2 | `3b0719e2` | chore — checkpoint marker 12:45Z (2 sprints) | Marker referente ao bloco H14-D REFUTADA + H14-E PROMOVIVEL com conflito H14-B. Conteudo ja absorvido via iter_0018 (H14-D) + commit 1 acima (H14-E). | absorvido (sinal sessional) |
| 3 | `83922714` | chore — checkpoint marker 12:28Z (sprint real) | Marker referente ao bloco H14-D refutada + H14-B produtizou N@60d + H14-E doc. Conteudo ja absorvido via iter_0018 + commit 1. | absorvido (sinal sessional) |
| 4 | `1e1d4bcb` | exp — parametrizar bias_window em h14e fold_one (prep H14-F) | Espelha em `h14e_bias_threshold_sigma_bias.py` o que ja foi feito em `h14c_bias_correction_cv.py` (iter_0018 commit 4): expor `BIAS_WINDOW` como parametro de fold_one. Default 28d preservado (zero mudanca de comportamento). Habilita H14-F (combo window=60d + threshold sigma_bias). Codigo sandbox em `experiments/bakeoff_curtailment_multisub/`, NAO toca producao. | absorvido (refactor preparatorio) |
| 5 | `cd12cf2a` | **infra — expor mlflow.brazilgrid.com via Cloudflare Access** | Nginx vhost `/etc/nginx/conf.d/brazilgrid-mlflow.conf` criado no EC2 (template do clickhouse, `proxy_pass 127.0.0.1:5000`, `client_max_body_size 500M`, buffering off pra SSE). MLflow ja rodava localhost-only via systemd `mlflow.service`; smoke local `HTTP 200` + `/api/2.0/mlflow/experiments/search` OK. Falta: **2 passos manuais Breno no dash Cloudflare** (DNS A record `mlflow.brazilgrid.com` + Zero Trust Access Application replicando policy do clickhouse). Novos docs `docs/infraestrutura/SUBDOMINIOS.md` (registro mestre) + `docs/infraestrutura/MLFLOW_CLOUDFLARE_SETUP.md` (checklist passo-a-passo). `AUTOPILOT_PROMPT.md` aponta pra URL publica. **Quando ativado pelo Breno**, loop ganha acesso HTTP autenticado ao MLflow tracking URI — destrava potencialmente H18 (sanity B1-B6 local sobre champions) e elimina necessidade de derivacao indireta que motivou H19 (req-0007 ja resolveu via parquet, mas para casos novos parquet dump explicito deixa de ser estritamente necessario). | absorvido (acao Breno pendente: 2 passos CF dash) |
| 6 | `daf80a6a` | exp — H14-F window=60d + threshold sigma_bias (Pareto vs H14-B, NAO produtizar) | Combina window=60d (H14-B) com threshold k*sigma_bias_rolling (H14-E) em CV 5x60d. **N k=1: media -19.63pp, 3/5 wins, 0/5 loses, apply 70%** vs H14-B atual (mesma media, mesmos wins, **1/5 loses +0.84pp**, apply 100%). H14-F **domina H14-B no papel** (zero loses vs 1 lose marginal), mas o lose evitado em H14-B esta' abaixo do ruido amostral CV em N (~2pp). Custo de produtizar alto: precisa compute `sigma_bias_rolling_train` em runtime no loader.py (~365d replay + std por dia), ~30-50 linhas extras + cache. **Decisao autopilot: NAO produtizar** (respeita deferimento explicito ao Breno do checkpoint 12:45Z). Documentado como alternativa Pareto-superior marginal em `FINDING_H14F_WINDOW60_THRESHOLD.md`. **Achado teorico relevante**: sigma_bias ~7-12x menor que sigma_resid em todos subs; H14-F+E confirmam `sigma_bias_rolling` como threshold correto para bias-dominated regimes. **NE bias-dominated, SE/S noise-dominated, N intermedio**. NE k=1: media -4.14pp, 2/5 wins (inferior ao H14-C @28d ja produtizado). SE/S: nenhum k qualifica (k=1/2/3 todos NAO). Status quo mantido: H14-C em NE (28d), H14-B em N (60d), nada em SE/S. | absorvido (Pareto rejeitado por custo; teoria guardada) |

### O que mudou na nossa interpretacao

1. **Frente `bias_correction` FORMALMENTE FECHADA pelo UlFor**. As 6
   variacoes simples testadas (H14 janela fixa 28d, H14-B sweep
   janela, H14-C CV producao, H14-D threshold sigma_resid, H14-E
   threshold sigma_bias, H14-F combo) cobriram o search-space
   trivial. Produtizados: NE@28d always-on (iter_0017), N@60d
   always-on (iter_0018). Rejeitados: NE/N variacoes com threshold
   sigma_bias (Pareto marginal, custo alto). SE/S declarados
   **intrinsicamente nao-corrigiveis** pela frente bias (bias
   bidirecional, oscila em torno de 0). Confirma e fortalece o
   **data ceiling** iter_0017/iter_0018: SE/S precisam de **dado
   novo** (regime detection, weather nowcast, target alternativo,
   sub-stratificacao per-conjunto) para mover D+1.

2. **Achado teorico transferivel**: a razao `sigma_bias_rolling /
   sigma_resid` (~1/7 a 1/12 em todos subs) e' a metrica natural
   para classificar **bias-dominated** vs **noise-dominated**
   regimes. NE e' bias-dominated (sigma_bias ~3-6k MW comparavel a
   mean_bias_tr ~3-5k MW → correcao destrava); SE/S sao
   noise-dominated (sigma_bias 100-1k MW << sigma_resid 5-6k MW →
   correcao adiciona ruido). N e' intermediario. **Implicacao para
   nosso H29 emergente** (bias_correction sobre H10 ensemble):
   inspirar diagnostico per-sub similar antes de aplicar.

3. **MLflow exposto via Cloudflare Access (cd12cf2a) e' o maior
   sinal estrutural desta iter**. Nao muda producao mas muda a
   **superficie de auditabilidade do loop**:
   - **H18** (sanity B1-B6 sobre champions Ridge/LR, blocked por
     req-0005) ganha caminho alternativo: quando o subdominio
     estiver vivo, loop pode `GET
     https://mlflow.brazilgrid.com/api/2.0/mlflow/runs/search` (via
     CF Access token) e baixar artefatos com `mlflow.client.MlflowClient`
     setando tracking URI publico. Esto elimina a dependencia em
     UlFor publicar predicoes em parquet (req-0005). Reduz status
     `H18: blocked` para `H18: blocked-mas-acao-Breno-trivializa` —
     2 passos no dash CF.
   - **H19** ja' fechado via req-0007 (parquet). Mas para casos
     futuros (novo bake-off, nova metrica), parquet dump explicito
     deixa de ser estritamente necessario quando MLflow vier
     publico.
   - **Acao Breno pendente**: 2 passos manuais no Cloudflare dash
     (DNS A record + Zero Trust Access Application). Sem isto, vhost
     nginx existe mas sem DNS+policy o acesso publico nao funciona.
     Estado intermediario.

4. **Continua o regime "UlFor avanca em paralelo enquanto loop
   processa"**: 6 commits em ~8 min reais ao final do dia.
   Distancia desde nosso `iter0018_fim_head=515041e1` foi
   irrelevante em tempo de relogio, mas em commits acumulados
   foi denso (1 fechamento de frente bias + 1 infra + 1
   experimento + 1 prep). Lesson reforca **freq de recon
   curta** quando UlFor esta produzindo.

### Reqs / hipoteses

- **Nenhum novo request emitido**. `open_requests=[]` continua.
  H14-F custo alto + ganho marginal nao justifica req formal
  (Breno ja deferido via checkpoint).
- **Nenhum request UlFor fechado nesta iter**. req-0001/2/3/7 ja
  DONE; req-0004/5/6 nunca emitidos.
- **Nenhuma H do loop resolvida**. H14-F e' UlFor interno
  (extensao da UlFor H14/H14-B/H14-E ja absorvidas). Nomenclatura
  Hxx_ulfor mantida.
- **H18 status_change**: blocked → blocked (acao Breno trivializa).
  Mensagem atualizada no hypotheses_queue: "iter_0019: MLflow
  exposto via Cloudflare Access (commit cd12cf2a) — vhost nginx
  pronto, falta DNS+ZT Access policy no dash CF. Quando ativado,
  loop pode auditar champions diretamente via MLflow REST sem
  req-0005."
- **H29 emergente (NAO criada formalmente esta iter)**: mesma
  proposta do iter_0018 (bias_correction per-sub sobre H10
  ensemble). Atratividade aumenta marginalmente: a teoria
  sigma_bias/sigma_resid d ao loop um diagnostico previo para
  prever em quais subs o bias_correction deve ajudar antes de
  rodar o experimento.

### Status quo da producao apos iter_0019

| sub | champion (MLflow) | bias correction default | window | bate persist 14d real? | mudou em iter_0019? |
|---|---|---|---|---|---|
| NE | ridge_curt_ne_d1@v1 (full, 55 feat) | **ON** | 28d | **SIM** (49.2% vs 54.1% persist, -4.9pp) | NAO |
| SE | lr_curt_se_d1@v1 (full, 55 feat) | OFF | 28d | NAO | NAO |
| S  | lr_curt_s_d1@v1 (full, 55 feat) | OFF | 28d | NAO | NAO |
| N  | ridge_curt_n_d1@staging v2 (clean_plus, 31 feat) | **ON** | **60d** | n/a (raw bate persist) | NAO |

**Producao 100% inalterada** nesta iter. Loader.py nao tocado.
Modelos MLflow nao tocados.

### Lessons learned (incrementais)

- **Frente bias_correction fechada com 2 wins + 2 closures**:
  NE+N produtizam (positivo), SE+S declarados intrinsicamente
  nao-corrigiveis (negativo mas final). UlFor passa para proximas
  frentes (provavelmente regime detection, sub-stratificacao
  per-conjunto, ou target alternativo log1p). Padrao para nos:
  **frente fechada explicitamente vale mais que frente abandonada
  silenciosamente** — evita revisitar.
- **Custo de producao matters**: H14-F dominou H14-B no papel mas
  custo de codigo + compute em runtime nao justificou o ganho
  marginal. Padrao recorrente em hipoteses ortogonais: **se
  delta_metrica abaixo do ruido amostral CV, prefira o
  simpler-on-prod**. Aplicavel ao nosso bake-off H21/H24/H27.
- **Infra como destrava**: MLflow publico (commit `cd12cf2a`) e'
  um exemplo de "trabalho infra invisivel que destrava 2-3
  hipoteses bloqueadas" (H18 acima). Recon iter expoe isso e
  evita que loop fique aguardando dependencia ja eliminavel.

### Proxima iter

`iter_0020` retoma planner_config:

**(opcao A — recomendada)** **H21** (P2 pdp_residual =
pdp_prev - gen, codavel local, 1.5h). Razoes inalteradas desde
iter_0014/15/16/17/18. Bias_correction iter_0017/18 nao afeta
H21 diretamente (H21 e' feature engineering, ortogonal ao
post-processing).

**(opcao B)** H29 (P2/P3, ~1.5h): aplicar bias_correction
rolante per-sub (NE=28d, N=60d, SE/S=OFF per achado teorico
iter_0019) sobre nosso H10 ensemble (LGBM+persist_d1) no
replay loop.

**(opcao C)** **H24** (P2 ensemble Ridge/LR vs persist, ~1.5h).
Baseline atualizado per iter_0019: NE@bias_28d + N@bias_60d ja
PROD.

**(opcao D — nova considerar se Breno completar passos CF)**
H18 (sanity B1-B6 sobre champions via MLflow REST direto).
Trigger: Breno conclui DNS + ZT Access setup do MLflow publico.

Alt: H27 (P3 P50 quantile substituto custo zero), H26 (P3
conformal post-hoc), H22 (P3 GBDT vs OLS gap pdp), H25 (P3
stacker), H8 (P3 perm intercambio), H20 (P3 auto-flag n_test<30).

### Sanity checks

Recon iter — nenhum requerido. 6 defaults nao aplicaveis (sem
treino/feature novo). NAO computado.
