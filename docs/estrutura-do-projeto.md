---
search:
  exclude: true
---

# Estrutura de Pastas e Arquivos

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
│   ├── security_check.py   # Revisão de código com IA + scan de segredos
│   ├── validate_commit_msg.py  # Hook de validação de Conventional Commits
│   ├── install_hooks.py    # Instala os Git Hooks localmente
│   └── requirements-gatekeep.txt
|
├── pages/
│   ├── dashboard.js        # Lógica JS do Dashboard (fetch Supabase)
│   └── dashboard.css       # Estilos do Dashboard
|
├── scripts/
│   ├── gerar_embedding.py  # CLI para reindexar a base de conhecimento
│   ├── utilitário.py       # Inserção manual na KB
│   └── supabase_migration_auto.bat  # Automação de migrations (Windows)
|
├── src/                    # Código-fonte principal
│   ├── app/
│   │   └── ui.py           # Componentes Streamlit (sidebar, dialog, CSS)
│   ├── core/
│   │   ├── database.py     # Conexão e operações com Supabase
│   │   ├── genai.py        # Integração com Google Gemini (LLM + embeddings)
│   │   ├── semantica.py    # Orquestração do pipeline RAG
│   │   └── chat.py         # Processamento de prompt + streaming
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
│   ├── conftest.py                    # Fixtures globais (mocks)
│   ├── test_database_functions.py     # Testes unitários do database.py
│   ├── test_gemini_integration.py     # Teste de integração real com Gemini
│   ├── test_supabase_connection.py    # Teste de conexão real com Supabase
│   └── test_security_check.py         # Testes do sistema de gatekeep
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