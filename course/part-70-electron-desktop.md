# ตอนที่ 70: Electron Desktop กับ TypeScript

## บทนำ

Electron เป็น framework สำหรับสร้างแอปพลิเคชัน desktop ด้วย web technologies (HTML, CSS, JavaScript/TypeScript) โดยใช้ Chromium และ Node.js เมื่อรวมกับ TypeScript จะได้ development experience ที่ยอดเยี่ยม

## 70.1 Electron + TypeScript Setup

### การสร้างโปรเจกต์

```bash
npm create electron-vite@latest my-electron-app -- --template react-ts
cd my-electron-app
npm install
```

### โครงสร้างโปรเจกต์

```
my-electron-app/
  src/
    main/           # Main process (Node.js)
      index.ts      # Entry point
      ipc/          # IPC handlers
      services/     # Main process services
    preload/        # Preload scripts
      index.ts
    renderer/       # Renderer process (React/Vue)
      App.tsx
      components/
      hooks/
      stores/
  electron.vite.config.ts
  package.json
  tsconfig.json
```

### tsconfig.json

```json
{
  "references": [
    { "path": "./tsconfig.node.json" },
    { "path": "./tsconfig.web.json" }
  ]
}
```

```json
// tsconfig.node.json (main + preload)
{
  "extends": "@electron-toolkit/tsconfig/tsconfig.node.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "out",
    "paths": {
      "@main/*": ["./src/main/*"],
      "@preload/*": ["./src/preload/*"]
    }
  },
  "include": ["src/main/**/*", "src/preload/**/*", "electron.vite.config.*"]
}
```

## 70.2 Main Process Typing

### Window Management

```typescript
// src/main/index.ts
import { app, BrowserWindow, shell, ipcMain, Menu, Tray } from 'electron'
import { join } from 'path'
import { electronApp, optimizer, is } from '@electron-toolkit/utils'
import { createMenu } from './menu'
import { createTray } from './tray'
import { setupIpcHandlers } from './ipc'
import icon from '../../resources/icon.png?asset'

// Window configuration type
interface WindowConfig {
  width: number
  height: number
  minWidth?: number
  minHeight?: number
  title?: string
  resizable?: boolean
  frame?: boolean
}

interface AppWindows {
  main: BrowserWindow | null
  settings: BrowserWindow | null
  splash: BrowserWindow | null
}

const windows: AppWindows = {
  main: null,
  settings: null,
  splash: null
}

function createMainWindow(config: Partial<WindowConfig> = {}): BrowserWindow {
  const defaultConfig: WindowConfig = {
    width: 1200,
    height: 800,
    minWidth: 800,
    minHeight: 600,
    title: 'My Electron App'
  }
  
  const finalConfig = { ...defaultConfig, ...config }
  
  const mainWindow = new BrowserWindow({
    ...finalConfig,
    show: false, // ซ่อนก่อนจนกว่าจะพร้อม
    autoHideMenuBar: true,
    icon,
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      sandbox: false,
      contextIsolation: true,
      nodeIntegration: false // Security
    }
  })
  
  // ป้องกัน navigation ออกนอก app
  mainWindow.webContents.setWindowOpenHandler(({ url }) => {
    if (url.startsWith('https:')) {
      shell.openExternal(url)
    }
    return { action: 'deny' }
  })
  
  // โหลด URL
  if (is.dev && process.env['ELECTRON_RENDERER_URL']) {
    mainWindow.loadURL(process.env['ELECTRON_RENDERER_URL'])
  } else {
    mainWindow.loadFile(join(__dirname, '../renderer/index.html'))
  }
  
  // แสดงเมื่อ ready
  mainWindow.once('ready-to-show', () => {
    mainWindow.show()
    if (is.dev) {
      mainWindow.webContents.openDevTools()
    }
  })
  
  return mainWindow
}

app.whenReady().then(() => {
  electronApp.setAppUserModelId('com.example.myapp')
  
  // Optimize on Windows
  app.on('browser-window-created', (_, window) => {
    optimizer.watchWindowShortcuts(window)
  })
  
  windows.main = createMainWindow()
  createMenu(windows.main)
  createTray(windows.main)
  setupIpcHandlers()
  
  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      windows.main = createMainWindow()
    }
  })
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})
```

## 70.3 Renderer Process Typing

