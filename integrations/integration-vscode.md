# integration-vscode

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-integration-vscode`

## Overview

VS Code extension providing native Amplifier integration within the IDE. Enables context-aware AI assistance, inline code actions, project-aware suggestions, and seamless access to all Amplifier capabilities.

### Value Proposition

| Without | With |
|---------|------|
| Context switching to terminal | Inline AI assistance |
| Manual context sharing | Auto-detect project context |
| Generic AI suggestions | Codebase-aware responses |
| Separate tool windows | Integrated experience |

---

## Features

### 1. Inline Chat

AI chat panel integrated in VS Code sidebar.

```typescript
// Chat with full project context
User: "How does the payment processing work?"

// Amplifier searches codebase, finds relevant files
Assistant: "Based on the codebase analysis:

The payment processing flow works as follows:

1. **Entry Point** (`src/api/payments.ts:45`)
   - Receives payment request via POST /api/payments

2. **Validation** (`src/services/PaymentValidator.ts`)
   - Validates card details
   - Checks fraud signals

3. **Processing** (`src/services/PaymentProcessor.ts`)
   - Integrates with Stripe API
   - Implements retry logic with exponential backoff

4. **Completion** (`src/services/PaymentNotifier.ts`)
   - Sends confirmation email
   - Updates order status

Would you like me to explain any specific part in detail?"
```

### 2. Inline Code Actions

Context menu and lightbulb actions for selected code.

```typescript
// Right-click on code or use lightbulb
Code Actions:
  🔧 Amplifier: Explain this code
  🔧 Amplifier: Suggest improvements
  🔧 Amplifier: Find bugs
  🔧 Amplifier: Generate tests
  🔧 Amplifier: Add documentation
  🔧 Amplifier: Refactor
```

### 3. Smart Completions

AI-powered code completions with codebase awareness.

```typescript
// While typing in payment.ts
async function processRefund(orderId: string) {
  // Amplifier suggests based on existing patterns
  const order = await this.orderService.getOrder(orderId);
  const payment = await this.paymentService.getPaymentByOrder(orderId);

  // Suggestion shows: "Based on pattern in processPayment..."
  if (!payment.refundable) {
    throw new PaymentError('Payment is not refundable', 'REFUND_NOT_ALLOWED');
  }
  //...
}
```

### 4. Project Commands

Command palette commands for project-wide operations.

```
Cmd/Ctrl + Shift + P:

> Amplifier: Explain Repository Architecture
> Amplifier: Find Dead Code
> Amplifier: Analyze Dependencies
> Amplifier: Generate Documentation
> Amplifier: Review Uncommitted Changes
> Amplifier: Find Similar Code
> Amplifier: Create Feature Plan
```

### 5. Problems Integration

AI-detected issues in VS Code Problems panel.

```
PROBLEMS (3)
  src/payments/gateway.ts
    ⚠️ [Amplifier] Potential SQL injection at line 45
    ⚠️ [Amplifier] Missing error handling for network timeout
  src/auth/login.ts
    ⚠️ [Amplifier] Weak password validation pattern
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        VS Code Extension                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Chat View    │  │ Code Actions │  │ Completions  │              │
│  │ Provider     │  │ Provider     │  │ Provider     │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                       │
│         └─────────────────┼─────────────────┘                       │
│                           ▼                                         │
│                  ┌─────────────────┐                                │
│                  │ Amplifier Client│                                │
│                  │   (TypeScript)  │                                │
│                  └────────┬────────┘                                │
│                           │                                         │
│                           ▼                                         │
│                  ┌─────────────────┐                                │
│                  │  Amplifier CLI  │◀── Local or Remote            │
│                  │    Process      │                                │
│                  └─────────────────┘                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Extension Configuration

