# LSAI Protocol v1.2 — Language Server for AI

> Revision: v1.2 (2026-02-15)
> Authors: Ladislav Sopko (0ics srl, Bologna, Italy)
> Status: Draft

---

## Abstract

LSAI is a protocol for AI consumption of compiler-semantic intelligence. It defines **11 semantic tools**, their **data contracts**, a **plugin capability system**, and **6 output format profiles** — enabling multi-language semantic analysis through a single MCP endpoint.

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

---

## Empirical Evidence — 5 Languages, 46 Symbols

Measured on the Zerox.Lsai reference implementation across 5 languages. Full data: [RESULTS.md](https://github.com/LadislavSopko/Zerox.Lsai/blob/master/RESULTS.md).

### LSAI vs Grep (text search)

| Language | Engine | Symbols | Search Savings | Usages Savings |
|----------|--------|:-------:|:--------------:|:--------------:|
| **C#** | Roslyn (native) | 10 | **92.9%** | **92.9%** |
| **Python** | Pyright (LSP) | 10 | **63.5%** | **78.6%** |
| **TypeScript** | tsserver (LSP) | 8 | **91.6%** | **85.5%** |
| **JavaScript** | tsserver (LSP) | 10 | **99.3%** | **98.3%** |
| **Java** | jdtls (LSP) | 8 | **55.8%** | **68.6%** |
| **Weighted Average** | | **46** | **81.2%** | **86.9%** |

### LSAI vs Raw LSP JSON

Even vs semantically correct Raw LSP JSON, LSAI's compact formats save:

| Language | Search Savings | Usages Savings |
|----------|:--------------:|:--------------:|
| **Python** | **82.8%** | **85.5%** |
| **TypeScript** | **85.0%** | **82.2%** |
| **Java** | **40.1%** | **31.9%** |
| **Weighted Avg** | **69.1%** | **75.0%** |

### What Actually Matters

| Factor | Impact | Evidence |
|--------|--------|----------|
| **Data completeness** | HIGH | Outline with return types + params eliminates follow-up Reads |
| **Relative paths** | HIGH | Saves ~80 chars/line, ~400 tokens per response |
| **Code context in usages** | MEDIUM | Shows HOW a symbol is used, not just WHERE |
| **Composite queries** | HIGH | `impact` = 5-8 LSP calls in one |
| **Summary counts** | LOW | `[5 usages in 3 files]` — quick assessment |
| **Output format style** | NEGLIGIBLE | Any compact format works with zero LLM friction |

---

## Tool Definitions

### Naming Convention

Semantic tools use short names: `search`, `info`, `usages`, `callers`, `callees`, `hierarchy`, `impact`, `rename`, `diagnostics`, `outline`, `deps`.

MCP tool names are prefixed with `lsai_`: `lsai_search`, `lsai_info`, etc.

### Common Parameters

All tools that reference symbols accept either:
- `symbol` — name or fully qualified name
- OR `file` + `line` + `col` — location-based lookup

All tools accept:
- `outputFormat` — optional, selects formatter (implementation-specific)

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

**Example output (CompactText):**
```
Contracts/IDocumentService.cs:8 interface IDocumentService
Services/DocumentService.cs:14 class DocumentService
[4 symbols]
```

**Example output (GrepOutput):**
```
Contracts/IDocumentService.cs:8:1: interface IDocumentService
Services/DocumentService.cs:14:1: class DocumentService
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
| `members` | 2 | Member count summary (e.g., "3 properties, 7 methods, 2 fields") |
| `constructors` | 2 | Constructor signatures |
| `return` | 1 | Return type (for methods) |
| `params` | 1 | Parameter list (for methods) |
| `containing` | 1 | Containing type (for members) |
| `genericConstraints` | 3 | Generic type constraints |
| `attributes` | 3 | Applied attributes |

**Example output (CompactText):**
```
Services/DocumentService.cs:14 class DocumentService
  namespace: EFine.Services.Domain
  modifiers: public sealed
  base: BaseService
  implements: IDocumentService, IDisposable
  members: 12 (3 properties, 7 methods, 2 fields)
  constructors: .ctor(IRepository, IValidator, ILogger)
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
| `line` | yes | Line number |
| `context` | yes | The actual code line at the usage site |

Results MUST be grouped by file. Response MUST end with summary: `[N usages in N files]`.

**Example output (CompactText):**
```
Controllers/DocController.cs
  :28 private readonly IDocumentService _service;
  :45 public DocController(IDocumentService service)
Services/DocumentService.cs
  :14 public class DocumentService : IDocumentService
[5 usages in 3 files]
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
| `method` | yes | Caller method name (`Type.Method`) |
| `file` | yes | Relative file path |
| `line` | yes | Line number |
| `depth` | yes | Call depth level (1 = direct) |

When `depth > 1`, results MUST be organized by depth level. Indirect callers reference which direct caller they chain from.

**Example output (CompactText):**
```
depth 1:
  DocController.HandleRequest  Controllers/DocController.cs:52
  BatchJob.RunBatch  Jobs/BatchJob.cs:88

depth 2:
  DocController.HandleRequest
    <- ApiMiddleware.Invoke  Middleware/ApiMiddleware.cs:23
  BatchJob.RunBatch
    <- Scheduler.Execute  Jobs/Scheduler.cs:67
[2 direct, 2 indirect]
```

**Tier:** 2 (extended)

---

### Tool 5: `callees`

What does this method call? Walks the call graph downward.

**Input:** Same as `callers`.

**Data Contract:** Same structure as `callers`, but direction is downward. Uses `->` instead of `<-` for depth > 1 chains.

**Example output (CompactText):**
```
_validator.Validate  Services/Validator.cs:22
_repository.GetById  Data/Repository.cs:45
_repository.Save  Data/Repository.cs:78
[3 calls]
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

Each type in the tree includes `name`, `file`, `line`.

**Example output (CompactText):**
```
up:
  BaseService  Services/BaseService.cs:8
    -> DocumentService  Services/DocumentService.cs:14
implements:
  IDocumentService  Contracts/IDocumentService.cs:8
  IDisposable  (external)
down: (sealed)
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
| `directUsages` | 1 | Count and file count |
| `callerChain` | 2 | Callers at each depth level |
| `publicApiPaths` | 3 | Entry points reachable from this symbol |
| `affectedTests` | 3 | Test methods that exercise this symbol |
| `risk` | 1 | `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `reason` | 1 | Human-readable risk explanation |

**Example output (Tier 3 — full):**
```
direct usages: 5 in 3 files
caller chain depth 1: DocController.HandleRequest, BatchJob.RunBatch
caller chain depth 2: +2 (ApiMiddleware, Scheduler)
public API paths: 1 (DocController)
tests: TestProcess, TestBatchRun
risk: HIGH — 1 public API entry point, 2 direct callers, 2 tests
```

**Example output (Tier 1 — degraded):**
```
direct usages: 5 in 3 files
caller chain: (not available)
tests: (not available)
risk: UNKNOWN — only direct usages available
```

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
| `changes` | yes | List of `{file, line, oldText, newText}` |
| `changeCount` | yes | Total number of changes |
| `fileCount` | yes | Number of files affected |
| `untouched` | no | Similar symbols NOT renamed (different symbol, string literals, etc.) |

**Data Contract (applied):**

| Field | Required | Description |
|-------|----------|-------------|
| `applied` | yes | `true` |
| `changeCount` | yes | Changes made |
| `fileCount` | yes | Files modified |
| `compilationResult` | no | Post-rename compilation status |

**Example output (preview):**
```
14 changes in 6 files:
  Contracts/IDocumentService.cs:12 ProcessDocument -> HandleDocument
  Services/DocumentService.cs:28 ProcessDocument -> HandleDocument
untouched:
  ProcessDocumentAsync (different symbol)
  "ProcessDocument" in string at Services/DocumentService.cs:30
[preview — not applied]
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

**Example output (CompilerOutput):**
```
Services/DocumentService.cs(32,5): error CS0103: '_repo' does not exist in the current context
Config/Startup.cs(44,12): error CS0121: ambiguous call between 'Configure(A)' and 'Configure(B)'
[2 errors]
```

This format is identical to `dotnet build` / `gcc` / `rustc` / `tsc` / `pyright` compiler output.

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
| `name` | yes | Member name |
| `kind` | yes | `class`, `struct`, `interface`, `method`, `property`, `field`, `event`, `constructor`, `function`, `variable`, `enum`, `enumMember` |
| `signature` | yes | **Full declaration signature** including return type, parameters, property accessors |
| `accessibility` | yes | `public`, `internal`, `protected`, `private` |
| `line` | yes | Line number |
| `isStatic` | no | Static modifier |
| `isAbstract` | no | Abstract modifier |
| `isVirtual` | no | Virtual modifier |

Results MUST include the containing type declaration with base class and interfaces. Members MUST be nested under their containing type. Response MUST end with summary count.

**The full signature is the most important field.** Without it, the AI must perform a follow-up file Read to understand what a method does — negating the efficiency gains of the tool. A method name without return type and parameters is nearly useless for code reasoning.

**Example output (CompactText):**
```
namespace EFine.Services.Domain
  public sealed class DocumentService : BaseService, IDocumentService [14]
    IRepository _repository [16]
    .ctor(IRepository, IValidator, ILogger) [20]
    async Task<DocumentDto> Process(int id, CancellationToken ct) [28]
    bool Validate(Document doc) [45]
    void Dispose() [62]
    bool IsInitialized { get; } [68]
[1 class, 6 members]
```

**Example output (LanguageSyntax):**
```csharp
public sealed class DocumentService : BaseService, IDocumentService  // :14
{
    private IRepository _repository;  // :16
    internal DocumentService(IRepository, IValidator, ILogger) { }  // :20
    public async Task<DocumentDto> Process(int id, CancellationToken ct) { }  // :28
    public bool Validate(Document doc) { }  // :45
    public void Dispose() { }  // :62
    public bool IsInitialized { get; }  // :68
}
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
| `files` | no | Source files grouped by directory (when `includeFiles=true`) |

This tool shows **project-to-project dependencies**, NOT individual NuGet packages or assembly references.

**Example output (CompactText):**
```
EFine.sln (5 projects)
  EFine.Api -> EFine.Services, EFine.Contracts
  EFine.Services -> EFine.Data, EFine.Contracts
  EFine.Data -> EFine.Contracts
  EFine.Contracts -> (none)
  EFine.Tests -> EFine.Api, EFine.Services
```

**Tier:** 1 (basic graph) / 2 (with file listing)

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
| `protocol` | yes | Protocol version (e.g., `LSAI/1.2`) |
| `plugins` | yes | List of loaded plugins with tier, languages, tool count |
| `workspaces` | yes | Open workspaces with language, path, file count |

**Example output:**
```
lsai v0.5.0
protocol: LSAI/1.2
plugins:
  roslyn-csharp  tier 3  [C#]          11 tools
  lsp-python     tier 2  [Python]       9 tools
  lsp-typescript tier 2  [TypeScript]   9 tools
  lsp-javascript tier 2  [JavaScript]   9 tools
  lsp-java       tier 2  [Java]         9 tools
workspaces:
  ws-abc123 | C#         | EFine.sln        | 142 files
  ws-def456 | Python     | /path/to/project | 23 files
```

---

## Output Format Profiles

The protocol defines **what data** each tool returns. **How** that data is serialized is an implementation choice. Implementations SHOULD support at least one format, and MAY support multiple.

### Profiles

| Profile | Style | Use Case |
|---------|-------|----------|
| **CompactText** | Minimal text, no context snippets | Default. Lowest token cost |
| **CompactTextVerbose** | Compact text with source line context | When code context needed alongside results |
| **TurboCompact** | Ultra-compressed: names, positions, counts only | Maximum token savings for bulk queries |
| **GrepOutput** | `file:line:col: match` (grep-like) | Familiar format for AI agents used to grep |
| **CompilerOutput** | `File.cs(line,col): severity CODE: msg` | Compiler-native format (dotnet/gcc/tsc) |
| **LanguageSyntax** | Language-specific syntax with structure | Shows results as valid language code with line annotations |

### Format Requirements

All profiles MUST:
- Use **relative paths** (never absolute)
- Include **summary counts** at end of response
- Include **full signatures** in outline (return types, parameters)
- Include **code context** in usages (for verbose profiles)
- Use **full words** for kinds (`class`, not `C`; `error`, not `E`)

### Why Format Doesn't Matter

Empirical testing across 6+ months with Claude Opus 3.5/4.6 and OpenAI Codex:

- Pipe-delimited (`method|internal|16,48`) — understood instantly, zero friction
- Colon-separated (`file:16 method`) — understood instantly, zero friction
- Compiler-native (`file:32 error CS0103: msg`) — understood instantly, zero friction
- Language syntax (`public void Method() { } // :28`) — understood instantly, zero friction

LLMs decode any compact format without legends or explanations. The format is a commodity. **Data richness and relative paths are where the real savings are.**

### What NOT to Do

- Do NOT return raw LSP JSON — 69-85% token waste (measured)
- Do NOT use absolute paths — wastes ~80 chars/line
- Do NOT omit return types from outlines — forces follow-up file Reads
- Do NOT omit code context from usages — forces follow-up file Reads
- Do NOT abbreviate kinds (`C`, `M`, `E`) — full words cost ~4 extra chars but eliminate ambiguity

---

## Multi-Language Plugin Architecture

LSAI supports multiple languages through a plugin system. Each plugin implements the `ILsaiPlugin` interface and declares its capabilities.

### Plugin Types

| Type | Description | Tier | Example |
|------|-------------|:----:|---------|
| **Native** | Direct compiler API access | 3 | Roslyn for C# |
| **LSP Bridge** | Wraps an LSP server via StreamJsonRpc | 2 | Pyright, tsserver, jdtls |

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

### Adding a New Language

1. Create an LSP language config JSON (server command, args, file extensions)
2. Deploy language server binary
3. Register in plugin directory
4. All 11 semantic tools work automatically through the LSP bridge

---

## Capability Tiers

| Tier | Tools | Minimum Backend |
|------|-------|-----------------|
| **1** | search, info (basic), usages, rename, diagnostics, outline, deps | Any LSP server |
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
    "outline": true, "deps": true
  },
  "backend": "roslyn-native"
}
```

```json
{
  "id": "lsp-python",
  "languages": ["python"],
  "extensions": [".py"],
  "tier": 2,
  "tools": {
    "search": true, "info": "basic", "usages": true,
    "callers": true, "callees": true, "hierarchy": false,
    "impact": "degraded", "rename": true, "diagnostics": true,
    "outline": true, "deps": true
  },
  "backend": "pyright-lsp"
}
```

---

## Transport

LSAI is an MCP server. It defines tool semantics and data contracts, not transport.

```
AI Agent ──MCP──> LSAI Server ──native/lsp──> Compiler / Language Server
                  (15 tools)                   (Roslyn, Pyright, tsserver, jdtls)
