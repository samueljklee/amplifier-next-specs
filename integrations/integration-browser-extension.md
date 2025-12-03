# integration-browser-extension

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-browser-extension`

## Overview

Browser extension (Chrome, Firefox, Edge) providing Amplifier assistance where developers already work: GitHub PRs, GitLab MRs, Stack Overflow, documentation sites, and any web page with code. Context-aware AI assistance without leaving the browser.

### Value Proposition

| Without | With |
|---------|------|
| Copy code to terminal, run Amplifier, copy back | Inline assistance on any page |
| Context-switch between tools | AI where you already work |
| Manual code review | PR review in GitHub UI |
| Generic web AI | Codebase-aware with local context |

---

## Features

### 1. GitHub PR Review Assistant

Inline PR review assistance directly in GitHub UI.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Pull Request #234: Add payment processing                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Files changed (12)                  [Amplifier: Review PR ▼]       │
│                                                                      │
│  src/payments/processor.ts  +142 -23                                │
│  ─────────────────────────────────────────────────────────────────  │
│  │  45 │   async processPayment(order: Order) {                    │
│  │  46 │+    const validated = await this.validate(order);         │
│  │  47 │+    if (!validated) {                                     │
│  │  48 │+      throw new Error("Invalid order");                   │
│  │     │      ┌──────────────────────────────────────────────────┐ │
│  │     │      │ 🤖 Amplifier                                     │ │
│  │     │      │                                                   │ │
│  │     │      │ ⚠️ Generic error message may leak validation     │ │
│  │     │      │ details. Consider using a custom error type:     │ │
│  │     │      │                                                   │ │
│  │     │      │ ```ts                                            │ │
│  │     │      │ throw new ValidationError(validated.errors);     │ │
│  │     │      │ ```                                              │ │
│  │     │      │                                                   │ │
│  │     │      │ [Add Comment] [Dismiss] [Ask Follow-up]          │ │
│  │     │      └──────────────────────────────────────────────────┘ │
│  │  49 │+    }                                                     │
│  │  50 │+    return this.stripe.charge(order.total);               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Code Explanation Popup

Select any code on any page for instant explanation.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Stack Overflow: How to implement retry logic                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ const retry = async (fn, retries = 3) => {                      ││
│  │   for (let i = 0; i < retries; i++) {                           ││
│  │     try {                                                        ││
│  │       return await fn();          ← [selected text]              ││
│  │     } catch (e) {                                                ││
│  │       if (i === retries - 1) throw e;                           ││
│  │       await new Promise(r => setTimeout(r, 1000 * 2 ** i));     ││
│  │     }                                                            ││
│  │   }                                                              ││
│  │ };                                                               ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 🤖 Amplifier                                           [×]    │  │
│  │                                                               │  │
│  │ This is an **exponential backoff retry** pattern:            │  │
│  │                                                               │  │
│  │ • Attempts the function up to 3 times                        │  │
│  │ • On failure, waits exponentially: 1s → 2s → 4s              │  │
│  │ • The `2 ** i` creates the exponential growth                │  │
│  │                                                               │  │
│  │ **In your codebase**: Similar pattern in                     │  │
│  │ `src/utils/network.ts:45` but with jitter.                   │  │
│  │                                                               │  │
│  │ [Copy] [Explain More] [Find in Codebase]                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Documentation Helper

Contextual assistance when reading documentation.

```
┌─────────────────────────────────────────────────────────────────────┐
│  React Docs: useEffect Hook                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  useEffect lets you synchronize a component with an external system.│
│                                                                      │
│  useEffect(setup, dependencies?)                                     │
│                                                                      │
│  Parameters:                                                         │
│  • setup: Function with your Effect's logic...                      │
│                                                                      │
│  ───────────────────────────────────────────────────────────────────│
│  🤖 Amplifier Assistant                                    [Expand] │
│  ───────────────────────────────────────────────────────────────────│
│  │                                                                  │
│  │ In **your codebase**, useEffect is used in:                     │
│  │                                                                  │
│  │ • `UserDashboard.tsx` - fetching user data                      │
│  │ • `PaymentForm.tsx` - Stripe integration                        │
│  │ • `NotificationBell.tsx` - WebSocket subscription               │
│  │                                                                  │
│  │ [Show Examples] [Ask Question]                                   │
│  │                                                                  │
│  └──────────────────────────────────────────────────────────────────│
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. GitLab MR Assistant

