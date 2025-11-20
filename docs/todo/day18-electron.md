# Day 18 TODO - 보안 및 엔터프라이즈 (Electron)

> **목표**: 엔터프라이즈급 보안, 감사, 규정 준수

## 전체 개요

Day 18은 Codex UI에 엔터프라이즈 기능을 추가합니다:
- 데이터 암호화 (at-rest, in-transit)
- 사용자 인증 (SSO, SAML)
- 권한 관리 (RBAC)
- 감사 로그
- 규정 준수 (GDPR, SOC2)
- 보안 업데이트

**Electron 특화:**
- Native encryption (safeStorage, Keychain)
- Secure IPC
- CSP (Content Security Policy)
- Sandboxing
- Auto-update with signature verification
- Native biometric authentication

---

## Commit 103: 데이터 암호화

### 📋 작업 내용

1. **At-rest 암호화**
2. **In-transit 암호화**
3. **키 관리**
4. **암호화 설정**

### 📁 파일 구조

```
src/main/security/
├── EncryptionManager.ts   # 암호화 관리
├── KeyManager.ts          # 키 관리
└── types.ts               # 보안 타입

src/main/handlers/
└── security.ts            # 보안 IPC

src/renderer/components/security/
├── SecuritySettings.tsx   # 보안 설정
└── EncryptionStatus.tsx   # 암호화 상태
```

### 1️⃣ 보안 타입 정의

**파일**: `src/renderer/types/security.ts`

```typescript
export interface EncryptionConfig {
  enabled: boolean;
  algorithm: 'aes-256-gcm' | 'aes-256-cbc';
  keyDerivation: 'pbkdf2' | 'scrypt';
  saltRounds: number;
}

export interface SecuritySettings {
  encryption: EncryptionConfig;
  requireAuth: boolean;
  sessionTimeout: number; // minutes
  autoLock: boolean;
  biometricAuth: boolean;
}

export interface AuditLog {
  id: string;
  timestamp: number;
  userId: string;
  action: string;
  resource: string;
  details: Record<string, any>;
  ipAddress?: string;
  userAgent?: string;
}

export interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'user' | 'viewer';
  permissions: string[];
  createdAt: number;
  lastLogin: number;
}
```

### 2️⃣ Encryption Manager

**파일**: `src/main/security/EncryptionManager.ts`

```typescript
import { safeStorage } from 'electron';
import crypto from 'crypto';
import fs from 'fs/promises';
import path from 'path';
import { app } from 'electron';

export class EncryptionManager {
  private algorithm = 'aes-256-gcm';
  private masterKey: Buffer | null = null;

  async initialize(password?: string): Promise<void> {
    if (safeStorage.isEncryptionAvailable()) {
      // Use Electron's safeStorage (Keychain on macOS, DPAPI on Windows)
      if (password) {
        const encrypted = safeStorage.encryptString(password);
        await this.saveMasterKey(encrypted);
      } else {
        const encrypted = await this.loadMasterKey();
        if (encrypted) {
          const decrypted = safeStorage.decryptString(encrypted);
          this.masterKey = this.deriveKey(decrypted);
        }
      }
    } else {
      console.warn('Encryption not available on this system');
    }
  }

  private deriveKey(password: string): Buffer {
    // Use PBKDF2 to derive encryption key
    const salt = Buffer.from('codex-ui-salt'); // In production, use random salt
    return crypto.pbkdf2Sync(password, salt, 100000, 32, 'sha256');
  }

  async encrypt(data: Buffer): Promise<Buffer> {
    if (!this.masterKey) {
      throw new Error('Encryption not initialized');
    }

    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.masterKey, iv);

    const encrypted = Buffer.concat([
      cipher.update(data),
      cipher.final(),
    ]);

    const authTag = cipher.getAuthTag();

    // Format: [iv (16)] [authTag (16)] [encrypted data]
    return Buffer.concat([iv, authTag, encrypted]);
  }

  async decrypt(data: Buffer): Promise<Buffer> {
    if (!this.masterKey) {
      throw new Error('Encryption not initialized');
    }

    const iv = data.slice(0, 16);
    const authTag = data.slice(16, 32);
    const encrypted = data.slice(32);

    const decipher = crypto.createDecipheriv(this.algorithm, this.masterKey, iv);
    decipher.setAuthTag(authTag);

    return Buffer.concat([
      decipher.update(encrypted),
      decipher.final(),
    ]);
  }

  async encryptFile(inputPath: string, outputPath: string): Promise<void> {
    const data = await fs.readFile(inputPath);
    const encrypted = await this.encrypt(data);
    await fs.writeFile(outputPath, encrypted);
  }

  async decryptFile(inputPath: string, outputPath: string): Promise<void> {
    const encrypted = await fs.readFile(inputPath);
    const decrypted = await this.decrypt(encrypted);
    await fs.writeFile(outputPath, decrypted);
  }

  private async saveMasterKey(encrypted: Buffer): Promise<void> {
    const keyPath = path.join(app.getPath('userData'), '.master-key');
    await fs.writeFile(keyPath, encrypted);
  }

  private async loadMasterKey(): Promise<Buffer | null> {
    const keyPath = path.join(app.getPath('userData'), '.master-key');

    try {
      return await fs.readFile(keyPath);
    } catch {
      return null;
    }
  }

  isInitialized(): boolean {
    return this.masterKey !== null;
  }
}

export const encryptionManager = new EncryptionManager();
```

