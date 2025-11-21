# DDD Plan: Notion Import/Export Integration

**Status**: Phase 1 Complete - Ready for Approval
**Date**: 2025-11-12
**Feature**: Add bidirectional Notion integration to product-context

---

## Problem Statement

**What we're solving**: Enable bidirectional sync between product-context documents and Notion pages.

**User value**:
- **Export**: Publish markdoc documents to Notion's 3.0 Help Center database
- **Import**: Fetch Notion pages and convert to markdoc for local editing
- **Sync tracking**: Maintain sync state in document frontmatter
- **CLI support**: Reusable from both CLI commands and Claude Code skills

**Current limitations**:
- Existing `/notion-publish` skill works but logic is scattered
- No import capability (can't fetch from Notion)
- No reusable module for CLI commands
- Conversion only works one-way (markdoc → Notion)

---

## Proposed Solution

**Hybrid Approach**: Thin module wrapper around Notion MCP tools

Create `src/modules/notion/` with:
- **NotionIntegration** class - orchestrates MCP calls
- **Bidirectional converter** - markdoc ↔ Notion markdown
- **Frontmatter manager** - sync metadata tracking
- **Content handler** - leverages existing renderer + adds reverse conversion

**Why this approach**:
1. Leverages MCP (no Notion API duplication)
2. Reusable from CLI and skills
3. Simpler than full module
4. Can evolve to notion-sdk-js later if needed

---

## Alternatives Considered

### Option 1: Skill-Based Only
**Approach**: Keep logic in skills, use MCP directly

**Pros**:
- Fastest to implement
- Zero duplication with MCP

**Cons**:
- Not reusable from CLI
- Hard to test
- Logic scattered

**Why rejected**: Not reusable, doesn't follow Intercom pattern

### Option 2: Full Module with notion-sdk-js
**Approach**: Build complete module with direct Notion API integration

**Pros**:
- Full control over blocks
- Precise formatting
- No MCP dependency

**Cons**:
- Reimplements what MCP does
- Most code to maintain
- "Library over library" anti-pattern

**Why rejected**: Violates ruthless simplicity, unnecessary complexity

### Option 3: Hybrid (CHOSEN)
**Approach**: Thin wrapper around MCP with conversion logic

**Pros**:
- Reusable module
- Leverages MCP
- Lighter implementation
- Follows Intercom pattern

**Cons**:
- Markdown fidelity (not pixel-perfect blocks)
- MCP dependency

**Why chosen**: Best balance of simplicity and modularity

---

## Architecture & Design

### Key Interfaces (The "Studs")

```typescript
// Main orchestrator class
class NotionIntegration {
  // Export operations
  publishDocument(doc: DocFrontmatter, content: string): Promise<DocFrontmatter>
  updateDocument(doc: DocFrontmatter, content: string): Promise<DocFrontmatter>

  // Import operations
  fetchPage(pageId: string): Promise<{ frontmatter: DocFrontmatter, content: string }>
  fetchPages(databaseId: string, filters?: object): Promise<Array<{...}>>

  // Utilities
  getDatabase(databaseId: string): Promise<DatabaseSchema>
  initialize(): Promise<void>
}

// Conversion functions
function convertMarkdocToNotion(content: string): string
function convertNotionToMarkdoc(notionMarkdown: string): string
```

### Module Boundaries

**Module responsibilities** (`src/modules/notion/`):
- Orchestrate MCP calls with business logic
- Manage frontmatter updates
- Handle content conversion
- Provide clean API for consumers

**MCP responsibilities** (external):
- Notion API authentication
- Create/read/update Notion pages
- Handle API errors and retries

**Consumer responsibilities** (CLI/skills):
- Load documents from filesystem
- Call module methods
- Write updated documents back

### Data Models

**Input**: DocFrontmatter + content string
```yaml
integrations:
  notion:
    pageId: "abc-123"                    # Set after publish
    databaseId: "28feebdaa1d280fd..."    # Required for create
    dataSourceId: "28feebda-a1d2-801b..."
    lastSynced: "2025-11-12T10:00:00Z"
    syncStatus: "success" | "failed" | "pending"
    workspace: "Dubsado"
    properties:
      status: "Draft"                     # Notion property
      type: "External"                    # Notion property
```

**Output**: Updated DocFrontmatter + Notion page ID

**Notion MCP format** (what we work with):
```xml
<page url="...">
  <properties>{"Name": "...", "Status": "..."}</properties>
  <content>
    # Heading

    <image source="https://..."></image>
    <callout icon="💡" color="blue_bg">Note</callout>
  </content>
</page>
```

---

## Files to Change

### Non-Code Files (Phase 2)

- [ ] **README.md** - Add Notion import/export to workflow examples
- [ ] **docs/document_driven_development/overview.md** - Not applicable
- [ ] **ai_context/FRONTMATTER_SCHEMA.md** - Already supports Notion, no changes needed
- [ ] **.claude/skills/notion-publish/SKILL.md** - Update to use new module instead of inline logic

### Code Files (Phase 4)

#### New Files to Create

- [ ] **src/modules/notion/index.ts** - NotionIntegration class (main orchestrator)
- [ ] **src/modules/notion/content-converter.ts** - Bidirectional conversion (markdoc ↔ Notion markdown)
- [ ] **src/modules/notion/types.ts** - TypeScript types for Notion integration
- [ ] **src/modules/notion/README.md** - Module regeneration spec
- [ ] **src/cli/commands/fetch-notion.ts** - CLI command to import from Notion
- [ ] **src/cli/commands/sync-notion.ts** - CLI command to sync both ways

#### Existing Files to Modify

- [ ] **src/lib/markdoc/notion-renderer.ts** - Extract to content-converter, keep as export function
- [ ] **src/cli/commands/convert-to-notion.ts** - Refactor to use module instead of direct renderer
- [ ] **src/cli/commands/publish-intercom.ts** - Reference pattern for Notion commands
- [ ] **package.json** - Add npm scripts for Notion commands
- [ ] **src/lib/schemas.ts** - Verify Notion frontmatter schema (likely already complete)

---

## Philosophy Alignment

### Ruthless Simplicity

✅ **Start minimal, grow as needed**:
- Begin with markdown-based conversion (simple)
- Can add notion-sdk-js later if pixel-perfect needed
- Only build what's required now

✅ **Avoid future-proofing**:
- Not building block-level APIs yet
- Not adding database schema management
- Not building sync conflict resolution

✅ **Clear over clever**:
- Thin wrapper pattern is obvious
- Conversion functions are pure
- MCP handles complexity

✅ **Minimize abstractions**:
- One class (NotionIntegration)
- Two conversion functions
- Direct MCP delegation

### Modular Design (Bricks & Studs)

✅ **Self-contained modules** ("bricks"):
- `modules/notion/` is complete unit
- Can regenerate from README spec
- No tight coupling to consumers

✅ **Clear interfaces** ("studs"):
- NotionIntegration class is public API
- Conversion functions are exported
- Types define contracts

✅ **Regeneratable from spec**:
- README.md will document complete module behavior
- Can rebuild from spec without reading code
- Follows Intercom module pattern

✅ **Human architects, AI builds**:
- This plan is the architecture
- Implementation can be delegated to agents
- Tests verify contract compliance

---

## Test Strategy

### Unit Tests

**Conversion functions**:
- `convertMarkdocToNotion()` - test all markdoc constructs
- `convertNotionToMarkdoc()` - test reverse conversion
- Edge cases: empty content, malformed markdown, unknown tags

**Frontmatter management**:
- Updates sync metadata correctly
- Preserves existing fields
- Handles missing notion section

**Module initialization**:
- Handles missing MCP gracefully
- Validates database IDs

### Integration Tests

**Full publish flow**:
1. Load markdoc document
2. Convert to Notion markdown
3. Mock MCP create-pages call
4. Verify frontmatter updated
5. Verify sync status set

**Full import flow**:
1. Mock MCP fetch call with Notion markdown
2. Convert to markdoc
3. Verify frontmatter created
4. Verify content matches expected

**Error handling**:
- MCP errors propagate correctly
- Frontmatter not updated on failure
- Clear error messages

### User Testing

**Export workflow**:
```bash
npm run publish-notion -- documents/3.0_product_docs/core-projects.md
# Verify: Page created in Notion
# Verify: Frontmatter updated with pageId
# Verify: Content renders correctly in Notion
```

**Import workflow**:
```bash
npm run fetch-notion -- 1b6eebda-a1d2-80ae-85ba-c72f52a6222f
# Verify: Document created locally
# Verify: Content converted to markdoc
# Verify: Frontmatter includes Notion metadata
```

**Roundtrip test**:
```bash
# 1. Export local doc to Notion
npm run publish-notion -- test-doc.md
# 2. Delete local doc
# 3. Import from Notion
npm run fetch-notion -- <pageId>
# 4. Compare original and imported
# Verify: Content substantially matches
```

---

## Implementation Approach

### Phase 2: Documentation Updates

**Update in this order**:
1. `.claude/skills/notion-publish/SKILL.md` - Document new module usage
2. `README.md` - Add Notion import/export workflow examples
3. `package.json` - Add npm script placeholders (no code yet)

**Retcon approach**:
- Write docs as if module already exists
- Show usage examples with module API
- Document expected behavior

### Phase 4: Code Implementation

**Implementation chunks** (in order):

1. **Content converter** (`src/modules/notion/content-converter.ts`):
   - Extract existing `renderMarkdocToNotion()` from `notion-renderer.ts`
   - Add reverse: `convertNotionToMarkdoc()`
   - Test both directions independently

2. **Module types** (`src/modules/notion/types.ts`):
   - NotionIntegration method signatures
   - Import/export result types
   - Error types

3. **Main integration class** (`src/modules/notion/index.ts`):
   - Initialize method (validate MCP available)
   - `publishDocument()` - orchestrate export
   - `fetchPage()` - orchestrate import
   - Frontmatter management logic

4. **CLI commands**:
   - `src/cli/commands/fetch-notion.ts` - import from Notion
   - `src/cli/commands/sync-notion.ts` - bidirectional sync
   - Update `publish-intercom.ts` as reference pattern

5. **Update existing files**:
   - Refactor `convert-to-notion.ts` to use module
   - Update skill to use module
   - Add npm scripts

**Verification at each step**:
- Write tests first
- Implement to make tests pass
- Verify philosophy compliance

---

## Success Criteria

### Documentation Quality

- ✅ Docs and code never diverge
- ✅ Examples work when copy-pasted
- ✅ Module README is regeneration spec
- ✅ CLI commands documented

### Process Quality

- ✅ Follows Intercom module pattern
- ✅ Philosophy principles followed
- ✅ All tests pass
- ✅ No unnecessary complexity

### Functionality

- ✅ Can export markdoc → Notion
- ✅ Can import Notion → markdoc
- ✅ Frontmatter tracks sync state
- ✅ Works from both CLI and skills
- ✅ Handles errors gracefully

### Code Quality

- ✅ Module is self-contained
- ✅ Clear interfaces
- ✅ Testable units
- ✅ No duplication with MCP

---

## Next Steps

✅ **Phase 1 Complete**: Planning approved
➡️ **Ready for**: `/ddd:2-docs` - Update all non-code files

**What happens next**:
1. User reviews and approves this plan
2. Phase 2: Update docs to target state (retcon)
3. Phase 3: Code implementation guided by plan
4. Phase 4: Testing and verification
5. Phase 5: Cleanup and finalize

---

## Dependencies

**External**:
- Notion MCP tools must be available (`mcp__notion__*`)
- Existing `markdoc-renderer.ts` provides export foundation

**Internal**:
- `src/lib/schemas.ts` - Frontmatter schema (already supports Notion)
- `src/lib/frontmatter.ts` - Parsing and stringifying
- `src/modules/intercom/` - Reference pattern

**Blockers**:
- None identified

---

## Risk Assessment

**Low risk**:
- Following proven Intercom pattern
- MCP handles Notion API complexity
- Markdown conversion is well-understood

**Medium risk**:
- Notion markdown → markdoc conversion (new code)
- Edge cases in content conversion
- Frontmatter updates could conflict

**Mitigation**:
- Write comprehensive tests first
- Start with simple documents
- Add complexity incrementally
- User can always edit converted docs manually

---

**Plan Version**: 1.0
**Last Updated**: 2025-11-12
**Next Review**: After Phase 2 (docs complete)
