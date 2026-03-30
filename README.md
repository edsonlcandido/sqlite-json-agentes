# Banco de Dados de Propostas

Sistema de extração, armazenamento e consulta inteligente de propostas técnicas de ventiladores industriais (centrífugos, axiais e mixed flow) utilizando agentes de IA com Azure OpenAI.

## Visão Geral

O projeto é composto por um **backend ASP.NET Core 9** e um **frontend Vue.js 3** com Tailwind CSS. A aplicação permite:

1. **Upload de propostas** — Arquivos `.txt` de propostas técnicas são enviados e processados por um agente de IA que extrai dados estruturados em JSON.
2. **Consulta inteligente** — O usuário faz perguntas em linguagem natural através de um chat, e uma cadeia de agentes converte a pergunta em SQL, executa no banco e retorna os resultados formatados.
3. **Visualização de equipamentos** — Dashboard com os detalhes dos equipamentos extraídos das propostas.

## Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Frontend | Vue.js 3, TypeScript, Tailwind CSS, Vite |
| Backend | ASP.NET Core 9 (Minimal API) |
| Banco de Dados | SQLite (com extensão JSON1) |
| IA | Azure OpenAI (gpt-4.1-mini) |
| Frameworks de IA | Microsoft Semantic Kernel + Microsoft Agent Framework (MAF) |
| ORM/Data | Dapper |

## Arquitetura de Agentes

O sistema utiliza **3 agentes de IA** organizados em dois fluxos independentes:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FLUXO 1 — EXTRAÇÃO (Upload)                     │
│                                                                     │
│  Arquivo .txt ──▶ AIAgentExtratorServiceMAF ──▶ JSON ──▶ SQLite    │
│                   (Microsoft Agent Framework)                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                   FLUXO 2 — CONSULTA (Chat)                         │
│                                                                     │
│  Pergunta ──▶ AgentOrchestrationService                             │
│                  │                                                   │
│                  ├─ Etapa 1: AIQueryBuilderService                   │
│                  │           (Pergunta ──▶ SQL)                      │
│                  │                                                   │
│                  └─ Etapa 2: AISQLExecutorService                    │
│                              (SQL ──▶ Tool Call ──▶ Resultados)      │
│                              └── SqliteExecutorTool (READ-ONLY)      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Fluxo 1 — Agente Extrator (`AIAgentExtratorServiceMAF`)

**Endpoint:** `POST /api/propostas/analisar`

Este agente utiliza o **Microsoft Agent Framework (MAF)** através do `ChatCompletionAgent` do Semantic Kernel. Ele é responsável por receber o conteúdo textual bruto de uma proposta técnica de ventilador e extrair um JSON estruturado com todas as informações relevantes.

#### Como funciona

1. O usuário faz upload de um arquivo `.txt` contendo a proposta técnica na tela de **Upload**.
2. O frontend lê o conteúdo do arquivo e envia via POST para o endpoint `/api/propostas/analisar`.
3. O `AIAgentExtratorServiceMAF` cria um `ChatCompletionAgent` com:
   - **Kernel** configurado com Azure OpenAI (`gpt-4.1-mini`)
   - **Instructions** carregadas do arquivo `Instructions/AgentExtratorInstructions.txt`
   - **Temperature/TopP** em `0.5` para balancear criatividade e precisão
   - **MaxTokens** de `16384` para suportar propostas com muitos equipamentos
4. O agente processa a proposta de forma **stateless** — cada requisição cria um novo `ChatHistory`.
5. A resposta JSON é limpa (remoção de markdown, Unicode escapes, texto extra) e salva na tabela `propostas_json` do SQLite.

#### Estrutura JSON extraída

```json
{
  "cliente": "Nome do Cliente",
  "projeto": "Nome do Projeto",
  "nossa_referencia": "339940-T5",
  "data": "11/11/2025",
  "preparado_por": "Eduardo Pontes",
  "equipamentos": [
    {
      "tipo": "centrifugo",
      "modelo": "MVC 1650.00.15",
      "vazao": [200, "kCFM"],
      "pressao": [22.25, "inWg"],
      "motor": {
        "potencia": [850, "HP"],
        "tensao": [4160, "V"],
        "polos": 4
      },
      "caracteristicas_construtivas": {
        "carcaca": { "material": "ASTM A-36" },
        "rotor": { "material": "SAR 80" }
      },
      "escopo": {
        "incluido": ["Motor elétrico", "Carcasa", "Silenciador"],
        "excluido": ["Pernos de anclaje"]
      }
    }
  ]
}
```

#### Por que MAF?

O `ChatCompletionAgent` do Microsoft Agent Framework oferece:
- Abstração limpa sobre o Semantic Kernel para processamento stateless
- Suporte nativo a streaming via `InvokeAsync`
- Integração direta com o ecossistema Microsoft (Azure OpenAI, Semantic Kernel)

---

### Fluxo 2 — Consulta via Chat (Orquestração de Agentes)

**Endpoint:** `POST /api/propostas/query-ai`

O fluxo de consulta é gerenciado pelo `AgentOrchestrationService`, que executa **duas etapas sequenciais**:

#### Etapa 1 — Agente Query Builder (`AIQueryBuilderService`)

Converte perguntas em linguagem natural para queries SQL válidas para SQLite com JSON1.

- Usa `ChatClient` do Azure OpenAI SDK diretamente (sem Semantic Kernel)
- **Temperature** `0.1` — muito baixa para gerar SQL determinístico e consistente
- Instruções carregadas de `Instructions/AgentQueryBuilder.txt`

**Exemplos de conversão:**

