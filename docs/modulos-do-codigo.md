# Módulos do Código — Referência Detalhada

## `vox_ai.py` — Ponto de Entrada Principal

Arquivo principal da aplicação Streamlit. Orquestra todos os outros modulos. É o único arquivo executado diretamente pelo Streamlit (configurado como `app_file` no `pyproject.toml`).

**Fluxo de inicialização:**

1. Importa `startup_patch` (sempre primeiro, para corrigir o problema do `torch.classes`).
2. Chama `configurar_pagina()` e `carregar_css()` da UI.
3. Gera o `session_id` (`uuid4()`) e o salva no Supabase se não existir no `session_state`.
4. Renderiza a sidebar com `carregar_sidebar(SIDEBAR_BODY, SIDEBAR_FOOTER)`.
5. Configura a API do Gemini com `configurar_api_gemini()`.
6. Inicializa os históricos de chat e o objeto de chat via `inicializar_chat_modelo()`.
7. Renderiza o historico de mensagens existente na sessão (`hist_exibir`).
8. Exibe a mensagem de boas-vindas (`SAUDACAO`) na primeira visita e chama `st.rerun()`.

**Fluxo de processamento de prompt:**

O arquivo gerencia dois tipos de entrada: texto via `st.chat_input` e áudio via `st.audio_input` (dentro de um `st.popover`). O ID do último áudio é salvo em `session_state` como `ultimo_audio_id` para evitar reprocessamento ao re-renderizar a página. O bloco `try` central chama `semantica()`, `gerar_resposta()`, `salvar_log_chat()` e `st.rerun()`. Em caso de exceção, chama `salvar_erro()` e exibe o `error_id` ao usuário.

---

## `src/config.py` — Configurações Globais

Centraliza todas as constantes e configurações do sistema. Importado por praticamente todos os outros módulos.

| Constante | Valor | Descrição |
|---|---|---|
| `GEMINI_MODEL_NAME` | `gemini-3.5-flash` | Modelo LLM principal para geração de respostas |
| `GEMINI_MODEL_GATEKEEP` | `gemini-3.1-flash-lite` | Modelo LLM para Code Review automático no Gatekeeper |
| `MODELO_SEMANTICO_NOME` | `gemini-embedding-001` | Modelo de embeddings para o RAG |
| `TAMANHO_VETOR_SEMANTICO` | `1536` | Dimensão do vetor (DEVE corresponder à coluna do banco) |
| `SEMANTICA_THRESHOLD` | `0.5` | Similaridade mínima para um chunk ser considerado relevante |
| `LIMITE_TEMAS` | `10` | Quantos chunks são buscados na primeira fase do RAG |
| `MAX_CHUNCK` | `25` | Máximo de chunks na Estratégia de Contexto Expandido |
| `CSS_PATH` | `static/css/style.css` | Caminho do CSS customizado injetado no Streamlit |

**`get_secret(key, default='')`:** Busca credencial com fallback robusto: (1) `st.secrets` (se disponível), (2) variáveis de ambiente com formatação padrão (`KEY_NAME` em caixa alta e sublinhados), (3) variável literal. Suporta chaves aninhadas com ponto como separador: `'supabase.url'` busca `st.secrets['supabase']['url']`.

---

## `src/core/database.py` e `src/core/db/` — Camada de Dados

O arquivo `src/core/database.py` atua como uma **Facade (Fachada) de re-exportação**, mantendo a compatibilidade retroativa para módulos antigos do projeto. Toda a implementação real e controle de acesso ao Supabase foi modularizada no pacote `src/core/db/`:

* **`src/core/db/client.py` (`get_db_client() -> Client`)**:
    * Cria e retorna o cliente Supabase singleton. Decorado com `@st.cache_resource`, garantindo uma única conexão por execução. Retorna `None` se as credenciais do banco não estiverem presentes.
* **`src/core/db/sessions.py` (`salvar_sessao(session_id)`)**:
    * Insere o registro da sessão na tabela `sessions` na inicialização do aplicativo.
* **`src/core/db/logs.py` (`salvar_log_chat(session_id, git_version, prompt, response, lista_kb_ids)`)**:
    * Salva a pergunta e resposta em `chat_logs` e registra os relacionamentos na tabela `chat_logs_kb` com suas respectivas similaridades.
    * **`salvar_erro(session_id, git_version, error_msg) -> str`**: Registra exceções no banco na tabela `error_logs` e retorna um `error_id` amigável (8 caracteres) para o usuário.
* **`src/core/db/reports.py` (`salvar_report(...) -> bool`, `get_categorias_erro() -> list`)**:
    * Permite o salvamento de denúncias de bugs/conteúdo e retorna a lista de categorias válidas.
