# Testes Automatizados

##  `conftest.py` — Fixtures Globais (`autouse=True`)

| Fixture | O que faz |
|---|---|
| `set_dummy_env_vars` | Define variáveis de ambiente ficticias para que as funções de configuração não falhem por "chave faltando". |
| `mock_supabase_global` | Faz patch de `supabase.create_client`, retornando `MagicMock` com toda a cadeia de chamadas. |
| `mock_gemini_global` | Faz patch de `google.genai.Client`, retornando mock com respostas pre-definidas. |

## 9.2 `test_database_functions.py` — Testes Unitarios do Banco

- `salvar_sessao`: verifica se a tabela correta (`sessions`) foi chamada com o `session_id` correto.
- `salvar_erro`: verifica se o `error_id` e valido e se a mensagem foi passada corretamente.
- `salvar_report`: verifica retorno `True` e a chamada a tabela `user_reports`.
- `get_categorias_erro`: verifica se retorna a lista de categorias mockada.
- `buscar_referencias_db`: verifica se a RPC `match_knowledge_base` e chamada com os parâmetros corretos.
- `recuperar_contexto_inteligente`: testa estratégia mista (2 tópicos diferentes) verificando se o contexto contem ambas as descrições e a fonte e "Tópicos mistos".

## 9.3 `test_gemini_integration.py`

Teste de integração REAL com a API Gemini. Busca a chave em `secrets.toml` ou variáveis de ambiente. Se não encontrar: `pytest.skip`.

> **ATENÇÃO:** Este teste faz chamadas reais a API e consome cota do Gemini. Não deve rodar em CI sem credenciais configuradas.

## 9.4 `test_supabase_connection.py`

Teste de integração REAL com Supabase. Pulado automaticamente se as credenciais não forem encontradas. Testa fazendo `SELECT` na tabela `report_categories`.

## 9.5 `test_security_check.py`

Cobre o modulo `security_check.py`: sanitização de segredos, comportamentos com diferentes respostas da IA e falha aberta em caso de exceção da API.

---