Same functionality for GitLab Merge Requests.

### 5. Sidebar Chat

Persistent sidebar for extended conversations.

```
┌─────────────────────────────────────────────────────────────────┬───┐
│  Any Web Page                                                    │ 🤖│
│                                                                  │   │
│  [Page content...]                                               │ A │
│                                                                  │ m │
│                                                                  │ p │
│                                                                  │ l │
│                                                                  │ i │
│                                                                  │ f │
│                                                                  │ i │
│                                                                  │ e │
│                                                                  │ r │
│                                                                  │   │
│                                                                  │[▶]│
└─────────────────────────────────────────────────────────────────┴───┘

[Click ▶ to expand]

┌─────────────────────────────────────────────────────────┬───────────┐
│  Any Web Page                                            │ 🤖 Chat   │
│                                                          │           │
│  [Page content...]                                       │ How does  │
│                                                          │ this code │
│                                                          │ compare   │
│                                                          │ to our    │
│                                                          │ impl?     │
│                                                          │           │
│                                                          │ [Send]    │
│                                                          │           │
│                                                          │ Based on  │
│                                                          │ your code │
│                                                          │ base...   │
└─────────────────────────────────────────────────────────┴───────────┘
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Browser Extension                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Content      │  │ Background   │  │ Popup/       │              │
│  │ Scripts      │  │ Service      │  │ Sidebar      │              │
│  │ (per page)   │  │ Worker       │  │ UI           │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                       │
│         └─────────────────┼─────────────────┘                       │
│                           │                                         │
│                  ┌────────┴────────┐                                │
│                  │ Message Bridge  │                                │
│                  └────────┬────────┘                                │
│                           │                                         │
│         ┌─────────────────┼─────────────────┐                       │
│         ▼                 ▼                 ▼                       │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐                  │
│  │ Local CLI  │   │ API Server │   │ Local      │                  │
│  │ (Native    │   │ (Remote)   │   │ Storage    │                  │
│  │  Messaging)│   │            │   │            │                  │
│  └────────────┘   └────────────┘   └────────────┘                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Site-Specific Features

### GitHub Integration

```typescript
// Content script for github.com
const githubFeatures = {
  // PR review
  prReview: {
    selector: '.js-diff-progressive-container',
    actions: ['review_file', 'review_line', 'suggest_fix']
  },

  // Issue assistance
  issues: {
    selector: '.js-issue-body',
    actions: ['analyze_issue', 'suggest_solution', 'find_related']
  },

  // Code browsing
  codeView: {
    selector: '.blob-code',
    actions: ['explain', 'find_usages', 'suggest_improvements']
  },

  // Search enhancement
  search: {
    selector: '.search-results',
    actions: ['filter_by_context', 'explain_results']
  }
};
```

### GitLab Integration

```typescript
// Content script for gitlab.com
const gitlabFeatures = {
  mrReview: {
    selector: '.diff-content',
    actions: ['review_file', 'review_line', 'suggest_fix']
  },

  issues: {
    selector: '.description',
    actions: ['analyze_issue', 'suggest_solution']
  },

  pipelines: {
    selector: '.ci-job-trace',
    actions: ['explain_failure', 'suggest_fix']
  }
};
```

### Stack Overflow Integration

```typescript
// Content script for stackoverflow.com
const stackOverflowFeatures = {
  questions: {
    selector: '.question',
    actions: ['relate_to_codebase', 'find_existing_solution']
  },

  answers: {
    selector: '.answer',
    actions: ['evaluate_for_codebase', 'adapt_to_style']
  },

  codeBlocks: {
    selector: 'pre code',
    actions: ['explain', 'compare_to_codebase', 'adapt']
  }
};
```

---

## Implementation

### Manifest V3 Structure

```json
// manifest.json
{
  "manifest_version": 3,
  "name": "Amplifier",
  "version": "1.0.0",
  "description": "AI-powered code assistance in your browser",

  "permissions": [
    "storage",
    "activeTab",
    "contextMenus",
    "sidePanel",
    "nativeMessaging"
  ],

  "host_permissions": [
    "https://github.com/*",
    "https://gitlab.com/*",
    "https://*.gitlab.io/*",
    "https://stackoverflow.com/*",
    "https://docs.microsoft.com/*"
  ],

  "background": {
    "service_worker": "background.js",
    "type": "module"
  },

  "content_scripts": [
    {
      "matches": ["https://github.com/*"],
      "js": ["content/github.js"],
      "css": ["content/github.css"]
    },
    {
      "matches": ["https://gitlab.com/*", "https://*.gitlab.io/*"],
      "js": ["content/gitlab.js"],
      "css": ["content/gitlab.css"]
    },
    {
      "matches": ["https://stackoverflow.com/*"],
      "js": ["content/stackoverflow.js"],
      "css": ["content/stackoverflow.css"]
    },
    {
      "matches": ["<all_urls>"],
      "js": ["content/universal.js"],
      "css": ["content/universal.css"]
    }
  ],

  "side_panel": {
    "default_path": "sidepanel.html"
  },

  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },

  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  },

  "commands": {
    "explain-selection": {
      "suggested_key": {
        "default": "Ctrl+Shift+E",
        "mac": "Command+Shift+E"
      },
      "description": "Explain selected code"
    },
    "open-sidebar": {
      "suggested_key": {
        "default": "Ctrl+Shift+A",
        "mac": "Command+Shift+A"
      },
      "description": "Open Amplifier sidebar"
    }
  }
}
```

### Background Service Worker

```typescript
// background.js
import { AmplifierClient } from './lib/client';