```typescript
// src/renderer/src/types/electron.d.ts
// ประกาศ types สำหรับ exposed APIs

interface ElectronAPI {
  // File operations
  openFile: (options?: OpenFileOptions) => Promise<FileResult | null>
  saveFile: (data: SaveFileData) => Promise<SaveFileResult>
  readFile: (filePath: string) => Promise<FileContent>
  watchFile: (filePath: string, callback: (event: FileEvent) => void) => () => void
  
  // System
  getSystemInfo: () => Promise<SystemInfo>
  openExternal: (url: string) => Promise<void>
  showNotification: (notification: NotificationOptions) => void
  
  // App
  getVersion: () => string
  checkForUpdates: () => Promise<UpdateCheckResult>
  
  // Events
  on: (channel: string, listener: (...args: any[]) => void) => void
  off: (channel: string, listener: (...args: any[]) => void) => void
}

interface OpenFileOptions {
  title?: string
  filters?: Array<{ name: string; extensions: string[] }>
  multiple?: boolean
}

interface FileResult {
  filePaths: string[]
  canceled: boolean
}

interface SaveFileData {
  content: string | Buffer
  filePath?: string
  filters?: Array<{ name: string; extensions: string[] }>
  title?: string
}

interface SaveFileResult {
  filePath: string | undefined
  canceled: boolean
  success: boolean
}

interface FileContent {
  content: string
  filePath: string
  encoding: string
  size: number
}

interface FileEvent {
  type: 'change' | 'rename'
  filePath: string
}

interface SystemInfo {
  platform: NodeJS.Platform
  arch: string
  version: string
  memory: {
    total: number
    free: number
    used: number
  }
  cpu: {
    model: string
    cores: number
    speed: number
  }
}

interface UpdateCheckResult {
  updateAvailable: boolean
  currentVersion: string
  latestVersion?: string
  releaseNotes?: string
}

// Extend Window interface
declare global {
  interface Window {
    electron: ElectronAPI
  }
}
```

## 70.4 IPC Communication พร้อม Types

### IPC Channels Definition

```typescript
// src/types/ipc.ts
// ประกาศ IPC channels และ payload types

export const IpcChannels = {
  // File operations
  OPEN_FILE: 'file:open',
  SAVE_FILE: 'file:save',
  READ_FILE: 'file:read',
  DELETE_FILE: 'file:delete',
  WATCH_FILE: 'file:watch',
  UNWATCH_FILE: 'file:unwatch',
  
  // System
  GET_SYSTEM_INFO: 'system:info',
  SHOW_NOTIFICATION: 'system:notification',
  OPEN_EXTERNAL: 'system:openExternal',
  
  // App
  GET_VERSION: 'app:version',
  CHECK_UPDATES: 'app:checkUpdates',
  INSTALL_UPDATE: 'app:installUpdate',
  
  // Settings
  GET_SETTINGS: 'settings:get',
  SET_SETTINGS: 'settings:set',
  RESET_SETTINGS: 'settings:reset',
  
  // Events (main -> renderer)
  FILE_CHANGED: 'event:fileChanged',
  UPDATE_AVAILABLE: 'event:updateAvailable',
  SETTINGS_CHANGED: 'event:settingsChanged'
} as const

export type IpcChannel = typeof IpcChannels[keyof typeof IpcChannels]

// Type mapping สำหรับ request/response
export interface IpcInvokeMap {
  [IpcChannels.OPEN_FILE]: {
    request: OpenFileOptions
    response: FileResult | null
  }
  [IpcChannels.SAVE_FILE]: {
    request: SaveFileData
    response: SaveFileResult
  }
  [IpcChannels.READ_FILE]: {
    request: { filePath: string }
    response: FileContent
  }
  [IpcChannels.GET_SYSTEM_INFO]: {
    request: void
    response: SystemInfo
  }
  [IpcChannels.CHECK_UPDATES]: {
    request: void
    response: UpdateCheckResult
  }
  [IpcChannels.GET_SETTINGS]: {
    request: void
    response: AppSettings
  }
  [IpcChannels.SET_SETTINGS]: {
    request: Partial<AppSettings>
    response: AppSettings
  }
}

export interface AppSettings {
  theme: 'light' | 'dark' | 'system'
  language: 'th' | 'en'
  autoUpdate: boolean
  notifications: boolean
  windowBounds: {
    x: number
    y: number
    width: number
    height: number
  }
  recentFiles: string[]
}
```