### 3️⃣ Key Manager

**파일**: `src/main/security/KeyManager.ts`

```typescript
import { safeStorage } from 'electron';
import { app } from 'electron';
import fs from 'fs/promises';
import path from 'path';
import crypto from 'crypto';

interface StoredKey {
  id: string;
  name: string;
  encrypted: Buffer;
  createdAt: number;
}

export class KeyManager {
  private keysPath: string;
  private keys: Map<string, StoredKey> = new Map();

  constructor() {
    this.keysPath = path.join(app.getPath('userData'), 'keys.json');
    this.loadKeys();
  }

  private async loadKeys(): Promise<void> {
    try {
      const data = await fs.readFile(this.keysPath, 'utf-8');
      const keys = JSON.parse(data) as StoredKey[];

      for (const key of keys) {
        this.keys.set(key.id, key);
      }
    } catch {
      // File doesn't exist
    }
  }

  private async saveKeys(): Promise<void> {
    const keys = Array.from(this.keys.values());
    await fs.writeFile(this.keysPath, JSON.stringify(keys, null, 2));
  }

  async storeKey(id: string, name: string, value: string): Promise<void> {
    if (!safeStorage.isEncryptionAvailable()) {
      throw new Error('Encryption not available');
    }

    const encrypted = safeStorage.encryptString(value);

    this.keys.set(id, {
      id,
      name,
      encrypted,
      createdAt: Date.now(),
    });

    await this.saveKeys();
  }

  async getKey(id: string): Promise<string | null> {
    const stored = this.keys.get(id);
    if (!stored) return null;

    if (!safeStorage.isEncryptionAvailable()) {
      throw new Error('Encryption not available');
    }

    return safeStorage.decryptString(stored.encrypted);
  }

  async deleteKey(id: string): Promise<void> {
    this.keys.delete(id);
    await this.saveKeys();
  }

  listKeys(): Array<{ id: string; name: string; createdAt: number }> {
    return Array.from(this.keys.values()).map(key => ({
      id: key.id,
      name: key.name,
      createdAt: key.createdAt,
    }));
  }

  async rotateKey(id: string, newValue: string): Promise<void> {
    await this.storeKey(id, this.keys.get(id)?.name || id, newValue);
  }
}

export const keyManager = new KeyManager();
```

### 4️⃣ Security Settings UI

**파일**: `src/renderer/components/security/SecuritySettings.tsx`

