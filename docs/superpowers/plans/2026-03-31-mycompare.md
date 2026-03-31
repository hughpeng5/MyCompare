# myCompare Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Windows desktop text diff tool with multi-tab support, line-level and character-level diff highlighting, packaged as a portable exe.

**Architecture:** Electron 28 main process manages the window and CLI args; React 18 renderer handles all UI; Monaco's built-in `createDiffEditor` provides diff computation and highlighting with zero custom diff logic. Tab state lives in a single `useReducer` hook in the renderer.

**Tech Stack:** Electron 28, React 18, @monaco-editor/react, electron-vite, electron-builder (portable exe target)

---

## File Map

| File | Responsibility |
|------|---------------|
| `electron/main.js` | BrowserWindow creation, app menu, CLI arg parsing, IPC |
| `src/main.jsx` | React entry point, renders `<App>` |
| `src/App.jsx` | Root layout: composes TabBar + Toolbar + DiffPanel + StatusBar |
| `src/hooks/useTabs.js` | All tab state via `useReducer`: add, close, switch, update content/options |
| `src/components/TabBar.jsx` | Renders tab list + "+" button; emits add/close/switch events |
| `src/components/Toolbar.jsx` | Prev/Next diff buttons, diff count display, ignoreWhitespace/ignoreCase toggles |
| `src/components/DiffPanel.jsx` | Renders Monaco DiffEditor for active tab; exposes `goToPrevDiff`/`goToNextDiff` via ref; tracks diff count |
| `src/components/StatusBar.jsx` | Version string + placeholder encoding info |
| `src/index.css` | Global styles: layout, tab bar, toolbar, dark theme shell |
| `vite.config.js` | electron-vite config |
| `package.json` | Scripts, dependencies, electron-builder config |

---

## Task 1: Project Scaffold

**Files:**
- Create: `package.json`
- Create: `vite.config.js`
- Create: `electron/main.js`
- Create: `src/main.jsx`
- Create: `src/App.jsx`
- Create: `src/index.css`

- [ ] **Step 1: Initialize project**

```bash
cd D:/MyProject/myCompare
npm init -y
```

- [ ] **Step 2: Install dependencies**

```bash
npm install react react-dom @monaco-editor/react
npm install -D electron electron-vite electron-builder vite @vitejs/plugin-react
```

- [ ] **Step 3: Create `vite.config.js`**

```js
import { defineConfig } from 'electron-vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  main: {
    build: {
      rollupOptions: {
        input: 'electron/main.js'
      }
    }
  },
  preload: {
    build: {
      rollupOptions: {
        input: { index: 'electron/preload.js' }
      }
    }
  },
  renderer: {
    plugins: [react()]
  }
})
```

- [ ] **Step 4: Create `electron/preload.js`** (minimal, required by electron-vite)

```js
// preload.js — no IPC needed for this app
```

- [ ] **Step 5: Create `electron/main.js`**

```js
import { app, BrowserWindow, Menu } from 'electron'
import { join } from 'path'
import fs from 'fs'

function createWindow() {
  const win = new BrowserWindow({
    width: 1280,
    height: 800,
    minWidth: 800,
    minHeight: 500,
    title: 'myCompare',
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      contextIsolation: true,
    }
  })

  // Parse CLI file args (argv[2] and argv[3] after filtering electron internals)
  const args = process.argv.slice(app.isPackaged ? 1 : 2).filter(a => !a.startsWith('-') && a !== '.')
  const file1 = args[0] || null
  const file2 = args[1] || null
  const initialFiles = {
    left: file1 && fs.existsSync(file1) ? fs.readFileSync(file1, 'utf-8') : '',
    right: file2 && fs.existsSync(file2) ? fs.readFileSync(file2, 'utf-8') : '',
  }

  if (process.env.NODE_ENV === 'development') {
    win.loadURL('http://localhost:5173')
  } else {
    win.loadFile(join(__dirname, '../renderer/index.html'))
  }

  // Pass initial file content to renderer after load
  win.webContents.once('did-finish-load', () => {
    win.webContents.send('init-files', initialFiles)
  })

  Menu.setApplicationMenu(null)
}

app.whenReady().then(createWindow)
app.on('window-all-closed', () => app.quit())
```