### Main Process IPC Handlers

```typescript
// src/main/ipc/index.ts
import { ipcMain, dialog, shell, app, nativeImage, Notification } from 'electron'
import { readFileSync, writeFileSync, existsSync, watchFile, unwatchFile } from 'fs'
import { join } from 'path'
import os from 'os'
import { IpcChannels, IpcInvokeMap } from '@/types/ipc'

// Type-safe ipcMain handler
function handleInvoke<C extends keyof IpcInvokeMap>(
  channel: C,
  handler: (
    event: Electron.IpcMainInvokeEvent,
    request: IpcInvokeMap[C]['request']
  ) => Promise<IpcInvokeMap[C]['response']>
): void {
  ipcMain.handle(channel, handler)
}

export function setupIpcHandlers(): void {
  // File: Open
  handleInvoke(IpcChannels.OPEN_FILE, async (_, options) => {
    const result = await dialog.showOpenDialog({
      title: options?.title,
      filters: options?.filters,
      properties: options?.multiple 
        ? ['openFile', 'multiSelections'] 
        : ['openFile']
    })
    
    return {
      filePaths: result.filePaths,
      canceled: result.canceled
    }
  })
  
  // File: Save
  handleInvoke(IpcChannels.SAVE_FILE, async (_, data) => {
    let filePath = data.filePath
    
    if (!filePath) {
      const result = await dialog.showSaveDialog({
        title: data.title,
        filters: data.filters
      })
      
      if (result.canceled || !result.filePath) {
        return { filePath: undefined, canceled: true, success: false }
      }
      
      filePath = result.filePath
    }
    
    try {
      writeFileSync(filePath, data.content)
      return { filePath, canceled: false, success: true }
    } catch (error) {
      return { filePath: undefined, canceled: false, success: false }
    }
  })
  
  // File: Read
  handleInvoke(IpcChannels.READ_FILE, async (_, { filePath }) => {
    if (!existsSync(filePath)) {
      throw new Error(`ไม่พบไฟล์: ${filePath}`)
    }
    
    const content = readFileSync(filePath, 'utf-8')
    const stats = require('fs').statSync(filePath)
    
    return {
      content,
      filePath,
      encoding: 'utf-8',
      size: stats.size
    }
  })
  
  // System: Info
  handleInvoke(IpcChannels.GET_SYSTEM_INFO, async () => {
    const cpus = os.cpus()
    const totalMemory = os.totalmem()
    const freeMemory = os.freemem()
    
    return {
      platform: process.platform,
      arch: process.arch,
      version: process.version,
      memory: {
        total: totalMemory,
        free: freeMemory,
        used: totalMemory - freeMemory
      },
      cpu: {
        model: cpus[0]?.model ?? 'Unknown',
        cores: cpus.length,
        speed: cpus[0]?.speed ?? 0
      }
    }
  })
  
  // App: Version
  ipcMain.handle(IpcChannels.GET_VERSION, () => app.getVersion())
  
  // Settings
  let settings = loadSettings()
  
  handleInvoke(IpcChannels.GET_SETTINGS, async () => settings)
  
  handleInvoke(IpcChannels.SET_SETTINGS, async (event, newSettings) => {
    settings = { ...settings, ...newSettings }
    saveSettings(settings)
    
    // Notify all windows
    const windows = require('electron').BrowserWindow.getAllWindows()
    windows.forEach(win => {
      win.webContents.send(IpcChannels.SETTINGS_CHANGED, settings)
    })
    
    return settings
  })
}

function loadSettings(): AppSettings {
  const settingsPath = join(app.getPath('userData'), 'settings.json')
  
  if (existsSync(settingsPath)) {
    try {
      return JSON.parse(readFileSync(settingsPath, 'utf-8'))
    } catch {}
  }
  
  return {
    theme: 'system',
    language: 'th',
    autoUpdate: true,
    notifications: true,
    windowBounds: { x: 0, y: 0, width: 1200, height: 800 },
    recentFiles: []
  }
}

function saveSettings(settings: AppSettings): void {
  const settingsPath = join(app.getPath('userData'), 'settings.json')
  writeFileSync(settingsPath, JSON.stringify(settings, null, 2))
}
```

