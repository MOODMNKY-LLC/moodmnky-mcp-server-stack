# MoodMNKY MCP Server Stack

> A **first-principles** guide to running, exposing, and integrating multiple MCP servers in Docker—enabling seamless access via Server-Sent Events (SSE) for OpenAI Agents SDK and other clients.

---

## Table of Contents

1. [Introduction](#introduction)
2. [First Principles of MCP & SSE](#first-principles-of-mcp--sse)

   1. [What Is MCP?](#what-is-mcp)
   2. [Why SSE?](#why-sse)
   3. [Streaming vs. SSE Transport](#streaming-vs-sse-transport)
3. [Repository Structure](#repository-structure)
4. [Prerequisites](#prerequisites)

   1. [Docker & Docker Compose](#docker--docker-compose)
   2. [Cloudflare Tunnel (Cloudflared)](#cloudflare-tunnel-cloudflared)
   3. [Domain & DNS Setup](#domain--dns-setup)
   4. [OpenAI Agents SDK](#openai-agents-sdk)
5. [Environment Variables](#environment-variables)

   1. [`.env.example`](#envexample)
   2. [Creating Your `.env`](#creating-your-env)
   3. [Filesystem MCP Default Path](#filesystem-mcp-default-path)
6. [Getting Started with Docker Compose](#getting-started-with-docker-compose)

   1. [Cloning the Repository](#cloning-the-repository)
   2. [Launching All MCP Servers](#launching-all-mcp-servers)
   3. [Verifying Local `/tools` Endpoints](#verifying-local-tools-endpoints)
7. [Exposing MCP Servers via Cloudflare Tunnel](#exposing-mcp-servers-via-cloudflare-tunnel)

   1. [Why Use a Tunnel?](#why-use-a-tunnel)
   2. [Installing & Authenticating `cloudflared`](#installing--authenticating-cloudflared)
   3. [Creating a Tunnel & DNS Routes](#creating-a-tunnel--dns-routes)
   4. [Cloudflared `config.yml` Example](#cloudflared-configyml-example)
   5. [Starting `cloudflared` as a Service](#starting-cloudflared-as-a-service)
   6. [Verifying Public SSE Endpoints](#verifying-public-sse-endpoints)
8. [Integrating with OpenAI Agents SDK](#integrating-with-openai-agents-sdk)

   1. [Example Python Code Snippet](#example-python-code-snippet)
   2. [Configuring `MCPServerSse`](#configuring-mcpserversse)
9. [SSE vs. Streamable HTTP: Tradeoffs](#sse-vs-streamable-http-tradeoffs)
10. [Troubleshooting & Best Practices](#troubleshooting--best-practices)

    1. [Health Checks](#health-checks)
    2. [Port Conflicts](#port-conflicts)
    3. [Securing Secrets](#securing-secrets)
    4. [Redis Persistence](#redis-persistence)
    5. [Updating MCP Versions](#updating-mcp-versions)
11. [Extending the MCP Stack](#extending-the-mcp-stack)
12. [License](#license)

---

## Introduction

The **MoodMNKY MCP Server Stack** is a standalone repository that orchestrates a diverse array of **Model Context Protocol (MCP)** servers—all containerized via Docker—behind one convenient `docker-compose.yml`. This stack enables:

* **Rapid deployment** of numerous tool‐providing MCP servers (e.g., Notion, Sequential Thinking, Brave Search, Tavily, Firecrawl, Fetch, GitHub, Supabase‐Dev, Context7, YouTube Transcript, Memory via Redis, and more).
* **Unified configuration** through a single `.env` file containing all host‐port bindings, API keys, and paths.
* **Out‐of‐the‐box integration** for **OpenAI Agents SDK** (via SSE) and other SSE‐capable clients.
* **Optional exposure** of each MCP endpoint over the public internet using **Cloudflare Tunnel**—preserving firewall safety while providing secure external access.

This README takes a **first-principles approach**: it explains the “why” behind each step (not just the “how”), so that developers unfamiliar with MCP, SSE, Docker, or Cloudflare Tunnels can still grok exactly what’s happening, why it matters, and how to adapt it to future needs.

---

## First Principles of MCP & SSE

### What Is MCP?

* **MCP (Model Context Protocol)** is an open‐source specification that allows AI **agents** (e.g., GPT‐4‐powered assistants) to call out to external “tool” services.
* Each MCP server publishes a set of **tool signatures** (names, arguments, descriptions) over an HTTP endpoint (typically `/tools`).
* When an agent needs to “CALL” a tool, it sends a JSON‐encoded POST to `/call`, and the MCP server executes the tool (e.g., fetch a web page, query a database, run code) and streams results back.

**Key Benefits**:

1. **Separation of Concerns**: The agent focuses on planning and reasoning; the MCP server handles specialized tasks.
2. **Language‐Agnostic Tools**: MCP servers can be written in any language (Node, Python, Go, etc.) as long as they speak the MCP spec.
3. **Extensibility**: Add new tools by simply adding new MCP servers to your stack; agents discover them at runtime.

---

### Why SSE?

* **SSE (Server‐Sent Events)** is an HTTP‐based, one‐way streaming protocol where the server can push JSON events to the client over a persistent connection.
* In the MCP context, SSE is used for two phases:

  1. **Tool Discovery (LIST)**: The agent sends a GET to `http://<mcp-url>/tools` and listens on the SSE stream for JSON describing each available tool.
  2. **Tool Invocation (CALL)**: The agent sends a POST to `http://<mcp-url>/call` with JSON arguments. The MCP server processes the request, then streams the result back over the existing SSE connection.

**Why SSE over WebSocket or Polling?**

* **Simplicity**: SSE is built on plain HTTP, so it works through most proxies, firewalls, and load balancers without special configuration.
* **Automatic Reconnect**: Native reconnection semantics handle transient network blips.
* **Low Overhead**: No WebSocket handshake or protocol framing—just text‐based events.
* **Unidirectional**: Agents still send tool invocations via normal HTTP. SSE is only used to receive streamed results, which aligns well with MCP’s model.

---

### Streaming vs. SSE Transport

* Some MCP servers implement a separate “streamable‐HTTP” transport (chunked transfer encoding) instead of SSE.
* **SSE (Favored Here)**:

  * Standardized event format (`data: { ... }`).
  * Built‐in reconnection.
  * Works across most HTTP/2‐compatible environments.
* **Streaming/Chunked HTTP**:

  * Simpler for extremely lightweight servers.
  * No built‐in “event” framing—clients must parse chunks manually.
* **Recommendation**: Unless you have a very specific reason, use **SSE**. All reference MCP servers in this stack default to SSE, ensuring uniform behavior.

---

## Repository Structure

```
moodmnky-mcp-server-stack/
├── README.md
├── docker-compose.yml    # Orchestrates all MCP services
└── .env.example          # Template for environment variables
```

* **`README.md`**: (this file) Explains setup, usage, first‐principles rationale, and tutorials.
* **`docker-compose.yml`**: Declares 18 services (MCP servers + Redis). Each references environment variables for secrets, host‐port bindings, or paths.
* **`.env.example`**: Lists all required variables as `<PLACEHOLDER>` values. Copy to `.env`, replace placeholders with real values, never commit your actual `.env`.

---

## Prerequisites

### Docker & Docker Compose

* **Docker Engine** (20.10+) and **Docker Compose** (v2+) installed on your host machine (Linux, macOS, or Windows).
* Confirm installation:

  ```bash
  docker --version
  docker-compose --version
  ```

### Cloudflare Tunnel (Cloudflared)

* If you plan to **expose MCP servers publicly** (e.g., agents hosted remotely), install **Cloudflare’s `cloudflared`** on the same host:

  1. **Download** the latest release for your OS from [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation) documentation.

  2. **Install** (e.g., on Ubuntu):

     ```bash
     curl -LO https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
     sudo dpkg -i cloudflared-linux-amd64.deb
     ```

  3. **Verify**:

     ```bash
     cloudflared version
     ```

### Domain & DNS Setup

* You need a **registered domain** managed by Cloudflare (e.g., `example.com`).
* For each MCP server you wish to expose publicly, you will create a **CNAME** record in Cloudflare’s DNS dashboard pointing to the tunnel’s auto‐provisioned hostname (e.g., `mcp-tunnel.cfargotunnel.com`).

### OpenAI Agents SDK

* To integrate these MCP servers, you need a **Python environment** with the **OpenAI Agents SDK** installed:

  ```bash
  pip install openai-agents
  ```

* Agents will reference MCP endpoints (SSE) in code via `MCPServerSse`.

---

## Environment Variables

### `.env.example`

Below is the complete `.env.example` file containing **placeholder values**. **Do not commit** a file named `.env`. Instead, copy this to `.env` and replace every placeholder (`<...>`) with your actual keys, host ports, and paths.

```ini
###############################################
# Rename this to ".env" and fill in your values
###############################################

###############################################
# 1. Notion MCP Server
###############################################
NOTION_MCP_HEADERS={"Authorization":"Bearer <YOUR_NOTION_PAT>","Notion-Version":"2022-06-28"}
NOTION_HOST_PORT=<YOUR_NOTION_HOST_PORT>  # e.g., 8100

###############################################
# 2. 21st-Dev Magic MCP Server
###############################################
MAGIC_API_KEY=<YOUR_MAGIC_API_KEY>
MAGIC_HOST_PORT=<YOUR_MAGIC_HOST_PORT>    # e.g., 8101

###############################################
# 3. Filesystem MCP Server
###############################################
FILESYSTEM_HOST_PATH=<ABSOLUTE_PATH_TO_SHARE>   # e.g., /home/ubuntu/new-wave-protocol
FILESYSTEM_HOST_PORT=<YOUR_FILESYSTEM_HOST_PORT> # e.g., 8102

###############################################
# 4. Discord MCP Server
###############################################
DISCORD_TOKEN=<YOUR_DISCORD_TOKEN>
DISCORD_BOT_PERMISSIONS=<YOUR_DISCORD_BOT_PERMISSIONS>
DISCORD_HOST_PORT=<YOUR_DISCORD_HOST_PORT>      # e.g., 8103

###############################################
# 5. Sequential Thinking MCP Server
###############################################
SEQUENTIAL_THINKING_HOST_PORT=<YOUR_SEQUENTIAL_THINKING_HOST_PORT>  # e.g., 8104

###############################################
# 6. Brave Search MCP Server
###############################################
BRAVE_API_KEY=<YOUR_BRAVE_API_KEY>
BRAVE_SEARCH_HOST_PORT=<YOUR_BRAVE_SEARCH_HOST_PORT>  # e.g., 8105

###############################################
# 7. Tavily Search MCP Server
###############################################
TAVILY_API_KEY=<YOUR_TAVILY_API_KEY>
TAVILY_HOST_PORT=<YOUR_TAVILY_HOST_PORT>  # e.g., 8106

###############################################
# 8. Firecrawl MCP Server
###############################################
FIRECRAWL_API_URL=<YOUR_FIRECRAWL_API_URL>                 # e.g., https://api.firecrawl.dev/v1
FIRECRAWL_RETRY_MAX_ATTEMPTS=<YOUR_FIRECRAWL_RETRY_MAX_ATTEMPTS>       # e.g., 5
FIRECRAWL_RETRY_INITIAL_DELAY=<YOUR_FIRECRAWL_RETRY_INITIAL_DELAY>     # e.g., 2000
FIRECRAWL_RETRY_MAX_DELAY=<YOUR_FIRECRAWL_RETRY_MAX_DELAY>             # e.g., 30000
FIRECRAWL_RETRY_BACKOFF_FACTOR=<YOUR_FIRECRAWL_RETRY_BACKOFF_FACTOR>   # e.g., 3
FIRECRAWL_CREDIT_WARNING_THRESHOLD=<YOUR_FIRECRAWL_CREDIT_WARNING_THRESHOLD>  # e.g., 2000
FIRECRAWL_CREDIT_CRITICAL_THRESHOLD=<YOUR_FIRECRAWL_CREDIT_CRITICAL_THRESHOLD> # e.g., 500
FIRECRAWL_API_KEY=<YOUR_FIRECRAWL_API_KEY>
FIRECRAWL_HOST_PORT=<YOUR_FIRECRAWL_HOST_PORT>  # e.g., 8107

###############################################
# 9. Fetch MCP Server
###############################################
FETCH_HOST_PORT=<YOUR_FETCH_HOST_PORT>    # e.g., 8108

###############################################
# 10. Playwright MCP Server
###############################################
PLAYWRIGHT_HOST_PORT=<YOUR_PLAYWRIGHT_HOST_PORT>  # e.g., 8109

###############################################
# 11. GitHub MCP Server
###############################################
GITHUB_PERSONAL_ACCESS_TOKEN=<YOUR_GITHUB_PERSONAL_ACCESS_TOKEN>
GITHUB_HOST_PORT=<YOUR_GITHUB_HOST_PORT>  # e.g., 8110

###############################################
# 12. Desktop Commander MCP Server
###############################################
DESKTOP_COMMANDER_HOST_PORT=<YOUR_DESKTOP_COMMANDER_HOST_PORT>  # e.g., 8111

###############################################
# 13. Supabase-Dev (Postgres MCP) Server
###############################################
SUPABASE_DEV_CONN=<YOUR_SUPABASE_DEV_CONNECTION_URL>  # e.g., postgresql://postgres:postgres@127.0.0.1:54322/postgres
SUPABASE_DEV_HOST_PORT=<YOUR_SUPABASE_DEV_HOST_PORT>  # e.g., 8112

###############################################
# 14. Context7 MCP Server
###############################################
CONTEXT7_HOST_PORT=<YOUR_CONTEXT7_HOST_PORT>  # e.g., 8113

###############################################
# 15. YouTube Transcript MCP Server
###############################################
YOUTUBE_TRANSCRIPT_HOST_PORT=<YOUR_YOUTUBE_TRANSCRIPT_HOST_PORT>  # e.g., 8114

###############################################
# 16. Redis (for Memory MCP)
###############################################
REDIS_HOST_PORT=<YOUR_REDIS_HOST_PORT>  # e.g., 6379

###############################################
# 17. Memory MCP Server
###############################################
MEMORY_HOST_PORT=<YOUR_MEMORY_HOST_PORT>  # e.g., 8115

###############################################
# 18. @magicuidesign/mcp MCP Server
###############################################
MAGICUIDESIGN_HOST_PORT=<YOUR_MAGICUIDESIGN_HOST_PORT>  # e.g., 8116
```

> **Note**: The only “production‐ready” values that go into your real `.env` are:
>
> * **API keys & tokens** (Notion PAT, Brave, Tavily, Firecrawl, GitHub, Discord, etc.).
> * **Absolute path** for `FILESYSTEM_HOST_PATH` (e.g., `/home/moodmnky/new-wave-protocol`).
> * **Host ports** are arbitrary but must not conflict with other services on your machine.

---

## Getting Started with Docker Compose

### 1. Clone the Repository

```powershell
git clone https://github.com/<your-org>/moodmnky-mcp-server-stack.git
cd moodmnky-mcp-server-stack
```

### 2. Copy & Edit the `.env`

```powershell
cp .env.example .env
notepad .env
```

* Replace each `<PLACEHOLDER>` with your actual values (API keys, host ports, paths).
* Leave all other lines intact and do **not** commit the resulting `.env` to Git.

### 3. Launch All MCP Servers

```powershell
docker-compose up -d
```

* Docker Compose will:

  1. Pull each image (e.g., `mcp/notion:latest`, `node:18-alpine`, `redis:7-alpine`).
  2. Create & start 18 containers, mapping each container port to the host port set in `.env`.
  3. Mount your host directory into the `filesystem` container at `/mnt/shared`.

### 4. Verify Local `/tools` Endpoints

Open a new PowerShell window or tab and run SSE‐style `curl` for each service:

```powershell
# Notion MCP
curl.exe -N http://localhost:8100/tools

# Magic MCP
curl.exe -N http://localhost:8101/tools

# Filesystem MCP
curl.exe -N http://localhost:8102/tools

# Discord MCP
curl.exe -N http://localhost:8103/tools

# Sequential Thinking MCP
curl.exe -N http://localhost:8104/tools

# Brave Search MCP
curl.exe -N http://localhost:8105/tools

# Tavily MCP
curl.exe -N http://localhost:8106/tools

# Firecrawl MCP
curl.exe -N http://localhost:8107/tools

# Fetch MCP
curl.exe -N http://localhost:8108/tools

# Playwright MCP
curl.exe -N http://localhost:8109/tools

# GitHub MCP
curl.exe -N http://localhost:8110/tools

# Desktop Commander MCP
curl.exe -N http://localhost:8111/tools

# Supabase-Dev (Postgres MCP)
curl.exe -N http://localhost:8112/tools

# Context7 MCP
curl.exe -N http://localhost:8113/tools

# YouTube Transcript MCP
curl.exe -N http://localhost:8114/tools

# Memory MCP (requires Redis)
curl.exe -N http://localhost:8115/tools

# @magicuidesign/mcp
curl.exe -N http://localhost:8116/tools
```

A successful response is an **infinite SSE stream** of JSON objects describing each tool (name, arguments, description). If the connection closes immediately, double‐check that your host port matches and that the container is up.

---

## Exposing MCP Servers via Cloudflare Tunnel

If you want your MCP endpoints to be accessible from outside your local network—while keeping your firewall locked—you can use **Cloudflare Tunnel (cloudflared)**. This section walks through a typical “Tunnel per Service” setup.

### Why Use a Tunnel?

* **No open inbound ports**: You never expose raw TCP/UDP—Cloudflare handles TLS and persistence.
* **TLS termination**: Cloudflare provides certificates for your domain/subdomains.
* **Automatic reconnection**: If the tunnel drops, `cloudflared` auto‐reconnects.
* **Access controls**: Optionally integrate with Zero Trust to restrict who can access your MCP services.

### 1. Install & Authenticate `cloudflared`

1. **Download & install** (example for Ubuntu Linux):

   ```bash
   wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
   sudo dpkg -i cloudflared-linux-amd64.deb
   ```

2. **Login** (this opens a browser tab to authenticate):

   ```bash
   cloudflared tunnel login
   ```

   * Select your Cloudflare account, then choose the domain where you want to create DNS records (e.g., `mcp.example.com`).
   * This step creates a `~/.cloudflared/cert.pem` file on your host.

### 2. Create a Named Tunnel

```bash
cloudflared tunnel create mcp-tunnel
```

* This outputs a **Tunnel ID** (e.g., `abcd1234-ef56-7890-ghij-klmnopqrstuv`).
* A credentials file (e.g., `~/.cloudflared/abcd1234-ef56-7890-ghij-klmnopqrstuv.json`) is created automatically.

### 3. Configure DNS Routes

For each MCP service, we’ll create a Cloudflare DNS CNAME record pointing to `<tunnel_id>.cfargotunnel.com`. Example subdomains:

| MCP Service                 | Local Port | Desired Subdomain               |
| --------------------------- | ---------- | ------------------------------- |
| Notion MCP                  | 8100       | `notion-mcp.example.com`        |
| Magic MCP                   | 8101       | `magic-mcp.example.com`         |
| Filesystem MCP              | 8102       | `filesystem-mcp.example.com`    |
| Discord MCP                 | 8103       | `discord-mcp.example.com`       |
| Sequential Thinking MCP     | 8104       | `sequential-mcp.example.com`    |
| Brave Search MCP            | 8105       | `brave-mcp.example.com`         |
| Tavily MCP                  | 8106       | `tavily-mcp.example.com`        |
| Firecrawl MCP               | 8107       | `firecrawl-mcp.example.com`     |
| Fetch MCP                   | 8108       | `fetch-mcp.example.com`         |
| Playwright MCP              | 8109       | `playwright-mcp.example.com`    |
| GitHub MCP                  | 8110       | `github-mcp.example.com`        |
| Desktop Commander MCP       | 8111       | `desktop-mcp.example.com`       |
| Supabase-Dev (Postgres MCP) | 8112       | `supabase-mcp.example.com`      |
| Context7 MCP                | 8113       | `context7-mcp.example.com`      |
| YouTube Transcript MCP      | 8114       | `yt-transcript-mcp.example.com` |
| Memory MCP                  | 8115       | `memory-mcp.example.com`        |
| @magicuidesign/mcp          | 8116       | `magicuidesign-mcp.example.com` |

Run the following commands (replace `<tunnel-id>` with your actual Tunnel ID and each subdomain accordingly):

```bash
cloudflared tunnel route dns mcp-tunnel notion-mcp.example.com
cloudflared tunnel route dns mcp-tunnel magic-mcp.example.com
cloudflared tunnel route dns mcp-tunnel filesystem-mcp.example.com
cloudflared tunnel route dns mcp-tunnel discord-mcp.example.com
cloudflared tunnel route dns mcp-tunnel sequential-mcp.example.com
cloudflared tunnel route dns mcp-tunnel brave-mcp.example.com
cloudflared tunnel route dns mcp-tunnel tavily-mcp.example.com
cloudflared tunnel route dns mcp-tunnel firecrawl-mcp.example.com
cloudflared tunnel route dns mcp-tunnel fetch-mcp.example.com
cloudflared tunnel route dns mcp-tunnel playwright-mcp.example.com
cloudflared tunnel route dns mcp-tunnel github-mcp.example.com
cloudflared tunnel route dns mcp-tunnel desktop-mcp.example.com
cloudflared tunnel route dns mcp-tunnel supabase-mcp.example.com
cloudflared tunnel route dns mcp-tunnel context7-mcp.example.com
cloudflared tunnel route dns mcp-tunnel yt-transcript-mcp.example.com
cloudflared tunnel route dns mcp-tunnel memory-mcp.example.com
cloudflared tunnel route dns mcp-tunnel magicuidesign-mcp.example.com
```

> **Note**: You must first create a corresponding **CNAME** record in Cloudflare’s dashboard (DNS → Add record → Type “CNAME” → Name = subdomain → target = `<tunnel-id>.cfargotunnel.com`). Alternatively, Cloudflare may provision it automatically when you run the `route dns` commands.

### 4. Create `cloudflared` Configuration

Create a file at `~/.cloudflared/config.yml` with the following content. Replace `<tunnel-id>` with your actual Tunnel ID, and adjust `hostname` entries to match your chosen subdomains:

```yaml
tunnel: <tunnel-id>
credentials-file: /home/<your-username>/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: notion-mcp.example.com
    service: http://localhost:8100

  - hostname: magic-mcp.example.com
    service: http://localhost:8101

  - hostname: filesystem-mcp.example.com
    service: http://localhost:8102

  - hostname: discord-mcp.example.com
    service: http://localhost:8103

  - hostname: sequential-mcp.example.com
    service: http://localhost:8104

  - hostname: brave-mcp.example.com
    service: http://localhost:8105

  - hostname: tavily-mcp.example.com
    service: http://localhost:8106

  - hostname: firecrawl-mcp.example.com
    service: http://localhost:8107

  - hostname: fetch-mcp.example.com
    service: http://localhost:8108

  - hostname: playwright-mcp.example.com
    service: http://localhost:8109

  - hostname: github-mcp.example.com
    service: http://localhost:8110

  - hostname: desktop-mcp.example.com
    service: http://localhost:8111

  - hostname: supabase-mcp.example.com
    service: http://localhost:8112

  - hostname: context7-mcp.example.com
    service: http://localhost:8113

  - hostname: yt-transcript-mcp.example.com
    service: http://localhost:8114

  - hostname: memory-mcp.example.com
    service: http://localhost:8115

  - hostname: magicuidesign-mcp.example.com
    service: http://localhost:8116

  # Fallback for any unmatched request
  - service: http_status:404
```

### 5. Run `cloudflared` as a System Service

#### On Linux (example for Ubuntu)

1. **Create a systemd unit file** at `/etc/systemd/system/cloudflared.service`:

   ```bash
   sudo tee /etc/systemd/system/cloudflared.service > /dev/null << 'EOF'
   [Unit]
   Description=Cloudflare Tunnel
   After=network.target

   [Service]
   Type=notify
   ExecStart=/usr/local/bin/cloudflared tunnel --config /home/<your-username>/.cloudflared/config.yml run
   Restart=on-failure
   RestartSec=5s

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. **Reload systemd** and **enable/start** the service:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable cloudflared
   sudo systemctl start cloudflared
   sudo systemctl status cloudflared
   ```

3. Ensure the service is **“active (running)”**. If so, Cloudflare Tunnel is now forwarding your public subdomains to localhost.

---

## Integrating with OpenAI Agents SDK

Once your MCP servers are running locally (and optionally exposed via Cloudflare Tunnels), you can connect an **OpenAI Agent** to them via `MCPServerSse`. Below is a Python snippet demonstrating how to wire in multiple MCP endpoints:

```python
import os
import asyncio
from agents import Agent, Runner
from agents.mcp.server import MCPServerSse

# 1. Read MCP SSE URLs (either localhost or public via Cloudflare)
NOTION_MCP_URL       = os.getenv("NOTION_SSE_URL", "http://localhost:8100")
MAGIC_MCP_URL        = os.getenv("MAGIC_SSE_URL",  "http://localhost:8101")
FILESYSTEM_MCP_URL   = os.getenv("FILESYSTEM_SSE_URL", "http://localhost:8102")
DISCORD_MCP_URL      = os.getenv("DISCORD_SSE_URL", "http://localhost:8103")
SEQUENTIAL_MCP_URL   = os.getenv("SEQUENTIAL_SSE_URL", "http://localhost:8104")
BRAVE_MCP_URL        = os.getenv("BRAVE_SSE_URL", "http://localhost:8105")
TAVILY_MCP_URL       = os.getenv("TAVILY_SSE_URL", "http://localhost:8106")
FIRECRAWL_MCP_URL    = os.getenv("FIRECRAWL_SSE_URL", "http://localhost:8107")
FETCH_MCP_URL        = os.getenv("FETCH_SSE_URL", "http://localhost:8108")
PLAYWRIGHT_MCP_URL   = os.getenv("PLAYWRIGHT_SSE_URL", "http://localhost:8109")
GITHUB_MCP_URL       = os.getenv("GITHUB_SSE_URL", "http://localhost:8110")
DESKTOP_MCP_URL      = os.getenv("DESKTOP_SSE_URL", "http://localhost:8111")
SUPABASE_MCP_URL     = os.getenv("SUPABASE_SSE_URL", "http://localhost:8112")
CONTEXT7_MCP_URL     = os.getenv("CONTEXT7_SSE_URL", "http://localhost:8113")
YOUTUBE_MCP_URL      = os.getenv("YOUTUBE_SSE_URL", "http://localhost:8114")
MEMORY_MCP_URL       = os.getenv("MEMORY_SSE_URL", "http://localhost:8115")
MAGICUIDESIGN_MCP_URL= os.getenv("MAGICUIDESIGN_SSE_URL", "http://localhost:8116")

def make_sse_server(url: str, name: str):
    return MCPServerSse(params={"url": url}, name=name)

async def main():
    # 2. Instantiate MCP servers
    mcp_servers = [
        make_sse_server(NOTION_MCP_URL,       "NotionMCP"),
        make_sse_server(MAGIC_MCP_URL,        "MagicMCP"),
        make_sse_server(FILESYSTEM_MCP_URL,   "FileMCP"),
        make_sse_server(DISCORD_MCP_URL,      "DiscordMCP"),
        make_sse_server(SEQUENTIAL_MCP_URL,   "SequentialMCP"),
        make_sse_server(BRAVE_MCP_URL,        "BraveMCP"),
        make_sse_server(TAVILY_MCP_URL,       "TavilyMCP"),
        make_sse_server(FIRECRAWL_MCP_URL,    "FirecrawlMCP"),
        make_sse_server(FETCH_MCP_URL,        "FetchMCP"),
        make_sse_server(PLAYWRIGHT_MCP_URL,   "PlaywrightMCP"),
        make_sse_server(GITHUB_MCP_URL,       "GitHubMCP"),
        make_sse_server(DESKTOP_MCP_URL,      "DesktopMCP"),
        make_sse_server(SUPABASE_MCP_URL,     "SupabaseMCP"),
        make_sse_server(CONTEXT7_MCP_URL,     "Context7MCP"),
        make_sse_server(YOUTUBE_MCP_URL,      "YouTubeMCP"),
        make_sse_server(MEMORY_MCP_URL,       "MemoryMCP"),
        make_sse_server(MAGICUIDESIGN_MCP_URL,"MagicUIDesignMCP"),
    ]

    # 3. Build an Agent with multiple tool sources
    agent = Agent(
        name="MoodMNKYUnifiedAgent",
        instructions=(
            "You have access to multiple MCP servers over SSE. "
            "Use NotionMCP to query Notion data, FileMCP to read files, "
            "MemoryMCP to store/retrieve memory, etc."
        ),
        tools=[],           # No built-in local tools
        mcp_servers=mcp_servers
    )

    # 4. Run a simple example prompt
    runner = Runner(agent)
    response = await runner.complete(
        "List all Notion pages, then fetch the contents of README.md from the filesystem."
    )
    print(response)

if __name__ == "__main__":
    asyncio.run(main())
```

* **Environment Variables**: You can override each `*_SSE_URL` to point to a **Cloudflare‐tunneled** URL (e.g., `https://notion-mcp.example.com`).
* **MCPServerSse**: Under the hood, `MCPServerSse.connect()` will open a persistent HTTP GET to `http://<url>/tools`, parse the JSON stream, and expose each tool. When the agent does `CALL <tool>`, the SDK POSTs to `http://<url>/call`, and the server streams results back.

---

## SSE vs. Streamable HTTP: Tradeoffs

1. **SSE (Server-Sent Events)**

   * **Protocol**: HTTP GET to `/tools` returns a continuous stream of `data: { ... }` events in text/event-stream format.
   * **Pros**:

     * Native auto‐reconnect semantics.
     * Works over HTTP/2 and through most proxies/load balancers.
     * Clear “event” framing.
   * **Cons**:

     * Only server → client. Agents must use regular POST for sending tool invocations.
2. **Streamable HTTP (Chunked Response)**

   * **Protocol**: Client sends a GET or POST; server responds with chunked transfer encoding (no explicit `event:` framing).
   * **Pros**:

     * Slightly simpler implementation in some languages.
   * **Cons**:

     * Clients must parse raw chunks manually.
     * No built‐in “reconnect” semantics.

**Why SSE Is Preferred** for most MCP servers:

* Uniform, well‐documented format across reference implementations.
* Automatic handling of reconnect after interruptions.
* Agent SDKs (e.g., OpenAI Agents SDK) natively support SSE (via `MCPServerSse`).

---

## Troubleshooting & Best Practices

### 1. Health Checks

Add a healthcheck to each service in `docker-compose.yml` to verify that `/tools` returns a 200 status (SSE handshake successful). Example:

```yaml
services:
  new-wave-protocol-notion:
    # … existing config …
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${NOTION_HOST_PORT}/tools"]
      interval: 30s
      timeout: 5s
      retries: 3
```

Repeat for other critical services.

---

### 2. Port Conflicts

* Ensure that **every** `${*_HOST_PORT}` in your `.env` is unique and not used by other local services.
* Use `ss -tulpn` (Linux) or `netstat -anb` (Windows) to verify port availability before assigning.

---

### 3. Securing Secrets

* **Never commit** your real `.env` file. Add it to `.gitignore` (provided).
* For **production**, consider:

  * **Docker Secrets**: Store API keys in Docker’s secret management, not in plain text.
  * **Vault Solutions**: HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault for dynamic injection at runtime.

---

### 4. Redis Persistence

* The `redis` service uses a named volume `redis-data` for durability. To back up or migrate:

  * **Backup**: `docker run --rm -v redis-data:/data -v $(pwd):/backup alpine:latest sh -c "cd /data && tar czf /backup/redis-backup.tar.gz ."`
  * **Restore**: Stop Redis, replace `redis-data` with extracted data, then restart.

---

### 5. Updating MCP Versions

* By default, `docker-compose.yml` uses `mcp/<service>:latest`. For stability, pin to a specific tag or digest:

  ```yaml
  image: mcp/notion:0.2.1
  ```
* Monitor upstream MCP repos for new releases:

  * [mcp/notion](https://hub.docker.com/r/mcp/notion)
  * [mcp/fetch](https://hub.docker.com/r/mcp/fetch)
  * [mcp/firecrawl](https://hub.docker.com/r/mcp/firecrawl)
  * etc.

When a new release arrives, update `docker-compose.yml` accordingly, test locally, then bump version in Git.

---

## Extending the MCP Stack

As **new MCP servers** or **tool implementations** emerge, you can extend this stack by:

1. **Adding a new service** in `docker-compose.yml` (or creating a separate `Dockerfile` if necessary).
2. **Adding matching variables** to `.env.example` (and then to your local `.env`).
3. **Testing** locally with `curl -N http://localhost:<NEW_PORT>/tools`.
4. **Adding a DNS route** & `cloudflared` configuration if you need public exposure.

#### Example: Adding a “Slack MCP Server”

1. **Append to `docker-compose.yml`**:

   ```yaml
   slack-mcp:
     image: mcp/slack:latest
     container_name: mcp-slack
     restart: unless-stopped
     ports:
       - "${SLACK_HOST_PORT}:8080"
     environment:
       SLACK_BOT_TOKEN: "${SLACK_BOT_TOKEN}"
       SLACK_SIGNING_SECRET: "${SLACK_SIGNING_SECRET}"
   ```

2. **Update `.env.example`**:

   ```ini
   ###############################################
   # 19. Slack MCP Server
   ###############################################
   SLACK_BOT_TOKEN=<YOUR_SLACK_BOT_TOKEN>
   SLACK_SIGNING_SECRET=<YOUR_SLACK_SIGNING_SECRET>
   SLACK_HOST_PORT=<YOUR_SLACK_HOST_PORT>  # e.g., 8117
   ```

3. **Pull changes** (if you cloned earlier) and **extend** your `.env` accordingly.

---

## License

This repository is distributed under the **MIT License**. See [LICENSE](LICENSE) for details.

---

### Conclusion

By following this **first-principles** README, you will:

* Understand **why** each component (MCP, SSE, Docker, Cloudflare Tunnel) exists.
* Know **how** to configure, launch, and expose your MCP servers.
* Be able to connect any SSE‐capable client (OpenAI Agents SDK, Node/Python scripts, etc.) to a rich ecosystem of tools.
* Have a clear, extensible foundation for adding future MCP servers and integrations.

Welcome to a fully containerized, secure, and extensible world of MCP servers—where your agents can call any “tool” they need, and you remain in full control of your environment.