```json
// .vscode/settings.json
{
  "amplifier.enabled": true,

  // Connection
  "amplifier.connection": {
    "type": "local",              // local | remote | cloud
    "path": "amplifier"           // CLI path for local
  },

  // Profile
  "amplifier.profile": "enterprise-dev:development",
  "amplifier.collection": "enterprise-dev",

  // Features
  "amplifier.features": {
    "inlineChat": true,
    "codeActions": true,
    "completions": true,
    "problemsIntegration": true,
    "hoverInfo": true
  },

  // Completions
  "amplifier.completions": {
    "enabled": true,
    "triggerCharacters": [".", "(", " "],
    "maxSuggestions": 3,
    "codebaseAware": true
  },

  // Security
  "amplifier.security": {
    "redactSecrets": true,
    "excludePatterns": [
      "*.env",
      "*.pem",
      "**/secrets/**"
    ]
  },

  // Keybindings
  "amplifier.keybindings": {
    "openChat": "cmd+shift+a",
    "explainSelection": "cmd+shift+e",
    "generateTests": "cmd+shift+t"
  }
}
```

---

## Implementation

### Extension Entry Point

```typescript
// extension.ts
import * as vscode from 'vscode';
import { AmplifierClient } from './client';
import { ChatViewProvider } from './providers/chat';
import { CodeActionProvider } from './providers/codeActions';
import { CompletionProvider } from './providers/completions';
import { ProblemsProvider } from './providers/problems';

export async function activate(context: vscode.ExtensionContext) {
  const config = vscode.workspace.getConfiguration('amplifier');

  // Initialize Amplifier client
  const client = new AmplifierClient({
    connection: config.get('connection'),
    profile: config.get('profile')
  });

  await client.connect();

  // Register Chat View
  const chatProvider = new ChatViewProvider(client, context);
  context.subscriptions.push(
    vscode.window.registerWebviewViewProvider(
      'amplifier.chatView',
      chatProvider
    )
  );

  // Register Code Actions
  const codeActionProvider = new CodeActionProvider(client);
  context.subscriptions.push(
    vscode.languages.registerCodeActionsProvider(
      { scheme: 'file' },
      codeActionProvider,
      { providedCodeActionKinds: [vscode.CodeActionKind.QuickFix] }
    )
  );

  // Register Completions
  if (config.get('features.completions')) {
    const completionProvider = new CompletionProvider(client);
    context.subscriptions.push(
      vscode.languages.registerCompletionItemProvider(
        { scheme: 'file' },
        completionProvider,
        ...config.get('completions.triggerCharacters')
      )
    );
  }

  // Register Problems Integration
  if (config.get('features.problemsIntegration')) {
    const problemsProvider = new ProblemsProvider(client, context);
    context.subscriptions.push(problemsProvider);
  }

  // Register Commands
  registerCommands(context, client);
}

function registerCommands(
  context: vscode.ExtensionContext,
  client: AmplifierClient
) {
  // Explain selection
  context.subscriptions.push(
    vscode.commands.registerCommand('amplifier.explainSelection', async () => {
      const editor = vscode.window.activeTextEditor;
      if (!editor) return;

      const selection = editor.selection;
      const text = editor.document.getText(selection);

      const result = await client.execute({
        prompt: `Explain this code:\n\n${text}`,
        context: {
          file: editor.document.fileName,
          language: editor.document.languageId
        }
      });

      // Show in chat panel
      vscode.commands.executeCommand('amplifier.chatView.focus');
      chatProvider.addMessage('assistant', result.response);
    })
  );

  // More commands...
}
```

### Chat View Provider