## 70.5 Context Bridge

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'
import { IpcChannels } from '../types/ipc'

// Type-safe invoke wrapper
function invoke<C extends keyof IpcInvokeMap>(
  channel: C,
  request: IpcInvokeMap[C]['request']
): Promise<IpcInvokeMap[C]['response']> {
  return ipcRenderer.invoke(channel, request)
}

// Expose typed API to renderer
const electronAPI: ElectronAPI = {
  // File operations
  openFile: (options) => invoke(IpcChannels.OPEN_FILE, options ?? {}),
  saveFile: (data) => invoke(IpcChannels.SAVE_FILE, data),
  readFile: (filePath) => invoke(IpcChannels.READ_FILE, { filePath }),
  
  watchFile: (filePath, callback) => {
    const handler = (_: any, event: FileEvent) => callback(event)
    ipcRenderer.on(`${IpcChannels.WATCH_FILE}:${filePath}`, handler)
    ipcRenderer.invoke(IpcChannels.WATCH_FILE, { filePath })
    
    // Return unsubscribe function
    return () => {
      ipcRenderer.off(`${IpcChannels.WATCH_FILE}:${filePath}`, handler)
      ipcRenderer.invoke(IpcChannels.UNWATCH_FILE, { filePath })
    }
  },
  
  // System
  getSystemInfo: () => invoke(IpcChannels.GET_SYSTEM_INFO, undefined),
  openExternal: (url) => invoke(IpcChannels.OPEN_EXTERNAL, { url }) as any,
  showNotification: (notification) => {
    ipcRenderer.send(IpcChannels.SHOW_NOTIFICATION, notification)
  },
  
  // App
  getVersion: () => ipcRenderer.sendSync(IpcChannels.GET_VERSION),
  checkForUpdates: () => invoke(IpcChannels.CHECK_UPDATES, undefined),
  
  // Events
  on: (channel, listener) => {
    ipcRenderer.on(channel, (_, ...args) => listener(...args))
  },
  off: (channel, listener) => {
    ipcRenderer.off(channel, listener)
  }
}

// Expose in main world
contextBridge.exposeInMainWorld('electron', electronAPI)
```

## 70.6 File System Access

```typescript
// src/renderer/src/hooks/useFileSystem.ts
import { useState, useCallback } from 'react'

interface FileSystemHook {
  openFile: (options?: OpenFileOptions) => Promise<FileContent | null>
  saveFile: (content: string, filePath?: string) => Promise<string | null>
  readFile: (filePath: string) => Promise<FileContent | null>
  recentFiles: string[]
  isLoading: boolean
  error: string | null
}

export function useFileSystem(): FileSystemHook {
  const [recentFiles, setRecentFiles] = useState<string[]>([])
  const [isLoading, setIsLoading] = useState<boolean>(false)
  const [error, setError] = useState<string | null>(null)
  
  const openFile = useCallback(async (
    options?: OpenFileOptions
  ): Promise<FileContent | null> => {
    setIsLoading(true)
    setError(null)
    
    try {
      const result = await window.electron.openFile(options)
      if (!result || result.canceled || result.filePaths.length === 0) {
        return null
      }
      
      const fileContent = await window.electron.readFile(result.filePaths[0])
      
      // เพิ่มในรายการล่าสุด
      setRecentFiles(prev => {
        const newList = [result.filePaths[0], ...prev.filter(f => f !== result.filePaths[0])]
        return newList.slice(0, 10) // เก็บ 10 ไฟล์ล่าสุด
      })
      
      return fileContent
    } catch (err) {
      setError(err instanceof Error ? err.message : 'ไม่สามารถเปิดไฟล์ได้')
      return null
    } finally {
      setIsLoading(false)
    }
  }, [])
  
  const saveFile = useCallback(async (
    content: string,
    filePath?: string
  ): Promise<string | null> => {
    setIsLoading(true)
    setError(null)
    
    try {
      const result = await window.electron.saveFile({
        content,
        filePath,
        filters: [
          { name: 'Text Files', extensions: ['txt', 'md'] },
          { name: 'JSON Files', extensions: ['json'] },
          { name: 'All Files', extensions: ['*'] }
        ]
      })
      
      if (result.canceled || !result.filePath) return null
      return result.filePath
    } catch (err) {
      setError(err instanceof Error ? err.message : 'ไม่สามารถบันทึกไฟล์ได้')
      return null
    } finally {
      setIsLoading(false)
    }
  }, [])
  
  const readFile = useCallback(async (
    filePath: string
  ): Promise<FileContent | null> => {
    setIsLoading(true)
    setError(null)
    
    try {
      return await window.electron.readFile(filePath)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'ไม่สามารถอ่านไฟล์ได้')
      return null
    } finally {
      setIsLoading(false)
    }
  }, [])
  
  return {
    openFile,
    saveFile,
    readFile,
    recentFiles,
    isLoading,
    error
  }
}
```

## 70.7 System Dialogs

```typescript
// src/main/dialogs.ts
import { dialog, BrowserWindow, MessageBoxReturnValue } from 'electron'

