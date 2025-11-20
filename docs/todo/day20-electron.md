# Day 20 TODO - 최종 통합 및 마무리 (Electron)

> **목표**: E2E 테스트, UI/UX 최종 개선, 문서화 완성

## 전체 개요

Day 20은 Codex UI의 최종 통합과 마무리 작업을 진행합니다:
- End-to-End 테스트
- UI/UX 최종 개선
- 접근성 강화
- 사용자 가이드 작성
- 온보딩 플로우
- 성능 벤치마크

**Electron 특화:**
- Native integration 테스트
- Multi-platform 검증
- Auto-updater 테스트
- Installer 최적화
- System integration

---

## Commit 113: E2E 테스트 완성

### 📋 작업 내용

1. **Playwright 테스트**
2. **시나리오 테스트**
3. **통합 테스트**
4. **성능 테스트**

### 📁 파일 구조

```
e2e/
├── tests/
│   ├── chat.spec.ts          # 채팅 테스트
│   ├── file-operations.spec.ts # 파일 작업
│   ├── search.spec.ts        # 검색 테스트
│   ├── plugins.spec.ts       # 플러그인 테스트
│   └── settings.spec.ts      # 설정 테스트
├── fixtures/
│   └── test-data.ts          # 테스트 데이터
└── utils/
    └── helpers.ts            # 헬퍼 함수
```

### 1️⃣ Chat E2E Test

**파일**: `e2e/tests/chat.spec.ts`

```typescript
import { test, expect, _electron as electron } from '@playwright/test';
import { ElectronApplication, Page } from 'playwright';
import path from 'path';

let electronApp: ElectronApplication;
let window: Page;

test.beforeAll(async () => {
  electronApp = await electron.launch({
    args: [path.join(__dirname, '../../dist-electron/main.js')],
  });

  window = await electronApp.firstWindow();
});

test.afterAll(async () => {
  await electronApp.close();
});

test.describe('Chat功能', () => {
  test('새 세션 생성', async () => {
    // Click new session button
    await window.click('[data-testid="new-session-btn"]');

    // Wait for session to be created
    await window.waitForSelector('[data-testid="message-input"]');

    // Verify empty chat
    const messageCount = await window.locator('[data-testid="message"]').count();
    expect(messageCount).toBe(0);
  });

  test('메시지 전송 및 응답 받기', async () => {
    // Type message
    await window.fill('[data-testid="message-input"]', 'Hello, world!');

    // Send message
    await window.click('[data-testid="send-btn"]');

    // Wait for user message to appear
    await window.waitForSelector('[data-testid="message"][data-role="user"]');

    const userMessage = await window.textContent(
      '[data-testid="message"][data-role="user"] .message-content'
    );
    expect(userMessage).toContain('Hello, world!');

    // Wait for assistant response
    await window.waitForSelector(
      '[data-testid="message"][data-role="assistant"]',
      { timeout: 30000 }
    );

    const assistantMessage = await window.textContent(
      '[data-testid="message"][data-role="assistant"] .message-content'
    );
    expect(assistantMessage).not.toBe('');
  });

  test('스트리밍 응답 렌더링', async () => {
    await window.fill('[data-testid="message-input"]', 'Tell me a story');
    await window.click('[data-testid="send-btn"]');

    // Wait for streaming to start
    await window.waitForSelector(
      '[data-testid="message"][data-role="assistant"][data-streaming="true"]'
    );

    // Verify streaming indicator
    const isStreaming = await window.isVisible('[data-testid="streaming-indicator"]');
    expect(isStreaming).toBe(true);

    // Wait for streaming to complete
    await window.waitForSelector(
      '[data-testid="message"][data-role="assistant"][data-streaming="false"]',
      { timeout: 60000 }
    );
  });

  test('이미지 첨부', async () => {
    // Click attach button
    await window.click('[data-testid="attach-btn"]');

    // Upload image
    const fileChooserPromise = window.waitForEvent('filechooser');
    await window.click('[data-testid="attach-image-btn"]');
    const fileChooser = await fileChooserPromise;

    const testImagePath = path.join(__dirname, '../fixtures/test-image.png');
    await fileChooser.setFiles(testImagePath);

    // Verify image preview
    await window.waitForSelector('[data-testid="image-preview"]');
    const imageSrc = await window.getAttribute('[data-testid="image-preview"] img', 'src');
    expect(imageSrc).toContain('test-image.png');
  });

  test('세션 저장 및 불러오기', async () => {
    // Send a message
    await window.fill('[data-testid="message-input"]', 'Remember this message');
    await window.click('[data-testid="send-btn"]');

    await window.waitForSelector('[data-testid="message"][data-role="assistant"]');

    // Get session ID
    const sessionId = await window.getAttribute('[data-testid="chat-view"]', 'data-session-id');

    // Navigate away and back
    await window.click('[data-testid="sessions-btn"]');
    await window.click(`[data-testid="session-${sessionId}"]`);

    // Verify messages are still there
    const messageText = await window.textContent(
      '[data-testid="message"][data-role="user"] .message-content'
    );
    expect(messageText).toContain('Remember this message');
  });
});
```

