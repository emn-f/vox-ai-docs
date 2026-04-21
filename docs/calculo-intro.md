# Cálculo Vetorial: Similaridade de Cosseno em Profundidade

## Da Linguagem Natural ao Espaço Vetorial

O modelo `gemini-embedding-001` projeta qualquer texto em um espaço de **R^1536** (1536 dimensões reais). Cada dimensão captura um aspecto semântico latente aprendido durante o treinamento — conceitos, intenções, relações entre entidades.

Um fragmento de texto `T` é representado como:

```
v(T) = [d₁, d₂, d₃, ..., d₁₅₃₆]  onde dᵢ ∈ ℝ
```

A magnitude do vetor é irrelevante para o significado semântico. O que importa é a **direção** no espaço multidimensional. Dois textos sobre o mesmo assunto apontam na mesma direção, independentemente do tamanho ou vocabulário exato.

## A Matemática da Similaridade de Cosseno

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

## Distância vs. Similaridade: O Operador `<=>` do pgvector

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


## O Threshold de 0.5 e sua Justificativa

O `SEMANTICA_THRESHOLD = 0.5` definido em `src/config.py` é um **corte de relevância mínima**. Qualquer chunk com similaridade inferior a 50% é descartado antes de ser enviado ao modelo Gemini como contexto.

**Por que 0.5?**

Em embeddings de linguagem de alta dimensão (1536d), scores abaixo de 0.5 geralmente indicam que o texto recuperado, embora possa compartilhar algumas palavras, não tem relação semântica substantiva com a pergunta. Incluir esses chunks como contexto pode:

1. **Confundir o modelo:** Contexto irrelevante aumenta o ruído, reduzindo a qualidade da resposta.
2. **Desperdiçar tokens:** Cada chunk ocupa espaço no contexto da API Gemini, aumentando custo e latência.
3. **Gerar alucinações:** O modelo pode tentar "conectar" um contexto irrelevante à pergunta, criando respostas falsas.

Um threshold de 0.5 foi escolhido empiricamente para o domínio LGBTQIA+, onde perguntas sobre um tema (ex: "PrEP") devem trazer apenas chunks sobre profilaxia, não chunks vagamente relacionados a "saúde" em geral.

## Indexação HNSW — Por Que a Busca é Rápida

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


## `task_type`: Por Que a Diferenciação é Crítica

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

## Pipeline RAG Completo com Cálculo Vetorial

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