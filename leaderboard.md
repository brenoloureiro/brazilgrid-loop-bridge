# Leaderboard — forecast-mega-loop

Estado de cada alvo atravessando o DAG. Linha por (layer, alvo).
Atualizado pelo watchdog ao final de cada iteração com ganho promovido.

| layer | alvo | baseline | melhor_metrica | iter_vencedor | sanity_ok | data_utc |
|---|---|---|---|---|---|---|
| _vazio_ | _aguardando iter 1_ | — | — | — | — | — |

---

## Legenda

- **baseline**: valor da métrica primária no baseline (persistência, climatologia, etc.)
- **melhor_metrica**: melhor valor já obtido por algum modelo desta camada
- **iter_vencedor**: número da iteração que produziu a melhor métrica
- **sanity_ok**: `[5/5]` = todos os 5 sanity checks passaram
- **data_utc**: ISO 8601 UTC do registro