| Pergunta | SQL Gerado |
|---|---|
| "Quais propostas têm motores acima de 500kW?" | `SELECT ... FROM propostas_json p, json_each(p.proposta, '$.equipamentos') AS e WHERE json_extract(e.value, '$.motor.potencia[0]') * CASE ... END > 500` |
| "Liste ventiladores centrífugos com pressão acima de 3000 Pa" | `SELECT ... WHERE json_extract(e.value, '$.tipo') LIKE '%centr_fugo%' AND (pressão normalizada) > 3000` |

**Recursos avançados das instruções do Query Builder:**

- Navegação em JSON aninhado via `json_extract` e `json_each`
- **Conversão automática de unidades**: pressão (mmCA→Pa, inWg→Pa), vazão (m³/h→m³/s), potência (HP→kW, cv→kW)
- Busca textual com suporte a acentos usando `LIKE` com wildcards (`%press_o%`)
- Consulta em arrays de escopo com `EXISTS` + `json_each`

#### Etapa 2 — Agente SQL Executor (`AISQLExecutorService`)

Executa a query SQL gerada e formata os resultados para o usuário. Este agente implementa **Tool Calling** nativo do OpenAI.

##### Como funciona o Tool Calling

1. O agente recebe o SQL gerado pela Etapa 1.
2. O LLM reconhece que precisa executar o SQL e faz uma **Tool Call** para a função `execute_sqlite_query`.
3. O serviço intercepta a Tool Call, extrai os parâmetros e executa via **`SqliteExecutorTool`**.
4. O resultado JSON da tool é enviado de volta ao LLM como `ToolChatMessage`.
5. O LLM formata os resultados em texto amigável (com emojis, listas numeradas, etc.).

```
LLM recebe SQL
    │
    ▼
Decide chamar tool "execute_sqlite_query"
    │
    ▼
SqliteExecutorTool.ExecuteQueryAsync(sql)  ← READ-ONLY
    │
    ▼
Resultado JSON retornado ao LLM
    │
    ▼
LLM formata resposta amigável para o usuário
```

##### SqliteExecutorTool — Segurança

A tool de execução SQL implementa múltiplas camadas de segurança:

- **Conexão READ-ONLY** (`SqliteOpenMode.ReadOnly`) — impede qualquer operação de escrita a nível do driver
- **Validação de comandos** — bloqueia `INSERT`, `UPDATE`, `DELETE`, `DROP`, `CREATE`, `ALTER`, `TRUNCATE` antes da execução
- **Timeout** de 30 segundos para evitar queries longas
- Retorna resultados em JSON estruturado com metadata (`success`, `total_resultados`, `sql_executado`)

---

## Estrutura do Banco de Dados

### Tabela `propostas_json`

```sql
CREATE TABLE propostas_json (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,
    proposta TEXT  -- JSON completo da proposta extraída
);
```

### View `detalhes-equipamento`

View que extrai e normaliza dados dos equipamentos para unidades SI:

- Pressão → Pa
- Vazão → m³/s
- Potência → kW

Utiliza `json_extract` e `json_each` para desnormalizar os equipamentos do JSON armazenado.

---

## Endpoints da API

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/api/propostas/analisar` | Envia proposta para extração JSON via Agente Extrator (MAF) |
| `POST` | `/api/propostas/query-ai` | Pergunta em linguagem natural → Orquestração de agentes → Resultados |
| `GET` | `/api/equipamentos/detalhes` | Lista detalhes de todos os equipamentos (via view SQL) |
| `POST` | `/api/test/sql-executor` | Teste direto do executor SQL (debug) |

---

## Páginas do Frontend

| Rota | View | Descrição |
|---|---|---|
| `/` | `ChatView` | Chat com agente de consulta (perguntas em linguagem natural) |
| `/upload` | `FileUploadView` | Upload de propostas `.txt` para extração |
| `/equipamentos` | `EquipamentoDetalhesView` | Dashboard de equipamentos extraídos |

---

## Como Executar

### Pré-requisitos

- .NET 9 SDK
- Node.js 20.19+ ou 22.12+
- Chave de API do Azure OpenAI configurada em `appsettings.json`

### Backend

```bash
cd banco-de-dados-propostas.Server
dotnet run
```

### Frontend

```bash
cd banco-de-dados-propostas.client
npm install
npm run dev
```

O servidor backend inicia em `http://localhost:5000` / `https://localhost:5001` e o frontend via Vite com SPA proxy.

---

## Diagrama de Sequência — Fluxo Completo de Consulta

```
Usuário          Frontend         Backend              QueryBuilder        SQLExecutor         SqliteTool          SQLite
  │                 │                │                      │                   │                   │                │
  │─ Pergunta ─────▶│                │                      │                   │                   │                │
  │                 │─ POST /query-ai▶│                      │                   │                   │                │
  │                 │                │─ ProcessQuestion() ──▶│                   │                   │                │
  │                 │                │                      │─ GenerateSql() ───▶│                   │                │
  │                 │                │                      │   (LLM: NL→SQL)   │                   │                │
  │                 │                │                      │◀── SQL ───────────│                   │                │
  │                 │                │◀─── SQL ────────────│                   │                   │                │
  │                 │                │─── ExecuteSql() ────────────────────────▶│                   │                │
  │                 │                │                      │                   │─ Tool Call ───────▶│                │
  │                 │                │                      │                   │                   │─ SELECT ... ──▶│
  │                 │                │                      │                   │                   │◀── rows ──────│
  │                 │                │                      │                   │◀── JSON ─────────│                │
  │                 │                │                      │                   │─ LLM formata ────▶│                │
  │                 │                │◀─── Resultado ──────────────────────────│                   │                │
  │                 │◀── JSON ──────│                      │                   │                   │                │
  │◀── Resposta ───│                │                      │                   │                   │                │
```
