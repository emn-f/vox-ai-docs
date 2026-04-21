# Sistema de Segurança e Code Review (Gatekeep)

## Visão Geral

O diretório `gatekeep/` contém um sistema de seguranca em duas camadas: validação local (Git Hooks) e validação remota (GitHub Actions).

## `install_hooks.py`

Instala 3 Git Hooks em `.git/hooks/`: `pre-commit`, `pre-push` e `commit-msg`. Gera os scripts shell dinamicamente, detectando o Python correto (venv local ou sistema). Execute uma vez após clonar:

```bash
python scripts/install_hooks.py
```

## `validate_commit_msg.py` — Conventional Commits

Hook `commit-msg` que valida a mensagem usando regex. Pula automaticamente commits de Merge e Revert automáticos.

```
# Regex usada:
^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?!?: .+$

# Exemplos VÁLIDOS:
feat: adiciona suporte a áudio
fix(ui): corrige quebra da sidebar no mobile

# Exemplos INVÁLIDOS (bloqueiam o commit):
atualiza código          # sem tipo
FEAT: nova feature       # tipo em maiúsculo
```

Saiba mais em 
## `security_check.py` — Revisão com IA

**1. Scan de Segredos (`sanitize_diff_for_ai`):** Analisa o diff com regex para detectar padrões de segredos comuns. Substitui valores suspeitos por `[REDACTED SECRET DETECTED]` antes de enviar para a IA.

**2. Revisão com IA (`run_ai_code_review`):**

| Resposta da IA | Resultado | Ação |
|---|---|---|
| `[PASS]` limpo | Aprovado | Permite silenciosamente. |
| `[PASS]` com sugestões | Aprovado com aviso | Permite, mas loga as sugestões. |
| `[BLOCK]` explícito | Bloqueado | Aborta com mensagem de erro. |
| `[PASS]` + keyword proibida | Bloqueado | Keywords como "password" ou "exposed" bloqueiam mesmo com PASS. |
| Exceção na API | Falha aberta | Permite continuar (para não bloquear por problemas de rede). |

---
