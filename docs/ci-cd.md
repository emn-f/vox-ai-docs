# Actions do Projeto

> Última atualização em 07/06/2026

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
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.13"
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      - name: Install Dependencies
        run: uv sync
      - name: Run Tests
        run: uv run pytest
```

### Job 2: `release_prod`

```yaml
  release_prod:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.TOKEN_DEPLOY_VOX }}

      - name: Configurar Git User
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"

      - name: Calcular Próxima Versão Prod
        id: get_tag
        run: |
          LAST_TAG=$(git tag --list "v*" --sort=-v:refname | head -n 1)
          if [ -z "$LAST_TAG" ]; then
            LAST_TAG="v0.0.0"
          fi

          VERSION=${LAST_TAG#v}
          IFS='.' read -r MAJOR MINOR PATCH <<< "$VERSION"
          PATCH=$((PATCH + 1))
          NEW_TAG="v${MAJOR}.${MINOR}.${PATCH}"

          echo "Nova versão Prod: $NEW_TAG"
          echo "new_tag=$NEW_TAG" >> $GITHUB_OUTPUT

      - name: Gerar Changelog (Git Cliff)
        uses: orhun/git-cliff-action@v4
        with:
          config: cliff.toml
          args: --unreleased --tag ${{ steps.get_tag.outputs.new_tag }} --prepend CHANGELOG.md

      - name: Commitar, Taggear e Push
        run: |
          NEW_TAG="${{ steps.get_tag.outputs.new_tag }}"
          git add CHANGELOG.md
          if git diff --staged --quiet; then
            echo "🚫 Changelog não mudou."
          else
            git commit -m "chore(release): changelog atualizado para versão ${NEW_TAG} [skip ci]"
          fi
          git tag $NEW_TAG
          git remote set-url origin https://emn-f:${{ secrets.TOKEN_DEPLOY_VOX }}@github.com/${{ github.repository }}.git
          git push origin main --tags
```

> **`fetch-depth: 0`:** Sem isso, o `actions/checkout` faz um shallow clone (apenas o commit mais recente). O cálculo da tag e o `git-cliff` precisam de **todo o histórico de tags** para funcionar corretamente.


## `security_review.yml` — Code Review com IA

**Arquivo:** `.github/workflows/security_review.yml`
**Trigger:** `pull_request` apontando para `develop` ou `main` excluindo arquivos markdown.

```yaml
on:
  pull_request:
    branches: [ "main", "develop" ]
    paths-ignore:
      - '**.md'
      - 'CHANGELOG.md'
```

**Diferencial técnico:** Este workflow usa um `requirements.txt` **isolado** (`gatekeep/requirements-gatekeep.txt`) para instalar apenas as dependências necessárias para o script de segurança via `setup-uv`, sem instalar as dependências pesadas da aplicação principal.

```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.13"

      - name: Install uv
        uses: astral-sh/setup-uv@v3

      - name: Install Dependencies
        run: |
          uv pip install -r gatekeep/requirements-gatekeep.txt --system

      - name: Run Security & AI Review
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_KEY_PROD: ${{ secrets.SUPABASE_KEY_PROD }}
          HF_TOKEN: ${{ secrets.HF_TOKEN }}
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
        run: |
          python gatekeep/security_check.py --mode pre-push
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
