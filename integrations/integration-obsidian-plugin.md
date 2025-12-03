# integration-obsidian-plugin

> **Priority**: P2 (Enhancement)
> **Status**: Draft
> **Module**: `amplifier-obsidian-plugin`

## Overview

Obsidian plugin connecting your knowledge base to your codebase through Amplifier. Generate documentation from code, link notes to source files, capture technical decisions, and query your combined knowledge (notes + code) with AI assistance.

### Value Proposition

| Without | With |
|---------|------|
| Separate knowledge and code | Unified knowledge graph |
| Manual documentation updates | Auto-generated from code |
| Lost architectural decisions | Captured and linked |
| Search notes OR code | Query both simultaneously |

---

## Features

### 1. Code-to-Documentation Generation

Generate markdown documentation from codebase.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Obsidian: Architecture Notes                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  # Payment Processing Architecture                                   │
│                                                                      │
│  [🤖 Generate from Code]                                            │
│                                                                      │
│  > Generated from `src/payments/` on 2025-01-15                     │
│                                                                      │
│  ## Overview                                                         │
│                                                                      │
│  The payment system handles transaction processing through          │
│  Stripe integration with retry logic and fraud detection.           │
│                                                                      │
│  ## Components                                                       │
│                                                                      │
│  - [[PaymentProcessor]] - Core processing logic                     │
│  - [[StripeGateway]] - Stripe API integration                       │
│  - [[FraudDetector]] - Fraud signal analysis                        │
│                                                                      │
│  ## Data Flow                                                        │
│                                                                      │
│  ```mermaid                                                          │
│  graph LR                                                            │
│    A[Order] --> B[Validator]                                        │
│    B --> C[FraudDetector]                                           │
│    C --> D[PaymentProcessor]                                        │
│    D --> E[StripeGateway]                                           │
│  ```                                                                 │
│                                                                      │
│  ## Source Links                                                     │
│  - `src/payments/processor.ts:45` [[#^processor-entry]]             │
│  - `src/payments/gateway.ts:12` [[#^gateway-init]]                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Codebase Query from Notes

Ask questions about your codebase from within Obsidian.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Obsidian: Technical Queries                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  # How does authentication work?                                     │
│                                                                      │
│  ```amplifier                                                        │
│  query: How does the authentication flow work in this codebase?     │
│  include: src/auth/**                                                │
│  ```                                                                 │
│                                                                      │
│  > 🤖 **Amplifier Response** (generated 2025-01-15 10:30)           │
│  >                                                                   │
│  > The authentication system uses JWT tokens with the following     │
│  > flow:                                                             │
│  >                                                                   │
│  > 1. **Login** (`src/auth/login.ts:23`)                            │
│  >    - Validates credentials against database                       │
│  >    - Generates JWT with 24h expiration                           │
│  >                                                                   │
│  > 2. **Middleware** (`src/auth/middleware.ts:45`)                  │
│  >    - Extracts token from Authorization header                     │
│  >    - Validates signature and expiration                          │
│  >                                                                   │
│  > 3. **Refresh** (`src/auth/refresh.ts:12`)                        │
│  >    - Issues new token before expiration                          │
│  >                                                                   │
│  > See also: [[Authentication ADR-003]]                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Decision Capture

Capture architectural decisions linked to code.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Obsidian: ADR-005 Retry Strategy                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  # ADR-005: Exponential Backoff for Payment Retries                 │
│                                                                      │
│  **Status**: Accepted                                                │
│  **Date**: 2025-01-15                                                │
│  **Deciders**: @alice, @bob                                         │
│                                                                      │
│  ## Context                                                          │
│                                                                      │
│  Payment API calls to Stripe occasionally fail due to rate          │
│  limiting and transient network issues.                              │
│                                                                      │
│  ```amplifier-context                                                │
│  related_code:                                                       │
│    - src/payments/processor.ts                                       │
│    - src/payments/retry.ts                                           │
│  issue: #234                                                         │
│  ```                                                                 │
│                                                                      │
│  ## Decision                                                         │
│                                                                      │
│  Implement exponential backoff with jitter:                         │
│  - Base delay: 1 second                                              │
│  - Max retries: 3                                                    │
│  - Jitter: ±20%                                                      │
│                                                                      │
│  ## Implementation                                                   │
│                                                                      │
│  [🤖 View Implementation Status]                                    │
│                                                                      │
│  > ✅ `src/payments/retry.ts` - Retry logic implemented             │
│  > ✅ `src/payments/processor.ts:89` - Integrated with processor    │
│  > ⚠️  Tests pending for edge cases                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Combined Knowledge Search

Search across notes AND codebase simultaneously.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Obsidian: Amplifier Search                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  🔍 Search: "payment validation"                    [Notes + Code]  │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  📝 Notes (3)                                                        │
│  ├── [[Payment Processing Architecture]]                            │
│  │   "...validation occurs in PaymentValidator before..."           │
│  ├── [[ADR-002 Validation Strategy]]                                │
│  │   "...decided to validate at API boundary..."                    │
│  └── [[Q4 Payment Improvements]]                                    │
│       "...improve validation error messages..."                      │
│                                                                      │
│  💻 Code (5)                                                         │
│  ├── src/payments/validator.ts:23                                   │
│  │   `export class PaymentValidator { validate(order)...`           │
│  ├── src/payments/processor.ts:45                                   │
│  │   `const validated = await this.validator.validate...`           │
│  ├── src/api/payments.ts:12                                         │
│  │   `// Validate payment request before processing`                │
│  ├── tests/payments/validator.test.ts:34                            │
│  │   `describe('PaymentValidator', () => {...`                      │
│  └── src/types/payment.ts:8                                         │
│       `interface ValidationResult { valid: boolean...`              │
│                                                                      │
│  🤖 AI Summary                                                       │
│  "Payment validation is handled by PaymentValidator class,          │
│   following the ADR-002 decision to validate at API boundaries.     │
│   Currently 5 files implement validation logic..."                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5. Code Block Sync

Keep code blocks in notes synchronized with actual code.

```markdown
# Example Usage

This code block is synced with the actual source:

```typescript src/payments/processor.ts:45-60 sync
// Amplifier will update this block when source changes
async processPayment(order: Order): Promise<PaymentResult> {
  const validated = await this.validator.validate(order);
  if (!validated.valid) {
    throw new ValidationError(validated.errors);
  }
  return this.gateway.charge(order.total);
}
```

> ⚠️ Source changed on 2025-01-15. [View diff] [Update block]
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Obsidian Plugin                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Doc          │  │ Query        │  │ Sync         │              │
│  │ Generator    │  │ Interface    │  │ Manager      │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                       │
│         └─────────────────┼─────────────────┘                       │
│                           ▼                                         │
│                  ┌─────────────────┐                                │
│                  │ Amplifier       │                                │
│                  │ Connector       │                                │
│                  └────────┬────────┘                                │
│                           │                                         │
│         ┌─────────────────┼─────────────────┐                       │
│         ▼                 ▼                 ▼                       │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐                  │
│  │ Local CLI  │   │ API Server │   │ File       │                  │
│  │            │   │            │   │ Watcher    │                  │
│  └────────────┘   └────────────┘   └────────────┘                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Plugin Structure

```
amplifier-obsidian-plugin/
├── src/
│   ├── main.ts                 # Plugin entry point
│   ├── settings.ts             # Settings management
│   ├── connector.ts            # Amplifier connection
│   ├── features/
│   │   ├── doc-generator.ts    # Documentation generation
│   │   ├── query-interface.ts  # Codebase queries
│   │   ├── decision-capture.ts # ADR management
│   │   ├── combined-search.ts  # Unified search
│   │   └── code-sync.ts        # Code block sync
│   ├── ui/
│   │   ├── search-modal.ts     # Search UI
│   │   ├── query-view.ts       # Query results
│   │   └── settings-tab.ts     # Settings UI
│   └── utils/
│       ├── markdown.ts         # Markdown processing
│       └── code-parser.ts      # Code reference parsing
├── styles.css
├── manifest.json
└── package.json
```

---

## Implementation

### Plugin Entry Point

```typescript
// src/main.ts
import { Plugin, MarkdownPostProcessorContext } from 'obsidian';
import { AmplifierConnector } from './connector';
import { DocGenerator } from './features/doc-generator';
import { QueryInterface } from './features/query-interface';
import { CombinedSearch } from './features/combined-search';
import { CodeSync } from './features/code-sync';
import { AmplifierSettingTab, AmplifierSettings } from './settings';

export default class AmplifierPlugin extends Plugin {
  settings: AmplifierSettings;
  connector: AmplifierConnector;

  async onload() {
    await this.loadSettings();

    // Initialize connector
    this.connector = new AmplifierConnector(this.settings);
    await this.connector.connect();

    // Initialize features
    const docGenerator = new DocGenerator(this.connector);
    const queryInterface = new QueryInterface(this.connector);
    const combinedSearch = new CombinedSearch(this.connector);
    const codeSync = new CodeSync(this.connector, this.app.vault);

    // Register commands
    this.addCommand({
      id: 'generate-docs',
      name: 'Generate Documentation from Code',
      callback: () => docGenerator.generate()
    });

    this.addCommand({
      id: 'query-codebase',
      name: 'Query Codebase',
      callback: () => queryInterface.openQueryModal()
    });

    this.addCommand({
      id: 'combined-search',
      name: 'Search Notes + Code',
      callback: () => combinedSearch.openSearchModal()
    });

    this.addCommand({
      id: 'sync-code-blocks',
      name: 'Sync Code Blocks in Current Note',
      callback: () => codeSync.syncCurrentNote()
    });

    // Register markdown processors
    this.registerMarkdownCodeBlockProcessor(
      'amplifier',
      (source, el, ctx) => queryInterface.processQueryBlock(source, el, ctx)
    );

    this.registerMarkdownCodeBlockProcessor(
      'amplifier-context',
      (source, el, ctx) => docGenerator.processContextBlock(source, el, ctx)
    );

    // Settings tab
    this.addSettingTab(new AmplifierSettingTab(this.app, this));

    // Ribbon icon
    this.addRibbonIcon('bot', 'Amplifier', () => {
      combinedSearch.openSearchModal();
    });
  }

  async onunload() {
    await this.connector.disconnect();
  }

  async loadSettings() {
    this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());
  }

  async saveSettings() {
    await this.saveData(this.settings);
  }
}
```

### Documentation Generator

```typescript
// src/features/doc-generator.ts
import { Modal, Notice, TFile } from 'obsidian';
import { AmplifierConnector } from '../connector';

export class DocGenerator {
  constructor(private connector: AmplifierConnector) {}

  async generate() {
    const modal = new GenerateDocsModal(this.connector);
    modal.open();
  }

  async generateFromPath(codePath: string, options: GenerateOptions): Promise<string> {
    const response = await this.connector.execute({
      prompt: `Generate comprehensive markdown documentation for the code at ${codePath}.

Include:
- Overview and purpose
- Component descriptions with links
- Data flow diagrams (mermaid)
- Source file references
- Related concepts

Format for Obsidian with [[wiki links]] for components.`,
      context: {
        type: 'doc_generation',
        code_path: codePath,
        options
      }
    });

    return response.response;
  }

  async processContextBlock(
    source: string,
    el: HTMLElement,
    ctx: MarkdownPostProcessorContext
  ) {
    // Parse context block
    const context = parseYaml(source);

    // Create UI for viewing related code
    const container = el.createDiv({ cls: 'amplifier-context' });

    if (context.related_code) {
      const codeList = container.createEl('ul');
      for (const codePath of context.related_code) {
        const item = codeList.createEl('li');
        item.createEl('a', {
          text: codePath,
          href: `vscode://file/${codePath}`
        });

        // Add status indicator
        const status = await this.checkImplementationStatus(codePath);
        item.createSpan({
          text: status.implemented ? ' ✅' : ' ⚠️',
          cls: status.implemented ? 'status-done' : 'status-pending'
        });
      }
    }
  }
}
```

### Query Interface

```typescript
// src/features/query-interface.ts
import { Modal, TextAreaComponent } from 'obsidian';
import { AmplifierConnector } from '../connector';

export class QueryInterface {
  constructor(private connector: AmplifierConnector) {}

  openQueryModal() {
    new QueryModal(this.connector).open();
  }

  async processQueryBlock(
    source: string,
    el: HTMLElement,
    ctx: MarkdownPostProcessorContext
  ) {
    const container = el.createDiv({ cls: 'amplifier-query' });

    // Parse query block
    const lines = source.split('\n');
    const queryLine = lines.find(l => l.startsWith('query:'));
    const includeLine = lines.find(l => l.startsWith('include:'));

    const query = queryLine?.replace('query:', '').trim();
    const include = includeLine?.replace('include:', '').trim();

    if (!query) {
      container.createEl('p', { text: 'No query specified' });
      return;
    }

    // Check for cached response
    const cacheKey = `query:${query}:${include}`;
    const cached = await this.getCache(cacheKey);

    if (cached) {
      this.renderResponse(container, cached);
      return;
    }

    // Show loading
    const loading = container.createEl('p', { text: '🤖 Querying codebase...' });

    try {
      const response = await this.connector.execute({
        prompt: query,
        context: {
          type: 'codebase_query',
          include_pattern: include
        }
      });

      loading.remove();
      this.renderResponse(container, response.response);

      // Cache response
      await this.setCache(cacheKey, response.response);
    } catch (error) {
      loading.remove();
      container.createEl('p', {
        text: `Error: ${error.message}`,
        cls: 'amplifier-error'
      });
    }
  }

  private renderResponse(container: HTMLElement, response: string) {
    const responseEl = container.createDiv({ cls: 'amplifier-response' });

    // Add timestamp
    responseEl.createEl('small', {
      text: `Generated ${new Date().toLocaleDateString()}`,
      cls: 'amplifier-timestamp'
    });

    // Render markdown response
    MarkdownRenderer.renderMarkdown(
      response,
      responseEl,
      '',
      null
    );

    // Add refresh button
    const refreshBtn = responseEl.createEl('button', {
      text: '🔄 Refresh',
      cls: 'amplifier-refresh'
    });
    refreshBtn.onclick = () => this.refreshQuery(container);
  }
}

class QueryModal extends Modal {
  private query: string = '';
  private result: string = '';

  constructor(private connector: AmplifierConnector) {
    super(connector.app);
  }

  onOpen() {
    const { contentEl } = this;

    contentEl.createEl('h2', { text: '🤖 Query Codebase' });

    // Query input
    new TextAreaComponent(contentEl)
      .setPlaceholder('Ask about your codebase...')
      .onChange(value => this.query = value);

    // Submit button
    const submitBtn = contentEl.createEl('button', { text: 'Query' });
    submitBtn.onclick = () => this.executeQuery();

    // Results area
    this.resultEl = contentEl.createDiv({ cls: 'query-results' });
  }

  async executeQuery() {
    this.resultEl.empty();
    this.resultEl.createEl('p', { text: 'Querying...' });

    try {
      const response = await this.connector.execute({
        prompt: this.query,
        context: { type: 'codebase_query' }
      });

      this.resultEl.empty();
      MarkdownRenderer.renderMarkdown(
        response.response,
        this.resultEl,
        '',
        null
      );
    } catch (error) {
      this.resultEl.empty();
      this.resultEl.createEl('p', {
        text: `Error: ${error.message}`,
        cls: 'error'
      });
    }
  }
}
```

### Combined Search

```typescript
// src/features/combined-search.ts
import { FuzzySuggestModal, TFile } from 'obsidian';
import { AmplifierConnector } from '../connector';

interface SearchResult {
  type: 'note' | 'code';
  title: string;
  excerpt: string;
  path: string;
  line?: number;
}

export class CombinedSearch {
  constructor(private connector: AmplifierConnector) {}

  openSearchModal() {
    new CombinedSearchModal(this.connector).open();
  }

  async search(query: string): Promise<{
    notes: SearchResult[];
    code: SearchResult[];
    summary: string;
  }> {
    // Search notes locally
    const notes = await this.searchNotes(query);

    // Search code via Amplifier
    const codeResponse = await this.connector.execute({
      prompt: `Search the codebase for: "${query}"

Return results as JSON array with fields:
- path: file path
- line: line number
- excerpt: relevant code snippet
- relevance: why this matches`,
      context: { type: 'code_search' }
    });

    const code = this.parseCodeResults(codeResponse.response);

    // Generate AI summary
    const summaryResponse = await this.connector.execute({
      prompt: `Summarize these search results for "${query}":

Notes found: ${notes.length}
${notes.map(n => `- ${n.title}: ${n.excerpt}`).join('\n')}

Code found: ${code.length}
${code.map(c => `- ${c.path}:${c.line}: ${c.excerpt}`).join('\n')}

Provide a brief summary connecting these results.`,
      context: { type: 'search_summary' }
    });

    return {
      notes,
      code,
      summary: summaryResponse.response
    };
  }
}
```

---

## Configuration

```yaml
# Amplifier settings in Obsidian
connection:
  mode: local  # local | api
  api_url: https://api.amplifier.example.com
  api_key: xxx

codebase:
  root: /path/to/project
  include:
    - src/**
    - lib/**
  exclude:
    - node_modules/**
    - dist/**

features:
  doc_generation: true
  query_interface: true
  combined_search: true
  code_sync: true
  decision_capture: true

sync:
  auto_sync: false
  check_interval: 300  # seconds

cache:
  enabled: true
  ttl: 3600  # seconds
```

---

## Code Block Syntax

### Query Block

```markdown
```amplifier
query: How does the payment system handle retries?
include: src/payments/**
profile: enterprise-dev:analysis
```
```

### Context Block

```markdown
```amplifier-context
related_code:
  - src/payments/processor.ts
  - src/payments/retry.ts
issue: #234
decision: ADR-005
```
```

### Synced Code Block

```markdown
```typescript src/payments/processor.ts:45-60 sync
// Code here will be kept in sync
```
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `obsidian:doc_generated` | Documentation created | path, source |
| `obsidian:query_executed` | Query completed | query, results_count |
| `obsidian:search_combined` | Combined search done | query, notes, code |
| `obsidian:code_synced` | Code block synced | file, lines |

---

## Open Questions

1. **Bidirectional sync**: Should code changes update notes automatically?
2. **Graph integration**: Visualize code-note relationships in graph view?
3. **Version control**: Track documentation versions with code versions?
4. **Collaboration**: Share queries and results across team?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