### 2️⃣ File Operations E2E Test

**파일**: `e2e/tests/file-operations.spec.ts`

```typescript
import { test, expect, _electron as electron } from '@playwright/test';
import { ElectronApplication, Page } from 'playwright';
import path from 'path';
import fs from 'fs/promises';

let electronApp: ElectronApplication;
let window: Page;

test.beforeAll(async () => {
  electronApp = await electron.launch({
    args: [path.join(__dirname, '../../dist-electron/main.js')],
  });
  window = await electronApp.firstWindow();
});

test.afterAll(async () => {
  await electronApp.close();
});

test.describe('파일 작업', () => {
  test('파일 탐색기 열기', async () => {
    await window.click('[data-testid="files-tab"]');

    await window.waitForSelector('[data-testid="file-explorer"]');

    const explorerVisible = await window.isVisible('[data-testid="file-explorer"]');
    expect(explorerVisible).toBe(true);
  });

  test('파일 업로드', async () => {
    await window.click('[data-testid="upload-file-btn"]');

    const fileChooserPromise = window.waitForEvent('filechooser');
    const fileChooser = await fileChooserPromise;

    const testFile = path.join(__dirname, '../fixtures/test.txt');
    await fileChooser.setFiles(testFile);

    await window.waitForSelector('[data-testid="file-item"][data-name="test.txt"]');

    const fileName = await window.textContent('[data-testid="file-item"][data-name="test.txt"] .file-name');
    expect(fileName).toBe('test.txt');
  });

  test('파일 열기 (Monaco Editor)', async () => {
    await window.click('[data-testid="file-item"][data-name="test.txt"]');

    await window.waitForSelector('[data-testid="monaco-editor"]');

    // Verify editor is visible
    const editorVisible = await window.isVisible('[data-testid="monaco-editor"]');
    expect(editorVisible).toBe(true);

    // Verify content is loaded
    const content = await window.evaluate(() => {
      return (window as any).monaco?.editor?.getModels()[0]?.getValue();
    });
    expect(content).not.toBe('');
  });

  test('파일 편집 및 저장', async () => {
    // Modify content
    await window.evaluate(() => {
      const model = (window as any).monaco?.editor?.getModels()[0];
      model?.setValue('Modified content');
    });

    // Save
    await window.keyboard.press('Control+S');

    // Wait for save indicator
    await window.waitForSelector('[data-testid="save-indicator"][data-saved="true"]');

    const saved = await window.getAttribute('[data-testid="save-indicator"]', 'data-saved');
    expect(saved).toBe('true');
  });

  test('파일 검색', async () => {
    await window.fill('[data-testid="file-search-input"]', 'test');

    await window.waitForSelector('[data-testid="file-item"]');

    const fileCount = await window.locator('[data-testid="file-item"]').count();
    expect(fileCount).toBeGreaterThan(0);
  });
});
```

### 3️⃣ Search E2E Test

**파일**: `e2e/tests/search.spec.ts`

```typescript
import { test, expect, _electron as electron } from '@playwright/test';
import { ElectronApplication, Page } from 'playwright';
import path from 'path';

let electronApp: ElectronApplication;
let window: Page;

test.beforeAll(async () => {
  electronApp = await electron.launch({
    args: [path.join(__dirname, '../../dist-electron/main.js')],
  });
  window = await electronApp.firstWindow();
});

test.afterAll(async () => {
  await electronApp.close();
});

test.describe('검색 기능', () => {
  test('글로벌 검색 (Cmd+Shift+F)', async () => {
    await window.keyboard.press('Control+Shift+F');

    await window.waitForSelector('[data-testid="search-bar"]');

    const searchVisible = await window.isVisible('[data-testid="search-bar"]');
    expect(searchVisible).toBe(true);
  });

  test('메시지 검색', async () => {
    await window.fill('[data-testid="search-input"]', 'hello');
    await window.keyboard.press('Enter');

    await window.waitForSelector('[data-testid="search-result"]');

    const resultCount = await window.locator('[data-testid="search-result"]').count();
    expect(resultCount).toBeGreaterThan(0);
  });

  test('검색 필터 적용', async () => {
    await window.click('[data-testid="search-filters-btn"]');

    await window.waitForSelector('[data-testid="search-filters"]');

    // Select date range
    await window.fill('[data-testid="date-from"]', '2024-01-01');
    await window.fill('[data-testid="date-to"]', '2024-12-31');

    // Select model
    await window.click('[data-testid="model-filter-gpt-4"]');

    // Apply filters
    await window.click('[data-testid="apply-filters-btn"]');

    await window.waitForSelector('[data-testid="search-result"]');

    const results = await window.locator('[data-testid="search-result"]').all();
    expect(results.length).toBeGreaterThan(0);
  });

  test('검색 결과 하이라이팅', async () => {
    await window.fill('[data-testid="search-input"]', 'world');
    await window.keyboard.press('Enter');

    await window.waitForSelector('[data-testid="search-result"]');

    // Check for highlights
    const highlightCount = await window.locator('[data-testid="search-result"] mark').count();
    expect(highlightCount).toBeGreaterThan(0);
  });
});
```

