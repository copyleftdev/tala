# Semantic Recall

Semantic recall is the ability to search your history by *meaning* rather than by string matching.

## The Problem with Regex

Today, if you want to find that command you ran last week to fix the nginx issue, you do:

```bash
history | grep nginx
```

You get every command that contains the string "nginx" — including irrelevant ones. You don't find the `systemctl restart` that actually fixed the problem, because it doesn't contain the word "nginx". You don't find the `journalctl` command that diagnosed the root cause, because it references a different service name.

String matching finds syntax. It doesn't find meaning.

## How Semantic Recall Works

TALA embeds every command into a 384-dimensional vector space. Commands with similar *meaning* — regardless of syntax — land near each other in this space.

A semantic query works like this:

1. Your search query is embedded into the same vector space
2. The HNSW index finds the nearest neighbors by cosine similarity
3. Results are ranked by semantic distance, not string overlap

```
query: "how did I fix the nginx issue?"
→ embedding: [0.12, -0.34, 0.56, ...]
→ HNSW search: top-10, ef=50
→ results:
    0.94  systemctl restart nginx.service
    0.91  journalctl -u nginx --since '1 hour ago'
    0.87  nginx -t
    0.85  vim /etc/nginx/sites-available/app.conf
    0.82  curl -s http://localhost/healthz
```

The search found the entire diagnostic and fix sequence — not because the commands contain the word "nginx", but because they're semantically related to the concept of fixing an nginx issue.

## HNSW Index

TALA uses a Hierarchical Navigable Small World (HNSW) index for approximate nearest neighbor search. Key properties:

- **Sub-millisecond search** — 139 microseconds for 10K vectors, top-10, ef=50
- **High recall** — HNSW provides excellent recall at practical ef values
- **Incremental inserts** — new intents are indexed immediately, no batch rebuilds
- **Configurable quality** — the `ef` parameter trades search time for recall accuracy

## Combining Semantic and Temporal

Semantic recall can be combined with temporal filtering. Find semantically similar commands within a time range:

```
query_semantic(embedding, k=10)  →  nearest by meaning
query_temporal(TimeRange)        →  all intents in a window
```

This lets you ask questions like "what did I do that was similar to this deploy, but only during last week's incident?"
