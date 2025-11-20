# Day 15 TODO - 클라우드 동기화 및 백업 (Electron)

> **목표**: 멀티 디바이스 동기화, 자동 백업, 복원 시스템 구축

## 전체 개요

Day 15는 Codex UI에 클라우드 기능을 추가합니다:
- 클라우드 스토리지 통합 (S3, Google Drive)
- 세션 자동 백업
- 멀티 디바이스 동기화
- 충돌 해결 (CRDT)
- 오프라인 모드
- 데이터 암호화

**Electron 특화:**
- Native crypto API로 암호화
- Background sync (백그라운드 동기화)
- System tray 동기화 상태
- Native notification (백업 완료)
- Power monitor (배터리 절약)
- Network monitor (연결 상태)

---

## Commit 85: 클라우드 스토리지 통합

### 📋 작업 내용

1. **S3 클라이언트 구현**
2. **Google Drive API**
3. **인증 플로우**
4. **스토리지 추상화**

### 📁 파일 구조

```
src/main/cloud/
├── CloudProvider.ts      # 추상 인터페이스
├── S3Provider.ts         # AWS S3 구현
├── GoogleDriveProvider.ts # Google Drive 구현
└── types.ts              # 클라우드 타입

src/main/handlers/
└── cloud.ts              # 클라우드 IPC

src/renderer/components/cloud/
├── CloudSettings.tsx     # 클라우드 설정
└── SyncStatus.tsx        # 동기화 상태
```

### 1️⃣ 클라우드 타입 정의

**파일**: `src/renderer/types/cloud.ts`

```typescript
export type CloudProvider = 's3' | 'google-drive' | 'dropbox' | 'local';

export interface CloudConfig {
  provider: CloudProvider;
  credentials: {
    s3?: {
      accessKeyId: string;
      secretAccessKey: string;
      region: string;
      bucket: string;
    };
    googleDrive?: {
      clientId: string;
      clientSecret: string;
      refreshToken: string;
    };
  };
  encryptionKey?: string;
  autoSync: boolean;
  syncInterval: number; // minutes
}

export interface CloudFile {
  path: string;
  size: number;
  modifiedAt: number;
  hash: string;
  encrypted: boolean;
}

export interface SyncStatus {
  status: 'idle' | 'syncing' | 'error';
  lastSync?: number;
  nextSync?: number;
  uploadedFiles: number;
  downloadedFiles: number;
  totalFiles: number;
  error?: string;
}
```

### 2️⃣ Cloud Provider 추상화

**파일**: `src/main/cloud/CloudProvider.ts`

```typescript
export interface ICloudProvider {
  name: string;

  // Auth
  authenticate(): Promise<void>;
  isAuthenticated(): Promise<boolean>;

  // File operations
  upload(localPath: string, remotePath: string): Promise<void>;
  download(remotePath: string, localPath: string): Promise<void>;
  delete(remotePath: string): Promise<void>;
  list(remotePath: string): Promise<CloudFile[]>;

  // Metadata
  getMetadata(remotePath: string): Promise<CloudFile>;
  exists(remotePath: string): Promise<boolean>;
}

export abstract class CloudProvider implements ICloudProvider {
  abstract name: string;

  abstract authenticate(): Promise<void>;
  abstract isAuthenticated(): Promise<boolean>;
  abstract upload(localPath: string, remotePath: string): Promise<void>;
  abstract download(remotePath: string, localPath: string): Promise<void>;
  abstract delete(remotePath: string): Promise<void>;
  abstract list(remotePath: string): Promise<CloudFile[]>;
  abstract getMetadata(remotePath: string): Promise<CloudFile>;
  abstract exists(remotePath: string): Promise<boolean>;
}
```

### 3️⃣ S3 Provider

**파일**: `src/main/cloud/S3Provider.ts`

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand, ListObjectsV2Command } from '@aws-sdk/client-s3';
import { CloudProvider, type CloudFile } from './CloudProvider';
import fs from 'fs/promises';
import crypto from 'crypto';

export class S3Provider extends CloudProvider {
  name = 's3';
  private client: S3Client;
  private bucket: string;

