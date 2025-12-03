# integration-electron-app

> **Priority**: P2 (Enhancement)
> **Status**: Draft
> **Module**: `amplifier-desktop`

## Overview

Standalone Electron desktop application for Amplifier. Full-featured GUI experience with offline capability, system tray integration, global hotkeys, native file access, and seamless local CLI integration. Cross-platform (macOS, Windows, Linux).

### Value Proposition

| Without | With |
|---------|------|
| Terminal-only interface | Rich GUI experience |
| Must be online | Offline-capable with local models |
| No system integration | Tray, hotkeys, notifications |
| Web browser required | Native app performance |

---

## Features

### 1. Main Chat Interface

Full-featured chat with rich rendering.

```
┌─────────────────────────────────────────────────────────────────────┐
│  ⚡ Amplifier                                    ─ □ ×             │
├─────────────────────────────────────────────────────────────────────┤
│ ┌───────────────┬───────────────────────────────────────────────────┤
│ │               │                                                   │
│ │  Sessions     │  Session: Code Analysis                          │
│ │  ──────────── │  Profile: enterprise-dev • claude-sonnet         │
│ │               │  ─────────────────────────────────────────────── │
│ │  ● Code       │                                                   │
│ │    Analysis   │  👤 You                              10:30 AM    │
│ │               │  How does the authentication system work?         │
│ │  ○ PR Review  │                                                   │
│ │    #234       │  🤖 Amplifier                        10:31 AM    │
│ │               │  ┌─────────────────────────────────────────────┐ │
│ │  ○ Incident   │  │ The authentication system uses JWT tokens:  │ │
│ │    2025-01-15 │  │                                             │ │
│ │               │  │ **1. Login Flow**                           │ │
│ │  + New        │  │ ```typescript                               │ │
│ │               │  │ // src/auth/login.ts:23                     │ │
│ │               │  │ async function login(creds: Credentials) {  │ │
│ │  ──────────── │  │   const user = await validate(creds);       │ │
│ │  Collections  │  │   return generateJWT(user);                 │ │
│ │  ──────────── │  │ }                                           │ │
│ │               │  │ ```                                         │ │
│ │  📦 enterprise│  │                                             │ │
│ │  📦 foundation│  │ [Open in Editor] [Copy] [Explain More]      │ │
│ │               │  └─────────────────────────────────────────────┘ │
│ │               │                                                   │
│ │               │  ─────────────────────────────────────────────── │
│ │               │  │ Ask about your code...              [📎] [↵] │ │
│ │               │  ─────────────────────────────────────────────── │
│ └───────────────┴───────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────────┘
```

### 2. System Tray Integration

Quick access from system tray.

```
┌──────────────────────────────────────┐
│  🤖 ← [System Tray Icon]             │
├──────────────────────────────────────┤
│                                      │
│  ┌────────────────────────────────┐  │
│  │ 🔍 Quick Query         ⌘+Space │  │
│  ├────────────────────────────────┤  │
│  │ Recent Sessions                │  │
│  │ ├── Code Analysis              │  │
│  │ ├── PR Review #234             │  │
│  │ └── Incident 2025-01-15        │  │
│  ├────────────────────────────────┤  │
│  │ ○ Status: Connected            │  │
│  │ ○ Profile: enterprise-dev      │  │
│  ├────────────────────────────────┤  │
│  │ Open Amplifier                 │  │
│  │ Settings                       │  │
│  │ ────────────────────────────── │  │
│  │ Quit                           │  │
│  └────────────────────────────────┘  │
│                                      │
└──────────────────────────────────────┘
```

### 3. Quick Query Spotlight

Global hotkey for instant queries.