- [ ] **Step 6: Create `src/main.jsx`**

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(<App />)
```

- [ ] **Step 7: Create `src/App.jsx`** (skeleton only — components filled in later tasks)

```jsx
import React from 'react'

export default function App() {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100vh', background: '#1e1e1e', color: '#d4d4d4' }}>
      <div style={{ padding: '8px', borderBottom: '1px solid #333' }}>myCompare loading...</div>
    </div>
  )
}
```

- [ ] **Step 8: Create `src/index.css`**

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html, body, #root { height: 100%; overflow: hidden; font-family: 'Segoe UI', sans-serif; }
body { background: #1e1e1e; color: #d4d4d4; }
```

- [ ] **Step 9: Update `package.json` scripts and electron-builder config**

Replace the `scripts` and add `build` config in `package.json`:

```json
{
  "name": "mycompare",
  "version": "1.0.0",
  "description": "Text diff tool",
  "main": "dist/main/main.js",
  "scripts": {
    "dev": "electron-vite dev",
    "build": "electron-vite build && electron-builder",
    "preview": "electron-vite preview"
  },
  "build": {
    "appId": "com.mycompare.app",
    "productName": "myCompare",
    "win": {
      "target": [{ "target": "portable", "arch": ["x64"] }]
    },
    "directories": {
      "output": "release"
    },
    "files": ["dist/**/*"]
  }
}
```

- [ ] **Step 10: Create `index.html`** in project root (electron-vite renderer entry)

```html
<!DOCTYPE html>
<html lang="zh">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>myCompare</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

- [ ] **Step 11: Verify dev server starts**

```bash
npm run dev
```

Expected: Electron window opens showing "myCompare loading..."

- [ ] **Step 12: Commit**

```bash
git add .
git commit -m "feat: scaffold Electron + React + electron-vite project"
```

---

## Task 2: Tab State Hook (`useTabs`)

**Files:**
- Create: `src/hooks/useTabs.js`

- [ ] **Step 1: Create `src/hooks/useTabs.js`**

```js
import { useReducer, useCallback } from 'react'

let nextId = 1

function createTab(leftContent = '', rightContent = '') {
  const id = String(nextId++)
  return {
    id,
    label: `对比 ${id}`,
    leftContent,
    rightContent,
    ignoreWhitespace: false,
    ignoreCase: false,
  }
}

function reducer(state, action) {
  switch (action.type) {
    case 'ADD_TAB': {
      const tab = createTab(action.leftContent, action.rightContent)
      return { tabs: [...state.tabs, tab], activeTabId: tab.id }
    }
    case 'CLOSE_TAB': {
      const remaining = state.tabs.filter(t => t.id !== action.id)
      if (remaining.length === 0) {
        const tab = createTab()
        return { tabs: [tab], activeTabId: tab.id }
      }
      const activeTabId = state.activeTabId === action.id
        ? remaining[remaining.length - 1].id
        : state.activeTabId
      return { tabs: remaining, activeTabId }
    }
    case 'SWITCH_TAB':
      return { ...state, activeTabId: action.id }
    case 'UPDATE_CONTENT': {
      const tabs = state.tabs.map(t =>
        t.id === action.id
          ? { ...t, leftContent: action.leftContent, rightContent: action.rightContent }
          : t
      )
      return { ...state, tabs }
    }
    case 'UPDATE_OPTIONS': {
      const tabs = state.tabs.map(t =>
        t.id === action.id
          ? { ...t, ignoreWhitespace: action.ignoreWhitespace, ignoreCase: action.ignoreCase }
          : t
      )
      return { ...state, tabs }
    }
    default:
      return state
  }
}