```typescript
import React, { useState, useEffect } from 'react';
import { Shield, Lock, Key, AlertTriangle } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Switch } from '@/components/ui/switch';
import { Label } from '@/components/ui/label';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { toast } from 'react-hot-toast';
import type { SecuritySettings as SecuritySettingsType } from '@/types/security';

export function SecuritySettings() {
  const [settings, setSettings] = useState<SecuritySettingsType>({
    encryption: {
      enabled: false,
      algorithm: 'aes-256-gcm',
      keyDerivation: 'pbkdf2',
      saltRounds: 100000,
    },
    requireAuth: false,
    sessionTimeout: 30,
    autoLock: false,
    biometricAuth: false,
  });

  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');

  useEffect(() => {
    loadSettings();
  }, []);

  const loadSettings = async () => {
    if (!window.electronAPI) return;

    const settings = await window.electronAPI.getSecuritySettings();
    setSettings(settings);
  };

  const handleEnableEncryption = async () => {
    if (password !== confirmPassword) {
      toast.error('Passwords do not match');
      return;
    }

    if (password.length < 8) {
      toast.error('Password must be at least 8 characters');
      return;
    }

    if (!window.electronAPI) return;

    try {
      await window.electronAPI.initializeEncryption(password);

      setSettings({
        ...settings,
        encryption: {
          ...settings.encryption,
          enabled: true,
        },
      });

      toast.success('Encryption enabled');
      setPassword('');
      setConfirmPassword('');
    } catch (error) {
      toast.error('Failed to enable encryption');
    }
  };

  const handleSaveSettings = async () => {
    if (!window.electronAPI) return;

    try {
      await window.electronAPI.updateSecuritySettings(settings);
      toast.success('Security settings saved');
    } catch (error) {
      toast.error('Failed to save settings');
    }
  };

  return (
    <div className="space-y-6">
      <div>
        <h3 className="font-semibold flex items-center gap-2 mb-4">
          <Shield className="h-4 w-4" />
          Security Settings
        </h3>

        {/* Encryption */}
        <div className="space-y-4 p-4 border rounded-lg">
          <div>
            <Label className="flex items-center gap-2">
              <Lock className="h-4 w-4" />
              Data Encryption
            </Label>
            <p className="text-sm text-muted-foreground mt-1">
              Encrypt all session data at rest
            </p>
          </div>

          {!settings.encryption.enabled && (
            <div className="space-y-3">
              <div>
                <Label>Master Password</Label>
                <Input
                  type="password"
                  placeholder="Enter master password"
                  value={password}
                  onChange={(e) => setPassword(e.target.value)}
                />
              </div>

              <div>
                <Label>Confirm Password</Label>
                <Input
                  type="password"
                  placeholder="Confirm master password"
                  value={confirmPassword}
                  onChange={(e) => setConfirmPassword(e.target.value)}
                />
              </div>

              <Button onClick={handleEnableEncryption} className="w-full">
                <Key className="h-4 w-4 mr-2" />
                Enable Encryption
              </Button>

              <div className="flex items-start gap-2 p-3 bg-amber-50 dark:bg-amber-950 border border-amber-200 dark:border-amber-800 rounded">
                <AlertTriangle className="h-4 w-4 text-amber-600 dark:text-amber-400 mt-0.5 flex-shrink-0" />
                <p className="text-xs text-amber-600 dark:text-amber-400">
                  Important: Store your master password securely. If you lose it, your data cannot be recovered.
                </p>
              </div>
            </div>
          )}

          {settings.encryption.enabled && (
            <div className="flex items-center gap-2 p-3 bg-green-50 dark:bg-green-950 border border-green-200 dark:border-green-800 rounded">
              <Shield className="h-4 w-4 text-green-600 dark:text-green-400" />
              <span className="text-sm text-green-600 dark:text-green-400">
                Encryption is enabled
              </span>
            </div>
          )}

          {settings.encryption.enabled && (
            <>
              <div>
                <Label>Algorithm</Label>
                <Select
                  value={settings.encryption.algorithm}
                  onValueChange={(v: any) => setSettings({
                    ...settings,
                    encryption: { ...settings.encryption, algorithm: v },
                  })}
                >
                  <SelectTrigger>
                    <SelectValue />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="aes-256-gcm">AES-256-GCM</SelectItem>
                    <SelectItem value="aes-256-cbc">AES-256-CBC</SelectItem>
                  </SelectContent>
                </Select>
              </div>

              <div>
                <Label>Key Derivation</Label>
                <Select
                  value={settings.encryption.keyDerivation}
                  onValueChange={(v: any) => setSettings({
                    ...settings,
                    encryption: { ...settings.encryption, keyDerivation: v },
                  })}
                >
                  <SelectTrigger>
                    <SelectValue />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="pbkdf2">PBKDF2</SelectItem>
                    <SelectItem value="scrypt">Scrypt</SelectItem>
                  </SelectContent>
                </Select>
              </div>
            </>
          )}
        </div>

        {/* Authentication */}
        <div className="space-y-3 p-4 border rounded-lg mt-4">
          <div className="flex items-center justify-between">
            <div>
              <Label>Require Authentication</Label>
              <p className="text-sm text-muted-foreground">
                Require login on app start
              </p>
            </div>
            <Switch
              checked={settings.requireAuth}
              onCheckedChange={(checked) => setSettings({
                ...settings,
                requireAuth: checked,
              })}
            />
          </div>

          <div className="flex items-center justify-between">
            <div>
              <Label>Auto Lock</Label>
              <p className="text-sm text-muted-foreground">
                Lock after inactivity
              </p>
            </div>
            <Switch
              checked={settings.autoLock}
              onCheckedChange={(checked) => setSettings({
                ...settings,
                autoLock: checked,
              })}
            />
          </div>

          {settings.autoLock && (
            <div>
              <Label>Session Timeout (minutes)</Label>
              <Input
                type="number"
                value={settings.sessionTimeout}
                onChange={(e) => setSettings({
                  ...settings,
                  sessionTimeout: parseInt(e.target.value),
                })}
              />
            </div>
          )}

          <div className="flex items-center justify-between">
            <div>
              <Label>Biometric Authentication</Label>
              <p className="text-sm text-muted-foreground">
                Use Touch ID / Windows Hello
              </p>
            </div>
            <Switch
              checked={settings.biometricAuth}
              onCheckedChange={(checked) => setSettings({
                ...settings,
                biometricAuth: checked,
              })}
            />
          </div>
        </div>

        {/* Save Button */}
        <Button onClick={handleSaveSettings} className="w-full mt-4">
          Save Security Settings
        </Button>
      </div>
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] At-rest 암호화
- [ ] safeStorage 통합
- [ ] 키 관리 시스템
- [ ] 보안 설정 UI
- [ ] 마스터 비밀번호

### 📝 Commit Message

```
feat(security): implement data encryption