```
[Press ⌘+Space anywhere]

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│                    🤖 Amplifier Quick Query                         │
│                                                                      │
│    ┌───────────────────────────────────────────────────────────┐    │
│    │ How does the payment retry logic work?                     │    │
│    └───────────────────────────────────────────────────────────┘    │
│                                                                      │
│    Recent:                                                           │
│    • What tests cover UserService?                                  │
│    • Explain the caching strategy                                   │
│    • Find dead code in src/utils                                    │
│                                                                      │
│    [↵ to search] [⎋ to close] [⌘+↵ open in main window]           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Project Workspace

Multi-project support with workspace management.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Workspaces                                                    [+]  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 📁 payment-service                                              ││
│  │    /Users/dev/projects/payment-service                          ││
│  │    Profile: enterprise-dev:backend                              ││
│  │    Last active: 2 hours ago                                     ││
│  │    [Open] [Configure] [Remove]                                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 📁 frontend-app                                                 ││
│  │    /Users/dev/projects/frontend-app                             ││
│  │    Profile: enterprise-dev:frontend                             ││
│  │    Last active: Yesterday                                       ││
│  │    [Open] [Configure] [Remove]                                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 📁 infrastructure                                               ││
│  │    /Users/dev/projects/infra                                    ││
│  │    Profile: platform-team:terraform                             ││
│  │    Last active: 3 days ago                                      ││
│  │    [Open] [Configure] [Remove]                                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
│  [+ Add Workspace]                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5. Offline Mode

Local model support for offline operation.

```
┌─────────────────────────────────────────────────────────────────────┐
│  ⚡ Amplifier                               [Offline Mode] ─ □ ×   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ⚠️ Offline Mode Active                                             │
│  Using local model: codellama-13b                                   │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  👤 You                                                10:30 AM    │
│  Explain this function:                                             │
│  ```ts                                                              │
│  function processPayment(order: Order) { ... }                      │
│  ```                                                                │
│                                                                      │
│  🤖 Amplifier (Local)                                  10:31 AM    │
│  This function processes a payment for an order...                  │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  [Reconnect] [Switch to Online]                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6. Collection Manager

Visual collection management.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Collections                                              [⚙️] [+]  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Installed                                                          │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ 📦 enterprise-dev                               v1.2.0   │      │
│  │    Enterprise development tools and profiles             │      │
│  │                                                          │      │
│  │    Profiles: base, development, review, ci               │      │
│  │    Agents: zen-architect, bug-hunter, code-reviewer      │      │
│  │    Tools: codebase-search, pr-context, jira-ops          │      │
│  │                                                          │      │
│  │    [Update Available: v1.3.0] [Configure] [Remove]       │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ 📦 foundation                                   v2.0.0   │      │
│  │    Core profiles and base configurations                 │      │
│  │                                                          │      │
│  │    Profiles: base, minimal                               │      │
│  │    Agents: general-assistant                             │      │
│  │                                                          │      │
│  │    [Up to date]                                          │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  Browse Collections                                                 │
│  ─────────────────────────────────────────────────────────────────  │
│  [Search collections...]                                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Electron App                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     Main Process                              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │ Window   │ │ Tray     │ │ Hotkey   │ │ IPC      │        │   │
│  │  │ Manager  │ │ Manager  │ │ Manager  │ │ Handler  │        │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │   │
│  │                           │                                   │   │
│  │         ┌─────────────────┼─────────────────┐                │   │
│  │         ▼                 ▼                 ▼                │   │
│  │  ┌────────────┐   ┌────────────┐   ┌────────────┐           │   │
│  │  │ Amplifier  │   │ Local      │   │ File       │           │   │
│  │  │ CLI        │   │ Models     │   │ System     │           │   │
│  │  └────────────┘   └────────────┘   └────────────┘           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              │ IPC                                   │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Renderer Process                           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │ Chat     │ │ Workspace│ │Collection│ │ Settings │        │   │
│  │  │ View     │ │ View     │ │ View     │ │ View     │        │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │   │
│  │                    React + TypeScript                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

```yaml
electron:
  version: 28.x
  builder: electron-builder

frontend:
  framework: React 18
  language: TypeScript
  styling: Tailwind CSS
  state: Zustand
  routing: React Router

backend:
  ipc: electron-ipc
  storage: electron-store
  cli: child_process spawn

local_models:
  runtime: llama.cpp / Ollama
  models:
    - codellama-13b
    - mistral-7b

auto_update:
  provider: electron-updater
  source: GitHub Releases
```

---

## Implementation

### Main Process