### 4️⃣ Performance Test

**파일**: `e2e/tests/performance.spec.ts`

```typescript
import { test, expect, _electron as electron } from '@playwright/test';
import { ElectronApplication, Page } from 'playwright';
import path from 'path';

let electronApp: ElectronApplication;
let window: Page;

test.beforeAll(async () => {
  electronApp = await electron.launch({
    args: [path.join(__dirname, '../../dist-electron/main.js')],
  });
  window = await electronApp.firstWindow();
});

test.afterAll(async () => {
  await electronApp.close();
});

test.describe('성능 테스트', () => {
  test('앱 시작 시간', async () => {
    const startTime = Date.now();

    await window.waitForSelector('[data-testid="app-ready"]');

    const loadTime = Date.now() - startTime;
    console.log(`App load time: ${loadTime}ms`);

    expect(loadTime).toBeLessThan(3000); // Should load in under 3 seconds
  });

  test('대량 메시지 렌더링', async () => {
    // Create session with many messages
    await electronApp.evaluate(async ({ app }) => {
      const { ipcMain } = require('electron');

      // Generate 1000 test messages
      const messages = Array.from({ length: 1000 }, (_, i) => ({
        id: `msg-${i}`,
        role: i % 2 === 0 ? 'user' : 'assistant',
        content: `Test message ${i}`,
        timestamp: Date.now() - (1000 - i) * 1000,
      }));

      // Load messages
      ipcMain.emit('test:load-messages', messages);
    });

    const startTime = Date.now();

    await window.waitForSelector('[data-testid="message"]');

    const renderTime = Date.now() - startTime;
    console.log(`1000 messages render time: ${renderTime}ms`);

    expect(renderTime).toBeLessThan(2000); // Should render in under 2 seconds
  });

  test('검색 성능', async () => {
    await window.fill('[data-testid="search-input"]', 'test query');

    const startTime = Date.now();

    await window.keyboard.press('Enter');
    await window.waitForSelector('[data-testid="search-result"]');

    const searchTime = Date.now() - startTime;
    console.log(`Search time: ${searchTime}ms`);

    expect(searchTime).toBeLessThan(500); // Should search in under 500ms
  });

  test('메모리 사용량', async () => {
    const metrics = await electronApp.evaluate(async ({ app }) => {
      return app.getAppMetrics();
    });

    const mainProcess = metrics.find(m => m.type === 'Browser');
    const rendererProcess = metrics.find(m => m.type === 'Renderer');

    console.log('Main process memory:', mainProcess?.memory.workingSetSize);
    console.log('Renderer process memory:', rendererProcess?.memory.workingSetSize);

    // Memory should be under 500MB
    expect(mainProcess?.memory.workingSetSize).toBeLessThan(500 * 1024 * 1024);
  });
});
```

### 5️⃣ Test Helpers

**파일**: `e2e/utils/helpers.ts`

```typescript
import { Page, ElectronApplication } from 'playwright';

export async function createTestSession(window: Page): Promise<string> {
  await window.click('[data-testid="new-session-btn"]');
  await window.waitForSelector('[data-testid="message-input"]');
  return await window.getAttribute('[data-testid="chat-view"]', 'data-session-id') || '';
}

export async function sendMessage(window: Page, message: string): Promise<void> {
  await window.fill('[data-testid="message-input"]', message);
  await window.click('[data-testid="send-btn"]');
  await window.waitForSelector('[data-testid="message"][data-role="assistant"]', {
    timeout: 30000,
  });
}

export async function uploadFile(window: Page, filePath: string): Promise<void> {
  await window.click('[data-testid="attach-btn"]');
  const fileChooserPromise = window.waitForEvent('filechooser');
  await window.click('[data-testid="attach-file-btn"]');
  const fileChooser = await fileChooserPromise;
  await fileChooser.setFiles(filePath);
}

export async function waitForSave(window: Page): Promise<void> {
  await window.waitForSelector('[data-testid="save-indicator"][data-saved="true"]');
}

export async function getMemoryUsage(app: ElectronApplication): Promise<number> {
  const metrics = await app.evaluate(async ({ app }) => {
    return app.getAppMetrics();
  });

  const total = metrics.reduce((sum, m) => sum + m.memory.workingSetSize, 0);
  return total;
}
```

