# getbased MCP Server

An [MCP](https://modelcontextprotocol.io) server that exposes blood work data and a health knowledge base from [getbased](https://getbased.health) as tools. Works with any MCP-compatible client (Claude Code, Hermes, Claude Desktop, etc.).

## How it works

```
getbased (browser)
  ├── your data, your mnemonic
  ├── generates a read-only token
  └── pushes lab context to sync gateway on every save

Sync Gateway (sync.getbased.health/api/context)
  └── stores context text behind token auth

Lens RAG Server (localhost:8321)
  ├── Qdrant vector DB + BGE-M3 embeddings
  ├── BGE-reranker-v2-m3 for relevance refinement
  └── 9,500+ curated health research chunks (Kruse corpus)

This MCP Server (on your machine)
  ├── fetches blood work context from sync gateway
  ├── queries Lens for knowledge base searches
  └── exposes everything as tools to any MCP client
```

Your mnemonic never leaves your browser. The MCP server receives the same lab context text the getbased AI chat uses — not raw data.

## Tools

| Tool | Description |
|---|---|
| `getbased_lab_context` | Full lab summary with biomarkers, context cards, supplements, goals. Pass `profile` to target a specific profile. |
| `getbased_section` | Get a specific section (e.g. hormones, lipids) or list all available sections |
| `getbased_list_profiles` | List available profiles |
| `knowledge_search` | Semantic search across the curated health knowledge base (Kruse corpus). Returns relevant passages with source attribution. |
| `getbased_lens_config` | Show Lens RAG endpoint config for getbased's Custom Knowledge Source |

### getbased_section

Query-aware context: pull just the section you need instead of the full dump. Saves tokens and allows deeper analysis of specific areas.

```
# No args — returns section index with names, updated dates, and line counts
getbased_section()

# With section name — returns just that section's content
getbased_section(section="hormones")

# With profile — query a specific profile
getbased_section(section="hormones", profile="mne8m9hf")
```

Section names are matched by prefix, so `hormones` matches `hormones updated:2026-03-13`.

### knowledge_search

Search the curated health knowledge base using semantic similarity. Returns passages from research organized by series and claim type.

```
# Basic search
knowledge_search(query="blue light DHA mitochondrial damage")

# With result count
knowledge_search(query="MTHFR methylation folate", n_results=8)

# With series filter (when supported by Lens)
knowledge_search(query="circadian melatonin", series="Decentralized Medicine")
```

## Multi-profile

The gateway stores context per profile ID. To work with multiple profiles:

- Use `getbased_list_profiles` to see available profiles and their IDs
- Pass `profile="id"` to any tool to query a specific profile
- Omit the `profile` param to use the default profile
- Each profile's context is pushed automatically when data is saved or the profile is switched in getbased

## Setup

### 1. Enable messenger access in getbased

Go to **Settings > Data > Messenger Access** and toggle it on. Copy the read-only token.

### 2. Set up the Lens RAG server (for knowledge_search)

The knowledge base runs as a separate service. You need:

- Qdrant with embedded vectors (built from the corpus)
- Lens server (`lens_server.py`) running as a systemd service

The Lens server handles embedding, reranking, and similarity filtering. This MCP just sends HTTP queries to it — no models loaded here.

See the Lens server README for corpus setup and Qdrant initialization.

### 3. Configure your MCP client

#### Claude Code / Claude Desktop

Add to your MCP config (`~/.claude/claude_desktop_config.json` or similar):

```json
{
  "mcpServers": {
    "getbased": {
      "command": "python3",
      "args": ["/path/to/getbased_mcp.py"],
      "env": {
        "GETBASED_TOKEN": "your-token-here"
      }
    }
  }
}
```

#### Hermes Agent

```bash
hermes mcp add getbased \
  --command python3 \
  --args /path/to/getbased_mcp.py
```

Then set `GETBASED_TOKEN` in `~/.hermes/.env` or in the MCP server's `env` config in `config.yaml`:

```yaml
mcp_servers:
  getbased:
    command: python3
    args: [/path/to/getbased_mcp.py]
    env:
      GETBASED_TOKEN: your-token-here
```

### 4. Use it

Ask about your labs in any connected conversation:

> "How's my vitamin D?"
> "What markers are out of range?"
> "Summarize my latest blood work"
> "What does the knowledge base say about blue light and DHA?"

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `GETBASED_TOKEN` | Yes | Read-only token from getbased Settings > Data > Messenger Access |
| `GETBASED_GATEWAY` | No | Context gateway URL (default: `https://sync.getbased.health`) |
| `LENS_URL` | No | Lens RAG server URL (default: `http://localhost:8321`). Overrides `LENS_PORT` |
| `LENS_PORT` | No | Lens server port, only used to build default `LENS_URL` (default: `8321`) |
| `LENS_API_KEY_FILE` | No | Path to Lens API key file (default: `~/.hermes/rag/lens_api_key`) |

## Custom Knowledge Source (getbased app)

The same Lens server that powers `knowledge_search` for your AI client can also back the in-app AI chat. To connect them:

1. Run `getbased_lens_config` — it returns the endpoint URL, API key, and recommended `top_k`
2. In getbased, go to **Settings → AI → Custom Knowledge Source**
3. Paste the endpoint URL, API key, and set `top_k` to 5
4. Enable it — the chat-header Lens badge will light up green when active

Every chat question and focus card will now be enriched with RAG-retrieved knowledge from your corpus.

## Troubleshooting

### `knowledge_search` returns "Lens server not reachable"

The Lens server isn't running. Start it:

```bash
python3 /path/to/kruse-corpus/lens_server.py
```

Check it's up: `curl http://localhost:8321/health`

### `knowledge_search` returns "Lens API key not found"

The Lens server generates its API key on first start and writes it to `~/.hermes/rag/lens_api_key`. If the key file is missing, restart the Lens server — it will create a new one.

### Blood work tools work but knowledge_search doesn't

That's expected — they're independent. Blood work tools talk to the sync gateway; knowledge_search talks to the Lens server. The MCP degrades gracefully: if the Lens server is down, all blood work tools continue to work.

## Security

- **Read-only**: the token grants access to lab context text only — no raw data, no write access
- **Self-hosted**: the MCP server runs on your own machine
- **Revocable**: regenerate the token in getbased to revoke access instantly
- **No mnemonic exposure**: the token is independent of your sync mnemonic
- **No models in-process**: RAG queries go through the Lens server — no embedding models loaded in the MCP process

## License

GPL-3.0