export function useTabs(initialLeft = '', initialRight = '') {
  const initial = createTab(initialLeft, initialRight)
  const [state, dispatch] = useReducer(reducer, { tabs: [initial], activeTabId: initial.id })

  const addTab = useCallback(() => {
    if (state.tabs.length >= 10) return
    dispatch({ type: 'ADD_TAB', leftContent: '', rightContent: '' })
  }, [state.tabs.length])

  const closeTab = useCallback((id) => dispatch({ type: 'CLOSE_TAB', id }), [])
  const switchTab = useCallback((id) => dispatch({ type: 'SWITCH_TAB', id }), [])
  const updateContent = useCallback((id, leftContent, rightContent) =>
    dispatch({ type: 'UPDATE_CONTENT', id, leftContent, rightContent }), [])
  const updateOptions = useCallback((id, ignoreWhitespace, ignoreCase) =>
    dispatch({ type: 'UPDATE_OPTIONS', id, ignoreWhitespace, ignoreCase }), [])

  const activeTab = state.tabs.find(t => t.id === state.activeTabId)

  return { tabs: state.tabs, activeTabId: state.activeTabId, activeTab, addTab, closeTab, switchTab, updateContent, updateOptions }
}
```

- [ ] **Step 2: Commit**

```bash
git add src/hooks/useTabs.js
git commit -m "feat: add useTabs hook with add/close/switch/update reducers"
```

---

## Task 3: TabBar Component

**Files:**
- Create: `src/components/TabBar.jsx`

- [ ] **Step 1: Create `src/components/TabBar.jsx`**

```jsx
import React from 'react'

const tabBarStyle = {
  display: 'flex',
  alignItems: 'flex-end',
  background: '#252526',
  borderBottom: '1px solid #444',
  padding: '0 8px',
  gap: '2px',
  flexShrink: 0,
  overflowX: 'auto',
}

const tabStyle = (active) => ({
  padding: '6px 12px',
  paddingRight: '6px',
  background: active ? '#1e1e1e' : '#2d2d2d',
  color: active ? '#fff' : '#888',
  border: '1px solid',
  borderColor: active ? '#555' : '#333',
  borderBottom: active ? '1px solid #1e1e1e' : '1px solid #333',
  borderRadius: '4px 4px 0 0',
  fontSize: '13px',
  cursor: 'pointer',
  display: 'flex',
  alignItems: 'center',
  gap: '6px',
  userSelect: 'none',
  whiteSpace: 'nowrap',
})

const closeBtn = {
  background: 'none',
  border: 'none',
  color: '#888',
  cursor: 'pointer',
  fontSize: '14px',
  lineHeight: 1,
  padding: '0 2px',
  borderRadius: '3px',
}

const addBtn = {
  background: 'none',
  border: 'none',
  color: '#888',
  cursor: 'pointer',
  fontSize: '20px',
  padding: '0 8px',
  lineHeight: 1,
  alignSelf: 'center',
}

