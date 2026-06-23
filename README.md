# WhatsApp AI Bot

Agente conversacional de IA integrado ao **WhatsApp** via **Evolution API**, com **RAG** (Retrieval-Augmented Generation) sobre uma base de conhecimento local (PDF/TXT) e **memória de conversa** persistente por usuário.

---

## Stack

- **Linguagem:** Python 3.13
- **HTTP Server:** FastAPI + Uvicorn
- **LLM / Embeddings:** Google Gemini (`langchain-google-genai`)
- **Orquestração RAG:** LangChain (history-aware retriever + stuff documents chain)
- **Vector Store:** ChromaDB (persistido em disco)
- **Buffer / Histórico:** Redis
- **WhatsApp Gateway:** Evolution API
- **Banco de apoio (Evolution):** PostgreSQL
- **Containerização:** Docker + Docker Compose

---

## Arquitetura

```
WhatsApp ─▶ Evolution API ─▶ POST /webhook (FastAPI)
                                   │
                                   ▼
                          message_buffer.py
                          (Redis list + debounce)
                                   │
                                   ▼
                  conversational_rag_chain (LangChain)
                          │                │
                          ▼                ▼
               RedisChatMessageHistory  ChromaDB (Gemini embeddings)
                          │                │
                          └─────► Gemini LLM ◄────┘
                                   │
                                   ▼
                  Evolution API ─▶ WhatsApp (resposta)
```

### Containers (`docker-compose.yml`)

| Serviço      | Imagem                              | Porta        | Função                              |
| ------------ | ----------------------------------- | ------------ | ----------------------------------- |
| `bot`        | build local (Dockerfile)            | `8002:8000`  | API FastAPI do agente               |
| `evolution-api` | `evoapicloud/evolution-api:latest` | `8080:8080`  | Gateway WhatsApp                    |
| `postgres`   | `postgres:15`                       | `5432:5432`  | Persistência da Evolution           |
| `redis`      | `redis:latest`                      | `6379:6379`  | Buffer, debounce e histórico de chat |

**Volumes / bind mounts**

- `evolution_instances` → instâncias da Evolution
- `postgres_data` → dados do Postgres
- `redis` → dados do Redis
- `vectorstore_data` (`./vectorstore_data:/app/vectorstore`) → índice Chroma persistido
- `rag_files` (`./rag_files:/app/rag_files`) → PDFs/TXTs da base de conhecimento

**Ordem de dependência:** `postgres` e `redis` → `evolution-api` → `bot`.

---

## Fluxo de Mensagens

1. O WhatsApp entrega a mensagem à **Evolution API**.
2. A Evolution faz `POST /webhook` no `bot` (`app.py`).
3. `app.py` extrai `remoteJid` (chat_id) e o texto da `conversation`, filtra grupos (`@g.us`) e chama `buffer_message()`.
4. `message_buffer.py` empilha a mensagem em uma **lista Redis** (`chat_id + BUFFER_KEY_SUFIX`) via `RPUSH` e aplica `EXPIRE` (TTL).
5. Cada nova mensagem **cancela e recria** um `asyncio.Task` de debounce por `chat_id`.
6. Após `DEBAUNCE_SECONDS` (10s) sem novas mensagens, a task lê o buffer, concatena tudo e invoca a `conversational_rag_chain` passando `session_id = chat_id`.
7. O histórico é gerenciado por `RedisChatMessageHistory` (`memory.py`).
8. O retriever faz busca semântica no **ChromaDB** (embeddings Gemini).
9. O LLM Gemini responde com base no prompt de sistema + contexto RAG + histórico.
10. A resposta é enviada ao WhatsApp via `evolution_api.send_whatsapp_message()`.
11. O buffer é limpo com `DEL`.

---

## Estrutura do Projeto

```
.
├── app.py                # Entrypoint FastAPI (webhook)
├── config.py             # Carrega .env e expõe constantes
├── message_buffer.py     # Debounce + envio da resposta
├── chains.py             # Pipeline RAG conversacional
├── memory.py             # Histórico de chat no Redis
├── vectorstore.py        # Ingestão + retriever Chroma
├── prompts.py            # Templates de prompt
├── evolution_api.py      # Cliente HTTP da Evolution API
├── requirements.txt      # Dependências Python
├── Dockerfile            # Imagem do bot
├── docker-compose.yml    # Stack completa
├── .env                  # Variáveis de ambiente (NÃO commitar)
├── rag_files/            # PDFs/TXTs a serem indexados
└── vectorstore_data/     # Persistência do Chroma
```

### Descrição dos módulos

