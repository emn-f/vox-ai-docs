# 4. Arquitetura do Sistema

## 4.1 Visão Geral

O Vox AI e uma aplicação monolítica em Python construída sobre Streamlit. Não há backend separado: o Streamlit gerencia o estado da sessão, a UI e orquestra as chamadas aos serviços externos (Gemini API e Supabase). Toda a lógica de negócio fica nos modulos da pasta `src/`.

## 4.2 Fluxo de Execução Principal

1. O usuário digita uma pergunta (texto) ou grava um áudio no chat Streamlit (`vox_ai.py`).
2. Se for áudio, o arquivo é enviado para `transcrever_audio()` em `genai.py`, que usa o Gemini para transcrição para pt-BR.
3. O texto (original ou transcrito) é passado para a função `semantica()` em `semantica.py`.
4. Dentro de `semantica()`, o texto e convertido em um vetor de 1536 dimensões pelo modelo `gemini-embedding-001` com `task_type='RETRIEVAL_QUERY'`.
5. O vetor é passado para `recuperar_contexto_inteligente()` em `database.py`, que executa a busca vetorial no Supabase via RPC (função `match_knowledge_base`).
6. A função RAG analisa os resultados e decide a estratégia: **Contexto Expandido** (tópico dominante com 3+ votos) ou **Tópicos Mistos** (top 5 de fragmentos).
7. O contexto recuperado e combinado com o prompt original e enviado para `gerar_resposta()` em `genai.py`.
8. O Gemini gera a resposta em streaming, exibida letra por letra na interface.
9. O log (prompt, resposta, IDs dos chunks usados, versão git) é salvo assincronamente no Supabase.

## 4.3 Estratégia RAG (Retrieval-Augmented Generation)

O coração inteligente do Vox AI é sua estratégia de recuperacao de conhecimento. O sistema não usa um RAG ingênuo — ele analisa os resultados para decidir como deve proceder.

### Estratégia 1: Contexto Expandido

Ativada quando um mesmo tópico aparece **3 ou mais vezes** nos 10 primeiros resultados da busca vetorial. O sistema então busca TODOS os chunks daquele tópico na base (até `MAX_CHUNCK = 25`), fornecendo contexto completo ao LLM.

> **Exemplo:** Usuário pergunta sobre PrEP. Se 5 dos 10 resultados são do tópico "PrEP", o sistema busca todos os 25 fragmentos disponíveis sobre esse tema na base de conhecimento (`knowledge_base`).

### Estratégia 2: Tópicos Mistos

Ativada quando os resultados são dispersos (nenhum tópico domina com 3+ votos). O sistema usa apenas os 5 fragmentos mais similares, que podem ser de temas diferentes. Útil para perguntas gerais ou interdisciplinares.

## 4.4 Sistema de `Sessions`

Cada usuário que acessa o Vox AI recebe um UUID único gerado pelo `uuid.uuid4()`. Este identificador anônimo é salvo na tabela `sessions` do Supabase e usado para correlacionar logs de chat, erros e reports. O ID vive apenas na sessão do Streamlit (`st.session_state`) e nunca é exposto ao usuário.

## 4.5 Configuração e Segredos

A função `get_secret()` em `src/config.py` implementa sistema de fallback: primeiro tenta `st.secrets` (Streamlit Cloud / `.streamlit/secrets.toml`), depois as variáveis de ambiente do sistema.

```toml
# .streamlit/secrets.toml (NUNCA commitar este arquivo!)

GEMINI_API_KEY = "AIza..."

[supabase]
url = "https://xxxxx.supabase.co"
key = "ey..."
```

---