# Requisitos do Datathon

Este documento mapeia cada etapa pedida no edital do Datathon (grupo 68) para o que existe
hoje no código, de forma objetiva: o que foi implementado, onde, e o que ainda falta. Foi
escrito a partir de uma leitura do PDF do desafio e de uma auditoria do repositório nesta
branch (`git log` em `main` no commit `21033bd`).

## Sumário

| Etapa | Requisito | Status |
|---|---|---|
| 0 | Organização do projeto | 🟡 parcial — falta conteúdo no `README.md` raiz |
| 1 | Base Kaggle + EDA | 🟡 parcial — EDA feita, link da base fora do `README.md` |
| 2 | Preparação da base | 🟢 atendido |
| 3 | Baseline + estratégia algorítmica | 🟢 atendido |
| 4 | Avaliação + casos de teste | 🟢 atendido (excede o pedido) |
| 5 | Serviço/interface demonstrável | 🟢 atendido (excede o pedido) |
| 6 | Arquitetura-alvo em nuvem | 🟢 atendido |
| 7 | Ciclo de vida MLOps (MLflow) | 🟢 atendido |
| 8 | Apresentação final (Demo Day) | 🔴 não feito — vídeo pitch pendente |
| — | Governança de dados (LGPD) | 🔴 não feito — documentação ausente |

🟢 atendido · 🟡 atendido parcialmente · 🔴 não atendido (issue aberta).

---

## Etapa 0 — Organização do projeto

**Pedido:** repositório público, `README.md` com visão do problema e instruções de execução,
código organizado com dependências declaradas.

**O que existe:**

- Repositório público (`andrevberaldo/datathon-7mlet-grupo-68`, fork de `JabuS2/datathon-7mlet-grupo-68`).
- Dependências declaradas por serviço (não há um único `requirements.txt` de raiz — é um monorepo
  de múltiplos serviços deployáveis independentemente):
  `api_service/pyproject.toml`, `model_service/pyproject.toml`, `agent_service/pyproject.toml`,
  `mcp_server/package.json`, `front_service/package.json`, `apps/dashboard/package.json`,
  `notebooks/requirements.txt`, `docs/requirements.txt`.
- Documentação além do README: site MkDocs completo (`mkdocs.yml`, `docs/`), publicado em
  <https://jabus2.github.io/datathon-7mlet-grupo-68/>.

**Gap:** o `README.md` da raiz não traz um parágrafo de "visão do problema" nem instruções de
execução local — só a tabela de serviços e o parágrafo de nuvem (Etapa 6). As instruções de
execução existem, mas espalhadas (`docs/infra.md`, READMEs de cada serviço). Ver issue de
pendências no fim deste documento.

## Etapa 1 — Base Kaggle e EDA

**Pedido:** escolher uma base Kaggle (lista sugerida ou outra justificada), linkar no
`README.md`, e fazer uma EDA simples (distribuição do target, valores nulos, correlações).

**O que existe:**

