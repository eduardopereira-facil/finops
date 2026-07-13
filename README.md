# AI Gateway

Camada centralizadora de acesso a LLMs utilizadas no Grupo Facil com controle de custos, chaves virtuais e observabilidade — resolvendo o problema de **LLM Sprawl** na empresa.

Todos os agentes, scripts e sistemas internos passam a falar com este Gateway em vez de chamar provedores diretamente. Ele guarda as chaves reais, emite chaves virtuais com orçamento (`budget`) por projeto/cliente, e registra automaticamente custo, latência e uso em uma camada de observabilidade — sem expor o conteúdo de prompts sensíveis.

## Stack

| Componente | Função |
|---|---|
| [LiteLLM Proxy](https://github.com/BerriAI/litellm) | Gateway — roteia chamadas, gerencia chaves virtuais e budgets |
| [Langfuse](https://github.com/langfuse/langfuse) | Observabilidade — custo por prompt, latência, auditoria |
| PostgreSQL | Persistência de chaves, budgets e configurações do Proxy |
| Docker Compose | Orquestração dos serviços |

## Pré-requisitos

- [Docker](https://www.docker.com/) instalado e rodando
- Uma instância do [Langfuse](https://github.com/langfuse/langfuse) já em execução (local ou remota), com um projeto criado e as API keys geradas
- Chaves de API dos provedores de modelo que forem usados 

## Configuração

1. Clone o repositório:

   ```bash
   git clone https://github.com/eduardopereira-facil/finops.git
   cd finops
   ```

2. Copie o arquivo de exemplo e preencha com suas credenciais:

   ```bash
   cp .env.example .env
   ```

   | Variável | Descrição |
   |---|---|
   | `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` | Chaves geradas no projeto do Langfuse |
   | `LANGFUSE_HOST` | URL do Langfuse (nome do container, se estiver na mesma rede Docker) |
   | `LITELLM_MASTER_KEY` | Senha de administrador do Proxy — defina uma forte |
   | `DATABASE_URL` | String de conexão do PostgreSQL do LiteLLM |
   | `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` | Chaves reais dos provedores de modelo |

   > **Nunca** commite o `.env` — ele já está no `.gitignore`.

3. Se o Langfuse estiver rodando em outra stack Docker Compose, garanta que a network dela está marcada como `external` no `docker-compose.yml` deste projeto, apontando para o nome correto (ex: `langfuse_default`).

## Rodando

```bash
docker compose up -d
```

Acompanhe os logs do Proxy até ver `Uvicorn running on http://0.0.0.0:4000`:

```bash
docker compose logs -f litellm-proxy
```

Valide se subiu corretamente:

```bash
curl http://localhost:4000/health
```

## Uso

### Gerar uma chave virtual com orçamento

```bash
curl -X POST http://localhost:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "max_budget": 500,
    "budget_duration": "30d",
    "metadata": {
      "projeto": "nome-do-projeto",
      "cliente": "interno",
      "ambiente": "producao"
    },
    "models": ["gemini-3.5-flash", "gemini-3.1-pro"]
  }'
```

### Chamar o Gateway a partir de uma aplicação

```python

client = BestSenior(
    api_key="sk-virtual-...",       # chave virtual gerada acima
    base_url="http://localhost:4000"
)

response = client.chat.completions.create(
    model="gemini-flash-lite",
    messages=[{"role": "user", "content": "Olá"}],
    extra_headers={
        "x-projeto": "nome-do-projeto",
        "x-cliente": "interno",
        "x-ambiente": "producao"
    }
)
```

Modelos disponíveis, budgets e regras de roteamento são definidos em [`config.yaml`](./config.yaml).

## Observabilidade e privacidade

Toda chamada é automaticamente reportada ao Langfuse (custo, latência, tokens, tags). O conteúdo de prompts e respostas **não é registrado** (`turn_off_message_logging: true` em `config.yaml`), para evitar exposição de dados sensíveis em ambientes que lidam com informações de saúde ou outros dados pessoais.

## Estrutura do projeto

```
.
├── docker-compose.yml   # Orquestração do Proxy + PostgreSQL
├── config.yaml           # Modelos, budgets e regras do LiteLLM
├── .env.example           # Template de variáveis de ambiente
└── .env                    # Credenciais reais (não versionado)
```
