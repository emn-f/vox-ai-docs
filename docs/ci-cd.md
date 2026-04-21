# CI/CD e Automação (GitHub Actions)

O Vox AI possui 9 workflows do GitHub Actions que automatizam todo o ciclo de vida do software.

## Mapa dos Workflows

| Arquivo | Trigger | Descrição |
|---|---|---|
| `versioning_dev.yml` | push em `develop` | Cria tag de versão `dev-v*` no commit atual |
| `security_review.yml` | PR para `main` ou `develop` | Revisão de código com IA (Gemini) + scan de segredos |
| `deploy_db.yml` | push em `main` (pasta migrations/) | Aplica `migrations `SQL no Supabase de produção |
| `changelog_dev.yml` | push em `main` (`CHANGELOG.md`) | Sincroniza o CHANGELOG da main para a develop |
| `production_pipeline.yml` | push em `main` | Pipeline principal: testes + release automática + CHANGELOG |
| `deploy_hugging_face.yml` | Após `production_pipeline.yml` | Espelha o código no Hugging Face Spaces |
| `deploy_pages.yml` | Após `production_pipeline` / push no CHANGELOG | Injeta credenciais e faz deploy do Dashboard no GitHub Pages |
| `manual_release.yml` | `workflow_dispatch` (manual) | Cria release manual escolhendo `major/minor/patch` |
| `push_homo.yml` | `workflow_dispatch` (manual) | Atualiza a branch `homo` com o conteúdo da `develop` |

## `production_pipeline.yml` — Pipeline Principal

Acionado em todo push para `main` (exceto mudanças apenas no `CHANGELOG.md`). Tem dois jobs sequenciais:

**Job 1: `test`** — Instala Python 3.13 e uv, instala dependências via `uv pip install -r requirements.txt --system`, e roda `pytest`. Se qualquer teste falhar, o workflow para.

**Job 2: `release_prod` (needs: test)** — So roda se o `test` passou. Calcula a próxima versão incrementando o PATCH da última tag `v*` (ex: `v3.2.14` -> `v3.2.15`). Usa a action `orhun/git-cliff-action@v4` com o `cliff.toml` para gerar o CHANGELOG. Commita com `[skip ci]` para evitar loop e cria a nova tag.

> **Nota:** O commit do CHANGELOG usa `[skip ci]` para não disparar outro run do workflow e criar loop infinito.

## `security_review.yml`

Roda em toda PR que aponte para `develop` ou `main`. Executa `gatekeep/security_check.py` com `--mode pre-push`. Tem dependências isoladas em `gatekeep/requirements-gatekeep.txt`.

## `deploy_db.yml`

Acionado apenas quando arquivos em `supabase/migrations/**` são alterados. Instala o Supabase CLI, linka ao projeto de produção via `supabase link --project-ref $SUPABASE_PROJECT_ID`, e executa `supabase db push`.

> **ATENÇÃO:** As migrations são IRREVERSÍVEIS em produção. Sempre revise o SQL antes de fazer `merge` para `main`.

## `deploy_hugging_face.yml`

Cria uma branch orfão (sem historico git) para um snapshot limpo, remove binarios de `docs/imgs`, e faz `git push --force` para a main do Hugging Face Spaces usando o token `HF_TOKEN`.

## `deploy_pages.yml`

Injeta as credenciais do Supabase Dev nos placeholders `__SUPABASE_URL_DEV__` e `__SUPABASE_ANON_KEY_DEV__` do `pages/dashboard.js` usando `sed` antes do deploy. Isso permite que o Dashboard público acesse dados sem expor credenciais no código-fonte.

## `versioning_dev.yml`

Em todo push para develop, calcula e cria tag no formato `dev-v{MAJOR}.{MINOR}.{PATCH}`. Nao gera changelog — apenas marca o commit para rastreabilidade.

## `manual_release.yml`

Acionado manualmente com input de seleção: `patch`, `minor` ou `major`. Útil para releases com quebras de compatibilidade que não seriam gerados automaticamente pelo `production_pipeline`.

---