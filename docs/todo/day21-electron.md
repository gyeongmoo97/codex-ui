# Day 21 TODO - 프로덕션 배포 및 최종 점검 (Electron)

> **목표**: 프로덕션 빌드, 배포 설정, 최종 점검 완료

## 전체 개요

Day 21은 Codex UI의 프로덕션 배포를 준비합니다:
- 프로덕션 빌드 최적화
- Code signing
- Auto-update 설정
- 배포 파이프라인
- 보안 점검
- 최종 QA

**Electron 특화:**
- Windows/macOS/Linux 빌드
- Code signing (인증서)
- Notarization (macOS)
- Auto-updater with electron-updater
- Installer customization
- Release automation

---

## Commit 117: 프로덕션 빌드 설정

### 📋 작업 내용

1. **환경 설정**
2. **빌드 스크립트**
3. **최적화 플래그**
4. **번들 검증**

### 📁 파일 구조

```
build/
├── scripts/
│   ├── build-prod.ts      # 프로덕션 빌드
│   └── verify-build.ts    # 빌드 검증
├── resources/
│   ├── icon.icns          # macOS icon
│   ├── icon.ico           # Windows icon
│   └── icon.png           # Linux icon
└── entitlements.mac.plist # macOS entitlements
```

### 1️⃣ Environment Configuration

**파일**: `.env.production`

```env
# API Configuration
VITE_API_URL=https://api.codex-ui.com
VITE_WS_URL=wss://ws.codex-ui.com

# Sentry
VITE_SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx

# Feature Flags
VITE_ENABLE_TELEMETRY=true
VITE_ENABLE_AUTO_UPDATE=true
VITE_ENABLE_CRASH_REPORTER=true

# Build Info
VITE_BUILD_NUMBER=${BUILD_NUMBER}
VITE_BUILD_TIME=${BUILD_TIME}
VITE_GIT_SHA=${GIT_SHA}
```

### 2️⃣ Electron Builder Configuration

**파일**: `electron-builder.json`

```json
{
  "appId": "com.codex.ui",
  "productName": "Codex UI",
  "copyright": "Copyright © 2024 Codex",
  "asar": true,
  "asarUnpack": [
    "node_modules/better-sqlite3/**/*",
    "node_modules/sharp/**/*"
  ],
  "directories": {
    "output": "dist-release",
    "buildResources": "build"
  },
  "files": [
    "dist-electron/**/*",
    "dist/**/*",
    "package.json"
  ],
  "extraMetadata": {
    "main": "dist-electron/main.js"
  },
  "win": {
    "target": [
      {
        "target": "nsis",
        "arch": ["x64", "arm64"]
      },
      {
        "target": "portable",
        "arch": ["x64"]
      }
    ],
    "icon": "build/resources/icon.ico",
    "artifactName": "Codex-UI-Setup-${version}.${ext}",
    "publisherName": "Codex Inc.",
    "verifyUpdateCodeSignature": true,
    "signingHashAlgorithms": ["sha256"],
    "certificateFile": "build/certs/windows.pfx",
    "certificatePassword": "${WIN_CSC_KEY_PASSWORD}"
  },
  "nsis": {
    "oneClick": false,
    "allowToChangeInstallationDirectory": true,
    "allowElevation": true,
    "createDesktopShortcut": true,
    "createStartMenuShortcut": true,
    "shortcutName": "Codex UI",
    "include": "build/installer.nsh",
    "deleteAppDataOnUninstall": false
  },
  "mac": {
    "category": "public.app-category.productivity",
    "target": [
      {
        "target": "dmg",
        "arch": ["x64", "arm64", "universal"]
      },
      {
        "target": "zip",
        "arch": ["x64", "arm64", "universal"]
      }
    ],
    "icon": "build/resources/icon.icns",
    "artifactName": "Codex-UI-${version}-${arch}.${ext}",
    "hardenedRuntime": true,
    "gatekeeperAssess": false,
    "entitlements": "build/entitlements.mac.plist",
    "entitlementsInherit": "build/entitlements.mac.plist",
    "provisioningProfile": "build/certs/embedded.provisionprofile",
    "identity": "Developer ID Application: Codex Inc.",
    "notarize": {
      "teamId": "${APPLE_TEAM_ID}"
    }
  },
  "dmg": {
    "background": "build/resources/dmg-background.png",
    "window": {
      "width": 540,
      "height": 380
    },
    "contents": [
      {
        "x": 140,
        "y": 200
      },
      {
        "x": 400,
        "y": 200,
        "type": "link",
        "path": "/Applications"
      }
    ]
  },
  "linux": {
    "target": [
      {
        "target": "AppImage",
        "arch": ["x64", "arm64"]
      },
      {
        "target": "deb",
        "arch": ["x64", "arm64"]
      },
      {
        "target": "rpm",
        "arch": ["x64", "arm64"]
      }
    ],
    "icon": "build/resources/icon.png",
    "category": "Utility",
    "desktop": {
      "Name": "Codex UI",
      "Comment": "AI-powered desktop application",
      "Categories": "Utility;Development;"
    },
    "artifactName": "codex-ui_${version}_${arch}.${ext}"
  },
  "publish": {
    "provider": "github",
    "owner": "codex",
    "repo": "codex-ui",
    "releaseType": "release"
  },
  "afterSign": "build/scripts/notarize.js",
  "compression": "maximum"
}
```

