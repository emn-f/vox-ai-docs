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