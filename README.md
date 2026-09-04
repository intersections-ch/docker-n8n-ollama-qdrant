# RAG classroom stack — n8n + Qdrant + Ollama

## Start

The stack is split into three compose files, one per service. They talk to each other
over a shared Docker network called `ragnet`, which the first file you start creates
automatically — there is nothing to set up by hand. You can also install all three 
containers in one go by using the main docker-compose.yml file. 
Note: ollama is a large image (more than 3GB) and will take a while to download.
For qdrant to work, ollama will need an embedding model like `nomic-embed-text`.
Install everything with the commands below. 


```bash
mkdir -p files

docker compose -f docker-compose.qdrant.yml up -d
docker compose -f docker-compose.ollama.yml up -d
docker compose -f docker-compose.n8n.yml up -d

docker exec -it ollama ollama pull nomic-embed-text   # ~274 MB, once per laptop
```

Start Qdrant and Ollama before n8n. There is no `depends_on` across compose files, but
n8n only needs them when a workflow actually runs, so the order is a convenience, not a
hard requirement.

`docker-compose.yml` still contains the original all-in-one stack. Use either that
(`docker compose up -d`) or the three split files — not both at once, they bind the
same ports.

### Picking a file

The flag is `-f`, and it goes **before** the subcommand:

```bash
docker compose -f docker-compose.qdrant.yml up -d     # start one service
docker compose -f docker-compose.qdrant.yml logs -f   # follow its logs
docker compose -f docker-compose.qdrant.yml down      # stop it
```

`docker compose up docker-compose.qdrant.yml` does **not** work — Compose reads that as
a service name.

### If a container name is already taken

Compose files can't branch, so the names are hardcoded. If a machine already has an
unrelated `n8n` container, `up` fails with *"container name /n8n is already in use"*.
Edit the file by hand — three lines in `docker-compose.n8n.yml`:

```yaml
name: n8n-2                 # project name: keeps volumes separate from the other stack
services:
  n8n:
    container_name: n8n-2   # the name that collided
    ports:
      - "5778:5678"         # left side only; 5678 is the port inside the container
```

Same shape for `docker-compose.qdrant.yml` (`6333`, `6334`) and
`docker-compose.ollama.yml` (`11434`). Change only the **left** side of each port
mapping — the right side is inside the container and must stay as-is.

If you rename Qdrant or Ollama, update the URLs in the n8n credentials to match
(`http://qdrant-2:6333`), and note that a renamed Ollama changes the pull command to
`docker exec -it ollama-2 ollama pull nomic-embed-text`.

**If you end up running two full stacks at once:** each has its own volumes (they are
namespaced by the `name:` you set), but both attach to `ragnet`, where the alias `qdrant`
then resolves to *both* Qdrant containers round-robin. Use the exact container name in
n8n's credentials rather than the bare alias whenever more than one stack is up.

Each file declares its own `name:` (`qdrant`, `ollama`, `n8n`), so the three are separate
Compose projects and stopping one never touches the others. Without those names Compose
would derive one shared project name from the directory and treat the other services as
orphans on every `up`.

Because the projects are separate, they would also get separate networks — which would
break `http://qdrant:6333` from inside n8n. Each file therefore puts its service on a
network pinned to the fixed name `ragnet`. The first `up` creates it, the rest join it.
On `down` you will see `Network ragnet ... Resource is still in use` whenever another
service is still attached; that is expected, and the last one down removes it.

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

Volumes are per-project, so reset only what you need:

```bash
docker compose -f docker-compose.n8n.yml    down -v   # wipes n8n workflows
docker compose -f docker-compose.qdrant.yml down -v   # wipes the vector store
docker compose -f docker-compose.ollama.yml down -v   # wipes pulled models (re-pull needed)
```

To tear the whole thing down (the `ragnet` network goes with the last service):

```
docker compose -f docker-compose.n8n.yml    down -v
docker compose -f docker-compose.qdrant.yml down -v
docker compose -f docker-compose.ollama.yml down -v
```