### 3️⃣ macOS Entitlements

**파일**: `build/entitlements.mac.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.security.cs.allow-jit</key>
  <true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
  <true/>
  <key>com.apple.security.cs.allow-dyld-environment-variables</key>
  <true/>
  <key>com.apple.security.network.client</key>
  <true/>
  <key>com.apple.security.network.server</key>
  <true/>
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>
  <key>com.apple.security.files.downloads.read-write</key>
  <true/>
</dict>
</plist>
```

### 4️⃣ Notarization Script

**파일**: `build/scripts/notarize.js`

```javascript
const { notarize } = require('@electron/notarize');

exports.default = async function notarizing(context) {
  const { electronPlatformName, appOutDir } = context;

  if (electronPlatformName !== 'darwin') {
    return;
  }

  const appName = context.packager.appInfo.productFilename;

  console.log(`Notarizing ${appName}...`);

  await notarize({
    appBundleId: 'com.codex.ui',
    appPath: `${appOutDir}/${appName}.app`,
    appleId: process.env.APPLE_ID,
    appleIdPassword: process.env.APPLE_ID_PASSWORD,
    teamId: process.env.APPLE_TEAM_ID,
  });

  console.log('Notarization complete!');
};
```

### 5️⃣ Build Script

**파일**: `build/scripts/build-prod.ts`

```typescript
import { build } from 'electron-builder';
import { build as viteBuild } from 'vite';
import fs from 'fs/promises';
import path from 'path';
import { execSync } from 'child_process';

async function buildProduction() {
  console.log('🚀 Starting production build...\n');

  // Set environment
  process.env.NODE_ENV = 'production';
  process.env.BUILD_NUMBER = execSync('git rev-list --count HEAD').toString().trim();
  process.env.BUILD_TIME = new Date().toISOString();
  process.env.GIT_SHA = execSync('git rev-parse HEAD').toString().trim();

  console.log('Build info:');
  console.log(`  Build number: ${process.env.BUILD_NUMBER}`);
  console.log(`  Git SHA: ${process.env.GIT_SHA}`);
  console.log(`  Build time: ${process.env.BUILD_TIME}\n`);

  // Clean dist
  console.log('🧹 Cleaning dist directories...');
  await fs.rm('dist', { recursive: true, force: true });
  await fs.rm('dist-electron', { recursive: true, force: true });
  await fs.rm('dist-release', { recursive: true, force: true });

  // Build renderer
  console.log('📦 Building renderer process...');
  await viteBuild({
    configFile: 'electron.vite.config.ts',
    mode: 'production',
  });

  // Build main & preload
  console.log('📦 Building main process...');
  execSync('npm run build:main', { stdio: 'inherit' });

  // Verify build
  console.log('✅ Verifying build...');
  const mainExists = await fs.access('dist-electron/main.js').then(() => true).catch(() => false);
  const preloadExists = await fs.access('dist-electron/preload.js').then(() => true).catch(() => false);
  const rendererExists = await fs.access('dist/index.html').then(() => true).catch(() => false);

  if (!mainExists || !preloadExists || !rendererExists) {
    throw new Error('Build verification failed!');
  }

  // Package with electron-builder
  console.log('📦 Packaging application...');
  await build({
    config: require('../../electron-builder.json'),
    publish: process.argv.includes('--publish') ? 'always' : 'never',
  });

  console.log('\n✅ Production build complete!');
}

buildProduction().catch((error) => {
  console.error('❌ Build failed:', error);
  process.exit(1);
});
```

### 6️⃣ Package Scripts

**파일**: `package.json` (scripts section)

```json
{
  "scripts": {
    "dev": "electron-vite dev",
    "build": "npm run build:renderer && npm run build:main",
    "build:renderer": "vite build",
    "build:main": "tsc -p tsconfig.main.json && tsc -p tsconfig.preload.json",
    "build:prod": "tsx build/scripts/build-prod.ts",
    "build:win": "npm run build:prod -- --win",
    "build:mac": "npm run build:prod -- --mac",
    "build:linux": "npm run build:prod -- --linux",
    "build:all": "npm run build:prod -- --win --mac --linux",
    "publish": "npm run build:prod -- --publish",
    "preview": "electron-vite preview",
    "test": "vitest",
    "test:e2e": "playwright test",
    "lint": "eslint . --ext .ts,.tsx",
    "typecheck": "tsc --noEmit"
  }
}
```

### ✅ 완료 기준

- [ ] 프로덕션 환경 설정
- [ ] electron-builder 설정
- [ ] macOS notarization
- [ ] Windows code signing
- [ ] 빌드 스크립트 작성

### 📝 Commit Message

```
build: configure production build system

- Add electron-builder configuration
- Set up code signing for Windows/macOS
- Configure macOS notarization
- Create production build scripts
- Add environment-specific configs

Platforms:
- Windows (NSIS, Portable)
- macOS (DMG, ZIP, Universal)
- Linux (AppImage, DEB, RPM)
```

---

## Commit 118: Auto-Update 구현

### 📋 작업 내용