  constructor(config: {
    accessKeyId: string;
    secretAccessKey: string;
    region: string;
    bucket: string;
  }) {
    super();

    this.client = new S3Client({
      region: config.region,
      credentials: {
        accessKeyId: config.accessKeyId,
        secretAccessKey: config.secretAccessKey,
      },
    });

    this.bucket = config.bucket;
  }

  async authenticate(): Promise<void> {
    // Test connection
    try {
      await this.client.send(new ListObjectsV2Command({
        Bucket: this.bucket,
        MaxKeys: 1,
      }));
    } catch (error) {
      throw new Error('Failed to authenticate with S3');
    }
  }

  async isAuthenticated(): Promise<boolean> {
    try {
      await this.authenticate();
      return true;
    } catch {
      return false;
    }
  }

  async upload(localPath: string, remotePath: string): Promise<void> {
    const content = await fs.readFile(localPath);

    await this.client.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: remotePath,
      Body: content,
    }));
  }

  async download(remotePath: string, localPath: string): Promise<void> {
    const response = await this.client.send(new GetObjectCommand({
      Bucket: this.bucket,
      Key: remotePath,
    }));

    const content = await response.Body?.transformToByteArray();
    if (content) {
      await fs.writeFile(localPath, content);
    }
  }

  async delete(remotePath: string): Promise<void> {
    await this.client.send(new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: remotePath,
    }));
  }

  async list(remotePath: string): Promise<CloudFile[]> {
    const response = await this.client.send(new ListObjectsV2Command({
      Bucket: this.bucket,
      Prefix: remotePath,
    }));

    return (response.Contents || []).map(obj => ({
      path: obj.Key!,
      size: obj.Size!,
      modifiedAt: obj.LastModified!.getTime(),
      hash: obj.ETag!.replace(/"/g, ''),
      encrypted: false,
    }));
  }

  async getMetadata(remotePath: string): Promise<CloudFile> {
    const files = await this.list(remotePath);
    const file = files.find(f => f.path === remotePath);

    if (!file) {
      throw new Error(`File not found: ${remotePath}`);
    }

    return file;
  }

  async exists(remotePath: string): Promise<boolean> {
    try {
      await this.getMetadata(remotePath);
      return true;
    } catch {
      return false;
    }
  }
}
```

### 4️⃣ Sync Manager

**파일**: `src/main/cloud/SyncManager.ts`

```typescript
import { CloudProvider } from './CloudProvider';
import { S3Provider } from './S3Provider';
import type { CloudConfig, SyncStatus, CloudFile } from '@/renderer/types/cloud';
import { app } from 'electron';
import path from 'path';
import fs from 'fs/promises';
import crypto from 'crypto';

export class SyncManager {
  private provider: CloudProvider | null = null;
  private config: CloudConfig | null = null;
  private syncInterval: NodeJS.Timeout | null = null;
  private status: SyncStatus = {
    status: 'idle',
    uploadedFiles: 0,
    downloadedFiles: 0,
    totalFiles: 0,
  };

  async initialize(config: CloudConfig): Promise<void> {
    this.config = config;

    // Create provider
    if (config.provider === 's3' && config.credentials.s3) {
      this.provider = new S3Provider(config.credentials.s3);
    }

    // Authenticate
    if (this.provider) {
      await this.provider.authenticate();
    }

    // Start auto-sync
    if (config.autoSync) {
      this.startAutoSync(config.syncInterval);
    }
  }

  async sync(): Promise<void> {
    if (!this.provider) {
      throw new Error('Cloud provider not initialized');
    }

    this.status.status = 'syncing';
    this.status.uploadedFiles = 0;
    this.status.downloadedFiles = 0;

    try {
      const localPath = path.join(app.getPath('userData'), 'sessions');
      const remotePath = 'codex-ui/sessions';

      // Get local files
      const localFiles = await this.getLocalFiles(localPath);

      // Get remote files
      const remoteFiles = await this.provider.list(remotePath);

      // Upload new/modified files
      for (const localFile of localFiles) {
        const remoteFile = remoteFiles.find(f => f.path.endsWith(localFile.name));

        if (!remoteFile || localFile.modifiedAt > remoteFile.modifiedAt) {
          await this.provider.upload(
            localFile.path,
            path.join(remotePath, localFile.name)
          );
          this.status.uploadedFiles++;
        }
      }

      // Download new/modified files
      for (const remoteFile of remoteFiles) {
        const fileName = path.basename(remoteFile.path);
        const localFile = localFiles.find(f => f.name === fileName);

        if (!localFile || remoteFile.modifiedAt > localFile.modifiedAt) {
          await this.provider.download(
            remoteFile.path,
            path.join(localPath, fileName)
          );
          this.status.downloadedFiles++;
        }
      }

      this.status.status = 'idle';
      this.status.lastSync = Date.now();
      this.status.totalFiles = localFiles.length;
    } catch (error) {
      this.status.status = 'error';
      this.status.error = (error as Error).message;
      throw error;
    }
  }

