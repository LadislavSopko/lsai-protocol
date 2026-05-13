# LSAI Protocol v1.4 — Language Server for AI

> Revision: v1.4 (2026-05-13)
> Authors: Ladislav Sopko (0ics srl, Bologna, Italy)
> Status: Draft

---

## Abstract

LSAI is a protocol for AI consumption of compiler-semantic intelligence. It defines **14 semantic tools**, their **data contracts**, a **plugin capability system**, and **2 output format profiles** — enabling multi-language semantic analysis through a single MCP endpoint.

Unlike LSP (designed for IDEs), LSAI optimizes for:
- **Data richness per token** — full signatures, code context, composite queries
- **Fewer round-trips** — one LSAI call replaces 3-7 LSP calls
- **Only semantic operations** — no file I/O, no build, no tests
- **Multi-language** — plugin architecture supporting native compilers and LSP bridges
- **Fallback resilience** — graceful degradation when upstream LSP lacks a capability

---

## Design Principles

1. **Data Richness Over Format** — What data is in the response matters more than how it's formatted. An outline with full method signatures eliminates follow-up file reads. A usage list with code context snippets tells the AI HOW a symbol is used, not just WHERE.

2. **Composite Queries** — One tool call returns what would require 3-7 LSP calls. `info` = hover + definition + type info + modifiers. `impact` = usages + callers + test coverage.

3. **Only What AI Can't Do** — No file read/write, no build, no test execution. Only compiler-semantic operations that require a type system. Everything else the AI can do with standard tools.

4. **Relative Paths Always** — All file paths are relative to the workspace root. Absolute paths waste ~80 characters per line — the single largest source of token waste in MCP tool responses.

