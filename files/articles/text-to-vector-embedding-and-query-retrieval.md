# How Text Becomes a Numerical Vector and Is Retrieved at Query Time

Modern search and retrieval-augmented generation (RAG) systems can find relevant text even when a query does not contain the same words as the source document. They do this by converting text into numerical vectors called **embeddings**, storing those vectors beside the original text, and searching for nearby vectors when a query arrives.

This guide follows that complete journey: raw text, tokenization, token IDs, contextual representations, one dense vector, vector-database indexing, query embedding, similarity search, reranking, and retrieval of the original text.

## The Most Important Idea

An embedding model does not normally turn a vector back into its original sentence.

At ingestion time, the system stores both:

- the numerical vector used for search;
- the original text and its metadata used for display or RAG context.

At query time, the database compares the query vector with stored vectors. It returns the identifiers and payloads belonging to the closest matches. The application then reads the saved text for those matches.

The process is therefore:

```text
document text -> embedding vector -> vector index
                      |                  |
                      +---- saved with original text and metadata

user query -> query vector -> nearest-neighbor search -> matching IDs -> original text
```

Retrieval is a lookup based on vector similarity, not a vector-to-text decoding operation.

## 1. Why Text Must Become Numbers

Machine-learning models operate on numbers. A sentence such as:

```text
How can I reset my account password?
```

must therefore be represented numerically before a model can process it. A basic representation might count words, but counts cannot capture context well. For example, these sentences express similar intent despite using different vocabulary:

```text
How can I reset my account password?
I forgot my login credentials and need to create a new secret.
```

A trained embedding model places semantically related texts near one another in a high-dimensional space. Unrelated texts should be farther apart.

The final output may contain 384, 768, 1,024, 1,536, 3,072, or another fixed number of floating-point values, depending on the model:

```text
[0.018, -0.442, 0.107, 0.006, ..., -0.231]
```

Individual dimensions usually do not have simple labels such as `password` or `sports`. Meaning is distributed across many dimensions and learned from patterns in training data.

## 2. Tokenization in Detail

Tokenization is the process of dividing text into units a model can map to integers. Those units are called **tokens**.

Tokens are not always words. A token can be:

- a complete common word;
- part of a longer or less common word;
- punctuation;
- whitespace or a whitespace-prefixed word;
- a number or part of a number;
- a special control symbol.

For example, a tokenizer might represent this text:

```text
Tokenization helps retrieval.
```

as conceptual pieces like:

```text
["Token", "ization", " helps", " retrieval", "."]
```

The exact split depends on the tokenizer. Never assume that one visible word equals one token.

### 2.1 Why Subword Tokenization Is Common

A vocabulary containing every possible word would be enormous and would still fail on new names, spelling variations, code identifiers, and technical terms. Character-level tokenization avoids unknown words, but produces very long sequences.

Subword tokenization provides a practical middle ground. Frequent words or fragments can have their own tokens, while uncommon words are assembled from smaller pieces. Common approaches include:

- **Byte Pair Encoding (BPE):** repeatedly learns frequent symbol combinations;
- **WordPiece:** selects subword units that improve a language-model objective;
- **Unigram tokenization:** starts with many candidate pieces and removes less useful ones;
- **SentencePiece:** trains directly from raw text and can implement BPE or unigram models;
- **byte-level tokenization:** operates from bytes, allowing almost any input to be represented without an unknown token.

The tokenizer and embedding model are a matched pair. Token IDs from one tokenizer should not be sent to a model trained with another vocabulary.

### 2.2 Vocabulary and Token IDs

The tokenizer has a fixed vocabulary that maps each token to an integer:

```text
"reset"    -> 8124
" password" -> 6219
"?"        -> 30
```

Suppose the input is:

```text
reset password?
```

A simplified tokenizer result could be:

```text
tokens:    ["reset", " password", "?"]
token_ids: [8124, 6219, 30]
```

The IDs are vocabulary addresses. Their numerical order does not express meaning: token ID `8124` is not more meaningful than token ID `30`.

### 2.3 Special Tokens

Models may add special tokens to communicate structure. Depending on the model, these can represent:

- sequence start;
- sequence end;
- padding;
- unknown content;
- separation between text segments;
- classification or pooling positions;
- masks used during training.

A conceptual model input might look like:

```text
[START] reset password ? [END] [PAD] [PAD]
```

Special tokens are part of the model contract. Their names and behavior differ across model families.

