# LSAI Protocol v1.3 — Language Server for AI

> Revision: v1.3 (2026-03-15)
> Authors: Ladislav Sopko (0ics srl, Bologna, Italy)
> Status: Draft

---

## Abstract

LSAI is a protocol for AI consumption of compiler-semantic intelligence. It defines **12 semantic tools**, their **data contracts**, a **plugin capability system**, and **2 output format profiles** — enabling multi-language semantic analysis through a single MCP endpoint.

Unlike LSP (designed for IDEs), LSAI optimizes for:
- **Data richness per token** — full signatures, code context, composite queries
- **Fewer round-trips** — one LSAI call replaces 3-7 LSP calls
- **Only semantic operations** — no file I/O, no build, no tests
- **Multi-language** — plugin architecture supporting native compilers and LSP bridges

---

## Design Principles

1. **Data Richness Over Format** — What data is in the response matters more than how it's formatted. An outline with full method signatures eliminates follow-up file reads. A usage list with code context snippets tells the AI HOW a symbol is used, not just WHERE.

2. **Composite Queries** — One tool call returns what would require 3-7 LSP calls. `info` = hover + definition + type info + modifiers. `impact` = usages + callers + test coverage.

3. **Only What AI Can't Do** — No file read/write, no build, no test execution. Only compiler-semantic operations that require a type system. Everything else the AI can do with standard tools.

4. **Relative Paths Always** — All file paths are relative to the workspace root. Absolute paths waste ~80 characters per line — the single largest source of token waste in MCP tool responses.