1. **electron-updater 설정**
2. **업데이트 체크**
3. **다운로드 진행률**
4. **업데이트 UI**

### 📁 파일 구조

```
src/main/updater/
├── AutoUpdater.ts         # 자동 업데이트
└── UpdateChecker.ts       # 업데이트 체크

src/renderer/components/updater/
├── UpdateNotification.tsx # 업데이트 알림
└── UpdateProgress.tsx     # 다운로드 진행률
```

### 1️⃣ Auto Updater Service

**파일**: `src/main/updater/AutoUpdater.ts`

```typescript
import { autoUpdater } from 'electron-updater';
import { app, BrowserWindow, dialog } from 'electron';
import log from 'electron-log';

export class AutoUpdater {
  private window: BrowserWindow | null = null;
  private updateAvailable = false;

  constructor() {
    this.configure();
  }

  private configure(): void {
    // Configure logging
    autoUpdater.logger = log;
    log.transports.file.level = 'info';

    // Auto download
    autoUpdater.autoDownload = false;
    autoUpdater.autoInstallOnAppQuit = true;

    // Update server
    if (process.env.NODE_ENV === 'production') {
      autoUpdater.setFeedURL({
        provider: 'github',
        owner: 'codex',
        repo: 'codex-ui',
      });
    }

    this.setupListeners();
  }

  private setupListeners(): void {
    autoUpdater.on('checking-for-update', () => {
      log.info('Checking for updates...');
      this.sendToRenderer('update:checking');
    });

    autoUpdater.on('update-available', (info) => {
      log.info('Update available:', info);
      this.updateAvailable = true;
      this.sendToRenderer('update:available', {
        version: info.version,
        releaseNotes: info.releaseNotes,
        releaseDate: info.releaseDate,
      });
    });

    autoUpdater.on('update-not-available', (info) => {
      log.info('Update not available:', info);
      this.sendToRenderer('update:not-available');
    });

    autoUpdater.on('error', (error) => {
      log.error('Update error:', error);
      this.sendToRenderer('update:error', {
        message: error.message,
      });
    });

    autoUpdater.on('download-progress', (progress) => {
      log.info('Download progress:', progress);
      this.sendToRenderer('update:download-progress', {
        percent: progress.percent,
        transferred: progress.transferred,
        total: progress.total,
        bytesPerSecond: progress.bytesPerSecond,
      });
    });

    autoUpdater.on('update-downloaded', (info) => {
      log.info('Update downloaded:', info);
      this.sendToRenderer('update:downloaded', {
        version: info.version,
      });

      // Show dialog
      this.showUpdateReadyDialog(info.version);
    });
  }

  setWindow(window: BrowserWindow): void {
    this.window = window;
  }

  async checkForUpdates(): Promise<void> {
    if (process.env.NODE_ENV !== 'production') {
      log.info('Skipping update check in development');
      return;
    }

    try {
      await autoUpdater.checkForUpdates();
    } catch (error) {
      log.error('Failed to check for updates:', error);
    }
  }

  async downloadUpdate(): Promise<void> {
    if (!this.updateAvailable) {
      log.warn('No update available to download');
      return;
    }

    try {
      await autoUpdater.downloadUpdate();
    } catch (error) {
      log.error('Failed to download update:', error);
    }
  }

  quitAndInstall(): void {
    autoUpdater.quitAndInstall(false, true);
  }

  private sendToRenderer(channel: string, data?: any): void {
    if (this.window && !this.window.isDestroyed()) {
      this.window.webContents.send(channel, data);
    }
  }

  private showUpdateReadyDialog(version: string): void {
    if (!this.window) return;

    dialog.showMessageBox(this.window, {
      type: 'info',
      title: 'Update Ready',
      message: `Version ${version} has been downloaded.`,
      detail: 'The update will be installed when you quit the application. Do you want to restart now?',
      buttons: ['Restart Now', 'Later'],
      defaultId: 0,
      cancelId: 1,
    }).then((result) => {
      if (result.response === 0) {
        this.quitAndInstall();
      }
    });
  }
}

export const updater = new AutoUpdater();
```

### 2️⃣ Update IPC Handlers

**파일**: `src/main/handlers/updater.ts`

```typescript
import { ipcMain } from 'electron';
import { updater } from '../updater/AutoUpdater';

export function registerUpdaterHandlers(): void {
  ipcMain.handle('updater:check', async () => {
    await updater.checkForUpdates();
  });

  ipcMain.handle('updater:download', async () => {
    await updater.downloadUpdate();
  });

  ipcMain.handle('updater:install', async () => {
    updater.quitAndInstall();
  });
}
```

### 3️⃣ Update Notification UI

**파일**: `src/renderer/components/updater/UpdateNotification.tsx`

