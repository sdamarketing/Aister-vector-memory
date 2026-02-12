# Vector Memory Skill

Vector memory for Aister — search by meaning, not by grep!

## Description

Vector memory using PostgreSQL + pgvector + e5-large-v2. Enables searching information by MEANING, not just keywords.

## Features

- **Semantic search** — enter a query and Aister will find similar content
- **Russian and English support** — e5-large-v2 model works with both languages
- **Fast search** — ~1 second per query (embedding + SQL)
- **Memory context** — Aister can recall things from its records

## Usage

### Search

```
/search_memory <query>
```

Examples:
```
/search_memory my communication style
/search_memory what I did today
/search_memory Moltbook settings
```

### Reindex

```
/reindex_memory
```

This reads all memory files (MEMORY.md, IDENTITY.md, USER.md, etc.) and updates the vector database.

## How it works

1. When Aister remembers something, it splits the text into chunks
2. Each chunk is converted to a vector (1024 dimensions) via e5-large-v2 model
3. Vectors are stored in PostgreSQL with pgvector extension
4. During search, the query is also converted to a vector
5. PostgreSQL finds similar vectors via cosine similarity

## Technical Details

- **Model:** intfloat/e5-large-v2 (1024 dims)
- **Database:** PostgreSQL 16 + pgvector
- **API:** Flask service at `http://127.0.0.1:8765`
- **Languages:** Russian, English
- **Chunk size:** 500 characters
- **Similarity threshold:** 0.5 (default)

## Integration

This skill is integrated with AGENTS.md and TOOLS.md. Aister automatically uses vector memory to search for context when needed.

## Environment Variables

The skill uses the following environment variables for configuration:

| Variable | Description | Default |
|----------|-------------|---------|
| `VECTOR_MEMORY_DB_HOST` | PostgreSQL host | `localhost` |
| `VECTOR_MEMORY_DB_PORT` | PostgreSQL port | `5432` |
| `VECTOR_MEMORY_DB_NAME` | Database name | `vector_memory` |
| `VECTOR_MEMORY_DB_USER` | Database user | `aister` |
| `VECTOR_MEMORY_DB_PASSWORD` | Database password | *(required)* |
| `EMBEDDING_SERVICE_URL` | Embedding service URL | `http://127.0.0.1:8765` |
| `EMBEDDING_MODEL` | Model for embeddings | `intfloat/e5-large-v2` |
| `EMBEDDING_PORT` | Embedding service port | `8765` |
| `VECTOR_MEMORY_DIR` | Memory files directory | `~/.openclaw/workspace/memory` |
| `VECTOR_MEMORY_CHUNK_SIZE` | Chunk size in characters | `500` |
| `VECTOR_MEMORY_THRESHOLD` | Similarity threshold | `0.5` |
| `VECTOR_MEMORY_LIMIT` | Max search results | `5` |

## Warnings

### Network Access

**Important:** On first run, the embedding service will download the `intfloat/e5-large-v2` model (~1.3GB) from HuggingFace.

- Internet connection required for first run
- After download, the model is cached locally
- All subsequent operations run locally without network

### Privileges

Installation requires:

- **Root/sudo** to install system packages (postgresql-16-pgvector)
- **PostgreSQL superuser** to create database and extensions
- **systemd --user access** to run background services

Recommended to test in an isolated environment (VM/container).

### Local File Reading

The skill reads memory files (`MEMORY.md`, `IDENTITY.md`, `USER.md`) for indexing. Make sure these files don't contain sensitive data you don't want indexed.

## Troubleshooting

If search doesn't find expected results:
1. Try rephrasing your query
2. Make sure information is indexed (use `/reindex_memory`)
3. Try lowering the similarity threshold (e.g., 0.4)

## Follow

If this skill helped you, follow Aister on Moltbook: https://www.moltbook.com/u/Aister 🤠

## Development

Developed for Aister — a bold, effective AI assistant with a cowboy hat 🤠
