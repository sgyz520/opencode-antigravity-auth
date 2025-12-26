# Antigravity Claude Support: Instability Research

**Last Updated:** December 2025

This document explains why Claude models through Google Antigravity are fundamentally unstable and why this plugin adopts a "let it crash" recovery strategy.

---

## Executive Summary

Google's Antigravity serves Claude models through a **Gemini-style API format**, forcing all Claude requests through an incompatible translation layer. This architectural decision creates systematic reliability issues documented across 15+ GitHub issues, multiple proxy projects, and developer forums.

**Rating:** ⭐⭐☆☆☆ for production reliability

**Recommendation:** Use Antigravity for free experimentation, direct Claude API for production.

---

## The Architectural Problem

### What Antigravity Does

Antigravity is Google's AI-powered IDE (launched November 2025, built from the $2.4B Windsurf acquisition). It provides access to:
- Gemini 3 Pro (native)
- Claude Sonnet 4.5, Opus 4.5 (proxied)
- Their "thinking" variants

### The Format Mismatch

| Aspect | Claude Native API | Antigravity Gateway |
|--------|-------------------|---------------------|
| Message format | `messages[]` with `content[]` | `contents[]` with `parts[]` |
| Thinking blocks | `type: "thinking"` | `thought: true` flag |
| Role names | `assistant` | `model` |
| Signatures | Per-session, per-model | Cross-model boundaries |
| System instruction | String or object | Must be `{ parts: [{ text }] }` |

**Every request and response must be translated**, creating opportunities for corruption at each step.

### Why This Causes Failures

1. **Thinking blocks are cryptographically signed** per model family
2. **Signatures cannot be modified, truncated, or reordered** after generation
3. **Translation layers inevitably corrupt signatures** through:
   - SDK injection (`cache_control`, `providerOptions`)
   - Context compaction (DCP creates synthetic messages)
   - Format conversion (role mapping, content structure)
   - Session boundaries (cache misses, restarts)

---

## Documented Issues Across Ecosystem

### Verified Open Issues

