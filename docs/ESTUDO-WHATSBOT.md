# Guia de Estudo — WhatsBot

> Documento de **onboarding**. Lê-se de cima para baixo, na ordem de aprendizado.
> Para a referência técnica densa (todas as tabelas, endpoints, gotchas), veja
> [CLAUDE.md](../CLAUDE.md) na raiz. Este guia explica o **porquê** e ensina o
> **fluxo de IA** em profundidade.

## Índice

1. [Objetivo — o que o WhatsBot é](#1-objetivo)
2. [Tecnologias — e por que cada uma](#2-tecnologias)
3. [Organização do código](#3-organização-do-código)
4. [Ciclo de vida da aplicação](#4-ciclo-de-vida-da-aplicação)
5. [DEEP-DIVE: Fluxo de IA e mensagens](#5-deep-dive-fluxo-de-ia-e-mensagens)
6. [Subsistemas — banco, plugins, frontend](#6-subsistemas)
7. [Roteiro de estudo](#7-roteiro-de-estudo)
8. [Como rodar localmente](#8-como-rodar-localmente)

---

## 1. Objetivo

O **WhatsBot** é um **bot de WhatsApp com Inteligência Artificial voltado para
usuários finais** (não desenvolvedores).

A ideia central:

- O usuário conecta seu número de WhatsApp ao app (escaneando um QR code).
- Quando alguém manda mensagem, um **LLM** lê, entende e **responde
  automaticamente**, como um atendente.
- A IA **aprende sobre cada contato** (nome, email, profissão, empresa) e guarda
  isso; sabe **transferir para um humano** quando necessário.
- Tudo é gerenciado por uma **interface web** (painel): contatos, conversas,
  custos de API, configuração.

Dois diferenciais de design importantes:

- **Distribuído como `.exe` único para Windows.** O usuário baixa, executa,
  escaneia o QR e está funcionando — sem instalar Python, banco, nada. O mesmo
  código também roda em Docker/servidor (Coolify) para deploys profissionais.
- **Sistema de plugins.** Dá para estender o comportamento do bot (novas
  capacidades, interceptadores, telas) **sem tocar no código principal**.

---

## 2. Tecnologias

Cada peça tem um motivo. Entender o "porquê" deixa a arquitetura coerente:

| Tecnologia | O que é | Por que está aqui |
|---|---|---|
| **Python 3.11+** | Linguagem principal | Ecossistema de IA maduro; fácil de empacotar em EXE |
| **FastAPI + uvicorn** | Framework web assíncrono + servidor ASGI | Serve a API REST, o WebSocket e o webhook. Async é essencial: o bot espera I/O o tempo todo (WhatsApp, LLM, banco) |
| **GOWA** (`go-whatsapp-web-multidevice` v8.5.0) | Programa em Go que fala o protocolo do WhatsApp | O WhatsApp não tem API oficial aberta para isso. O GOWA roda como **subprocesso** (`bin/gowa.exe`) e expõe uma API REST local em `localhost:3000`. É a "ponte" com o WhatsApp |
| **OpenRouter** | Provedor de LLM (API compatível com OpenAI) | Dá acesso a vários modelos por uma única API. O bot usa a SDK `openai` apontada para `https://openrouter.ai/api/v1` |
| **SQLAlchemy 2.0 Core + Alembic** | Camada de dados + migrações | "Core" = SQL explícito, **sem ORM mágico**. Permite o mesmo código rodar em SQLite **e** PostgreSQL |
| **SQLite** | Banco padrão (modo WAL) | Zero configuração — é só um arquivo. Perfeito para o EXE no PC do usuário |
| **PostgreSQL** | Banco opcional | Para quem roda em servidor com várias réplicas. Trocável pela própria interface |
| **Preact + HTM + Tailwind** | Frontend | Preact = "React minúsculo". HTM = JSX sem compilador. **Sem build step** — os `.js` são servidos como estão. Decisão deliberada: simplicidade e empacotamento fácil |
| **PyInstaller** | Empacotador | Transforma o projeto Python + binários no `.exe` final |

A espinha dorsal mental:

```
WhatsApp ⇄ GOWA ⇄ FastAPI (Python) ⇄ LLM (OpenRouter)
                      │
                      └─ SQLite/Postgres (persistência)  +  Frontend Preact (painel)
```

---

## 3. Organização do código

Módulos com responsabilidades bem separadas:

```
main.py            → ponto de entrada: arranca tudo
server/            → backend FastAPI (rotas, webhook, websocket, tarefas)
gowa/              → ponte com o WhatsApp (subprocesso + cliente HTTP)
agent/             → o "cérebro": LLM, tools, memória dos contatos
db/                → banco de dados (engine, tabelas, repositórios, migrações)
plugins/           → motor do sistema de plugins (core)
config/            → carregar/salvar configurações
web/               → frontend Preact (HTML + JS, sem build)
assets/            → recursos: plugins de exemplo, templates
storages/          → dados em runtime (banco, sessão WhatsApp, plugins instalados)
bin/gowa.exe       → binário da ponte WhatsApp (pré-compilado, não editar)
```

**Regra de ouro:** [plugins/](../plugins/) é o **motor**; `storages/plugins/` são
os **plugins instalados** pelo usuário. O core nunca é tocado para estender o bot.

Arquivos-chave por módulo:

- [main.py](../main.py) — boot.
- [server/app.py](../server/app.py) — monta a app FastAPI, `lifespan`, middleware.
- [server/routes/webhook.py](../server/routes/webhook.py) — recebe mensagens do WhatsApp (o arquivo mais importante do fluxo).
- [server/background.py](../server/background.py) — tarefas em segundo plano (GOWA, status, QR, avatares).
- [gowa/manager.py](../gowa/manager.py) — controla o subprocesso GOWA.
- [gowa/client.py](../gowa/client.py) — cliente HTTP para a API do GOWA.
- [agent/handler.py](../agent/handler.py) — o `AgentHandler`, núcleo da IA.
- [agent/memory.py](../agent/memory.py) — memória por contato.
- [agent/tools/](../agent/tools/) — tools que o LLM pode chamar.

---

## 4. Ciclo de vida da aplicação

O que acontece do boot até o bot estar pronto para receber mensagens:

```
T=0s   python main.py
       ├─ init_db()                → resolve URL, cria engine, roda migrações Alembic
       ├─ Settings().load()        → carrega config da tabela `config`
       ├─ configura logging        → stdout + logs/whatsbot.log
       ├─ instancia componentes    → GOWAManager, GOWAClient, AgentHandler
       ├─ create_app()             → monta FastAPI (rotas, plugins, middleware)
       └─ uvicorn.run(host=0.0.0.0, port=8080)

T=+0.5s (lifespan startup — server/app.py)
       ├─ start_gowa_task()        → sobe gowa.exe como subprocesso, cria "device"
       ├─ status_poll_loop()       → checa conexão a cada 5s
       ├─ qr_poll_loop()           → busca QR enquanto desconectado
       ├─ avatar_fetch_task()      → busca avatares dos contatos após conectar
       └─ emite eventos app.startup + plugin.loaded

T=+1.5s abre o navegador em http://127.0.0.1:8080 (se não for Docker)

T=+Ns  usuário escaneia o QR → WhatsApp autentica → bot PRONTO
```

Pontos para entender bem:

- O **GOWA demora ~5s** para aceitar conexões — por isso o polling de QR/status
  tolera falhas em silêncio.
- O GOWA v8.5.0 é **multi-device**: é obrigatório criar um "device" via
  `POST /devices` **antes** de qualquer outro endpoint.
- A configuração resolve em camadas: **variável de ambiente → tabela `config`
  (SQLite) → defaults** em [config/settings.py](../config/settings.py).

---

## 5. DEEP-DIVE: Fluxo de IA e mensagens

Esta é a seção principal. Vamos seguir uma mensagem do momento em que chega
no WhatsApp até a resposta da IA voltar para o contato.

### Visão geral do pipeline

```
WhatsApp
   │  (1) mensagem chega
   ▼
GOWA  ──POST──►  /api/webhook        [server/routes/webhook.py]
   │
   │  (2) detecta tipo de mídia, aplica filter.webhook.payload
   ▼
state.pending_messages[phone].append(...)   ← acumula
   │
   │  (3) _schedule_orchestrator(phone)
   ▼
_orchestrate(phone)                  ← espera o "digitando..." parar
   │                                   espera message_batch_delay (~3s)
   │                                   agrupa mensagens do mesmo contato
   ▼
_run_one_cycle()                     ← separa texto x mídia
   │
   ├─ (4) transcrição de áudio / descrição de imagem  (_maybe_transcribe)
   │
   ▼
AgentHandler.aprocess_message()      [agent/handler.py]
   │
   ├─ (5) monta system prompt dinâmico  (_build_system_prompt)
   ├─ (6) chama o LLM via OpenRouter
   ├─ (7) loop de tool calling          (_dispatch_tool)
   ├─ (8) follow-up call se o LLM só chamou tools
   ▼
ProcessResult(reply=...)
   │
   ▼
_send_reply(phone, reply)            ← (9) split em partes + delays + envio
   │
   ▼
GOWA  ──►  WhatsApp                  + broadcast WebSocket → frontend
```

### Etapa 1-2 — Recepção no webhook

Tudo entra por `POST /api/webhook` em
[server/routes/webhook.py](../server/routes/webhook.py). O GOWA envia um payload
JSON com campos como `body`, `from`, `id`, `is_from_me`, `timestamp`, e — para
mídia — campos `image`, `audio`, `video`, `document`, `location`, etc.

O webhook:

- Detecta o **tipo de mídia** (são ~14 tipos: `image`, `audio`, `video`,
  `sticker`, `document`, `location`, `poll`, `contact`, ...). Cada um vira um
  `media_type` + `media_path` + `media_extras`.
- Trata outros eventos sem resposta (reações, edições, acks de leitura,
  presença "digitando", entrada/saída de grupo, chamadas).
- Aplica o filtro de plugin `filter.webhook.payload` **antes de qualquer
  parse** — um plugin pode inspecionar ou abortar.

> **Importante:** o WhatsBot **não usa polling**. Toda recepção é via webhook.

### Etapa 3 — Batching (agrupamento)

Em vez de responder cada mensagem imediatamente, o bot **acumula** e **agrupa**.
Por quê? Pessoas mandam mensagem picada ("Oi" / "tudo bem?" / "queria saber..."),
e responder cada fragmento isolado seria ruim.

Como funciona:

1. Cada mensagem que chega é adicionada a `state.pending_messages[phone]`.
2. `_schedule_orchestrator(phone)` agenda (ou re-agenda) uma task assíncrona
   `_orchestrate(phone)`. Se já havia uma esperando, ela é cancelada e recriada
   — assim cada mensagem nova "estende" a janela.
3. `_orchestrate()`:
   - espera o contato **parar de digitar** (`_wait_typing_paused`, até 30s);
   - dorme `message_batch_delay` (padrão **3.0s**, configurável);
   - espera de novo (a digitação pode ter retomado);
   - tira um *snapshot* das mensagens pendentes e chama `_run_one_cycle()`.
4. `_run_one_cycle()` junta os textos com `\n` e processa.

O parâmetro `message_batch_delay` vive em
[config/settings.py](../config/settings.py) (override por env `WHATSBOT_BATCH_DELAY`).

> Detalhe de robustez: o envio (`SEND`) é marcado como atômico em
> `state.sending[phone]`. Se mensagens novas chegam **durante** o envio, um novo
> ciclo é agendado depois — nada é perdido.

### Etapa 4 — Transcrição de áudio e descrição de imagem

Texto puro vai direto para o LLM. Mídia precisa virar texto primeiro.

`_run_one_cycle` separa **texto** de **mídia**. Para cada item de mídia chama
`_maybe_transcribe(media_kind, path, ...)` no webhook, que:

1. **Checa a config** — `audio_transcription_mode` (`received`/`sent`/`both`) e
   `image_transcription_enabled`. Se desligado, retorna `""`.
2. Aplica o filtro `filter.transcription.should_run` — um plugin pode abortar
   (ex.: não transcrever áudios de um grupo específico).
3. Chama o `AgentHandler`:
   - **áudio** → `transcribe_audio()` ([agent/handler.py:315](../agent/handler.py#L315)):
     lê o arquivo, codifica em base64, manda para o `audio_model` com o prompt
     "Transcreva este áudio fielmente em português".
   - **imagem** → `describe_image()` ([agent/handler.py:372](../agent/handler.py#L372)):
     manda a imagem como `data:` URL para o `image_model` com "Descreva
     detalhadamente o conteúdo desta imagem".
4. Aplica `filter.transcription.result` — um plugin pode reescrever o texto.

O resultado é combinado com o texto da mensagem (imagens entram com um prefixo
tipo `[Descrição da imagem]: ...`) antes de ir para o LLM principal.

### Etapa 5 — Montagem do system prompt dinâmico

Aqui o bot vira "esperto". `_build_system_prompt()`
([agent/handler.py:425](../agent/handler.py#L425)) monta o prompt em camadas,
partindo do `system_prompt` base configurado pelo usuário e **anexando**:

- **Contexto de grupo** — se o contato é um grupo, explica o formato
  `[Nome]: mensagem`.
- **Informações conhecidas do contato** — nome, email, profissão, empresa,
  endereço, observações (`contact.get_info_summary()`). Inclui a instrução
  explícita de **não** perguntar nem re-salvar dados que já estão ali.
- **Tags do contato** — classificações aplicadas por operadores humanos.
- **Instruções sobre mensagens de operador** — distingue mensagens do atendente
  humano e "notas privadas" do painel (que o contato nunca viu).
- **Fragmentos de plugins** — cada plugin pode injetar texto via
  `PROMPT_FRAGMENTS`. Erros são isolados (um plugin com bug não derruba o request).
- **Data e hora atuais** — no fuso BRT (-3h).
- **Formato de resposta** — se `split_messages` está ligado (padrão), instrui o
  LLM a responder **sempre como um array JSON de strings**, onde cada string é
  uma mensagem separada do WhatsApp.

Depois de montado, o prompt passa pelo filtro `filter.system_prompt`.

### Etapa 6 — A chamada ao LLM

`aprocess_message()` ([agent/handler.py:519](../agent/handler.py#L519)) é o
coração. Existe em duas versões — `aprocess_message` (async, usada no fluxo real,
**cancelável**) e `process_message` (síncrona, usada no Sandbox). A lógica é a
mesma. Passo a passo da versão async:

1. Carrega/cria a `ContactMemory` do remetente.
2. Salva a mensagem do usuário no banco (`contact.add_message("user", ...)`).
3. Lê o histórico recente: `contact.get_context_messages(max_context_messages)`
   — só as últimas N mensagens (padrão 10) vão para o LLM.
4. Monta `messages = [system, *histórico]`.
5. Aplica os filtros `filter.llm.messages` e `filter.llm.tools` — plugins podem
   reescrever o histórico ou remover tools disponíveis.
6. Emite o evento `llm.before`.
7. Chama `client.chat.completions.create(...)` no OpenRouter, com:
   - `model` configurado (default de construção: `deepseek/deepseek-v4-pro`);
   - `max_tokens=1024`;
   - `tools` + `tool_choice="auto"` (se houver tools ativas).
8. Registra o consumo de tokens/custo (`_record_usage`).

> Modelos são **configuráveis** pela tela de configuração; os valores no
> construtor de `AgentHandler` são apenas defaults.

### Etapa 7 — O loop de tool calling

A IA não só conversa — ela **age**. Se a resposta do LLM contém `tool_calls`,
para cada uma ([agent/handler.py:613](../agent/handler.py#L613)):

1. Faz `json.loads` dos argumentos.
2. Aplica `filter.tool.args` — um plugin pode reescrever os args ou **vetar** a
   chamada (retornando `None`).
3. Emite `tool.before`.
4. Despacha via `_dispatch_tool()` ([agent/handler.py:224](../agent/handler.py#L224)):
   busca o executor no registry `_tool_executors`, monta um `ToolContext` e
   executa. Erros são capturados (uma tool com bug não quebra o request).
5. Emite `tool.after`.
6. Aplica `filter.tool.result` no feedback que volta para o LLM.

> **Nota de arquitetura:** o dispatch é **genérico via registry** — não há
> `if/elif` por nome de tool. Toda tool (core ou plugin) é uma tupla
> `(schema, executor)` registrada em `_tool_executors`. É isso que permite
> plugins adicionarem tools sem tocar no handler.

### Etapa 8 — O follow-up call

Se o LLM retornou **apenas** tool calls e nenhum texto (`not msg.content`), o
handler:

1. Anexa a mensagem do assistant e os resultados das tools ao `messages`.
2. Faz uma **segunda chamada** ao LLM.
3. Essa segunda resposta é a `reply` final.

Se o LLM já retornou texto junto com as tools, esse texto é usado direto.

No fim, emite `llm.after` (com `reply`, `tool_calls`, `usage`, `latency_ms`) e
devolve um `ProcessResult`.

### As tools core

Hoje existem **duas** tools core, em [agent/tools/](../agent/tools/):

| Tool | Arquivo | O que faz |
|---|---|---|
| `save_contact_info` | [save_contact_info.py](../agent/tools/save_contact_info.py) | Quando o LLM detecta dados pessoais na conversa (nome, email, profissão, empresa, endereço, observação), salva no contato automaticamente |
| `transfer_to_human` | [transfer_to_human.py](../agent/tools/transfer_to_human.py) | Desliga a IA para aquele contato, cria/aplica a tag `transferido_atendente` e alerta o operador |

Ambas estão registradas em `CORE_TOOLS` no
[agent/tools/__init__.py](../agent/tools/__init__.py). Plugins adicionam quantas
quiserem sem mexer aqui.

### A memória do contato

[agent/memory.py](../agent/memory.py) define:

- **`ContactMemory`** — wrapper de um contato. No construtor faz
  `contact_repo.get_or_create(phone)`. Expõe `add_message()`, `update_info()`,
  `get_context_messages()`, `add_usage()`, `add_tag()`, etc. As mensagens são
  **lazy-loaded do banco** — não ficam todas em memória; só as últimas N vão ao
  LLM. A última imagem do usuário é inlinada em base64 no contexto.
- **`TagRegistry`** — registro global de tags (nome + cor).

Tudo é persistido via os repositórios em [db/repositories/](../db/repositories/).
O histórico **sobrevive a reinícios** do app.

### Etapa 9 — Envio da resposta

`_send_reply(phone, reply)` no webhook:

1. Aplica `filter.reply.raw` na resposta bruta.
2. Se `split_messages` está ligado, faz `parse_split_reply()`
   ([server/helpers.py](../server/helpers.py)) — interpreta o array JSON que o
   LLM produziu. Se o parse falha, usa a resposta inteira como uma única
   mensagem (fallback seguro).
3. Aplica `filter.reply.parts` na lista de partes.
4. Para **cada parte**:
   - aplica `filter.reply.part`;
   - entre partes, dorme um **delay aleatório** (~2s ±0.5s) e re-envia o
     indicador "digitando..." — para parecer humano;
   - envia via `gowa_client.send_message()`;
   - faz **broadcast WebSocket** (`new_message`) → o frontend atualiza em tempo real;
   - emite o evento `message.sent`;
   - salva a mensagem no banco.
5. Ao final, para o indicador "digitando...".

### Events e Filters no caminho

Repare que **em cada etapa** aparecem ganchos de plugin. Dois mecanismos
complementares (mesmo padrão do WordPress: *actions* + *filters*):

- **Events** — avisos *fire-and-forget* ("isto aconteceu"). O plugin reage sem
  bloquear o pipeline. Ex.: `message.saved`, `llm.after`, `tool.after`.
- **Filters** — interceptadores síncronos no pipeline. Recebem um valor e
  retornam o valor modificado, ou `None` para **abortar** a ação. Ex.:
  `filter.system_prompt`, `filter.reply.part`, `filter.message.before_save`.

Sequência resumida de ganchos numa mensagem:

```
filter.webhook.payload
  → filter.message.before_save
    → filter.transcription.should_run / .result
      → filter.system_prompt
        → filter.llm.messages / filter.llm.tools     [evento llm.before]
          → [LLM call]                                [evento llm.after]
            → filter.tool.args                        [evento tool.before]
              → [tool exec]                           [evento tool.after]
                → filter.tool.result
                  → filter.reply.raw
                    → filter.reply.parts
                      → filter.reply.part             [evento message.sent]
```

A lista completa de eventos e filtros está no [CLAUDE.md](../CLAUDE.md) (seção
"Events e Filters").

---

## 6. Subsistemas

Resumo + ponteiros. Cada um merece um estudo próprio depois.

### Banco de dados — [db/](../db/)

- **SQLAlchemy 2.0 Core**, sem ORM. Cada tabela é um `Table` em
  [db/tables.py](../db/tables.py) (~13 tabelas: `contacts`, `messages`, `usage`,
  `tags`, `plugins`, ...).
- Os **repositórios** em [db/repositories/](../db/repositories/) montam o SQL
  (um arquivo por domínio). Leitura: `with get_engine().connect()`. Escrita:
  `with get_engine().begin()`.
- A **URL do banco** é resolvida em [db/engine.py](../db/engine.py): env
  `DATABASE_URL` → arquivo `storages/database.json` → SQLite padrão.
- **Alembic** roda as migrações no boot (`init_db()` em
  [db/connection.py](../db/connection.py)).
- O mesmo código roda em SQLite e Postgres porque nada é específico de dialeto
  (UPSERTs usam um helper que escolhe o dialeto).

### Plugins — [plugins/](../plugins/)

- Um plugin vive em `storages/plugins/<id>/` e pode agregar: **tools**,
  **fragmentos de prompt**, **endpoints REST**, **telas Preact**, **migrações
  SQL** (prefixo `plugin_<id>_` obrigatório), **settings** declarativas
  (Pydantic), e **events/filters**.
- O motor: [loader.py](../plugins/loader.py) (descoberta + carregamento),
  [manifest.py](../plugins/manifest.py) (parser do `plugin.yaml`),
  [migrator.py](../plugins/migrator.py) (migrações SQL),
  [context.py](../plugins/context.py) (`ToolContext`, `EventContext`, ...),
  [events.py](../plugins/events.py) (bus de events/filters).
- Exemplos prontos em [assets/plugin_examples/](../assets/plugin_examples/):
  `auto_signature`, `blacklist`, `event_logger`, `lembretes`,
  `transcricao_grupos`.
- Para criar um plugin novo: comando `/new-plugin` no Claude Code.

### Frontend — [web/](../web/)

- **Preact + HTM + Tailwind**, **sem build step** — os `.js` são servidos crus.
  Libs vendorizadas em `web/static/vendor/`.
- Entry point: [web/index.html](../web/index.html) →
  [web/static/js/app.js](../web/static/js/app.js) (router).
- Telas em [web/static/js/components/](../web/static/js/components/): Contatos,
  Dashboard, Sandbox, Custos, Execuções, Plugins, Tools.
- Comunica com o backend por **REST** (`/api/...`,
  [services/api.js](../web/static/js/services/api.js)) e por **WebSocket**
  (`/ws`, [services/websocket.js](../web/static/js/services/websocket.js)) para
  atualizações em tempo real.

---

## 7. Roteiro de estudo

Ordem recomendada de leitura dos arquivos, do geral ao específico:

1. [CLAUDE.md](../CLAUDE.md) — panorama técnico completo (leia por cima).
2. [main.py](../main.py) — como tudo arranca.
3. [server/app.py](../server/app.py) — montagem da app, `lifespan`, middleware.
4. [server/background.py](../server/background.py) — tarefas de fundo.
5. [gowa/manager.py](../gowa/manager.py) + [gowa/client.py](../gowa/client.py) — a ponte com o WhatsApp.
6. **[server/routes/webhook.py](../server/routes/webhook.py)** — o fluxo de mensagens (batching, `_orchestrate`, `_send_reply`). O arquivo mais importante.
7. **[agent/handler.py](../agent/handler.py)** — o cérebro de IA (`aprocess_message`, tool calling).
8. [agent/memory.py](../agent/memory.py) + [agent/tools/](../agent/tools/) — memória e tools.
9. [db/tables.py](../db/tables.py) + [db/repositories/](../db/repositories/) — a camada de dados.
10. [plugins/loader.py](../plugins/loader.py) + [plugins/events.py](../plugins/events.py) — o sistema de plugins.
11. [assets/plugin_examples/auto_signature/](../assets/plugin_examples/auto_signature/) — um plugin simples, de ponta a ponta.
12. [web/static/js/app.js](../web/static/js/app.js) — o frontend.

Para o **fluxo de IA** (foco deste guia), os itens **6 e 7** são o essencial —
leia-os com este documento aberto ao lado.

---

## 8. Como rodar localmente

| Quero... | Comando |
|---|---|
| Desenvolver no Windows (hot-reload) | `windows_start.bat` (baixa Python sozinho na 1ª vez) |
| Desenvolver no Linux/macOS | `./linux_start.sh` |
| Testar o build Docker | `./docker_start.sh` |
| Rodar os testes de endpoint | `python tests/test_endpoints.py` |
| Parar (Windows dev) | `windows_stop.bat` |

Depois de subir, o painel abre em `http://127.0.0.1:8080`. Escaneie o QR code
exibido para conectar um WhatsApp e ver o fluxo de mensagens funcionando.

Para testar o fluxo de IA sem WhatsApp real, use a tela **Sandbox** no painel —
ela chama `process_message` direto, sem passar pelo webhook nem pelo GOWA.