- **`app.py`** — Cria `FastAPI()` e expõe `POST /webhook`. Lê JSON, valida que não é grupo e delega para `buffer_message()`.
- **`config.py`** — Carrega `.env` com `python-dotenv` e expõe constantes (chaves Gemini, prompts, paths, credenciais Evolution, URL Redis e parâmetros de buffer).
- **`message_buffer.py`** — Cliente `redis.asyncio` global, instancia a chain RAG uma única vez, mantém `defaultdict(asyncio.Task)` com uma task de debounce por chat. `buffer_message()` faz push + cancela task anterior + cria nova; `handle_debounce()` aplica sleep, concatena mensagens, invoca a chain, envia ao WhatsApp e limpa o buffer.
- **`chains.py`** — `get_rag_chain()` monta LLM Gemini + retriever Chroma + `create_history_aware_retriever` + `create_stuff_documents_chain` → `create_retrieval_chain`. `get_conversational_rag_chain()` embrulha com `RunnableWithMessageHistory` por `session_id`.
- **`memory.py`** — `get_session_history(session_id)` retorna `RedisChatMessageHistory` apontando para o mesmo Redis (DB 6).
- **`vectorstore.py`** — `load_documents()` lê todos `.pdf` (PyPDFLoader) e `.txt` (TextLoader) de `rag_files/` e move os processados para `rag_files/processed/`. `get_vectorstore()` faz chunking (`RecursiveCharacterTextSplitter`, `chunk_size=3000`, `overlap=300`), gera embeddings com `gemini-embedding-2` e persiste no Chroma. Se não há documentos novos, apenas carrega o store existente.
- **`prompts.py`** — `contextualize_prompt` (reformula a pergunta usando o histórico, sem responder) e `qa_prompt` (usa `AI_SYSTEM_PROMPT` com placeholder `{context}`, injeta `chat_history` e a pergunta atual).
- **`evolution_api.py`** — `send_whatsapp_message(number, text)` faz `POST` em `{EVOLUTION_API_URL}/message/sendText/{EVOLUTION_INSTANCE_NAME}` com header `apikey`.

---

## Variáveis de Ambiente (`.env`)

| Variável                       | Exemplo                              | Uso                                  |
| ------------------------------ | ------------------------------------ | ------------------------------------ |
| `GEMINI_API_KEY`               | `AQ.Ab8R…`                           | Auth Google Gemini                   |
| `GEMINI_MODEL_NAME`            | `gemini-2.5-flash`                   | Modelo LLM                           |
| `GEMINI_MODEL_TEMPERATURE`     | `0`                                  | Criatividade                         |
| `AI_CONTEXTUALIZE_PROMPT`      | prompt PT-BR                         | Reformulação de pergunta             |
| `AI_SYSTEM_PROMPT`             | prompt PT-BR c/ `{context}`          | Resposta final                       |
| `VECTOR_STORE_PATH`            | `vectorstore`                        | Persistência Chroma                  |
| `RAG_FILES_DIR`                | `rag_files`                          | Pasta de documentos                  |
| `EVOLUTION_API_URL`            | `http://evolution-api:8080`          | URL do gateway                       |
| `EVOLUTION_INSTANCE_NAME`      | `GawSystems`                         | Instância WhatsApp                   |
| `AUTHENTICATION_API_KEY`       | `@Guilherme8`                        | apikey da Evolution                  |
| `CACHE_REDIS_URI`              | `redis://redis:6379/6`               | Conexão Redis (DB 6)                 |
| `BUFFER_KEY_SUFIX`             | `_msg_buffer`                        | Sufixo da chave do buffer            |
| `DEBAUNCE_SECONDS`             | `10`                                 | Tempo de espera do debounce          |
| `BUFFER_TTL`                   | `300`                                | TTL da chave no Redis (segundos)     |
| `DATABASE_*`                   | URI Postgres                         | Configuração da Evolution            |
| `CACHE_*`                      | flags                                | Configuração de cache da Evolution   |

> O `.env` real está no `.gitignore`. **Nunca** comite credenciais.

---

## Dependências (`requirements.txt`)

**Frameworks & servidor**

- `fastapi`, `uvicorn`, `starlette`, `httpx`, `requests`, `websockets`, `websocket-client`

**LangChain / orquestração**

- `langchain`, `langchain-classic`, `langchain-core`, `langchain-community`, `langchain-chroma`, `langchain-text-splitters`
- `langchain-google-genai`, `google-genai`
- `langgraph`, `langgraph-checkpoint`, `langgraph-prebuilt`, `langgraph-sdk`
- `langsmith`, `langchain-protocol`

**LLM / Embeddings / Vetor**

- `google-auth`, `googleapis-common-protos`, `grpcio`, `protobuf`
- `chromadb`, `onnxruntime` (embedding default do Chroma)
- `numpy`, `tokenizers`, `huggingface_hub`