5. **Format Agnostic** — The protocol defines what data each tool returns (the contract). How that data is serialized is an implementation choice. See [Output Format Profiles](#output-format-profiles).

6. **Capability Declaration** — Each language plugin declares what it supports via tiers. The AI adapts to available capabilities.

7. **Zero Redundancy** — Every token in the output must carry information. No duplicate names, no repetitive tags, no verbose footers. If something can be inferred from context, omit it.

---

## Empirical Evidence — 5 Languages, 25+ Queries

Measured on the Zerox.Lsai reference implementation (v1.0.60) across 5 languages using live MCP calls.

### LSAI vs Grep (text search)

| Metric | Savings |
|--------|:-------:|
| **Overall LSAI vs grep** | **~93%** |
| **Search** | 25-67% fewer tokens than old format |
| **Usages** | 29-74% fewer tokens |
| **Outline** | 43-79% fewer tokens |
| **Impact** | 61% fewer tokens |
| **Callers/Callees** | 8-73% fewer tokens |

### Per-Language Engine Performance

| Language | Engine | Tier | Search | Usages | Outline |
|----------|--------|:----:|:------:|:------:|:-------:|
| **C#** | Roslyn (native) | 3 | 44% | 29% | **79%** |
| **Python** | Pyright (LSP) | 2 | 25% | **74%** | **69%** |
| **TypeScript** | tsserver (LSP) | 2 | **67%** | **60%** | **55%** |
| **JavaScript** | tsserver (LSP) | 2 | 21% | — | — |
| **Java** | jdtls (LSP) | 2 | 25% | 25% | **64%** |

Savings are measured as new Compact format vs old CompactText format (v1.2), on identical queries.

### What Actually Matters

| Factor | Impact | Evidence |
|--------|--------|----------|
| **Data completeness** | HIGH | Outline with return types + params eliminates follow-up Reads |
| **Relative paths** | HIGH | Saves ~80 chars/line, ~400 tokens per response |
| **Zero redundancy** | HIGH | No brackets, no footers, no duplicate names = 37% savings |
| **Getter/setter collapsing** | HIGH | C# outline 51→19 lines (79% savings) |
| **Composite queries** | HIGH | `impact` = 5-8 LSP calls in one |
| **Code context in usages** | MEDIUM | Shows HOW a symbol is used (Verbose format) |

---

## Tool Definitions

### Naming Convention

Semantic tools use short names: `search`, `info`, `usages`, `callers`, `callees`, `hierarchy`, `impact`, `rename`, `diagnostics`, `outline`, `deps`, `source`.

MCP tool names are prefixed with `lsai_`: `lsai_search`, `lsai_info`, etc.

### Common Parameters

All tools that reference symbols accept either:
- `symbol` — name or fully qualified name
- OR `file` + `line` + `col` — location-based lookup

All tools accept:
- `outputFormat` — optional: `Compact` (default) or `Verbose`

---

### Tool 1: `search`

Find symbols by name pattern.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | yes | Search pattern. Supports `*` wildcard |
| `kind` | string[] | no | Filter: `class`, `interface`, `method`, `property`, `field`, `enum`, `struct`, `delegate`, `event` |
| `scope` | string | no | Namespace or project filter |
| `limit` | int | no | Max results. Default: 50 |

**Data Contract — each result contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Symbol name |
| `kind` | yes | Full word: `class`, `interface`, `method`, `property`, `field`, `enum`, `struct`, `delegate`, `event` |
| `file` | yes | Relative file path |
| `line` | yes | Line number (1-based) |
| `namespace` | no | Containing namespace |

**Example output (Compact):**
```
src/Services/DocumentService.cs:14 class DocumentService public EFine.Services.Domain
  base:BaseService implements:IDocumentService,IDisposable 12m
src/Contracts/IDocumentService.cs:8 interface IDocumentService public EFine.Contracts
  4m
```

**Tier:** 1 (required)

---

### Tool 2: `info`

Comprehensive symbol information in one call. Replaces LSP hover + definition + type info + modifiers.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes* | Name or fully qualified name |
| `file` | string | yes* | File path (relative) |
| `line` | int | yes* | Line number |
| `col` | int | yes* | Column number |

*Either `symbol` OR (`file` + `line` + `col`).

**Data Contract:**

| Field | Tier | Description |
|-------|------|-------------|
| `name` | 1 | Symbol name |
| `kind` | 1 | `class`, `interface`, `method`, `property`, `field`, etc. |
| `file` | 1 | Relative file path |
| `line` | 1 | Line number |
| `namespace` | 1 | Containing namespace |
| `modifiers` | 1 | `public`, `internal`, `sealed`, `static`, `async`, etc. |
| `signature` | 1 | Full declaration signature |
| `base` | 2 | Base class (for types) |
| `implements` | 2 | Implemented interfaces (for types) |
| `members` | 2 | Member count |
| `return` | 1 | Return type (for methods) |
| `params` | 1 | Parameter list (for methods) |
| `containing` | 1 | Containing type (for members) |

**Example output (Compact):**
```
src/Services/DocumentService.cs:14 class DocumentService public EFine.Services.Domain
  base:BaseService implements:IDocumentService,IDisposable 12m
```

**Tier:** 1 (basic) / 2 (extended) / 3 (complete)

---

### Tool 3: `usages`

Semantic references — knows that `Process` in a comment is not a usage.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes* | Symbol name |
| `file` | string | yes* | File containing the symbol |
| `line` | int | yes* | Line of symbol |
| `col` | int | yes* | Column of symbol |
| `limit` | int | no | Max results. Default: 100 |

**Data Contract — each result contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `file` | yes | Relative file path |
| `lines` | yes | Comma-separated line numbers |
| `context` | Verbose only | Code line at each usage site |

Results MUST be grouped by file with comma-separated line numbers.

**Example output (Compact):**
```
Controllers/DocController.cs:28,45
Services/DocumentService.cs:14
Tests/DocServiceTests.cs:12,33,67
```

**Example output (Verbose):**
```
Controllers/DocController.cs:
  :28 ref — private readonly IDocumentService _service;
  :45 ref — public DocController(IDocumentService service)
Services/DocumentService.cs:
  :14 ref — public class DocumentService : IDocumentService
```

**Tier:** 1 (required)

---

### Tool 4: `callers`

Who calls this method? Walks the call graph upward.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes* | Method or property name |
| `file` | string | yes* | File containing the symbol |
| `line` | int | yes* | Line of symbol |
| `col` | int | yes* | Column of symbol |
| `depth` | int | no | Levels up. Default: 1. Max: 5 |

**Data Contract — each result contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `method` | yes | Caller method as `Type.Method` |
| `file` | yes | Relative file path |
| `line` | yes | Line number |

**Example output (Compact):**
```
DocController.HandleRequest Controllers/DocController.cs:52
BatchJob.RunBatch Jobs/BatchJob.cs:88
```

**Example output (Verbose):**
```
Task<IActionResult> DocController.HandleRequest(int id, CancellationToken ct)
  Controllers/DocController.cs:52
  var doc = await _service.ProcessDocument(id);
```

**Tier:** 2 (extended)

---

### Tool 5: `callees`

What does this method call? Walks the call graph downward.

**Input:** Same as `callers`.

**Data Contract:** Same structure as `callers`, but direction is downward.

**Example output (Compact):**
```
Validator.Validate Services/Validator.cs:22
Repository.GetById Data/Repository.cs:45
Repository.Save Data/Repository.cs:78
```

**Tier:** 2 (extended)

---

### Tool 6: `hierarchy`

Inheritance/implementation tree for a type.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes* | Class, interface, or struct name |
| `file` | string | yes* | File containing the symbol |
| `line` | int | yes* | Line of symbol |
| `col` | int | yes* | Column of symbol |
| `direction` | string | no | `up`, `down`, `both`. Default: `both` |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `baseTypes` | yes | Ancestor chain (excluding `System.Object`) |
| `interfaces` | yes | Directly implemented interfaces |
| `derivedTypes` | yes | Types that inherit/implement this type |

**Example output (Compact):**
```
class DocumentService : BaseService
  implements: IDocumentService, IDisposable
  derived: CachedDocumentService
```

**Tier:** 2 (extended)

**Note:** Availability depends on LSP server support. Pyright (Python) does not support `textDocument/prepareTypeHierarchy`.

---

### Tool 7: `impact`

Composite analysis: if I change this symbol, what breaks? Combines usages + transitive callers + test coverage.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes* | Symbol to analyze |
| `file` | string | yes* | File containing the symbol |
| `line` | int | yes* | Line of symbol |
| `col` | int | yes* | Column of symbol |
| `depth` | int | no | Transitive caller depth. Default: 2 |

**Data Contract:**

| Field | Tier | Description |
|-------|------|-------------|
| `risk` | 1 | `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `usageCount` | 1 | Total references |
| `callerCount` | 2 | Total callers |
| `testCount` | 3 | Affected tests |
| `usages` | 1 | Compact file:lines |
| `callers` | 2 | Method:line pairs |
| `tests` | 3 | Test method names |

**Example output (Compact):**
```
MEDIUM 8refs 3callers 2tests
refs:DocController.cs:28,45 DocumentService.cs:14 DocServiceTests.cs:12,33,67
callers:DocController.HandleRequest:52 BatchJob.RunBatch:88 Scheduler.Execute:67
tests:TestProcess,TestBatchRun
```

**Example output (Verbose):** Full listing with code context for each usage and caller, recommendations.

**Tier:** 3 (full) / 1 (degraded — usages only)

---

### Tool 8: `rename`

Safe semantic rename with preview.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes* | Symbol to rename |
| `file` | string | yes* | File containing the symbol |
| `line` | int | yes* | Line of symbol |
| `col` | int | yes* | Column of symbol |
| `newName` | string | yes | New name |
| `preview` | bool | no | Preview without applying. Default: true |

**Data Contract (preview):**

| Field | Required | Description |
|-------|----------|-------------|
| `changes` | yes | Summary with file:line list |
| `changeCount` | yes | Total number of changes |

**Example output (Compact):**
```
Renamed ProcessDocument→HandleDocument 14 changes
  IDocumentService.cs:12 DocumentService.cs:28,45 DocController.cs:52,67,89
```

**Tier:** 1 (required)

---

### Tool 9: `diagnostics`

Compiler errors and warnings.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `scope` | string | no | File path, project name, or `*`. Default: `*` |
| `severity` | string[] | no | `error`, `warning`, `info`, `hint`. Default: `["error","warning"]` |
| `limit` | int | no | Max results. Default: 100 |

**Data Contract — each result contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `file` | yes | Relative file path |
| `line` | yes | Line number |
| `severity` | yes | Full word: `error`, `warning`, `info`, `hint` |
| `code` | yes | Diagnostic code (e.g., `CS0103`, `reportMissingImports`) |
| `message` | yes | Diagnostic message |

Implementations MUST filter out auto-generated files (`obj/`, `bin/`).

**Example output (Compact):**
```
Services/DocumentService.cs:32:5 error CS0103: '_repo' does not exist in the current context
Config/Startup.cs:44:12 error CS0121: ambiguous call between 'Configure(A)' and 'Configure(B)'
```

**Tier:** 1 (required)

---

### Tool 10: `outline`

Document structure as the compiler sees it.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | string | yes* | File path (relative) |
| `typeName` | string | yes* | Type name to outline |
| `depth` | int | no | Nesting depth. Default: 3 |

*Either `file` OR `typeName`.

**Data Contract — each member contains:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Member name (stripped of type prefix) |
| `kind` | yes | `class`, `method`, `property`, `field`, `function`, `variable`, `enum`, `enumMember` |
| `signature` | yes | Return type extracted from full declaration |
| `accessibility` | yes | `public`, `internal`, `protected`, `private` |
| `line` | yes | Line number |
| `isStatic` | no | Static modifier |

**Outline optimization rules:**
- Strip containing type prefix from member names (`LsaiOptions.DataPath` → `DataPath`)
- Collapse getter/setter pairs into `{get;set}` notation (both `Name.get`/`Name.set` and `get_Name`/`set_Name` patterns)
- Use `:line` notation (not `[line]` brackets)
- No footer counts

**The full signature is the most important field.** Without it, the AI must perform a follow-up file Read to understand what a method does — negating the efficiency gains of the tool.

**Example output (Compact):**
```
  SectionName string static :5
  DataPath string {get;set} :8
  PluginPath string {get;set} :9
  GitToken string? {get;set} :10
  ToolTimeoutSeconds int {get;set} :15
  CloudMode bool {get;set} :20
  Auth AuthOptions {get;set} :29
```

**Tier:** 1 (required)

---

### Tool 11: `deps`

Project dependency graph.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project` | string | no | Specific project. Default: all |
| `includeFiles` | bool | no | Include source files per project. Default: false |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `projects` | yes | List of projects with their project-to-project references |

**Example output (Compact):**
```
EFine.Api -> EFine.Services, EFine.Contracts
EFine.Services -> EFine.Data, EFine.Contracts
EFine.Data -> EFine.Contracts
EFine.Contracts -> (none)
EFine.Tests -> EFine.Api, EFine.Services
```

**Tier:** 1 (basic graph) / 2 (with file listing)

---

### Tool 12: `source`

Get source code of a specific symbol. Use after `search` to inspect implementation details without reading entire files.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | yes | Symbol name (class, method, function, enum, etc.) |
| `file` | string | no | File path to narrow scope (relative) |

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Symbol name |
| `kind` | yes | `class`, `method`, `function`, `enum`, `property`, `field`, etc. |
| `file` | yes | Relative file path |
| `line` | yes | Line number (1-based) |
| `source` | yes | Complete source code of the symbol body |

**Example output (Compact):**
```
src/Services/UserService.cs:18 method GetById
public User? GetById(int id) => _users.FirstOrDefault(u => u.Id == id);
```

**Token savings (measured):**

| Symbol Type | Source Size | File Size | Savings |
|-------------|:----------:|:---------:|:-------:|
| Method | 138 chars | 593 chars | **77%** |
| Function | 180 chars | 1129 chars | **84%** |
| Enum | 69 chars | 1129 chars | **94%** |
| Class (small) | 284 chars | 828 chars | **66%** |

**Tier:** 1 (required)

---

### Workspace & Meta Tools

These tools manage workspace lifecycle. They are not part of the semantic tool spec but are required for any LSAI server implementation.

#### `workspace_open`

Open a project/solution for analysis.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | yes | Path to project file (.csproj, .sln, .slnx) or directory |
| `language` | string | no | Override auto-detection (e.g., `C#`, `Python`, `TypeScript`, `Java`) |

**Returns:** Workspace ID (string) for use in subsequent tool calls.

#### `workspace_list`

List all open workspaces. No input. Returns workspace IDs, paths, and detected languages.

#### `workspace_close`

Close an open workspace and free resources.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `workspaceId` | string | yes | Workspace ID from `workspace_open` |

#### `server`

Capability discovery. No input.

**Data Contract:**

| Field | Required | Description |
|-------|----------|-------------|
| `version` | yes | Server version |
| `protocol` | yes | Protocol version (e.g., `LSAI/1.3`) |
| `plugins` | yes | List of loaded plugins with tier, languages, tool count |
| `workspaces` | yes | Open workspaces with language, path |

---

## Output Format Profiles

The protocol defines **what data** each tool returns. **How** that data is serialized is an implementation choice. Implementations MUST support at least one format.

### Profiles

| Profile | Style | Use Case |
|---------|-------|----------|
| **Compact** | Minimal tokens, no context snippets, no footers, no brackets | Default. Lowest token cost for AI consumption |
| **Verbose** | Compact structure with code context snippets and full signatures | When the AI needs to understand HOW symbols are used, not just WHERE |

### Compact Format Rules

All implementations of the Compact profile MUST follow these rules:

1. **No square brackets** — metadata is space-separated, not `[bracketed]`
2. **No duplicate names** — symbol name appears once, not repeated in metadata
3. **No footer counts** — no `[N symbols]`, `[N usages in M files]`, `[N callers]`
4. **Comma-separated lines** — usages grouped as `file:line1,line2,line3`
5. **Collapsed getter/setter** — properties show `{get;set}` not separate `.get`/`.set` lines
6. **Stripped type prefix** — outline members show `DataPath` not `LsaiOptions.DataPath`
7. **Type.Method callers** — callers show `Type.Method file:line`, not full signature
8. **Summary impact** — first line has `RISK Nrefs Ncallers Ntests`, details below
9. **Summary rename** — `Renamed Old→New N changes` with file:lines, no file content
10. **Abbreviated member count** — `3m` not `3 members`

### Verbose Format Rules

Verbose extends Compact with:
- **Code context** in usages (the actual code line at each reference site)
- **Abbreviated kind** in usages (`ref`, `def`, `impl` instead of full words)
- **Full signatures** in callers/callees (return type + parameters)
- **Code context** in callers/callees
- **Full listing** in impact (not just summary)

### Format Requirements (Both Profiles)

- Use **relative paths** (never absolute)
- Include **full signatures** in outline (return types, parameters)
- Use **full words** for kinds (`class`, not `C`; `error`, not `E`)

### Why Only Two Formats

v1.2 defined 6 format profiles. Empirical analysis on 223 real MCP queries across 5 languages showed:

- 4 out of 6 formats produced **identical or near-identical output** for most tools
- `LanguageSyntax` added `//` prefixes — MORE tokens, not fewer
- `CompilerOutput` used `()` instead of `:` — no information difference
- `GrepOutput` was nearly identical to `CompactText`
- `TurboCompact` saved marginal tokens but lost readability

**Conclusion:** Format style is not where savings come from. Eliminating redundancy (brackets, footers, duplicate names, uncollapsed getter/setter) is. Two formats cover all real use cases: Compact for minimum tokens, Verbose when code context is needed.

---

## Multi-Language Plugin Architecture

LSAI supports multiple languages through a plugin system. Each plugin implements the `ILsaiPlugin` interface and declares its capabilities.

### Plugin Types

| Type | Description | Tier | Example |
|------|-------------|:----:|---------|
| **Native** | Direct compiler API access | 3 | Roslyn for C# |
| **LSP Bridge** | Wraps an LSP server via StreamJsonRpc | 2 | Pyright, tsserver, jdtls |
| **SCIP** | Static index from SCIP protobuf files | 2 | scip-java, scip-typescript, scip-python |

### Supported Languages (Reference Implementation)

| Language | Engine | Plugin | Extensions |
|----------|--------|--------|------------|
| C# | Roslyn | Native | `.cs`, `.csproj`, `.sln`, `.slnx` |
| Python | Pyright | LSP Bridge | `.py` |
| TypeScript | tsserver | LSP Bridge | `.ts`, `.tsx` |
| JavaScript | tsserver | LSP Bridge | `.js`, `.jsx` |
| Java | Eclipse JDT (jdtls) | LSP Bridge | `.java` |

### Known LSP Server Limitations

These are upstream LSP server behaviors, not LSAI limitations:

| Language | Limitation |
|----------|-----------|
| JavaScript | `workspace/symbol` not supported for CommonJS `.js` files |
| JavaScript | `<unknown>` symbol names for anonymous CommonJS exports |
| Python | `textDocument/prepareTypeHierarchy` not implemented by Pyright |
| Java | `workspace/symbol` does not expose fields or record components |
| Java | Fuzzy matching on `workspace/symbol` (may return extra results) |

---

## Capability Tiers

| Tier | Tools | Minimum Backend |
|------|-------|-----------------|
| **1** | search, info (basic), usages, rename, diagnostics, outline, deps, source | Any LSP server |
| **2** | + callers, callees, hierarchy, info (extended), deps (with files) | LSP 3.17+ |
| **3** | + impact (full), info (complete), cross-project | Native compiler API |

### Plugin Manifest

```json
{
  "id": "roslyn-csharp",
  "languages": ["csharp"],
  "extensions": [".cs", ".csproj", ".sln", ".slnx"],
  "tier": 3,
  "tools": {
    "search": true, "info": "full", "usages": true,
    "callers": true, "callees": true, "hierarchy": true,
    "impact": true, "rename": true, "diagnostics": true,
    "outline": true, "deps": true, "source": true
  },
  "backend": "roslyn-native"
}
```

---

## Transport

LSAI is an MCP server. It defines tool semantics and data contracts, not transport.

```
AI Agent ──MCP──> LSAI Server ──native/lsp──> Compiler / Language Server
                  (16 tools)                   (Roslyn, Pyright, tsserver, jdtls)
```

---

## Changes from v1.2

| What | v1.2 | v1.3 | Why |
|------|------|------|-----|
| **Output formats** | 6 profiles (CompactText, CompactTextVerbose, TurboCompact, GrepOutput, CompilerOutput, LanguageSyntax) | 2 profiles (Compact, Verbose) | 4 formats were redundant — empirically proven on 223 queries |
| **Compact optimizations** | Brackets, footers, duplicate names, separate getter/setter | No brackets, no footers, no duplicates, collapsed `{get;set}`, comma-separated usages, summary impact/rename | 37% fewer tokens on same queries |
| **Token savings** | 89% vs grep | **93% vs grep** | Format optimization + redundancy elimination |
| **Zero Redundancy principle** | Not stated | Design Principle #7 | Formalized what the optimization proved |
| **SCIP plugin type** | Not available | Static index via SCIP protobuf | SaaS deployment without running LSP servers |
| **Outline rules** | Signatures required | + getter/setter collapsing, type prefix stripping, `:line` notation | Measured 79% outline savings on C# |
| **Impact format** | Verbose listing | Summary first line + compact refs/callers | 61% savings, same information |
| **Rename format** | Full file content dumps | Summary + file:line list | 99% savings (2188→~20 tokens) |

---

## Versioning

- **LSAI/1.0** — Initial spec. 11 tools, 3 tiers, pipe-delimited format.
- **LSAI/1.1** — Format-agnostic, data contracts, enriched fields.
- **LSAI/1.2** — Multi-language, 12 semantic tools (added `source`), 6 output formats, empirical benchmarks.
- **LSAI/1.3** — This document. Output format optimization (6→2 formats), 37% token reduction, zero redundancy principle, SCIP plugin type.
- Future versions add tools, never remove. Tiers may be extended.

---

## License

Protocol specification licensed under CC BY-NC 4.0. Free for study, research, and non-commercial use. Commercial implementations require a separate license — contact ladislav.sopko@gmail.com

---

*LSAI — Language Server for AI. Compiler intelligence, optimized for LLMs.*

*Protocol designed by Ladislav Sopko, 0ics srl, Bologna, Italy.*
*March 2026*
