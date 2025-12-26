## Background: Why Claude via Antigravity is Fundamentally Unstable

Google's Antigravity serves Claude models through a **Gemini-style API format**, forcing all Claude requests through an incompatible translation layer. This architectural decision creates systematic reliability issues that no amount of client-side fixes can fully resolve.

### The Core Problem

Claude's API expects:
- `messages[]` with `content[]` blocks
- `type: "thinking"` for reasoning blocks
- Cryptographically signed thinking blocks per session

Antigravity serves:
- `contents[]` with `parts[]` (Gemini format)
- `thought: true` flag instead of type
- Signatures that cross model family boundaries

**This format mismatch causes cascade failures** when thinking blocks, tool calls, or signatures don't translate correctly.

### Documented Ecosystem Issues

| Project | Issue | Status |
|---------|-------|--------|
| [CLIProxyAPI#714](https://github.com/router-for-me/CLIProxyAPI/issues/714) | `thinking.cache_control` error | Open |
| [CLIProxyAPI#650](https://github.com/router-for-me/CLIProxyAPI/issues/650) | 403 PERMISSION_DENIED | Documented |
| [oh-my-opencode#187](https://github.com/code-yeongyu/oh-my-opencode/issues/187) | Claude Opus thinking failures | Open |
| [gemini-cli#15182](https://github.com/google-gemini/gemini-cli/issues/15182) | Server-side experiment conflicts | Closed (platform) |
| [opencode#6064](https://github.com/sst/opencode/issues/6064) | Thinking block validation | Related |

### Critical Error Patterns

**1. Thinking block validation (most common):**
```
Expected 'thinking' or 'redacted_thinking', but found 'text'
```

**2. Server-side experiment conflicts (Antigravity-specific):**
```
encountered conflicting override on experiment: cascade-knowledge-config
agent executor error: model unreachable: HTTP 400 Bad Request
```

**3. Signature corruption:**
```json
{"message": "Invalid `signature` in `thinking` block"}
```

**4. Tool result orphaning:**
```
tool_use ids were found without tool_result blocks immediately after
```

### Community Sentiment

> "Working with Antigravity was full of frustration" vs Claude Code being "a breeze"

> "The LLM aspect is unusable" —Hacker News

> "Need reliability now? Claude Code with Sonnet 4.5 still offers the most stable experience" —jduncan.io

**Consensus**: Use Antigravity for free experimentation, direct Claude API for production.

---

## The Problem (This Plugin)

After extensive testing and porting code from CLIProxyAPI and LLM-API-Key-Proxy projects, we discovered that **no amount of signature caching, validation, or restoration reliably works** for long multi-turn sessions.

**The issues:**
- OpenCode's layer must comply with its own conversation management
- Long sessions inevitably corrupt thinking blocks (SDK injection, context compaction, cache misses)
- Google's Antigravity architecture makes Claude proxying inherently fragile
- Even CLIProxyAPI has the same issues: [router-for-me/CLIProxyAPI#714](https://github.com/router-for-me/CLIProxyAPI/issues/714)

**Two critical bugs cause permanent session corruption:**

```
tool_use ids were found without tool_result blocks immediately after: tool-call-45
```

```
Expected 'thinking' or 'redacted_thinking', but found 'text'
```

Before: Session permanently broken. User loses entire conversation.

---

## The Solution: Let It Crash

After trying many patterns, the solution is simple: **don't fight corruption, just clean up and start fresh.**

> "If agent crash, let it crash, that is expected, then we clean up and start fresh turn (still keep context but fresh tool and this ensures model quality not degrade as Opus 4.5 stated)"

This aligns with oh-my-opencode's pattern: detect crash → recover → auto-resume.

---

## How It Works

1. **Detect crash** via `session.error` event
2. **Let it crash** - don't try to fix corrupted state
3. **Clean up** - inject synthetic messages to close corrupted turn
4. **Start fresh** - auto-resume with "continue" prompt
5. **Preserve context** - conversation history kept, just fresh thinking/tools

---

## Recovery Types

| Error | Recovery |
|-------|----------|
| `tool_result_missing` (ESC pressed) | Inject synthetic `tool_result`: "Operation cancelled" |
| `thinking_block_order` (corrupted) | Close turn, start fresh |
| `thinking_disabled_violation` | Strip thinking blocks |

---

## New Features

### Configuration File

`~/.config/opencode/antigravity.json`:

```json
{
  "session_recovery": true,
  "auto_resume": true,
  "resume_text": "continue"
}
```

### Known Plugin Interactions

- **DCP**: Must load before antigravity (we fix its synthetic messages)
- **oh-my-opencode**: Parallel subagents may hit rate limits (workaround: add more accounts)

### Documentation Cleanup

Consolidated 4 outdated docs into single `docs/ARCHITECTURE.md`.

---

## Why 1.2.4 Logic + Recovery

v1.2.4 is the most stable implementation from all prototype efforts. This PR adds the recovery layer on top - same core logic, but now sessions can survive crashes instead of dying permanently.

---

## Stability Assessment

| Dimension | Rating | Notes |
|-----------|--------|-------|
| **Reliability** | ⭐⭐☆☆☆ | HTTP 400 errors, server-side conflicts |
| **Error handling** | ⭐⭐⭐⭐☆ | Now recoverable with this PR |
| **Rate limits** | ⭐⭐☆☆☆ | Aggressive throttling, multi-account helps |
| **Documentation** | ⭐⭐⭐⭐☆ | Consolidated, accurate |

**Bottom line**: Antigravity Claude support is a "brilliant, messy prototype." This PR makes it survivable for real work by accepting crashes as expected and recovering gracefully.
