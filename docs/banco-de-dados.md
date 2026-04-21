# Banco de Dados — Schema Completo (Supabase / PostgreSQL)

## Visão Geral

O Vox AI usa o Supabase como backend de dados completo. O banco é PostgreSQL 17 com:

- **7 tabelas**
- **3 funções customizadas** (incluindo 2 usadas como triggers)
- **3 triggers ativas**
- **7 índices de performance** (incluindo 1 HNSW para busca vetorial)
- **6 constraints de chave estrangeira**
- **RLS (Row Level Security) habilitada em todas as tabelas**
- **5 extensões ativas**

Esta seção foi gerada diretamente a partir do dump do schema SQL do banco de produção.

## Extensões Ativas

| Extensão | Schema | Finalidade |
|---|---|---|
| **vector** | public | Adiciona o tipo `vector`, o operador de distância coseno (`<=>`), funções de similaridade e suporte a índices HNSW. É o núcleo do sistema RAG. |
| **pg_graphql** | graphql | Expoe o schema do banco como uma API GraphQL automaticamente. Usado pelo Supabase Studio. |
| **pg_stat_statements** | extensions | Coleta estatísticas de execução de queries SQL (tempo, chamadas, rows). Útil para profiling e otimização. |
| **pgcrypto** | extensions | Funções criptográficas (hash, UUID, criptografia simetrica). Usada internamente pelo Supabase Auth. |
| **supabase_vault** | vault | Cofre seguro para armazenar segredos (API keys, tokens) diretamente no banco, criptografados. |
| **uuid-ossp** | extensions | Gera UUIDs v1, v3, v4 e v5 via SQL. Usada para geração de IDs únicos em contextos SQL puros. |

## Tabelas

### `knowledge_base` — Base de Conhecimento

A tabela mais importante do sistema. Cada linha representa um "chunk" de conhecimento curado pela equipe. E aqui que o RAG busca as informações para embasar as respostas da IA.

| Coluna | Tipo SQL | Nullable | Default | Descrição |
|---|---|---|---|---|
| `kb_id` | text | NOT NULL | `'vox-kb-' \|\| lpad(nextval('kb_id_seq'),4,'0')` | PK. ID legivel gerado automaticamente (ex: `vox-kb-0001`). Usa a sequence `kb_id_seq` compartilhada com a tabela ETL. |
| `topico` | text | NULL | — | Nome do tópico macro (ex: "PrEP"). Indexado (`idx_kb_topico`). |
| `eixo_tematico` | text | NULL | — | Eixo temático superior (ex: "Saúde"). Indexado (`idx_kb_eixo`). |
| `descricao` | text | NOT NULL | — | O texto do conhecimento em si. É este campo que é convertido em vetor e recuperado pelo RAG. |
| `referencias` | text | NULL | — | Fontes e referências bibliográficas do conhecimento. |
| `tags` | text[] | NULL | — | Array de tags para categorização auxiliar. |
| `autor` | text | NULL | — | Responsável pela inserção do chunk. |
| `created_at` | timestamptz | NULL | `now()` | Data de criação do registro. |
| `kb_count` | numeric | NULL | — | Contador de quantas vezes o chunk foi recuperado. Incrementado automaticamente pela trigger `tg_update_kb_usage`. |
| `embedding` | vector(1536) | NULL | — | Vetor semântico de 1536 dimensões. Indexado com HNSW (`idx_kb_embedding`). Gerado pelo modelo `gemini-embedding-001`. |
| `modificado_em` | timestamptz | NULL | `now()` | Data da última modificação. Atualizado automaticamente pela trigger `update_kb_modificado_em`. |

> **Nota:** O `kb_id` e gerado automaticamente pelo banco usando a expressão `DEFAULT`. A sequence `kb_id_seq` é compartilhada entre `knowledge_base` e `knowledge_base_etl`, o que garante que os IDs são únicos globalmente entre as duas tabelas.

### `knowledge_base_etl` — Tabela de Staging (ETL)