const client = new AmplifierClient();

// Handle messages from content scripts
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'execute') {
    handleExecute(message, sender).then(sendResponse);
    return true; // Will respond asynchronously
  }

  if (message.type === 'review_pr') {
    handlePRReview(message, sender).then(sendResponse);
    return true;
  }

  if (message.type === 'explain_code') {
    handleExplainCode(message, sender).then(sendResponse);
    return true;
  }
});

async function handleExecute(message, sender) {
  const { prompt, context } = message;

  // Add page context
  const fullContext = {
    ...context,
    url: sender.tab?.url,
    title: sender.tab?.title
  };

  try {
    const result = await client.execute({
      prompt,
      context: fullContext
    });
    return { success: true, response: result.response };
  } catch (error) {
    return { success: false, error: error.message };
  }
}

async function handlePRReview(message, sender) {
  const { prUrl, diff } = message;

  const result = await client.execute({
    prompt: `Review this pull request for issues:\n\nURL: ${prUrl}\n\nDiff:\n${diff}`,
    profile: 'enterprise-dev:review',
    context: {
      type: 'pr_review',
      url: prUrl
    }
  });

  return { success: true, review: result.response };
}

async function handleExplainCode(message, sender) {
  const { code, language, pageContext } = message;

  const result = await client.execute({
    prompt: `Explain this ${language || 'code'}:\n\n${code}`,
    context: {
      type: 'code_explanation',
      page: pageContext
    }
  });

  return { success: true, explanation: result.response };
}

// Context menu
chrome.contextMenus.create({
  id: 'amplifier-explain',
  title: 'Explain with Amplifier',
  contexts: ['selection']
});

chrome.contextMenus.onClicked.addListener((info, tab) => {
  if (info.menuItemId === 'amplifier-explain') {
    chrome.tabs.sendMessage(tab.id, {
      type: 'explain_selection',
      selection: info.selectionText
    });
  }
});
```

### GitHub Content Script

```typescript
// content/github.js
import { createPopup, createInlineWidget } from '../lib/ui';

