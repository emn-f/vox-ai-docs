# Glossário

| Termo | Definição |
|---|---|
| **RAG (Retrieval-Augmented Generation)** | Técnica de IA que combina busca de informações relevantes em uma base de dados com geração de respostas por um LLM. Evita alucinações ao fornecer contexto real ao modelo. |
| **Embedding / Vetor Semantico** | Representação numérica de um texto como array de números que captura o significado semântico. Textos com significados similares tem vetores matematicamente próximos. |
| **pgvector** | Extensão do PostgreSQL que adiciona o tipo `vector`, o operador de distância cosseno (`<=>`), e suporte a índices HNSW. |
| **Distância Cosseno / Operador <=>** | Mede o ângulo entre dois vetores. Quanto menor o ângulo, mais similares semanticamente são os textos. |
| **HNSW (Hierarchical Navigable Small World)** | Algoritmo de indexação para vetores que permite buscas de K-vizinhos mais próximos de forma aproximada e muito eficiente. |
| **LLM (Large Language Model)** | Modelo de linguagem de grande escala. No Vox AI: Gemini AI. |
| **Streamlit** | Framework Python para criar apps web sem HTML/CSS/JS. Gerencia estado via `st.session_state`. |
| **Conventional Commits** | Especificação para mensagens de commit estruturadas. Permite geração automática de changelogs. |
| **Git Cliff** | Ferramenta que lê o histórico de commits e gera automaticamente um `CHANGELOG.md` formatado. |
| **Supabase** | Plataforma open-source de backend-as-a-service baseada em PostgreSQL. |
| **RPC (Remote Procedure Call)** | No Supabase: funções PostgreSQL customizadas chamadas como endpoints REST via `client.rpc()`. |
| **RLS (Row Level Security)** | Recurso do PostgreSQL que controla acesso a linhas individuais de uma tabela por usuário/role. |
| **uv** | Gerenciador de pacotes Python ultrarrápido (escrito em Rust) que substitui pip+venv. |
| **Session State** | Dicionário global do Streamlit (`st.session_state`) que persiste variáveis entre re-renders. |
| **system_instruction** | Texto enviado ao LLM antes de qualquer mensagem do usuário, definindo personalidade e regras. |
| **Chunk** | Fragmento de conhecimento na `knowledge_base`. Cada linha é um pedaço de informação recuperavel pelo RAG. |
| **kb_count** | Contador na `knowledge_base` incrementado automaticamente por trigger cada vez que o chunk e usado em uma resposta. Mede a utilidade real de cada fragmento de conhecimento. |
| **ETL (Extract, Transform, Load)** | Processo de mover dados da `knowledge_base_etl` (staging) para a `knowledge_base` (produção) após validação. |
| **Git Hook** | Script executado pelo Git em eventos especificos (`pre-commit`, `commit-msg`, `pre-push`). |

---