# Scripts Utilitários

> Última atualização em 07/06/2026

## `scripts/gerar_embedding.py` — Reindexação da KB

Deve ser executado quando ha registros com `embedding` nulo ou quando o modelo de embeddings e trocado. A função `reindexar()` busca todos os registros onde `embedding IS NULL`, itera gerando vetores com `task_type='RETRIEVAL_DOCUMENT'`, e salva apenas se a coluna ainda estiver nula (previne sobrescrita acidental). Tem delay de 500ms entre requisições para respeitar os rate limits da API.

## `pages/dashboard.js` — Dashboard de Transparencia

JavaScript do Dashboard público no GitHub Pages. Ao carregar, executa três operações no Supabase (ambiente de dev):

1. Conta os registros de `knowledge_base` usando o header `Content-Range` do Supabase (subtrai 1 para descontar o placeholder `N/A`).
2. Busca a data de `modificado_em` mais recente e formata como versão `v{ano}.{mes}.{dia}`.
3. Busca e renderiza os últimos 5 blocos do `CHANGELOG.md` usando a biblioteca `marked.js`.

As credenciais (`__SUPABASE_URL_DEV__` e `__SUPABASE_ANON_KEY_DEV__`) são placeholders injetados pelo workflow `deploy_pages.yml` via `sed` antes do deploy.

---