```

---

## Changes from v1.1

| What | v1.1 | v1.2 | Why |
|------|------|------|-----|
| **Multi-language** | Not addressed | 5 languages, plugin architecture | Core evolution |
| **Output formats** | 2 profiles (CompactJson, CompactText) | 6 profiles (CompactText, CompactTextVerbose, TurboCompact, GrepOutput, CompilerOutput, LanguageSyntax) | Real implementation needs |
| **Benchmarks** | VS-MCP C# only | 5 languages, 46 symbols, vs grep AND vs raw LSP | Empirical validation |
| **Workspace tools** | Not specified | workspace_open, workspace_list, workspace_close | Required for multi-language |
| **LSP limitations** | Not documented | Per-language limitation table | Transparency |
| **Plugin manifest** | Single example | Multiple examples (native + LSP bridge) | Multi-language clarity |
| **Outline kinds** | 8 kinds | 12 kinds (added `function`, `variable`, `enum`, `enumMember`) | Multi-language symbols |
| **Protocol version** | LSAI/1.1 | LSAI/1.2 | Semantic versioning |

---

## Versioning

- **LSAI/1.0** — Initial spec. 11 tools, 3 tiers, pipe-delimited format.
- **LSAI/1.1** — Format-agnostic, data contracts, enriched fields.
- **LSAI/1.2** — This document. Multi-language, 6 output formats, empirical benchmarks, workspace tools.
- Future versions add tools, never remove. Tiers may be extended.

---

## License

Protocol specification licensed under CC BY-NC 4.0. Free for study, research, and non-commercial use. Commercial implementations require a separate license — contact ladislav.sopko@gmail.com

---

*LSAI — Language Server for AI. Compiler intelligence, optimized for LLMs.*

*Protocol designed by Ladislav Sopko, 0ics srl, Bologna, Italy.*
*February 2026*