// Detect PR page
if (window.location.pathname.includes('/pull/')) {
  initPRReview();
}

function initPRReview() {
  // Add review button to PR header
  const prHeader = document.querySelector('.gh-header-actions');
  if (prHeader) {
    const reviewButton = document.createElement('button');
    reviewButton.className = 'btn btn-sm';
    reviewButton.innerHTML = '🤖 Amplifier Review';
    reviewButton.onclick = startPRReview;
    prHeader.prepend(reviewButton);
  }

  // Add inline review on diff lines
  observeDiffLines();
}

async function startPRReview() {
  const prUrl = window.location.href;
  const diff = extractDiff();

  const response = await chrome.runtime.sendMessage({
    type: 'review_pr',
    prUrl,
    diff
  });

  if (response.success) {
    displayReviewResults(response.review);
  }
}

function observeDiffLines() {
  // Watch for diff content to load
  const observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
      mutation.addedNodes.forEach((node) => {
        if (node.classList?.contains('blob-code-addition') ||
            node.classList?.contains('blob-code-deletion')) {
          addLineActions(node);
        }
      });
    });
  });

  observer.observe(document.body, {
    childList: true,
    subtree: true
  });
}

function addLineActions(lineElement) {
  const actionButton = document.createElement('button');
  actionButton.className = 'amplifier-line-action';
  actionButton.innerHTML = '🤖';
  actionButton.title = 'Analyze with Amplifier';
  actionButton.onclick = (e) => {
    e.stopPropagation();
    analyzeLineContext(lineElement);
  };

  lineElement.prepend(actionButton);
}

async function analyzeLineContext(lineElement) {
  const code = extractLineContext(lineElement);

  const response = await chrome.runtime.sendMessage({
    type: 'explain_code',
    code,
    language: detectLanguage(),
    pageContext: {
      file: getCurrentFile(),
      line: getCurrentLine(lineElement),
      pr: getPRNumber()
    }
  });

  if (response.success) {
    createInlineWidget(lineElement, response.explanation);
  }
}

// Universal code selection handler
document.addEventListener('mouseup', async (e) => {
  const selection = window.getSelection().toString().trim();

  if (selection.length > 10 && looksLikeCode(selection)) {
    // Show quick action popup
    const popup = createPopup({
      x: e.pageX,
      y: e.pageY,
      actions: [
        { label: 'Explain', action: () => explainSelection(selection) },
        { label: 'Find in codebase', action: () => findInCodebase(selection) },
        { label: 'Improve', action: () => suggestImprovement(selection) }
      ]
    });
  }
});