Tabela de staging com estrutura identica a `knowledge_base`, mas com a coluna `embedding` sem dimensão fixa (aceita qualquer tamanho de vetor). Usada para importar, processar e validar novos dados da curadoria antes de promove-los para a tabela principal. Possui os mesmos triggers de `modificado_em`.

| Diferenca em relacao a `knowledge_base` | Detalhe |
|---|---|
| Coluna `embedding` | `vector` sem dimensão fixa — aceita qualquer tamanho de vetor |
| Sem índice HNSW | Não há `idx_kb_embedding` nesta tabela (dados ainda não são buscados) |
| Sequence `kb_id_seq` compartilhada | O ID gerado aqui tambem consome da mesma sequence da `knowledge_base` |

> **Nota:** Pense no `knowledge_base_etl` como uma "fila de aprovação": novos chunks entram aqui, são validados pela curadoria e depois movidos (INSERT + DELETE) para a `knowledge_base`, momento em que o embedding com dimensão correta é gerado.

### `sessions`

| Coluna | Tipo SQL | Descrição |
|---|---|---|
| `session_id` | text (PK) | UUID anônimo gerado pelo front-end via `uuid.uuid4()`. Identifica a sessão do usuário. |
| `id` | bigint (UNIQUE, IDENTITY) | ID numérico auto-incremental interno. Gerado `BY DEFAULT AS IDENTITY`. |
| `created_at` | timestamptz | Momento de criação da sessão. DEFAULT `now()`. |

> **Nota:** A tabela `sessions` é a âncora de integridade referencial do banco. As tabelas `chat_logs`, `error_logs` e `user_reports` tem FK para `sessions.session_id` com `ON DELETE RESTRICT`, ou seja, uma sessão só pode ser deletada se não houver registros dependentes.

### `chat_logs`

| Coluna | Tipo SQL | Descrição |
|---|---|---|
| `chat_id` | bigint (PK, IDENTITY) | ID auto-incremental gerado `BY DEFAULT AS IDENTITY`. |
| `session_id` | text (FK -> sessions) | Referencia a sessão. `ON DELETE RESTRICT`. |
| `prompt` | text | Pergunta enviada pelo usuário. |
| `response` | text | Resposta gerada pelo Vox AI. |
| `git_version` | text | Versão do software no momento da conversa (para rastrear bugs em releases). |
| `created_at` | timestamptz | Data e hora da interação. DEFAULT `now()`. |

### `chat_logs_kb` — Pivot de Auditoria RAG

Tabela fundamental para auditabilidade. Conecta cada resposta da IA (`chat_logs`) com os fragmentos exatos de conhecimento (`knowledge_base`) que foram usados para gerá-la. Permite rastrear a origem de possíveis alucinações e medir a utilidade de cada chunk.

| Coluna | Tipo SQL | Descrição |
|---|---|---|
| `chat_log_kb_id` | bigint (PK, UNIQUE, IDENTITY) | ID auto-incremental gerado `BY DEFAULT AS IDENTITY`. |
| `chat_id` | bigint (FK -> chat_logs) | Referencia ao log de chat. `ON UPDATE CASCADE`. |
| `kb_id` | text (FK -> knowledge_base) | Referencia ao chunk usado. `ON UPDATE CASCADE`. |
| `similarity` | double precision | Score de similaridade cosseno (0 a 1). Pode ser NULL quando a Estratégia de Contexto Expandido e usada (busca direta por tópico, sem score de similaridade). |
| `created_at` | timestamptz | Data do registro. DEFAULT `now()`. |

> **Nota:** O INSERT nesta tabela aciona automaticamente o trigger `tg_update_kb_usage`, que chama `increment_kb_count()` e incrementa `kb_count` na `knowledge_base`. Isso cria um sistema automático de métricas de utilidade dos chunks sem nenhuma lógica extra no Python.

### `user_reports`

