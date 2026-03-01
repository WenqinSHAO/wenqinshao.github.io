---
layout: post
title: Self-hosted Web search within Claude Code
date: 2026-03-01
---

本着省钱干大事的心态，我用 Claude Code CLI + DeepSeek API 体验了一把“天花板” Agentic CLI App。

核心痛点是使用非 Anthropic backend，内置 web search不可用，实际使用会受限。  
尝试了一个低成本方案：`SearXNG + MCP`，不依赖公共 MCP server，也不需要额外订阅。

这篇是偏实操的参考笔记，不展开原理。

> Fact-check snapshot (2026-03-01)
> - Claude Code MCP 目前官方文档使用 `--scope local|project|user`，旧说法 `global` 对应现在的 `user`。
> - MCP 的 `local/user` scope 都存放在 `~/.claude.json`；`project` scope 存放在项目根目录 `.mcp.json`。
> - SearXNG 若未启用 `json` format，`format=json` 请求会被拒绝（常见是 403）。

# SearXNG + Claude Code MCP — Reference Notes

---

## 1. SearXNG Docker Config

Create the config file before running the container:

```bash
mkdir -p ~/searxng-config
cat > ~/searxng-config/settings.yml << 'EOF'
use_default_settings: true

search:
  formats:
    - html
    - json          # REQUIRED — MCP needs JSON format

server:
  secret_key: "changeme123"
  limiter: false    # Disable rate limiter for local use

engines:
  - name: google
    disabled: false
  - name: bing
    disabled: false
  - name: duckduckgo
    disabled: false
  - name: brave
    disabled: true  # Flaky, tends to timeout

outgoing:
  request_timeout: 10.0
  max_request_timeout: 20.0
  pool_connections: 100
  pool_maxsize: 20
EOF
```

> **Important:** `json` must be listed under `search.formats`, otherwise requests with `format=json` may fail (commonly `403 Forbidden`).

---

## 2. Run SearXNG Docker — Status & Testing

### Start the container

```bash
docker run -d \
  -p 8080:8080 \
  -v ~/searxng-config/settings.yml:/etc/searxng/settings.yml:ro \
  -e SEARXNG_BASE_URL=http://localhost:8080/ \
  --name searxng \
  --restart unless-stopped \
  --dns 8.8.8.8 \
  --dns 1.1.1.1 \
  searxng/searxng
```

### Check status

```bash
docker ps | grep searxng          # Is it running?
docker logs searxng               # View logs
docker logs searxng --tail 50     # Last 50 lines
```

### Test JSON search from host

```bash
# Quick test
curl "http://localhost:8080/search?q=test&format=json" | head -c 200

# Pretty-printed with result count
curl "http://localhost:8080/search?q=python&format=json" | python3 -c "
import json,sys
d=json.load(sys.stdin)
print('Results:', len(d['results']))
print('Unresponsive engines:', d['unresponsive_engines'])
if d['results']: print('First result:', d['results'][0]['url'])
"
```

### Container management

```bash
docker stop searxng
docker start searxng
docker restart searxng
docker rm -f searxng             # Remove (need to re-run docker run after)
```

### Network troubleshooting — MTU (USB tethering issue)

If running on a machine connected via USB tethering, Docker SSL connections may fail due to MTU mismatch. Fix:

```bash
# Find your tether interface (look for enx... prefix)
ip link show

# Lower MTU on tether interface (replace enx0a01... with your interface name)
sudo ip link set enx0a01facbfb67 mtu 1400

# Set Docker daemon MTU globally
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "mtu": 1400,
  "dns": ["8.8.8.8", "1.1.1.1"]
}
EOF
sudo systemctl restart docker
```

> **Note:** WiFi hotspot often avoids this issue. In my setup, USB tethering triggered MTU fragmentation and broke Docker SSL.

---

## 3. Add SearXNG MCP to Claude Code

### User scope (recommended — works in all projects)

Use `--scope user` in current Claude Code docs.

```bash
claude mcp add --scope user searxng \
  --env SEARXNG_URL=http://localhost:8080 \
  -- npx -y mcp-searxng
```

### Project-scoped (current directory only)

```bash
claude mcp add --scope project searxng \
  --env SEARXNG_URL=http://localhost:8080 \
  -- npx -y mcp-searxng
```

### Verify it was added

```bash
claude mcp list
claude mcp get searxng
```

> Compatibility note: older posts/versions may still show `--global`; that maps to current `--scope user`.

---

## 4. Troubleshoot MCP Within Claude

### Inside a Claude Code session

```bash
/mcp          # List all connected MCP servers and their status
/doctor       # Run diagnostics — shows Claude version, config path, search status
```

### Common issues and fixes

**MCP not showing after manual config edits**
- Prefer `claude mcp add ...` over hand-editing random settings files
- For shared config, use `.mcp.json` (`--scope project`)
- For personal cross-project config, use `--scope user` (stored in `~/.claude.json`)

**MCP added but only works in one directory**
- It was likely added in local/project scope
- Re-add with `--scope user` if you want it across projects

**Web search shows "Did 0 searches"**
- This can happen when your active backend/tooling path doesn’t expose Anthropic web search
- Fix: use an MCP-based search server (like SearXNG), which is backend-agnostic from Claude Code’s perspective

**MCP server shows as disconnected**
- Check SearXNG container is running: `docker ps | grep searxng`
- Check npx can resolve the package: `npx -y mcp-searxng --help`
- Check the SEARXNG_URL is reachable: `curl http://localhost:8080/search?q=test&format=json`

---

## 5. User vs Project MCP Configuration — Best Practices

### Config file locations

| File | Scope | Used for |
|---|---|---|
| `~/.claude.json` | User + local | MCP servers (user scope and local scope entries) |
| `.mcp.json` (project root) | Project | Team-shared MCP servers |
| `.claude/settings.local.json` | Local settings | General Claude settings (not MCP server list) |

### When to use user scope (`--scope user`)

- Developer tools you use everywhere: search, file system, git helpers
- MCP servers tied to your machine (e.g. locally running SearXNG)
- Personal API integrations (e.g. your own Brave Search key)

```bash
claude mcp add --scope user <name> -- <command>
```

### When to use project scope (`--scope project`)

- Project-specific integrations (e.g. a company's internal API)
- MCP servers that only make sense in one repo context
- When you want teammates to share the same MCP config via `.mcp.json` committed to git

```bash
# Writes to .mcp.json in current directory (safe to commit)
claude mcp add --scope project <name> -- <command>
```

### Verify what's configured where

```bash
claude mcp list
claude mcp get searxng
cat ~/.claude.json | python3 -m json.tool | grep -A 30 mcpServers
```

### Remove an MCP server

```bash
claude mcp remove searxng
```