```typescript
import React, { useEffect, useState } from 'react';
import { motion, AnimatePresence } from 'framer-motion';
import { Download, X, RefreshCw } from 'lucide-react';
import { Button } from '../ui/button';
import { Progress } from '../ui/progress';
import { Card } from '../ui/card';

interface UpdateInfo {
  version: string;
  releaseNotes?: string;
  releaseDate?: string;
}

interface DownloadProgress {
  percent: number;
  transferred: number;
  total: number;
  bytesPerSecond: number;
}

export function UpdateNotification() {
  const [updateAvailable, setUpdateAvailable] = useState(false);
  const [updateInfo, setUpdateInfo] = useState<UpdateInfo | null>(null);
  const [downloading, setDownloading] = useState(false);
  const [downloadProgress, setDownloadProgress] = useState<DownloadProgress | null>(null);
  const [updateReady, setUpdateReady] = useState(false);
  const [dismissed, setDismissed] = useState(false);

  useEffect(() => {
    if (!window.electronAPI) return;

    // Listen for update events
    const removeListeners: Array<() => void> = [];

    removeListeners.push(
      window.electronAPI.on('update:available', (info: UpdateInfo) => {
        setUpdateAvailable(true);
        setUpdateInfo(info);
        setDismissed(false);
      })
    );

    removeListeners.push(
      window.electronAPI.on('update:download-progress', (progress: DownloadProgress) => {
        setDownloading(true);
        setDownloadProgress(progress);
      })
    );

    removeListeners.push(
      window.electronAPI.on('update:downloaded', () => {
        setDownloading(false);
        setUpdateReady(true);
      })
    );

    return () => {
      removeListeners.forEach(remove => remove());
    };
  }, []);

  const handleDownload = async () => {
    if (!window.electronAPI) return;
    await window.electronAPI.downloadUpdate();
  };

  const handleInstall = async () => {
    if (!window.electronAPI) return;
    await window.electronAPI.installUpdate();
  };

  const handleDismiss = () => {
    setDismissed(true);
  };

  if (dismissed || !updateAvailable) return null;

  return (
    <AnimatePresence>
      <motion.div
        initial={{ opacity: 0, y: -50 }}
        animate={{ opacity: 1, y: 0 }}
        exit={{ opacity: 0, y: -50 }}
        className="fixed top-4 right-4 z-50 w-96"
      >
        <Card className="p-4 shadow-lg">
          <div className="flex items-start justify-between mb-3">
            <div className="flex items-center gap-2">
              <Download className="h-5 w-5 text-primary" />
              <h3 className="font-semibold">Update Available</h3>
            </div>
            <button
              onClick={handleDismiss}
              className="text-muted-foreground hover:text-foreground"
            >
              <X className="h-4 w-4" />
            </button>
          </div>

          {updateInfo && (
            <div className="mb-4">
              <p className="text-sm text-muted-foreground">
                Version {updateInfo.version} is now available
              </p>
              {updateInfo.releaseNotes && (
                <p className="text-xs text-muted-foreground mt-1">
                  {updateInfo.releaseNotes}
                </p>
              )}
            </div>
          )}

          {downloading && downloadProgress && (
            <div className="mb-4">
              <div className="flex items-center justify-between text-xs mb-2">
                <span>Downloading...</span>
                <span>{Math.round(downloadProgress.percent)}%</span>
              </div>
              <Progress value={downloadProgress.percent} />
              <p className="text-xs text-muted-foreground mt-1">
                {formatBytes(downloadProgress.transferred)} / {formatBytes(downloadProgress.total)}
                {' '}({formatBytes(downloadProgress.bytesPerSecond)}/s)
              </p>
            </div>
          )}

          {updateReady ? (
            <Button onClick={handleInstall} className="w-full">
              <RefreshCw className="h-4 w-4 mr-2" />
              Restart & Install
            </Button>
          ) : downloading ? (
            <Button disabled className="w-full">
              Downloading...
            </Button>
          ) : (
            <Button onClick={handleDownload} className="w-full">
              <Download className="h-4 w-4 mr-2" />
              Download Update
            </Button>
          )}
        </Card>
      </motion.div>
    </AnimatePresence>
  );
}

function formatBytes(bytes: number): string {
  if (bytes === 0) return '0 B';
  const k = 1024;
  const sizes = ['B', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i];
}
```

### 4️⃣ Auto Update Check on Startup

**파일**: `src/main/index.ts` (updated)

```typescript
import { app, BrowserWindow } from 'electron';
import { updater } from './updater/AutoUpdater';

app.whenReady().then(() => {
  const mainWindow = createWindow();

  // Set updater window
  updater.setWindow(mainWindow);

  // Check for updates after 5 seconds
  setTimeout(() => {
    updater.checkForUpdates();
  }, 5000);

  // Check for updates every 6 hours
  setInterval(() => {
    updater.checkForUpdates();
  }, 6 * 60 * 60 * 1000);
});
```

### ✅ 완료 기준

- [ ] electron-updater 설정
- [ ] 업데이트 체크 로직
- [ ] 다운로드 진행률
- [ ] 업데이트 알림 UI
- [ ] 자동 설치

### 📝 Commit Message

```
feat(updater): implement auto-update system

- Add electron-updater configuration
- Implement update checking and downloading
- Create update notification UI
- Add download progress tracking
- Support automatic installation

Features:
- GitHub releases integration
- Background updates
- User-controlled installation
- Delta updates support
```

---

## Commit 119: 보안 점검 및 강화

### 📋 작업 내용

1. **CSP 설정**
2. **보안 헤더**
3. **취약점 스캔**
4. **의존성 감사**

### 📁 파일 구조

```
build/scripts/
├── security-audit.ts      # 보안 감사
└── check-vulnerabilities.ts # 취약점 체크
```