function looksLikeCode(text) {
  // Heuristics to detect code
  const codeIndicators = [
    /function\s+\w+/,
    /const\s+\w+\s*=/,
    /let\s+\w+\s*=/,
    /var\s+\w+\s*=/,
    /class\s+\w+/,
    /import\s+.*from/,
    /=>/,
    /\{\s*\n/,
    /\)\s*\{/
  ];

  return codeIndicators.some(pattern => pattern.test(text));
}
```

### Sidebar Panel

```html
<!-- sidepanel.html -->
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="sidepanel.css">
</head>
<body>
  <div id="amplifier-sidebar">
    <header>
      <h1>🤖 Amplifier</h1>
      <button id="settings-btn">⚙️</button>
    </header>

    <div id="context-info">
      <span id="page-type">GitHub PR</span>
      <span id="connection-status">● Connected</span>
    </div>

    <div id="chat-messages"></div>

    <div id="chat-input-container">
      <textarea id="chat-input" placeholder="Ask about this page..."></textarea>
      <button id="send-btn">Send</button>
    </div>

    <div id="quick-actions">
      <button data-action="explain-page">Explain Page</button>
      <button data-action="review-code">Review Code</button>
      <button data-action="find-related">Find Related</button>
    </div>
  </div>

  <script src="sidepanel.js" type="module"></script>
</body>
</html>
```

---

## Connection Modes

### 1. Local CLI (Native Messaging)

Direct connection to local Amplifier CLI:

```json
// com.amplifier.native.json (Native messaging host)
{
  "name": "com.amplifier.native",
  "description": "Amplifier Native Messaging Host",
  "path": "/usr/local/bin/amplifier-native-host",
  "type": "stdio",
  "allowed_origins": [
    "chrome-extension://[extension-id]/"
  ]
}
```

### 2. API Server (Remote)

Connect to hosted Amplifier API:

```typescript
// lib/client.ts
class AmplifierClient {
  private mode: 'native' | 'api';
  private apiUrl?: string;
  private apiKey?: string;

  async connect() {
    const settings = await chrome.storage.sync.get(['mode', 'apiUrl', 'apiKey']);

    this.mode = settings.mode || 'api';
    this.apiUrl = settings.apiUrl;
    this.apiKey = settings.apiKey;
  }

  async execute(request: ExecuteRequest): Promise<ExecuteResponse> {
    if (this.mode === 'native') {
      return this.executeNative(request);
    } else {
      return this.executeApi(request);
    }
  }

  private async executeNative(request: ExecuteRequest) {
    return new Promise((resolve, reject) => {
      chrome.runtime.sendNativeMessage(
        'com.amplifier.native',
        request,
        (response) => {
          if (chrome.runtime.lastError) {
            reject(new Error(chrome.runtime.lastError.message));
          } else {
            resolve(response);
          }
        }
      );
    });
  }

  private async executeApi(request: ExecuteRequest) {
    const response = await fetch(`${this.apiUrl}/v1/sessions/execute`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey
      },
      body: JSON.stringify(request)
    });

    return response.json();
  }
}
```

---

## Configuration

```typescript
// Extension settings
interface ExtensionSettings {
  // Connection
  mode: 'native' | 'api';
  apiUrl?: string;
  apiKey?: string;

  // Features
  features: {
    githubPrReview: boolean;
    gitlabMrReview: boolean;
    codeExplanation: boolean;
    documentationHelper: boolean;
    sidebar: boolean;
  };

  // UI
  ui: {
    theme: 'light' | 'dark' | 'system';
    sidebarPosition: 'left' | 'right';
    popupDelay: number;
  };

  // Privacy
  privacy: {
    sendPageContent: boolean;
    sendSelectionOnly: boolean;
    excludedDomains: string[];
  };

  // Keyboard shortcuts
  shortcuts: {
    explain: string;
    openSidebar: string;
    review: string;
  };
}
```

---

## Privacy & Security

### Content Handling

```typescript
// Only send what's necessary
function sanitizeContext(context: PageContext): SanitizedContext {
  return {
    // Never send full page HTML
    url: context.url,
    title: context.title,

    // Only selected code/text
    selection: context.selection,

    // Metadata only
    language: context.language,
    fileType: context.fileType,

    // No cookies, local storage, or sensitive data
  };
}

// Exclude sensitive sites
const EXCLUDED_DOMAINS = [
  'mail.google.com',
  'online.banking.*',
  '*.1password.com',
  '*.lastpass.com'
];
```

### Permissions

Request only necessary permissions:
- `activeTab`: Only access current tab when user invokes
- `storage`: Save user settings
- `contextMenus`: Right-click menu
- `sidePanel`: Sidebar chat

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `extension:installed` | Extension installed | version |
| `extension:activated` | Extension activated on page | url, site_type |
| `pr:review_started` | PR review initiated | pr_url |
| `code:explained` | Code explanation shown | language, length |
| `sidebar:opened` | Sidebar opened | page_type |

---

## Open Questions

1. **Codebase context**: How to efficiently sync local codebase context?
2. **Rate limiting**: Per-site or global limits?
3. **Offline mode**: Cache explanations for offline viewing?
4. **Team features**: Shared reviews, team settings?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
