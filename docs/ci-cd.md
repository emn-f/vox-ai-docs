# Actions do Projeto

## Mapa Geral e Dependências

```
Push → develop
    └─▶ versioning_dev.yml      (tag dev-v*)
    └─▶ [PR aberta para develop/main]
              └─▶ security_review.yml   (security_check.py --mode pre-push)

Push → main (qualquer arquivo exceto CHANGELOG.md)
    └─▶ production_pipeline.yml
              ├─▶ Job: test             (pytest)
              └─▶ Job: release_prod     (git-cliff + tag + CHANGELOG)
                        └─ (trigger por_calls)
                                  ├─▶ deploy_hugging_face.yml
                                  └─▶ deploy_pages.yml

Push → main (apenas CHANGELOG.md)
    └─▶ deploy_pages.yml        (deploy do dashboard)
    └─▶ changelog_dev.yml       (sync CHANGELOG → develop)

Push → supabase/migrations/**
    └─▶ deploy_db.yml           (supabase db push)

workflow_dispatch
    ├─▶ manual_release.yml      (release manual major/minor/patch)
    └─▶ push_homo.yml           (atualiza branch homo)
```

## `production_pipeline.yml` — Pipeline Principal

**Arquivo:** `.github/workflows/production_pipeline.yml`
**Trigger:** `push` para `main`, excluindo mudanças apenas em `CHANGELOG.md`

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - 'CHANGELOG.md'
```

**Permissões necessárias:**

```yaml
permissions:
  contents: write   # Para criar tags e commitar CHANGELOG
```

### Job 1: `test`

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: "🐍 Setup Python 3.13"
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"

      - name: "📦 Instalar uv"
        run: pip install uv

      - name: "📥 Instalar dependências"
        run: uv pip install -r requirements.txt --system

      - name: "🧪 Executar pytest"
        run: pytest tests/ -v
        env:
          GEMINI_API_KEY:       ${{ secrets.GEMINI_API_KEY }}
          SUPABASE_URL:         ${{ secrets.SUPABASE_URL }}
          SUPABASE_ANON_KEY:    ${{ secrets.SUPABASE_ANON_KEY }}
```

> **Por que `--system` no uv?** O ambiente do GitHub Actions não tem um venv ativo. `--system` instrui o `uv` a instalar no Python do sistema em vez de tentar criar/ativar um venv.

### Job 2: `release_prod`

```yaml
  release_prod:
    needs: test          # Só executa se test passou
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # CRÍTICO: busca TODO o histórico de tags

      - name: "🔢 Calcular próxima versão"
        id: version
        run: |
          LAST=$(git tag --list 'v*' --sort=-version:refname | head -n 1)
          # Fallback se não houver nenhuma tag ainda
          LAST=${LAST:-v0.0.0}
          IFS='.' read -r MAJOR MINOR PATCH <<< "${LAST#v}"
          echo "new_version=v${MAJOR}.${MINOR}.$((PATCH+1))" >> $GITHUB_OUTPUT

      - name: "📝 Gerar CHANGELOG com git-cliff"
        uses: orhun/git-cliff-action@v4
        with:
          config: cliff.toml
          args: --verbose --tag ${{ steps.version.outputs.new_version }}
        env:
          OUTPUT: CHANGELOG.md
          GITHUB_REPO: ${{ github.repository }}

      - name: "💾 Commit CHANGELOG e criar tag"
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add CHANGELOG.md
          git commit -m "chore(release): ${{ steps.version.outputs.new_version }} [skip ci]"
          git tag ${{ steps.version.outputs.new_version }}
          git push origin main --follow-tags
```

> **`fetch-depth: 0`:** Sem isso, o `actions/checkout` faz um shallow clone (apenas o commit mais recente). O `git-cliff` e o cálculo da versão precisam de **todo o histórico de tags** para funcionar corretamente.


## `security_review.yml` — Code Review com IA

**Arquivo:** `.github/workflows/security_review.yml`
**Trigger:** `pull_request` apontando para `develop` ou `main`