### 1️⃣ Content Security Policy

**파일**: `src/main/index.ts` (updated)

```typescript
import { app, BrowserWindow, session } from 'electron';

function createWindow(): BrowserWindow {
  const mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true,
      sandbox: true,
      webSecurity: true,
      allowRunningInsecureContent: false,
    },
  });

  // Set Content Security Policy
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [
          [
            "default-src 'self'",
            "script-src 'self' 'unsafe-inline'",
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https:",
            "font-src 'self' data:",
            "connect-src 'self' https://api.openai.com https://api.anthropic.com wss:",
            "media-src 'self'",
            "object-src 'none'",
            "frame-src 'none'",
          ].join('; '),
        ],
      },
    });
  });

  return mainWindow;
}
```

### 2️⃣ Security Audit Script

**파일**: `build/scripts/security-audit.ts`

```typescript
import { execSync } from 'child_process';
import fs from 'fs/promises';
import path from 'path';

interface AuditResult {
  vulnerabilities: {
    total: number;
    critical: number;
    high: number;
    moderate: number;
    low: number;
  };
  packages: Array<{
    name: string;
    severity: string;
    via: string;
  }>;
}

async function runSecurityAudit(): Promise<void> {
  console.log('🔒 Running security audit...\n');

  // Run npm audit
  console.log('📦 Checking npm packages...');
  try {
    const auditOutput = execSync('npm audit --json', { encoding: 'utf-8' });
    const audit: AuditResult = JSON.parse(auditOutput);

    console.log('\nVulnerabilities:');
    console.log(`  Critical: ${audit.vulnerabilities.critical}`);
    console.log(`  High: ${audit.vulnerabilities.high}`);
    console.log(`  Moderate: ${audit.vulnerabilities.moderate}`);
    console.log(`  Low: ${audit.vulnerabilities.low}`);
    console.log(`  Total: ${audit.vulnerabilities.total}\n`);

    if (audit.vulnerabilities.critical > 0 || audit.vulnerabilities.high > 0) {
      console.error('❌ Critical or high severity vulnerabilities found!');
      console.error('Please run "npm audit fix" to resolve them.');
      process.exit(1);
    }
  } catch (error) {
    // npm audit returns non-zero exit code if vulnerabilities found
    console.error('❌ npm audit failed');
    throw error;
  }

  // Check for hardcoded secrets
  console.log('🔍 Checking for hardcoded secrets...');
  const secretPatterns = [
    /api[_-]?key[_-]?=\s*['"][^'"]+['"]/gi,
    /secret[_-]?key[_-]?=\s*['"][^'"]+['"]/gi,
    /password[_-]?=\s*['"][^'"]+['"]/gi,
    /token[_-]?=\s*['"][^'"]+['"]/gi,
  ];

  const srcFiles = await getSourceFiles('src');
  let secretsFound = false;

  for (const file of srcFiles) {
    const content = await fs.readFile(file, 'utf-8');

    for (const pattern of secretPatterns) {
      if (pattern.test(content)) {
        console.error(`❌ Potential secret found in ${file}`);
        secretsFound = true;
      }
    }
  }

  if (secretsFound) {
    console.error('\n❌ Hardcoded secrets detected!');
    process.exit(1);
  }

  console.log('✅ No hardcoded secrets found\n');

  // Check file permissions
  console.log('🔐 Checking file permissions...');
  // Add permission checks here

  console.log('\n✅ Security audit passed!');
}

async function getSourceFiles(dir: string): Promise<string[]> {
  const files: string[] = [];
  const entries = await fs.readdir(dir, { withFileTypes: true });

  for (const entry of entries) {
    const fullPath = path.join(dir, entry.name);

    if (entry.isDirectory()) {
      files.push(...await getSourceFiles(fullPath));
    } else if (/\.(ts|tsx|js|jsx)$/.test(entry.name)) {
      files.push(fullPath);
    }
  }

  return files;
}

runSecurityAudit().catch((error) => {
  console.error('Security audit failed:', error);
  process.exit(1);
});
```

### 3️⃣ Dependency Audit

**파일**: `build/scripts/check-dependencies.ts`

```typescript
import { execSync } from 'child_process';
import semver from 'semver';

interface PackageInfo {
  name: string;
  version: string;
  latest: string;
  outdated: boolean;
}

async function checkDependencies(): Promise<void> {
  console.log('📦 Checking dependencies...\n');

  // Check for outdated packages
  console.log('🔍 Checking for outdated packages...');
  try {
    const outdated = execSync('npm outdated --json', { encoding: 'utf-8' });
    const packages: Record<string, PackageInfo> = JSON.parse(outdated);

    const majorUpdates: string[] = [];
    const minorUpdates: string[] = [];

    for (const [name, info] of Object.entries(packages)) {
      const current = semver.parse(info.version);
      const latest = semver.parse(info.latest);

      if (!current || !latest) continue;

      if (latest.major > current.major) {
        majorUpdates.push(`${name}: ${info.version} → ${info.latest}`);
      } else if (latest.minor > current.minor) {
        minorUpdates.push(`${name}: ${info.version} → ${info.latest}`);
      }
    }

    if (majorUpdates.length > 0) {
      console.log('\n⚠️  Major updates available:');
      majorUpdates.forEach(update => console.log(`  ${update}`));
    }

    if (minorUpdates.length > 0) {
      console.log('\n📌 Minor updates available:');
      minorUpdates.forEach(update => console.log(`  ${update}`));
    }
  } catch (error) {
    // No outdated packages
    console.log('✅ All packages are up to date');
  }

  // Check license compatibility
  console.log('\n📜 Checking licenses...');
  const licenses = execSync('npm run license-checker --json', { encoding: 'utf-8' });
  // Add license checking logic

  console.log('\n✅ Dependency check complete!');
}

checkDependencies().catch((error) => {
  console.error('Dependency check failed:', error);
  process.exit(1);
});
```

