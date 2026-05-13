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

Real measurements on **production codebases**: C# (366 source files) and JavaScript/Express (97 source files). Not toy fixtures.

### Comparable Tools — LSAI vs grep

These tools have a grep equivalent. LSAI returns the same information in fewer tokens:

| Tool | Avg Savings | What LSAI Does Better |
|------|:-----------:|----------------------|
| **search** | **87%** | Returns structured `file:line kind name namespace` — grep returns full lines with path noise |
| **info** | **90%** | Returns signature + docs + type in one call — grep needs `grep -A5` and still misses metadata |
| **usages** | **79%** | Semantic references only — grep includes comments, strings, partial matches |
| **source** | **72%** | Extracts exact method body — grep -A30 includes surrounding code |
| **deps** | **94%** | Parsed imports per file — grep returns raw lines from all files |
| **outline** | **~0%** | LSAI returns MORE data (full signatures, types) — richer, not just compact |

### Unique Tools — grep CANNOT do this

These tools provide capabilities that have **no grep equivalent**:

| Tool | Avg Response | What It Does |
|------|:------------:|-------------|
| **callers** | 167 tokens | Semantic call graph upward: `Type.Method file:line` |
| **callees** | 12 tokens | Semantic call graph downward |
| **hierarchy** | 12 tokens | Inheritance tree: base types, interfaces, derived types |
| **impact** | 238 tokens | Change risk: usages + callers + affected tests |
| **file_refs** | 666 tokens | Cross-file reference map with symbol names |
| **context** | composite | outline + diagnostics + usages + callers + risk in one call |

To do what `callers` does via grep, you would need to: grep for the symbol name (gets false positives from comments/strings), then for each match read the file to determine if it's a call site (not a definition, import, or comment), then find the containing method name. That's 3-7 tool calls per match, and still less accurate.

### Overall

**Comparable tools: 72-94% token savings** on real projects (300+ files).

**Unique tools: enable semantic operations that are impossible with text search.**

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