### 2.4 Attention Masks

Batches often contain sequences of different lengths. Shorter sequences are padded to a common length. An **attention mask** tells the transformer which positions contain real tokens and which are padding:

```text
token_ids:      [101, 8124, 6219, 30, 102,   0,   0]
attention_mask: [  1,    1,    1,  1,   1,   0,   0]
```

Without the correct mask, padding can incorrectly influence the final embedding.

### 2.5 Token Limits, Truncation, and Context Windows

Embedding models accept a maximum number of tokens. If a chunk exceeds that limit, the application or tokenizer may truncate it. Truncation can silently remove the part containing the answer.

Token counts matter when choosing:

- document chunk size;
- overlap between chunks;
- batch size and memory usage;
- model input limits;
- context size sent to the final language model;
- API cost and latency.

Count tokens with the tokenizer used by the actual model. Character counts and word counts are only rough estimates.

### 2.6 Tokenization Across Languages, Code, and Numbers

Token efficiency varies by language and domain. A tokenizer trained mostly on English may require more tokens for another language. Long identifiers, source code, URLs, product codes, and decimal numbers may split into many pieces.

This affects retrieval because excessive fragmentation can:

- reduce the amount of information that fits in one chunk;
- weaken representations of domain-specific terms;
- increase latency and cost;
- make exact identifiers harder to retrieve semantically.

For identifiers, names, dates, and codes, hybrid retrieval that combines semantic vectors with keyword or lexical search is often stronger than embeddings alone.

## 3. From Token IDs to Contextual Token Representations

After tokenization, the embedding model processes the integer sequence.

### 3.1 Token Embedding Lookup

The model contains a learned embedding table. Each token ID selects one row from that table. If the internal hidden size is 768, each token begins as a vector with 768 learned values.

```text
token ID 8124 -> [0.12, -0.08, 0.31, ..., 0.04]
token ID 6219 -> [0.42,  0.11, 0.03, ..., -0.16]
```

This initial lookup vector is only the starting representation. It does not yet fully reflect the token's meaning in the current sentence.

### 3.2 Position Information

Transformers need information about token order. The model adds or applies positional information so that:

```text
dog bites person
```

is distinguishable from:

```text
person bites dog
```

Models may use learned positional embeddings, sinusoidal encodings, rotary position embeddings, or relative-position mechanisms.

### 3.3 Transformer Layers and Self-Attention

The sequence passes through multiple transformer layers. Self-attention allows each token to gather information from other relevant tokens.

The representation of `bank` should differ in these sentences:

```text
The bank approved the business loan.
We sat on the river bank.
```

After attention and feed-forward transformations, every token has a **contextual token representation**. The vector now reflects both the token and its surrounding text.

Conceptually:

```text
token IDs
  -> token lookup vectors
  -> position information
  -> transformer layers
  -> contextual vector for every token
```

## 4. From Many Token Vectors to One Text Vector

A sentence or document chunk contains many tokens, but a vector database usually stores one fixed-size vector per chunk. The model therefore needs a pooling strategy.

Common strategies include:

- **mean pooling:** average the contextual vectors of all non-padding tokens;
- **CLS pooling:** use the output at a designated classification token;
- **last-token pooling:** use the final valid token representation;
- **weighted pooling:** combine tokens with learned or position-based weights;
- **model-specific pooling:** use the method expected during that embedding model's training.

For mean pooling with token vectors `h1` through `hn`:

```text
chunk_vector = (h1 + h2 + ... + hn) / n
```

Padding positions must be excluded. The pooling method should match the model's documented usage; changing it can significantly reduce retrieval quality.

### 4.1 Projection and Output Dimensions

Some models apply a learned projection after pooling to produce the required output dimension. A model with hidden size 768 might return a 384-dimensional vector, for example.

The dimensionality is fixed for that model. Every stored document vector and every query vector searched in the same index must have compatible dimensions.

### 4.2 Vector Normalization

Many systems L2-normalize embeddings so each vector has length 1:

```text
normalized_vector = vector / sqrt(sum(vector_i * vector_i))
```

For normalized vectors, cosine similarity and dot-product ranking are closely related. Normalization must be consistent between document and query embeddings.

## 5. What the Numerical Vector Represents

An embedding is a compressed, learned representation useful for measuring semantic relatedness. Nearby points often share meaning, intent, topic, or usage.

Consider three chunks:

