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
| **Output format** | Verbose JSON with TextEdit ranges, Position objects | Compact, AI-native (6 format profiles) |
| **Capabilities** | Varies by server, no standard capability tiers | Declared tiers (1/2/3) per plugin |
| **Composite queries** | Not supported — client must orchestrate | Built-in (`impact` = usages + callers + tests) |
| **Multi-language** | One server per language, manual setup | Plugin architecture, one endpoint, 5 languages |

---

## Measured Token Savings — 5 Languages, 46 Symbols

Real measurements from the Zerox.Lsai reference implementation across **5 languages**, **46 symbols**, and **2 core operations** (Search + Usages). Full data in [RESULTS.md](https://github.com/LadislavSopko/Zerox.Lsai/blob/master/RESULTS.md).

### LSAI vs Grep (text search)

| Language | Engine | Symbols | Search Savings | Usages Savings |
|----------|--------|:-------:|:--------------:|:--------------:|
| **C#** | Roslyn (native) | 10 | **92.9%** | **92.9%** |
| **Python** | Pyright (LSP) | 10 | **63.5%** | **78.6%** |
| **TypeScript** | tsserver (LSP) | 8 | **91.6%** | **85.5%** |
| **JavaScript** | tsserver (LSP) | 10 | **99.3%** | **98.3%** |
| **Java** | jdtls (LSP) | 8 | **55.8%** | **68.6%** |
| **Weighted Average** | | **46** | **81.2%** | **86.9%** |

### LSAI vs Raw LSP JSON (same semantic data, different format)

Even vs semantically correct Raw LSP JSON (not dumb text grep), LSAI's compact formats save significant tokens:

| Language | Engine | Search Savings | Usages Savings |
|----------|--------|:--------------:|:--------------:|
| **Python** | Pyright | **82.8%** | **85.5%** |
| **TypeScript** | tsserver | **85.0%** | **82.2%** |
| **Java** | jdtls | **40.1%** | **31.9%** |
| **Weighted Avg** | | **69.1%** | **75.0%** |

**Key insight**: The JSON skeleton alone (`{"name":"X","kind":5,"location":{"uri":"file:///...","range":{"start":{"line":N,"character":C},...}}}`) costs ~220 chars per symbol. LSAI's `file:line:col: signature` format eliminates this overhead entirely.

---

## Tools

LSAI defines 11 semantic tools organized in 3 capability tiers, plus 4 workspace/meta tools. All tools operate through MCP (Model Context Protocol) with the `lsai_` prefix.

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

### Workspace & Meta Tools

| Tool | Description |
|------|-------------|
| `workspace_open` | Open a project/solution for analysis, returns workspace ID |
| `workspace_list` | List all open workspaces with IDs, paths, languages |
| `workspace_close` | Close an open workspace and free resources |
| `server` | Capability discovery: version, plugins, open workspaces |

### Tools With No Grep Equivalent

6 out of 11 semantic tools provide capabilities that text search **cannot replicate**:

| Tool | What It Does | Grep Alternative? |
|------|-------------|-------------------|
| **Info** | Full symbol signature, documentation, accessibility | None |
| **Callers** | Semantic call graph — who calls this method | Grep `methodName` includes definitions, comments |
| **Callees** | What methods does this method call | None |
| **Hierarchy** | Base types, interfaces, derived types | `class X(Y)` regex — misses interfaces, derived |
| **Impact** | Usages + callers + affected tests in one call | 3+ Grep queries, still misses callers |
| **Deps** | Project dependencies and import graph | `<PackageReference` regex — XML noise |

---

## Output Format Profiles

LSAI is format-agnostic — the protocol defines data contracts, not serialization. The reference implementation supports 6 profiles:

| Profile | Style | Use Case |
|---------|-------|----------|
| **CompactText** | Minimal text, no context snippets | Default. Lowest token cost |
| **CompactTextVerbose** | Compact text with source line context | When code context needed |
| **TurboCompact** | Ultra-compressed: names, positions, counts only | Maximum token savings |
| **GrepOutput** | `file:line:col: match` (grep-like) | Familiar format, used in benchmarks |
| **CompilerOutput** | `File.cs(line,col): error CS1234: msg` | Compiler-native output |
| **LanguageSyntax** | Language-specific syntax structure | Language-aware display |

LLMs decode any compact format with zero friction. Empirical testing across Claude Opus 3.5/4.6 and OpenAI Codex confirms: pipe-delimited, colon-separated, JSON — all work equally well without legends.

---

## Capability Tiers

Plugins declare their tier. The AI adapts to what's available.

| Tier | What's included | Minimum backend requirement |
|------|----------------|---------------------------|
| **Tier 1** | search, info (basic), usages, rename, diagnostics, outline, deps | Any language server or LSP bridge |
| **Tier 2** | + callers, callees, hierarchy, info (extended), deps (with files) | LSP 3.17+ with call hierarchy support |
| **Tier 3** | + impact (full analysis), info (complete), cross-project queries | Native compiler API (Roslyn, rustc, tsc) |

A Tier 1 plugin is useful. A Tier 3 plugin is transformative — `impact` alone replaces 5-8 manual tool calls.

---

## Protocol Design Principles

1. **Data Richness Over Format** — An outline with full method signatures eliminates follow-up file reads. Usages with code context show HOW a symbol is used, not just WHERE.

2. **Composite Queries** — One tool call returns what would require 3-7 LSP calls. `info` = hover + definition + type info + modifiers. `impact` = usages + callers + test coverage.

3. **Only What AI Can't Do** — No file read/write, no build, no test execution. Only compiler-semantic operations that require a type system. Everything else the AI already has tools for.

4. **Relative Paths Always** — Absolute paths waste ~80 characters per line. On a response with 20 lines, that's ~400 tokens wasted on path prefixes alone.

5. **Format Agnostic** — The protocol defines data contracts (what each tool returns). Serialization format is an implementation choice. 6 profiles available in the reference implementation.

6. **Capability Declaration** — Each plugin declares its tier. The AI knows what's available without trial and error.

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

These are upstream LSP server limitations, not LSAI bugs. LSAI passes through whatever the language server provides.

---

## Implementations

### VS-MCP (Production Reference)

| Property | Value |
|----------|-------|
| **Project** | [VS-MCP](https://github.com/LadislavSopko/zerox-msvc-info) — Visual Studio MCP Server |
| **Tier** | 3 (full semantic analysis) |
| **Language** | C# (via Visual Studio Roslyn integration) |
| **Tools** | 20 tools (superset of LSAI — includes file navigation, formatting) |
| **Response time** | 2-5ms typical |
| **Production usage** | 6+ months, Claude Opus 3.5/4.6, daily development use |
| **Key insight** | Battle-tested proof that compact AI-native formats work with zero LLM friction |

VS-MCP is the production system that validated LSAI's core thesis: compiler semantic data in a compact format dramatically improves AI coding agent performance. The LSAI protocol formalizes what VS-MCP proved empirically.

### Zerox.Lsai Reference Server

| Property | Value |
|----------|-------|
| **Project** | [Zerox.Lsai](https://github.com/LadislavSopko/Zerox.Lsai) — standalone LSAI server |
| **Tier** | 3 (C#) / 2 (Python, TypeScript, JavaScript, Java) |
| **Languages** | 5: C# (Roslyn), Python (Pyright), TypeScript, JavaScript (tsserver), Java (jdtls) |
| **Transport** | HTTP (Streamable HTTP via MCP SDK) |
| **Tools** | 15 MCP tools (11 semantic + 3 workspace + 1 meta) |
| **Tests** | 465 unit tests + 15 E2E integration tests |
| **Output formats** | 6 profiles (CompactText, CompactTextVerbose, TurboCompact, GrepOutput, CompilerOutput, LanguageSyntax) |
| **Deployment** | Docker (1.4 GB image, all 5 languages included) or self-hosted |

---

## Prior Art

Every existing tool has at least one critical gap that LSAI addresses.

| Tool | Semantic Analysis | Multi-Language | AI-Native Output | Standalone (no IDE) | Composite Queries |
|------|:-:|:-:|:-:|:-:|:-:|
| **LSAI** | Yes (compiler-level) | Yes (5 languages) | Yes (6 format profiles) | Yes | Yes (`impact`) |
| **GitHub MCP Server** | No (file/PR/issue operations) | N/A | No (raw API JSON) | Yes | No |
| **Serena** | Partial (LSP bridge) | Yes (via LSP) | No (verbose LSP JSON) | Yes | No |
| **JetBrains MCP** | Yes (IDE internals) | Yes (via IDE) | No (IDE-centric format) | No (requires running IDE) | No |
| **tree-sitter tools** | No (syntax only, no types) | Yes (grammar-based) | Partial | Yes | No |
| **LSP MCP bridges** | Yes (via LSP) | Yes (one per language) | No (raw LSP protocol) | Yes | No |

**Key differentiators:**
- **GitHub MCP Server** — operates on repositories and issues, not code semantics. Different domain entirely.
- **Serena** — wraps LSP, inherits LSP's chattiness and verbose output. No composite queries, no token optimization.
- **JetBrains MCP** — powerful but requires a running JetBrains IDE. Not standalone, not deployable to CI/headless environments.
- **tree-sitter tools** — fast syntax parsing but no type resolution. Can't answer "who implements this interface?" or "what breaks if I rename this?".
- **Generic LSP bridges** — expose raw LSP protocol to AI, which is verbose, cursor-centric, and requires multiple round-trips for basic questions.

---

## Specification

The full protocol specification is in [`spec/LSAI-v1.2.md`](spec/LSAI-v1.2.md).

It defines:
- 11 semantic tool definitions with input parameters and data contracts
- 6 output format profiles
- Capability tier system and plugin manifests
- Multi-language plugin architecture
- Transport (MCP-based, protocol-agnostic)
- Design principles with empirical evidence from 5 languages

---

## Contributing

LSAI is an open protocol. Implementations in any language are welcome.

To implement an LSAI-compliant server:
1. Read the [spec](spec/LSAI-v1.2.md)
2. Implement Tier 1 tools (7 tools) as MCP tools with the `lsai_` prefix
3. Declare your plugin's tier via the `server` meta-tool
4. Use relative paths, include summary counts, provide full signatures in outlines

---

## License

Protocol specification licensed under **CC BY-NC 4.0**. Free for study, research, and non-commercial use. Commercial implementations require a separate license — contact ladislav.sopko@gmail.com

---

Protocol designed by Ladislav Sopko, 0ics srl, Bologna, Italy — February 2026