**Redis / persistência**

- `redis`, `SQLAlchemy`

**Validação / configuração**

- `pydantic`, `pydantic-settings`, `pydantic_core`, `python-dotenv`, `PyYAML`

**Documentos / texto**

- `pypdf` (carrega PDFs), `filetype`

**Telemetria / observabilidade**

- `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-grpc`, `opentelemetry-exporter-otlp-proto-common`, `opentelemetry-proto`, `opentelemetry-semantic-conventions`

**Utilitários**

- `tenacity` (retries), `tqdm`, `rich`, `typer`, `shellingham`
- `orjson`, `ormsgpack`, `zstandard`, `xxhash` — serialização/compactação
- `bcrypt`, `cryptography` — segurança
- `kubernetes` — client K8s (puxado por dependência transitiva)
- Demais: `aiohttp`, `aiosignal`, `anyio`, `attrs`, `multidict`, `yarl`, `frozenlist`, `propcache`, `fsspec`, `referencing`, `rpds-py`, `jsonschema`, `packaging`, entre outras.

> Versões completas estão fixadas em `requirements.txt`.

---

## Endpoints

| Método | Path        | Origem   | Descrição                                          |
| ------ | ----------- | -------- | -------------------------------------------------- |
| `POST` | `/webhook`  | `app.py` | Recebe eventos da Evolution API e dispara o pipeline |

---

## Como Rodar

### 1. Pré-requisitos

- Docker + Docker Compose
- Chave de API do Google Gemini
- Instância configurada na Evolution API (criar via UI/manager da Evolution)

### 2. Configurar o `.env`

Crie um arquivo `.env` na raiz com base na tabela de variáveis acima. Defina pelo menos:

```env
GEMINI_API_KEY=sua_chave_gemini
GEMINI_MODEL_NAME=gemini-2.5-flash
GEMINI_MODEL_TEMPERATURE=0

EVOLUTION_API_URL=http://evolution-api:8080
EVOLUTION_INSTANCE_NAME=sua_instancia
AUTHENTICATION_API_KEY=sua_apikey_evolution

CACHE_REDIS_URI=redis://redis:6379/6

VECTOR_STORE_PATH=vectorstore
RAG_FILES_DIR=rag_files

BUFFER_KEY_SUFIX=_msg_buffer
DEBAUNCE_SECONDS=10
BUFFER_TTL=300
```

### 3. Subir a stack

```bash
docker compose up -d --build
```

### 4. Acompanhar logs

```bash
docker logs -f bot
```

### 5. Configurar o webhook na Evolution

Apontar o webhook da instância para `http://bot:8000/webhook` (rede interna do Compose) ou `http://host:8002/webhook` (acesso externo).

### 6. Adicionar nova base de conhecimento

Copie arquivos `.pdf` ou `.txt` para `./rag_files/` e reinicie o container `bot`:

```bash
docker restart bot
```

Os arquivos processados são movidos automaticamente para `./rag_files/processed/`.

---

## Estrutura do Prompt

**`AI_CONTEXTUALIZE_PROMPT`** (reformulação)

> Dado um histórico de conversa e a pergunta mais recente do usuário, que pode fazer referência ao contexto anterior, formule uma pergunta independente, que possa ser compreendida sem o histórico da conversa. **NÃO** responda à pergunta — apenas reformule se necessário; caso contrário, retorne a pergunta como está.

**`AI_SYSTEM_PROMPT`** (resposta)

> Você é um assistente virtual que irá responder dúvidas dos clientes. Use os seguintes trechos de contexto recuperado para responder à pergunta. Se você não souber a resposta, diga que não sabe. Use no máximo três frases e mantenha a resposta concisa. `{context}`

---

## Pontos de Atenção / Melhorias Futuras

1. **Segurança do `.env`** — chave Gemini e apikey da Evolution em texto plano; considerar Docker secrets ou vault.
2. **Tamanho de chunk 3000** é alto para `gemini-embedding-2`; avaliar `chunk_size` 500–1000 com overlap proporcional.
3. **Reindexação a cada request** — `get_vectorstore()` recria embeddings quando há documentos novos; ideal é um job único de ingestão.
4. **Validação do webhook** é mínima — considerar validar assinatura/secret da Evolution.
5. **Logs estruturados** (atualmente `print`); integrar com OpenTelemetry já presente nas dependências.
6. **Tratamento de erros** em `send_whatsapp_message` (sem retry, sem checagem de status HTTP).
7. **Encoding do `.env`** — normalizar valores para evitar aspas literais.

---

## Licença

Defina a licença do projeto conforme a estratégia do repositório (MIT, Apache-2.0, proprietária etc.).
