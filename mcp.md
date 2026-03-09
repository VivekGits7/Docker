# MCP Server Installation Methods for Claude Code

There are **6 methods** to add MCP servers to Claude Code.
This guide covers **Method 3** and **Method 5** in detail.

---

## All 6 Methods (Overview)

| # | Method | Persistent? | Speed | Best for |
|---|--------|------------|-------|----------|
| 1 | npx | No | Slow | Quick testing |
| 2 | docker run | No | Medium | Occasional use |
| **3** | **Compose + docker exec** | **Yes** | **Fast** | **Daily use** |
| 4 | HTTP/SSE | Yes | Depends | Remote/cloud servers |
| **5** | **Docker MCP Toolkit** | **Yes** | **Fast** | **GUI, easiest setup** |
| 6 | Local binary | Yes | Fast | Custom servers |

---
---

# METHOD 3: Docker Compose + `docker exec` (Recommended)

Containers run permanently in the background. Claude/Gemini connect via `docker exec`.
Fastest response, full control, works with both CLIs.

---

### Step 1: Create `.env` file

```bash
cd /Users/vivek/Documents/Docker/mcp
cp .env.example .env
```

Edit `.env` and add your tokens:

```env
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
POSTGRES_CONNECTION_STRING=postgresql://user:password@host:5432/database
```

**How to get GitHub token:**
1. Go to https://github.com/settings/tokens
2. Click "Generate new token (classic)"
3. Select scopes: `repo`, `workflow`, `read:org`
4. Copy the token into `.env`

---

### Step 2: Start MCP servers

```bash
cd /Users/vivek/Documents/Docker/mcp

# Start all servers
docker compose up -d

# Check they are running
docker ps --filter "name=mcp-"
```

You should see containers: `mcp-github`, `mcp-docker`, `mcp-filesystem`, `mcp-postgres`

To start only specific servers:

```bash
# Only GitHub + Docker
docker compose up -d mcp-github mcp-docker
```

---

### Step 3: Add to Claude Code

Run these commands in your **terminal** (not inside Claude Code):

```bash
# GitHub MCP Server
claude mcp add github --scope user -- docker exec -i mcp-github /ko-app/github-mcp-server stdio

# Docker MCP Server
claude mcp add docker --scope user -- docker exec -i mcp-docker docker-mcp-server

# Filesystem MCP Server
claude mcp add filesystem --scope user -- docker exec -i mcp-filesystem npx -y @modelcontextprotocol/server-filesystem /data

# Postgres MCP Server
claude mcp add postgres --scope user -- docker exec -i mcp-postgres npx -y @modelcontextprotocol/server-postgres
```

> `--scope user` makes the server available across all your projects.

---

### Step 3 (Alternative): Edit `~/.claude.json` directly

If you prefer editing the config file:

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": ["exec", "-i", "mcp-github", "/ko-app/github-mcp-server", "stdio"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      }
    },
    "docker": {
      "command": "docker",
      "args": ["exec", "-i", "mcp-docker", "docker-mcp-server"]
    },
    "filesystem": {
      "command": "docker",
      "args": ["exec", "-i", "mcp-filesystem", "npx", "-y", "@modelcontextprotocol/server-filesystem", "/data"]
    },
    "postgres": {
      "command": "docker",
      "args": ["exec", "-i", "mcp-postgres", "npx", "-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "postgresql://user:pass@host:5432/mydb"
      }
    }
  }
}
```

---

### Step 4: Add to Gemini CLI (same servers, same containers)

Edit `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": ["exec", "-i", "mcp-github", "/ko-app/github-mcp-server", "stdio"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      }
    },
    "docker": {
      "command": "docker",
      "args": ["exec", "-i", "mcp-docker", "docker-mcp-server"]
    }
  }
}
```

Both Claude Code and Gemini CLI now share the **same running containers**.

---

### Step 5: Verify everything works

```bash
# Check servers are running
docker ps --filter "name=mcp-"

# Check Claude Code sees them
claude mcp list

# Inside Claude Code, type:
/mcp
```

---

### Managing servers

```bash
# Stop all MCP servers
docker compose -f /Users/vivek/Documents/Docker/mcp/docker-compose.yml down

# Restart a specific server
docker restart mcp-github

# View logs
docker logs mcp-github

# Start servers again
docker compose -f /Users/vivek/Documents/Docker/mcp/docker-compose.yml up -d
```

---
---

# METHOD 5: Docker MCP Toolkit (Easiest)

Docker Desktop extension with 200+ pre-built MCP servers.
One-click enable, no docker-compose files needed, auto-managed.

---

### Prerequisites

- Docker Desktop **v4.48 or newer**
- Check your version: Docker Desktop → Settings → About

---

### Step 1: Install MCP Toolkit extension

1. Open **Docker Desktop**
2. Click **Extensions** in the left sidebar
3. Search for **"MCP Toolkit"**
4. Click **Install**

---

### Step 2: Enable MCP servers

1. Open the **MCP Toolkit** extension in Docker Desktop
2. Browse the catalog (200+ servers available)
3. **Toggle ON** the servers you want:
   - GitHub
   - Docker
   - Filesystem
   - Postgres
   - Puppeteer
   - Slack
   - and many more...
4. For servers that need credentials (like GitHub), the toolkit will ask you to enter your token in the UI

---

### Step 3: Add to Claude Code

Run in your **terminal**:

```bash
claude mcp add-json docker-mcp-toolkit '{"command":"docker","args":["mcp","gateway","run"]}'
```

Or add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "docker-mcp-toolkit": {
      "command": "docker",
      "args": ["mcp", "gateway", "run"]
    }
  }
}
```

This single entry gives Claude Code access to **ALL** servers you enabled in the toolkit.

---

### Step 4: Add to Gemini CLI

Edit `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "docker-mcp-toolkit": {
      "command": "docker",
      "args": ["mcp", "gateway", "run"]
    }
  }
}
```

---

### Step 5: Verify

```bash
# Inside Claude Code
/mcp

# Or in terminal
claude mcp list
```

---

### Managing servers

- Open Docker Desktop → MCP Toolkit extension
- Toggle servers ON/OFF as needed
- Update credentials in the UI
- No docker-compose files to manage

---
---

# Method 3 vs Method 5 Comparison

| Feature | Method 3 (Compose) | Method 5 (Toolkit) |
|---------|-------------------|-------------------|
| Setup effort | Medium (edit files) | Easy (click buttons) |
| Server count | Add manually | 200+ pre-built |
| Customization | Full control | Limited to catalog |
| Docker Desktop needed | No (just Docker Engine) | Yes (v4.48+) |
| Config files | docker-compose.yml + .env | None (GUI managed) |
| Credential handling | .env file | Built-in UI |
| Works with Claude | Yes | Yes |
| Works with Gemini | Yes | Yes |

**Recommendation:** Start with **Method 5** for quick setup. Switch to **Method 3** when you need more control or custom servers.
