# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Build JAR
mvn clean package

# Run in HTTP mode (recommended for development)
DISCORD_TOKEN=xxx SPRING_PROFILES_ACTIVE=http java -jar target/discord-mcp-*.jar

# Run in stdio mode (legacy, for MCP clients that require it)
DISCORD_TOKEN=xxx java -jar target/discord-mcp-*.jar

# Docker (multi-arch amd64/arm64)
docker compose up
```

There are no automated tests or lint tools configured in this project.

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `DISCORD_TOKEN` | Yes | Discord bot token — app fails to start without it |
| `DISCORD_GUILD_ID` | No | Default guild; tools accept per-call `guildId` that falls back to this |
| `SPRING_PROFILES_ACTIVE` | No | Set to `http` for streamable HTTP transport; defaults to stdio |

## Architecture

**Stack**: Spring Boot 4.0.2 + Spring AI 2.0.0-M4 (MCP server) + JDA 6.4.1 (Discord API)

**Transport modes**:
- `http` profile → streamable HTTP on port 8085 at `/mcp` (modern, recommended)
- Default → stdio transport (legacy, MCP protocol over stdin/stdout)

`logback-spring.xml` is profile-aware: stdio mode logs to file only (`target/logs/discord-mcp-server.log`) to avoid stdout collisions with the MCP protocol.

**Tool registration flow**:
1. `DiscordMcpConfig` creates the JDA bean (blocks on `.awaitReady()` until Discord connection is ready)
2. `MethodToolCallbackProvider` scans all 15 service beans and registers every `@Tool`-annotated method as an MCP tool
3. Spring AI MCP auto-exposes registered tools via the configured transport

**Services** (all in `src/main/java/dev/saseq/services/`): Each service is a thin `@Service` wrapper over JDA. There is no DAO layer. Services cover: messages, channels, categories, roles, voice channels, webhooks, threads, moderation, scheduled events, invites, channel permissions, emojis, forums, users, and server info.

## Adding a New MCP Tool

1. Add a public method to the appropriate service in `services/`
2. Annotate with `@Tool(name = "snake_case_name", description = "...")`
3. Annotate parameters with `@ToolParam(description = "...")`; all parameters should be `String`
4. Return a `String` result (or throw to signal an error)
5. The tool is auto-registered — no changes needed in `DiscordMcpConfig`

**Optional guild ID pattern** — every tool that targets a guild accepts a nullable `guildId` and resolves it via:
```java
private String resolveGuildId(String guildId) {
    if ((guildId == null || guildId.isEmpty()) && defaultGuildId != null && !defaultGuildId.isEmpty()) {
        return defaultGuildId;
    }
    return guildId;
}
```
New tools should follow this same pattern and inject `@Value("${discord.guild.id:}") String defaultGuildId`.

## Deployment

CI (`.github/workflows/docker-publish.yml`) builds multi-arch images (amd64 + arm64) and pushes to DockerHub on every push to `main` or on version tags (`v*`). The healthcheck endpoint is `/actuator/health`.
