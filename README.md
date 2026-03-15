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
| **Round-trips for "who calls X and what tests cover it?"** | 5-8 calls (hover + definition + references + call hierarchy + symbol search) | 1 call (`impact`) |
| **Data per response** | Cursor-position hover text, raw JSON spans | Full signatures, code context, grouped by file |
| **Path format** | Absolute URIs (`file:///home/user/project/src/...`) | Relative paths (`src/Services/MyService.cs`) |
| **Output format** | Verbose JSON with TextEdit ranges, Position objects | Compact, AI-native (2 format profiles) |
| **Capabilities** | Varies by server, no standard capability tiers | Declared tiers (1/2/3) per plugin |
| **Composite queries** | Not supported — client must orchestrate | Built-in (`impact` = usages + callers + tests) |
| **Multi-language** | One server per language, manual setup | Plugin architecture, one endpoint, 5 languages |

---

## Measured Token Savings — 93% vs Grep

Real measurements from the Zerox.Lsai reference implementation (v1.0.60) across **5 languages** and **25+ live MCP queries**.

### LSAI Compact vs Grep

| Metric | Savings |
|--------|:-------:|
| **Overall** | **~93%** |
| Search | 25-67% |
| Usages | 29-74% |
| Outline | 43-79% |
| Impact | 61% |
| Callers | 8-73% |

### Per-Tool Highlights

| Query | Before (old format) | After (Compact v1.3) | Savings |
|-------|:-------------------:|:--------------------:|:-------:|
| C# outline (51 members) | 641 tokens | 132 tokens | **79%** |
| Python usages (9 refs) | 50 tokens | 13 tokens | **74%** |
| C# impact analysis | 375 tokens | 148 tokens | **61%** |
| Rename (2 files) | 547 tokens | ~20 tokens | **96%** |

### LSAI vs Raw LSP JSON

Even vs semantically correct Raw LSP JSON (not dumb text grep), LSAI's compact format saves:

| Language | Search Savings | Usages Savings |
|----------|:--------------:|:--------------:|
| **Python** | **82.8%** | **85.5%** |
| **TypeScript** | **85.0%** | **82.2%** |
| **Java** | **40.1%** | **31.9%** |

---

## Tools

LSAI defines 12 semantic tools organized in 3 capability tiers, plus 4 workspace/meta tools. All tools operate through MCP (Model Context Protocol) with the `lsai_` prefix.

### Semantic Tools (Protocol-defined)

| Tool | Tier | Description | Replaces (LSP equivalent) |
|------|------|-------------|--------------------------|
| `search` | 1 | Find symbols by name pattern with kind/scope filters | `workspace/symbol` |
| `info` | 1/2/3 | Full symbol details: signature, modifiers, base types, members | `hover` + `definition` + `typeDefinition` |
| `usages` | 1 | Semantic references grouped by file with code context | `references` |
| `rename` | 1 | Safe rename with preview, cross-file, compilation check | `rename` |
| `diagnostics` | 1 | Compiler errors/warnings, filtered (no `obj/` noise) | `diagnostics` |
| `outline` | 1 | Document structure with **full method signatures** | `documentSymbol` (but richer) |
| `deps` | 1/2 | Project dependency graph | No LSP equivalent |
| `callers` | 2 | Who calls this method? Call graph upward | `callHierarchy/incomingCalls` |
| `callees` | 2 | What does this method call? Call graph downward | `callHierarchy/outgoingCalls` |
| `hierarchy` | 2 | Inheritance tree: base types, interfaces, derived types | `typeHierarchy` |
| `impact` | 3 | Composite: usages + transitive callers + affected tests + risk | No LSP equivalent |
| `source` | 1 | Get symbol source code — method body, class definition | `definition` + file read (2 calls) |

### Workspace & Meta Tools

| Tool | Description |
|------|-------------|
| `workspace_open` | Open a project/solution for analysis, returns workspace ID |
| `workspace_list` | List all open workspaces with IDs, paths, languages |
| `workspace_close` | Close an open workspace and free resources |
| `server` | Capability discovery: version, plugins, open workspaces |

---

## Output Format Profiles

LSAI is format-agnostic — the protocol defines data contracts, not serialization. Two profiles cover all real use cases:

| Profile | Style | Use Case |
|---------|-------|----------|
| **Compact** | Minimal tokens: no brackets, no footers, comma-separated, collapsed properties | Default. Lowest token cost for AI agents |
| **Verbose** | Compact structure with code context snippets and full signatures | When the AI needs to understand HOW symbols are used |