- Base escolhida: [Santander Product Recommendation](https://www.kaggle.com/competitions/santander-product-recommendation)
  — fora da lista sugerida no PDF, justificada e documentada em `data/kaggle/README.md`
  (fonte, licença, arquivos esperados) e `data/README.md` (cadeia completa de transformação,
  o que é versionado e por quê).
- EDA: `notebooks/eda_golden_set.ipynb`, reprodutível via `make notebooks`.

**Gap:** o link da base Kaggle está em `data/kaggle/README.md`, não no `README.md` raiz como
o edital pede explicitamente. Ver pendências no fim deste documento.

## Etapa 2 — Preparação da base

**Pedido:** features e um target claro, prontos para o modelo (limpeza, encoding,
tratamento de nulos).

**O que existe:** pipeline de dados em 4 camadas, documentada em `data/README.md`:

```
data/kaggle/ (bruto, não versionado)
  → scripts/prepare_data.py       → data/processed/ (~1,36M linhas × 46 col, sem vazamento temporal)
  → scripts/generate_synthetic_br.py → data/synthetic_enrichment/ (colunas PT-BR)
  → scripts/generate_golden_sample.py → data/golden_set/golden_clients.csv (2.595 clientes, 48 col) +
                                          offer_catalog.json (10 braços/target)
```

Target = clique (`reward = click`, binário), documentado em `offer_catalog.json.catalog_metadata.reward_definition`.
Todas as etapas usam `random_state=42` (determinístico). Dicionário completo de colunas em
`data/golden_set/README.md`.

## Etapa 3 — Baseline e estratégia algorítmica adaptativa

**Pedido:** um baseline simples (regra fixa ou modelo simples) e uma estratégia adaptativa
(bandit/RL), com a métrica do adaptativo superando o baseline.

**O que existe:**

- `model_service/models/baseline.py` — regra determinística (maior receita esperada elegível).
- `model_service/models/thompson.py` — Thompson Sampling (Beta).
- `model_service/models/linucb.py` — LinUCB contextual (produção, atualização via Sherman-Morrison).
- Comparação em `notebooks/mab_exploracao_algoritmos.ipynb`: simulação de cada política por
  `N_ROUNDS`, repetida em `N_RUNS` sementes independentes (mesma semente em todas as políticas
  por rodada — comparação pareada), medindo reward acumulado, regret, conversão e receita simulada.
  **Conclusão do notebook:** "O LinUCB vence tanto o baseline determinístico quanto o Thompson
  global, em reward *e* em receita — porque é o único que usa o contexto do cliente *e* se
  adapta sozinho."
- A política ativa em produção é o LinUCB (`data/golden_set/offer_catalog.json`, v2.0.0-prod);
  `baseline` e `thompson` continuam registrados como políticas `shadow` para comparação
  (`docs/backend-roadmap.md`, E5).

## Etapa 4 — Avaliação e casos de teste

**Pedido:** métrica de avaliação (ex.: CTR simulado) e um golden set simplificado de ~5
clientes mostrando que a recomendação faz sentido.

**O que existe (excede o pedido):**

- Harness de avaliação offline: `model_service/evaluation/harness.py`, rodado por
  `scripts/run_evaluation.py` (`make evaluate`), sem precisar de Docker/banco.
- Golden set de teste: `data/golden_set/evaluation_cases.jsonl` — **24 casos** (não apenas 5),
  gerados por `scripts/generate_evaluation_cases.py`, cobrindo 3 tipos:
  - `edge` (8 casos): braço inelegível pelo catálogo não pode aparecer no ranking — bloqueante.
  - `adversarial` (8 casos): variar atributo protegido não muda o que o cliente pode receber — bloqueante.
  - `typical` (8 casos): conformidade com a regra do baseline — não bloqueante (divergir é aprender).
  Cada caso tem `rationale` (por que aquele resultado é esperado).
- Relatório gerado: `reports/evaluation-report.md` — as 3 políticas (`baseline`, `thompson`,
  `linucb`) passam 100% das propriedades bloqueantes (`edge`+`adversarial`); `linucb` diverge do
  baseline em 8/8 casos `typical`, o que é esperado (é onde ele aprendeu algo além da regra fixa).
- Demonstração interativa de recomendações individuais (o que a "vitrine" mostraria a um
  cliente): `notebooks/simulacao_portal_linucb.ipynb` — dois usuários com perfis distintos,
  ranking de ofertas explicado, sessão de clique→atualização→nova recomendação.

**Nota:** é um relatório de **propriedades** (elegibilidade, invariância de atributo protegido),
não de performance (regret/conversão/lift) — essas métricas dependem de tráfego real e vêm do
monitoramento (`GET /api/v1/monitoring/metrics`), não do golden set estático. Ver
`reports/README.md` para o motivo.

## Etapa 5 — Serviço ou interface demonstrável

**Pedido:** uma API simples (ex.: Flask/FastAPI) que receba dados de um cliente e retorne a
oferta recomendada — ou uma interface simples equivalente.

**O que existe (excede o pedido):**

- `model_service` (FastAPI): `POST /api/v1/rank` — ranqueia as ofertas elegíveis para um
  cliente/contexto (`model_service/api/v1/endpoints/rank.py`); `POST /api/v1/update` aplica o
  feedback (clique) e persiste o novo estado — loop de aprendizado compute-on-read.
- `api_service` (FastAPI): `POST /api/v1/decide` (decisão + log auditável), `GET /api/v1/me/recommendations`
  (sugestão escopada no usuário logado), consumido pelo `front_service` (Angular) — vitrine real
  com cadastro, recomendação ao vivo e feedback de clique.
- Reprodução ponta a ponta sem UI: `python scripts/reproduce.py` — migração → seed → ciclo
  decisão/feedback/reward, imprimindo o braço escolhido, reason codes, versão da política e
  `decision_id`.

## Etapa 6 — Arquitetura-alvo em nuvem

**Pedido:** um parágrafo no `README.md` listando os serviços de nuvem (ex.: AWS) que seriam
usados em produção e por quê.

**O que existe (excede o pedido):**

- `README.md` (seção "Tecnologias de nuvem"): parágrafo com o alvo de produção — **Amazon EKS**
  atrás de ALB, **RDS Multi-AZ**, **ElastiCache**, **S3**, **CloudWatch**/**X-Ray**.
- Detalhamento componente a componente com as lacunas atuais (sem pipeline de imagem/deploy,
  sem secrets gerenciados): `docs/architecture/cloud-resources.md` — inclui uma tabela
  componente-hoje → recurso AWS-alvo → motivo, e uma nota explícita recomendando **não
  provisionar** OpenSearch/Neo4j (resquício de um desenho anterior, sem consumidor no código).

## Etapa 7 — Ciclo de vida do modelo (MLOps)

**Pedido:** registrar parâmetros e métricas do experimento em uma ferramenta simples (ex.:
MLflow).

**O que existe:**

- `model_service/registry/mlflow_registry.py` — `ModelRegistry` (CRUD de modelos registrados no
  MLflow Model Registry): `register_version` loga parâmetros (`algorithm`, `n_arms`, `d`) e
  versiona o estado do bandit como artefato pyfunc (reprodutível, sem pickle frágil).
- `mlflow` como serviço no `docker-compose.yml` (porta 5000), backend store hoje em SQLite
  (documentado como limitação em `docs/architecture/cloud-resources.md`, seção MLflow).
- Métricas de produção (não as do notebook, mas do tráfego real — mais forte que o pedido):
  `GET /api/v1/monitoring/metrics` apura conversão, reward, regret e PSI (drift);
  `scripts/retrain_cycle.py --publish` anexa essas métricas a um ciclo de retreino
  (`candidate`) que o `model_service` registra, versionando o snapshot no MLflow
  (`registry_version` do ciclo).
- Governança em torno do registro: promoção de política exige **approval gate humano**
  (`POST /api/v1/approvals`, `model_service/governance/service.py`), com rollback auditável —
  ver `docs/domain-model.md` e `docs/backend-roadmap.md` (E10).

## Etapa 8 — Apresentação final (Demo Day)

**Pedido:** vídeo pitch de até 5 minutos explicando o problema, o modelo usado e mostrando a
Etapa 5 rodando.

**Status: não feito.** Não há vídeo, link ou menção a um pitch gravado em nenhum lugar do
repositório. Todo o conteúdo que o vídeo precisaria demonstrar já existe e funciona (ver Etapas
0, 3 e 5 acima) — falta gravar e publicar o link. Issue aberta abaixo.

---

## Requisito transversal — dados sensíveis e LGPD

**Pedido (seção "Dados, regras e bases Kaggle" do edital):** não usar dados reais de clientes,
identificadores, patrimônio, renda, gênero, raça ou regras comerciais privadas; manter decisões
sensíveis com humano no loop; documentar base legal, finalidade, minimização e retenção.

**O que existe:**

- Nenhum dado real de cliente: a base é a competição pública do Kaggle (Santander) mais uma
  camada de enriquecimento sintético brasileiro (`scripts/generate_synthetic_br.py`) — sem PII
  real em nenhuma camada (`data/README.md`).
- Humano no loop: **atendido** — approval gate humano bloqueia a promoção de qualquer política
  nova (`POST /api/v1/approvals`, `model_service/governance/service.py`,
  `api_service/services/bandit/governance/service.py`), com rollback auditável.

**Gap:** não há, em nenhum lugar do repositório, documentação de **base legal, finalidade,
minimização e retenção** dos dados usados pelo modelo — `docs/governance/` existe como pasta
reservada mas só tem um `.gitkeep`. Issue aberta abaixo.

---

## Pendências (requisitos não atendidos)

As issues abaixo foram redigidas para os três gaps identificados acima. **A criação delas via
API do GitHub falhou** — o repositório `andrevberaldo/datathon-7mlet-grupo-68` está com **Issues
desabilitadas** (erro `410` da API) e esta sessão não tem permissão de escrita no repositório
base (`JabuS2/datathon-7mlet-grupo-68`) para abri-las lá. Texto pronto para abertura manual (ou
após habilitar Issues no fork):

1. **README.md raiz sem visão do problema, link da base Kaggle e instruções de execução**
   (Etapas 0 e 1) — ver detalhe acima.
2. **Vídeo pitch do Demo Day (até 5 min) não gravado/publicado** (Etapa 8) — ver detalhe acima.
3. **Documentar base legal, finalidade, minimização e retenção (LGPD) dos dados sensíveis**
   (seção "Dados, regras e bases Kaggle") — ver detalhe acima.
