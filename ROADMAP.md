# Zerox.Lsai — Roadmap & Vision

> Document for project context. Last updated: 2026-02-15

---

## Current State (v0.5 — Multi-Language Production)

- 15 MCP tools (11 semantic + 3 workspace + 1 meta)
- 5 languages: C# (Roslyn), Python (Pyright), TypeScript (tsserver), JavaScript (tsserver), Java (jdtls)
- 465 unit tests + 15 E2E integration tests across 5 languages
- 6 output format profiles (CompactText, CompactTextVerbose, TurboCompact, GrepOutput, CompilerOutput, LanguageSyntax)
- Plugin architecture: Roslyn native (Tier 3) + LSP bridge (Tier 2) for all other languages
- Streamable HTTP transport via MCP SDK
- Docker packaging: 1.4 GB image with all 5 languages
- Measured token savings: **81.2% search, 86.9% usages** vs grep (46 symbols, 5 languages)
- Measured token savings: **69-75%** vs raw LSP JSON (same semantic data, compact format)

---

## Completed Phases

### Phase 5: Multi-Language Support (COMPLETE — Feb 2026)

All 5 target languages are fully integrated and tested:

| Language | Engine | Plugin Type | Tier | E2E Tests |
|----------|--------|-------------|:----:|:---------:|
| **C#** | Roslyn | Native compiler API | 3 | 5/5 PASS |
| **Python** | Pyright | LSP bridge (StreamJsonRpc) | 2 | 5/5 PASS |
| **TypeScript** | tsserver | LSP bridge (StreamJsonRpc) | 2 | 4/4 PASS |
| **JavaScript** | tsserver | LSP bridge (StreamJsonRpc) | 2 | 1/1 PASS |
| **Java** | Eclipse JDT (jdtls) | LSP bridge (StreamJsonRpc) | 2 | 5/5 PASS |

**Key technical decisions:**
- OmniSharp removed, replaced with StreamJsonRpc + own LSP types (LspTypes.cs, 35+ types)
- Deterministic workspace readiness: AsyncReady config per language (Java uses background init, others block)
- LSP capabilities handshake: explicit `workspace.symbol` declaration required by Pyright

**Token savings per language (Search, vs Grep):**
- C#: 92.9% | Python: 63.5% | TypeScript: 91.6% | JavaScript: 99.3% | Java: 55.8%

### Phase 8: Docker Packaging (COMPLETE — Feb 2026)

- 3-stage Dockerfile: jdtls download → .NET build → runtime (aspnet:10.0 + Node.js 22 + JDK 21)
- Single 1.4 GB image with all 5 languages
- All 15 MCP tools verified in Docker
- Compose files for both build-from-source and run-from-image

---

## Phase 1: External Git Repository Loading (Next)

### The Idea

Allow AI agents to load any external git repository as a **read-only semantic workspace**. The AI gets full compiler-level intelligence (outline, hierarchy, callers, impact) on libraries it uses — without reading thousands of source files.

### Why This Matters

Today, when an AI agent needs to understand a library (e.g., MediatR, Newtonsoft.Json, EF Core):
- It reads source files one by one (slow, expensive, incomplete)
- Or it relies on documentation (often outdated or incomplete)
- Or it guesses from NuGet package signatures (no implementation detail)

With external repo loading:
```
lsai_workspace_open(gitUrl: "https://github.com/jbogard/MediatR", readOnly: true)
```
→ Clone, restore, compile, load into Roslyn → AI has full semantic access.

### Use Cases

1. **Library understanding**: "How do I implement a custom `JsonConverter`?" → `hierarchy("JsonConverter")` shows all implementations in the Newtonsoft repo
2. **API discovery**: "What methods does `IMediator` expose?" → `outline("IMediator")` with full signatures
3. **Migration analysis**: "I'm upgrading from v12 to v13, what changed?" → compare outlines between tags
4. **Cross-repo impact**: "If this library method changes, what breaks in MY code?" → `impact` across workspaces
5. **OSS contribution**: "I want to add a feature to this library, where do I start?" → `callers`, `callees`, `hierarchy` to understand code structure