```yaml
on:
  pull_request:
    branches: [develop, main]
```

**Diferencial técnico:** Este workflow usa um `requirements.txt` **isolado** (`gatekeep/requirements-gatekeep.txt`) para instalar apenas as dependências necessárias para o script de segurança, sem instalar as dependências pesadas da aplicação principal (Streamlit, etc.).

```yaml
jobs:
  security-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          # fetch-depth: 0 necessário para que o git diff
          # contra GITHUB_BASE_REF funcione corretamente

      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"

      - name: "📦 Instalar deps do gatekeep"
        run: pip install -r gatekeep/requirements-gatekeep.txt

      - name: "🔒 Executar security_check"
        run: python gatekeep/security_check.py --mode pre-push
        env:
          GEMINI_API_KEY:  ${{ secrets.GEMINI_API_KEY }}
          GITHUB_ACTIONS:  "true"
          # GITHUB_BASE_REF é injetado automaticamente pelo GitHub
          # em eventos de pull_request (ex: "develop" ou "main")
```

**Comportamento em fork PRs:** O GitHub não injeta secrets em PRs de forks por segurança. Nesses casos, `GEMINI_API_KEY` será vazio e o script entra em modo `fail-open`, permitindo a PR sem revisão da IA.


## `deploy_db.yml` — Migrations do Banco de Dados

**Arquivo:** `.github/workflows/deploy_db.yml`
**Trigger:** Push para `main` que modifique arquivos em `supabase/migrations/**`

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'supabase/migrations/**'
```

**Fluxo de execução:**

```yaml
jobs:
  deploy-migrations:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: "🛠️ Instalar Supabase CLI"
        uses: supabase/setup-cli@v1
        with:
          version: latest

      - name: "🔗 Linkar ao projeto de produção"
        run: supabase link --project-ref ${{ secrets.SUPABASE_PROJECT_ID }}
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}

      - name: "🚀 Aplicar migrations"
        run: supabase db push
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
```

**Como o Supabase CLI rastreia migrations:**

O CLI mantém uma tabela interna `supabase_migrations.schema_migrations` no banco de produção. Ao executar `supabase db push`, ele compara os arquivos `.sql` presentes em `supabase/migrations/` com os timestamps já registrados nessa tabela, e executa **apenas os arquivos novos** em ordem cronológica.

> **⚠️ ATENÇÃO:** O `supabase db push` executa SQL diretamente em produção. Não há rollback automático. Uma migration mal-escrita pode corromper dados ou bloquear a aplicação.


## `deploy_hugging_face.yml` — Espelhamento no HF Spaces

**Arquivo:** `.github/workflows/deploy_hugging_face.yml`
**Trigger:** Conclusão bem-sucedida do `production_pipeline.yml`

```yaml
on:
  workflow_run:
    workflows: ["production_pipeline"]
    types: [completed]
    branches: [main]
```

**Técnica do branch orfão:**

O Hugging Face Spaces espera um repositório git com apenas os arquivos do app — sem o histórico de desenvolvimento, sem binários pesados da pasta `docs/imgs`. Para isso, o workflow cria um snapshot limpo:

```yaml
steps:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0

  - name: "🧹 Preparar snapshot limpo"
    run: |
      # Cria branch orfão (sem histórico)
      git checkout --orphan deploy-snapshot

      # Remove binários pesados da documentação
      git rm -rf docs/imgs/

      # Adiciona tudo o que restou
      git add -A
      git commit -m "deploy: snapshot for HF Spaces"

  - name: "🚀 Push para Hugging Face"
    run: |
      git remote add huggingface \
        https://emn-f:${{ secrets.HF_TOKEN }}@huggingface.co/spaces/emn-f/vox-ai
      git push huggingface deploy-snapshot:main --force
```

O `--force` é necessário porque cada deploy é um commit único sem histórico relacionado ao anterior. O HF Spaces detecta o push e reinicia o container automaticamente.


## `deploy_pages.yml` — Dashboard de Transparência

**Arquivo:** `.github/workflows/deploy_pages.yml`
**Trigger:** Conclusão do `production_pipeline.yml` OU push direto no `CHANGELOG.md` da `main`

```yaml
on:
  workflow_run:
    workflows: ["production_pipeline"]
    types: [completed]
    branches: [main]
  push:
    branches: [main]
    paths:
      - 'CHANGELOG.md'