| Coluna | Tipo SQL | Descrição |
|---|---|---|
| `id` | bigint (PK, IDENTITY) | ID auto-incremental gerado `ALWAYS AS IDENTITY`. |
| `session_id` | text (FK -> sessions) | Sessão que originou o report. `ON DELETE RESTRICT`. |
| `category_id` | bigint (FK -> report_categories) | Categoria da denuncia. `ON UPDATE CASCADE`. |
| `comment` | text | Comentário textual do usuário descrevendo o problema. |
| `chat_history` | text | Histórico completo da conversa no momento do report (JSON serializado como string). |
| `git_version` | text | Versão do software para correlacionar com releases. |
| `created_at` | timestamptz | Data do report. DEFAULT `now()`. |

### `report_categories`

Tabela de referencia com as categorias disponíveis para classificar um report. Possui COMMENT no schema: *"Tags para o erro que o usuário deseja reportar."*

| Coluna | Tipo SQL | Descrição |
|---|---|---|
| `id` | integer (PK, SERIAL) | ID auto-incremental via sequence `report_categories_id_seq`. |
| `label` | text | Nome da categoria exibido ao usuário (ex: "Alucinacao", "Conteúdo Ofensivo"). |
| `description` | text | Descrição detalhada do que se enquadra nessa categoria. |
| `created_at` | timestamptz | Data de criação. DEFAULT `now()`. |

### `error_logs`

| Coluna | Tipo SQL | Descrição |
|---|---|---|
| `id` | bigint (PK, IDENTITY) | ID auto-incremental gerado `ALWAYS AS IDENTITY`. |
| `error_id` | text | Hash curto de 8 caracteres gerado por `uuid4()[:8]`. Exibido ao usuário para facilitar o reporte a equipe. |
| `error_message` | text | Mensagem de erro completa capturada pelo bloco `except`. |
| `session_id` | text (FK -> sessions) | Sessão onde o erro ocorreu. `ON DELETE RESTRICT`. |
| `git_version` | text | Versão do software onde o erro foi detectado. |
| `created_at` | timestamptz | Data e hora do erro. DEFAULT `now()`. |

## Funções PostgreSQL

### `match_knowledge_base()` — Busca Vetorial RPC

A função mais crítica do sistema. Executa a busca vetorial por similaridade de cosseno usando o operador `<=>` do pgvector. E exposta como RPC pelo Supabase e chamada pelo Python via `client.rpc()`.

```sql
CREATE OR REPLACE FUNCTION public.match_knowledge_base(
    query_embedding  vector,           -- vetor de 1536 dimensões
    match_threshold  double precision, -- similaridade mínima (ex: 0.5)
    match_count      integer,          -- máximo de resultados (ex: 10)
    filter_topic     text DEFAULT NULL -- filtro opcional por tópico
) RETURNS TABLE (
    id text, topico text, eixo_tematico text, descrição text, similarity double precision
)

-- Fórmula da similaridade coseno:
-- similarity = 1 - (embedding <=> query_embedding)
-- Onde <=> e o operador de distância coseno do pgvector.
-- Quanto menor a distância, maior a similaridade.
-- Resultados ordenados pela distância (ASC) = maior similaridade primeiro.
```

> **Nota:** O operador `<=>` calcula a distância coseno entre dois vetores. A formula `1 - distância` converte isso em um score de similaridade entre 0 e 1, onde 1 significa idêntico e 0 significa completamente diferente. O threshold de 0.5 descarta resultados com mais de 50% de diferença semântica.

### `increment_kb_count()` — Trigger Function

Função chamada automaticamente pelo trigger `tg_update_kb_usage` após cada INSERT na tabela `chat_logs_kb`. Incrementa o campo `kb_count` na `knowledge_base` para o `kb_id` recém-inserido.

```sql
CREATE OR REPLACE FUNCTION public.increment_kb_count()
RETURNS trigger LANGUAGE plpgsql AS $$
begin
  UPDATE knowledge_base
  set kb_count = kb_count + 1
  where kb_id = new.kb_id;
  return new;
end
$$;
```

### `update_modificado_em()` — Trigger Function

Função chamada automaticamente pelos triggers `update_kb_modificado_em` e `update_kb_etl_modificado_em` antes de qualquer UPDATE nas tabelas `knowledge_base` e `knowledge_base_etl`. Garante que o campo `modificado_em` seja sempre atualizado para o timestamp atual.

