# DataCost-Guard

# DataCost Guard

Analisa um ambiente **Databricks / Spark** e responde a uma pergunta que o
financeiro entende: **"quanto dá pra economizar por mês, e mexendo em quê?"**

Não vende tecnologia bonita — vende dinheiro economizado. O relatório final é
um número grande ("R$ X/mês") e uma lista priorizada de ações, cada uma com a
conta por trás.

```
┌─────────────────┐   ┌──────────────┐   ┌─────────────────┐   ┌──────────────┐
│  Snapshot do    │ → │   Loader     │ → │   9 Analyzers   │ → │   Engine     │
│  workspace      │   │ (JSON → obj) │   │  (heurísticas)  │   │ dedup + custo│
│  (APIs/tables)  │   └──────────────┘   └─────────────────┘   └──────┬───────┘
└─────────────────┘                                                   │
                                              ┌───────────────┐       │
                                              │ Relatório HTML │ ←─────┘
                                              │  (financeiro)  │
                                              └───────────────┘
```

## Como rodar

```bash
# usa os dados de exemplo inclusos
python run_analysis.py --input sample_data --output datacost_report.html

# câmbio customizado
python run_analysis.py --input sample_data --fx 5.25 --output relatorio.html
```

Imprime um resumo no terminal e gera o relatório HTML (arquivo único, abre em
qualquer navegador).

## O que ele detecta (9 categorias)

| # | Categoria | Como a economia é estimada |
|---|-----------|----------------------------|
| 1 | Cluster superdimensionado | CPU/mem ociosa → redimensiona para um pico-alvo e troca por nós menores |
| 2 | Anti-padrão de código (`repartition`/`coalesce`) | regex no código → % do runtime do job desperdiçado × custo do cluster |
| 3 | Tabela Delta com arquivos pequenos | tempo de leitura recuperável com `OPTIMIZE` × jobs que leem a tabela |
| 4 | Query sem filtro de partição | bytes podáveis com partition pruning × tempo de warehouse |
| 5 | Join com skew | cauda do straggler (task lenta vs mediana) × execuções |
| 6 | Job agendado em excesso | output não consumido, ou cadência maior que a mudança da fonte |
| 7 | Recurso ligado sem uso | horas ociosas de cluster/warehouse recuperáveis (auto-stop) |
| 8 | Falta de tags de custo | % do gasto sem atribuição de time/projeto (governança) |
| 9 | Configuração de Spark subótima | AQE/skewJoin/optimizeWrite ausentes → ganho conservador |

## Por que o número final é defensável

A ingenuidade clássica dessas ferramentas é somar todas as economias e chegar a
"você desperdiça 89% da conta" — ninguém compra isso. Vários problemas atacam
**o mesmo recurso** (redimensionar, corrigir código, ajustar config e tirar
skew consomem as mesmas horas de cluster). O `engine` aplica:

- **Teto de sobreposição por recurso** (`OVERLAP_CAP`, padrão 55%): a soma das
  economias de um mesmo cluster/warehouse não passa de 55% do custo dele. Você
  não zera 100% de um recurso que continua usando.
- **Taxa de realização global** (`GLOBAL_REALIZATION`, padrão 70%): nem toda
  recomendação é implementada e estimativas são otimistas; o total inteiro é
  descontado para uma taxa de captura realista.

Cada achado guarda o valor **bruto** e o **efetivo**; o relatório mostra os
dois. Com os dados de exemplo, isso leva de ~73% (bruto, irreal) para ~35% da
conta (efetivo, dentro da faixa que praticantes de FinOps acreditam).

## O que é real e o que é amostra

**Real (código de produção, rodando de verdade):** os analisadores, o motor de
custo (DBU + VM + câmbio), a deduplicação e o gerador de relatório.

**Amostra:** os dados em `sample_data/` são sintéticos, no **formato das APIs e
system tables do Databricks**. Conectado a um workspace real, o mesmo motor lê
dados reais sem mudança de código — só troca a camada de ingestão.

### Como conectaria ao ambiente real

- **REST APIs:** `Jobs`, `Clusters`, `SQL Warehouses` (config, uso, runs).
- **System tables:** `system.billing.usage` (custo real em DBU), `system.compute.*`
  (utilização de nós), `system.query.history` (queries, bytes, duração).
- **Delta:** `DESCRIBE DETAIL <tabela>` (nº de arquivos, tamanho, partições) e
  `DESCRIBE HISTORY` (frequência de mudança, último `OPTIMIZE`).
- **Cloud:** APIs de custo de AWS/Azure/GCP para o preço real de VM por tipo.

## Estrutura

```
datacost_guard/
  models.py          dataclasses espelhando as APIs/system tables
  pricing.py         DBU + VM + conversão USD→BRL (tarifas ilustrativas)
  findings.py        o "Finding": unidade de saída (achado + R$ + ação)
  loader.py          monta o Workspace a partir dos JSONs
  engine.py          roda os analyzers + dedup/cap + custo total
  report.py          gera o relatório HTML (tema financeiro)
  analyzers/         um arquivo por categoria de desperdício
run_analysis.py      CLI
sample_data/         snapshot sintético (clusters, jobs, tabelas, queries...)
```

## Aviso de preço

As tarifas de DBU/VM e o câmbio (`US$1 = R$5,40` por padrão) são **ilustrativos**.
Em produção, as tarifas reais do contrato e do faturamento substituem essas
constantes. As economias assumem que as recomendações são implementadas — trate
como faixa, não promessa.
