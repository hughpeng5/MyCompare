# myCompare — 文本比对工具设计文档

**日期：** 2026-03-31  
**技术栈：** Electron + React + Monaco Editor  
**平台：** Windows only

---

## 1. 项目概述

myCompare 是一款类似 Beyond Compare 的桌面文本比对工具，专注于文本内容比对功能。支持左右双栏输入，自动高亮行级差异和行内字符级差异，支持多 Tab 同时比对多组内容，最终打包为可双击运行的 Windows exe 文件。

---

## 2. 功能需求

### 核心功能
- **左右双栏文本编辑**：两个可编辑文本区域，用户直接输入或粘贴内容
- **自动实时比对**：内容变化时自动触发 diff 计算并更新高亮
- **行级差异高亮**：整行背景色标记（删除行红色、新增行绿色、修改行黄色）
- **字符级差异高亮**：行内具体修改的字符/词语额外高亮，精确到字符
- **多 Tab 支持**：可同时打开多个比对 Tab，Tab 间独立，支持新建、关闭、切换
- **差异导航**：工具栏"上一处"/"下一处"按钮，快速跳转到各处差异
- **差异计数**：状态栏或工具栏显示当前共有几处差异

### 辅助功能
- **忽略空白选项**：可切换是否将空格/缩进差异计入 diff
- **忽略大小写选项**：可切换大小写不敏感比对
- **键盘快捷键**：F7/F8 上下导航差异（参考 Beyond Compare 习惯）

### 打包与启动
- 打包为单个 Windows exe（使用 electron-builder）
- 支持双击直接启动
- 支持命令行启动：`mycompare.exe`，可选参数传入两个文件路径直接打开并比对

---

## 3. 架构设计

### 技术选型
| 层次 | 技术 |
|------|------|
| 桌面框架 | Electron 28+ |
| 前端框架 | React 18 |
| 编辑器 | Monaco Editor（`@monaco-editor/react`） |
| Diff 引擎 | Monaco 内置 `createDiffEditor` |
| 打包工具 | electron-builder |
| 构建工具 | Vite + electron-vite |

### 进程架构
- **Main Process（Electron）**：窗口管理、菜单、文件读写（若将来支持打开文件）、命令行参数解析
- **Renderer Process（React）**：全部 UI 逻辑，Monaco Diff Editor 实例管理，Tab 状态管理

### 组件结构

```
App
├── TitleBar（自定义标题栏，可选）
├── MenuBar（文件/视图/帮助菜单）
├── TabBar
│   ├── TabItem × N（每个比对 Tab）
│   └── AddTabButton
├── Toolbar（差异导航、选项开关）
└── DiffPanel（当前激活 Tab 的内容）
    ├── PanelHeader（左/右标签）
    └── MonacoDiffEditor（Monaco createDiffEditor）
```

### 状态管理
使用 React 内置状态（useState / useReducer），无需引入 Redux。

```
tabs: Tab[]           // 所有 Tab 列表
activeTabId: string   // 当前激活 Tab ID

Tab {
  id: string
  label: string       // "对比 1", "对比 2" ...
  leftContent: string
  rightContent: string
  ignoreWhitespace: boolean
  ignoreCase: boolean
}
```

每个 Tab 对应一个 Monaco DiffEditor 实例，切换 Tab 时切换显示，未激活的实例保留在内存中保持内容。

### Diff 计算
完全委托给 Monaco 内置 diff 引擎：
- 行级高亮：Monaco DiffEditor 原生渲染
- 字符级高亮：Monaco DiffEditor `renderSideBySide: true` + `enableSplitViewResizing` 模式下原生支持
- 忽略空白：通过 `ignoreTrimWhitespace` 选项传给 Monaco
- 忽略大小写：在传入 Monaco 前将两侧内容统一转小写（仅影响 diff 计算，不修改显示内容）

---

## 4. UI 设计

### 主窗口布局（从上到下）
1. **菜单栏**：文件 | 视图 | 帮助
2. **Tab 栏**：各 Tab 标签 + "+" 新建按钮
3. **工具栏**：`← 上一处差异` `下一处差异 →` `共 N 处差异` | `忽略空白 □` `忽略大小写 □`
4. **面板标题行**：左侧 | 右侧
5. **Monaco Diff 编辑区**：占满剩余高度，左右分栏
6. **状态栏**：版本号 | 编码信息

### 主题
默认暗色主题（Monaco `vs-dark`），与截图线框图一致。

### Tab 行为
- 最多允许 10 个 Tab
- 新 Tab 默认标签为"对比 N"（N 自增）
- Tab 右键或点击 × 关闭，关闭最后一个 Tab 时自动新建一个空 Tab

---

## 5. 打包方案

使用 `electron-builder` 打包：
- 输出：NSIS 安装包（可选）或 portable exe（直接运行，无需安装，优先）
- 打包命令：`npm run build`
- 目标：`win` `portable`

命令行参数支持：
```
mycompare.exe                         # 直接启动，空白 Tab
mycompare.exe file1.txt file2.txt     # 启动后自动加载两个文件内容到左右面板
```

Main Process 在 `app.ready` 时解析 `process.argv` 实现。

---

## 6. 项目结构

```
myCompare/
├── electron/
│   └── main.js          # Electron Main Process
├── src/
│   ├── App.jsx
│   ├── components/
│   │   ├── TabBar.jsx
│   │   ├── Toolbar.jsx
│   │   ├── DiffPanel.jsx
│   │   └── StatusBar.jsx
│   ├── hooks/
│   │   └── useTabs.js   # Tab 状态管理逻辑
│   └── main.jsx         # React 入口
├── public/
├── package.json
└── vite.config.js
```

---

## 7. 不在范围内（本版本不实现）

- 文件夹比对
- 三路合并
- 语法高亮（非 diff 相关）
- 保存/导出 diff 结果
- macOS / Linux 支持
- 云同步或历史记录
