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
knowledge_search(query="MTHFR methylation folate", n_results=12)

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
| `LENS_URL` | No | Lens RAG server URL (default: `http://localhost:8321`) |
| `LENS_PORT` | No | Lens server port, used to build default LENS_URL (default: `8321`) |
| `LENS_API_KEY_FILE` | No | Path to Lens API key file (default: `~/.hermes/rag/lens_api_key`) |

## Security

- **Read-only**: the token grants access to lab context text only — no raw data, no write access
- **Self-hosted**: the MCP server runs on your own machine
- **Revocable**: regenerate the token in getbased to revoke access instantly
- **No mnemonic exposure**: the token is independent of your sync mnemonic
- **No models in-process**: RAG queries go through the Lens server — no embedding models loaded in the MCP process

## License

GPL-3.0