### ✅ 완료 기준

- [ ] Chat E2E 테스트
- [ ] File operations 테스트
- [ ] Search 테스트
- [ ] Performance 테스트
- [ ] 모든 테스트 통과

### 📝 Commit Message

```
test(e2e): complete end-to-end tests

- Add comprehensive chat flow tests
- Test file operations and Monaco editor
- Implement search functionality tests
- Add performance benchmarks

Test coverage:
- Chat (새 세션, 메시지, 스트리밍)
- Files (업로드, 편집, 저장)
- Search (필터, 하이라이팅)
- Performance (시작시간, 렌더링, 메모리)
```

---

## Commit 114: UI/UX 최종 개선

### 📋 작업 내용

1. **애니메이션 개선**
2. **로딩 상태**
3. **에러 상태**
4. **빈 상태**

### 📁 파일 구조

```
src/renderer/components/ui/
├── LoadingState.tsx       # 로딩 상태
├── ErrorState.tsx         # 에러 상태
├── EmptyState.tsx         # 빈 상태
└── animations.ts          # 애니메이션 설정
```

### 1️⃣ Smooth Animations

**파일**: `src/renderer/components/ui/animations.ts`

```typescript
export const fadeIn = {
  initial: { opacity: 0 },
  animate: { opacity: 1 },
  exit: { opacity: 0 },
  transition: { duration: 0.2 },
};

export const slideUp = {
  initial: { y: 20, opacity: 0 },
  animate: { y: 0, opacity: 1 },
  exit: { y: -20, opacity: 0 },
  transition: { duration: 0.3, ease: 'easeOut' },
};

export const slideIn = {
  initial: { x: -20, opacity: 0 },
  animate: { x: 0, opacity: 1 },
  exit: { x: 20, opacity: 0 },
  transition: { duration: 0.3, ease: 'easeOut' },
};

export const scaleIn = {
  initial: { scale: 0.9, opacity: 0 },
  animate: { scale: 1, opacity: 1 },
  exit: { scale: 0.9, opacity: 0 },
  transition: { duration: 0.2 },
};

export const staggerChildren = {
  animate: {
    transition: {
      staggerChildren: 0.05,
    },
  },
};
```

### 2️⃣ Loading States

**파일**: `src/renderer/components/ui/LoadingState.tsx`

```typescript
import React from 'react';
import { motion } from 'framer-motion';
import { Loader2 } from 'lucide-react';

interface LoadingStateProps {
  message?: string;
  size?: 'sm' | 'md' | 'lg';
}

export function LoadingState({ message = 'Loading...', size = 'md' }: LoadingStateProps) {
  const sizeClasses = {
    sm: 'h-4 w-4',
    md: 'h-8 w-8',
    lg: 'h-12 w-12',
  };

  return (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
      className="flex flex-col items-center justify-center p-8"
    >
      <Loader2 className={`${sizeClasses[size]} animate-spin text-primary`} />
      {message && (
        <p className="mt-4 text-sm text-muted-foreground">{message}</p>
      )}
    </motion.div>
  );
}

// Skeleton loader
export function SkeletonLoader({ count = 3 }: { count?: number }) {
  return (
    <div className="space-y-3">
      {Array.from({ length: count }).map((_, i) => (
        <div key={i} className="space-y-2">
          <div className="h-4 bg-muted animate-pulse rounded w-3/4" />
          <div className="h-4 bg-muted animate-pulse rounded w-1/2" />
        </div>
      ))}
    </div>
  );
}
```

### 3️⃣ Error States

**파일**: `src/renderer/components/ui/ErrorState.tsx`

```typescript
import React from 'react';
import { motion } from 'framer-motion';
import { AlertCircle, RefreshCw } from 'lucide-react';
import { Button } from './button';

interface ErrorStateProps {
  title?: string;
  message: string;
  onRetry?: () => void;
  showRetry?: boolean;
}

export function ErrorState({
  title = 'Something went wrong',
  message,
  onRetry,
  showRetry = true,
}: ErrorStateProps) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 10 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -10 }}
      className="flex flex-col items-center justify-center p-8 text-center"
    >
      <AlertCircle className="h-12 w-12 text-destructive mb-4" />
      <h3 className="font-semibold text-lg mb-2">{title}</h3>
      <p className="text-muted-foreground max-w-md mb-4">{message}</p>
      {showRetry && onRetry && (
        <Button onClick={onRetry} variant="outline">
          <RefreshCw className="h-4 w-4 mr-2" />
          Try Again
        </Button>
      )}
    </motion.div>
  );
}
```

