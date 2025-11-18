## 目标
- 监控两类变化：
  - 编辑器内文档变化：脏态、保存、关闭。
  - 磁盘级文件变化：创建、修改、删除（包括外部程序改动）。
- 可配置监控范围：单文件 `d:\test6\test1\text.txt` 或整个工作区目录。
- 在界面中实时显示状态（状态栏/输出面板/提示）。

## 技术路线
- 首选：VSCode 扩展实现（与套壳 VSCode 同生态，API稳定）。
  - 使用 `workspace.onDidChangeTextDocument`、`onDidSaveTextDocument`、`onDidCloseTextDocument` 获得编辑器层事件。
  - 使用 `workspace.createFileSystemWatcher` 获得磁盘变化事件。
  - UI 用 `StatusBarItem` 与 `OutputChannel` 展示当前状态与事件日志。
- 备选：Electron 主进程文件监控（若不便做扩展）。
  - 使用 `chokidar` 或 `fs.watch/fs.watchFile` 监听目录；通过 IPC 将事件推送到渲染层。
  - 渲染层更新界面元素显示状态。

## 扩展实现步骤
1. 初始化扩展包结构与 `package.json`，声明 `activationEvents` 与入口 `main`。
2. 在 `activate` 注册编辑器事件与文件系统事件，维护轻量状态并输出到状态栏与输出面板。
3. 支持过滤：仅监控目标目录或文件模式（如 `**/*.txt`），避免全局监听带来性能压力。
4. 增加命令：开启/关闭监控、查看最近事件。
5. 防抖与合并：对短时间内的连续写入做 150–300ms 合并，避免刷屏。

## 扩展示例接口（无注释版）
```ts
import * as vscode from 'vscode'
export function activate(ctx: vscode.ExtensionContext) {
  const channel = vscode.window.createOutputChannel('Watcher')
  const status = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Left, 100)
  status.text = 'Watcher: idle'
  status.show()
  const watcher = vscode.workspace.createFileSystemWatcher('**/*')
  const update = (t: string, p: string) => { channel.appendLine(`${t} ${p}`); status.text = `${t}` }
  ctx.subscriptions.push(
    vscode.workspace.onDidChangeTextDocument(e => update('dirty', e.document.uri.fsPath)),
    vscode.workspace.onDidSaveTextDocument(doc => update('saved', doc.uri.fsPath)),
    vscode.workspace.onDidCloseTextDocument(doc => update('closed', doc.uri.fsPath)),
    watcher.onDidChange(uri => update('change', uri.fsPath)),
    watcher.onDidCreate(uri => update('create', uri.fsPath)),
    watcher.onDidDelete(uri => update('delete', uri.fsPath)),
    watcher, status, channel
  )
}
```

## Electron 备选实现步骤
1. 主进程创建目录监控，推荐 `chokidar` 并启用 `awaitWriteFinish`。
2. 通过 `webContents.send` 推送事件到渲染层，渲染层用 `ipcRenderer.on` 更新 UI。

## Electron 示例（无注释版）
```js
const { app, BrowserWindow } = require('electron')
const chokidar = require('chokidar')
function startWatch(win, pattern) {
  const w = chokidar.watch(pattern, { ignoreInitial: true, awaitWriteFinish: { stabilityThreshold: 200, pollInterval: 100 } })
  const send = (type, path) => win.webContents.send('watcher:event', { type, path, ts: Date.now() })
  w.on('add', p => send('add', p))
  w.on('change', p => send('change', p))
  w.on('unlink', p => send('unlink', p))
  return w
}
app.whenReady().then(() => { const win = new BrowserWindow({}); startWatch(win, 'd:/test6/test1/**/*') })
```
渲染层：
```js
const { ipcRenderer } = require('electron')
ipcRenderer.on('watcher:event', (_, e) => { document.body.innerText = `${e.type} ${e.path}` })
```

## 配置与性能
- 目录范围与文件模式可配置，默认限制到工作区目录以避免过大监听。
- Windows 上建议避免网络映射盘；必要时使用 `fs.watchFile` 轮询或提高 `awaitWriteFinish` 窗口。
- 大量事件时仅保留最近 N 条到输出面板，状态栏显示摘要。

## 验证方案
- 在 `d:\test6\test1\text.txt` 中键入、保存、关闭，并用外部程序改动同一文件。
- 期望：状态栏及时更新为 `dirty/saved/change`；输出面板记录事件顺序与路径。
- 压测：批量写入与重命名，观察防抖是否生效与 UI 是否流畅。

## 交付物
- 一个可安装的 VSCode 扩展（源码与打包），或集成到现有 Electron 主进程的监控模块。
- 可选配置：监控目录、文件模式、防抖窗口、是否记录到文件、是否弹出通知。

## 下一步
- 你确认采用“扩展”或“Electron”哪条路线后，我将按上面步骤落地并为 `d:\test6\test1\text.txt` 预置监控，接着演示验证流程。🙂