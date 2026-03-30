# Banco de Dados de Motores Elétricos

Sistema de extração, armazenamento e consulta inteligente de catálogos e fichas técnicas de motores elétricos industriais utilizando agentes de IA com Azure OpenAI.

## Visão Geral

O projeto é composto por um **backend ASP.NET Core 9** e um **frontend Vue.js 3** com Tailwind CSS. A aplicação permite:

1. **Upload de fichas técnicas** — Arquivos `.txt` de catálogos/fichas técnicas de motores elétricos são enviados e processados por um agente de IA que extrai dados estruturados em JSON.
2. **Consulta inteligente** — O usuário faz perguntas em linguagem natural através de um chat, e uma cadeia de agentes converte a pergunta em SQL, executa no banco e retorna os resultados formatados.
3. **Visualização de equipamentos** — Dashboard com os detalhes dos motores elétricos extraídos das fichas técnicas.

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

Este agente utiliza o **Microsoft Agent Framework (MAF)** através do `ChatCompletionAgent` do Semantic Kernel. Ele é responsável por receber o conteúdo textual bruto de uma ficha técnica de motor elétrico e extrair um JSON estruturado com todas as informações relevantes.

#### Como funciona

1. O usuário faz upload de um arquivo `.txt` contendo a ficha técnica do motor na tela de **Upload**.
2. O frontend lê o conteúdo do arquivo e envia via POST para o endpoint `/api/propostas/analisar`.
3. O `AIAgentExtratorServiceMAF` cria um `ChatCompletionAgent` com:
   - **Kernel** configurado com Azure OpenAI (`gpt-4.1-mini`)
   - **Instructions** carregadas do arquivo `SystemPrompts/AgentExtratorInstructions.txt`
   - **Temperature/TopP** em `0.5` para balancear criatividade e precisão
   - **MaxTokens** de `16384` para suportar fichas com muitos motores
4. O agente processa a ficha de forma **stateless** — cada requisição cria um novo `ChatHistory`.
5. A resposta JSON é limpa (remoção de markdown, Unicode escapes, texto extra) e salva na tabela `propostas_json` do SQLite.

#### Estrutura JSON extraída

```json
{
  "fabricante": "WEG",
  "catalogo": "Motores Trifásicos - Linha W22",
  "referencia": "CAT-W22-2026",
  "data": "15/03/2026",
  "equipamentos": [
    {
      "tipo": "trifasico",
      "carcaca": "355M/L",
      "modelo": "W22 IR3 Premium",
      "aplicacao": "Bomba centrífuga",
      "potencia": [250, "kW"],
      "tensao": [440, "V"],
      "corrente_nominal": [420, "A"],
      "frequencia": [60, "Hz"],
      "polos": 4,
      "rotacao": [1785, "rpm"],
      "rendimento": [96.2, "%"],
      "fator_potencia": 0.86,
      "ip": "IP55",
      "classe_isolamento": "F",
      "regime": "S1",
      "fator_servico": 1.15,
      "caracteristicas_construtivas": {
        "carcaca": { "material": "Ferro fundido" },
        "rotor": { "material": "Alumínio injetado" },
        "eixo": { "material": "SAE 1045" },
        "rolamentos": {
          "dianteiro": "6317-C3",
          "traseiro": "6313-C3",
          "lubrificacao": "Graxa"
        }
      },
      "caracteristicas_eletricas": {
        "corrente_partida_in": 7.5,
        "torque_partida_tn": 2.2,
        "torque_maximo_tn": 3.0,
        "metodo_partida": "Inversor de frequência"
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
- Instruções carregadas de `SystemPrompts/AgentQueryBuilder.txt`

**Exemplos de conversão:**

| Pergunta | SQL Gerado |
|---|---|
| "Quais motores têm potência acima de 500kW?" | `SELECT ... FROM propostas_json p, json_each(p.proposta, '$.equipamentos') AS e WHERE json_extract(e.value, '$.potencia[0]') * CASE ... END > 500` |
| "Liste motores trifásicos com rendimento acima de 95%" | `SELECT ... WHERE json_extract(e.value, '$.tipo') LIKE '%trif_sico%' AND json_extract(e.value, '$.rendimento[0]') > 95` |
| "Motores com tensão 440V e 6 polos" | `SELECT ... WHERE json_extract(e.value, '$.tensao[0]') = 440 AND json_extract(e.value, '$.polos') = 6` |

**Recursos avançados das instruções do Query Builder:**

- Navegação em JSON aninhado via `json_extract` e `json_each`
- **Conversão automática de unidades**: potência (HP→kW, cv→kW), corrente (mA→A), tensão (kV→V)
- Busca textual com suporte a acentos usando `LIKE` com wildcards (`%trif_sico%`)
- Consulta em arrays de características com `EXISTS` + `json_each`

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
    proposta TEXT  -- JSON completo da ficha técnica extraída
);
```

### View `detalhes-equipamento`

View que extrai e normaliza dados dos motores para unidades SI:

- Potência → kW
- Corrente → A
- Tensão → V

Utiliza `json_extract` e `json_each` para desnormalizar os equipamentos do JSON armazenado.

---

## Endpoints da API

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/api/propostas/analisar` | Envia ficha técnica para extração JSON via Agente Extrator (MAF) |
| `POST` | `/api/propostas/query-ai` | Pergunta em linguagem natural → Orquestração de agentes → Resultados |
| `GET` | `/api/equipamentos/detalhes` | Lista detalhes de todos os motores (via view SQL) |
| `POST` | `/api/test/sql-executor` | Teste direto do executor SQL (debug) |

---

## Páginas do Frontend

| Rota | View | Descrição |
|---|---|---|
| `/` | `ChatView` | Chat com agente de consulta (perguntas em linguagem natural) |
| `/upload` | `FileUploadView` | Upload de fichas técnicas `.txt` para extração |
| `/equipamentos` | `EquipamentoDetalhesView` | Dashboard de motores elétricos extraídos |

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
