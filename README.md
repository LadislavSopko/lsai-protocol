# LSAI Protocol — Language Server for AI

**LSAI is an open protocol that gives AI coding agents access to compiler-level semantic intelligence.**
Instead of guessing code structure through regex and text search, AI agents get type-resolved symbols, call graphs, impact analysis, and semantic rename — the same data a compiler has.

---

## The Problem

AI coding tools today are **semantically blind**. They operate on text, not on code structure.

When an AI agent needs to understand a codebase, it does glorified `grep`: text search, regex matching, file reading. It has no idea what a symbol's type is, who calls a method, what implements an interface, or what breaks if you rename something.

This leads to:
- **Wasted tokens** — reading entire files to find a return type that the compiler already knows
- **Wrong answers** — confusing a `Process` in a comment with the actual `Process` method
- **Slow iteration** — 5-10 round trips to gather what one semantic query could return
- **Missed context** — no way to discover transitive callers, implementations, or test coverage

**The compiler already knows all of this.** LSAI makes that knowledge available to AI agents.

---

## Why Not Just Use LSP?

LSP was designed for IDEs — cursor-centric, single-file, chatty. AI agents need something different.

| Dimension | LSP (via MCP bridge) | LSAI |
|-----------|---------------------|------|
| **Round-trips for "who calls X and what tests cover it?"** | 5-8 calls | 1 call (`impact`) |
| **Data per response** | Cursor-position hover text, raw JSON spans | Full signatures, code context, grouped by file |
| **Path format** | Absolute URIs (`file:///home/user/project/src/...`) | Relative paths (`src/Services/MyService.cs`) |
| **Output format** | Verbose JSON with TextEdit ranges | Compact, AI-native (2 format profiles) |
| **Composite queries** | Not supported — client must orchestrate | Built-in (`impact`, `context`, `file_refs`) |
| **Multi-language** | One server per language, manual setup | Plugin architecture, one endpoint, 10 languages |
| **Fallback resilience** | Fails if server lacks capability | Graceful degradation with alternative strategies |

---

## Measured Token Savings — v1.0.176 Benchmark

Real measurements from the Zerox.Lsai reference implementation across **8 languages**, **12 tools**, and **96 live MCP queries**.

### Summary: LSAI vs grep by Language

| Language | Avg Savings | Best Tool | Notes |
|----------|:-----------:|-----------|-------|
| **Python** | 85% | hierarchy (100%), search (97%) | All tools positive |
| **TypeScript** | 78% | hierarchy (100%), deps (100%) | source -18% on tiny methods |
| **C#** | 75% | hierarchy (100%), search (98%) | Large project, biggest absolute savings |
| **Rust** | 76% | callees (100%), usages (90%) | — |
| **PHP** | 61% | hierarchy (99%), deps (99%) | source -36% on 1-line methods |
| **JavaScript** | 66% | hierarchy (100%), deps (100%) | Express fork, large codebase |
| **Go** | 55% | callees (88%), callers (87%) | Small fixture, overhead visible |
| **Java** | 48% | callers (100%), callees (100%) | Small fixture, jdtls metadata overhead |

### Per-Tool Savings Range

| Tool | Min | Max | Where LSAI Wins Big |
|------|:---:|:---:|---------------------|
| **hierarchy** | 9% | 100% | Always — grep can't trace inheritance |
| **search** | -6% | 98% | Large projects: 73-98% savings |
| **callees** | 27% | 100% | Call graph impossible with grep |
| **callers** | 19% | 100% | Call graph impossible with grep |
| **usages** | 19% | 90% | Semantic (no comment/string false positives) |
| **impact** | 54% | 88% | Composite: 5-8 grep calls in one |
| **info** | -12% | 91% | Overhead on tiny files |
| **deps** | 30% | 100% | Import parsing vs grep noise |
| **source** | -47% | 88% | Negative on tiny methods (metadata overhead) |
| **outline** | 26% | 68% | LSAI returns MORE data (richer) |
| **file_refs** | 46% | 99% | Cross-file map impossible with grep |

### Overall

**Average savings: 66%** across 64 valid measurements (excluding N/A and negative outliers on tiny fixtures).

On real-world projects (>100 files): **75-93% savings** — the more code, the bigger the win.

---

## Tools

LSAI defines **14 semantic tools** plus 4 workspace/meta tools. All tools operate through MCP with the `lsai_` prefix.

### Semantic Tools

