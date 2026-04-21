# Projeto Vox AI 🏳️‍🌈

> Repositório principal: [github.com/emn-f/vox-ai](https://github.com/emn-f/vox-ai)

O **Vox AI** é um ecossistema digital seguro que oferece orientações confiáveis sobre direitos, saúde, cidadania e acolhimento emocional. O projeto utiliza uma arquitetura moderna de **RAG (Retrieval-Augmented Generation)** para garantir que as respostas da IA sejam baseadas em uma base de conhecimento curada, reduzindo alucinações.


## 🚀 Links Rápidos

* **App em Produção:** [vox-ai.streamlit.app](https://vox-ai.streamlit.app/)
* **Dashboard de Transparência:** [emn-f.github.io/vox-ai/](https://emn-f.github.io/vox-ai/)
* **Documentação Técnica:** [Docs Online](https://emn-f.github.io/vox-ai-docs/)

## 🛠️ Stack Tecnológica

O projeto é construído integralmente em **Python 3.13** e utiliza as seguintes tecnologias:

* **Interface:** [Streamlit](https://streamlit.io/)
* **Inteligência Artificial:** Google Gemini API (Modelos `flash-preview` e `embedding-001`)
* **Banco de Dados:** [Supabase](https://supabase.com/) (PostgreSQL 17 + pgvector)
* **Gerenciador de Pacotes:** `uv`
* **Voz (TTS):** gTTS para conversão de texto em áudio

## 🏗️ Arquitetura e RAG

O Vox AI utiliza duas estratégias inteligentes de recuperação para garantir precisão:

1.  **Contexto Expandido:** Se um tópico (ex: PrEP) domina os resultados da busca vetorial, o sistema recupera todos os fragmentos relacionados para dar uma resposta completa.
2.  **Tópicos Mistos:** Utilizada quando a pergunta é interdisciplinar, combinando os melhores fragmentos de diferentes temas.

Toda a infraestrutura é monitorada por um sistema de **Auditoria RAG**, que vincula cada resposta da IA aos fragmentos exatos de conhecimento utilizados no banco de dados.

## 💻 Quer contribuir?

Toda ajuda é bem vinda! Saiba mais no nosso [Guia de Contribuição](https://github.com/emn-f/vox-ai?tab=contributing-ov-file).


*🤖 Vox AI: conversas que importam 🏳️‍🌈*