```typescript
// src/main/index.ts
import {
  app,
  BrowserWindow,
  Tray,
  Menu,
  globalShortcut,
  ipcMain,
  nativeTheme
} from 'electron';
import { spawn } from 'child_process';
import Store from 'electron-store';
import { autoUpdater } from 'electron-updater';

const store = new Store();
let mainWindow: BrowserWindow | null = null;
let tray: Tray | null = null;
let quickQueryWindow: BrowserWindow | null = null;

// App lifecycle
app.whenReady().then(() => {
  createMainWindow();
  createTray();
  registerGlobalShortcuts();
  checkForUpdates();
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

// Main window
function createMainWindow() {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    minWidth: 800,
    minHeight: 600,
    titleBarStyle: 'hiddenInset',
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  });

  if (isDev) {
    mainWindow.loadURL('http://localhost:3000');
  } else {
    mainWindow.loadFile('dist/index.html');
  }

  mainWindow.on('close', (event) => {
    if (store.get('minimizeToTray', true)) {
      event.preventDefault();
      mainWindow?.hide();
    }
  });
}

// System tray
function createTray() {
  tray = new Tray(path.join(__dirname, 'assets/tray-icon.png'));

  const contextMenu = Menu.buildFromTemplate([
    {
      label: 'Quick Query',
      accelerator: 'CmdOrCtrl+Space',
      click: () => showQuickQuery()
    },
    { type: 'separator' },
    {
      label: 'Recent Sessions',
      submenu: getRecentSessionsMenu()
    },
    { type: 'separator' },
    {
      label: `Status: ${getConnectionStatus()}`,
      enabled: false
    },
    {
      label: `Profile: ${store.get('currentProfile', 'default')}`,
      enabled: false
    },
    { type: 'separator' },
    {
      label: 'Open Amplifier',
      click: () => mainWindow?.show()
    },
    {
      label: 'Settings',
      click: () => {
        mainWindow?.show();
        mainWindow?.webContents.send('navigate', '/settings');
      }
    },
    { type: 'separator' },
    {
      label: 'Quit',
      click: () => app.quit()
    }
  ]);

  tray.setToolTip('Amplifier');
  tray.setContextMenu(contextMenu);

  tray.on('click', () => {
    mainWindow?.show();
  });
}

// Global shortcuts
function registerGlobalShortcuts() {
  // Quick query spotlight
  globalShortcut.register('CmdOrCtrl+Space', () => {
    showQuickQuery();
  });

  // Toggle main window
  globalShortcut.register('CmdOrCtrl+Shift+A', () => {
    if (mainWindow?.isVisible()) {
      mainWindow.hide();
    } else {
      mainWindow?.show();
    }
  });
}

// Quick query window
function showQuickQuery() {
  if (quickQueryWindow) {
    quickQueryWindow.show();
    quickQueryWindow.focus();
    return;
  }

  quickQueryWindow = new BrowserWindow({
    width: 600,
    height: 400,
    frame: false,
    transparent: true,
    alwaysOnTop: true,
    skipTaskbar: true,
    resizable: false,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true
    }
  });

  quickQueryWindow.loadFile('dist/quick-query.html');

  quickQueryWindow.on('blur', () => {
    quickQueryWindow?.hide();
  });
}

// IPC handlers
ipcMain.handle('amplifier:execute', async (event, { prompt, profile, context }) => {
  return executeAmplifier(prompt, profile, context);
});

ipcMain.handle('amplifier:execute-stream', async (event, { prompt, profile, context }) => {
  const process = spawn('amplifier', ['run', '--output-format', 'json', prompt], {
    env: { ...process.env, AMPLIFIER_PROFILE: profile }
  });

  process.stdout.on('data', (data) => {
    event.sender.send('amplifier:stream-data', data.toString());
  });

  process.on('close', (code) => {
    event.sender.send('amplifier:stream-end', { code });
  });
});

ipcMain.handle('collections:list', async () => {
  return executeAmplifierCommand(['collection', 'list', '--json']);
});

ipcMain.handle('collections:install', async (event, { source }) => {
  return executeAmplifierCommand(['collection', 'add', source]);
});

ipcMain.handle('profiles:list', async () => {
  return executeAmplifierCommand(['profile', 'list', '--json']);
});

// Amplifier CLI wrapper
async function executeAmplifier(prompt: string, profile: string, context: any) {
  return new Promise((resolve, reject) => {
    const args = ['run', '--output-format', 'json'];
    if (profile) args.push('--profile', profile);
    args.push(prompt);

    const process = spawn('amplifier', args, {
      env: {
        ...process.env,
        AMPLIFIER_CONTEXT: JSON.stringify(context)
      }
    });

    let stdout = '';
    let stderr = '';

    process.stdout.on('data', (data) => stdout += data);
    process.stderr.on('data', (data) => stderr += data);

    process.on('close', (code) => {
      if (code === 0) {
        resolve(JSON.parse(stdout));
      } else {
        reject(new Error(stderr));
      }
    });
  });
}

// Auto updater
function checkForUpdates() {
  autoUpdater.checkForUpdatesAndNotify();

  autoUpdater.on('update-available', () => {
    mainWindow?.webContents.send('update:available');
  });

  autoUpdater.on('update-downloaded', () => {
    mainWindow?.webContents.send('update:ready');
  });
}
```

