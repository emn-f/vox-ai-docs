# 3. Stack Tecnológica e Ferramentas

## 3.1 Core da Aplicação

| Ferramenta | Versão | Contexto no Vox AI | Docs Oficiais |
|---|---|---|---|
| Python | 3.13 | Linguagem principal do projeto inteiro | [docs.python.org/3.13](https://docs.python.org/3.13/) |
| Streamlit | 1.52.2 | Framework da interface web (chat, sidebar, botões, estado de sessão) | [docs.streamlit.io](https://docs.streamlit.io/) |
| uv | latest | Gerenciador de pacotes e ambientes virtuais ultrarrápido (substitui pip+venv) | [docs.astral.sh/uv](https://docs.astral.sh/uv/) |
| gTTS | 2.5.4 | Text-to-Speech: converte respostas em áudio MP3 com voz pt-BR | [gtts.readthedocs.io](https://gtts.readthedocs.io/en/latest/) |
| python-dotenv | 1.2.1 | Carregamento de variáveis de ambiente para desenvolvimento local | [pypi.org/project/python-dotenv](https://pypi.org/project/python-dotenv/) |

## 3.2 Inteligência Artificial

| Ferramenta | Contexto no Vox AI | Docs Oficiais |
|---|---|---|
| Google GenAI SDK (google-genai) | Biblioteca oficial para acessar a API do Gemini. Usada para gerar respostas (LLM) e embeddings (RAG) | [ai.google.dev/gemini-api/docs](https://ai.google.dev/gemini-api/docs?hl=pt-br) |
| Gemini Flash (gemini-3-flash-preview) | Modelo LLM principal. Recebe o system prompt + contexto RAG e gera as respostas em streaming | [ai.google.dev/gemini-api/docs/models](https://ai.google.dev/gemini-api/docs/models?hl=pt-br) |
| Gemini Embeddings (gemini-embedding-001) | Modelo de embeddings. Converte textos em vetores de 1536 dimensões para busca semântica | [ai.google.dev/gemini-api/docs/embeddings](https://ai.google.dev/gemini-api/docs/embeddings?hl=pt-br) |

## 3.3 Banco de Dados

| Ferramenta | Contexto no Vox AI | Docs Oficiais |
|---|---|---|
| Supabase | Backend-as-a-Service completo: PostgreSQL, API REST automática, RLS, SDK Python e dashboard | [supabase.com/docs](https://supabase.com/docs) |
| PostgreSQL 17 | Banco relacional onde ficam todas as tabelas, funções, triggers e índices | [postgresql.org/docs](https://www.postgresql.org/docs/17/index.html) |
| pgvector (extensão vector) | Adiciona o tipo vector, operador de distância cosseno (<=>) e índices HNSW para busca semântica | [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector) |
| Supabase CLI | Gerencia o projeto localmente, gera e aplica migrations via `db diff`/`push` | [supabase.com/docs/guides/cli](supabase.com/docs/guides/cli) |

## 3.4 DevOps e CI/CD

| Ferramenta | Contexto no Vox AI | Docs Oficiais |
|---|---|---|
| GitHub Actions | Plataforma de CI/CD. Orquestra os 9 workflows: testes, releases, deploys e seguranca | [docs.github.com/actions](https://docs.github.com/pt/actions) |
| Git Cliff (git-cliff) | Gerador de CHANGELOG.md automático a partir de Conventional Commits e tags Git | [git-cliff.org/docs](git-cliff.org/docs) |
| Pytest | Framework de testes Python. Roda testes unitários e de integração no pipeline | [docs.pytest](https://docs.pytest.org/en/stable/) |
| Hugging Face Spaces | Plataforma de deploy alternativa (mirror). App Streamlit sincronizado após cada release | [huggingface.co/docs/hub/spaces](huggingface.co/docs/hub/spaces) |
| GitHub Pages | Hospedagem estática do Dashboard de transparencia pública | [docs.github.com/pages](docs.github.com/pages) |

---