### 4️⃣ Empty States

**파일**: `src/renderer/components/ui/EmptyState.tsx`

```typescript
import React from 'react';
import { motion } from 'framer-motion';
import { LucideIcon } from 'lucide-react';
import { Button } from './button';

interface EmptyStateProps {
  icon: LucideIcon;
  title: string;
  description: string;
  action?: {
    label: string;
    onClick: () => void;
  };
}

export function EmptyState({ icon: Icon, title, description, action }: EmptyStateProps) {
  return (
    <motion.div
      initial={{ opacity: 0, scale: 0.95 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.95 }}
      className="flex flex-col items-center justify-center p-12 text-center"
    >
      <div className="rounded-full bg-muted p-6 mb-4">
        <Icon className="h-8 w-8 text-muted-foreground" />
      </div>
      <h3 className="font-semibold text-lg mb-2">{title}</h3>
      <p className="text-muted-foreground max-w-md mb-6">{description}</p>
      {action && (
        <Button onClick={action.onClick}>
          {action.label}
        </Button>
      )}
    </motion.div>
  );
}
```

### 5️⃣ Improved Message Animations

**파일**: `src/renderer/components/chat/Message.tsx`

```typescript
import React from 'react';
import { motion } from 'framer-motion';
import { User, Bot } from 'lucide-react';
import { fadeIn, slideUp } from '../ui/animations';
import type { Message as MessageType } from '@/types/chat';

interface MessageProps {
  message: MessageType;
  isStreaming?: boolean;
}

export const Message = React.memo(({ message, isStreaming }: MessageProps) => {
  const isUser = message.role === 'user';

  return (
    <motion.div
      {...slideUp}
      data-testid="message"
      data-role={message.role}
      data-streaming={isStreaming}
      className={`flex gap-3 p-4 ${isUser ? 'bg-muted/50' : ''}`}
    >
      {/* Avatar */}
      <div className={`
        flex-shrink-0 h-8 w-8 rounded-full flex items-center justify-center
        ${isUser ? 'bg-primary text-primary-foreground' : 'bg-secondary text-secondary-foreground'}
      `}>
        {isUser ? <User className="h-4 w-4" /> : <Bot className="h-4 w-4" />}
      </div>

      {/* Content */}
      <div className="flex-1 space-y-2">
        <div className="font-medium text-sm">
          {isUser ? 'You' : 'Assistant'}
        </div>
        <motion.div
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          transition={{ duration: 0.3 }}
          className="message-content prose dark:prose-invert max-w-none"
        >
          {message.content}
        </motion.div>

        {/* Streaming indicator */}
        {isStreaming && (
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            className="flex items-center gap-2 text-xs text-muted-foreground"
            data-testid="streaming-indicator"
          >
            <div className="flex gap-1">
              <motion.div
                className="h-1.5 w-1.5 bg-current rounded-full"
                animate={{ opacity: [0.4, 1, 0.4] }}
                transition={{ duration: 1, repeat: Infinity, delay: 0 }}
              />
              <motion.div
                className="h-1.5 w-1.5 bg-current rounded-full"
                animate={{ opacity: [0.4, 1, 0.4] }}
                transition={{ duration: 1, repeat: Infinity, delay: 0.2 }}
              />
              <motion.div
                className="h-1.5 w-1.5 bg-current rounded-full"
                animate={{ opacity: [0.4, 1, 0.4] }}
                transition={{ duration: 1, repeat: Infinity, delay: 0.4 }}
              />
            </div>
            <span>Generating...</span>
          </motion.div>
        )}
      </div>
    </motion.div>
  );
});

Message.displayName = 'Message';
```

### 6️⃣ Toast Notifications

**파일**: `src/renderer/components/ui/toast-provider.tsx`

```typescript
import React from 'react';
import { Toaster } from 'react-hot-toast';

export function ToastProvider() {
  return (
    <Toaster
      position="bottom-right"
      toastOptions={{
        duration: 4000,
        style: {
          background: 'var(--background)',
          color: 'var(--foreground)',
          border: '1px solid var(--border)',
        },
        success: {
          iconTheme: {
            primary: 'var(--primary)',
            secondary: 'var(--primary-foreground)',
          },
        },
        error: {
          iconTheme: {
            primary: 'var(--destructive)',
            secondary: 'var(--destructive-foreground)',
          },
        },
      }}
    />
  );
}
```

### ✅ 완료 기준