interface DialogOptions {
  type?: 'none' | 'info' | 'error' | 'question' | 'warning'
  title?: string
  message: string
  detail?: string
  buttons?: string[]
  defaultId?: number
  cancelId?: number
}

interface DialogResult {
  response: number
  checkboxChecked: boolean
}

export async function showMessageDialog(
  window: BrowserWindow,
  options: DialogOptions
): Promise<DialogResult> {
  return dialog.showMessageBox(window, {
    type: options.type ?? 'info',
    title: options.title,
    message: options.message,
    detail: options.detail,
    buttons: options.buttons ?? ['ตกลง'],
    defaultId: options.defaultId ?? 0,
    cancelId: options.cancelId
  })
}

export async function showConfirmDialog(
  window: BrowserWindow,
  message: string,
  detail?: string
): Promise<boolean> {
  const result = await showMessageDialog(window, {
    type: 'question',
    message,
    detail,
    buttons: ['ใช่', 'ไม่'],
    defaultId: 0,
    cancelId: 1
  })
  
  return result.response === 0
}

export async function showErrorDialog(
  window: BrowserWindow,
  title: string,
  error: Error | string
): Promise<void> {
  const message = error instanceof Error ? error.message : error
  const detail = error instanceof Error ? error.stack : undefined
  
  await showMessageDialog(window, {
    type: 'error',
    title,
    message,
    detail
  })
}

// Open/Save dialogs
export async function showOpenFileDialog(
  window: BrowserWindow,
  options: OpenFileOptions = {}
): Promise<string[] | null> {
  const result = await dialog.showOpenDialog(window, {
    title: options.title ?? 'เปิดไฟล์',
    filters: options.filters,
    properties: options.multiple
      ? ['openFile', 'multiSelections']
      : ['openFile']
  })
  
  return result.canceled ? null : result.filePaths
}

export async function showSaveFileDialog(
  window: BrowserWindow,
  options: SaveDialogOptions = {}
): Promise<string | null> {
  const result = await dialog.showSaveDialog(window, {
    title: options.title ?? 'บันทึกไฟล์',
    defaultPath: options.defaultPath,
    filters: options.filters,
    buttonLabel: options.buttonLabel ?? 'บันทึก'
  })
  
  return result.canceled ? null : result.filePath
}
```

## 70.8 Menu และ Tray

```typescript
// src/main/menu.ts
import { Menu, MenuItem, MenuItemConstructorOptions, BrowserWindow, app, shell } from 'electron'