v1.2 defined 6 format profiles. Empirical analysis on 223 queries proved 4 were redundant — producing identical or near-identical output. The optimization from 6→2 formats plus redundancy elimination yielded **37% fewer tokens** on the same queries.

---

## Capability Tiers

Plugins declare their tier. The AI adapts to what's available.

| Tier | What's included | Minimum backend requirement |
|------|----------------|---------------------------|
| **Tier 1** | search, info (basic), usages, rename, diagnostics, outline, deps, source | Any language server or LSP bridge |
| **Tier 2** | + callers, callees, hierarchy, info (extended), deps (with files) | LSP 3.17+ with call hierarchy support |
| **Tier 3** | + impact (full analysis), info (complete), cross-project queries | Native compiler API (Roslyn, rustc, tsc) |

A Tier 1 plugin is useful. A Tier 3 plugin is transformative — `impact` alone replaces 5-8 manual tool calls.

---

## Multi-Language Support

LSAI uses a plugin architecture to support multiple languages through a single endpoint.

| Language | Engine | Plugin Type | Tier | Status |
|----------|--------|-------------|:----:|--------|
| **C#** | Roslyn | Native compiler API | 3 | Production |
| **Python** | Pyright | LSP bridge | 2 | Production |
| **TypeScript** | tsserver | LSP bridge | 2 | Production |
| **JavaScript** | tsserver | LSP bridge | 2 | Production |
| **Java** | Eclipse JDT (jdtls) | LSP bridge | 2 | Production |

### Known LSP Server Limitations

| Language | Limitation | Impact |
|----------|-----------|--------|
| JavaScript | `workspace/symbol` not supported for CommonJS `.js` files | Search limited to outline-indexed symbols |
| JavaScript | `<unknown>` symbol names for anonymous CommonJS exports | Outline shows `<unknown>` for `module.exports = ...` |
| Python | `textDocument/prepareTypeHierarchy` not implemented by Pyright | Hierarchy tool unavailable for Python |
| Java | `workspace/symbol` does not expose fields/record components | Search returns 0 for fields |
| Java | Fuzzy matching on `workspace/symbol` | May return extra results |

---

## Implementations

### VS-MCP (Production Reference)

| Property | Value |
|----------|-------|
| **Project** | [VS-MCP](https://github.com/LadislavSopko/zerox-msvc-info) — Visual Studio MCP Server |
| **Tier** | 3 (full semantic analysis) |
| **Language** | C# (via Visual Studio Roslyn integration) |
| **Tools** | 20 tools (superset of LSAI — includes file navigation, formatting) |
| **Production usage** | 6+ months, Claude Opus 3.5/4.6, daily development use |

### Zerox.Lsai Reference Server

| Property | Value |
|----------|-------|
| **Project** | [Zerox.Lsai](https://github.com/LadislavSopko/Zerox.Lsai) — standalone LSAI server |
| **Tier** | 3 (C#) / 2 (Python, TypeScript, JavaScript, Java) |
| **Languages** | 5: C# (Roslyn), Python (Pyright), TypeScript, JavaScript (tsserver), Java (jdtls) |
| **Transport** | HTTP (Streamable HTTP via MCP SDK) |
| **Tools** | 20 MCP tools (12 semantic + 3 workspace + 1 meta + 4 composite) |
| **Tests** | 413+ unit tests + 16 E2E integration tests |
| **Output formats** | 2 profiles (Compact, Verbose) |
| **Deployment** | Docker (4 combo images: web 877MB, dotnet 1.82GB, jvm 1.44GB, full 2.38GB) |

---

## Documentation

| Document | Audience | Description |
|----------|----------|-------------|
| [`spec/LSAI-v1.3.md`](spec/LSAI-v1.3.md) | Implementers | Full protocol specification — 12 semantic tools, data contracts, tiers |
| [`USAGE-GUIDE.md`](USAGE-GUIDE.md) | AI agents | How to use LSAI tools effectively — workflows, token tips, tool selection |

---

## Contributing

LSAI is an open protocol. Implementations in any language are welcome.

To implement an LSAI-compliant server:
1. Read the [spec](spec/LSAI-v1.3.md)
2. Implement Tier 1 tools (8 tools) as MCP tools with the `lsai_` prefix
3. Declare your plugin's tier via the `server` meta-tool
4. Use relative paths, provide full signatures in outlines, follow Compact format rules

---

## License

Protocol specification licensed under **CC BY-NC 4.0**. Free for study, research, and non-commercial use. Commercial implementations require a separate license — contact ladislav.sopko@gmail.com

---

Protocol designed by Ladislav Sopko, 0ics srl, Bologna, Italy — March 2026
