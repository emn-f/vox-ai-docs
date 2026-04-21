# 11. Convenções do Projeto

## 11.1 Conventional Commits

Formato: `tipo(escopo opcional): descrição em minúsculas`

| Tipo | Quando Usar | Aparece no CHANGELOG? | Exemplo |
|---|---|---|---|
| `feat` | Nova funcionalidade para o usuário | Sim (Funcionalidades) | `feat: adiciona botão de TTS na resposta` |
| `fix` | Correção de bug | Sim (Correções) | `fix(ui): corrige crash na sidebar mobile` |
| `docs` | Mudanças na documentação | Sim (Documentação) | `docs: atualiza README com setup local` |
| `perf` | Melhoria de performance | Sim (Performance) | `perf: otimiza carregamento do JSON` |
| `refactor` | Refatoração sem mudança de comportamento | Nao (pulado) | `refactor: simplifica lógica de busca` |
| `style` | Formatação, CSS (sem mudar lógica) | Nao (pulado) | `style: aplica PEP8 em utils.py` |
| `test` | Adição ou correção de testes | Nao (pulado) | `test: adiciona teste para salvar_erro` |
| `ci` | Mudanças em workflows do GitHub Actions | Nao (pulado) | `ci: corrige script de deploy HF` |
| `build` | Dependências, sistema de build | Nao (pulado) | `build: atualiza streamlit para 1.52` |
| `chore` | Tarefas internas diversas | Nao (pulado) | `chore: remove arquivos de cache` |

## 11.2 Conventional Migrations

Formato: `<timestamp>_<verbo>_<objeto>_<contexto>.sql`

| Verbo | Quando Usar | Exemplo |
|---|---|---|
| `create` | Criar tabela nova | `create_table_profiles` |
| `add` | Adicionar coluna/função em elemento existente | `add_email_to_users` |
| `update` | Alterar tipo de coluna, defaults ou lógica | `update_function_calculate_total` |
| `alter` | Mudanças estruturais (renomear, constraints) | `alter_users_set_email_unique` |
| `drop` | Remover tabela, coluna ou função | `drop_table_legacy_logs` |
| `fix` | Correções de lógica ou dados | `fix_rls_policy_on_profiles` |
| `seed` | Inserção de dados iniciais ou de teste | `seed_initial_categories` |
| `normalize` | Separar dados em novas tabelas | `normalize_report_categories` |

> **ATENÇÃO:** NUNCA use nomes genéricos como `update_db`, `migration_1` ou `changes`. A migration deve ser autoexplicativa sem precisar abrir o SQL.

## 11.3 Fluxo de Branches

| Branch | Propósito | Regras |
|---|---|---|
| `main` | Código em produção estavel | Protegida. Cada push gera uma release automática. |
| `develop` | Branch principal de desenvolvimento | PRs devem apontar para aqui. Todo push gera uma tag `dev-v*`. |
| `homo` | Ambiente de homologação | Atualizado manualmente via `push_homo.yml` a partir da develop. |
| `feat/*` | Feature branches | Criadas a partir de develop. Ex: `feat/minha-nova-feature` |

---