- Add EncryptionManager with AES-256-GCM
- Integrate safeStorage (Keychain/DPAPI)
- Implement KeyManager for API keys
- Add SecuritySettings UI

Features:
- At-rest encryption
- PBKDF2 key derivation
- Master password protection
- Native keychain integration

Electron-specific:
- safeStorage for credentials
- Native encryption APIs
- Secure key storage
```

---

## Commit 104: 사용자 인증 (SSO)

### 📋 작업 내용

1. **로컬 인증**
2. **SSO 통합 (OAuth)**
3. **SAML 지원**
4. **세션 관리**

### 📁 파일 구조

```
src/main/auth/
├── AuthManager.ts         # 인증 관리
├── OAuthProvider.ts       # OAuth
├── SAMLProvider.ts        # SAML
└── SessionManager.ts      # 세션 관리

src/renderer/components/auth/
├── LoginForm.tsx          # 로그인 폼
├── SSOLogin.tsx           # SSO 로그인
└── SessionIndicator.tsx   # 세션 상태
```

### 1️⃣ Auth Manager

**파일**: `src/main/auth/AuthManager.ts`

```typescript
import { BrowserWindow } from 'electron';
import crypto from 'crypto';
import { keyManager } from '../security/KeyManager';
import type { User } from '@/renderer/types/security';

export class AuthManager {
  private currentUser: User | null = null;
  private sessionToken: string | null = null;
  private sessionExpiry: number | null = null;

  async login(email: string, password: string): Promise<User> {
    // Hash password
    const passwordHash = this.hashPassword(password);

    // In production, verify against user database
    // For now, create a demo user
    const user: User = {
      id: crypto.randomUUID(),
      email,
      name: email.split('@')[0],
      role: 'user',
      permissions: ['read', 'write'],
      createdAt: Date.now(),
      lastLogin: Date.now(),
    };

    this.currentUser = user;
    this.sessionToken = this.generateSessionToken();
    this.sessionExpiry = Date.now() + (30 * 60 * 1000); // 30 minutes

    return user;
  }

  async loginWithSSO(provider: 'google' | 'github' | 'microsoft'): Promise<User> {
    // Open OAuth window
    const authUrl = this.getOAuthUrl(provider);
    const code = await this.openOAuthWindow(authUrl);

    // Exchange code for token
    const token = await this.exchangeCodeForToken(provider, code);

    // Get user info
    const userInfo = await this.getUserInfo(provider, token);

    const user: User = {
      id: userInfo.id,
      email: userInfo.email,
      name: userInfo.name,
      role: 'user',
      permissions: ['read', 'write'],
      createdAt: Date.now(),
      lastLogin: Date.now(),
    };

    this.currentUser = user;
    this.sessionToken = token;
    this.sessionExpiry = Date.now() + (60 * 60 * 1000); // 1 hour

    return user;
  }

