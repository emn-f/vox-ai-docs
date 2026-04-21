# 12. Como Rodar Localmente (Setup para Novos Colaboradores)

## 12.1 Pré-requisitos

- Python 3.13 instalado
- `uv` instalado (gerenciador de pacotes)
- Git instalado
- Docker Desktop instalado (para uso do Supabase CLI localmente)
- Conta no Google AI Studio para obter a `GEMINI_API_KEY`

## 12.2 Passo a Passo

**1. Clone o repositório:**
```bash
git clone https://github.com/SEU-USUARIO/vox-ai.git
cd vox-ai
```

**2. Instale as dependências:**
```bash
uv sync
```

**3. Crie o arquivo de secrets:**
```toml
# .streamlit/secrets.toml na raiz do projeto

GEMINI_API_KEY = "SUA_CHAVE_AQUI"

[supabase]
url = "SUA_URL_SUPABASE"
key = "SUA_CHAVE_ANON_SUPABASE"
```

> **Dica:** Sem as credenciais do Supabase, o projeto roda normalmente mas sem o sistema RAG. A IA responderá usando apenas seu conhecimento geral.

**4. Instale os Git Hooks:**
```bash
python scripts/install_hooks.py
```

**5. Rode o projeto:**
```bash
uv run streamlit run vox_ai.py
```

**6.** Acesse em: `http://localhost:8501`

## 12.3 Adicionar Novas Dependências

```bash
# Adicionar biblioteca:
uv add nome-da-biblioteca

# Atualizar requirements.txt (usado no deploy):
uv export --no-hashes > requirements.txt
```

## 12.4 Rodando os Testes

```bash
# Todos os testes unitários:
pytest

# Testes de integração (requerem credenciais reais):
pytest tests/test_gemini_integration.py
pytest tests/test_supabase_connection.py
```

---