### Implementation Plan

```
lsai_workspace_open(
    gitUrl: "https://github.com/...",   // remote git URL
    ref: "main",                         // branch, tag, or commit (default: main)
    readOnly: true                       // disables rename, enables only query tools
)
```

Steps:
1. Clone repo to temp directory (or local cache path)
2. `dotnet restore` (for .NET projects)
3. Load into Roslyn via MSBuildWorkspace
4. Register as read-only workspace session
5. All query tools work: search, info, usages, callers, callees, hierarchy, impact, outline, deps
6. Disabled tools: rename (read-only workspace)

### Technical Considerations

- Auto-detect solution/project file in cloned repo
- Support `ref` parameter for specific branches/tags/commits
- Timeout for clone + build (configurable, default 5 min)
- Disk space management: configurable cache directory + max size
- Error handling: repos that don't build, repos without .NET projects

---

## Phase 2: Semantic Index Cache

### The Idea

Don't recompile every time. Serialize the semantic index (symbols, call graph, hierarchy, cross-references) to disk. Next time you open the same repo at the same commit, load from cache in milliseconds instead of compiling for minutes.

### What Gets Cached Per Repository

| Data | Approx Size | Enables |
|------|-------------|---------|
| Symbol index (names, kinds, signatures, locations) | ~1-5 MB | `search`, `info`, `outline` instant |
| Call graph (caller/callee relationships) | ~2-10 MB | `callers`, `callees` without recompiling |
| Hierarchy tree (base types, interfaces, derived) | ~0.5-2 MB | `hierarchy` instant |
| Cross-reference map (usage locations + context) | ~5-20 MB | `usages`, `impact` ready |
| Diagnostic snapshot | ~0.1-1 MB | `diagnostics` instant |

**Total: ~10-40 MB per repo** (estimated for medium-large repos)

### Cache Strategy

```
Cache key: {gitUrl}:{commitHash}
Cache path: ~/.zerox-lsai/cache/{repoHash}/{commitHash}/

Cache invalidation:
- Same commit hash → cache hit, load instantly
- Different commit → recompile, update cache
- Manual: lsai_workspace_open(..., forceRecompile: true)
```

### Cache Levels

```
Level 1: In-memory (current session)
    - Already loaded workspace
    - Fastest, lost on server restart

Level 2: Local disk cache
    - Serialized semantic index
    - Survives server restarts
    - Load in ~100ms instead of 30-120s compile time

Level 3: Shared/cloud cache (future — see Phase 3)
    - Pre-built indexes for popular repos
    - Download ~20MB instead of clone+compile
```

### Implementation Plan

1. Define serialization format for semantic index (protobuf or MessagePack)
2. Build `SemanticIndexSerializer` that extracts and persists:
   - All symbols with full metadata
   - Call graph edges
   - Hierarchy relationships
   - Usage locations with code context
3. Build `CachedPlugin` wrapper that checks cache before delegating to real plugin
4. Cache management: LRU eviction, configurable max size, `lsai_cache_list`, `lsai_cache_clear` tools

### API Extension

```
lsai_workspace_open(
    gitUrl: "...",
    ref: "v13.0.1",
    readOnly: true,
    useCache: true          // default: true
)

# New tools:
lsai_cache_list()           // show cached repos with sizes
lsai_cache_clear(repo?)     // clear cache for specific repo or all
```

---

## Phase 3: Shared Semantic Index (Cloud)

### The Vision

A **global semantic index of open source code**, optimized for AI consumption. Every popular library pre-indexed and ready to query.

### How It Works

```
Developer runs LSAI locally
    → Opens "https://github.com/dotnet/runtime"
    → Cache miss locally
    → Checks shared index: HIT
    → Downloads ~30MB semantic index (instead of cloning 2GB repo + 10 min compile)
    → Full semantic access in seconds
```

### The Network Effect

```
More users → more repos indexed → more cache hits → more value for everyone
```

This is a **moat** — a competitive advantage that grows over time. Every user who indexes a public repo contributes to the shared pool.

