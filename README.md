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
| **Output format** | Verbose JSON with TextEdit ranges, Position objects | Compact, AI-native (format-agnostic) |
| **Capabilities** | Varies by server, no standard capability tiers | Declared tiers (1/2/3) per plugin |
| **Composite queries** | Not supported — client must orchestrate | Built-in (`impact` = usages + callers + tests) |
| **Multi-language** | One server per language | Plugin architecture, one endpoint |

### Measured: Grep/Read vs LSAI-style semantic tools

Real measurements on the Zerox.Lsai codebase (7 C# projects, 313 tests), comparing standard AI agent tools (Grep + Read) against VS-MCP semantic tools. Same queries, same codebase, same session.

| Operation | Grep/Read (standard AI tools) | LSAI-style (VS-MCP) | Char savings | Call savings |
|-----------|-------------------------------|----------------------|:------------:|:-----------:|
| **Find symbol** `WorkspaceSessionManager` | 696 chars, 5 results (3 noise from .md files) | 133 chars, 1 precise result | **81%** | 1 call vs 1 call (but 100% precision) |
| **Find all usages** (48 refs) | 7,027 chars, 52 lines, absolute paths, text match | 2,189 chars, 48 usages, relative paths, grouped by scope | **69%** | 1 vs 1 (but semantic, no false positives) |
| **Understand a class** (outline) | 3,865 chars: Grep to find file + Read entire file | 556 chars: structured outline with members and types | **86%** | 2 calls vs 1 call |
| **Find callers** of `OpenAsync` | 3,361 chars from Grep + need 5-10 Read calls to verify actual call sites | 1,973 chars, 16 semantic callers with Type.Method + file:line | **41-70%** | 5-10 calls vs 1 call |

**Average char savings: 69-81%.** But the real win is precision and fewer round-trips — Grep returns text matches (including comments, strings, markdown), LSAI returns compiler-verified semantic references.

---

## Tools

LSAI defines 11 semantic tools organized in 3 capability tiers. All tools operate through MCP (Model Context Protocol) with the `lsai_` prefix.

| Tool | Tier | Description | Replaces (LSP equivalent) |
|------|------|-------------|--------------------------|
| `search` | 1 | Find symbols by name pattern with kind/scope filters | `workspace/symbol` |
| `info` | 1/2/3 | Full symbol details: signature, modifiers, base types, members | `hover` + `definition` + `typeDefinition` |
| `usages` | 1 | Semantic references grouped by file with code context | `references` |
| `rename` | 1 | Safe rename with preview, cross-file, compilation check | `rename` |
| `diagnostics` | 1 | Compiler errors/warnings, filtered (no `obj/` noise) | `diagnostics` |
| `outline` | 1 | Document structure with **full method signatures** | `documentSymbol` (but richer) |
| `deps` | 1/2 | Project dependency graph | No LSP equivalent |
| `callers` | 2 | Who calls this method? Multi-depth call graph upward | `callHierarchy/incomingCalls` |
| `callees` | 2 | What does this method call? Call graph downward | `callHierarchy/outgoingCalls` |
| `hierarchy` | 2 | Inheritance tree: base types, interfaces, derived types | `typeHierarchy` |
| `impact` | 3 | Composite: usages + transitive callers + affected tests + risk | No LSP equivalent |

Plus `server` meta-tool for capability discovery (loaded plugins, tiers, open workspaces).

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

5. **Format Agnostic** — The protocol defines data contracts (what each tool returns). Serialization format (JSON, pipe-delimited, plain text) is an implementation choice. Empirical testing across Claude Opus 3.5/4.6 and OpenAI Codex confirms: LLMs decode any compact format with zero friction.

6. **Capability Declaration** — Each plugin declares its tier. The AI knows what's available without trial and error.

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
| **Output format** | Compact pipe-delimited JSON |
| **Key insight** | Battle-tested proof that compact AI-native formats work with zero LLM friction |

VS-MCP is the production system that validated LSAI's core thesis: compiler semantic data in a compact format dramatically improves AI coding agent performance. The LSAI protocol formalizes what VS-MCP proved empirically.

### LSAI Reference Server

| Property | Value |
|----------|-------|
| **Project** | [Zerox.Lsai](https://github.com/LadislavSopko/Roslyn.Poc) — standalone LSAI server |
| **Tier** | 3 |
| **Languages** | C# (Roslyn plugin), extensible via plugin architecture |
| **Transport** | HTTP (Streamable HTTP via MCP SDK) |
| **Status** | Working prototype, 11 tools, 313 tests |

---

## Prior Art

Every existing tool has at least one critical gap that LSAI addresses.

| Tool | Semantic Analysis | Multi-Language | AI-Native Output | Standalone (no IDE) | Composite Queries |
|------|:-:|:-:|:-:|:-:|:-:|
| **LSAI** | Yes (compiler-level) | Yes (plugin arch) | Yes (format-agnostic, token-optimized) | Yes | Yes (`impact`) |
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

The full protocol specification is in [`spec/LSAI-v1.1.md`](spec/LSAI-v1.1.md).

It defines:
- 11 tool definitions with input parameters and data contracts
- Output format profiles (CompactJson, CompactText)
- Capability tier system and plugin manifests
- Transport (MCP-based, protocol-agnostic)
- Design principles with empirical evidence

---

## Contributing

LSAI is an open protocol. Implementations in any language are welcome.

To implement an LSAI-compliant server:
1. Read the [spec](spec/LSAI-v1.1.md)
2. Implement Tier 1 tools (7 tools) as MCP tools with the `lsai_` prefix
3. Declare your plugin's tier via the `server` meta-tool
4. Use relative paths, include summary counts, provide full signatures in outlines

---

## License

MIT. See [LICENSE](LICENSE).

---

Protocol designed by Ladislav Sopko, 0ics srl, Bologna, Italy — February 2026