```sql
CREATE OR REPLACE FUNCTION public.update_modificado_em()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    NEW.modificado_em = now();
    RETURN NEW;
END;
$$;
```

> **Nota:** Este campo e utilizado pelo `dashboard.js` para calcular a "versão" da base de conhecimento. Formato: `v{ano}.{mes}.{dia}` baseado na data da última modificação.

## Triggers

| Nome do Trigger | Tabela | Evento | Momento | Função Chamada | Descrição |
|---|---|---|---|---|---|
| `tg_update_kb_usage` | chat_logs_kb | INSERT | AFTER | `increment_kb_count()` | Incrementa `kb_count` no chunk da KB sempre que ele é registrado como utilizado em uma resposta. |
| `update_kb_modificado_em` | knowledge_base | UPDATE | BEFORE | `update_modificado_em()` | Atualiza o timestamp `modificado_em` automaticamente em qualquer UPDATE na KB. |
| `update_kb_etl_modificado_em` | knowledge_base_etl | UPDATE | BEFORE | `update_modificado_em()` | Mesma função, aplicada a tabela de staging ETL. |

## Índices

| Nome | Tabela | Tipo | Coluna(s) | Finalidade |
|---|---|---|---|---|
| `idx_kb_embedding` | knowledge_base | **HNSW** | embedding (vector_cosine_ops) | Índice principal do RAG. Permite busca por vizinhos mais próximos (cosine) em milissegundos mesmo com milhares de chunks. |
| `idx_kb_topico` | knowledge_base | B-tree | tópico | Acelera a estratégia de Contexto Expandido (busca `WHERE topico = ?`). |
| `idx_kb_eixo` | knowledge_base | B-tree | eixo_temático | Acelera filtros por eixo temático. |
| `idx_chat_logs_session_id` | chat_logs | B-tree | session_id | Acelera a recuperação do historico de uma sessão específica. |
| `idx_chat_logs_kb_log` | chat_logs_kb | B-tree | chat_log_kb_id | Suporte a queries na tabela pivot de auditoria. |
| `idx_error_logs_session_id` | error_logs | B-tree | session_id | Acelera a busca de erros por sessão. |
| `idx_user_reports_session_id` | user_reports | B-tree | session_id | Acelera a busca de reports por sessão. |

> **Nota:** O índice HNSW (Hierarchical Navigable Small World) é um algoritmo de busca aproximada de vizinhos mais próximos. Ele constroi um grafo hierárquico de nos conectados e permite navegar para o vizinho mais próximo em tempo sub-linear. É a escolha padrão para produção com `pgvector`.

## Foreign Keys e Integridade Referencial

| Constraint | De | Para | ON UPDATE | ON DELETE |
|---|---|---|---|---|
| `chat_log_kb_chat_id_fkey` | chat_logs_kb.chat_id | chat_logs.chat_id | CASCADE | — |
| `chat_log_kb_kb_id_fkey` | chat_logs_kb.kb_id | knowledge_base.kb_id | CASCADE | — |
| `chat_logs_session_id_fkey` | chat_logs.session_id | sessions.session_id | — | RESTRICT |
| `error_logs_session_id_fkey` | error_logs.session_id | sessions.session_id | — | RESTRICT |
| `user_reports_session_id_fkey` | user_reports.session_id | sessions.session_id | — | RESTRICT |
| `user_reports_category_id_fkey` | user_reports.category_id | report_categories.id | CASCADE | — |

> **Nota:** O `ON DELETE RESTRICT` na sessão significa que não e possivel deletar uma sessão diretamente enquanto houver registros dependentes. Para deletar uma sessão, é necessário primeiro deletar todos os `chat_logs`, `error_logs` e `user_reports` associados a ela.

## Row Level Security (RLS)

RLS está habilitada em **todas** as tabelas do schema public. Por padrão, nenhum usuário pode ler ou escrever em nenhuma tabela sem uma policy explícita autorizando.