  private getOAuthUrl(provider: string): string {
    const redirectUri = 'http://localhost:3000/auth/callback';

    switch (provider) {
      case 'google':
        return `https://accounts.google.com/o/oauth2/v2/auth?client_id=YOUR_CLIENT_ID&redirect_uri=${redirectUri}&response_type=code&scope=openid%20email%20profile`;
      case 'github':
        return `https://github.com/login/oauth/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=${redirectUri}&scope=user`;
      case 'microsoft':
        return `https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=${redirectUri}&response_type=code&scope=openid%20email%20profile`;
      default:
        throw new Error(`Unknown provider: ${provider}`);
    }
  }

  private async openOAuthWindow(authUrl: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const authWindow = new BrowserWindow({
        width: 500,
        height: 600,
        webPreferences: {
          nodeIntegration: false,
          contextIsolation: true,
        },
      });

      authWindow.loadURL(authUrl);

      authWindow.webContents.on('will-redirect', (event, url) => {
        const urlObj = new URL(url);
        const code = urlObj.searchParams.get('code');

        if (code) {
          authWindow.close();
          resolve(code);
        }
      });

      authWindow.on('closed', () => {
        reject(new Error('Auth window closed'));
      });
    });
  }

  private async exchangeCodeForToken(provider: string, code: string): Promise<string> {
    // In production, exchange code for access token
    // For now, return mock token
    return `mock_token_${provider}_${code}`;
  }

  private async getUserInfo(provider: string, token: string): Promise<any> {
    // In production, fetch user info from provider
    // For now, return mock data
    return {
      id: crypto.randomUUID(),
      email: 'user@example.com',
      name: 'Demo User',
    };
  }

  logout(): void {
    this.currentUser = null;
    this.sessionToken = null;
    this.sessionExpiry = null;
  }

  getCurrentUser(): User | null {
    return this.currentUser;
  }

  isAuthenticated(): boolean {
    if (!this.currentUser || !this.sessionExpiry) {
      return false;
    }

    // Check if session expired
    if (Date.now() > this.sessionExpiry) {
      this.logout();
      return false;
    }

    return true;
  }

  refreshSession(): void {
    if (this.isAuthenticated()) {
      this.sessionExpiry = Date.now() + (30 * 60 * 1000);
    }
  }

  private hashPassword(password: string): string {
    return crypto.createHash('sha256').update(password).digest('hex');
  }

  private generateSessionToken(): string {
    return crypto.randomBytes(32).toString('hex');
  }
}

export const authManager = new AuthManager();
```

### 2️⃣ Login Form UI

**파일**: `src/renderer/components/auth/LoginForm.tsx`

```typescript
import React, { useState } from 'react';
import { Lock, Mail, LogIn } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { toast } from 'react-hot-toast';