### Preload Script

```typescript
// src/main/preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('amplifier', {
  // Execution
  execute: (prompt: string, profile?: string, context?: any) =>
    ipcRenderer.invoke('amplifier:execute', { prompt, profile, context }),

  executeStream: (prompt: string, profile?: string, context?: any) => {
    ipcRenderer.invoke('amplifier:execute-stream', { prompt, profile, context });
    return {
      onData: (callback: (data: string) => void) => {
        ipcRenderer.on('amplifier:stream-data', (_, data) => callback(data));
      },
      onEnd: (callback: (result: any) => void) => {
        ipcRenderer.on('amplifier:stream-end', (_, result) => callback(result));
      }
    };
  },

  // Collections
  listCollections: () => ipcRenderer.invoke('collections:list'),
  installCollection: (source: string) =>
    ipcRenderer.invoke('collections:install', { source }),

  // Profiles
  listProfiles: () => ipcRenderer.invoke('profiles:list'),

  // Navigation
  onNavigate: (callback: (path: string) => void) => {
    ipcRenderer.on('navigate', (_, path) => callback(path));
  },

  // Updates
  onUpdateAvailable: (callback: () => void) => {
    ipcRenderer.on('update:available', callback);
  },
  onUpdateReady: (callback: () => void) => {
    ipcRenderer.on('update:ready', callback);
  },
  installUpdate: () => ipcRenderer.invoke('update:install')
});
```

### Renderer - Chat Component

```typescript
// src/renderer/components/Chat.tsx
import React, { useState, useRef, useEffect } from 'react';
import { useStore } from '../store';
import { MessageList } from './MessageList';
import { ChatInput } from './ChatInput';
import { SessionSidebar } from './SessionSidebar';

export function Chat() {
  const [input, setInput] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const {
    currentSession,
    messages,
    addMessage,
    updateLastMessage,
    profile
  } = useStore();

  const handleSend = async () => {
    if (!input.trim()) return;

    const userMessage = { role: 'user', content: input };
    addMessage(userMessage);
    setInput('');
    setIsStreaming(true);

    // Add placeholder for assistant message
    addMessage({ role: 'assistant', content: '' });

    // Stream response
    const stream = window.amplifier.executeStream(input, profile, {
      session_id: currentSession?.id
    });

    let fullContent = '';

    stream.onData((data: string) => {
      try {
        const parsed = JSON.parse(data);
        if (parsed.content) {
          fullContent += parsed.content;
          updateLastMessage({ role: 'assistant', content: fullContent });
        }
      } catch {
        // Partial JSON, ignore
      }
    });

    stream.onEnd(() => {
      setIsStreaming(false);
    });
  };

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  return (
    <div className="flex h-full">
      <SessionSidebar />

      <div className="flex-1 flex flex-col">
        <div className="flex-1 overflow-y-auto p-4">
          <MessageList messages={messages} isStreaming={isStreaming} />
          <div ref={messagesEndRef} />
        </div>

        <ChatInput
          value={input}
          onChange={setInput}
          onSend={handleSend}
          disabled={isStreaming}
        />
      </div>
    </div>
  );
}
```

### Quick Query Window

