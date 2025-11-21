# Code Implementation Plan: Notion Integration Module

**Generated**: 2025-11-12
**Based on**: Phase 1 plan (`plan.md`) + Phase 2 documentation (README, skill, package.json)
**Status**: Ready for Phase 4 implementation

---

## Summary

Implement bidirectional Notion integration following the Intercom module pattern. Create a thin wrapper around Notion MCP tools that handles:
- **Export**: Convert markdoc documents to Notion pages (create/update)
- **Import**: Fetch Notion pages and convert to markdoc documents
- **Sync tracking**: Maintain sync state in document frontmatter
- **CLI & Skills**: Usable from both npm commands and Claude Code skills

**Philosophy**: Ruthless simplicity via hybrid approach - leverage MCP for API calls, add only essential orchestration and conversion logic.

---

## Current State Analysis

### What Exists and Works ✅

**File**: `src/lib/markdoc/notion-renderer.ts` (236 lines)
- **Purpose**: Converts markdoc AST to Notion-flavored markdown
- **Exports**: `renderMarkdocToNotion()`, `debugMarkdocTransform()`
- **Quality**: Clean, focused, well-tested converter
- **Status**: Keep and move to module

**File**: `src/cli/commands/convert-to-notion.ts` (96 lines)
- **Purpose**: CLI wrapper around notion-renderer
- **Status**: Keep as legacy/debug tool, will be superseded by publish-notion

**File**: `src/lib/schemas.ts` (250 lines)
- **Notion support**: Lines 47-117 define `NotionMetadataSchema`
- **Issue**: Too rigid - uses discriminated union requiring exact database IDs
- **Status**: Needs flexibility update

**File**: `src/modules/intercom/` (reference pattern)
- **Structure**: index.ts (orchestrator) + specialized managers + types.ts
- **Pattern**: `publishDocument()`, `fetchArticle()`, `syncDocument()` methods
- **Status**: Follow this pattern exactly

### What's Missing ❌

**Module files** (all new):
- `src/modules/notion/index.ts` - NotionIntegration class
- `src/modules/notion/content-converter.ts` - Bidirectional conversion
- `src/modules/notion/types.ts` - TypeScript interfaces
- `src/modules/notion/README.md` - Regeneration spec

**CLI commands** (all new):
- `src/cli/commands/publish-notion.ts` - Publish workflow
- `src/cli/commands/fetch-notion.ts` - Import workflow
- `src/cli/commands/sync-notion.ts` - Sync workflow