```typescript
// providers/chat.ts
export class ChatViewProvider implements vscode.WebviewViewProvider {
  private view?: vscode.WebviewView;
  private messages: Message[] = [];

  constructor(
    private client: AmplifierClient,
    private context: vscode.ExtensionContext
  ) {}

  resolveWebviewView(webviewView: vscode.WebviewView) {
    this.view = webviewView;

    webviewView.webview.options = {
      enableScripts: true,
      localResourceRoots: [this.context.extensionUri]
    };

    webviewView.webview.html = this.getHtml();

    // Handle messages from webview
    webviewView.webview.onDidReceiveMessage(async (message) => {
      if (message.type === 'userMessage') {
        await this.handleUserMessage(message.text);
      }
    });
  }

  async handleUserMessage(text: string) {
    // Add user message
    this.addMessage('user', text);

    // Get project context
    const context = await this.gatherContext();

    // Execute with Amplifier
    const result = await this.client.execute({
      prompt: text,
      context: context
    });

    // Add assistant response
    this.addMessage('assistant', result.response);
  }

  async gatherContext(): Promise<ProjectContext> {
    const editor = vscode.window.activeTextEditor;
    const workspaceFolder = vscode.workspace.workspaceFolders?.[0];

    return {
      // Current file context
      currentFile: editor?.document.fileName,
      currentSelection: editor?.selection
        ? editor.document.getText(editor.selection)
        : undefined,
      currentLanguage: editor?.document.languageId,

      // Project context
      workspaceRoot: workspaceFolder?.uri.fsPath,
      openFiles: vscode.window.visibleTextEditors.map(e => e.document.fileName),

      // Git context
      gitBranch: await this.getGitBranch(),
      gitStatus: await this.getGitStatus()
    };
  }

  addMessage(role: 'user' | 'assistant', content: string) {
    this.messages.push({ role, content, timestamp: new Date() });
    this.updateView();
  }

  private updateView() {
    this.view?.webview.postMessage({
      type: 'updateMessages',
      messages: this.messages
    });
  }
}
```

### Code Action Provider

```typescript
// providers/codeActions.ts
export class CodeActionProvider implements vscode.CodeActionProvider {
  constructor(private client: AmplifierClient) {}

  async provideCodeActions(
    document: vscode.TextDocument,
    range: vscode.Range
  ): Promise<vscode.CodeAction[]> {
    const actions: vscode.CodeAction[] = [];

    // Only show when there's a selection
    if (range.isEmpty) return actions;

    const selectedText = document.getText(range);

    // Explain code
    const explainAction = new vscode.CodeAction(
      '🔧 Amplifier: Explain this code',
      vscode.CodeActionKind.QuickFix
    );
    explainAction.command = {
      command: 'amplifier.explainSelection',
      title: 'Explain Code'
    };
    actions.push(explainAction);

    // Suggest improvements
    const improveAction = new vscode.CodeAction(
      '🔧 Amplifier: Suggest improvements',
      vscode.CodeActionKind.QuickFix
    );
    improveAction.command = {
      command: 'amplifier.suggestImprovements',
      title: 'Suggest Improvements'
    };
    actions.push(improveAction);

    // Generate tests
    const testAction = new vscode.CodeAction(
      '🔧 Amplifier: Generate tests',
      vscode.CodeActionKind.QuickFix
    );
    testAction.command = {
      command: 'amplifier.generateTests',
      title: 'Generate Tests'
    };
    actions.push(testAction);

    // Add documentation
    const docAction = new vscode.CodeAction(
      '🔧 Amplifier: Add documentation',
      vscode.CodeActionKind.QuickFix
    );
    docAction.command = {
      command: 'amplifier.addDocumentation',
      title: 'Add Documentation'
    };
    actions.push(docAction);

    return actions;
  }
}
```

### Problems Integration