```typescript
// src/renderer/QuickQuery.tsx
import React, { useState, useEffect } from 'react';

export function QuickQuery() {
  const [query, setQuery] = useState('');
  const [recentQueries, setRecentQueries] = useState<string[]>([]);

  useEffect(() => {
    // Load recent queries from store
    const recent = JSON.parse(localStorage.getItem('recentQueries') || '[]');
    setRecentQueries(recent);

    // Focus input
    document.getElementById('query-input')?.focus();

    // Handle escape
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        window.close();
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, []);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    if (!query.trim()) return;

    // Save to recent
    const newRecent = [query, ...recentQueries.slice(0, 4)];
    localStorage.setItem('recentQueries', JSON.stringify(newRecent));

    // Execute and show in main window
    // For now, open main window with query
    window.amplifier.execute(query);
    window.close();
  };

  return (
    <div className="quick-query-container">
      <div className="quick-query-header">
        🤖 Amplifier Quick Query
      </div>

      <form onSubmit={handleSubmit}>
        <input
          id="query-input"
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Ask about your code..."
          className="quick-query-input"
        />
      </form>

      {recentQueries.length > 0 && (
        <div className="recent-queries">
          <div className="recent-label">Recent:</div>
          {recentQueries.map((q, i) => (
            <button
              key={i}
              className="recent-item"
              onClick={() => {
                setQuery(q);
                handleSubmit(new Event('submit') as any);
              }}
            >
              • {q}
            </button>
          ))}
        </div>
      )}

      <div className="quick-query-footer">
        [↵ to search] [⎋ to close] [⌘+↵ open in main window]
      </div>
    </div>
  );
}
```

---

## Offline Mode

### Local Model Integration

```typescript
// src/main/local-models.ts
import { spawn } from 'child_process';
import path from 'path';

export class LocalModelManager {
  private ollamaProcess: ChildProcess | null = null;
  private modelsPath: string;

  constructor() {
    this.modelsPath = path.join(app.getPath('userData'), 'models');
  }

  async startOllama(): Promise<void> {
    this.ollamaProcess = spawn('ollama', ['serve'], {
      env: { ...process.env, OLLAMA_MODELS: this.modelsPath }
    });
  }

  async executeLocal(prompt: string, model: string = 'codellama'): Promise<string> {
    const response = await fetch('http://localhost:11434/api/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model,
        prompt,
        stream: false
      })
    });

    const data = await response.json();
    return data.response;
  }

  async downloadModel(model: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const process = spawn('ollama', ['pull', model]);
      process.on('close', (code) => {
        if (code === 0) resolve();
        else reject(new Error(`Failed to download ${model}`));
      });
    });
  }

  getAvailableModels(): string[] {
    // List downloaded models
    return ['codellama', 'mistral'];
  }

  async stop(): Promise<void> {
    this.ollamaProcess?.kill();
  }
}
```

---

## Distribution

### Build Configuration

```yaml
# electron-builder.yml
appId: com.amplifier.desktop
productName: Amplifier
directories:
  output: dist
  buildResources: build

files:
  - dist/**/*
  - package.json

mac:
  category: public.app-category.developer-tools
  icon: build/icon.icns
  hardenedRuntime: true
  gatekeeperAssess: false
  entitlements: build/entitlements.mac.plist
  entitlementsInherit: build/entitlements.mac.plist
  target:
    - dmg
    - zip

win:
  icon: build/icon.ico
  target:
    - nsis
    - portable

linux:
  icon: build/icon.png
  category: Development
  target:
    - AppImage
    - deb
    - rpm

nsis:
  oneClick: false
  allowToChangeInstallationDirectory: true

publish:
  provider: github
  owner: microsoft
  repo: amplifier-desktop
```

### Auto-Update

```typescript
// Auto-update configuration
import { autoUpdater } from 'electron-updater';

autoUpdater.autoDownload = true;
autoUpdater.autoInstallOnAppQuit = true;

// Check for updates every 4 hours
setInterval(() => {
  autoUpdater.checkForUpdates();
}, 4 * 60 * 60 * 1000);
```

---

## Configuration

```yaml
# App settings (electron-store)
settings:
  general:
    startOnLogin: true
    minimizeToTray: true
    theme: system  # light | dark | system

  shortcuts:
    quickQuery: CmdOrCtrl+Space
    toggleWindow: CmdOrCtrl+Shift+A
    newSession: CmdOrCtrl+N

  amplifier:
    defaultProfile: enterprise-dev
    cliPath: /usr/local/bin/amplifier

  offline:
    enabled: true
    defaultModel: codellama
    autoDownloadModels: true

  updates:
    autoCheck: true
    autoInstall: true
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `desktop:query_sent` | Query submitted | source (main/quick) |
| `desktop:session_created` | New session | session_id, project |
| `desktop:offline_mode` | Switched to offline | model |
| `desktop:update_installed` | App updated | version |

---

## Open Questions

1. **Code indexing**: Local embedding for offline semantic search?
2. **Plugin system**: Allow third-party plugins?
3. **Cloud sync**: Sync sessions across devices?
4. **IDE integration**: Deep links to open files in IDE?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
