# `git-cliff` e Geração Automática de CHANGELOG

## O que é o `git-cliff`

`git-cliff` é um gerador de CHANGELOG escrito em Rust que analisa o histórico de commits git e transforma mensagens no formato Conventional Commits em um arquivo `CHANGELOG.md` estruturado. No Vox AI, ele é invocado via a GitHub Action `orhun/git-cliff-action@v4` dentro do workflow `production_pipeline.yml`.

## O Arquivo `cliff.toml` — Configuração Completa

O `cliff.toml` na raiz do repositório define como os commits são classificados, filtrados e renderizados no CHANGELOG.

**Seção `[changelog]`:**

```toml
[changelog]
# Template Tera (Jinja2-like) para o corpo do CHANGELOG
body = """
{% if version %}\
# [{{ version | trim_start_matches(pat="v") }}] - {{ timestamp | date(format="%d/%m/%Y") }}
{% else %}\
# [Não Lançado]
{% endif %}\
{% for group, commits in commits | group_by(attribute="group") %}
## {{ group | upper_first }}
{% for commit in commits %}
- {% if commit.scope %}**{{ commit.scope }}**: {% endif %}{{ commit.message | upper_first }}\
{% if commit.breaking %} [BREAKING]{% endif %} \
([{{ commit.id | truncate(length=7, end="") }}]({{ remote.url }}/commit/{{ commit.id }}))\
{% endfor %}
{% endfor %}\n
"""

# Adiciona as mudanças do commit mais recente ao topo do arquivo
prepend = "CHANGELOG.md"

# Não inclui commits cujas mensagens sejam apenas o número da versão
trim = true
```

**Seção `[git]`:**

```toml
[git]
# Analisa apenas commits com mensagens no formato Conventional Commits
conventional_commits = true

# Ignora commits que não seguem a convenção (não bloqueia, apenas descarta)
filter_unconventional = true

# Ordena commits por data dentro de cada grupo
sort_commits = "newest"

# Padrão de tags consideradas como releases
tag_pattern = "v[0-9].*"

# Commits a IGNORAR (não aparecem no CHANGELOG):
ignore_tags = "dev-v.*"   # Tags de desenvolvimento (prefixo dev-v*)
skip_tags = "^$"

# Mapeamento tipo → grupo no CHANGELOG:
commit_parsers = [
  { message = "^feat",     group = "Funcionalidades"  },
  { message = "^fix",      group = "Correções de Bug" },
  { message = "^docs",     group = "Documentação"     },
  { message = "^perf",     group = "Performance"      },
  { message = "^refactor", skip = true                },
  { message = "^style",    skip = true                },
  { message = "^test",     skip = true                },
  { message = "^ci",       skip = true                },
  { message = "^build",    skip = true                },
  { message = "^chore",    skip = true                },
  { message = "\\[skip ci\\]", skip = true            },
]
```


## Fluxo de Geração no `production_pipeline.yml`

A geração do CHANGELOG ocorre no **Job 2** (`release_prod`) do pipeline de produção, executado apenas após os testes passarem. O fluxo completo:

**Etapa 1 — Cálculo da próxima versão:**

```bash
# Busca a última tag v* (release de produção)
LAST_TAG=$(git tag --list 'v*' --sort=-version:refname | head -n 1)
# Resultado exemplo: v3.2.14

# Extrai MAJOR, MINOR e PATCH
IFS='.' read -r MAJOR MINOR PATCH <<< "${LAST_TAG#v}"
# MAJOR=3, MINOR=2, PATCH=14

# Incrementa PATCH (estratégia padrão do pipeline automático)
NEW_PATCH=$((PATCH + 1))
NEW_VERSION="v${MAJOR}.${MINOR}.${NEW_PATCH}"
# Resultado: v3.2.15
```

> Para incrementar MINOR ou MAJOR, usa-se o workflow `manual_release.yml`.

**Etapa 2 — Invocação do `git-cliff`:**

```yaml
- name: "📝 Gerar CHANGELOG"
  uses: orhun/git-cliff-action@v4
  with:
    config: cliff.toml
    args: --verbose --tag ${{ env.NEW_VERSION }}
  env:
    OUTPUT: CHANGELOG.md
    GITHUB_REPO: ${{ github.repository }}
```

O `git-cliff` analisa **todos os commits desde a tag anterior** (`LAST_TAG..HEAD`), filtra pelos `commit_parsers` do `cliff.toml`, e renderiza o template Tera, **adicionando a nova seção no topo** do `CHANGELOG.md` existente.

**Etapa 3 — Commit e tag com `[skip ci]`:**

```bash
git config user.name  "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"

git add CHANGELOG.md
git commit -m "chore(release): $NEW_VERSION [skip ci]"

git tag "$NEW_VERSION"
git push origin main --follow-tags
```

A flag `[skip ci]` na mensagem do commit é reconhecida pelo `commit_parser` do `cliff.toml` (regra `skip = true`) E também pelo próprio `production_pipeline.yml` via filtro de path:

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - 'CHANGELOG.md'
```

Isso cria um **duplo mecanismo anti-loop**: o commit do CHANGELOG não reaparece no histórico do próximo CHANGELOG, e o workflow não é re-acionado.

## Estratégia de Versioning por Branch

| Branch | Padrão de Tag | Gerador | Trigger |
|---|---|---|---|
| `main` | `v{MAJOR}.{MINOR}.{PATCH}` | `production_pipeline.yml` | Automático em todo push |
| `develop` | `dev-v{MAJOR}.{MINOR}.{PATCH}` | `versioning_dev.yml` | Automático em todo push |
| Qualquer | `v{X}.{Y}.{Z}` | `manual_release.yml` | Manual via `workflow_dispatch` |

As tags `dev-v*` são ignoradas pelo `git-cliff` (`ignore_tags = "dev-v.*"`), então nunca aparecem no CHANGELOG de produção.

## Exemplo de Saída do CHANGELOG

```markdown
# [3.2.15] - 21/04/2026

## Funcionalidades
- **tts**: Adiciona botão de conversão de texto em áudio na resposta ([a1b2c3d](https://github.com/emn-f/vox-ai/commit/a1b2c3d...))
- Suporte a upload de arquivos PDF na base de conhecimento ([7f8e9d0](https://github.com/emn-f/vox-ai/commit/7f8e9d0...))

## Correções de Bug
- **ui**: Corrige crash da sidebar no mobile com texto longo ([2c3d4e5](https://github.com/emn-f/vox-ai/commit/2c3d4e5...))

## Documentação
- Atualiza README com instruções de setup local ([9a8b7c6](https://github.com/emn-f/vox-ai/commit/9a8b7c6...))
```

---