- [ ] 부드러운 애니메이션
- [ ] 로딩 상태 개선
- [ ] 에러 상태 개선
- [ ] 빈 상태 UI
- [ ] Toast 알림

### 📝 Commit Message

```
feat(ui): improve UI/UX with animations and states

- Add smooth animations with Framer Motion
- Implement comprehensive loading states
- Create error state components
- Design empty state screens
- Improve toast notifications

UX improvements:
- Consistent animations
- Clear loading feedback
- User-friendly error messages
- Helpful empty states
```

---

## Commit 115: 접근성 강화

### 📋 작업 내용

1. **키보드 네비게이션**
2. **스크린 리더 지원**
3. **ARIA 속성**
4. **Focus 관리**

### 📁 파일 구조

```
src/renderer/components/a11y/
├── FocusTrap.tsx          # Focus trap
├── SkipLink.tsx           # Skip link
└── VisuallyHidden.tsx     # Screen reader only

src/renderer/hooks/
├── useKeyboardNav.ts      # 키보드 네비게이션
└── useFocusManagement.ts  # Focus 관리
```

### 1️⃣ Keyboard Navigation Hook

**파일**: `src/renderer/hooks/useKeyboardNav.ts`

```typescript
import { useEffect, useCallback } from 'react';

interface KeyboardShortcut {
  key: string;
  ctrl?: boolean;
  shift?: boolean;
  alt?: boolean;
  action: () => void;
}

export function useKeyboardNav(shortcuts: KeyboardShortcut[]) {
  const handleKeyDown = useCallback(
    (event: KeyboardEvent) => {
      for (const shortcut of shortcuts) {
        const ctrlMatch = shortcut.ctrl ? event.ctrlKey || event.metaKey : !event.ctrlKey && !event.metaKey;
        const shiftMatch = shortcut.shift ? event.shiftKey : !event.shiftKey;
        const altMatch = shortcut.alt ? event.altKey : !event.altKey;

        if (
          event.key === shortcut.key &&
          ctrlMatch &&
          shiftMatch &&
          altMatch
        ) {
          event.preventDefault();
          shortcut.action();
          break;
        }
      }
    },
    [shortcuts]
  );

  useEffect(() => {
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [handleKeyDown]);
}

// Usage:
// useKeyboardNav([
//   { key: 'n', ctrl: true, action: () => createNewSession() },
//   { key: 'f', ctrl: true, shift: true, action: () => openSearch() },
// ]);
```

### 2️⃣ Focus Trap Component

**파일**: `src/renderer/components/a11y/FocusTrap.tsx`

```typescript
import React, { useRef, useEffect } from 'react';

interface FocusTrapProps {
  children: React.ReactNode;
  active?: boolean;
}

export function FocusTrap({ children, active = true }: FocusTrapProps) {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!active || !containerRef.current) return;

    const container = containerRef.current;
    const focusableElements = container.querySelectorAll<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement?.focus();
        }
      } else {
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement?.focus();
        }
      }
    };

    container.addEventListener('keydown', handleKeyDown);
    firstElement?.focus();

    return () => {
      container.removeEventListener('keydown', handleKeyDown);
    };
  }, [active]);

  return <div ref={containerRef}>{children}</div>;
}
```

### 3️⃣ Skip Link

**파일**: `src/renderer/components/a11y/SkipLink.tsx`

```typescript
import React from 'react';

export function SkipLink() {
  return (
    <a
      href="#main-content"
      className="
        sr-only focus:not-sr-only
        fixed top-4 left-4 z-50
        bg-primary text-primary-foreground
        px-4 py-2 rounded
        focus:outline-none focus:ring-2 focus:ring-ring
      "
    >
      Skip to main content
    </a>
  );
}
```

### 4️⃣ ARIA-enhanced Components

**파일**: `src/renderer/components/ui/button.tsx` (updated)

```typescript
import React from 'react';
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
  loading?: boolean;
  'aria-label'?: string;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, loading, children, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';

    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        disabled={loading || props.disabled}
        aria-busy={loading}
        {...props}
      >
        {loading ? (
          <>
            <span className="sr-only">Loading...</span>
            <span aria-hidden="true">{children}</span>
          </>
        ) : (
          children
        )}
      </Comp>
    );
  }
);

Button.displayName = 'Button';

export { Button, buttonVariants };
```

### 5️⃣ Accessible Modal

**파일**: `src/renderer/components/ui/dialog.tsx` (updated)

