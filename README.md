# Velixo plugins for Claude

Connects Claude to your ERP through the Velixo MCP server, with Velixo's finance skills on top.

> Generated from [Velixo-Intelligence-Plugins](https://github.com/VelixoSolutions/Velixo-Intelligence-Plugins). Don't edit files here; changes are overwritten on the next publish.

## Add the marketplace

```
/plugin marketplace add VelixoSolutions/Velixo-Claude-Plugin
```

## Plugins

| Plugin | Environment | MCP server | Install |
|---|---|---|---|
| `velixo` | Velixo | `https://app.velixo.com/mcp` | /plugin install velixo@velixo-plugins |
| `velixo-nx-dev` | Velixo nx-dev | `https://nx-dev.velixo.com/mcp` | /plugin install velixo-nx-dev@velixo-plugins |
| `velixo-nx-staging` | Velixo nx-staging | `https://nx-staging.velixo.com/mcp` | /plugin install velixo-nx-staging@velixo-plugins |
| `velixo-nx-stage-zero` | Velixo nx-stage-zero | `https://nx-stage-zero.velixo.com/mcp` | /plugin install velixo-nx-stage-zero@velixo-plugins |

Use the first one (`velixo`) unless you've been asked to test against a Velixo pre-release environment.
Authentication is OAuth: you're prompted to sign in on first use.

Version 2026.11.28.