  private async getLocalFiles(dirPath: string): Promise<Array<{ name: string; path: string; modifiedAt: number }>> {
    const entries = await fs.readdir(dirPath, { withFileTypes: true });
    const files: Array<{ name: string; path: string; modifiedAt: number }> = [];

    for (const entry of entries) {
      if (entry.isFile()) {
        const filePath = path.join(dirPath, entry.name);
        const stats = await fs.stat(filePath);

        files.push({
          name: entry.name,
          path: filePath,
          modifiedAt: stats.mtimeMs,
        });
      }
    }

    return files;
  }

  private startAutoSync(intervalMinutes: number): void {
    if (this.syncInterval) {
      clearInterval(this.syncInterval);
    }

    this.syncInterval = setInterval(() => {
      this.sync().catch(console.error);
    }, intervalMinutes * 60 * 1000);
  }

  stopAutoSync(): void {
    if (this.syncInterval) {
      clearInterval(this.syncInterval);
      this.syncInterval = null;
    }
  }

  getStatus(): SyncStatus {
    return this.status;
  }
}

export const syncManager = new SyncManager();
```

### 5️⃣ Cloud Settings UI

**파일**: `src/renderer/components/cloud/CloudSettings.tsx`

```typescript
import React, { useState } from 'react';
import { Cloud, Upload, Download, Check } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Switch } from '@/components/ui/switch';
import { toast } from 'react-hot-toast';