```typescript
import * as React from 'react';
import * as DialogPrimitive from '@radix-ui/react-dialog';
import { X } from 'lucide-react';
import { cn } from '@/lib/utils';

const Dialog = DialogPrimitive.Root;
const DialogTrigger = DialogPrimitive.Trigger;
const DialogPortal = DialogPrimitive.Portal;
const DialogClose = DialogPrimitive.Close;

const DialogOverlay = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Overlay>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Overlay>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Overlay
    ref={ref}
    className={cn(
      'fixed inset-0 z-50 bg-background/80 backdrop-blur-sm data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0',
      className
    )}
    {...props}
  />
));
DialogOverlay.displayName = DialogPrimitive.Overlay.displayName;

const DialogContent = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Content>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Content>
>(({ className, children, ...props }, ref) => (
  <DialogPortal>
    <DialogOverlay />
    <DialogPrimitive.Content
      ref={ref}
      className={cn(
        'fixed left-[50%] top-[50%] z-50 grid w-full max-w-lg translate-x-[-50%] translate-y-[-50%] gap-4 border bg-background p-6 shadow-lg duration-200 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[state=closed]:slide-out-to-left-1/2 data-[state=closed]:slide-out-to-top-[48%] data-[state=open]:slide-in-from-left-1/2 data-[state=open]:slide-in-from-top-[48%] sm:rounded-lg',
        className
      )}
      aria-describedby={undefined}
      {...props}
    >
      {children}
      <DialogPrimitive.Close
        className="absolute right-4 top-4 rounded-sm opacity-70 ring-offset-background transition-opacity hover:opacity-100 focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2 disabled:pointer-events-none data-[state=open]:bg-accent data-[state=open]:text-muted-foreground"
        aria-label="Close dialog"
      >
        <X className="h-4 w-4" />
        <span className="sr-only">Close</span>
      </DialogPrimitive.Close>
    </DialogPrimitive.Content>
  </DialogPortal>
));
DialogContent.displayName = DialogPrimitive.Content.displayName;

const DialogTitle = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Title>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Title>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Title
    ref={ref}
    className={cn('text-lg font-semibold leading-none tracking-tight', className)}
    {...props}
  />
));
DialogTitle.displayName = DialogPrimitive.Title.displayName;

export {
  Dialog,
  DialogPortal,
  DialogOverlay,
  DialogClose,
  DialogTrigger,
  DialogContent,
  DialogTitle,
};
```

### ✅ 완료 기준

- [ ] 키보드 네비게이션
- [ ] Focus trap
- [ ] ARIA 속성
- [ ] 스크린 리더 지원
- [ ] Skip links

### 📝 Commit Message

```
feat(a11y): enhance accessibility

- Implement keyboard navigation shortcuts
- Add focus trap for modals
- Create skip links for screen readers
- Add comprehensive ARIA attributes
- Improve focus management

Accessibility features:
- WCAG 2.1 AA compliance
- Full keyboard support
- Screen reader optimized
- Focus indicators
```

---

## Commit 116: 사용자 가이드 및 문서화

### 📋 작업 내용

1. **사용자 가이드**
2. **개발자 문서**
3. **API 문서**
4. **FAQ**

### 📁 파일 구조

```
docs/
├── user-guide/
│   ├── getting-started.md
│   ├── features.md
│   ├── shortcuts.md
│   └── troubleshooting.md
├── developer/
│   ├── architecture.md
│   ├── contributing.md
│   └── api-reference.md
└── FAQ.md
```

### 1️⃣ User Guide - Getting Started

**파일**: `docs/user-guide/getting-started.md`

```markdown
# Getting Started with Codex UI

Welcome to Codex UI! This guide will help you get started.

## Installation

### Windows
1. Download `Codex-UI-Setup-x.x.x.exe`
2. Run the installer
3. Follow the installation wizard

### macOS
1. Download `Codex-UI-x.x.x.dmg`
2. Open the DMG file
3. Drag Codex UI to Applications folder

### Linux
```bash
# Debian/Ubuntu
sudo dpkg -i codex-ui_x.x.x_amd64.deb

# AppImage
chmod +x Codex-UI-x.x.x.AppImage
./Codex-UI-x.x.x.AppImage
```

## First Launch

When you first launch Codex UI, you'll need to:

1. **Enter your API key**
   - Click Settings (⚙️)
   - Navigate to "API Keys"
   - Enter your OpenAI or Anthropic API key

2. **Choose your theme**
   - Light, Dark, or System

3. **Set up preferences**
   - Default model
   - Language
   - Keyboard shortcuts

## Creating Your First Session

1. Click "New Session" (Cmd/Ctrl + N)
2. Type your message
3. Press Enter or click Send
4. Wait for the AI response

## Key Features

### Chat with AI
- Support for multiple AI models (GPT-4, Claude, etc.)
- Streaming responses
- Message history
- Code syntax highlighting

### File Operations
- Upload files
- Edit code with Monaco Editor
- Syntax highlighting
- Auto-save

### Search
- Full-text search across all messages
- Semantic search with embeddings
- Advanced filters

### Plugins
- Extend functionality with plugins
- Custom tools and commands

## Keyboard Shortcuts

| Action | Windows/Linux | macOS |
|--------|--------------|-------|
| New Session | Ctrl + N | Cmd + N |
| Search | Ctrl + Shift + F | Cmd + Shift + F |
| Settings | Ctrl + , | Cmd + , |
| Toggle Sidebar | Ctrl + B | Cmd + B |

## Next Steps

- Explore [Features](./features.md)
- Learn [Keyboard Shortcuts](./shortcuts.md)
- Check out [Troubleshooting](./troubleshooting.md)
```