export function LoginForm({ onSuccess }: { onSuccess: () => void }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault();

    if (!window.electronAPI) return;

    setLoading(true);

    try {
      const user = await window.electronAPI.login(email, password);
      toast.success(`Welcome, ${user.name}!`);
      onSuccess();
    } catch (error) {
      toast.error('Login failed');
    } finally {
      setLoading(false);
    }
  };

  const handleSSOLogin = async (provider: 'google' | 'github' | 'microsoft') => {
    if (!window.electronAPI) return;

    setLoading(true);

    try {
      const user = await window.electronAPI.loginWithSSO(provider);
      toast.success(`Welcome, ${user.name}!`);
      onSuccess();
    } catch (error) {
      toast.error('SSO login failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="w-full max-w-md mx-auto space-y-6 p-8">
      <div className="text-center">
        <h2 className="text-2xl font-bold">Sign In</h2>
        <p className="text-muted-foreground mt-2">
          Sign in to your Codex UI account
        </p>
      </div>

      <form onSubmit={handleLogin} className="space-y-4">
        <div>
          <Label htmlFor="email">Email</Label>
          <div className="relative">
            <Mail className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
            <Input
              id="email"
              type="email"
              placeholder="you@example.com"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="pl-10"
              required
            />
          </div>
        </div>

        <div>
          <Label htmlFor="password">Password</Label>
          <div className="relative">
            <Lock className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
            <Input
              id="password"
              type="password"
              placeholder="••••••••"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="pl-10"
              required
            />
          </div>
        </div>

        <Button type="submit" className="w-full" disabled={loading}>
          <LogIn className="h-4 w-4 mr-2" />
          {loading ? 'Signing in...' : 'Sign In'}
        </Button>
      </form>

      <div className="relative">
        <div className="absolute inset-0 flex items-center">
          <div className="w-full border-t" />
        </div>
        <div className="relative flex justify-center text-xs">
          <span className="bg-background px-2 text-muted-foreground">
            Or continue with
          </span>
        </div>
      </div>

      <div className="grid grid-cols-3 gap-3">
        <Button
          variant="outline"
          onClick={() => handleSSOLogin('google')}
          disabled={loading}
        >
          Google
        </Button>
        <Button
          variant="outline"
          onClick={() => handleSSOLogin('github')}
          disabled={loading}
        >
          GitHub
        </Button>
        <Button
          variant="outline"
          onClick={() => handleSSOLogin('microsoft')}
          disabled={loading}
        >
          Microsoft
        </Button>
      </div>
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 로컬 인증
- [ ] OAuth 통합
- [ ] 세션 관리
- [ ] 로그인 UI

### 📝 Commit Message

```
feat(auth): implement user authentication

- Add AuthManager with local auth
- Implement OAuth SSO (Google, GitHub, Microsoft)
- Create session management
- Add LoginForm UI

Features:
- Password hashing
- OAuth window flow
- Session expiry
- SSO providers
```

---

## Commit 105: 권한 관리 (RBAC)

### 📋 작업 내용

1. **역할 정의**
2. **권한 체크**
3. **리소스 보호**
4. **권한 UI**

### 📁 파일 구조

```
src/main/auth/
├── RBACManager.ts         # 권한 관리
└── permissions.ts         # 권한 정의

src/renderer/components/auth/
├── RoleSelector.tsx       # 역할 선택
└── PermissionGate.tsx     # 권한 게이트
```

### 1️⃣ RBAC Manager

**파일**: `src/main/auth/RBACManager.ts`

```typescript
import type { User } from '@/renderer/types/security';

export type Permission =
  | 'read'
  | 'write'
  | 'delete'
  | 'admin'
  | 'settings'
  | 'users'
  | 'export';

export type Role = 'admin' | 'user' | 'viewer';

const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: ['read', 'write', 'delete', 'admin', 'settings', 'users', 'export'],
  user: ['read', 'write', 'export'],
  viewer: ['read'],
};

export class RBACManager {
  hasPermission(user: User | null, permission: Permission): boolean {
    if (!user) return false;

    // Check user's custom permissions
    if (user.permissions.includes(permission)) {
      return true;
    }

    // Check role-based permissions
    const rolePermissions = ROLE_PERMISSIONS[user.role] || [];
    return rolePermissions.includes(permission);
  }

  hasAnyPermission(user: User | null, permissions: Permission[]): boolean {
    return permissions.some(p => this.hasPermission(user, p));
  }

  hasAllPermissions(user: User | null, permissions: Permission[]): boolean {
    return permissions.every(p => this.hasPermission(user, p));
  }

  canAccessResource(user: User | null, resource: string, action: Permission): boolean {
    // Resource-specific access control
    // For example: canAccessResource(user, 'session:abc123', 'delete')

    if (!this.hasPermission(user, action)) {
      return false;
    }

    // Additional resource-specific checks
    // e.g., check if user owns the resource

    return true;
  }

  getRolePermissions(role: Role): Permission[] {
    return ROLE_PERMISSIONS[role] || [];
  }

  assignRole(user: User, role: Role): User {
    return {
      ...user,
      role,
    };
  }

  grantPermission(user: User, permission: Permission): User {
    if (user.permissions.includes(permission)) {
      return user;
    }

    return {
      ...user,
      permissions: [...user.permissions, permission],
    };
  }

  revokePermission(user: User, permission: Permission): User {
    return {
      ...user,
      permissions: user.permissions.filter(p => p !== permission),
    };
  }
}

export const rbacManager = new RBACManager();
```

### 2️⃣ Permission Gate Component

**파일**: `src/renderer/components/auth/PermissionGate.tsx`

```typescript
import React from 'react';
import { useAuth } from '@/hooks/useAuth';
import type { Permission } from '@/main/auth/RBACManager';

interface PermissionGateProps {
  permission: Permission | Permission[];
  requireAll?: boolean;
  fallback?: React.ReactNode;
  children: React.ReactNode;
}

export function PermissionGate({
  permission,
  requireAll = false,
  fallback = null,
  children,
}: PermissionGateProps) {
  const { user, hasPermission } = useAuth();

  if (!user) {
    return <>{fallback}</>;
  }

  const permissions = Array.isArray(permission) ? permission : [permission];

  const hasAccess = requireAll
    ? permissions.every(p => hasPermission(p))
    : permissions.some(p => hasPermission(p));

  if (!hasAccess) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
}
```

### ✅ 완료 기준

- [ ] RBAC 시스템
- [ ] 권한 체크 함수
- [ ] PermissionGate 컴포넌트
- [ ] 역할별 권한 정의

### 📝 Commit Message

```
feat(auth): implement RBAC system

- Add RBACManager with role-based access control
- Define roles (admin, user, viewer)
- Implement permission checking
- Create PermissionGate component

Permissions:
- read, write, delete
- admin, settings, users
- Resource-specific access control
```

---

## Commit 106: 감사 로그

### 📋 작업 내용

1. **활동 로깅**
2. **로그 저장소**
3. **로그 조회**
4. **로그 내보내기**

### 📁 파일 구조

```
src/main/audit/
├── AuditLogger.ts         # 감사 로거
└── AuditStore.ts          # 로그 저장소

src/renderer/components/audit/
├── AuditLog.tsx           # 로그 뷰어
└── AuditFilters.tsx       # 필터
```

### 1️⃣ Audit Logger

**파일**: `src/main/audit/AuditLogger.ts`

```typescript
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';
import type { AuditLog } from '@/renderer/types/security';

export class AuditLogger {
  private db: Database.Database;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'audit.db');
    this.db = new Database(dbPath);
    this.initialize();
  }

  private initialize(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS audit_logs (
        id TEXT PRIMARY KEY,
        timestamp INTEGER NOT NULL,
        user_id TEXT NOT NULL,
        action TEXT NOT NULL,
        resource TEXT NOT NULL,
        details TEXT NOT NULL,
        ip_address TEXT,
        user_agent TEXT
      );

      CREATE INDEX IF NOT EXISTS idx_timestamp ON audit_logs(timestamp);
      CREATE INDEX IF NOT EXISTS idx_user_id ON audit_logs(user_id);
      CREATE INDEX IF NOT EXISTS idx_action ON audit_logs(action);
    `);
  }

  log(
    userId: string,
    action: string,
    resource: string,
    details: Record<string, any> = {},
    ipAddress?: string,
    userAgent?: string
  ): void {
    const log: AuditLog = {
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      userId,
      action,
      resource,
      details,
      ipAddress,
      userAgent,
    };

    this.db.prepare(`
      INSERT INTO audit_logs (id, timestamp, user_id, action, resource, details, ip_address, user_agent)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    `).run(
      log.id,
      log.timestamp,
      log.userId,
      log.action,
      log.resource,
      JSON.stringify(log.details),
      log.ipAddress || null,
      log.userAgent || null
    );
  }

  getLogs(options: {
    userId?: string;
    action?: string;
    startDate?: number;
    endDate?: number;
    limit?: number;
    offset?: number;
  }): AuditLog[] {
    let query = 'SELECT * FROM audit_logs WHERE 1=1';
    const params: any[] = [];

    if (options.userId) {
      query += ' AND user_id = ?';
      params.push(options.userId);
    }

    if (options.action) {
      query += ' AND action = ?';
      params.push(options.action);
    }

    if (options.startDate) {
      query += ' AND timestamp >= ?';
      params.push(options.startDate);
    }

    if (options.endDate) {
      query += ' AND timestamp <= ?';
      params.push(options.endDate);
    }

    query += ' ORDER BY timestamp DESC';

    if (options.limit) {
      query += ' LIMIT ?';
      params.push(options.limit);
    }

    if (options.offset) {
      query += ' OFFSET ?';
      params.push(options.offset);
    }

    const rows = this.db.prepare(query).all(...params) as any[];

    return rows.map(row => ({
      id: row.id,
      timestamp: row.timestamp,
      userId: row.user_id,
      action: row.action,
      resource: row.resource,
      details: JSON.parse(row.details),
      ipAddress: row.ip_address,
      userAgent: row.user_agent,
    }));
  }

  exportLogs(startDate: number, endDate: number): AuditLog[] {
    return this.getLogs({ startDate, endDate });
  }

  cleanup(olderThan: number): void {
    this.db.prepare('DELETE FROM audit_logs WHERE timestamp < ?').run(olderThan);
  }

  close(): void {
    this.db.close();
  }
}