export function createMenu(mainWindow: BrowserWindow): void {
  const template: MenuItemConstructorOptions[] = [
    {
      label: 'ไฟล์',
      submenu: [
        {
          label: 'เปิดไฟล์ใหม่',
          accelerator: 'CmdOrCtrl+N',
          click: () => mainWindow.webContents.send('menu:newFile')
        },
        {
          label: 'เปิดไฟล์...',
          accelerator: 'CmdOrCtrl+O',
          click: () => mainWindow.webContents.send('menu:openFile')
        },
        {
          label: 'บันทึก',
          accelerator: 'CmdOrCtrl+S',
          click: () => mainWindow.webContents.send('menu:save')
        },
        { type: 'separator' },
        {
          label: 'ออก',
          accelerator: process.platform === 'darwin' ? 'Cmd+Q' : 'Alt+F4',
          click: () => app.quit()
        }
      ]
    },
    {
      label: 'แก้ไข',
      submenu: [
        { label: 'เลิกทำ', role: 'undo' },
        { label: 'ทำซ้ำ', role: 'redo' },
        { type: 'separator' },
        { label: 'ตัด', role: 'cut' },
        { label: 'คัดลอก', role: 'copy' },
        { label: 'วาง', role: 'paste' },
        { label: 'เลือกทั้งหมด', role: 'selectAll' }
      ]
    },
    {
      label: 'มุมมอง',
      submenu: [
        { label: 'โหลดซ้ำ', role: 'reload' },
        { type: 'separator' },
        { label: 'ขยาย', role: 'zoomIn' },
        { label: 'ย่อ', role: 'zoomOut' },
        { label: 'ขนาดจริง', role: 'resetZoom' },
        { type: 'separator' },
        { label: 'เต็มจอ', role: 'togglefullscreen' }
      ]
    },
    {
      label: 'ช่วยเหลือ',
      submenu: [
        {
          label: 'เอกสาร',
          click: () => shell.openExternal('https://docs.example.com')
        },
        {
          label: 'รายงานปัญหา',
          click: () => shell.openExternal('https://github.com/example/issues')
        },
        { type: 'separator' },
        {
          label: 'เกี่ยวกับ',
          click: () => mainWindow.webContents.send('menu:about')
        }
      ]
    }
  ]
  
  const menu = Menu.buildFromTemplate(template)
  Menu.setApplicationMenu(menu)
}

// src/main/tray.ts
import { Tray, Menu, BrowserWindow, nativeImage, app } from 'electron'
import { join } from 'path'

export function createTray(mainWindow: BrowserWindow): Tray {
  const icon = nativeImage.createFromPath(join(__dirname, '../../resources/tray-icon.png'))
  const tray = new Tray(icon.resize({ width: 16, height: 16 }))
  
  const contextMenu = Menu.buildFromTemplate([
    {
      label: 'เปิดแอป',
      click: () => {
        mainWindow.show()
        mainWindow.focus()
      }
    },
    {
      label: 'ตั้งค่า',
      click: () => mainWindow.webContents.send('menu:settings')
    },
    { type: 'separator' },
    {
      label: 'ออก',
      click: () => app.quit()
    }
  ])
  
  tray.setToolTip('My Electron App')
  tray.setContextMenu(contextMenu)
  
  tray.on('double-click', () => {
    if (mainWindow.isVisible()) {
      mainWindow.focus()
    } else {
      mainWindow.show()
    }
  })
  
  return tray
}
```

## 70.9 Auto-Updater

```typescript
// src/main/updater.ts
import { autoUpdater, UpdateInfo, ProgressInfo } from 'electron-updater'
import { BrowserWindow, ipcMain } from 'electron'
import log from 'electron-log'

interface UpdaterEvents {
  onCheckingForUpdate: () => void
  onUpdateAvailable: (info: UpdateInfo) => void
  onUpdateNotAvailable: (info: UpdateInfo) => void
  onDownloadProgress: (progress: ProgressInfo) => void
  onUpdateDownloaded: (info: UpdateInfo) => void
  onError: (error: Error) => void
}

export function setupAutoUpdater(mainWindow: BrowserWindow): void {
  // Config
  autoUpdater.autoDownload = false
  autoUpdater.logger = log
  
  const sendToRenderer = (channel: string, data?: any): void => {
    mainWindow.webContents.send(channel, data)
  }
  
  // Events
  autoUpdater.on('checking-for-update', () => {
    sendToRenderer('update:checking')
    log.info('กำลังตรวจสอบอัพเดต...')
  })
  
  autoUpdater.on('update-available', (info: UpdateInfo) => {
    sendToRenderer('update:available', {
      version: info.version,
      releaseDate: info.releaseDate,
      releaseNotes: info.releaseNotes
    })
    log.info(`พบอัพเดตใหม่: ${info.version}`)
  })
  
  autoUpdater.on('update-not-available', (info: UpdateInfo) => {
    sendToRenderer('update:notAvailable', { version: info.version })
    log.info('ใช้เวอร์ชันล่าสุดแล้ว')
  })
  
  autoUpdater.on('download-progress', (progress: ProgressInfo) => {
    sendToRenderer('update:downloadProgress', {
      percent: Math.round(progress.percent),
      bytesPerSecond: progress.bytesPerSecond,
      transferred: progress.transferred,
      total: progress.total
    })
  })
  
  autoUpdater.on('update-downloaded', (info: UpdateInfo) => {
    sendToRenderer('update:downloaded', {
      version: info.version,
      releaseNotes: info.releaseNotes
    })
    log.info(`ดาวน์โหลดอัพเดต ${info.version} เสร็จแล้ว`)
  })
  
  autoUpdater.on('error', (error: Error) => {
    sendToRenderer('update:error', { message: error.message })
    log.error('Auto-updater error:', error)
  })
  
  // IPC handlers
  ipcMain.handle('update:check', () => autoUpdater.checkForUpdates())
  ipcMain.handle('update:download', () => autoUpdater.downloadUpdate())
  ipcMain.handle('update:install', () => autoUpdater.quitAndInstall(false, true))
}
```

## 70.10 Security Best Practices

```typescript
// src/main/security.ts
import { session, BrowserWindow, app } from 'electron'