### What Gets Shared

Only for **public repositories**:
- Symbol index
- Call graph
- Hierarchy tree
- Cross-references
- NO source code (just semantic metadata)

Private repos: NEVER shared. Local cache only.

### Business Model

| Tier | What | Price |
|------|------|-------|
| **Free** | Local indexing, local cache only | $0 |
| **Pro** | Shared index access: top 1000 libraries pre-indexed (5 languages) | $29/mo |
| **Team** | Pro + private shared cache within organization | $99/mo |
| **Enterprise** | Self-hosted shared index server | Custom |

### Pre-Indexed Libraries (Initial Set)

**.NET**: dotnet/runtime, dotnet/aspnetcore, dotnet/efcore, Newtonsoft.Json, MediatR, AutoMapper, FluentValidation, Serilog, Polly, Dapper, MassTransit, xUnit, NSubstitute, FluentAssertions

**Python**: requests, fastapi, django, sqlalchemy, pytest, pydantic, flask, celery

**TypeScript/JavaScript**: react, next.js, express, prisma, zod, angular, vue, nestjs

**Java**: spring-boot, spring-framework, hibernate, jackson, junit5, mockito

**Future**: Rust (tokio, serde, actix-web), Go (gin, gorm, cobra)

---

## Phase 4: Cross-Workspace Intelligence (Future)

### The Idea

Query across YOUR code AND the libraries you use simultaneously.

```
"If MediatR changes IRequestHandler, what breaks in MY code?"
```

This requires two workspaces loaded:
1. Your project (read-write)
2. MediatR repo at specific version (read-only, from cache)

LSAI resolves cross-workspace type references and shows impact across boundaries.

### Use Cases

1. **Upgrade planning**: Load library at v12 and v13, diff the outlines, show impact on your code
2. **Dependency audit**: "Which of my classes inherit from library types?" → cross-workspace hierarchy
3. **Security**: "This library method has a CVE, am I calling it?" → cross-workspace callers
4. **Tech debt**: "How coupled is my code to this library?" → cross-workspace usage count

---

## Phase 6: Additional Languages (Future)

With the LSP bridge plugin architecture proven for 4 languages, adding new ones is mechanical:

| Priority | Language | Plugin Backend | Tier Target | Effort |
|----------|----------|---------------|:-----------:|--------|
| 1 | **Rust** | rust-analyzer | 2 | Medium |
| 2 | **Go** | gopls | 2 | Medium |
| 3 | **C/C++** | clangd | 2 | Medium |
| 4 | **Kotlin** | kotlin-language-server | 2 | Medium |

### Plugin Development Kit

Provide a template + test suite for community plugin development:
- `ILsaiPlugin` interface (already defined in Core)
- Reference test suite: "your plugin must pass these tests to be Tier N compliant"
- Plugin submission to registry

---

## Summary: The Evolution

```
DONE:       Multi-language semantic server (C#, Python, TS, JS, Java)
DONE:       Docker packaging, 6 output formats, 465+ tests
                    ↓
PHASE 1:    + Load any git repo as read-only workspace
                    ↓
PHASE 2:    + Cache semantic index → instant reload
                    ↓
PHASE 3:    + Shared cloud index → global semantic knowledge base
                    ↓
PHASE 4:    + Cross-workspace queries → "what breaks if library X changes?"
                    ↓
PHASE 6:    + More languages → Rust, Go, C++, Kotlin
```

Each phase builds on the previous one. Each phase adds value. Each phase is independently useful.

The end state: **a global semantic index of all code, optimized for AI agents**. The Google of code intelligence.

---

## IP & Licensing Reminder

- **LSAI Protocol spec**: CC BY-NC 4.0 (public, non-commercial)
- **Zerox.Lsai implementation**: Proprietary (all rights reserved)
- **Semantic index data**: Generated data, not copyrightable source code
- **Consider**: Provisional patent on composite queries, impact analysis, tier-based capability system, semantic index caching architecture

---

*Ladislav Sopko, 0ics srl, Bologna, Italy — February 2026*