| Tabela | RLS | Policy Ativa | Detalhe |
|---|---|---|---|
| knowledge_base | Habilitada | 1 policy pública | `FOR SELECT TO anon USING (true)` — qualquer usuário anônimo pode **ler** a KB. Escrita requer `service_role`. |
| chat_logs | Habilitada | Nenhuma | Apenas `service_role` pode inserir (SDK Python do backend). |
| chat_logs_kb | Habilitada | Nenhuma | Apenas `service_role` pode inserir. |
| sessions | Habilitada | Nenhuma | Apenas `service_role` pode inserir. |
| user_reports | Habilitada | Nenhuma | Apenas `service_role` pode inserir. |
| error_logs | Habilitada | Nenhuma | Apenas `service_role` pode inserir. |
| report_categories | Habilitada | Nenhuma | Apenas `service_role` pode modificar. |
| knowledge_base_etl | Habilitada | Nenhuma | Uso interno pela equipe de curadoria. |

> **ATENÇÃO:** A chave `anon` do Supabase é segura de expor no frontend (Dashboard) porque as policies RLS garantem que mesmo com ela em mãos, um atacante so consegue fazer `SELECT` na `knowledge_base`. Todas as operações de escrita exigem a chave `service_role`, que fica apenas no backend/CI.

## Migrations

### Migration 1: `20260410192141_remote_schema.sql`

Schema inicial completo do banco. Cria toda a estrutura descrita nesta seção: extensões, sequences, tabelas, funções, triggers, índices, foreign keys, grants de permissão e RLS policies. E o estado base do banco antes de qualquer alteração posterior.

### Migration 2: `20260418194905_alter_vetor_tamanho_1536_knowledge_base.sql`

Necessária para mudar o modelo de embeddings de uma versão anterior (vetores de 768 dimensões) para o `gemini-embedding-001` (1536 dimensões). Como o PostgreSQL não permite alterar a dimensão de uma coluna `vector` diretamente com dados, a migration faz o processo em duas etapas:

```sql
-- Passo 1: Zerar todos os embeddings existentes
UPDATE knowledge_base SET embedding = null WHERE embedding IS NOT NULL;

-- Passo 2: Alterar o tipo da coluna para o novo tamanho
ALTER TABLE public.knowledge_base
  ALTER COLUMN embedding
  SET DATA TYPE public.vector(1536)
  USING embedding::public.vector(1536);
```

> **ATENÇÃO:** Após esta migration, todos os registros tiveram seus embeddings zerados. Foi necessário rodar `scripts/gerar_embedding.py` para reindexar toda a base de conhecimento com os novos vetores de 1536 dimensões.

## Sequences (Auto-increment)

| Sequence | Tipo | Usada por | Comportamento |
|---|---|---|---|
| `kb_id_seq` | SEQUENCE (bigint) | `knowledge_base.kb_id` e `knowledge_base_etl.kb_id` | Compartilhada entre as duas tabelas. Gera o numero do ID legivel (ex: `vox-kb-0001`). Comeca em 1, incrementa 1. |
| `chat_log_kb_chat_log_kb_id_seq` | IDENTITY (bigint) | `chat_logs_kb.chat_log_kb_id` | Gerenciada automaticamente pelo `GENERATED BY DEFAULT AS IDENTITY`. |
| `chat_logs_id_seq` | IDENTITY (bigint) | `chat_logs.chat_id` | Idem. |
| `error_logs_id_seq` | IDENTITY (bigint) | `error_logs.id` | `GENERATED ALWAYS AS IDENTITY` — não aceita valores manuais. |
| `report_categories_id_seq` | SEQUENCE (integer) | `report_categories.id` | Sequence clássica linkada via `OWNED BY`. |
| `sessions_id_seq` | IDENTITY (bigint) | `sessions.id` | `GENERATED BY DEFAULT AS IDENTITY`. |
| `user_reports_id_seq` | IDENTITY (bigint) | `user_reports.id` | `GENERATED ALWAYS AS IDENTITY`. |

---