**Conversion logic**:
- Reverse converter: Notion markdown → markdoc (doesn't exist)

---

## Files to Change

### 1. src/lib/schemas.ts

**Current State** (lines 47-117):
```typescript
// Too rigid - discriminated union with literal database IDs
export const NotionMetadataSchema = z.discriminatedUnion('databaseId', [
  NotionHelpCenter30Schema,  // Requires exact literals
  NotionEdenHelpCenterSchema,
]);
```

**Required Changes**:
Make schema flexible to support both:
- **Help-center docs**: Require `databaseId` and `dataSourceId`
- **Core docs (internal)**: Don't require database IDs (Notion internal articles)

**Specific Modifications**:
```typescript
// Replace discriminatedUnion with flexible schema
export const NotionMetadataSchema = z.object({
  pageId: z.string().optional(),  // Populated after publish
  databaseId: z.string().optional(),  // Only for help-center docs
  dataSourceId: z.string().optional(),  // Only for help-center docs
  lastSynced: z.string().datetime().optional(),
  syncStatus: z.enum(['success', 'failed', 'pending']).optional(),
  workspace: z.string().default('Dubsado'),
  properties: z.object({
    status: z.string().optional(),  // Notion property value
    type: z.enum(['External', 'Internal']).optional(),
  }).optional(),
});
```

**Dependencies**: None - can be done early
**Agent Suggestion**: modular-builder
**Estimated lines**: ~30 lines changed

---

### 2. src/modules/notion/types.ts (NEW)

**Purpose**: TypeScript type definitions for Notion integration
**Exports**: Module interfaces and types
**Estimated lines**: ~80 lines

**Specific Content**:
```typescript
import type { DocFrontmatter } from '../../lib/schemas.js';

// Result of publishing a document
export interface PublishResult {
  pageId: string;
  databaseId?: string;
  dataSourceId?: string;
  status: string;
  collectionId?: string;
}

// Fetched article with frontmatter + content
export interface FetchedPage {
  frontmatter: DocFrontmatter;
  content: string;  // Markdoc content
}

// Options for fetching multiple pages
export interface FetchPagesOptions {
  databaseId?: string;
  filters?: Record<string, any>;
  perPage?: number;
  page?: number;
}

// Sync status enum
export type SyncStatus = 'synced' | 'needs_sync' | 'conflict';
```

**Dependencies**: schemas.ts (imports DocFrontmatter)
**Agent Suggestion**: modular-builder
**Test Strategy**: Type-only file, validated by TypeScript compiler

---

### 3. src/modules/notion/content-converter.ts (NEW)

**Purpose**: Bidirectional markdoc ↔ Notion markdown conversion
**Exports**: `convertMarkdocToNotion()`, `convertNotionToMarkdoc()`
**Estimated lines**: ~300 lines

**Specific Modifications**:

**Move existing function**:
```typescript
// Extract from notion-renderer.ts:202-220
export function convertMarkdocToNotion(content: string): string {
  // MOVE: All logic from renderMarkdocToNotion()
  // Keep existing implementation - it works well
}
```

**Add new reverse converter**:
```typescript
export function convertNotionToMarkdoc(notionMarkdown: string): string {
  // Convert Notion-flavored markdown to markdoc
  // Handle:
  // - <callout> tags → {% callout %} tags
  // - <image source="..."> → standard markdown images
  // - <table_of_contents/> → {% toc /%}
  // - Standard markdown stays as-is
}
```

**Dependencies**: None (pure functions)
**Agent Suggestion**: modular-builder
**Test Strategy**: Unit tests for both directions, roundtrip tests
**Estimated tests**: ~15 test cases

---

### 4. src/modules/notion/index.ts (NEW - Orchestrator)

**Purpose**: Main NotionIntegration class - orchestrates MCP calls with business logic
**Exports**: `NotionIntegration` class
**Estimated lines**: ~350 lines

**Class Structure** (follow Intercom pattern):
```typescript
export class NotionIntegration {
  private initialized = false;

  constructor() {
    // No constructor args needed - uses MCP tools directly
  }

  // Initialize (validate MCP available)
  async initialize(): Promise<void>

  // Export operations
  async publishDocument(doc: DocFrontmatter, content: string): Promise<DocFrontmatter>
  async updateDocument(doc: DocFrontmatter, content: string): Promise<DocFrontmatter>

  // Import operations
  async fetchPage(pageId: string): Promise<FetchedPage>
  async fetchPages(options: FetchPagesOptions): Promise<FetchedPage[]>

  // Sync operations
  async syncDocument(doc: DocFrontmatter, content: string): Promise<DocFrontmatter>
  async syncStatus(pageId: string): Promise<SyncStatus>

  // Helper methods (private)
  private ensureInitialized(): Promise<void>
  private validateDocument(doc: DocFrontmatter): void
  private updateFrontmatter(doc: DocFrontmatter, result: PublishResult): DocFrontmatter
}
```

**Implementation Details**:

**publishDocument()**:
- Validate frontmatter (help-center needs databaseId/dataSourceId)
- Convert content: `convertMarkdocToNotion(content)`
- Check if doc has pageId (update vs create)
- Call MCP: `mcp__notion__notion-create-pages` or `mcp__notion__notion-update-page`
- Update frontmatter with pageId, lastSynced, syncStatus
- Return updated frontmatter

**fetchPage()**:
- Call MCP: `mcp__notion__notion-fetch` with pageId
- Parse response (Notion-flavored markdown)
- Convert content: `convertNotionToMarkdoc(notionMarkdown)`
- Build frontmatter from Notion properties
- Return FetchedPage

**syncDocument()**:
- If pageId exists: update
- If no pageId: publish
- Return updated frontmatter

**Dependencies**:
- types.ts (PublishResult, FetchedPage)
- content-converter.ts (conversion functions)
- schemas.ts (DocFrontmatter, validation)

**Agent Suggestion**: modular-builder
**Test Strategy**: Integration tests with mocked MCP responses
**Estimated tests**: ~20 test cases

---

### 5. src/modules/notion/README.md (NEW - Module Spec)

**Purpose**: Module regeneration specification (the "blueprint")
**Estimated lines**: ~200 lines markdown

**Required Sections**:
```markdown
# Notion Integration Module

## Purpose
Bidirectional sync between markdoc documents and Notion pages using MCP.

## Public API

### NotionIntegration Class
[Document all public methods with signatures, parameters, return types]

### Content Conversion Functions
[Document convertMarkdocToNotion() and convertNotionToMarkdoc()]

## Architecture
[Explain the hybrid approach, MCP usage, frontmatter management]

## Usage Examples
[CLI and programmatic usage examples]

## Dependencies
- Notion MCP tools (mcp__notion__*)
- DocFrontmatter schema
- Markdoc parser

## Testing
[How to test the module]

## Regeneration
This module can be regenerated from this specification.
```

**Dependencies**: None (documentation)
**Agent Suggestion**: modular-builder
**Test Strategy**: Manual review for completeness

---

### 6. src/cli/commands/publish-notion.ts (NEW)

**Purpose**: CLI command to publish documents to Notion
**Pattern**: Follow `publish-intercom.ts` exactly
**Estimated lines**: ~110 lines

**Implementation**:
```typescript
#!/usr/bin/env node
import { resolve } from 'path';
import { readFile, writeFile } from 'fs/promises';
import { config } from 'dotenv';
import { NotionIntegration } from '../../modules/notion/index.js';
import { parseFrontmatter, stringifyDocument } from '../../lib/frontmatter.js';
import { DocFrontmatterSchema } from '../../lib/schemas.js';

config({ path: '.env.local' });
config();

const docPath = process.argv[2];
if (!docPath) {
  console.error('Usage: npm run publish-notion -- <doc-path>');
  process.exit(1);
}

async function main() {
  const absolutePath = resolve(process.cwd(), docPath);
  console.log('📤 Publishing to Notion...');

  // 1. Read and parse
  const content = await readFile(absolutePath, 'utf-8');
  const { frontmatter, markdown } = parseFrontmatter(content);

  // 2. Validate schema
  const validationResult = DocFrontmatterSchema.safeParse(frontmatter);
  // ... handle validation errors

  // 3. Initialize module
  const notion = new NotionIntegration();
  await notion.initialize();

  // 4. Publish
  const updatedFrontmatter = await notion.publishDocument(frontmatter, markdown);

  // 5. Save updated document
  const updatedContent = stringifyDocument(updatedFrontmatter, markdown);
  await writeFile(absolutePath, updatedContent, 'utf-8');

  console.log('✅ Successfully published to Notion!');
}

main().catch(error => {
  console.error('❌ Error:', error.message);
  process.exit(1);
});
```

**Dependencies**:
- NotionIntegration module
- frontmatter utils
- schemas

**Agent Suggestion**: modular-builder
**Test Strategy**: Manual CLI testing
**User Test Commands**:
```bash
npm run publish-notion -- documents/3.0_product_docs/test-doc.md
```

---

### 7. src/cli/commands/fetch-notion.ts (NEW)

**Purpose**: CLI command to import pages from Notion
**Pattern**: Follow `fetch-intercom.ts` pattern
**Estimated lines**: ~130 lines

**Implementation**:
```typescript
#!/usr/bin/env node
import { resolve, join } from 'path';
import { writeFile, mkdir } from 'fs/promises';
import { config } from 'dotenv';
import { NotionIntegration } from '../../modules/notion/index.js';
import { stringifyDocument } from '../../lib/frontmatter.js';

config({ path: '.env.local' });
config();

// Parse args: --database <id> OR <pageId>
const args = process.argv.slice(2);
const databaseFlag = args.indexOf('--database');
const isDatabaseMode = databaseFlag !== -1;

async function main() {
  const notion = new NotionIntegration();
  await notion.initialize();

  let pages: FetchedPage[];

  if (isDatabaseMode) {
    // Fetch multiple pages from database
    const databaseId = args[databaseFlag + 1];
    pages = await notion.fetchPages({ databaseId });
  } else {
    // Fetch single page by ID
    const pageId = args[0];
    const page = await notion.fetchPage(pageId);
    pages = [page];
  }

  // Save to documents/sync/notion/
  const outputDir = resolve('documents/sync/notion');
  await mkdir(outputDir, { recursive: true });

  for (const page of pages) {
    const filename = `${page.frontmatter.id}.md`;
    const filepath = join(outputDir, filename);
    const content = stringifyDocument(page.frontmatter, page.content);
    await writeFile(filepath, content, 'utf-8');
    console.log(`✅ Imported: ${filename}`);
  }

  console.log(`\n✅ Imported ${pages.length} page(s) to ${outputDir}`);
}

main().catch(error => {
  console.error('❌ Error:', error.message);
  process.exit(1);
});
```

**Dependencies**: NotionIntegration module
**Agent Suggestion**: modular-builder
**Test Strategy**: Manual CLI testing with actual Notion pages
**User Test Commands**:
```bash
npm run fetch-notion -- 1b6eebda-a1d2-80ae-85ba-c72f52a6222f
npm run fetch-notion -- --database 28feebdaa1d280fd9259f06ff691c56e
```

---

### 8. src/cli/commands/sync-notion.ts (NEW)

**Purpose**: CLI command for bidirectional sync
**Pattern**: Wrapper around `syncDocument()` method
**Estimated lines**: ~100 lines

**Implementation**:
```typescript
#!/usr/bin/env node
import { resolve } from 'path';
import { readFile, writeFile } from 'fs/promises';
import { config } from 'dotenv';
import { NotionIntegration } from '../../modules/notion/index.js';
import { parseFrontmatter, stringifyDocument } from '../../lib/frontmatter.js';

config({ path: '.env.local' });
config();

const docPath = process.argv[2];
if (!docPath) {
  console.error('Usage: npm run sync-notion -- <doc-path>');
  process.exit(1);
}

async function main() {
  const absolutePath = resolve(process.cwd(), docPath);
  console.log('🔄 Syncing with Notion...');

  const content = await readFile(absolutePath, 'utf-8');
  const { frontmatter, markdown } = parseFrontmatter(content);

  const notion = new NotionIntegration();
  await notion.initialize();

  // syncDocument handles create vs update logic
  const updatedFrontmatter = await notion.syncDocument(frontmatter, markdown);

  const updatedContent = stringifyDocument(updatedFrontmatter, markdown);
  await writeFile(absolutePath, updatedContent, 'utf-8');

  const pageId = updatedFrontmatter.integrations?.notion?.pageId;
  console.log(`✅ Synced successfully! Page ID: ${pageId}`);
}

main().catch(error => {
  console.error('❌ Error:', error.message);
  process.exit(1);
});
```

**Dependencies**: NotionIntegration module
**Agent Suggestion**: modular-builder
**Test Strategy**: Manual testing with existing and new docs

---

### 9. src/lib/markdoc/notion-renderer.ts (REFACTOR)

**Current State**: Contains `renderMarkdocToNotion()` at line 202
**Required Changes**: Mark as legacy, add deprecation notice

**Specific Modifications**:
```typescript
// At top of file, add comment:
// DEPRECATED: Use src/modules/notion/content-converter.ts instead
// This file kept for backward compatibility only

// Keep existing exports but add deprecation warning
/**
 * @deprecated Use convertMarkdocToNotion from modules/notion/content-converter.ts
 */
export function renderMarkdocToNotion(content: string): string {
  // Keep existing implementation for backward compatibility
}
```

**Why not delete**:
- `convert-to-notion.ts` CLI still uses it
- Allows gradual migration
- Tests may reference it

**Agent Suggestion**: modular-builder
**Estimated changes**: ~10 lines (comments only)

---

### 10. src/cli/commands/convert-to-notion.ts (UPDATE)

**Current State**: Uses `renderMarkdocToNotion()` from notion-renderer
**Required Changes**: Add note about new publish-notion command

**Specific Modifications**:
```typescript
// Add at top of file:
// NOTE: This is a legacy conversion-only tool.
// For full publishing workflow, use: npm run publish-notion
```

**Why not change**: Keep as debug/conversion tool
**Agent Suggestion**: modular-builder
**Estimated changes**: ~5 lines (comment only)

---

## Implementation Chunks

Break implementation into logical, testable chunks with clear dependencies.

### Chunk 1: Core Types and Schema (Foundation)

**Files**:
- `src/lib/schemas.ts` (modify)
- `src/modules/notion/types.ts` (new)

**Description**:
Update frontmatter schema for flexibility, create TypeScript interfaces for module.

**Why First**: All other code depends on these types being correct.

**Test Strategy**:
- TypeScript compilation validates types
- Schema validation tests for both help-center and core docs

**Dependencies**: None

**Estimated Time**: 1-2 hours

**Commit Point**: After types compile and schema tests pass

---

### Chunk 2: Content Conversion (Pure Logic)

**Files**:
- `src/modules/notion/content-converter.ts` (new)

**Description**:
Extract existing converter + add reverse converter. Pure functions, no dependencies on MCP.

**Why Second**: Can be tested in isolation, needed by orchestrator.

**Test Strategy**:
- Unit tests for markdoc → Notion
- Unit tests for Notion → markdoc
- Roundtrip tests (markdoc → Notion → markdoc)
- Edge cases: empty content, malformed markdown, unknown tags

**Dependencies**: Chunk 1 (types)

**Estimated Time**: 3-4 hours

**Commit Point**: After all conversion tests pass

---

### Chunk 3: NotionIntegration Module (Orchestrator)

**Files**:
- `src/modules/notion/index.ts` (new)
- `src/modules/notion/README.md` (new)

**Description**:
Main module class that orchestrates MCP calls, manages frontmatter, handles business logic.

**Why Third**: Depends on conversion functions, used by all CLI commands.

**Test Strategy**:
- Integration tests with mocked MCP responses
- Test publish flow (create and update)
- Test fetch flow (single page and multiple)
- Test sync flow (handles both cases)
- Test error handling (invalid docs, MCP failures)

**Dependencies**: Chunk 1 (types), Chunk 2 (conversion)

**Estimated Time**: 4-5 hours

**Commit Point**: After integration tests pass

---

### Chunk 4: CLI Commands (User Interface)

**Files**:
- `src/cli/commands/publish-notion.ts` (new)
- `src/cli/commands/fetch-notion.ts` (new)
- `src/cli/commands/sync-notion.ts` (new)

**Description**:
CLI wrappers that handle arg parsing, file I/O, user feedback.

**Why Fourth**: Depends on module being complete, adds user-facing interface.

**Test Strategy**:
- Manual CLI testing with real documents
- Test validation error messages
- Test success output formatting
- Test file save/update workflow

**Dependencies**: Chunk 3 (module)

**Estimated Time**: 2-3 hours

**Commit Point**: After manual user testing confirms all workflows work

---

### Chunk 5: Legacy Code Updates (Documentation)

**Files**:
- `src/lib/markdoc/notion-renderer.ts` (modify)
- `src/cli/commands/convert-to-notion.ts` (modify)

**Description**:
Add deprecation notices and migration guidance.

**Why Last**: Non-breaking changes, documentation only.

**Test Strategy**:
- Verify existing tests still pass
- Verify backward compatibility maintained

**Dependencies**: All other chunks (so we can reference new code)

**Estimated Time**: 30 minutes

**Commit Point**: With Chunk 4 or separately

---

## Sequential Implementation

**Must be sequential** (dependencies between chunks):

```
Chunk 1: Types & Schema
    ↓
Chunk 2: Content Conversion
    ↓
Chunk 3: NotionIntegration Module
    ↓
Chunk 4: CLI Commands
    ↓
Chunk 5: Legacy Updates
```

**Why Sequential**: Each chunk depends on the previous chunk's outputs.

**Total Estimated Time**: 11-15 hours of implementation + testing

---

## Testing Strategy

### Unit Tests (60%)

**File**: `src/modules/notion/content-converter.test.ts`
- Test `convertMarkdocToNotion()`:
  - Callouts with different types
  - Table of contents
  - Images with captions
  - Code blocks with language
  - Standard markdown (headings, lists, bold, italic)
  - Edge case: empty content
  - Edge case: malformed markdown
- Test `convertNotionToMarkdoc()`:
  - Reverse of all above
  - Notion-specific XML tags
  - Edge case: unknown Notion tags
- Test roundtrip:
  - markdoc → Notion → markdoc (should match)

**File**: `src/modules/notion/index.test.ts`
- Test `publishDocument()`:
  - Create new page (no pageId)
  - Update existing page (has pageId)
  - Help-center validation (requires databaseId)
  - Core doc (no databaseId needed)
  - Error: invalid frontmatter
  - Error: MCP call fails
- Test `fetchPage()`:
  - Successful fetch
  - Conversion to markdoc
  - Frontmatter creation
  - Error: page not found
- Test `syncDocument()`:
  - Update path (has pageId)
  - Create path (no pageId)

### Integration Tests (30%)

**File**: `tests/integration/notion-workflow.test.ts`
- Test full publish flow:
  1. Load markdoc document
  2. Publish to Notion (mocked MCP)
  3. Verify frontmatter updated
  4. Verify sync status set
- Test full import flow:
  1. Mock Notion page fetch
  2. Convert to markdoc
  3. Verify frontmatter structure
  4. Verify content converted correctly
- Test error handling:
  - MCP unavailable
  - Invalid page ID
  - Permission errors

### User Testing (10%)

**Manual CLI Testing**:

```bash
# Test publish workflow
npm run publish-notion -- documents/3.0_product_docs/test-doc.md
# Verify: Page created in Notion
# Verify: Frontmatter updated with pageId
# Verify: Content renders correctly

# Test import workflow
npm run fetch-notion -- 1b6eebda-a1d2-80ae-85ba-c72f52a6222f
# Verify: Document created in documents/sync/notion/
# Verify: Content converted to markdoc
# Verify: Frontmatter includes Notion metadata

# Test sync workflow
npm run sync-notion -- documents/3.0_product_docs/test-doc.md
# Verify: Updates existing page if pageId present
# Verify: Creates new page if pageId missing

# Test roundtrip
npm run publish-notion -- test-roundtrip.md
npm run fetch-notion -- <pageId-from-above>
# Compare: Original and imported should be substantially similar
```

**Acceptance Criteria**:
- ✅ Can publish new documents
- ✅ Can update existing documents
- ✅ Can import from Notion
- ✅ Frontmatter tracks sync state correctly
- ✅ Content converts cleanly both directions
- ✅ Error messages are clear and actionable

---

## Agent Orchestration Strategy

### Primary Agent: modular-builder

Use `modular-builder` for all implementation chunks:

```
Task modular-builder: "Implement Chunk 1 (Types & Schema) according to
code_plan.md. Follow specifications exactly. Update schema for flexibility,
create types.ts with all interfaces."
```

**Why modular-builder**:
- Specializes in creating self-contained modules
- Follows specifications precisely
- Generates clean, regeneratable code

### Supporting Agents

**zen-architect** - For architecture review before implementation:
```
Task zen-architect: "Review code_plan.md for compliance with
IMPLEMENTATION_PHILOSOPHY and MODULAR_DESIGN_PHILOSOPHY. Flag any
complexity concerns or simplification opportunities."
```

**test-coverage** - For test planning after each chunk:
```
Task test-coverage: "Review Chunk 2 (content-converter.ts) and suggest
comprehensive test cases covering all edge cases and error conditions."
```

**bug-hunter** - If issues arise during implementation:
```
Task bug-hunter: "Debug issue where convertNotionToMarkdoc fails on
nested callouts. Provide root cause analysis and fix."
```

### Execution Strategy: Sequential

**Chunk-by-chunk with verification**:

1. Implement Chunk 1 with modular-builder
2. Verify tests pass
3. Commit
4. Implement Chunk 2 with modular-builder
5. Verify tests pass
6. Commit
7. ... continue for all chunks

**Why Sequential**: Dependencies between chunks require previous chunk completion.

**Parallel NOT recommended**: Module components tightly coupled, parallel implementation would create integration issues.

---

## Philosophy Compliance

### Ruthless Simplicity ✅

**What we're NOT doing** (YAGNI):
- ❌ Block-level API manipulation (markdown is enough)
- ❌ Database schema management (not needed)
- ❌ Sync conflict resolution (out of scope)
- ❌ Caching layer (not needed yet)
- ❌ Elaborate error retry logic (fail fast initially)

**What keeps it simple**:
- ✅ Thin wrapper around MCP (no API reimplementation)
- ✅ Markdown-based conversion (not pixel-perfect blocks)
- ✅ Direct MCP tool calls (no abstraction layers)
- ✅ Frontmatter-based contract (same as Intercom)
- ✅ Reuse existing markdoc converter (just move it)

### Modular Design (Bricks & Studs) ✅

**Self-contained module** ("brick"):
- `src/modules/notion/` is complete unit
- README.md is regeneration spec
- Can rebuild from spec without reading code
- No tight coupling to consumers

**Clear interfaces** ("studs"):
- `NotionIntegration` class is public API
- `convertMarkdocToNotion()` and `convertNotionToMarkdoc()` are exported functions
- Types define contracts in types.ts
- CLI commands are thin wrappers

**Regeneratable**:
- Module README contains complete behavior spec
- Types document all interfaces
- Tests verify contract compliance
- Can regenerate entire module from docs + tests

### Library vs Custom Code ✅

**Using libraries** (complexity they handle):
- Notion MCP tools - API calls, authentication, error handling
- Markdoc - AST parsing and transformation
- Zod - Schema validation

**Custom code** (our unique logic):
- Conversion between formats (our specific needs)
- Frontmatter management (our workflow)
- Orchestration logic (our business rules)

**Trade-off accepted**: Markdown-based conversion means less control over exact Notion formatting, but dramatically simpler implementation. Can evolve to notion-sdk-js later if needed.

---

## Commit Strategy

### Commit 1: Foundation (Chunk 1)

```
feat(notion): Add flexible Notion schema and module types

- Update NotionMetadataSchema for help-center and core docs
- Add src/modules/notion/types.ts with all interfaces
- Tests: Schema validation for both doc types
- Philosophy: Ruthless simplicity - flexible, not rigid

BREAKING: NotionMetadataSchema no longer uses discriminated union
Migration: Update existing Notion frontmatter to new flexible format
```

**Files**: schemas.ts, types.ts, tests

---

### Commit 2: Content Conversion (Chunk 2)

```
feat(notion): Add bidirectional markdoc-Notion conversion

- Extract convertMarkdocToNotion from notion-renderer
- Add convertNotionToMarkdoc for import workflow
- Tests: Unit tests for both directions + roundtrip
- Philosophy: Pure functions, no side effects

No breaking changes - new functionality only
```

**Files**: content-converter.ts, tests

---

### Commit 3: Core Module (Chunk 3)

```
feat(notion): Add NotionIntegration module for bidirectional sync

- Add NotionIntegration class (orchestrator)
- Methods: publishDocument, fetchPage, syncDocument
- Thin wrapper around MCP tools (hybrid approach)
- Module README with regeneration spec
- Tests: Integration tests with mocked MCP

Philosophy: Follows Intercom pattern, leverages MCP, minimal abstraction
```

**Files**: index.ts, README.md, tests

---

### Commit 4: CLI Commands (Chunk 4)

```
feat(notion): Add CLI commands for publish, fetch, sync

- Add publish-notion.ts - publish documents to Notion
- Add fetch-notion.ts - import pages from Notion
- Add sync-notion.ts - bidirectional sync
- All follow publish-intercom.ts pattern
- Tests: Manual user testing verified

Philosophy: CLI as thin wrapper, module does heavy lifting
```

**Files**: publish-notion.ts, fetch-notion.ts, sync-notion.ts

---

### Commit 5: Documentation (Chunk 5)

```
docs(notion): Add deprecation notices to legacy code

- Mark notion-renderer.ts as deprecated
- Add migration guidance to convert-to-notion.ts
- Update comments to point to new module

No functional changes - documentation only
```

**Files**: notion-renderer.ts, convert-to-notion.ts

---

## Risk Assessment

### High Risk Changes

**Schema flexibility change**:
- **Risk**: Might break existing Notion frontmatter parsing
- **Mitigation**:
  - Test with actual documents that have Notion metadata
  - Make schema permissive (optional fields)
  - Provide clear migration guide if needed

**MCP tool dependency**:
- **Risk**: MCP tools might change or be unavailable
- **Mitigation**:
  - Add clear error when MCP unavailable
  - Document MCP requirement in README
  - Fail fast with helpful message

### Medium Risk

**Reverse conversion (Notion → markdoc)**:
- **Risk**: New code, might miss edge cases
- **Mitigation**:
  - Comprehensive test coverage
  - Start with simple conversions
  - Test with real Notion pages
  - Accept imperfect conversion (can manually edit)

**Frontmatter updates**:
- **Risk**: Could corrupt document frontmatter
- **Mitigation**:
  - Use existing `stringifyDocument()` (proven)
  - Test frontmatter preservation
  - Keep original file if publish fails

### Low Risk

**Following Intercom pattern**:
- **Risk**: Minimal - pattern is proven
- **Mitigation**: Copy structure exactly

**Pure conversion functions**:
- **Risk**: Minimal - easy to test
- **Mitigation**: Comprehensive unit tests

---

## Dependencies to Watch

### External Dependencies

**Notion MCP** (`mcp__notion__*` tools):
- Required for all Notion API calls
- Must be configured in Claude Code
- Version: Current stable (no specific version constraint)

**@markdoc/markdoc**:
- Current version: Already in package.json
- Used for AST parsing
- No version changes needed

**Zod**:
- Current version: Already in package.json
- Used for schema validation
- No version changes needed

### Internal Dependencies

**src/lib/frontmatter.ts**:
- `parseFrontmatter()` - used by all CLI commands
- `stringifyDocument()` - used to save updated docs
- Must work correctly with updated schema

**src/lib/schemas.ts**:
- `DocFrontmatterSchema` - validates all documents
- Must handle flexible Notion metadata

### No Breaking Changes Expected

All changes are additive:
- New module (doesn't change existing code)
- New CLI commands (don't replace existing)
- Schema update is permissive (optional fields)
- Legacy code marked deprecated but still works

---

## Success Criteria

### Code is Ready When:

**Functionality**:
- ✅ Can publish documents to Notion (create and update)
- ✅ Can import pages from Notion (single and multiple)
- ✅ Can sync bidirectionally (auto create-or-update)
- ✅ Frontmatter tracks sync state correctly
- ✅ Content converts cleanly both directions

**Quality**:
- ✅ All tests passing (`make check`)
- ✅ TypeScript compiles with no errors
- ✅ User testing works as documented
- ✅ No regressions in existing functionality

**Philosophy**:
- ✅ Code follows ruthless simplicity principle
- ✅ Module is self-contained (brick with studs)
- ✅ Module README can regenerate code
- ✅ No unnecessary abstractions
- ✅ Clear, readable, maintainable

**Documentation**:
- ✅ Module README complete
- ✅ All functions have clear docstrings
- ✅ CLI usage examples work when copy-pasted
- ✅ Error messages are actionable

---

## Next Steps

### Phase 3 Complete ✅

Code plan is complete and detailed. Every file has:
- Purpose clearly stated
- Current state analyzed
- Required changes specified
- Dependencies identified
- Test strategy defined
- Philosophy compliance verified

### Ready for User Approval

Present to user:
- This code_plan.md document
- Summary of 5 implementation chunks
- Estimated 11-15 hours total effort
- Sequential implementation strategy
- All files accounted for

### When Approved: `/ddd:4-code`

Phase 4 will:
1. Implement Chunk 1 (Types & Schema)
2. Get user approval for commit
3. Implement Chunk 2 (Content Conversion)
4. Get user approval for commit
5. ... continue through all chunks

Each chunk will be implemented, tested, and committed before moving to the next.

---

**Plan Version**: 1.0
**Last Updated**: 2025-11-12
**Ready for**: Phase 4 implementation