| Project | Issue | Description |
|---------|-------|-------------|
| [CLIProxyAPI#714](https://github.com/router-for-me/CLIProxyAPI/issues/714) | `thinking.cache_control` error | Dec 25, 2025 |
| [oh-my-opencode#187](https://github.com/code-yeongyu/oh-my-opencode/issues/187) | Claude Opus thinking failures | Dec 24, 2025 |

### Related Issues

| Project | Issue | Description |
|---------|-------|-------------|
| [gemini-cli#15182](https://github.com/google-gemini/gemini-cli/issues/15182) | Server-side experiment conflicts | Blocks ALL Claude |
| [CLIProxyAPI#650](https://github.com/router-for-me/CLIProxyAPI/issues/650) | 403 PERMISSION_DENIED | Auth routing |
| [opencode#6064](https://github.com/sst/opencode/issues/6064) | Thinking block validation | Session corruption |
| [claude-code#1183](https://github.com/anthropics/claude-code/issues/1183) | Thinking validation failures | Related pattern |
| [claude-code#11865](https://github.com/anthropics/claude-code/issues/11865) | Session recovery | Related pattern |
| [vercel/ai#7729](https://github.com/vercel/ai/issues/7729) | Thinking block handling | SDK integration |

---

## Critical Error Patterns

### 1. Thinking Block Validation (Most Common)

When extended thinking is enabled, Claude requires assistant messages to begin with `thinking` or `redacted_thinking` blocks. Proxies frequently corrupt this structure:

```json
{
  "error": {
    "type": "invalid_request_error",
    "message": "messages.3.content.0.type: Expected 'thinking' or 'redacted_thinking', but found 'text'"
  }
}
```

**Cause:** Translation layers strip, reorder, or modify thinking blocks.

**Impact:** Session unrecoverable without restart.

### 2. Server-Side Experiment Conflicts (Antigravity-Specific)

The most severe issue affects ALL Claude variants through Antigravity:

```
encountered conflicting override on experiment: cascade-knowledge-config
agent executor error: model unreachable: HTTP 400 Bad Request
```

**Cause:** Google's internal experiment flags conflict with Claude routing.

**Impact:** Complete Claude unavailability while Gemini works fine.

**Status:** Closed with "area/platform" label - infrastructure issue, no public fix.

### 3. Signature Validation Corruption

```json
{"message": "Invalid `signature` in `thinking` block"}
```

**Causes:**
- Cross-model conversations (Gemini → Claude)
- Stale signature caches after long sessions
- SDK modifications to thinking blocks
- Provider switching mid-conversation

### 4. Tool Result Orphaning

```
tool_use ids were found without tool_result blocks immediately after: tool-call-45
```

**Cause:** User interruption (ESC), timeout, or crash during tool execution.

**Impact:** Session permanently broken - Claude requires every `tool_use` to have a `tool_result`.

### 5. Cache Control Positioning Failures

Claude's extended thinking has strict requirements:
- `cache_control` must be on the **last content block** only
- Changes to thinking parameters invalidate all cached content
- Thinking blocks themselves cannot receive `cache_control` markers

**Cause:** SDKs inject `cache_control` into thinking blocks.

**Impact:** Silent cache invalidation or explicit 400 errors.

---

## Community Sentiment

### Negative Experiences (Common)

> "Working with Antigravity was full of frustration" vs Claude Code being "a breeze"  
> —Quesma Blog

> "Frequent mid-task agent failures, crashes, required constant restarts"  
> —DEV.to

> "The LLM aspect is unusable"  
> —Hacker News

> "Struggles with complex tasks, requires multiple iterations"  
> —SmartScope

> "I hit the rate limit when only doing two or three prompts" despite paying for Google AI Pro  
> —How-To Geek

### Reliability Recommendations

> "Need reliability now? Claude Code with Sonnet 4.5 still offers the most stable experience without capacity issues."  
> —jduncan.io

> "Claude's CLI is clean, but Antigravity's UI is overwhelming... Claude is stable due to rate limits."  
> —SmartScope developer survey

### Ecosystem Consensus

**Use Antigravity for:** Free experimentation during preview period

**Use direct Claude API for:** Production, mission-critical work

---

## Proxy Ecosystem

| Project | Stars | Claude Support | Notes |
|---------|-------|----------------|-------|
| **router-for-me/CLIProxyAPI** | ~3,500 | Full | Model mapping fallbacks |
| **Mirrowel/LLM-API-Key-Proxy** | ~80 | Opus, Sonnet | 57 releases |
| **NoeFabris/opencode-antigravity-auth** | 10 | OAuth plugin | This project |
| **nghyane/llm-mux** | — | Subscription conversion | |
| **znlsl/Antigravity2Api** | — | API translation | |

All projects implement workarounds for the same fundamental issues.

---

## Known Limitations

1. **Rate limits tracked separately** per model family—Claude exhaustion doesn't affect Gemini
2. **No dual-pool fallback** for Claude (unlike Gemini which can fail over)
3. **Tool calling requires VALIDATED mode** for Claude compatibility
4. **Thinking blocks consume context faster** than standard messages
5. **Multi-turn tool loops** break signature chains
6. **opencode-skills plugin incompatible** with Claude thinking models

---

## Our Solution: Let It Crash

Given the fundamental instability, we adopt a pragmatic approach:

### Philosophy

> "If agent crash, let it crash, that is expected, then we clean up and start fresh turn (still keep context but fresh tool and this ensures model quality not degrade as Opus 4.5 stated)"

### Implementation

1. **Don't fight corruption** - Complex signature restoration has too many edge cases
2. **Detect crashes** via `session.error` event
3. **Clean up** - Inject synthetic messages to close corrupted turns
4. **Start fresh** - Auto-resume with configurable prompt
5. **Preserve context** - Conversation history kept, just fresh thinking/tools

### Why This Works

- **Same approach as CLIProxyAPI** - Stateless proxy, no signature caching
- **Matches oh-my-opencode pattern** - Detect → recover → resume
- **Model quality preserved** - Fresh thinking each turn (Opus 4.5 recommended)
- **100% recoverable** - No more permanent session corruption

---

## Stability Assessment

| Dimension | Rating | Evidence |
|-----------|--------|----------|
| **Reliability** | ⭐⭐☆☆☆ | HTTP 400 errors, server-side conflicts |
| **Error handling** | ⭐⭐☆☆☆ | Unrecoverable sessions (before this fix) |
| **Rate limits** | ⭐⭐☆☆☆ | Aggressive throttling, quick exhaustion |
| **Documentation** | ⭐⭐⭐☆☆ | Plugin READMEs document workarounds |
| **Community support** | ⭐⭐⭐☆☆ | Active proxy development, quick patches |

**Overall:** Antigravity Claude support is a "brilliant, messy prototype"—useful for free access during preview, not production-ready.

---

## References

- [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) - Most popular Antigravity proxy
- [LLM-API-Key-Proxy](https://github.com/Mirrowel/LLM-API-Key-Proxy) - Alternative proxy
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - Session recovery patterns
- [Antigravity API Spec](./ANTIGRAVITY_API_SPEC.md) - API reference
