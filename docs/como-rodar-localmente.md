# Como Rodar Localmente (Setup para Novos Colaboradores)

##  Pré-requisitos

- Python 3.13 instalado
- `uv` instalado (gerenciador de pacotes)
- Git instalado
- Docker Desktop instalado (para uso do Supabase CLI localmente)
- Conta no Google AI Studio para obter a `GEMINI_API_KEY`

## Passo a Passo

1. **Fork** este repositório.
2. **Clone** o seu fork:
    ```bash
    git clone https://github.com/SEU-USUARIO/vox-ai.git
    cd vox-ai
    ```
3.  **Instale o uv** (Gerenciador de pacote):
    ```bash
    # Windows
    powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

    # Mac/Linux
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```
4.  **Instale as dependências:**
    ```bash
    uv sync
    ```
    > **Nota:** Para adicionar novas bibliotecas, use `uv add [NOME_DA_LIB]`.
    > O arquivo `requirements.txt` é gerado automaticamente para deploy.
    > Para atualizá-lo: `uv export --no-hashes > requirements.txt`
5.  **Configure as Variáveis de Ambiente:**
    Crie um arquivo `.streamlit/secrets.toml` na raiz do projeto.
    O arquivo deve seguir este formato:

    ```toml
    GEMINI_API_KEY = "SUA_CHAVE_AQUI"
    
    [supabase]
    url = "SUA_URL_SUPABASE"
    key = "SUA_CHAVE_ANON_SUPABASE"
    ```

    > **🔒 Credenciais do Supabase (Interno):**
    > O Vox utiliza o **Supabase** para RAG e Logs. Essas credenciais não são públicas.
    > 
    > * **Sem credenciais:** <u>O projeto rodará sem conexão com a base de dados do projeto usando apenas a resposta da IA</u>. Você verá avisos de conexão no terminal, o que é esperado.
    > * **Precisa de acesso ao banco?** Se a feature que você deseja implementar depende estritamente do acesso ao banco de dados, envie um e-mail para a equipe. Podemos fornecer credenciais temporárias ou um ambiente de sandbox.
6.  **Instale os Git Hooks (Segurança):**
    Para garantir que nenhum segredo seja commitado, que o banco de dados esteja consistente e que as **mensagens de commit estejam no padrão**, instale os hooks de pré-commit:
    ```bash
    python scripts/install_hooks.py
    ```

7.  **Execute o projeto:**
    ```bash
    uv run streamlit run vox_ai.py
    ```


8. Acesse em: `http://localhost:8501`

## Adicionar Novas Dependências

```bash
# Adicionar biblioteca:
uv add nome-da-biblioteca

# Atualizar requirements.txt (usado no deploy):
uv export --no-hashes > requirements.txt
```

## Rodando os Testes

```bash
# Todos os testes unitários:
pytest

# Testes de integração (requerem credenciais reais):
pytest tests/test_gemini_integration.py
pytest tests/test_supabase_connection.py
```

---