export function CloudSettings() {
  const [provider, setProvider] = useState<'s3' | 'google-drive'>('s3');
  const [autoSync, setAutoSync] = useState(true);
  const [syncInterval, setSyncInterval] = useState(30);

  // S3 credentials
  const [accessKeyId, setAccessKeyId] = useState('');
  const [secretAccessKey, setSecretAccessKey] = useState('');
  const [region, setRegion] = useState('us-east-1');
  const [bucket, setBucket] = useState('');

  const handleSave = async () => {
    if (!window.electronAPI) return;

    try {
      await window.electronAPI.configureCloud({
        provider,
        credentials: {
          s3: {
            accessKeyId,
            secretAccessKey,
            region,
            bucket,
          },
        },
        autoSync,
        syncInterval,
      });

      toast.success('Cloud configuration saved');
    } catch (error) {
      toast.error('Failed to save configuration');
    }
  };

  const handleSync = async () => {
    if (!window.electronAPI) return;

    try {
      await window.electronAPI.syncCloud();
      toast.success('Sync completed');
    } catch (error) {
      toast.error('Sync failed');
    }
  };

  return (
    <div className="space-y-6">
      <div>
        <h3 className="font-semibold mb-4">Cloud Sync</h3>

        {/* Provider */}
        <div className="space-y-3">
          <div>
            <Label>Provider</Label>
            <Select value={provider} onValueChange={(v: any) => setProvider(v)}>
              <SelectTrigger>
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="s3">Amazon S3</SelectItem>
                <SelectItem value="google-drive">Google Drive</SelectItem>
              </SelectContent>
            </Select>
          </div>

          {/* S3 Credentials */}
          {provider === 's3' && (
            <>
              <div>
                <Label>Access Key ID</Label>
                <Input
                  type="password"
                  value={accessKeyId}
                  onChange={(e) => setAccessKeyId(e.target.value)}
                />
              </div>
              <div>
                <Label>Secret Access Key</Label>
                <Input
                  type="password"
                  value={secretAccessKey}
                  onChange={(e) => setSecretAccessKey(e.target.value)}
                />
              </div>
              <div>
                <Label>Region</Label>
                <Input
                  value={region}
                  onChange={(e) => setRegion(e.target.value)}
                />
              </div>
              <div>
                <Label>Bucket</Label>
                <Input
                  value={bucket}
                  onChange={(e) => setBucket(e.target.value)}
                />
              </div>
            </>
          )}

          {/* Auto Sync */}
          <div className="flex items-center justify-between">
            <Label>Auto Sync</Label>
            <Switch checked={autoSync} onCheckedChange={setAutoSync} />
          </div>

          {autoSync && (
            <div>
              <Label>Sync Interval (minutes)</Label>
              <Input
                type="number"
                value={syncInterval}
                onChange={(e) => setSyncInterval(parseInt(e.target.value))}
              />
            </div>
          )}

          {/* Actions */}
          <div className="flex gap-2 pt-4">
            <Button onClick={handleSave} className="flex-1">
              <Check className="h-4 w-4 mr-2" />
              Save Configuration
            </Button>
            <Button onClick={handleSync} variant="outline">
              <Cloud className="h-4 w-4 mr-2" />
              Sync Now
            </Button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] S3 provider 작동
- [ ] Google Drive provider 작동
- [ ] 인증 플로우 완성
- [ ] 클라우드 설정 UI
- [ ] Sync manager 구현

### 📝 Commit Message

```
feat(cloud): implement cloud storage integration

- Add CloudProvider abstraction
- Implement S3Provider with AWS SDK
- Create SyncManager for file synchronization
- Add CloudSettings UI
- Support auto-sync with configurable interval

Providers:
- AWS S3
- Google Drive (upcoming)

Electron-specific:
- Background sync
- Native crypto for encryption
```

---

## Commits 86-90: 백업, 암호화, 충돌 해결, 오프라인, 복원

*Remaining commits summarized*

### Commit 86: 자동 백업 시스템
- 주기적 백업 (시간/일/주)
- Incremental backup
- Backup compression
- Backup rotation (최대 N개 유지)

### Commit 87: 데이터 암호화
- AES-256 암호화
- Native crypto API
- 마스터 키 관리
- Keychain 통합

**암호화 구현**:
```typescript
import crypto from 'crypto';

export class Encryption {
  private algorithm = 'aes-256-gcm';

  encrypt(data: Buffer, key: Buffer): Buffer {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, key, iv);

    const encrypted = Buffer.concat([
      cipher.update(data),
      cipher.final(),
    ]);

    const authTag = cipher.getAuthTag();

    return Buffer.concat([iv, authTag, encrypted]);
  }

  decrypt(data: Buffer, key: Buffer): Buffer {
    const iv = data.slice(0, 16);
    const authTag = data.slice(16, 32);
    const encrypted = data.slice(32);

    const decipher = crypto.createDecipheriv(this.algorithm, key, iv);
    decipher.setAuthTag(authTag);

    return Buffer.concat([
      decipher.update(encrypted),
      decipher.final(),
    ]);
  }
}
```

### Commit 88: 충돌 해결
- CRDT 기반 병합
- 3-way merge
- Conflict UI
- Manual resolution

### Commit 89: 오프라인 모드
- 오프라인 큐
- Network monitor
- Auto-retry
- Sync on reconnect

### Commit 90: 백업 복원
- Backup 목록 조회
- Point-in-time 복원
- Selective restore
- 복원 미리보기

---

## 🎯 Day 15 완료 체크리스트

### 기능 완성도
- [ ] S3 통합
- [ ] 자동 백업
- [ ] 데이터 암호화
- [ ] 충돌 해결
- [ ] 오프라인 모드
- [ ] 백업 복원

### Electron 통합
- [ ] Native crypto
- [ ] Background sync
- [ ] Network monitor
- [ ] System tray 상태

---

## 📦 Dependencies

```json
{
  "dependencies": {
    "@aws-sdk/client-s3": "^3.470.0",
    "googleapis": "^128.0.0"
  }
}
```

---

**다음**: Day 16에서는 고급 검색 및 인덱싱을 구현합니다.