export default function TabBar({ tabs, activeTabId, onSwitch, onAdd, onClose }) {
  return (
    <div style={tabBarStyle}>
      {tabs.map(tab => (
        <div
          key={tab.id}
          style={tabStyle(tab.id === activeTabId)}
          onClick={() => onSwitch(tab.id)}
        >
          {tab.label}
          <button
            style={closeBtn}
            onClick={e => { e.stopPropagation(); onClose(tab.id) }}
            title="关闭"
          >×</button>
        </div>
      ))}
      {tabs.length < 10 && (
        <button style={addBtn} onClick={onAdd} title="新建对比">+</button>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Wire TabBar into App.jsx**

Replace `src/App.jsx`:

```jsx
import React from 'react'
import TabBar from './components/TabBar'
import { useTabs } from './hooks/useTabs'

export default function App() {
  const { tabs, activeTabId, activeTab, addTab, closeTab, switchTab, updateContent, updateOptions } = useTabs()

  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100vh', background: '#1e1e1e', color: '#d4d4d4', overflow: 'hidden' }}>
      <TabBar
        tabs={tabs}
        activeTabId={activeTabId}
        onSwitch={switchTab}
        onAdd={addTab}
        onClose={closeTab}
      />
      <div style={{ flex: 1, display: 'flex', alignItems: 'center', justifyContent: 'center', color: '#555' }}>
        Toolbar + DiffPanel coming soon
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Verify in dev — tabs can be added (up to 10), switched, and closed**

```bash
npm run dev
```

Expected: Tab bar renders, clicking + adds tabs, clicking × closes them, switching highlights active tab.

- [ ] **Step 4: Commit**

```bash
git add src/components/TabBar.jsx src/App.jsx
git commit -m "feat: add TabBar component with add/close/switch"
```

---

## Task 4: DiffPanel Component (Monaco Diff Editor)

**Files:**
- Create: `src/components/DiffPanel.jsx`

- [ ] **Step 1: Create `src/components/DiffPanel.jsx`**

```jsx
import React, { useRef, useEffect, useImperativeHandle, forwardRef, useState } from 'react'
import { DiffEditor } from '@monaco-editor/react'

const headerStyle = {
  display: 'flex',
  background: '#252526',
  borderBottom: '1px solid #333',
  flexShrink: 0,
}

const headerCell = {
  flex: 1,
  padding: '4px 12px',
  fontSize: '12px',
  color: '#9cdcfe',
  borderRight: '1px solid #333',
}

export default forwardRef(function DiffPanel(
  { tab, onContentChange },
  ref
) {
  const editorRef = useRef(null)
  const [diffCount, setDiffCount] = useState(0)

  useImperativeHandle(ref, () => ({
    goToPrevDiff() {
      editorRef.current?.getModifiedEditor().trigger('keyboard', 'editor.action.diffReview.prev', null)
    },
    goToNextDiff() {
      editorRef.current?.getModifiedEditor().trigger('keyboard', 'editor.action.diffReview.next', null)
    },
    getDiffCount() {
      return diffCount
    }
  }))

  function handleEditorMount(editor) {
    editorRef.current = editor

    // Track diff count whenever the diff model changes
    editor.onDidUpdateDiff(() => {
      const changes = editor.getLineChanges()
      setDiffCount(changes ? changes.length : 0)
    })

    // Sync left (original) edits back to state
    editor.getOriginalEditor().onDidChangeModelContent(() => {
      const left = editor.getOriginalEditor().getValue()
      const right = editor.getModifiedEditor().getValue()
      onContentChange(left, right)
    })

    // Sync right (modified) edits back to state
    editor.getModifiedEditor().onDidChangeModelContent(() => {
      const left = editor.getOriginalEditor().getValue()
      const right = editor.getModifiedEditor().getValue()
      onContentChange(left, right)
    })
  }

  // Compute values for diff — if ignoreCase, compare lowercased but display originals
  const displayLeft = tab.leftContent
  const displayRight = tab.rightContent

  return (
    <div style={{ display: 'flex', flexDirection: 'column', flex: 1, overflow: 'hidden' }}>
      <div style={headerStyle}>
        <div style={headerCell}>左侧内容</div>
        <div style={{ ...headerCell, borderRight: 'none' }}>右侧内容</div>
      </div>
      <div style={{ flex: 1, overflow: 'hidden' }}>
        <DiffEditor
          height="100%"
          theme="vs-dark"
          original={displayLeft}
          modified={displayRight}
          options={{
            renderSideBySide: true,
            ignoreTrimWhitespace: tab.ignoreWhitespace,
            readOnly: false,
            originalEditable: true,
            minimap: { enabled: false },
            wordWrap: 'off',
            scrollBeyondLastLine: false,
            fontSize: 14,
            lineNumbers: 'on',
            diffCodeLens: false,
            renderOverviewRuler: false,
          }}
          onMount={handleEditorMount}
        />
      </div>
    </div>
  )
})
```

- [ ] **Step 2: Add ignoreCase handling note**

The `ignoreCase` option cannot be passed to Monaco directly — it must be handled by normalizing content before diff. Update `DiffPanel.jsx` to pass normalized content to Monaco while keeping original content in state. Replace the `displayLeft`/`displayRight` block:

```jsx
  // If ignoreCase, pass lowercased content to Monaco for diff computation
  // Monaco shows the lowercased text; original case is preserved in state
  const displayLeft = tab.ignoreCase ? tab.leftContent.toLowerCase() : tab.leftContent
  const displayRight = tab.ignoreCase ? tab.rightContent.toLowerCase() : tab.rightContent
```

- [ ] **Step 3: Wire DiffPanel into App.jsx**

Replace `src/App.jsx`:

```jsx
import React, { useRef } from 'react'
import TabBar from './components/TabBar'
import DiffPanel from './components/DiffPanel'
import { useTabs } from './hooks/useTabs'

export default function App() {
  const { tabs, activeTabId, activeTab, addTab, closeTab, switchTab, updateContent, updateOptions } = useTabs()
  const diffPanelRef = useRef(null)

  function handleContentChange(left, right) {
    updateContent(activeTabId, left, right)
  }

  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100vh', background: '#1e1e1e', color: '#d4d4d4', overflow: 'hidden' }}>
      <TabBar
        tabs={tabs}
        activeTabId={activeTabId}
        onSwitch={switchTab}
        onAdd={addTab}
        onClose={closeTab}
      />
      <div style={{ flex: 1, display: 'flex', flexDirection: 'column', overflow: 'hidden' }}>
        {activeTab && (
          <DiffPanel
            key={activeTabId}
            tab={activeTab}
            onContentChange={handleContentChange}
            ref={diffPanelRef}
          />
        )}
      </div>
    </div>
  )
}
```

Note: `key={activeTabId}` causes Monaco to remount per tab — this is intentional to isolate editor state per tab cleanly.

- [ ] **Step 4: Verify in dev — paste text in left and right, diff highlights appear**

```bash
npm run dev
```

Expected: Monaco Diff Editor renders, typing in either pane shows line-level and character-level highlights.

- [ ] **Step 5: Commit**

```bash
git add src/components/DiffPanel.jsx src/App.jsx
git commit -m "feat: add DiffPanel with Monaco DiffEditor, diff count tracking"
```

---

## Task 5: Toolbar Component

**Files:**
- Create: `src/components/Toolbar.jsx`

- [ ] **Step 1: Create `src/components/Toolbar.jsx`**

```jsx
import React from 'react'

const toolbarStyle = {
  display: 'flex',
  alignItems: 'center',
  gap: '8px',
  padding: '5px 12px',
  background: '#1e1e1e',
  borderBottom: '1px solid #333',
  flexShrink: 0,
}

const btnStyle = {
  background: '#2d2d2d',
  border: '1px solid #555',
  color: '#d4d4d4',
  borderRadius: '3px',
  padding: '3px 10px',
  fontSize: '12px',
  cursor: 'pointer',
}

const labelStyle = {
  color: '#888',
  fontSize: '12px',
}

const checkboxLabel = {
  display: 'flex',
  alignItems: 'center',
  gap: '4px',
  color: '#aaa',
  fontSize: '12px',
  cursor: 'pointer',
}

export default function Toolbar({ diffCount, ignoreWhitespace, ignoreCase, onPrevDiff, onNextDiff, onIgnoreWhitespaceChange, onIgnoreCaseChange }) {
  return (
    <div style={toolbarStyle}>
      <button style={btnStyle} onClick={onPrevDiff} title="上一处差异 (F7)">↑ 上一处</button>
      <button style={btnStyle} onClick={onNextDiff} title="下一处差异 (F8)">↓ 下一处</button>
      <span style={labelStyle}>{diffCount > 0 ? `共 ${diffCount} 处差异` : '无差异'}</span>
      <div style={{ flex: 1 }} />
      <label style={checkboxLabel}>
        <input
          type="checkbox"
          checked={ignoreWhitespace}
          onChange={e => onIgnoreWhitespaceChange(e.target.checked)}
        />
        忽略空白
      </label>
      <label style={checkboxLabel}>
        <input
          type="checkbox"
          checked={ignoreCase}
          onChange={e => onIgnoreCaseChange(e.target.checked)}
        />
        忽略大小写
      </label>
    </div>
  )
}
```

- [ ] **Step 2: Wire Toolbar into App.jsx and hook up keyboard shortcuts**

Replace `src/App.jsx`:

```jsx
import React, { useRef, useState, useEffect } from 'react'
import TabBar from './components/TabBar'
import Toolbar from './components/Toolbar'
import DiffPanel from './components/DiffPanel'
import { useTabs } from './hooks/useTabs'

export default function App() {
  const { tabs, activeTabId, activeTab, addTab, closeTab, switchTab, updateContent, updateOptions } = useTabs()
  const diffPanelRef = useRef(null)
  const [diffCount, setDiffCount] = useState(0)

  function handleContentChange(left, right) {
    updateContent(activeTabId, left, right)
  }

  function handleIgnoreWhitespaceChange(val) {
    updateOptions(activeTabId, val, activeTab.ignoreCase)
  }

  function handleIgnoreCaseChange(val) {
    updateOptions(activeTabId, activeTab.ignoreWhitespace, val)
  }

  // Expose diffCount from DiffPanel via polling the ref
  // DiffPanel updates diffCount internally; we read it on each render via callback
  function handleDiffCountChange(count) {
    setDiffCount(count)
  }

  // F7/F8 keyboard shortcuts
  useEffect(() => {
    function onKeyDown(e) {
      if (e.key === 'F7') { e.preventDefault(); diffPanelRef.current?.goToPrevDiff() }
      if (e.key === 'F8') { e.preventDefault(); diffPanelRef.current?.goToNextDiff() }
    }
    window.addEventListener('keydown', onKeyDown)
    return () => window.removeEventListener('keydown', onKeyDown)
  }, [])

  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100vh', background: '#1e1e1e', color: '#d4d4d4', overflow: 'hidden' }}>
      <TabBar
        tabs={tabs}
        activeTabId={activeTabId}
        onSwitch={switchTab}
        onAdd={addTab}
        onClose={closeTab}
      />
      <Toolbar
        diffCount={diffCount}
        ignoreWhitespace={activeTab?.ignoreWhitespace ?? false}
        ignoreCase={activeTab?.ignoreCase ?? false}
        onPrevDiff={() => diffPanelRef.current?.goToPrevDiff()}
        onNextDiff={() => diffPanelRef.current?.goToNextDiff()}
        onIgnoreWhitespaceChange={handleIgnoreWhitespaceChange}
        onIgnoreCaseChange={handleIgnoreCaseChange}
      />
      <div style={{ flex: 1, display: 'flex', flexDirection: 'column', overflow: 'hidden' }}>
        {activeTab && (
          <DiffPanel
            key={activeTabId}
            tab={activeTab}
            onContentChange={handleContentChange}
            onDiffCountChange={handleDiffCountChange}
            ref={diffPanelRef}
          />
        )}
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Update DiffPanel to call `onDiffCountChange` prop**

In `src/components/DiffPanel.jsx`, add `onDiffCountChange` to props and call it when diff count changes. Replace the `handleEditorMount` function:

```jsx
export default forwardRef(function DiffPanel(
  { tab, onContentChange, onDiffCountChange },
  ref
) {
  // ... existing code ...

  function handleEditorMount(editor) {
    editorRef.current = editor

    editor.onDidUpdateDiff(() => {
      const changes = editor.getLineChanges()
      const count = changes ? changes.length : 0
      setDiffCount(count)
      onDiffCountChange?.(count)
    })

    editor.getOriginalEditor().onDidChangeModelContent(() => {
      const left = editor.getOriginalEditor().getValue()
      const right = editor.getModifiedEditor().getValue()
      onContentChange(left, right)
    })

    editor.getModifiedEditor().onDidChangeModelContent(() => {
      const left = editor.getOriginalEditor().getValue()
      const right = editor.getModifiedEditor().getValue()
      onContentChange(left, right)
    })
  }
  // ... rest unchanged ...
})
```

- [ ] **Step 4: Verify — toolbar shows diff count, prev/next buttons navigate, F7/F8 work, checkboxes toggle options**

```bash
npm run dev
```

Expected: Paste different text in both panes. Toolbar shows "共 N 处差异". Clicking ↑/↓ or pressing F7/F8 jumps between diffs. Toggling "忽略空白" recalculates diff.

- [ ] **Step 5: Commit**

```bash
git add src/components/Toolbar.jsx src/components/DiffPanel.jsx src/App.jsx
git commit -m "feat: add Toolbar with diff navigation, F7/F8 shortcuts, ignore options"
```

---

## Task 6: StatusBar Component

**Files:**
- Create: `src/components/StatusBar.jsx`

- [ ] **Step 1: Create `src/components/StatusBar.jsx`**

```jsx
import React from 'react'

const statusStyle = {
  display: 'flex',
  justifyContent: 'space-between',
  alignItems: 'center',
  background: '#007acc',
  color: '#fff',
  padding: '2px 12px',
  fontSize: '12px',
  flexShrink: 0,
}

export default function StatusBar() {
  return (
    <div style={statusStyle}>
      <span>myCompare v1.0.0</span>
      <span>UTF-8</span>
    </div>
  )
}
```

- [ ] **Step 2: Add StatusBar to App.jsx**

In `src/App.jsx`, import and add `<StatusBar />` at the bottom of the root flex column:

```jsx
import StatusBar from './components/StatusBar'

// Inside the return, after the DiffPanel div:
      <StatusBar />
```

- [ ] **Step 3: Verify status bar appears at bottom**

```bash
npm run dev
```

Expected: Blue status bar at the bottom showing "myCompare v1.0.0" and "UTF-8".

- [ ] **Step 4: Commit**

```bash
git add src/components/StatusBar.jsx src/App.jsx
git commit -m "feat: add StatusBar"
```

---

## Task 7: CLI File Arguments (Main Process)

**Files:**
- Modify: `electron/main.js`
- Modify: `src/App.jsx`

The main process already reads CLI args and sends them via `init-files` IPC. This task adds the renderer-side listener to pre-populate the first tab.

- [ ] **Step 1: Add IPC listener in App.jsx**

In `src/App.jsx`, add a `useEffect` that listens for `init-files` from the main process. Add this import at the top:

```jsx
// Add to existing imports — works only in Electron context
const ipcRenderer = window?.electron?.ipcRenderer ?? null
```

Then add this effect inside the `App` component (after the keyboard shortcut effect):

```jsx
  useEffect(() => {
    if (!ipcRenderer) return
    ipcRenderer.once('init-files', (_event, { left, right }) => {
      if (left || right) {
        updateContent(activeTabId, left, right)
      }
    })
  }, []) // eslint-disable-line react-hooks/exhaustive-deps
```

- [ ] **Step 2: Expose ipcRenderer via preload**

Replace `electron/preload.js`:

```js
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electron', {
  ipcRenderer: {
    once: (channel, listener) => ipcRenderer.once(channel, listener),
  }
})
```

- [ ] **Step 3: Verify CLI args work**

```bash
# Create two test files
echo "hello world" > /tmp/left.txt
echo "hello earth" > /tmp/right.txt

npm run dev -- /tmp/left.txt /tmp/right.txt
```

Expected: App opens with "hello world" in left pane and "hello earth" in right pane, with "world"/"earth" highlighted as a diff.

- [ ] **Step 4: Commit**

```bash
git add electron/main.js electron/preload.js src/App.jsx
git commit -m "feat: support CLI file arguments to pre-load diff content"
```

---

## Task 8: Package as Portable exe

**Files:**
- Modify: `package.json` (verify build config)

- [ ] **Step 1: Verify electron-builder config in package.json**

Confirm `package.json` contains:

```json
"build": {
  "appId": "com.mycompare.app",
  "productName": "myCompare",
  "win": {
    "target": [{ "target": "portable", "arch": ["x64"] }]
  },
  "directories": { "output": "release" },
  "files": ["dist/**/*"]
}
```

- [ ] **Step 2: Build the app**

```bash
npm run build
```

Expected: `release/myCompare*.exe` created (portable, no installer needed).

- [ ] **Step 3: Test the exe**

Double-click `release/myCompare*.exe` — app should open.

Run from command line:
```
release\myCompare*.exe C:\path\to\file1.txt C:\path\to\file2.txt
```

Expected: App opens with both files loaded in left/right panes.

- [ ] **Step 4: Commit**

```bash
git add package.json
git commit -m "chore: verify electron-builder portable exe packaging"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] 左右双栏文本编辑 — Task 4 (DiffPanel with originalEditable)
- [x] 自动实时比对 — Task 4 (Monaco DiffEditor reacts to content changes)
- [x] 行级差异高亮 — Task 4 (Monaco native)
- [x] 字符级差异高亮 — Task 4 (Monaco native with renderSideBySide)
- [x] 多 Tab 支持 — Tasks 2 + 3
- [x] 差异导航 — Task 5 (prev/next buttons)
- [x] 差异计数 — Tasks 4 + 5
- [x] 忽略空白 — Task 5 (ignoreTrimWhitespace option)
- [x] 忽略大小写 — Tasks 4 + 5 (content normalization)
- [x] F7/F8 快捷键 — Task 5
- [x] Portable exe — Task 8
- [x] CLI 文件参数 — Task 7
- [x] 最多 10 个 Tab — Task 2 (reducer guard) + Task 3 (hide + button)
- [x] 关闭最后一个 Tab 自动新建 — Task 2 (reducer CLOSE_TAB branch)
