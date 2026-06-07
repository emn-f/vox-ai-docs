# Testes Automatizados

##  `conftest.py` — Fixtures Globais (`autouse=True`)

| Fixture | O que faz |
|---|---|
| `set_dummy_env_vars` | Define variáveis de ambiente ficticias para que as funções de configuração não falhem por "chave faltando". |
| `mock_supabase_global` | Faz patch de `supabase.create_client`, retornando `MagicMock` com toda a cadeia de chamadas. |
| `mock_gemini_global` | Faz patch de `google.genai.Client`, retornando mock com respostas pre-definidas. |

## Estrutura de Diretórios dos Testes

Os testes são organizados da seguinte forma:
- `tests/unit/`: Testes unitários com mocks globais (rápidos e isolados).
- `tests/integration/`: Testes de integração reais com Supabase e Gemini (exigem conexão e chaves reais).

---

## Testes Unitários (`tests/unit/`)

### 1. `test_database_functions.py`
* **`test_salvar_sessao`**: Verifica se a tabela `sessions` foi chamada corretamente com o `session_id`.
* **`test_salvar_erro`**: Garante que o `error_id` curto é gerado e que a mensagem é enviada à tabela `error_logs`.
* **`test_salvar_report`**: Valida a persistência na tabela `user_reports`.
* **`test_get_categorias_erro`**: Garante que retorna as categorias cadastradas.
* **`test_buscar_referencias_db`**: Verifica se a RPC `match_knowledge_base` é chamada com os parâmetros corretos.
* **`test_recuperar_contexto_inteligente`**: Valida a escolha estratégica do RAG inteligente.

### 2. `test_security_check.py`
* Cobre o módulo `security_check.py`, testando a higienização de segredos e chaves de API, as tomadas de decisão baseadas nos status de retorno da IA (`[PASS]` e `[BLOCK]`) e a resiliência a falhas da API (fail-open).

### 3. `test_genai_error_handling.py`
* Cobre a lógica de interceptação de erros estruturados (`APIError`) e a renderização de alertas amigáveis para cotas (429) e indisponibilidades (503).

---

## Testes de Integração (`tests/integration/`)

### 1. `test_gemini_integration.py`
* Realiza chamadas reais de geração de conteúdo e de transcrição no Gemini utilizando a chave de API real. Pula o teste caso a chave não esteja disponível.

### 2. `test_supabase_connection.py`
* Testa a conexão real de leitura com a tabela do Supabase. Pulado se as credenciais reais de banco não forem encontradas no ambiente de execução.

---