### 2️⃣ Developer Documentation

**파일**: `docs/developer/architecture.md`

```markdown
# Architecture

Codex UI is built with Electron, React, and TypeScript.

## Overview

```
┌─────────────────────────────────────┐
│         Electron Main Process       │
│  - Window management                │
│  - IPC handlers                     │
│  - Native integrations              │
│  - Database (SQLite)                │
└─────────────────────────────────────┘
              │
              │ IPC
              │
┌─────────────────────────────────────┐
│      Electron Renderer Process      │
│  - React UI                         │
│  - Zustand state management         │
│  - Monaco Editor                    │
│  - WebSocket client                 │
└─────────────────────────────────────┘
```

## Tech Stack

### Main Process
- **Electron**: Desktop app framework
- **better-sqlite3**: Database
- **WebSocket**: Real-time communication

### Renderer Process
- **React 18**: UI framework
- **TypeScript**: Type safety
- **Zustand**: State management
- **Vite**: Build tool
- **Tailwind CSS**: Styling
- **Monaco Editor**: Code editing

## Project Structure

```
codex-ui/
├── src/
│   ├── main/              # Main process
│   │   ├── handlers/      # IPC handlers
│   │   ├── services/      # Business logic
│   │   └── index.ts       # Entry point
│   ├── preload/           # Preload scripts
│   │   └── index.ts       # IPC bridge
│   └── renderer/          # Renderer process
│       ├── components/    # React components
│       ├── hooks/         # Custom hooks
│       ├── store/         # Zustand stores
│       └── main.tsx       # Entry point
├── electron.vite.config.ts
└── package.json
```

## Key Concepts

### IPC Communication

All communication between main and renderer processes uses typed IPC:

```typescript
// Main process
ipcMain.handle('load-session', async (event, sessionId: string) => {
  return await sessionService.loadSession(sessionId);
});

// Renderer process
const session = await window.electronAPI.loadSession(sessionId);
```

### State Management

Zustand with Immer for immutable updates:

```typescript
const useStore = create<State>()(
  immer((set) => ({
    sessions: [],
    addSession: (session) => set((state) => {
      state.sessions.push(session);
    }),
  }))
);
```

### Database Schema

SQLite with better-sqlite3:

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  role TEXT NOT NULL,
  content TEXT NOT NULL,
  timestamp INTEGER NOT NULL,
  FOREIGN KEY (session_id) REFERENCES sessions(id)
);
```

## Building

```bash
# Development
npm run dev

# Build
npm run build

# Package
npm run build:win   # Windows
npm run build:mac   # macOS
npm run build:linux # Linux
```

## Testing

```bash
# Unit tests
npm run test

# E2E tests
npm run test:e2e
```
```

### ✅ 완료 기준

- [ ] 사용자 가이드 작성
- [ ] 개발자 문서 작성
- [ ] API 레퍼런스
- [ ] FAQ 작성

### 📝 Commit Message

```
docs: add comprehensive documentation

- Create user guide (getting started, features, shortcuts)
- Add developer documentation (architecture, contributing)
- Write API reference
- Create troubleshooting guide and FAQ

Documentation:
- Installation instructions
- Feature overview
- Architecture diagrams
- Code examples
```

---

## 🎯 Day 20 완료 체크리스트

### 테스트
- [ ] E2E 테스트 완성
- [ ] 성능 테스트
- [ ] 통합 테스트

### UI/UX
- [ ] 애니메이션 개선
- [ ] 로딩/에러 상태
- [ ] 접근성 강화

### 문서화
- [ ] 사용자 가이드
- [ ] 개발자 문서
- [ ] API 레퍼런스

---

## 📦 Dependencies

```json
{
  "devDependencies": {
    "@playwright/test": "^1.40.0",
    "@testing-library/react": "^14.1.2",
    "@testing-library/jest-dom": "^6.1.5"
  }
}
```

---

**다음**: Day 21에서는 프로덕션 배포와 최종 점검을 진행합니다.
