# Sistema de Segurança e Code Review (Gatekeep)
## Visão Geral

O diretório `gatekeep/` contém um sistema de seguranca em duas camadas: validação local (Git Hooks) e validação remota (GitHub Actions).

## Git Hooks: Arquitetura e Implementação

### O Sistema de Hooks Locais

O Vox AI utiliza **Git Hooks nativos** (não usa frameworks como `pre-commit` ou `husky`) para garantir qualidade antes que o código deixe a máquina do desenvolvedor. Os hooks são instalados pelo script `scripts/install_hooks.py`.

Três hooks são instalados no diretório `.git/hooks/`:

| Hook | Fase | Função |
|---|---|---|
| `commit-msg` | Ao criar mensagem de commit | Valida formato Conventional Commits |
| `pre-commit` | Antes de registrar o commit | Reservado para expansão futura |
| `pre-push` | Antes de enviar ao remoto | Executa `security_check.py` |


### `install_hooks.py` — Instalação Dinâmica

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

### `validate_commit_msg.py` — Hook `commit-msg`

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

### Hook `pre-push` — Gatilho do Security Check

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

### Hook `pre-commit` (Reservado)

O hook `pre-commit` é instalado mas implementado como um **no-op** (exit 0 imediato) na versão atual. Sua presença serve como ponto de extensão para futuras validações que devem ocorrer antes de finalizar o commit, como:

- Verificação de cobertura de testes mínima
- Linting com `ruff` ou `flake8`
- Verificação de arquivos grandes acidentalmente adicionados

## Fluxo Completo do `security_check.py`

### Visão Geral e Posicionamento no Sistema

O `gatekeep/security_check.py` é o núcleo do sistema de revisão de código com IA. Ele opera em **dois contextos distintos**:

| Contexto | Acionador | `--mode` | O que analisa |
|---|---|---|---|
| Git Hook local | `pre-push` no terminal do dev | `pre-push` | Diff entre `HEAD` e o branch remoto (`origin/<branch>`) |
| GitHub Actions | PR aberta para `develop` ou `main` | `pre-push` | Diff do PR completo via `GITHUB_BASE_REF` |

O script é projetado para **falhar aberto** em caso de erro de rede (`fail-open`): se a API do Gemini estiver indisponível, o push/merge é permitido para não bloquear o fluxo de desenvolvimento por razões de infraestrutura.

---

### Fase 1 — Coleta do Diff

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

### Fase 2 — Sanitização do Diff (`sanitize_diff_for_ai`)

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

### Fase 3 — Revisão com IA (`run_ai_code_review`)

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

### Fase 4 — Análise da Resposta e Decisão Final

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

### Diagrama de Fluxo Completo

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