export function setupSecurityPolicies(): void {
  // Content Security Policy
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [
          "default-src 'self'; " +
          "script-src 'self'; " +
          "style-src 'self' 'unsafe-inline'; " +
          "img-src 'self' data: https:; " +
          "connect-src 'self' https://api.example.com"
        ]
      }
    })
  })
  
  // Prevent navigation to unauthorized URLs
  app.on('web-contents-created', (_, contents) => {
    contents.on('will-navigate', (event, url) => {
      const { origin } = new URL(url)
      const allowedOrigins = ['https://example.com', 'https://api.example.com']
      
      if (!allowedOrigins.includes(origin) && !url.startsWith('file://')) {
        event.preventDefault()
      }
    })
    
    // Block new windows
    contents.setWindowOpenHandler(({ url }) => {
      // Allow opening in default browser
      require('electron').shell.openExternal(url)
      return { action: 'deny' }
    })
  })
}

// Secure window creation
export function createSecureWindow(): BrowserWindow {
  return new BrowserWindow({
    webPreferences: {
      nodeIntegration: false,        // ไม่เปิด Node.js ใน renderer
      contextIsolation: true,        // แยก context
      sandbox: true,                 // เปิด sandbox
      webSecurity: true,             // เปิด web security
      allowRunningInsecureContent: false
    }
  })
}
```

## 70.11 Packaging และ Distribution

```typescript
// electron-builder.config.ts
import { Configuration } from 'electron-builder'

const config: Configuration = {
  appId: 'com.example.myapp',
  productName: 'My App',
  copyright: `Copyright © ${new Date().getFullYear()} Example Inc.`,
  
  // ไฟล์ที่จะรวม
  files: [
    'dist/**/*',
    'resources/**/*',
    '!node_modules/**/*'
  ],
  
  // macOS
  mac: {
    category: 'public.app-category.productivity',
    icon: 'resources/icon.icns',
    hardenedRuntime: true,
    entitlements: 'resources/entitlements.mac.plist',
    entitlementsInherit: 'resources/entitlements.mac.plist',
    gatekeeperAssess: false,
    target: [
      { target: 'dmg', arch: ['x64', 'arm64'] },
      { target: 'zip', arch: ['x64', 'arm64'] }
    ]
  },
  
  // Windows
  win: {
    icon: 'resources/icon.ico',
    target: [
      { target: 'nsis', arch: ['x64'] },
      { target: 'portable', arch: ['x64'] }
    ],
    signingHashAlgorithms: ['sha256']
  },
  
  // Linux
  linux: {
    icon: 'resources/icon.png',
    category: 'Utility',
    target: [
      { target: 'AppImage', arch: ['x64'] },
      { target: 'deb', arch: ['x64'] },
      { target: 'rpm', arch: ['x64'] }
    ]
  },
  
  // Auto-update
  publish: {
    provider: 'github',
    owner: 'example',
    repo: 'my-app',
    private: false
  },
  
  // NSIS installer config
  nsis: {
    oneClick: false,
    allowElevation: true,
    allowToChangeInstallationDirectory: true,
    installerIcon: 'resources/installer.ico',
    installerHeaderIcon: 'resources/installer-header.ico',
    createDesktopShortcut: true,
    createStartMenuShortcut: true,
    shortcutName: 'My App'
  }
}

