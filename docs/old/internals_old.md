# Internals Avançados — Documentação Técnica Expandida

> Repositório principal do Vox AI: [https://github.com/emn-f/vox-ai](https://github.com/emn-f/vox-ai/tree/develop)

Este documento aprofunda os mecanismos internos do Vox AI que não foram cobertos na documentação base: o fluxo completo do `security_check.py`, a arquitetura dos Git Hooks locais, o pipeline de geração de CHANGELOG com `git-cliff`, a matemática por trás do cálculo vetorial de similaridade de cosseno e a especificação técnica de cada GitHub Actions workflow.

---

## Capítulo 1 — Fluxo Completo do `security_check.py`

### 1.1 Visão Geral e Posicionamento no Sistema

O `gatekeep/security_check.py` é o núcleo do sistema de revisão de código com IA. Ele opera em **dois contextos distintos**:

| Contexto | Acionador | `--mode` | O que analisa |
|---|---|---|---|
| Git Hook local | `pre-push` no terminal do dev | `pre-push` | Diff entre `HEAD` e o branch remoto (`origin/<branch>`) |
| GitHub Actions | PR aberta para `develop` ou `main` | `pre-push` | Diff do PR completo via `GITHUB_BASE_REF` |

O script é projetado para **falhar aberto** em caso de erro de rede (`fail-open`): se a API do Gemini estiver indisponível, o push/merge é permitido para não bloquear o fluxo de desenvolvimento por razões de infraestrutura.

---

### 1.2 Fase 1 — Coleta do Diff

A primeira etapa consiste em obter o diff do código que será submetido. O script executa o seguinte raciocínio:

```python
# Detecta se está rodando no ambiente de CI (GitHub Actions)
# ou localmente (Git Hook)
if os.getenv("GITHUB_ACTIONS"):
    base_ref = os.getenv("GITHUB_BASE_REF")  # ex: "develop" ou "main"
    result = subprocess.run(
        ["git", "diff", f"origin/{base_ref}...HEAD"],
        capture_output=True, text=True
    )
else:
    # Modo local: compara HEAD com o estado remoto do branch atual
    branch = subprocess.run(
        ["git", "rev-parse", "--abbrev-ref", "HEAD"],
        capture_output=True, text=True
    ).stdout.strip()
    result = subprocess.run(
        ["git", "diff", f"origin/{branch}...HEAD"],
        capture_output=True, text=True
    )

diff_text = result.stdout
```

Se o diff estiver **vazio** (nada mudou em relação ao remoto, ou é o primeiro push), o script encerra com código `0` (sucesso) sem chamar a API.

---

### 1.3 Fase 2 — Sanitização do Diff (`sanitize_diff_for_ai`)

Antes de enviar qualquer dado para a API externa, o diff passa por uma camada de sanitização que detecta e redige padrões sensíveis via expressões regulares.

**Padrões detectados:**

```python
SECRET_PATTERNS = [
    # Chaves de API genéricas (ex: sk-..., AIza...)
    r'(?i)(api[_-]?key|apikey)\s*[=:]\s*[\'""]?([A-Za-z0-9_\-]{20,})',
    # Tokens Bearer e JWT
    r'(?i)(bearer\s+[A-Za-z0-9\-_\.]+)',
    r'eyJ[A-Za-z0-9\-_]+\.[A-Za-z0-9\-_]+\.[A-Za-z0-9\-_]+',
    # URLs de banco com credenciais embutidas
    r'(?i)(postgres|mysql|mongodb)://[^@\s]+@[^\s]+',
    # Variáveis nomeadas como password/secret/token com valor
    r'(?i)(password|passwd|secret|token|private_key)\s*[=:]\s*[\'""]?.{6,}',
    # Chaves do Supabase (formato específico: eyJ... de 100+ chars)
    r'eyJ[A-Za-z0-9+/=]{80,}',
]
```

Cada match é substituído por `[REDACTED SECRET DETECTED]`. O diff sanitizado é o único dado enviado à API — **os valores originais dos segredos nunca saem da máquina**.

> **Nota de Segurança:** A substituição acontece no texto do diff, não no arquivo original. O código-fonte local permanece inalterado.

---

### 1.4 Fase 3 — Revisão com IA (`run_ai_code_review`)

O diff sanitizado é enviado ao Gemini com um prompt de sistema especializado em revisão de segurança. O prompt instrui o modelo a:

1. Identificar segredos hardcoded, chaves de API, tokens, credenciais.
2. Detectar vulnerabilidades óbvias: SQL Injection, XSS, SSRF, execução arbitrária de código.
3. Avaliar a segurança da lógica de autenticação e autorização.
4. Verificar exposição de dados sensíveis em logs ou respostas de API.

**Formato de resposta exigido da IA:**

O modelo é instruído a responder **obrigatoriamente** em um dos dois formatos:

```
[PASS] <justificativa opcional ou sugestões>
```
ou
```
[BLOCK] <razão detalhada do bloqueio>
```

---

### 1.5 Fase 4 — Análise da Resposta e Decisão Final

A resposta da IA é analisada pela função `parse_ai_response()`, que implementa uma **matriz de decisão de 5 estados**:

```
┌─────────────────────────────────┬──────────────┬─────────────────────────────────────┐
│ Resposta da IA                  │ Resultado    │ Ação do Sistema                     │
├─────────────────────────────────┼──────────────┼─────────────────────────────────────┤
│ [PASS] sem keywords proibidas   │ ✅ APROVADO  │ Exit code 0. Push/merge prossegue.  │
│ [PASS] + sugestões de melhoria  │ ✅ APROVADO  │ Exit code 0. Sugestões logadas.     │
│ [BLOCK] com justificativa       │ ❌ BLOQUEADO │ Exit code 1. Mensagem exibida.      │
│ [PASS] + keyword proibida       │ ❌ BLOQUEADO │ Exit code 1. Mesmo com PASS.        │
│ Exceção na API (timeout, quota) │ ✅ FAIL-OPEN │ Exit code 0. Aviso logado.          │
└─────────────────────────────────┴──────────────┴─────────────────────────────────────┘
```

**Keywords proibidas** que bloqueiam mesmo um `[PASS]`:

```python
BLOCK_KEYWORDS = [
    "password", "exposed", "hardcoded", "credentials",
    "secret", "private key", "api key", "token"
]
```

A lógica é: se a IA disse `[PASS]` mas o texto da resposta contém palavras como `"hardcoded"` ou `"exposed"`, provavelmente o modelo encontrou algo suspeito mas usou o formato errado. O sistema bloqueia por precaução.

---

### 1.6 Diagrama de Fluxo Completo

```
pre-push / GitHub Actions
         │
         ▼
   Coletar git diff
         │
    diff vazio? ──── SIM ──▶ [EXIT 0] OK
         │
        NÃO
         │
         ▼
  sanitize_diff_for_ai()
  (regex → [REDACTED])
         │
         ▼
  Chamar Gemini API
  (prompt de segurança)
         │
   API falhou? ──── SIM ──▶ [EXIT 0] FAIL-OPEN + aviso
         │
        NÃO
         │
         ▼
   parse_ai_response()
         │
    ┌────┴────┐
  [PASS]    [BLOCK]
    │           │
    ▼           ▼
keyword    [EXIT 1]
proibida?  BLOQUEADO
    │
  SIM ──▶ [EXIT 1]
    │
   NÃO
    │
    ▼
[EXIT 0]
APROVADO
```

---

## Capítulo 2 — Git Hooks: Arquitetura e Implementação

### 2.1 O Sistema de Hooks Locais

O Vox AI utiliza **Git Hooks nativos** (não usa frameworks como `pre-commit` ou `husky`) para garantir qualidade antes que o código deixe a máquina do desenvolvedor. Os hooks são instalados pelo script `scripts/install_hooks.py`.

Três hooks são instalados no diretório `.git/hooks/`:

| Hook | Fase | Função |
|---|---|---|
| `commit-msg` | Ao criar mensagem de commit | Valida formato Conventional Commits |
| `pre-commit` | Antes de registrar o commit | Reservado para expansão futura |
| `pre-push` | Antes de enviar ao remoto | Executa `security_check.py` |

---

### 2.2 `install_hooks.py` — Instalação Dinâmica

O script de instalação não copia arquivos estáticos — ele **gera os scripts shell dinamicamente** para garantir que apontam para o Python correto do ambiente de execução.

**Algoritmo de detecção do Python:**

```python
def find_python():
    # Prioridade 1: venv local (.venv/bin/python)
    venv_python = Path(".venv/bin/python")
    if venv_python.exists():
        return str(venv_python.resolve())
    
    # Prioridade 2: python3 do sistema
    result = subprocess.run(["which", "python3"], capture_output=True, text=True)
    if result.returncode == 0:
        return result.stdout.strip()
    
    # Prioridade 3: python genérico
    return "python"
```

**Script gerado para o hook `pre-push`:**

```bash
#!/bin/bash
# Gerado automaticamente por install_hooks.py
# NÃO edite este arquivo manualmente

PYTHON="/caminho/absoluto/.venv/bin/python"
SCRIPT="gatekeep/security_check.py"

if [ ! -f "$SCRIPT" ]; then
    echo "[HOOK] security_check.py não encontrado. Pulando."
    exit 0
fi

$PYTHON $SCRIPT --mode pre-push
exit $?
```

**Script gerado para o hook `commit-msg`:**

```bash
#!/bin/bash
# Gerado automaticamente por install_hooks.py

PYTHON="/caminho/absoluto/.venv/bin/python"
COMMIT_MSG_FILE="$1"  # Git passa o path do arquivo com a mensagem

$PYTHON gatekeep/validate_commit_msg.py "$COMMIT_MSG_FILE"
exit $?
```

O script define **permissão executável** (`chmod +x`) automaticamente após criar cada arquivo. A instalação é idempotente — pode ser executada múltiplas vezes sem efeitos colaterais.

---

### 2.3 `validate_commit_msg.py` — Hook `commit-msg`

Este hook valida que toda mensagem de commit segue a especificação **Conventional Commits** antes de registrar o commit no histórico git.

**Regex de validação:**

```python
CONVENTIONAL_COMMIT_PATTERN = re.compile(
    r'^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)'
    r'(\(.+\))?'   # escopo opcional entre parênteses
    r'!?'          # "!" opcional para BREAKING CHANGE
    r': '          # dois-pontos e espaço obrigatórios
    r'.+$',        # descrição: ao menos um caractere
    re.MULTILINE
)
```

**Lógica de bypass automático:**

O hook pula a validação para commits gerados automaticamente pelo próprio git, evitando falsos positivos em operações de manutenção:

```python
def should_skip(msg: str) -> bool:
    skip_prefixes = [
        "Merge branch",
        "Merge pull request",
        "Revert \"",
        "Initial commit",
    ]
    return any(msg.startswith(prefix) for prefix in skip_prefixes)
```

**Exemplos de comportamento:**

```bash
# ✅ VÁLIDO — tipo simples
git commit -m "feat: adiciona suporte a áudio no chat"

# ✅ VÁLIDO — com escopo
git commit -m "fix(ui): corrige quebra da sidebar no mobile"

# ✅ VÁLIDO — breaking change
git commit -m "refactor(db)!: muda schema de knowledge_base"

# ❌ INVÁLIDO — sem tipo (bloqueia o commit)
git commit -m "atualiza código da sidebar"
# Saída: [COMMIT-MSG] ERRO: Mensagem não segue Conventional Commits.

# ❌ INVÁLIDO — tipo em maiúsculas
git commit -m "FEAT: nova feature"
# Saída: [COMMIT-MSG] ERRO: Tipos devem ser minúsculos.

# ✅ BYPASS automático — merge commit
git merge develop
# Mensagem: "Merge branch 'develop' into main" → hook pula validação
```

---

### 2.4 Hook `pre-push` — Gatilho do Security Check

O hook `pre-push` é o ponto de integração entre o fluxo local do desenvolvedor e o `security_check.py`. Ele é invocado **após** o `git push` ser chamado mas **antes** de qualquer dado ser enviado ao servidor remoto.

**Sequência de eventos:**

```
git push origin develop
       │
       ▼
 Git executa .git/hooks/pre-push
       │
       ▼
 Shell invoca: python gatekeep/security_check.py --mode pre-push
       │
       ├── [EXIT 0] → Git prossegue com o push
       │
       └── [EXIT 1] → Git aborta o push
                      Exibe: "error: failed to push some refs"
```

**Variáveis disponíveis para o hook:**

O Git injeta informações sobre o push via stdin do hook no formato:
```
<local-ref> <local-sha1> <remote-ref> <remote-sha1>
```

O script usa o `remote-ref` para determinar contra qual branch remoto calcular o diff.

---

### 2.5 Hook `pre-commit` (Reservado)

O hook `pre-commit` é instalado mas implementado como um **no-op** (exit 0 imediato) na versão atual. Sua presença serve como ponto de extensão para futuras validações que devem ocorrer antes de finalizar o commit, como:

- Verificação de cobertura de testes mínima
- Linting com `ruff` ou `flake8`
- Verificação de arquivos grandes acidentalmente adicionados

---

## Capítulo 3 — `git-cliff` e Geração Automática de CHANGELOG

### 3.1 O que é o `git-cliff`

`git-cliff` é um gerador de CHANGELOG escrito em Rust que analisa o histórico de commits git e transforma mensagens no formato Conventional Commits em um arquivo `CHANGELOG.md` estruturado. No Vox AI, ele é invocado via a GitHub Action `orhun/git-cliff-action@v4` dentro do workflow `production_pipeline.yml`.

---

### 3.2 O Arquivo `cliff.toml` — Configuração Completa

O `cliff.toml` na raiz do repositório define como os commits são classificados, filtrados e renderizados no CHANGELOG.

**Seção `[changelog]`:**

```toml
[changelog]
# Template Tera (Jinja2-like) para o corpo do CHANGELOG
body = """
{% if version %}\
## [{{ version | trim_start_matches(pat="v") }}] - {{ timestamp | date(format="%d/%m/%Y") }}
{% else %}\
## [Não Lançado]
{% endif %}\
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
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

---

### 3.3 Fluxo de Geração no `production_pipeline.yml`

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

---

### 3.4 Estratégia de Versioning por Branch

| Branch | Padrão de Tag | Gerador | Trigger |
|---|---|---|---|
| `main` | `v{MAJOR}.{MINOR}.{PATCH}` | `production_pipeline.yml` | Automático em todo push |
| `develop` | `dev-v{MAJOR}.{MINOR}.{PATCH}` | `versioning_dev.yml` | Automático em todo push |
| Qualquer | `v{X}.{Y}.{Z}` | `manual_release.yml` | Manual via `workflow_dispatch` |

As tags `dev-v*` são ignoradas pelo `git-cliff` (`ignore_tags = "dev-v.*"`), então nunca aparecem no CHANGELOG de produção.

---

### 3.5 Exemplo de Saída do CHANGELOG

```markdown
## [3.2.15] - 21/04/2026

### Funcionalidades
- **tts**: Adiciona botão de conversão de texto em áudio na resposta ([a1b2c3d](https://github.com/emn-f/vox-ai/commit/a1b2c3d...))
- Suporte a upload de arquivos PDF na base de conhecimento ([7f8e9d0](https://github.com/emn-f/vox-ai/commit/7f8e9d0...))

### Correções de Bug
- **ui**: Corrige crash da sidebar no mobile com texto longo ([2c3d4e5](https://github.com/emn-f/vox-ai/commit/2c3d4e5...))

### Documentação
- Atualiza README com instruções de setup local ([9a8b7c6](https://github.com/emn-f/vox-ai/commit/9a8b7c6...))
```

---

## Capítulo 4 — Cálculo Vetorial: Similaridade de Cosseno em Profundidade

### 4.1 Da Linguagem Natural ao Espaço Vetorial

O modelo `gemini-embedding-001` projeta qualquer texto em um espaço de **R^1536** (1536 dimensões reais). Cada dimensão captura um aspecto semântico latente aprendido durante o treinamento — conceitos, intenções, relações entre entidades.

Um fragmento de texto `T` é representado como:

```
v(T) = [d₁, d₂, d₃, ..., d₁₅₃₆]  onde dᵢ ∈ ℝ
```

A magnitude do vetor é irrelevante para o significado semântico. O que importa é a **direção** no espaço multidimensional. Dois textos sobre o mesmo assunto apontam na mesma direção, independentemente do tamanho ou vocabulário exato.

---

### 4.2 A Matemática da Similaridade de Cosseno

A **similaridade de cosseno** entre dois vetores `a` e `b` é definida como:

```
                 a · b
sim(a, b) = ─────────────
             ‖a‖ × ‖b‖
```

Onde:
- `a · b` é o **produto escalar** (dot product): `Σ(aᵢ × bᵢ)` para i de 1 a 1536
- `‖a‖` é a **norma euclidiana** de `a`: `√(Σaᵢ²)`
- O resultado está sempre no intervalo `[-1, 1]`

**Interpretação dos valores:**

| Valor de `sim(a,b)` | Interpretação Semântica |
|---|---|
| `1.0` | Vetores idênticos — textos semanticamente equivalentes |
| `0.8 – 0.99` | Alta similaridade — mesma intenção, diferentes palavras |
| `0.5 – 0.79` | Similaridade moderada — tópicos relacionados |
| `0.2 – 0.49` | Baixa similaridade — tópicos distintos com alguma relação |
| `≈ 0.0` | Sem relação semântica |
| `< 0` | Oposição semântica (raro em embeddings de linguagem) |

---

### 4.3 Distância vs. Similaridade: O Operador `<=>` do pgvector

O **pgvector** no PostgreSQL implementa a **distância de cosseno**, não a similaridade diretamente. A relação entre os dois é:

```
distância_cosseno(a, b) = 1 - sim(a, b)
```

Portanto:
- Distância `0.0` = vetores idênticos (similaridade 1.0)
- Distância `0.5` = similaridade moderada (similaridade 0.5)
- Distância `1.0` = sem relação (similaridade 0.0)

A função SQL `match_knowledge_base` no Supabase converte isso de volta para similaridade:

```sql
CREATE OR REPLACE FUNCTION match_knowledge_base(
    query_embedding vector(1536),
    match_threshold float,
    match_count int,
    filter_topic text DEFAULT NULL
)
RETURNS TABLE (
    id bigint,
    content text,
    topic text,
    similarity float
)
LANGUAGE sql STABLE
AS $$
    SELECT
        id,
        content,
        topic,
        -- Converte distância de cosseno em score de similaridade [0,1]
        1 - (knowledge_base.embedding <=> query_embedding) AS similarity
    FROM knowledge_base
    WHERE
        -- Aplica filtro de threshold ANTES de retornar (eficiente com índice HNSW)
        1 - (knowledge_base.embedding <=> query_embedding) > match_threshold
        -- Filtro opcional por tópico
        AND (filter_topic IS NULL OR topic = filter_topic)
    ORDER BY knowledge_base.embedding <=> query_embedding ASC
    LIMIT match_count;
$$;
```

---

### 4.4 O Threshold de 0.5 e sua Justificativa

O `SEMANTICA_THRESHOLD = 0.5` definido em `src/config.py` é um **corte de relevância mínima**. Qualquer chunk com similaridade inferior a 50% é descartado antes de ser enviado ao modelo Gemini como contexto.

**Por que 0.5?**

Em embeddings de linguagem de alta dimensão (1536d), scores abaixo de 0.5 geralmente indicam que o texto recuperado, embora possa compartilhar algumas palavras, não tem relação semântica substantiva com a pergunta. Incluir esses chunks como contexto pode:

1. **Confundir o modelo:** Contexto irrelevante aumenta o ruído, reduzindo a qualidade da resposta.
2. **Desperdiçar tokens:** Cada chunk ocupa espaço no contexto da API Gemini, aumentando custo e latência.
3. **Gerar alucinações:** O modelo pode tentar "conectar" um contexto irrelevante à pergunta, criando respostas falsas.

Um threshold de 0.5 foi escolhido empiricamente para o domínio LGBTQIA+, onde perguntas sobre um tema (ex: "PrEP") devem trazer apenas chunks sobre profilaxia, não chunks vagamente relacionados a "saúde" em geral.

---

### 4.5 Indexação HNSW — Por Que a Busca é Rápida

Sem um índice, a busca vetorial requer calcular a distância entre a query e **cada linha** da tabela `knowledge_base` — complexidade O(n). Com 10.000 chunks, isso significa 10.000 cálculos de dot product em vetores de 1536 dimensões.

O índice **HNSW (Hierarchical Navigable Small World)** resolve isso construindo um grafo de múltiplas camadas:

```
Camada 2 (esparsa):   •──────────────────•
                      │                  │
Camada 1 (média):     •──•──────•────────•──•
                      │  │     │         │  │
Camada 0 (densa):  •──•──•──•──•──•──•──•──•──•  (todos os vetores)
```

**Algoritmo de busca:**
1. Começa em um nó de entrada aleatório na camada superior (mais esparsa).
2. Em cada camada, avança em direção ao vizinho mais próximo da query.
3. Desce para a camada inferior quando não há mais progresso.
4. Na camada 0, executa busca local greedy para refinar os candidatos.

**Resultado:** Complexidade aproximada de **O(log n)** para encontrar os k vizinhos mais próximos, com uma pequena perda de precisão em troca de velocidade muito superior (busca aproximada, não exata).

**Criação do índice no Supabase:**

```sql
CREATE INDEX ON knowledge_base
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

- `m = 16`: número de conexões por nó no grafo (maior = mais preciso, mais memória)
- `ef_construction = 64`: tamanho do beam search durante a construção (maior = grafo melhor, construção mais lenta)

---

### 4.6 `task_type`: Por Que a Diferenciação é Crítica

O modelo `gemini-embedding-001` foi treinado com dois papéis distintos de vetorização:

| `task_type` | Usado em | Otimização |
|---|---|---|
| `RETRIEVAL_DOCUMENT` | Scripts de ETL (ingestão da KB) | Vetor otimizado para ser **encontrado** |
| `RETRIEVAL_QUERY` | `src/core/semantica.py` (chat) | Vetor otimizado para **encontrar** |

Usar `RETRIEVAL_QUERY` para documentos (ou vice-versa) produz um mismatch no espaço de embeddings: os vetores não são comparáveis de forma otimizada, reduzindo a precisão da busca.

**No código:**

```python
# Em semantica.py — para a pergunta do usuário
embedding_response = gemini_client.models.embed_content(
    model="gemini-embedding-001",
    contents=prompt,
    config=types.EmbedContentConfig(
        task_type="RETRIEVAL_QUERY",
        output_dimensionality=1536
    )
)

# Em scripts/etl_*.py — para os chunks da base de conhecimento
embedding_response = gemini_client.models.embed_content(
    model="gemini-embedding-001",
    contents=chunk_text,
    config=types.EmbedContentConfig(
        task_type="RETRIEVAL_DOCUMENT",
        output_dimensionality=1536
    )
)
```

---

### 4.7 Pipeline RAG Completo com Cálculo Vetorial

```
Pergunta do usuário: "Como funciona o processo de retificação de nome?"
            │
            ▼
  gemini-embedding-001 (RETRIEVAL_QUERY)
            │
            ▼
  v_query = [0.023, -0.147, 0.891, ..., 0.042]  # vetor 1536d
            │
            ▼
  Supabase RPC: match_knowledge_base(v_query, threshold=0.5, limit=10)
            │
            ▼
  PostgreSQL executa: 1 - (embedding <=> v_query) > 0.5
  (índice HNSW acelera a busca)
            │
            ▼
  Resultados ordenados por similaridade decrescente:
  ┌────────────────────────────────────────┬────────────┬───────────┐
  │ content                                │ topic      │ similarity│
  ├────────────────────────────────────────┼────────────┼───────────┤
  │ "Para retificar o nome, é necessário..."│ retificacao│  0.912    │
  │ "A Lei nº 14.931 estabelece..."        │ retificacao│  0.887    │
  │ "O cartório deve aceitar o pedido..."  │ retificacao│  0.843    │
  │ "Documentos necessários: RG, CPF..."   │ retificacao│  0.798    │
  │ "Processo transexualizador no SUS..."  │ saude      │  0.623    │
  └────────────────────────────────────────┴────────────┴───────────┘
            │
            ▼
  recuperar_contexto_inteligente():
  - Contagem por tópico: retificacao=4, saude=1
  - Tópico dominante (≥3 votos): "retificacao"
  - Estratégia: CONTEXTO EXPANDIDO
            │
            ▼
  buscar_chunks_por_topico("retificacao", limit=25)
  → Retorna TODOS os chunks sobre retificação
            │
            ▼
  Contexto enviado ao Gemini (como system context, não como mensagem do user)
            │
            ▼
  Resposta gerada com base na KB curada ✅
```

---

## Capítulo 5 — GitHub Actions: Especificação Técnica de Cada Workflow

### 5.1 Mapa Geral e Dependências

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

---

### 5.2 `production_pipeline.yml` — Pipeline Principal

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

#### Job 1: `test`

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

#### Job 2: `release_prod`

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

---

### 5.3 `security_review.yml` — Code Review com IA

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

---

### 5.4 `deploy_db.yml` — Migrations do Banco de Dados

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

---

### 5.5 `deploy_hugging_face.yml` — Espelhamento no HF Spaces

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

---

### 5.6 `deploy_pages.yml` — Dashboard de Transparência

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

---

### 5.7 `versioning_dev.yml` — Tagging na Branch `develop`

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

---

### 5.8 `manual_release.yml` — Release Manual

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

---

### 5.9 `push_homo.yml` — Atualização da Branch de Homologação

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

---

### 5.10 `changelog_dev.yml` — Sincronização do CHANGELOG

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

---

## Apêndice — Diagrama de Segredos e Variáveis de Ambiente

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

> Última atualização em 21/04/2026 — Vox AI v3.3.6  
> Documentação por Emanuel Ferreira | Licença GNU GPLv3