```text
A: Steps for changing a forgotten account password.
B: Reset your login credentials from the security page.
C: The cricket match starts at 7:30 PM.
```

The vectors for A and B should usually be closer to each other than either is to C. The values themselves are difficult to interpret one dimension at a time. Their geometry is what matters.

An embedding is also lossy. It is not an encrypted copy of the sentence and should not be treated as a reversible representation. Sensitive data still requires proper access control because embeddings and stored payloads can leak information through search behavior or attacks.

## 6. Document Ingestion: Creating the Searchable Knowledge Base

Before users can search, documents pass through an ingestion pipeline.

### Step 1: Load and Parse Documents

The system extracts usable content from sources such as HTML, PDF, Markdown, databases, support tickets, or application records. Navigation text, repeated headers, broken OCR, and irrelevant boilerplate should be cleaned when possible.

### Step 2: Split Documents into Chunks

Embedding an entire long document into one vector can hide important local details. Documents are normally split into smaller chunks based on headings, paragraphs, sentences, token limits, or semantic boundaries.

Each chunk should be understandable enough to answer a focused question. Useful metadata should preserve its relationship to the source.

### Step 3: Tokenize Each Chunk

The matched tokenizer converts every chunk into token IDs and attention masks. Inputs are truncated or padded according to the model contract.

### Step 4: Run the Embedding Model

The model creates contextual token representations, pools them, and returns one vector per chunk. Production systems usually embed chunks in batches for efficiency.

### Step 5: Store Vector, Text, and Metadata

A vector record might conceptually contain:

```json
{
  "id": "handbook-42-section-3",
  "vector": [0.018, -0.442, 0.107],
  "text": "To reset a forgotten password, open Account Security...",
  "metadata": {
    "document_id": "handbook-42",
    "title": "Account Security Handbook",
    "section": "Password recovery",
    "source_url": "/handbook/security#password-recovery",
    "tenant_id": "company-a",
    "access_level": "employee",
    "updated_at": "2026-07-13"
  }
}
```

Real vectors have many more dimensions than this shortened example.

### Step 6: Build or Update the Vector Index

The database organizes vectors into an index that can find close neighbors efficiently. The original chunk may be stored directly as payload or in a separate document store referenced by the vector record ID.

## 7. Query-Time Retrieval, Step by Step

Suppose a user asks:

```text
I cannot sign in. How do I create a new password?
```

The query-time pipeline works as follows.

### Step 1: Validate and Normalize the Query

The application validates input, checks identity and permissions, and may normalize harmless formatting. Avoid aggressive rewriting that removes names, numbers, or technical terms.

### Step 2: Tokenize the Query

The same tokenizer family used for document embedding converts the query to token IDs and an attention mask.

### Step 3: Create the Query Embedding

The same embedding model, version, pooling method, and normalization policy produce a query vector:

```text
query_vector = [0.021, -0.401, 0.094, ..., -0.205]
```

Using incompatible models for documents and queries usually makes the vector coordinates incomparable. Some retrieval models are specifically trained with separate query and passage instructions, but both sides still belong to the same trained embedding space.

### Step 4: Apply Security and Metadata Filters

The search should be limited to records the user is allowed to access. Filters may include:

- tenant or organization ID;
- user or group permissions;
- language;
- content type;
- product or region;
- publication status;
- date range.

Authorization must be enforced by the retrieval service, not left to the language model.

### Step 5: Search for Nearest Vectors

The vector database compares the query vector with indexed document vectors and finds the closest candidates. It returns record IDs, similarity scores, payloads, and metadata.

A simplified result might be:

```text
1. handbook-42-section-3   score: 0.91
2. faq-18-password         score: 0.87
3. support-7-login         score: 0.74
```

### Step 6: Retrieve the Original Text

For each selected ID, the system gets the original stored chunk:

```text
To reset a forgotten password, open Account Security and select Reset Password...
```

This is how the text comes back. The vector points the search toward a stored record; the database or document store supplies the original text.

### Step 7: Rerank the Candidates

Fast vector search may retrieve 20 to 100 candidates. A reranker can score each query-chunk pair more precisely and keep the best few.

Common reranking approaches include:

- a cross-encoder model that reads query and chunk together;
- a late-interaction retrieval model;
- lexical and vector score fusion;
- a carefully constrained language-model judge;
- business rules for authority, freshness, or source quality.

Reranking is slower than initial retrieval, so it is normally applied only to a small candidate set.

### Step 8: Assemble RAG Context

