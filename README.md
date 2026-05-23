# Fast Simple Router for Open WebUI

A lightweight, intelligent filter that routes user queries to the best expert model (each with its own RAG knowledge base), improving answer accuracy and reducing hallucinations.

## Features

- Automatic expert selection using a small LLM (`llama3.2:3b`), with a keyword fallback for robustness.
- Domain‑aware query rewriting — transforms the user's question into a precise, page‑title‑like phrase for better vector search.
- Built‑in RAG guidance — inserts a system message that tells the model to use only the expert's dedicated knowledge collection.
- Conversation continuity — a hybrid threshold prevents unnecessary expert switches during a chat.
- Performance optimisations:
  - Async‑safe classification cache with configurable max size and TTL.
  - Async‑safe rewrite cache with configurable max size and TTL.
  - Minimal tokens and temperature for fast generation.
- Configurable via Open WebUI valves — no need to edit code to add, remove, or change experts.
- Debug mode for detailed logging.

## How it works

### 1. Classification
- First, checks an async‑safe cache of previous queries (with TTL expiry).
- If not cached, asks the small LLM ("router") to pick the best expert (or `generalista`).
- Falls back to keyword matching if the LLM fails.

### 2. Query rewriting *(optional, enabled by default)*
- If the expert has a `knowledge_base` description, the small LLM rewrites the query into a short, technical phrase that matches that knowledge base.
- Results are cached by `(query, expert_id)`, with configurable max size and TTL.

### 3. Expert switch decision
- A `change_threshold` checks shared words with the last user messages; if the topic seems stable, the current expert is kept.

### 4. RAG guidance *(optional, enabled by default)*
- A system message is inserted instructing the model to answer only from the expert's knowledge collection (e.g., `archwiki`).

### 5. Model assignment
- The `model` field in the API body is set to the selected expert (or `generalista`).
- A notification is shown only when the expert actually changes.

## Valves

All configuration is done through Open WebUI's valve interface.

| Valve | Default | Description |
|---|---|---|
| `experts_json` | JSON with Neovim and Arch Linux examples | JSON array of expert definitions (see below). |
| `default_model` | `generalista` | Model ID used when no expert matches. |
| `change_threshold` | `2` | Minimum shared words with previous context to stay on the current expert. |
| `notify_change` | `true` | Show a UI notification when the expert changes. |
| `notification_template` | `🎯 Using expert: {expert}` | Template for the notification; `{expert}` is replaced by the expert name. |
| `LLM_BASE_URL` | `http://host.docker.internal:11434/` | Base URL for the LLM provider (Ollama or OpenAI‑compatible). |
| `LLM_API_TOKEN` | `""` | API token if required (leave empty for local Ollama). |
| `classifier_model` | `ollama/llama3.2:3b` | Small model used for classification and query rewriting. |
| `classifier_temperature` | `0.0` | Temperature for the classifier LLM (deterministic). |
| `classifier_timeout` | `10` | Timeout (seconds) for classifier LLM calls. |
| `enable_query_rewriting` | `true` | Rewrite the user query before RAG retrieval. |
| `enable_rag_injection` | `true` | Insert a system message that forces the model to use only the expert's knowledge collection. |
| `string_cache_max_size` | `1000` | Maximum number of cached expert classification results. Set to `0` for unlimited. |
| `string_cache_ttl` | `1800` | Time-to-live (seconds) for classification cache entries. Set to `0` for no expiry. |
| `rewrite_cache_max_size` | `500` | Maximum number of cached rewritten queries (keyed by original query + expert). |
| `rewrite_cache_ttl` | `3600` | Time-to-live (seconds) for rewritten query cache entries. Set to `0` for no expiry. |
| `DEBUG` | `false` | Enable verbose logging to help troubleshooting. |

## Expert JSON format

Each expert in `experts_json` is an object with these fields:

| Field | Required | Description |
|---|---|---|
| `id` | Yes | Unique identifier (used as the model name). |
| `name` | Yes | Human‑readable name. |
| `keywords` | Yes | List of keywords for the fallback matcher. |
| `description` | No | Short description of the expert's domain (helps the classifier). |
| `examples` | No | List of example user queries (few‑shot for the classifier). |
| `knowledge_base` | No | Description of the knowledge base (used for query rewriting). |
| `collection_name` | No | Name of the Open WebUI collection to which the RAG guidance should point. |

**Example:**

```json
{
  "id": "experto-en-arch-linux",
  "name": "Arch Linux",
  "keywords": ["arch", "aur", "pacman", "yay"],
  "description": "Installation, maintenance, and troubleshooting of Arch Linux",
  "examples": ["how to install yay", "pacman update error"],
  "knowledge_base": "the official Arch Linux wiki",
  "collection_name": "archwiki"
}
```

## Setup & Usage

1. Add the filter in Open WebUI (**Workspace → Functions → Add Function**) and paste the Python code.
2. Edit the valves to match your environment:
   - `LLM_BASE_URL` – your Ollama endpoint (e.g., `http://localhost:11434/`).
   - `classifier_model` – any small, fast model (tested with `llama3.2:3b`).
   - `experts_json` – customise with your own experts and collection names.
3. Enable the filter for the chats where you want automatic routing.
4. Ensure the expert's collection exists in Open WebUI (upload documents and assign them to the collection named in `collection_name`).

The filter will now intercept every message, classify it, rewrite it, and guide the RAG accordingly.

## Performance notes

- The small LLM (`llama3.2:3b`) runs on CPU and adds about 1–2 seconds per uncached query. With caches enabled, repeated queries are instant.
- Both caches (classification and rewrite) are async‑safe and support independent TTL and max size configuration. Setting TTL to `0` disables expiry; setting max size to `0` allows unlimited entries.
- Cache configuration is synchronised automatically at each request — no restart needed after changing valve values.
- Embedding generation is not used by the filter (Open WebUI handles it internally), so no extra GPU/CPU load.
- If VRAM is limited, the classifier model may swap — see Open WebUI's documentation on model keep‑alive.

## Debugging

Set `DEBUG = true` to see detailed logs in the Open WebUI server console, including classifier decisions, rewrite results, and cache hits/misses.

---

*License: GPL3 (feel free to adapt).*
