# RAG classroom stack — n8n + Qdrant + Ollama

## Start

```bash
mkdir -p files
docker compose up -d
docker exec -it ollama ollama pull nomic-embed-text   # ~274 MB, once per laptop
```

| Service | URL | Notes |
|---|---|---|
| n8n | http://localhost:5678 | create a local owner account on first visit |
| Qdrant dashboard | http://localhost:6333/dashboard | inspect collections and points |
| Ollama | http://localhost:11434 | embeddings only |

## Internal hostnames (use these inside n8n, NOT localhost)

- Qdrant: `http://qdrant:6333`
- Ollama: `http://ollama:11434`

`localhost` inside the n8n container points at n8n itself — this is the #1 thing students get wrong.

## n8n credentials to create

1. **Qdrant** → URL `http://qdrant:6333`, API key: leave empty.
2. **Ollama** → Base URL `http://ollama:11434`.
3. **Anthropic** → students' own API key.

## Ingest workflow (write side)

`Outline (HTTP Request → documents.list / documents.info)`
→ `Default Data Loader` + `Recursive Character Text Splitter` (chunk 1000, overlap 200)
→ `Embeddings Ollama` (model `nomic-embed-text`)
→ `Qdrant Vector Store` in **Insert** mode, collection e.g. `wiki`

Outline API: `POST https://app.getoutline.com/api/documents.list`
with header `Authorization: Bearer <token>`, JSON body `{"collectionId": "<their collection>"}`.
Document text is in `data[].text` (markdown).

## Query workflow (read side)

`Chat Trigger` → `AI Agent`
- Chat Model: **Anthropic Chat Model** (Claude)
- Tool: **Vector Store Q&A** wrapping the `Qdrant Vector Store` node in **Retrieve** mode,
  with the same `Embeddings Ollama` node attached.

Same embedding model must be used on both sides — mismatched dimensions is the second
most common student error.

## Reset

```bash
docker compose down -v    # wipes n8n workflows AND the vector store
```