export const auditLogger = new AuditLogger();
```

### 2️⃣ Audit Log Viewer

**파일**: `src/renderer/components/audit/AuditLog.tsx`

```typescript
import React, { useState, useEffect } from 'react';
import { Shield, Download, Filter } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { formatDistanceToNow } from 'date-fns';
import type { AuditLog as AuditLogType } from '@/types/security';

export function AuditLog() {
  const [logs, setLogs] = useState<AuditLogType[]>([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    loadLogs();
  }, []);

  const loadLogs = async () => {
    if (!window.electronAPI) return;

    setLoading(true);

    try {
      const logs = await window.electronAPI.getAuditLogs({ limit: 100 });
      setLogs(logs);
    } catch (error) {
      console.error('Failed to load audit logs:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleExport = async () => {
    if (!window.electronAPI) return;

    const startDate = Date.now() - (30 * 24 * 60 * 60 * 1000); // Last 30 days
    const endDate = Date.now();

    const logs = await window.electronAPI.exportAuditLogs(startDate, endDate);

    // Download as JSON
    const blob = new Blob([JSON.stringify(logs, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `audit-logs-${Date.now()}.json`;
    a.click();
  };

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <h3 className="font-semibold flex items-center gap-2">
          <Shield className="h-4 w-4" />
          Audit Log
        </h3>
        <div className="flex gap-2">
          <Button size="sm" variant="outline">
            <Filter className="h-4 w-4 mr-2" />
            Filter
          </Button>
          <Button size="sm" onClick={handleExport}>
            <Download className="h-4 w-4 mr-2" />
            Export
          </Button>
        </div>
      </div>

      {loading ? (
        <div className="text-center py-8 text-muted-foreground">
          Loading audit logs...
        </div>
      ) : (
        <div className="space-y-2">
          {logs.map(log => (
            <div key={log.id} className="p-3 border rounded-lg">
              <div className="flex items-start justify-between">
                <div>
                  <div className="font-medium">{log.action}</div>
                  <div className="text-sm text-muted-foreground">
                    {log.resource}
                  </div>
                  <div className="text-xs text-muted-foreground mt-1">
                    {formatDistanceToNow(log.timestamp, { addSuffix: true })} by {log.userId}
                  </div>
                </div>
                <div className="text-xs text-muted-foreground">
                  {log.ipAddress}
                </div>
              </div>
              {Object.keys(log.details).length > 0 && (
                <pre className="mt-2 text-xs bg-muted p-2 rounded overflow-x-auto">
                  {JSON.stringify(log.details, null, 2)}
                </pre>
              )}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 활동 로깅
- [ ] 로그 저장소
- [ ] 로그 조회 UI
- [ ] 로그 내보내기

### 📝 Commit Message

```
feat(audit): implement audit logging

- Add AuditLogger with SQLite storage
- Track user actions and resources
- Create AuditLog viewer UI
- Support log export

Logged actions:
- User authentication
- Data access
- Settings changes
- Resource modifications
```

---

## 🎯 Day 18 완료 체크리스트

### 기능 완성도
- [ ] 데이터 암호화
- [ ] 사용자 인증 (SSO)
- [ ] 권한 관리 (RBAC)
- [ ] 감사 로그
- [ ] 보안 설정 UI

### Electron 통합
- [ ] safeStorage (Keychain/DPAPI)
- [ ] Secure IPC
- [ ] CSP
- [ ] OAuth window flow
- [ ] Native biometric auth

---

## 📦 Dependencies

```json
{
  "dependencies": {
    "bcrypt": "^5.1.1",
    "jsonwebtoken": "^9.0.2"
  }
}
```

---

## 🎉 Week 3 완료!

Week 3에서는 다음 엔터프라이즈 기능들을 구현했습니다:

### Day 15: 클라우드 동기화
- AWS S3, Google Drive 통합
- 자동 백업 시스템
- 데이터 암호화
- 충돌 해결
- 오프라인 모드

### Day 16: 고급 검색
- SQLite FTS5 전문 검색
- 시맨틱 검색 (Embeddings)
- 파일 내용 인덱싱
- 고급 필터
- 검색 히스토리

### Day 17: AI 향상
- 스마트 컨텍스트 선택
- 프롬프트 템플릿 시스템
- AI 응답 캐싱
- 스트리밍 최적화
- 토큰 사용량 추적
- Model comparison

### Day 18: 보안 & 엔터프라이즈
- At-rest/in-transit 암호화
- SSO (OAuth, SAML)
- RBAC 권한 관리
- 감사 로그
- 규정 준수

---

**다음**: Week 4 (Day 19-21)에서는 마지막 마무리 작업을 진행합니다.