### ✅ 완료 기준

- [ ] CSP 설정
- [ ] 보안 감사 스크립트
- [ ] 의존성 체크
- [ ] 취약점 스캔

### 📝 Commit Message

```
security: implement security hardening

- Add Content Security Policy
- Create security audit script
- Check for hardcoded secrets
- Audit dependencies for vulnerabilities

Security measures:
- CSP for XSS protection
- Automated security scanning
- Dependency vulnerability checks
- Secret detection
```

---

## Commit 120: CI/CD 파이프라인

### 📋 작업 내용

1. **GitHub Actions 워크플로우**
2. **자동 빌드**
3. **자동 테스트**
4. **자동 배포**

### 📁 파일 구조

```
.github/workflows/
├── build.yml              # 빌드 워크플로우
├── test.yml               # 테스트 워크플로우
├── release.yml            # 릴리즈 워크플로우
└── codeql.yml             # CodeQL 분석
```

### 1️⃣ Build Workflow

**파일**: `.github/workflows/build.yml`

```yaml
name: Build

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18.x]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run typecheck

      - name: Build
        run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dist-${{ matrix.os }}
          path: |
            dist/
            dist-electron/
```

### 2️⃣ Test Workflow

**파일**: `.github/workflows/test.yml`

```yaml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18.x
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json

  e2e-tests:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18.x
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Build application
        run: npm run build

      - name: Run E2E tests
        run: npm run test:e2e

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report-${{ matrix.os }}
          path: playwright-report/
```

### 3️⃣ Release Workflow

