---
search:
  exclude: true
---

# Estrutura de Pastas e Arquivos

> Última atualização em 07/06/2026

```
vox-ai/
├── .github/
|   ├── workflows/          # Workflows GitHub Actions
|   ├── ISSUE_TEMPLATE/     # Templates de bug report e feature request
|   ├── PULL_REQUEST_TEMPLATE.md
|   ├── CODE_OF_CONDUCT.md
|   ├── CONTRIBUTING.md
|   ├── SECURITY.md
|   └── SUPPORT.md
|
├── data/prompts/
│   ├── system_prompt.py    # System prompt do Vox (instruções ao LLM)
│   └── ui_content.py       # Textos da interface (saudação, sidebar)
|
├── docs/
│   ├── ARCHITECTURE.md
│   ├── ASSETS.md
│   ├── CONVENTIONAL_COMMITS.md
│   ├── CONVENTIONAL_MIGRATIONS.md
│   └── legal/
│       ├── PRIVACY_POLICY.md
│       └── TERMS_OF_USE.md
|
├── gatekeep/               # Sistema de segurança e validação
│   ├── security_check.py   # Orquestrador local do Gatekeeper (pre-commit/pre-push)
│   ├── validate_commit_msg.py # Validador de Conventional Commits
│   ├── colors.py           # Utilitários de colorização de logs
│   ├── config_loader.py    # Leitor do secrets.toml usando tomllib
│   ├── db_check.py         # Validador de migrations e conexões Supabase
│   ├── secrets_check.py    # Scanner de chaves de API e tokens locais
│   ├── ai_review.py        # Módulo de code review automático via Gemini API
│   └── requirements-gatekeep.txt
|
├── pages/
│   ├── dashboard.js        # Lógica JS do Dashboard (fetch Supabase)
│   └── dashboard.css       # Estilos do Dashboard
|
├── scripts/
│   ├── gerar_embedding.py  # CLI para reindexar a base de conhecimento
│   ├── install_hooks.py    # Instala os Git Hooks localmente (.git/hooks)
│   └── old/                # Scripts utilitários legados
|
├── src/                    # Código-fonte principal
│   ├── app/
│   │   └── ui.py           # Componentes Streamlit (sidebar, dialog, CSS)
│   ├── core/
│   │   ├── db/             # Camada modularizada de acesso ao Supabase
│   │   │   ├── client.py     # Inicializador singleton do cliente Supabase
│   │   │   ├── sessions.py   # Registro e limpeza de sessões
│   │   │   ├── logs.py       # Registro de chats e logs de erros
│   │   │   ├── reports.py    # Registro de denúncias de bugs
│   │   │   └── retrieval.py  # Busca vetorial pgvector e RAG inteligente
│   │   ├── database.py     # Facade (Fachada) de compatibilidade retroativa
│   │   ├── genai.py        # Integração com Google Gemini (LLM + transcrição)
│   │   └── semantica.py    # Orquestração do pipeline RAG (Gemini Embedding)
│   ├── config.py           # Configurações globais, constantes, get_secret()
│   ├── utils.py            # Utilitários: TTS, versão git, limpeza de texto
│   └── external_links.py   # Centralização de todas as URLs externas
|
├── static/
|   └──css/
|      └──style.css    # Estilos globais da interface Streamlit
|
├── supabase/
│   ├── config.toml         # Configuração do Supabase CLI local
│   └── migrations/
│       ├── 20260410192141_remote_schema.sql    # Schema inicial completo
│       └── 20260418194905_alter_vetor_1536.sql # Resize do vetor para 1536d
|
├── tests/
│   ├── conftest.py                    # Fixtures globais e mocks do Pytest
│   ├── unit/                          # Testes unitários (rápidos com mocks)
│   │   ├── test_database_functions.py
│   │   ├── test_genai_error_handling.py
│   │   └── test_security_check.py
│   └── integration/                   # Testes de integração (reais com rede)
│       ├── test_gemini_integration.py
│       └── test_supabase_connection.py
│
├── .python-version         # Versão do Python fixada (3.13)
├── CHANGELOG.md            # Histórico de versões gerado automaticamente
├── cliff.toml              # Config do gerador de changelog (git-cliff)
├── LICENSE                 # Licença GNU GPLv3
├── main.py                 # Entry point CLI minimal
├── pyproject.toml          # Configuração do projeto e dependências (uv)
├── README.md               # Documentação pública
├── requirements.txt        # Dependências geradas automaticamente pelo uv
├── startup_patch.py        # Patch de compatibilidade com torch.classes
├── uv.lock                 # Lock file do gerenciador uv
└── vox_ai.py               # Ponto de entrada principal (Streamlit app)

```

---