| Tool | Description | Replaces (LSP/grep equivalent) |
|------|-------------|-------------------------------|
| `search` | Find symbols by name across the workspace | `workspace/symbol` / `grep -rn` |
| `info` | Symbol details: signature, docs, type, modifiers | `hover` + `definition` + `typeDefinition` |
| `usages` | Semantic references grouped by file | `references` / `grep -rn` |
| `outline` | Document structure with full method signatures | `documentSymbol` (but richer) |
| `source` | Get symbol implementation source code | `definition` + file read (2 calls) |
| `callers` | Call graph upward: who calls this method? | `callHierarchy/incomingCalls` |
| `callees` | Call graph downward: what does this method call? | `callHierarchy/outgoingCalls` |
| `hierarchy` | Type inheritance: base types, interfaces, derived | `typeHierarchy` |
| `impact` | Composite: usages + callers + affected tests + risk | No equivalent (5-8 calls) |
| `deps` | File-level imports/includes/dependencies | No equivalent |
| `file_refs` | Cross-file reference map | No equivalent |
| `context` | Composite: outline + diagnostics + usages + callers + risk | No equivalent (7+ calls) |
| `diagnostics` | Compiler errors/warnings | `diagnostics` |
| `rename` | Safe semantic rename across workspace | `rename` |

### Workspace & Meta Tools

| Tool | Description |
|------|-------------|
| `workspace_open` | Open a project for analysis (usually auto-opened from cwd) |
| `workspace_list` | List open workspaces with IDs, paths, languages |
| `workspace_close` | Close a workspace and free resources |
| `server` | Capability discovery: version, plugins, workspaces |

---

## Language Support

| Language | Engine | All 14 Tools | Fallbacks |
|----------|--------|:------------:|-----------|
| **C#** | Roslyn (native) | Yes | — |
| **Python** | ty (Astral) | Yes | CallersFallback via references |
| **TypeScript** | typescript-language-server | Yes | — |
| **JavaScript** | typescript-language-server | Yes | — |
| **Java** | Eclipse JDT (jdtls) | Yes | — |
| **PHP** | intelephense | Yes | CallersFallback; rename N/A (free tier) |
| **Rust** | rust-analyzer | Yes | — |
| **Go** | gopls | Yes | — |
| **C** | clangd | Yes | CalleesFallback via regex+symbol |
| **C++** | clangd | Yes | CalleesFallback via regex+symbol |

All 14 tools work on all 10 languages. Where an upstream LSP lacks a capability, LSAI provides fallback strategies that still return useful data.

---

## Output Format Profiles

| Profile | Style | Use Case |
|---------|-------|----------|
| **Compact** | Minimal tokens: no brackets, no footers, comma-separated, collapsed properties | Default. Lowest token cost |
| **Verbose** | Compact + code context snippets | When the AI needs to understand HOW symbols are used |

---

## Documentation

| Document | Audience | Description |
|----------|----------|-------------|
| [`spec/LSAI-v1.4.md`](spec/LSAI-v1.4.md) | Implementers | Full protocol specification — 14 tools, data contracts, fallbacks |
| [`spec/LSAI-v1.3.md`](spec/LSAI-v1.3.md) | Reference | Previous version (12 tools, 5 languages) |
| [`USAGE-GUIDE.md`](USAGE-GUIDE.md) | AI agents | How to use LSAI tools effectively |

---

## Reference Implementation

| Property | Value |
|----------|-------|
| **Project** | [Zerox.Lsai](https://github.com/0ics-srls/Zerox.Lsai.Public) |
| **Version** | v1.0.176 (May 2026) |
| **Languages** | 10: C#, Python, TypeScript, JavaScript, Java, PHP, Rust, Go, C, C++ |
| **Transport** | stdio (MCP) |
| **Tools** | 14 semantic + 4 workspace/meta |
| **Install** | `curl -fsSL .../install.sh \| bash` (one command, auto-detects toolchains) |
| **Platforms** | Linux, macOS, WSL, Windows |
| **Validated** | 8 langs x 12 tools E2E matrix (Linux + Windows VM) |

---

## Contributing

LSAI is an open protocol. Implementations in any language are welcome.

To implement an LSAI-compliant server:
1. Read the [spec](spec/LSAI-v1.4.md)
2. Implement Tier 1 tools (9 tools) as MCP tools with the `lsai_` prefix
3. Declare your plugin's capabilities via the `server` meta-tool
4. Use relative paths, provide full signatures in outlines, follow Compact format rules
5. Implement fallback strategies for Tier 2 tools where the underlying LSP lacks support

---

## License

Protocol specification licensed under **CC BY-NC 4.0**. Free for study, research, and non-commercial use. Commercial implementations require a separate license — contact ladislav.sopko@gmail.com

---

Protocol designed by Ladislav Sopko, 0ics srl, Bologna, Italy — May 2026