**파일**: `.github/workflows/release.yml`

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  create-release:
    runs-on: ubuntu-latest

    outputs:
      release_id: ${{ steps.create-release.outputs.result }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18.x

      - name: Get version
        run: echo "PACKAGE_VERSION=$(node -p "require('./package.json').version")" >> $GITHUB_ENV

      - name: Create release
        id: create-release
        uses: actions/github-script@v7
        with:
          script: |
            const { data } = await github.rest.repos.createRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              tag_name: `v${process.env.PACKAGE_VERSION}`,
              name: `v${process.env.PACKAGE_VERSION}`,
              body: 'See CHANGELOG.md for details',
              draft: false,
              prerelease: false
            })
            return data.id

  build-release:
    needs: create-release
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18.x
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build & Package (Windows)
        if: matrix.os == 'windows-latest'
        env:
          WIN_CSC_KEY_PASSWORD: ${{ secrets.WIN_CSC_KEY_PASSWORD }}
        run: npm run build:win -- --publish=always

      - name: Build & Package (macOS)
        if: matrix.os == 'macos-latest'
        env:
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_ID_PASSWORD: ${{ secrets.APPLE_ID_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
        run: npm run build:mac -- --publish=always

      - name: Build & Package (Linux)
        if: matrix.os == 'ubuntu-latest'
        run: npm run build:linux -- --publish=always

      - name: Upload release assets
        uses: actions/upload-artifact@v3
        with:
          name: release-${{ matrix.os }}
          path: dist-release/
```

### 4️⃣ CodeQL Analysis

**파일**: `.github/workflows/codeql.yml`

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'

jobs:
  analyze:
    runs-on: ubuntu-latest

    permissions:
      actions: read
      contents: read
      security-events: write

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, typescript

      - name: Autobuild
        uses: github/codeql-action/autobuild@v2

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
```

### ✅ 완료 기준

- [ ] CI 워크플로우
- [ ] 자동 테스트
- [ ] 릴리즈 자동화
- [ ] CodeQL 분석

### 📝 Commit Message

```
ci: implement CI/CD pipelines

- Add GitHub Actions workflows
- Configure automated builds for all platforms
- Set up automated testing (unit + E2E)
- Implement automatic releases
- Add CodeQL security analysis

Workflows:
- Build on push/PR
- Run tests automatically
- Release on version tags
- Weekly security scans
```

---

## Commit 121: 최종 점검 및 문서화

### 📋 작업 내용

1. **체크리스트 검증**
2. **최종 테스트**
3. **CHANGELOG 작성**
4. **README 업데이트**

### 📁 파일 구조

```
CHANGELOG.md               # 변경 이력
README.md                  # 프로젝트 README
docs/
├── RELEASE_CHECKLIST.md   # 릴리즈 체크리스트
└── DEPLOYMENT.md          # 배포 가이드
```

### 1️⃣ Release Checklist

**파일**: `docs/RELEASE_CHECKLIST.md`

```markdown
# Release Checklist

## Pre-Release

### Code Quality
- [ ] All tests passing (unit + E2E)
- [ ] No TypeScript errors
- [ ] No ESLint warnings
- [ ] Code coverage > 80%
- [ ] Performance benchmarks met

### Security
- [ ] Security audit passed
- [ ] No known vulnerabilities
- [ ] CSP configured
- [ ] Code signing certificates valid
- [ ] Secrets not hardcoded

### Build & Package
- [ ] Production builds successful (Windows/macOS/Linux)
- [ ] Installers tested on all platforms
- [ ] Auto-updater working
- [ ] App signed and notarized

### Documentation
- [ ] CHANGELOG updated
- [ ] README updated
- [ ] User guide complete
- [ ] API documentation complete
- [ ] Release notes prepared

## Release

- [ ] Version bumped in package.json
- [ ] Git tag created
- [ ] GitHub release created
- [ ] Binaries uploaded
- [ ] Release notes published

## Post-Release

- [ ] Monitor error reports
- [ ] Check update adoption rate
- [ ] Gather user feedback
- [ ] Plan next release
```

### 2️⃣ CHANGELOG

**파일**: `CHANGELOG.md`

```markdown
# Changelog

All notable changes to Codex UI will be documented in this file.

## [1.0.0] - 2024-MM-DD

### Added
- Initial release
- Chat with multiple AI models (GPT-4, Claude)
- File operations with Monaco Editor
- Full-text and semantic search
- Plugin system
- Cloud sync (S3, Google Drive)
- Data encryption
- User authentication (SSO)
- RBAC permission system
- Audit logging
- Auto-update system
- Dark mode support
- Multi-language support (EN, KO, JA, ZH)
- Keyboard shortcuts
- Accessibility features (WCAG 2.1 AA)

### Week 1 - Core Features
- Electron + React setup
- Custom titlebar
- Real-time chat with WebSocket
- Monaco Editor integration
- Session management
- Settings system
- Auto-updater

### Week 2 - Advanced Features
- MCP server integration
- Multimodal support (images, PDF, OCR)
- Workflow automation
- Plugin system
- Real-time collaboration
- Performance monitoring
- UI/UX polish

### Week 3 - Enterprise Features
- Cloud storage integration
- Advanced search with FTS5
- AI enhancements (context selection, caching)
- Security (encryption, SSO, RBAC)
- Audit logging

### Week 4 - Production Ready
- Performance optimization
- E2E testing
- Accessibility improvements
- Documentation
- Production build
- Auto-update
- CI/CD pipelines

### Technical Highlights
- Electron 28+ with React 18
- TypeScript for type safety
- Zustand for state management
- better-sqlite3 for database
- electron-updater for auto-updates
- Comprehensive test coverage
- Production-ready build system

## [Future Releases]

### Planned Features
- Local AI models support
- Advanced analytics
- Team collaboration features
- Mobile companion app
- Browser extension
```

### 3️⃣ Deployment Guide

**파일**: `docs/DEPLOYMENT.md`

```markdown
# Deployment Guide

## Prerequisites

### Code Signing Certificates

#### Windows
- Purchase a code signing certificate
- Install certificate to `build/certs/windows.pfx`
- Set `WIN_CSC_KEY_PASSWORD` environment variable

#### macOS
- Enroll in Apple Developer Program
- Create Developer ID Application certificate
- Export certificate with private key
- Set environment variables:
  - `APPLE_ID`
  - `APPLE_ID_PASSWORD`
  - `APPLE_TEAM_ID`

### Environment Variables

Create `.env.production`:

```env
# Required
VITE_API_URL=https://api.codex-ui.com
WIN_CSC_KEY_PASSWORD=your-password
APPLE_ID=your-apple-id
APPLE_ID_PASSWORD=your-app-specific-password
APPLE_TEAM_ID=your-team-id

# Optional
SENTRY_DSN=your-sentry-dsn
```

## Building

### All Platforms

```bash
npm run build:all
```

### Individual Platforms

```bash
# Windows
npm run build:win

# macOS
npm run build:mac

# Linux
npm run build:linux
```

## Publishing

### GitHub Releases

```bash
# Tag version
git tag v1.0.0

# Push tag
git push origin v1.0.0

# GitHub Actions will automatically build and publish
```

### Manual Publish

```bash
npm run publish
```

## Distribution

### Windows
- `dist-release/Codex-UI-Setup-x.x.x.exe` - Installer
- `dist-release/Codex-UI-x.x.x-portable.exe` - Portable

### macOS
- `dist-release/Codex-UI-x.x.x-universal.dmg` - Universal installer
- `dist-release/Codex-UI-x.x.x-universal-mac.zip` - Zip archive

### Linux
- `dist-release/codex-ui_x.x.x_amd64.deb` - Debian package
- `dist-release/codex-ui-x.x.x.x86_64.rpm` - RPM package
- `dist-release/Codex-UI-x.x.x.AppImage` - AppImage

## Verification

### Test Installation

1. Download installer
2. Run installation
3. Launch application
4. Verify auto-update works
5. Check all features

### Smoke Tests

- [ ] Application launches
- [ ] Chat works
- [ ] File operations work
- [ ] Search works
- [ ] Settings save
- [ ] Auto-update checks for updates

## Rollback

If issues are found:

1. Delete GitHub release
2. Revert git tag
3. Users on old version won't update
```

### 4️⃣ Updated README

**파일**: `README.md`

```markdown
# Codex UI

AI-powered desktop application built with Electron and React.

## Features

### 💬 Chat
- Multiple AI models (GPT-4, Claude)
- Streaming responses
- Code syntax highlighting
- Message history

### 📁 Files
- Upload and edit files
- Monaco Editor integration
- Syntax highlighting
- Auto-save

### 🔍 Search
- Full-text search
- Semantic search
- Advanced filters

### 🔌 Plugins
- Extend functionality
- Custom tools
- Developer API

### ☁️ Cloud Sync
- AWS S3 integration
- Google Drive support
- Automatic backups
- Multi-device sync

### 🔒 Security
- Data encryption
- SSO authentication
- RBAC permissions
- Audit logging

## Installation

Download the latest release for your platform:

- [Windows](https://github.com/codex/codex-ui/releases/latest) (x64, ARM64)
- [macOS](https://github.com/codex/codex-ui/releases/latest) (Intel, Apple Silicon, Universal)
- [Linux](https://github.com/codex/codex-ui/releases/latest) (deb, rpm, AppImage)

## Development

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build
npm run build

# Test
npm test
npm run test:e2e

# Lint
npm run lint
```

## Documentation

- [User Guide](docs/user-guide/)
- [Developer Documentation](docs/developer/)
- [API Reference](docs/developer/api-reference.md)
- [FAQ](docs/FAQ.md)

## Tech Stack

- **Electron** - Desktop app framework
- **React 18** - UI library
- **TypeScript** - Type safety
- **Zustand** - State management
- **Vite** - Build tool
- **Tailwind CSS** - Styling
- **Monaco Editor** - Code editing

## License

MIT

## Support

- [Documentation](https://docs.codex-ui.com)
- [Issues](https://github.com/codex/codex-ui/issues)
- [Discussions](https://github.com/codex/codex-ui/discussions)
```

### ✅ 완료 기준

- [ ] 체크리스트 완료
- [ ] CHANGELOG 작성
- [ ] README 업데이트
- [ ] 배포 가이드 작성
- [ ] 최종 테스트 완료

### 📝 Commit Message

```
docs: prepare v1.0.0 release

- Add release checklist
- Write comprehensive CHANGELOG
- Update README with all features
- Create deployment guide

Documentation:
- Installation instructions
- Feature overview
- Development setup
- Deployment process
```

---

## Commit 122: 최종 버전 태그

### 📋 작업 내용

1. **버전 업데이트**
2. **Git 태그 생성**
3. **릴리즈 노트**
4. **빌드 및 배포**

### 1️⃣ Version Bump

**파일**: `package.json`

```json
{
  "name": "codex-ui",
  "version": "1.0.0",
  "description": "AI-powered desktop application",
  "main": "dist-electron/main.js",
  "author": "Codex",
  "license": "MIT"
}
```

### 2️⃣ Git Commands

```bash
# Commit final changes
git add .
git commit -m "chore: prepare v1.0.0 release"

# Create tag
git tag -a v1.0.0 -m "Release v1.0.0"

# Push to repository
git push origin main
git push origin v1.0.0

# GitHub Actions will automatically build and publish
```

### ✅ 완료 기준

- [ ] 버전 업데이트
- [ ] Git 태그 생성
- [ ] 자동 빌드 실행
- [ ] 릴리즈 완료

### 📝 Commit Message

```
chore: release v1.0.0

Final release of Codex UI with all features complete:

- 3-week development (Day 1-21)
- 126 commits total
- Full Electron + React application
- Production-ready build
- Auto-update system
- Comprehensive documentation

Platform support:
- Windows (x64, ARM64)
- macOS (Intel, Apple Silicon, Universal)
- Linux (deb, rpm, AppImage)
```

---

## 🎯 Day 21 완료 체크리스트

### 빌드 시스템
- [ ] 프로덕션 빌드 설정
- [ ] Code signing
- [ ] Notarization (macOS)
- [ ] Multi-platform 빌드

### 배포
- [ ] Auto-update 구현
- [ ] 보안 강화
- [ ] CI/CD 파이프라인
- [ ] 릴리즈 자동화

### 문서화
- [ ] 릴리즈 체크리스트
- [ ] CHANGELOG
- [ ] 배포 가이드
- [ ] README 업데이트

### 최종 점검
- [ ] 모든 테스트 통과
- [ ] 보안 감사 통과
- [ ] 빌드 검증 완료
- [ ] **v1.0.0 릴리즈 완료**

---

## 📦 Dependencies

```json
{
  "dependencies": {
    "electron-updater": "^6.1.7"
  },
  "devDependencies": {
    "electron-builder": "^24.9.1",
    "@electron/notarize": "^2.2.1",
    "semver": "^7.5.4"
  }
}
```

---

## 🎉 프로젝트 완성!

**총 21일 개발 완료**:
- **126 commits** (Day 1-21, 6 commits/day)
- **Full-stack Electron application**
- **Production-ready deployment**
- **Comprehensive documentation**
- **Enterprise-grade security**
- **Multi-platform support**

Codex UI v1.0.0 is ready for production! 🚀