5. **Format Agnostic** — The protocol defines what data each tool returns (the contract). How that data is serialized is an implementation choice. See [Output Format Profiles](#output-format-profiles).

6. **Capability Declaration** — Each language plugin declares what it supports via manifests. The AI adapts to available capabilities.

7. **Zero Redundancy** — Every token in the output must carry information. No duplicate names, no repetitive tags, no verbose footers. If something can be inferred from context, omit it.

8. **Parasitic Architecture** — LSAI never builds, compiles, or modifies projects. It attaches to already-built workspaces and reads compiler artifacts. Projects must be pre-built by their native toolchains.

9. **Fallback Resilience** — When an upstream LSP server lacks a capability (e.g. callHierarchy), the implementation SHOULD provide a fallback strategy that still returns useful data rather than failing.

---

## Tool Definitions

### Tool Summary

| # | Tool | Purpose | Tier |
|---|------|---------|:----:|
| 1 | `search` | Find symbols by name | 1 |
| 2 | `info` | Symbol details — signature, docs, type | 1 |
| 3 | `usages` | All references to a symbol | 1 |
| 4 | `callers` | Call graph: who calls this method | 2 |
| 5 | `callees` | Call graph: what does this method call | 2 |
| 6 | `hierarchy` | Type inheritance chain | 2 |
| 7 | `impact` | Change risk: usages + callers + tests | 3 |
| 8 | `rename` | Semantic rename across workspace | 1 |
| 9 | `diagnostics` | Compiler errors and warnings | 1 |
| 10 | `outline` | Document/type structure with signatures | 1 |
| 11 | `deps` | File-level dependency analysis | 1 |
| 12 | `source` | Symbol implementation source code | 1 |
| 13 | `file_refs` | Cross-file reference map for a file | 1 |
| 14 | `context` | Composite: outline + diagnostics + usages + callers + risk | 3 |

### Naming Convention

Semantic tools use short names: `search`, `info`, `usages`, `callers`, `callees`, `hierarchy`, `impact`, `rename`, `diagnostics`, `outline`, `deps`, `source`, `file_refs`, `context`.

MCP tool names are prefixed with `lsai_`: `lsai_search`, `lsai_info`, etc.

### Common Parameters

All tools that reference symbols accept either:
- `symbol` — name or fully qualified name
- OR `file` + `line` + `col` — location-based lookup

All tools accept:
- `workspaceId` — required: identifies the open workspace
- `outputFormat` — optional: `Compact` (default) or `Verbose`

---

### Tool 1: `search`

Find symbols by name pattern.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | yes | Search pattern |
| `kind` | string[] | no | Filter: `class`, `interface`, `method`, `property`, `field`, `enum`, `struct` |
| `limit` | int | no | Max results. Default: 50 |

**Data Contract — each result contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Symbol name |
| `kind` | yes | Full word: `class`, `interface`, `method`, `property`, `field`, `enum`, `struct` |
| `file` | yes | Relative file path |
| `line` | yes | Line number (1-based) |
| `namespace` | no | Containing namespace |

**Tier:** 1 (required)

---

### Tool 2: `info`

Comprehensive symbol information in one call. Replaces LSP hover + definition + type info + modifiers.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes* | Name or fully qualified name |
| `file` | string | no | File path to narrow scope |

*Either `symbol` OR (`file` + `line` + `col`).

**Data Contract:**

| Field | Description |
|-------|-------------|
| `name` | Symbol name |
| `kind` | `class`, `interface`, `method`, `property`, `field`, etc. |
| `file` | Relative file path |
| `line` | Line number |
| `namespace` | Containing namespace |
| `modifiers` | `public`, `internal`, `sealed`, `static`, `async`, etc. |
| `signature` | Full declaration signature |
| `base` | Base class (for types) |
| `implements` | Implemented interfaces (for types) |
| `members` | Member count |
| `documentation` | Docstring / XML doc / JSDoc |

**Tier:** 1

---

### Tool 3: `usages`

Semantic references — knows that `Process` in a comment is not a usage.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes | Symbol name |
| `file` | string | no | File path to narrow scope |

**Data Contract — results MUST be grouped by file:**

| Field | Required | Description |
|-------|----------|-------------|
| `file` | yes | Relative file path |
| `lines` | yes | Comma-separated line numbers |
| `context` | Verbose only | Code line at each usage site |

**Tier:** 1 (required)

---

### Tool 4: `callers`

Who calls this method? Walks the call graph upward.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes | Method or function name |
| `file` | string | no | File path to narrow scope |

**Data Contract — each result contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `method` | yes | Caller method as `Type.Method` |
| `file` | yes | Relative file path |
| `line` | yes | Line number |
| `context` | Verbose only | Code line at call site |

**Fallback:** When the upstream LSP does not implement `callHierarchy/incomingCalls`, implementations SHOULD fall back to `textDocument/references` (exclude declaration) and resolve the containing function/method via `textDocument/documentSymbol` for each reference location.

**Tier:** 2 (extended)

---

### Tool 5: `callees`

What does this method call? Walks the call graph downward.

**Input:** Same as `callers`.

**Data Contract:** Same structure as `callers`, but direction is downward.

**Fallback:** When the upstream LSP does not implement `callHierarchy/outgoingCalls`, implementations SHOULD fall back to reading the method body range from `textDocument/documentSymbol`, regex-scanning for call-site identifiers, and resolving each via `workspace/symbol`. Matches whose `SymbolKind` is Function/Method/Constructor are kept. When the LSP returns qualified names (e.g. `Calc::compute` in C++), the name matching MUST strip namespace qualifiers before comparison.

**Tier:** 2 (extended)

---

### Tool 6: `hierarchy`

Inheritance/implementation tree for a type.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes | Class, interface, struct, or trait name |
| `file` | string | no | File path to narrow scope |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `typeName` | yes | Type name |
| `kind` | yes | `class`, `interface`, `struct`, `trait`, `enum` |
| `baseType` | no | Base class name |
| `interfaces` | yes | List of implemented interfaces/traits with optional location |
| `derivedTypes` | yes | List of types that inherit/implement this type |

**Fallback:** When the upstream LSP does not implement `textDocument/prepareTypeHierarchy`, implementations SHOULD return a minimal node with the type name and kind, and empty interfaces/derivedTypes, rather than throwing an error.

**Tier:** 2 (extended)

---

### Tool 7: `impact`

Composite analysis: if I change this symbol, what breaks?

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes | Symbol to analyze |
| `file` | string | no | File path to narrow scope |

**Data Contract:**

| Field | Description |
|-------|-------------|
| `risk` | `HIGH`, `MEDIUM`, `LOW` |
| `usageCount` | Total references |
| `callerCount` | Total callers |
| `testCount` | Affected tests |
| `usages` | Compact file:lines |
| `callers` | Method:line pairs |
| `tests` | Test file names |

**Resilience:** Impact combines usages + callers internally. If the callers sub-operation fails (e.g. `RemoteMethodNotFoundException` from an LSP without callHierarchy), the implementation MUST catch the exception and return empty callers rather than propagating the error.

**Tier:** 3 (full) / 1 (degraded — usages only)

---

### Tool 8: `rename`

Safe semantic rename.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes | Symbol to rename |
| `newName` | string | yes | New name |
| `file` | string | no | File path to narrow scope |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `changes` | yes | Summary with file:line list |
| `changeCount` | yes | Total number of changes |

**Note:** Some LSP servers do not support rename (e.g. intelephense free tier for PHP). Implementations SHOULD return a clean `ToolNotSupported` error rather than crashing.

**Tier:** 1 (required where LSP supports it)

---

### Tool 9: `diagnostics`

Compiler errors and warnings.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | no | Filter by file |
| `projectName` | string | no | Filter by project |
| `minSeverity` | string | no | `error`, `warning`, `info`. Default: all |

**Data Contract — each result contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `file` | yes | Relative file path |
| `line` | yes | Line number |
| `severity` | yes | `error`, `warning`, `info`, `hint` |
| `code` | yes | Diagnostic code |
| `message` | yes | Diagnostic message |

**Tier:** 1 (required)

---

### Tool 10: `outline`

Document structure as the compiler sees it.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes* | File path (relative) |
| `typeName` | string | yes* | Type name to outline |

*Either `filePath` OR `typeName`.

**Data Contract — each member contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Member name (stripped of type prefix) |
| `kind` | yes | `class`, `method`, `property`, `field`, `function`, `variable`, `enum` |
| `signature` | yes | Return type / full declaration |
| `accessibility` | no | `public`, `internal`, `protected`, `private` |
| `line` | yes | Line number |

**Outline optimization rules:**
- Strip containing type prefix from member names
- Collapse getter/setter pairs into `{get;set}` notation
- Use `:line` notation (not `[line]` brackets)
- No footer counts

**Tier:** 1 (required)

---

### Tool 11: `deps`

File-level dependency analysis. Parses import/include/use/require statements from source files.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectName` | string | no | Specific project. Default: all |
| `dependents` | bool | no | Include reverse dependents. Default: false |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `file` | yes | Relative file path |
| `dependencies` | yes | Comma-separated import/include names |
| `dependents` | no | Files that import this module (if requested) |

**Language-aware parsing:** Implementations MUST dispatch on the workspace language to select the correct import parser:
- Python: `import X` / `from X import Y`
- TypeScript/JavaScript: `import ... from 'X'` / `require('X')`
- Java: `import X;`
- PHP: `use X;` / `require_once 'X'`
- Rust: `use X;` / `mod X;`
- C/C++: `#include <X>` / `#include "X"`

**Note:** Deps is a purely local operation (no LSP call). It parses source files directly.

**Tier:** 1 (basic)

---

### Tool 12: `source`

Get source code of a specific symbol.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes | Symbol name |
| `file` | string | no | File path to narrow scope |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Symbol name |
| `kind` | yes | `class`, `method`, `function`, `enum`, etc. |
| `file` | yes | Relative file path |
| `line` | yes | Line number (1-based) |
| `source` | yes | Complete source code of the symbol body |

**Tier:** 1 (required)

---

### Tool 13: `file_refs` (new in v1.4)

Cross-file reference map for a given file. Shows which other files reference symbols defined in the target file, and which symbols are referenced.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | File path (relative) to analyze |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `file` | yes | The analyzed file |
| `direction` | yes | `incoming` (who references this file) |
| `references` | yes | List of referencing files with symbol names and ref counts |

**Implementation:** Composite operation — calls `outline` on the target file, then `usages` for each symbol, aggregates by referencing file.

**Tier:** 1 (composite, no special LSP requirement)

---

### Tool 14: `context` (new in v1.4)

Composite overview of a file or symbol — combines outline + diagnostics + usages + callers + risk assessment in a single call. Designed for the AI to quickly understand a file before making changes.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes* | File to analyze |
| `symbolName` | string | yes* | Symbol to analyze |

*Either `filePath` OR `symbolName`.

**Data Contract:**

| Field | Description |
|-------|-------------|
| `outline` | Members list (from `outline`) |
| `diagnostics` | Errors/warnings (from `diagnostics`) |
| `usages` | References per symbol (from `usages`) |
| `callers` | Call graph for key methods (from `callers`) |
| `risk` | Overall risk assessment |

**Tier:** 3 (composite)

---

### Workspace & Meta Tools

These tools manage workspace lifecycle. They are not part of the semantic tool spec but are required for any LSAI server implementation.

#### `workspace_open`

Open a project/solution for analysis. LSAI auto-opens the workspace from the server's cwd; this tool is needed only when analyzing a path outside the cwd.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | yes | Path to project file or directory |
| `language` | string | no | Override auto-detection |

**Returns:** Workspace ID (string) for use in subsequent tool calls.

#### `workspace_list`

List all open workspaces. No input. Returns workspace IDs, paths, languages, and readiness status.

#### `workspace_close`

Close an open workspace and free resources.

#### `server`

Capability discovery. No input.

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `version` | yes | Server version |
| `plugins` | yes | List of loaded plugins with tier, languages, tool count |
| `workspaces` | yes | Open workspaces with language, path, status |

---

## Output Format Profiles

The protocol defines **what data** each tool returns. **How** that data is serialized is an implementation choice. Implementations MUST support at least one format.

### Profiles

| Profile | Style | Use Case |
|---------|-------|----------|
| **Compact** | Minimal tokens, no context snippets, no footers, no brackets | Default. Lowest token cost for AI consumption |
| **Verbose** | Compact structure with code context snippets and full signatures | When the AI needs to understand HOW symbols are used, not just WHERE |

### Compact Format Rules

1. **No square brackets** — metadata is space-separated
2. **No duplicate names** — symbol name appears once
3. **No footer counts** — no `[N symbols]`, `[N usages]`
4. **Comma-separated lines** — usages grouped as `file:line1,line2,line3`
5. **Collapsed getter/setter** — properties show `{get;set}`
6. **Stripped type prefix** — outline members show `DataPath` not `LsaiOptions.DataPath`
7. **Type.Method callers** — callers show `Type.Method file:line`
8. **Summary impact** — first line has `RISK Nrefs Ncallers Ntests`
9. **Summary rename** — `Renamed Old->New N changes`
10. **Abbreviated member count** — `3m` not `3 members`

---

## Multi-Language Plugin Architecture

LSAI supports multiple languages through a plugin system. Each plugin implements the `ILsaiPlugin` interface and declares its capabilities.

### Plugin Types

| Type | Description | Example |
|------|-------------|---------|
| **Native** | Direct compiler API access | Roslyn for C# |
| **LSP Bridge** | Wraps an LSP server via StreamJsonRpc | ty, tsserver, jdtls, intelephense, rust-analyzer, gopls, clangd |

### Supported Languages (Reference Implementation v1.0.176)

| Language | Engine | Plugin | Extensions |
|----------|--------|--------|------------|
| C# | Roslyn | Native | `.cs`, `.csproj`, `.sln`, `.slnx` |
| Python | ty (Astral) | LSP Bridge | `.py`, `.pyi` |
| TypeScript | typescript-language-server | LSP Bridge | `.ts`, `.tsx` |
| JavaScript | typescript-language-server | LSP Bridge | `.js`, `.jsx`, `.mjs`, `.cjs` |
| Java | Eclipse JDT (jdtls) | LSP Bridge | `.java` |
| PHP | intelephense | LSP Bridge | `.php` |
| Rust | rust-analyzer | LSP Bridge | `.rs` |
| Go | gopls | LSP Bridge | `.go` |
| C | clangd | LSP Bridge | `.c`, `.h` |
| C++ | clangd | LSP Bridge | `.cpp`, `.cxx`, `.cc`, `.hpp`, `.hxx` |

### Fallback Strategies

When an upstream LSP server does not implement a protocol method, LSAI provides fallback strategies:

| Tool | LSP Method | Fallback Strategy | Affected Languages |
|------|-----------|-------------------|-------------------|
| `callers` | `callHierarchy/incomingCalls` | `textDocument/references` + `documentSymbol` containment | ty (Python) |
| `callees` | `callHierarchy/outgoingCalls` | Method body regex + `workspace/symbol` resolution | clangd (C/C++) |
| `hierarchy` | `prepareTypeHierarchy` | Return minimal node (name + kind, empty relationships) | ty (Python), some LSPs |
| `impact` | (uses callers internally) | Catch caller failures, return usages-only result | Any language where callers fails |

### Known LSP Server Limitations

These are upstream LSP server behaviors, not LSAI limitations:

| Language | Limitation |
|----------|-----------|
| JavaScript | `workspace/symbol` not supported for CommonJS `.js` files |
| JavaScript | `<unknown>` symbol names for anonymous CommonJS exports |
| Python | `textDocument/prepareTypeHierarchy` not implemented by ty |
| Java | `workspace/symbol` does not expose fields or record components |
| Java | External Maven/Gradle dependencies not indexed by jdtls |
| PHP | `rename` not supported by intelephense free tier |
| PHP | Workspace symbol index may retain stale entries after file revert |
| TypeScript | `typeHierarchy/supertypes` may not populate interface relationships |
| C++ | `callHierarchy/outgoingCalls` not implemented by clangd (LLVM PR #117673) |
| C++ | `workspace/symbol` returns qualified names (e.g. `Calc::add`) |

---

## Capability Tiers

All tools are available on all languages in the reference implementation (v1.0.176). Tiers indicate the minimum LSP backend requirement:

| Tier | Tools | Minimum Backend |
|------|-------|-----------------|
| **1** | search, info, usages, rename, diagnostics, outline, deps, source, file_refs | Any LSP server |
| **2** | + callers, callees, hierarchy (with fallbacks) | LSP 3.17+ or fallback |
| **3** | + impact, context (composites) | Tier 2 + composite logic |

### Plugin Manifest

```json
{
  "name": "Zerox.Lsai.Plugin.Lsp",
  "version": "1.0.176",
  "language": "Python",
  "tier": 3,
  "supportedTools": [
    "Search", "Info", "Usages", "Rename", "Diagnostics",
    "Outline", "Callers", "Callees", "Hierarchy", "Impact", "Deps"
  ]
}
```

---

## Transport

LSAI is an MCP server. It defines tool semantics and data contracts, not transport.

```
AI Agent --MCP--> LSAI Server --native/lsp--> Compiler / Language Server
                  (18 tools)                   (Roslyn, ty, tsserver, jdtls,
                                                intelephense, rust-analyzer,
                                                gopls, clangd)
```

---

## Live Editing

LSAI tracks file changes in real-time via file system watchers. When the AI edits a file:

1. The file system watcher detects the change
2. LSAI notifies the underlying LSP server via `textDocument/didChange`
3. The LSP server re-indexes
4. The next tool call reflects the updated state

This enables a seamless edit-then-query workflow without manual refresh.

**Implementation notes:**
- WSL2/NTFS uses polling-based watchers (inotify unavailable through bind mount)
- Debounce window: 300ms (configurable)
- Skip directories: `.git`, `node_modules`, `bin`, `obj`, `build`, `target`, `dist`, `__pycache__`, `vendor`

---

## Changes from v1.3

| What | v1.3 | v1.4 | Why |
|------|------|------|-----|
| **Tool count** | 12 semantic tools | **14 semantic tools** | Added `file_refs` (cross-file map) and `context` (composite overview) |
| **Languages** | 5 (C#, Python, TS, JS, Java) | **10** (+ PHP, Rust, Go, C, C++) | Reference implementation expanded |
| **Python engine** | Pyright | **ty** (Astral) | Upstream migration to faster type checker |
| **Fallback strategies** | Not specified | **Documented** for callers, callees, hierarchy, impact | Ensures all tools work on all languages |
| **Known limitations** | 5 items | **10 items** | Added PHP, Rust, Go, C++ specific behaviors |
| **Parasitic principle** | Implicit | **Design Principle #8** | Formalized: LSAI never builds |
| **Fallback resilience** | Not stated | **Design Principle #9** | LSP gaps handled gracefully |
| **Live editing** | Not documented | **New section** | FSW + flush contract documented |
| **SCIP plugin type** | Listed | **Removed** (local-only spec) | SCIP is SaaS-only, not in local reference implementation |
| **Tier system** | Variable per language | All languages Tier 3 in ref impl | All tools enabled everywhere via config |

---

## Versioning

- **LSAI/1.0** — Initial spec. 11 tools, 3 tiers, pipe-delimited format.
- **LSAI/1.1** — Format-agnostic, data contracts, enriched fields.
- **LSAI/1.2** — Multi-language, 12 semantic tools (added `source`), 6 output formats.
- **LSAI/1.3** — Output format optimization (6->2 formats), 37% token reduction, zero redundancy.
- **LSAI/1.4** — This document. 14 tools (+file_refs, +context), 10 languages (+PHP, Rust, Go, C, C++), Python engine ty, fallback strategies, live editing, parasitic principle.
- Future versions add tools, never remove.

---

## License

Protocol specification licensed under CC BY-NC 4.0. Free for study, research, and non-commercial use. Commercial implementations require a separate license — contact ladislav.sopko@gmail.com

---

*LSAI — Language Server for AI. Compiler intelligence, optimized for LLMs.*

*Protocol designed by Ladislav Sopko, 0ics srl, Bologna, Italy.*
*May 2026*