The final selected chunks are deduplicated, ordered, and fitted into a token budget. Their source identifiers are preserved for citations.

The language model receives:

- the user's question;
- trusted system instructions;
- the retrieved text chunks;
- source metadata or citation labels;
- rules for handling missing evidence.

### Step 9: Generate and Validate the Answer

The language model generates an answer grounded in the retrieved evidence. A production system may then validate citations, detect unsupported claims, enforce output schemas, or abstain when retrieval quality is too low.

## 8. Similarity and Distance Measures

The vector database needs a numerical rule for closeness.

### 8.1 Cosine Similarity

Cosine similarity measures the angle between two vectors:

```text
cosine(a, b) = dot(a, b) / (length(a) * length(b))
```

Higher values generally mean greater similarity. For unit-normalized vectors, it becomes the dot product.

### 8.2 Dot Product

The dot product multiplies corresponding dimensions and sums them:

```text
dot(a, b) = sum(a_i * b_i)
```

It is fast and is commonly used when the model was trained for that metric.

### 8.3 Euclidean Distance

Euclidean distance measures straight-line distance:

```text
distance(a, b) = sqrt(sum((a_i - b_i) * (a_i - b_i)))
```

Lower values indicate closer vectors.

Choose the metric recommended for the embedding model. A mismatch between training assumptions and search configuration can hurt ranking.

## 9. Exact Search and Approximate Nearest Neighbors

An exact search compares the query with every stored vector. It gives exact nearest neighbors but becomes expensive for millions or billions of records.

Approximate nearest-neighbor (ANN) indexes search a carefully chosen subset of the space. They trade a small amount of recall for much lower latency.

Common index families include:

- **HNSW:** builds a multi-layer graph and navigates from broad connections to nearby vectors;
- **IVF:** groups vectors into coarse clusters and searches the most promising clusters;
- **product quantization (PQ):** compresses vectors to reduce memory and speed comparisons;
- **IVF-PQ:** combines clustered search with compressed vectors;
- **tree or partition-based methods:** divide the vector space into searchable regions.

Important tuning parameters control:

- build time;
- memory consumption;
- query latency;
- candidate count;
- recall of true nearest neighbors.

Evaluate the index with representative queries. Fast search is not useful if it consistently misses the chunks containing the answer.

## 10. A Small End-to-End Example

Assume the knowledge base contains three chunks:

```text
Chunk A: Employees can reset passwords from Account Security.
Chunk B: Expense reports must be submitted before Friday.
Chunk C: Login credentials can be recovered by verifying an email address.
```

During ingestion:

```text
Chunk A -> tokenize -> embed -> vector_A -> store with Chunk A
Chunk B -> tokenize -> embed -> vector_B -> store with Chunk B
Chunk C -> tokenize -> embed -> vector_C -> store with Chunk C
```

The user asks:

```text
What should I do if I forgot my sign-in secret?
```

During retrieval:

```text
query -> tokenize -> embed -> query_vector
query_vector -> ANN search -> [vector_C, vector_A, vector_B]
record IDs -> load [Chunk C, Chunk A, Chunk B]
reranker -> [Chunk A, Chunk C]
context builder -> prompt with source labels
language model -> grounded answer with citations
```

The query shares few exact words with Chunk A, but a good embedding model recognizes the semantic relationship among `forgot`, `sign-in secret`, `password`, and `reset`.

## 11. How Tokenization Affects Retrieval Quality

Tokenization is not just preprocessing. It influences what the embedding model can see and how document chunks should be designed.

### Chunk Size

Very small chunks can lose context. Very large chunks can mix unrelated topics, making a single vector less specific. Start with a token-based size appropriate for the documents and model, then tune using evaluation data.

### Chunk Overlap

Overlap can keep an answer from being split at a boundary, but excessive overlap creates duplicate results, increases storage, and wastes context tokens.

### Headings and Structure

Including a concise heading or document title can give a chunk essential context. Repeating large navigation blocks or entire document titles in every chunk can dominate the representation.

### Domain Terms

Medical codes, legal references, model numbers, and internal acronyms may tokenize poorly. Domain-trained embedding models, keyword search, metadata filters, and query expansion can compensate.

### Query and Passage Instructions

Some models expect prefixes or task instructions such as conceptual `query:` and `passage:` markers. Follow the model's required format at both ingestion and query time.

### Model Upgrades

