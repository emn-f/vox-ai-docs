# Cálculo Vetorial e Busca Semântica

## O que são Embeddings?

No Vox AI, um "embedding" é uma representação numérica de um texto em um espaço multidimensional. Utilizamos o modelo `gemini-embedding-001`, que transforma cada fragmento de texto (chunk) em um vetor de **1536 dimensões**.

Nesse espaço, textos que possuem significados semânticos semelhantes são posicionados matematicamente próximos um do outro, independentemente de compartilharem as mesmas palavras-chave.

## A Métrica: Distância de Cosseno

Para determinar a relevância de um conteúdo para a pergunta do usuário, o sistema não procura por palavras exatas, mas sim pela menor distância entre o vetor da pergunta e os vetores armazenados na `knowledge_base`.

### O Operador `<=>`
O Vox AI utiliza a **Distância de Cosseno**, implementada no PostgreSQL através da extensão `pgvector` com o operador `<=>`.

* **Matemática do Score:** O cálculo mede o ângulo entre dois vetores.
* **Similaridade vs. Distância:** A função `match_knowledge_base` converte a distância em um score de similaridade usando a fórmula `1 - (vetor_a <=> vetor_b)`.
* **Resultado:** Um score de `1.0` indica vetores idênticos, enquanto scores próximos a `0` indicam textos sem relação semântica.

## Processamento e Threshold

O pipeline de cálculo segue estas etapas técnicas:

1.  **Vetorização da Query:** A pergunta do usuário é enviada para a API do Gemini com o `task_type='RETRIEVAL_QUERY'` para gerar seu vetor de 1536d.
2.  **Busca Vetorial (RPC):** O vetor é enviado ao Supabase, onde a função `match_knowledge_base` executa a comparação contra a coluna `embedding` da tabela `knowledge_base`.
3.  **Filtro de Relevância (Threshold):** Definimos um `SEMANTICA_THRESHOLD = 0.5` no arquivo `src/config.py`. Qualquer resultado com similaridade inferior a 50% é descartado para evitar que o modelo receba contextos irrelevantes.

## Indexação HNSW (Hierarchical Navigable Small World)

Para garantir que a busca seja instantânea mesmo com milhares de registros, utilizamos um índice **HNSW** na coluna de embeddings.

* **Como funciona:** O HNSW constrói um grafo de camadas onde os vetores são conectados aos seus vizinhos mais próximos.
* **Vantagem:** Em vez de comparar a pergunta com cada linha do banco (busca linear), o algoritmo "salta" pelo grafo para encontrar os resultados mais próximos em tempo sub-linear.

## Tipos de Tarefa (Task Types)

É crucial notar que o Vox AI diferencia o cálculo conforme o papel do texto:

* **`RETRIEVAL_DOCUMENT`:** Usado ao gerar embeddings para a base de conhecimento (scripts de ETL). O modelo otimiza o vetor para ser um "alvo" de busca.
* **`RETRIEVAL_QUERY`:** Usado para a pergunta do usuário no chat. O modelo otimiza o vetor para agir como uma "busca".

---