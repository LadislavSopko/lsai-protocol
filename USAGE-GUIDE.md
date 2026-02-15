# LSAI Usage Guide for AI Agents

> This guide explains how to use LSAI tools effectively. It is written for AI coding agents (Claude, GPT, Codex, etc.) that have LSAI available as an MCP server.

---

## Quick Start

1. **Open a workspace** — before any analysis, open the project:
   ```
   lsai_workspace_open(path="/path/to/project")
   → returns workspaceId: "ws-abc123"
   ```

2. **Search for symbols** — find what you're looking for:
   ```
   lsai_search(workspaceId="ws-abc123", query="UserService")
   → Services/UserService.cs:14 class UserService
   ```

3. **Inspect implementation** — get the source code:
   ```
   lsai_source(workspaceId="ws-abc123", symbolName="GetById")
   → Services/UserService.cs:18 method GetById [public User? GetById(int id)]
     public User? GetById(int id) => _users.FirstOrDefault(u => u.Id == id);
   ```

4. **Close when done** — free server resources:
   ```
   lsai_workspace_close(workspaceId="ws-abc123")
   ```

---

## Two Use Cases

LSAI serves two distinct purposes. Understanding which one applies determines how you use the tools.

### Use Case 1: Local Project Development

You have file access to the project. LSAI provides **semantic intelligence** that file reads cannot:

- **Search** — find symbols by name across the entire project (faster and more precise than grep)
- **Usages** — who references this symbol? (semantic, not text matching)
- **Callers/Callees** — call graph navigation
- **Hierarchy** — inheritance chains, interface implementations
- **Impact** — what breaks if I change this?
- **Outline** — document structure with full signatures
- **Diagnostics** — compiler errors without running a build

In this mode, prefer `Read` for viewing source code — it's cheaper than `lsai_source` when you already know the file path. Use LSAI for semantic queries that text search cannot answer.

### Use Case 2: Library Documentation

You do NOT have file access. LSAI is your only window into the library's source code:

- **Search** → find symbols in the library
- **Source** → read their implementation (this is the ONLY way to see code)
- **Outline** → understand file/type structure
- **Info** → get symbol details, signatures, documentation
- **Hierarchy** → understand type relationships

In this mode, `lsai_source` is essential — without it, you cannot inspect implementations at all.

---

## Tool Selection Guide

| I want to... | Use | Not |
|--------------|-----|-----|
| Find a symbol by name | `search` | `grep` (misses types, hits comments) |
| See a method's implementation | `source` | `Read` entire file (wastes tokens on imports/comments) |
| See who uses a symbol | `usages` | `grep symbolName` (includes definitions, comments, strings) |
| See who calls a method | `callers` | Not possible with grep |
| See what a method calls | `callees` | Not possible with grep |
| Understand inheritance | `hierarchy` | Regex on `class X(Y)` — misses interfaces, derived types |
| See all members of a class | `outline` | `Read` file (includes implementation noise) |
| Get full symbol details | `info` | `hover` + `definition` + multiple LSP calls |
| Check for compiler errors | `diagnostics` | Build the project (slower, more noise) |
| Assess change impact | `impact` | Multiple queries + manual analysis |
| Rename safely | `rename` | Find-and-replace (misses cross-file, hits strings) |
| See project dependencies | `deps` | Parse XML/JSON config files manually |

---

## Common Workflows

### Workflow 1: Understand a Symbol

```
search("UserService")          → find it
info("UserService")            → full details: modifiers, base types, member count
outline(typeName="UserService") → all members with signatures
source("GetById")              → specific method body
```

### Workflow 2: Before Making a Change

```
search("ProcessDocument")      → find the symbol
usages("ProcessDocument")      → who references it?
callers("ProcessDocument")     → who calls it?
impact("ProcessDocument")      → full risk assessment
```

### Workflow 3: Learn a Library API

```
search("*Service")             → find service classes
outline(typeName="UserService") → see public API (methods + signatures)
source("CreateUser")           → how does it work?
hierarchy("UserService")       → what does it extend/implement?
```

### Workflow 4: Debug a Compilation Error

```
diagnostics()                  → get all errors
info("_repo")                  → does this symbol exist? What type is it?
usages("_repo")                → where is it used?
```

---

## Token Efficiency Tips

### 1. Search Before Reading

**Bad:** Read `UserService.cs` (593 chars) to find `GetById`
**Good:** `source("GetById")` → 138 chars (77% savings)

### 2. Use Outline Instead of File Read for Structure

**Bad:** Read entire file to understand its structure
**Good:** `outline(file="Services/UserService.cs")` → member list with signatures

### 3. Use Source for Focused Inspection

`source` extracts only the requested symbol body. Token savings by symbol type:

| Target | Typical Savings vs File Read |
|--------|:----------------------------:|
| Method/Function | 77-93% |
| Enum | 90-94% |
| Small class | 50-66% |
| Single-class file | ~0% (file read is equivalent) |

**Rule of thumb:** If you need ONE symbol from a multi-symbol file, use `source`. If you need the entire file, use `Read`.

### 4. Use CompactText Format (Default)

All LSAI tools default to CompactText — the lowest token format. Only switch to a different format when you need:
- `CompactTextVerbose` — code context lines alongside results
- `LanguageSyntax` — language-native output (useful for presenting code to users)
- `GrepOutput` — familiar `file:line:col: match` format

### 5. Chain Tools Efficiently

Don't call tools you don't need. A typical efficient chain:

```
search → source  (when you need implementation)
search → info    (when you need API surface)
search → usages  (when you need references)
```

Avoid: `search → info → outline → source → usages` all at once. Start with what you need, add tools only if the first result is insufficient.

---

## Output Formats

All tools accept an optional `outputFormat` parameter:

| Format | When to Use |
|--------|-------------|
| **CompactText** (default) | General use. Lowest token cost |
| **CompactTextVerbose** | When you need source line context alongside results |
| **TurboCompact** | Bulk queries where you only need names and positions |
| **GrepOutput** | If your workflow expects `file:line:col: match` format |
| **CompilerOutput** | Diagnostics in compiler-native format |
| **LanguageSyntax** | When presenting code structure to users |

---

## Multi-Language Notes

LSAI works across 5 languages through a single endpoint. Language-specific behaviors:

| Language | Notes |
|----------|-------|
| **C#** | Full Tier 3 support via Roslyn. All 12 tools available |
| **Python** | Tier 2 via Pyright. No `hierarchy` (Pyright limitation) |
| **TypeScript** | Tier 2 via tsserver. Full Tier 2 support |
| **JavaScript** | Tier 2 via tsserver. `search` limited for CommonJS `.js` files |
| **Java** | Tier 2 via jdtls. `search` does not return fields |

The workspace automatically detects the language from file extensions. Use `language` override in `workspace_open` only when auto-detection fails.

---

## What LSAI Does NOT Do

LSAI provides **read-only semantic analysis**. It does not:

- Read or write files (use your file tools)
- Build or compile projects (use `dotnet build`, `npm run build`, etc.)
- Run tests (use your test runner)
- Execute code
- Manage git or version control

LSAI answers the question: **"What does the compiler know about this code?"** Everything else is outside its scope.

---

*LSAI — Language Server for AI. Compiler intelligence, optimized for LLMs.*