Changing the embedding model, tokenizer, dimensions, pooling, or normalization usually requires re-embedding the document collection. Store the model name and version with each index to prevent incompatible vectors from being mixed.

## 12. Hybrid Retrieval

Vector retrieval is strong at semantic matching, but it can be weak for exact strings. Keyword retrieval such as BM25 is often better for:

- names and IDs;
- exact error messages;
- quoted phrases;
- rare acronyms;
- dates, versions, and part numbers.

A hybrid system runs lexical and vector searches, then combines their rankings. Reciprocal Rank Fusion is one common approach:

```text
combined_score(document) = sum(1 / (constant + rank_in_each_result_list))
```

Hybrid retrieval often gives a better production baseline than relying on one method for every query.

## 13. Common Failure Modes

### Incompatible Embedding Spaces

Documents were embedded with one model and queries with another. Re-embed documents or route queries to the matching index.

### Incorrect Normalization or Metric

Document and query vectors use different normalization, or the index metric does not match the model. Apply one consistent contract.

### Important Text Was Truncated

The tokenizer cut off the answer because the chunk exceeded the model limit. Inspect token counts and truncation logs.

### Weak Chunking

Chunks are too broad, too small, or detached from their headings. Test structure-aware and semantic chunking strategies.

### Missing Access Filters

The retriever searches data outside the user's permissions. Enforce authorization filters before results are returned.

### Duplicate or Stale Content

Repeated chunks crowd out diverse evidence, while outdated records outrank current policy. Deduplicate at ingestion and apply freshness rules.

### Similar Topic, Wrong Answer

The vector search finds topically related text that does not answer the specific question. Retrieve more candidates and use a strong reranker.

### Exact Terms Are Missed

Semantic search overlooks an error code or product ID. Add lexical search and structured metadata filters.

### No Confidence Policy

The system always sends weak results to the language model. Define thresholds, evidence checks, and a useful abstention response.

## 14. Production Design Checklist

- Use the tokenizer and input format required by the embedding model.
- Measure chunk sizes in tokens, not only characters or words.
- Store original text, stable IDs, source details, and authorization metadata.
- Version the embedding model, tokenizer, pooling, normalization, and index.
- Use the same compatible embedding space for documents and queries.
- Select the similarity metric recommended for the model.
- Apply tenant and permission filters inside the retrieval layer.
- Combine vector and lexical retrieval when exact terms matter.
- Rerank a broader candidate set before constructing the final context.
- Preserve source metadata so generated answers can cite evidence.
- Evaluate retrieval separately from generation.
- Track recall, ranking quality, latency, empty results, and stale content.
- Rebuild embeddings when the representation contract changes.
- Protect both vectors and original payloads as potentially sensitive data.
- Let the system abstain when it cannot retrieve sufficient evidence.

## 15. How to Evaluate the Pipeline

Create a test dataset containing real questions, expected source chunks, and relevance labels. Evaluate each stage independently.

For retrieval, measure:

- **Recall@k:** whether a relevant chunk appears in the top `k` results;
- **Precision@k:** how many of the top `k` results are relevant;
- **Mean Reciprocal Rank:** how early the first relevant result appears;
- **nDCG:** whether highly relevant chunks rank above partly relevant chunks;
- filter correctness and permission safety;
- latency at realistic index sizes.

For the full RAG system, also measure answer correctness, faithfulness to context, citation accuracy, completeness, and abstention quality.

When an answer is wrong, identify the first failing stage:

```text
parsing -> chunking -> tokenization -> embedding -> indexing
        -> candidate retrieval -> reranking -> context building -> generation
```

This is more actionable than changing prompts whenever the final answer looks poor.

## Final Summary

Text becomes searchable through a sequence of learned numerical transformations:

1. A tokenizer splits text into subword or byte-level tokens.
2. Vocabulary entries convert those tokens into integer IDs.
3. A transformer converts IDs into contextual token representations.
4. Pooling and optional projection produce one fixed-size vector.
5. The vector is stored with the original chunk, identifiers, and metadata.
6. The same compatible process converts a user query into a query vector.
7. A vector index finds nearby document vectors using a similarity metric.
8. The system loads the original text associated with those vector records.
9. Reranking selects the strongest evidence.
10. A RAG application sends that evidence to a language model to generate a grounded answer.

The central principle is simple: **the vector enables matching; the stored record returns the text**. Tokenization, chunking, model compatibility, indexing, filters, and reranking determine how reliably that match works in production.
