# MLOps Challenge — Neural Machine Translation (EN → PT)

Pipeline end-to-end de MLOps para modelo de tradução automática Inglês → Português, incluindo automação, orquestração, observabilidade e controle de acesso.

## 📋 Arquitetura

```
┌─────────────────────────────────────────────────────────────────────┐
│                         n8n (Orquestrador)                         │
│                        localhost:5678                               │
│                                                                     │
│  Webhook ──► Preparar Dados ──► Treinar ──► Validar ──► Publicar ──► Deploy │
└─────────────────────────────────────────────────────────────────────┘
                                                          │
                                                          ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────────────┐
│   Cliente    │─────►│   Gateway    │─────►│    Inference API     │
│              │      │ NGINX (:8080)│      │  FastAPI (:8000)     │
│              │      │ • API Key    │      │  • /predict          │
│              │      │ • Rate Limit │      │  • /health           │
│              │      │ • Logging    │      │  • /metrics          │
└──────────────┘      └──────────────┘      │  • /model            │
                                            │  • /reload           │
                                            └──────────┬───────────┘
                                                       │
                              ┌─────────────────────────┤
                              ▼                         ▼
                    ┌──────────────┐          ┌──────────────┐
                    │  Prometheus  │─────────►│   Grafana    │
                    │    (:9090)   │          │   (:3000)    │
                    │  Coleta cada │          │  Dashboards  │
                    │    15s       │          │              │
                    └──────────────┘          └──────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      CI/CD (GitHub Actions)                        │
│  Push/PR ──► Lint (ruff) ──► Test (pytest) ──► Build ──► Publish  │
│                                                  (GHCR)           │
└─────────────────────────────────────────────────────────────────────┘
```

## 🛠️ Pré-requisitos

- Docker e Docker Compose
- Git

## 🚀 Como Executar

### 1. Clonar o repositório

```bash
git clone https://github.com/codeonthespectrum/mlops-challenge-handtalk.git
cd mlops-challenge-handtalk
```

### 2. Preparar o Dataset

```bash
docker compose --profile prepare up --build
```

Gera os TFRecords em `data/processed/`. Variáveis opcionais: `TRAIN_RECORDS`, `VAL_RECORDS`, `MAX_TOKENS`, `SEED`.

### 3. Treinar o Modelo

```bash
docker compose --profile train up
```

Treina o Transformer e exporta o SavedModel em `artifacts/<run_id>/`. Variáveis opcionais: `EPOCHS`, `BATCH_SIZE`, `THRESHOLD`.

### 4. Subir a API de Inferência

```bash
DEFAULT_RUN_ID=<run_id> docker compose --profile api up
```

API disponível em `http://localhost:8000`. Substitua `<run_id>` pelo ID gerado no treino.

### 5. Subir o API Gateway

```bash
docker compose --profile gateway --profile api up
```

Gateway disponível em `http://localhost:8080`. Requer header `X-Api-Key: handtalk-secret-key-2026`.

Exemplo de uso:

```bash
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: handtalk-secret-key-2026" \
  -d '{"text": "Hello, how are you?"}'
```

### 6. Subir Observabilidade (Prometheus + Grafana)

```bash
docker compose --profile monitoring --profile api up
```

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000` (login padrão: admin/admin)

### 7. Subir o Orquestrador n8n

```bash
docker compose --profile n8n up
```

n8n disponível em `http://localhost:5678`. O workflow pode ser importado a partir do arquivo `n8n_workflow.json`.

### 8. Executar Testes

```bash
docker compose --profile tests up
```

## 📦 Estrutura do Projeto

```
mlops-challenge-handtalk/
├── .github/workflows/
│   └── ci.yml                  # Pipeline CI/CD (GitHub Actions)
├── ml/                         # Pipeline de ML
│   ├── prepare_dataset.py      # Preparação do dataset
│   ├── train.py                # Treinamento do modelo
│   ├── model.py                # Arquitetura Transformer
│   ├── tokenizers.py           # Tokenizers
│   └── common.py               # Utilitários
├── inference_api/              # API de inferência (FastAPI)
│   ├── main.py                 # Endpoints
│   ├── model_manager.py        # Gerenciamento de modelos
│   ├── schemas.py              # Schemas Pydantic
│   ├── metrics.py              # Métricas da aplicação
│   └── logging_config.py       # Logging estruturado
├── gateway/                    # API Gateway (NGINX)
│   ├── Dockerfile              # Imagem do gateway
│   └── nginx.conf              # Configuração NGINX
├── monitoring/                 # Observabilidade
│   └── prometheus.yml          # Configuração do Prometheus
├── tests/                      # Testes automatizados
│   └── test_api_contract.py    # Testes de contrato
├── docker-compose.yml          # Orquestração de todos os serviços
├── Dockerfile                  # Imagem base do projeto ML
├── requirements.txt            # Dependências Python
├── n8n_workflow.json           # Workflow exportado do n8n
└── CHALLENGE.md                # Especificação do desafio
```

## 🔐 API Gateway

O gateway NGINX implementa:

- **Autenticação**: Requer header `X-Api-Key` com chave válida
- **Rate Limiting**: 10 requisições/segundo por chave, com burst de 20
- **Logging**: Registra IP, API key, método, status e tempo de resposta

Respostas de erro:
- `401` — API key não fornecida
- `403` — API key inválida
- `429` — Rate limit excedido

## 🔄 CI/CD

O pipeline GitHub Actions (`.github/workflows/ci.yml`) executa em todo push/PR na branch `main`:

| Stage   | Descrição                                      |
|---------|-------------------------------------------------|
| Lint    | Verificação de qualidade com `ruff`             |
| Test    | Execução dos testes com `pytest`                |
| Build   | Build da imagem Docker                          |
| Publish | Push da imagem para GitHub Container Registry   |

Cada stage depende da anterior — se o lint falhar, os testes não rodam; se os testes falharem, o build não acontece.

## 📊 Observabilidade

| Pilar        | Ferramenta       | Descrição                                  |
|-------------|------------------|--------------------------------------------|
| Métricas    | Prometheus       | Coleta `/metrics` da API a cada 15s        |
| Dashboards  | Grafana          | Visualização das métricas coletadas        |
| Logs        | Docker Compose   | Logs estruturados (JSON) da API e pipeline |
| Health Check| API `/health`    | Status do serviço e modelo carregado       |

## 🎵 Orquestração (n8n)

O workflow n8n orquestra o pipeline completo via webhook:

```
Webhook (POST) → Preparar Dados → Treinar → Validar → Publicar → Deploy
```

- Disparado via HTTP POST no webhook do n8n
- Cada etapa executa o container correspondente
- A etapa de validação bloqueia o deploy se o modelo não atingir o threshold de qualidade
- O `run_id` é passado entre etapas para rastreabilidade

## 📝 Decisões Técnicas

- **NGINX como gateway**: Escolhido pela simplicidade de configuração e ampla adoção na indústria. Implementa autenticação, rate limiting e logging sem dependências extras.
- **Prometheus + Grafana**: Stack padrão de observabilidade, se integra nativamente com endpoints `/metrics` já existentes na API.
- **n8n como orquestrador**: Ferramenta visual que permite montar e modificar pipelines sem alterar código, alinhada com a proposta do desafio.
- **GitHub Actions para CI/CD**: Integração nativa com o repositório GitHub, sem necessidade de infraestrutura adicional.