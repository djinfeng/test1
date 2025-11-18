## 目标
- 在 `d:\test6\test1\main.js` 中插入 1 段代码，所有新建窗口自动打开 DevTools（分离模式）。

## 操作步骤
1. 打开 `d:\test6\test1\main.js`，搜索 `from 'electron'` 的导入位置。
2. 根据导入写法插入对应代码：
   - 若是 `import { app as mie } from 'electron'`：
     ```js
     mie.on('browser-window-created', (e, win) => {
       win.webContents.once('did-finish-load', () => win.webContents.openDevTools({ mode: 'detach' }))
     })
     ```
   - 若是 `import { app } from 'electron'`：
     ```js
     app.on('browser-window-created', (e, win) => {
       win.webContents.once('did-finish-load', () => win.webContents.openDevTools({ mode: 'detach' }))
     })
     ```
3. 保存文件，重新启动 `Trae.exe`，DevTools 会自动弹出。

## 备用快速法
- 如果未改代码，尝试快捷键 `Ctrl+Shift+I`（部分 Electron 应用开启了此快捷键）。
- 或在创建窗口后统一打开：
  ```js
  require('electron').BrowserWindow.getAllWindows().forEach(w => w.webContents.openDevTools({ mode: 'detach' }))
  ```

## 验证
- 打开后在 DevTools 的 Console 输入选择器，确认能获取到目标按钮并监听变化。