export default config
```

## 70.12 Complete App Example

```typescript
// src/renderer/src/App.tsx
import React, { useEffect, useState } from 'react'
import { useFileSystem } from './hooks/useFileSystem'
import { useSettings } from './hooks/useSettings'
import { useUpdater } from './hooks/useUpdater'

interface AppState {
  currentFile: string | null
  content: string
  isDirty: boolean
  isFullScreen: boolean
}

function App(): React.JSX.Element {
  const [state, setState] = useState<AppState>({
    currentFile: null,
    content: '',
    isDirty: false,
    isFullScreen: false
  })
  
  const { openFile, saveFile, isLoading } = useFileSystem()
  const { settings, updateSettings } = useSettings()
  const { updateAvailable, downloadUpdate } = useUpdater()
  
  // Listen to menu events
  useEffect(() => {
    const handleNewFile = () => {
      if (state.isDirty && !confirm('ยกเลิกการเปลี่ยนแปลงหรือไม่?')) return
      setState(prev => ({ ...prev, currentFile: null, content: '', isDirty: false }))
    }
    
    const handleOpenFile = async () => {
      if (state.isDirty && !confirm('ยกเลิกการเปลี่ยนแปลงหรือไม่?')) return
      const file = await openFile({
        filters: [
          { name: 'Text Files', extensions: ['txt', 'md', 'json'] },
          { name: 'All Files', extensions: ['*'] }
        ]
      })
      
      if (file) {
        setState(prev => ({
          ...prev,
          currentFile: file.filePath,
          content: file.content,
          isDirty: false
        }))
      }
    }
    
    const handleSave = async () => {
      const savedPath = await saveFile(state.content, state.currentFile ?? undefined)
      if (savedPath) {
        setState(prev => ({ ...prev, currentFile: savedPath, isDirty: false }))
      }
    }
    
    window.electron.on('menu:newFile', handleNewFile)
    window.electron.on('menu:openFile', handleOpenFile)
    window.electron.on('menu:save', handleSave)
    
    return () => {
      window.electron.off('menu:newFile', handleNewFile)
      window.electron.off('menu:openFile', handleOpenFile)
      window.electron.off('menu:save', handleSave)
    }
  }, [state.isDirty, state.content, state.currentFile, openFile, saveFile])
  
  const handleContentChange = (newContent: string): void => {
    setState(prev => ({ ...prev, content: newContent, isDirty: true }))
  }
  
  return (
    <div className={`app theme-${settings.theme}`}>
      <header className="titlebar">
        <span className="title">
          {state.currentFile ? 
            `${state.currentFile}${state.isDirty ? ' *' : ''}` : 
            'ไม่มีชื่อไฟล์'
          }
        </span>
      </header>
      
      {updateAvailable && (
        <div className="update-banner">
          <span>มีอัพเดตใหม่!</span>
          <button onClick={downloadUpdate}>ดาวน์โหลด</button>
        </div>
      )}
      
      <main className="editor">
        {isLoading ? (
          <div className="loading">กำลังโหลด...</div>
        ) : (
          <textarea
            className="text-editor"
            value={state.content}
            onChange={e => handleContentChange(e.target.value)}
            placeholder="เริ่มพิมพ์หรือเปิดไฟล์..."
          />
        )}
      </main>
      
      <footer className="statusbar">
        <span>ตัวอักษร: {state.content.length}</span>
        <span>บรรทัด: {state.content.split('\n').length}</span>
        {state.isDirty && <span className="unsaved">ยังไม่บันทึก</span>}
      </footer>
    </div>
  )
}

export default App
```

## สรุป

Electron กับ TypeScript ให้ประโยชน์มากมาย:

1. **Main/Renderer separation** พร้อม typed IPC channels
2. **Context Bridge** ที่ปลอดภัยและ type-safe
3. **File system access** พร้อม typed APIs
4. **System dialogs** ที่ใช้งานง่าย
5. **Auto-updater** พร้อม progress tracking
6. **Security practices** ที่สำคัญสำหรับ production
7. **Packaging** สำหรับทุก platforms

TypeScript ช่วยให้การพัฒนา desktop apps ด้วย Electron เป็นเรื่องที่มีโครงสร้างชัดเจน ลด bugs และบำรุงรักษาง่ายในระยะยาว