```typescript
// providers/problems.ts
export class ProblemsProvider implements vscode.Disposable {
  private diagnosticCollection: vscode.DiagnosticCollection;
  private debounceTimer?: NodeJS.Timeout;

  constructor(
    private client: AmplifierClient,
    context: vscode.ExtensionContext
  ) {
    this.diagnosticCollection = vscode.languages.createDiagnosticCollection('amplifier');

    // Analyze on save
    context.subscriptions.push(
      vscode.workspace.onDidSaveTextDocument((doc) => {
        this.analyzeDocument(doc);
      })
    );

    // Analyze open documents
    vscode.workspace.textDocuments.forEach((doc) => {
      this.analyzeDocument(doc);
    });
  }

  async analyzeDocument(document: vscode.TextDocument) {
    // Debounce
    if (this.debounceTimer) {
      clearTimeout(this.debounceTimer);
    }

    this.debounceTimer = setTimeout(async () => {
      const result = await this.client.execute({
        prompt: 'Analyze this code for potential issues',
        context: {
          file: document.fileName,
          content: document.getText(),
          language: document.languageId
        },
        scenario: 'quick-analysis'
      });

      const diagnostics = this.parseIssues(result, document);
      this.diagnosticCollection.set(document.uri, diagnostics);
    }, 1000);
  }

  private parseIssues(
    result: ExecutionResult,
    document: vscode.TextDocument
  ): vscode.Diagnostic[] {
    const diagnostics: vscode.Diagnostic[] = [];

    for (const issue of result.issues || []) {
      const range = new vscode.Range(
        issue.line - 1, 0,
        issue.line - 1, document.lineAt(issue.line - 1).text.length
      );

      const severity = {
        critical: vscode.DiagnosticSeverity.Error,
        high: vscode.DiagnosticSeverity.Error,
        medium: vscode.DiagnosticSeverity.Warning,
        low: vscode.DiagnosticSeverity.Information
      }[issue.severity] || vscode.DiagnosticSeverity.Information;

      const diagnostic = new vscode.Diagnostic(
        range,
        `[Amplifier] ${issue.message}`,
        severity
      );
      diagnostic.source = 'amplifier';
      diagnostic.code = issue.code;

      diagnostics.push(diagnostic);
    }

    return diagnostics;
  }

  dispose() {
    this.diagnosticCollection.dispose();
  }
}
```

---

## User Interface

### Chat Panel

```html
<!-- Chat webview HTML -->
<div class="amplifier-chat">
  <div class="chat-messages" id="messages">
    <!-- Messages rendered here -->
  </div>

  <div class="chat-input">
    <textarea
      id="input"
      placeholder="Ask about your code..."
      rows="3"
    ></textarea>
    <button onclick="sendMessage()">Send</button>
  </div>

  <div class="chat-actions">
    <button onclick="clearChat()">Clear</button>
    <button onclick="exportChat()">Export</button>
  </div>
</div>
```

### Status Bar

```typescript
// Status bar integration
const statusBarItem = vscode.window.createStatusBarItem(
  vscode.StatusBarAlignment.Right,
  100
);

statusBarItem.text = "$(hubot) Amplifier";
statusBarItem.tooltip = "Amplifier AI Assistant - Click to open chat";
statusBarItem.command = "amplifier.chatView.focus";
statusBarItem.show();

// Update status based on connection
client.on('connected', () => {
  statusBarItem.text = "$(hubot) Amplifier";
  statusBarItem.backgroundColor = undefined;
});

client.on('disconnected', () => {
  statusBarItem.text = "$(hubot) Amplifier (Offline)";
  statusBarItem.backgroundColor = new vscode.ThemeColor(
    'statusBarItem.warningBackground'
  );
});
```

---

## Commands Reference

| Command | Keybinding | Description |
|---------|------------|-------------|
| `amplifier.openChat` | `Cmd+Shift+A` | Open chat panel |
| `amplifier.explainSelection` | `Cmd+Shift+E` | Explain selected code |
| `amplifier.generateTests` | `Cmd+Shift+T` | Generate tests |
| `amplifier.suggestImprovements` | - | Suggest improvements |
| `amplifier.addDocumentation` | - | Add documentation |
| `amplifier.findSimilarCode` | - | Find similar code |
| `amplifier.reviewChanges` | - | Review uncommitted changes |
| `amplifier.explainArchitecture` | - | Explain repository architecture |

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