```

**Injeção segura de credenciais:**

O arquivo `pages/dashboard.js` no repositório contém **placeholders** em vez de credenciais reais:

```javascript
// Em pages/dashboard.js (no repositório)
const SUPABASE_URL = "__SUPABASE_URL_DEV__";
const SUPABASE_ANON_KEY = "__SUPABASE_ANON_KEY_DEV__";
```

O workflow substitui os placeholders antes do deploy usando `sed`:

```yaml
- name: "🔑 Injetar credenciais Supabase Dev"
  run: |
    sed -i \
      's|__SUPABASE_URL_DEV__|${{ secrets.SUPABASE_URL_DEV }}|g' \
      pages/dashboard.js
    sed -i \
      's|__SUPABASE_ANON_KEY_DEV__|${{ secrets.SUPABASE_ANON_KEY_DEV }}|g' \
      pages/dashboard.js
```

> **Por que `DEV` e não produção?** O Dashboard é público (GitHub Pages). Usar a `ANON_KEY` do banco de desenvolvimento limita o impacto de eventuais abusos — os dados exibidos são os de homologação, não de produção.


## `versioning_dev.yml` — Tagging na Branch `develop`

**Arquivo:** `.github/workflows/versioning_dev.yml`
**Trigger:** Push para `develop`

**Objetivo:** Criar uma tag `dev-v*` em cada commit da branch de desenvolvimento para rastreabilidade, sem gerar CHANGELOG ou release notes.

```yaml
jobs:
  tag-dev:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: "🔢 Calcular tag dev"
        id: devtag
        run: |
          # Busca a última tag de produção para manter o MAJOR.MINOR em sync
          PROD_TAG=$(git tag --list 'v*' --sort=-version:refname | head -n 1)
          PROD_TAG=${PROD_TAG:-v0.0.0}
          IFS='.' read -r MAJOR MINOR PATCH <<< "${PROD_TAG#v}"

          # Busca a última tag dev para incrementar o PATCH
          LAST_DEV=$(git tag --list 'dev-v*' --sort=-version:refname | head -n 1)
          if [ -z "$LAST_DEV" ]; then
            NEW_TAG="dev-v${MAJOR}.${MINOR}.$((PATCH+1))"
          else
            IFS='.' read -r _ _ DEV_PATCH <<< "${LAST_DEV#dev-v}"
            NEW_TAG="dev-v${MAJOR}.${MINOR}.$((DEV_PATCH+1))"
          fi
          echo "tag=$NEW_TAG" >> $GITHUB_OUTPUT

      - name: "🏷️ Criar e push da tag"
        run: |
          git tag ${{ steps.devtag.outputs.tag }}
          git push origin ${{ steps.devtag.outputs.tag }}
```

As tags `dev-v*` são visíveis no repositório e são usadas pela função `git_version()` em `src/utils.py` para exibir a versão correta no rodapé da interface quando o app está rodando a partir da branch `develop`.


## `manual_release.yml` — Release Manual

**Arquivo:** `.github/workflows/manual_release.yml`
**Trigger:** `workflow_dispatch` (acionamento manual pela interface do GitHub ou CLI)

**Input de usuário:**

```yaml
on:
  workflow_dispatch:
    inputs:
      bump_type:
        description: "Tipo de incremento de versão"
        required: true
        type: choice
        options:
          - patch
          - minor
          - major
        default: patch
```

**Lógica de cálculo por tipo:**

```bash
LAST=$(git tag --list 'v*' --sort=-version:refname | head -n 1)
LAST=${LAST:-v0.0.0}
IFS='.' read -r MAJOR MINOR PATCH <<< "${LAST#v}"