* **`src/core/db/retrieval.py` (`buscar_referencias_db(...)`, `buscar_chunks_por_topico(...)`, `recuperar_contexto_inteligente(...)`)**:
    * **`buscar_referencias_db`**: Executa a busca vetorial usando pgvector (`match_knowledge_base`).
    * **`buscar_chunks_por_topico`**: Retorna os chunks vinculados a um determinado tema.
    * **`recuperar_contexto_inteligente`**: Decide dinamicamente a melhor estratégia de RAG: **Contexto Expandido** (tópico dominante com 3+ ocorrências) ou **Tópicos Mistos** (fallback dos top-5 chunks).

---

## `src/core/genai.py` — Integração com Gemini

* **`configurar_api_gemini() -> genai.Client`:** Cria o cliente da API Gemini. Armazena em `st.session_state.gemini_client`. Chama `st.stop()` se falhar.
* **`inicializar_chat_modelo() -> chat`:** Gerencia dois históricos: `hist` (enviado ao modelo, comeca com `INSTRUCOES`) e `hist_exibir` (exibido na UI, começa vazio). Cria objeto de chat do Gemini com `GenerateContentConfig` contendo `system_instruction=INSTRUCOES`.
* **`gerar_resposta(chat, prompt, info_adicional) -> str`:** Constroi o prompt final diferenciando o prompt do usuário do contexto interno da KB. Usa `send_message_stream()` para streaming, acumulando chunks de texto.
* **`transcrever_audio(audio_file) -> str`:** Envia áudio como bytes via `types.Part.from_bytes(mime_type='audio/mp3')`. Instrui o modelo a transcrever para português do Brasil. Retorna `None` em caso de erro.

---

## `src/core/semantica.py` — Pipeline RAG

* **`semantica(prompt) -> tuple(str, str, list)`:** Gera o embedding do prompt via `gemini-embedding-001` com `task_type='RETRIEVAL_QUERY'` e delega para `recuperar_contexto_inteligente()`. Retorna `(fonte, contexto, ids)` ou `(None, None, None)`.

> **Nota:** A distinção de `task_type` é critica: `RETRIEVAL_QUERY` para o prompt do usuário e `RETRIEVAL_DOCUMENT` para os textos da KB. O Gemini otimiza os vetores de forma diferente para cada papel, melhorando a qualidade da busca.

---

## `src/app/ui.py` — Interface de Usuário

* **`configurar_pagina()`:** Define `page_title` e `page_icon` via `st.set_page_config()` e renderiza o cabeçalho.
* **`carregar_css(path=CSS_PATH)`:** Lê o arquivo `static/css/style.css` e injeta via `st.markdown(unsafe_allow_html=True)`.
* **`@st.dialog — dialog_reportar()`:** Modal de denuncia. Busca categorias via `get_categorias_erro()`, exibe `selectbox` e `text_area`. Ao confirmar, chama `salvar_report()`.
* **`carregar_sidebar(sidebar_content, sidebar_footer)`:** Renderiza sidebar completa: HTML de `SIDEBAR_BODY` (links sociais, descrição), botões "Limpar conversa" e "Reportar", e `SIDEBAR_FOOTER` (links legais, versão git).
* **`stream_resposta(resposta) -> generator`:** Gerador que itera caractere a caractere com delay de 9ms. Usada com `st.write_stream()` para efeito de digitação.

---

## `src/utils.py` — Utilitários

* **`git_version() -> str`:** Detecta branch atual. Se `main`: busca tags `v*`. Caso contrário: busca `dev-v*`. Fallback: le versão do `CHANGELOG.md` com regex.
* **`texto_para_audio(texto) -> BytesIO`:** Converte texto em MP3 via gTTS (pt-br). Aplica `limpeza_texto()` antes para remover Markdown e caracteres especiais.
* **`limpeza_texto(texto) -> str`:** Remove caracteres não-alfanuméricos exceto pontuação básica e acentos do português via regex.

---

## `startup_patch.py`

Cria um modulo vazio e o registra em `sys.modules['torch.classes']`. Evita `ImportError` em ambientes como o Streamlit Cloud onde dependências indiretas tentam importar `torch.classes` na inicialização. **Deve ser importado como primeira linha do `vox_ai.py`.**

---

## `data/prompts/system_prompt.py`

Contém a constante `INSTRUCOES` com o `system prompt` completo. Organizado em seções: Origem e Propósito, Pilares de Conhecimento, Diretrizes de Personalidade, Regras de Ouro (nunca inventar leis/endereços, zero tolerância a pornografia, respeito absoluto a pronomes) e parceria com a Casa de Cultura Marielle Franco.

---

## `data/prompts/ui_content.py`

Define as strings da interface: `SAUDACAO` (mensagem de boas-vindas), `SIDEBAR_BODY` (HTML da sidebar com links sociais) e `SIDEBAR_FOOTER` (HTML do rodapé com links legais e versão git). Usa f-strings com URLs de `external_links.py` e `git_version()`.

---