case "${{ inputs.bump_type }}" in
  patch)
    NEW_VERSION="v${MAJOR}.${MINOR}.$((PATCH+1))"
    ;;
  minor)
    NEW_VERSION="v${MAJOR}.$((MINOR+1)).0"
    ;;
  major)
    NEW_VERSION="v$((MAJOR+1)).0.0"
    ;;
esac
```

**Casos de uso para `minor` e `major`:**

- **`minor`:** Novas features significativas (ex: novo módulo de busca, nova fonte de dados na KB).
- **`major`:** Quebras de compatibilidade (ex: mudança no schema do banco que requer migração de dados, refatoração completa da API).

O restante do fluxo é idêntico ao `release_prod` do `production_pipeline.yml`: git-cliff gera o CHANGELOG, commita com `[skip ci]` e cria a tag.


## `push_homo.yml` — Atualização da Branch de Homologação

**Arquivo:** `.github/workflows/push_homo.yml`
**Trigger:** `workflow_dispatch` (manual)

**Objetivo:** Sincronizar a branch `homo` com o estado atual da `develop` para criar um ambiente de homologação estável.

```yaml
jobs:
  push-to-homo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: develop
          fetch-depth: 0

      - name: "🔄 Atualizar homo com develop"
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git push origin develop:homo --force
```

> A branch `homo` não tem proteção de branch e aceita `--force`. Ela representa um snapshot da `develop` em um momento específico escolhido pela equipe para testes de homologação.


## `changelog_dev.yml` — Sincronização do CHANGELOG

**Arquivo:** `.github/workflows/changelog_dev.yml`
**Trigger:** Push para `main` que modifique apenas o `CHANGELOG.md`

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'CHANGELOG.md'
```

**Objetivo:** Quando o `production_pipeline.yml` commita o `CHANGELOG.md` na `main`, este workflow propaga essa mudança para a `develop`, mantendo o histórico de releases acessível na branch de desenvolvimento.

```yaml
steps:
  - uses: actions/checkout@v4
    with:
      ref: develop
      fetch-depth: 0

  - name: "🔄 Cherry-pick CHANGELOG da main"
    run: |
      git config user.name  "github-actions[bot]"
      git config user.email "github-actions[bot]@users.noreply.github.com"

      # Busca o SHA do commit mais recente da main (o commit do CHANGELOG)
      CHANGELOG_COMMIT=$(git log origin/main -1 --pretty=format:"%H")

      # Cherry-pick apenas aquele commit
      git cherry-pick "$CHANGELOG_COMMIT" --no-commit

      # Commita na develop com [skip ci] para não disparar outros workflows
      git commit -m "chore: sync CHANGELOG from main [skip ci]"
      git push origin develop
```

> **Por que `cherry-pick` e não `merge`?** A `develop` pode ter commits que ainda não foram para a `main`. Um merge traria todos esses commits de volta para a `main` na próxima PR, criando um ciclo. O `cherry-pick` traz **apenas** o arquivo `CHANGELOG.md` atualizado, sem misturar históricos.


# Apêndice — Diagrama de Segredos e Variáveis de Ambiente

Todos os segredos são armazenados como **GitHub Repository Secrets** e nunca aparecem nos logs:

| Secret | Usado em | Descrição |
|---|---|---|
| `GEMINI_API_KEY` | `production_pipeline`, `security_review`, app | Chave da Google AI Studio |
| `SUPABASE_URL` | `production_pipeline`, app | URL do projeto Supabase de produção |
| `SUPABASE_ANON_KEY` | `production_pipeline`, app | Chave anônima do Supabase de produção |
| `SUPABASE_URL_DEV` | `deploy_pages` | URL do projeto Supabase de desenvolvimento |
| `SUPABASE_ANON_KEY_DEV` | `deploy_pages` | Chave anônima do Supabase de desenvolvimento |
| `SUPABASE_PROJECT_ID` | `deploy_db` | ID do projeto para o CLI do Supabase |
| `SUPABASE_ACCESS_TOKEN` | `deploy_db` | Token de acesso admin para o CLI |
| `HF_TOKEN` | `deploy_hugging_face` | Token de acesso ao Hugging Face |

---