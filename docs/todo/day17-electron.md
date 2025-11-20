# Day 17 TODO - AI 향상 기능 (Electron)

> **목표**: AI 기능 고도화, 스트리밍 최적화, 컨텍스트 관리

## 전체 개요

Day 17은 Codex UI의 AI 기능을 강화합니다:
- 스마트 컨텍스트 선택
- 프롬프트 템플릿 시스템
- AI 응답 캐싱
- 스트리밍 최적화
- 토큰 사용량 추적
- Model comparison

**Electron 특화:**
- Native AI model switching
- Offline AI support (local models)
- GPU acceleration
- System resource monitoring
- Power-efficient AI processing

---

## Commit 97: 스마트 컨텍스트 선택

### 📋 작업 내용

1. **컨텍스트 랭킹**
2. **자동 요약**
3. **토큰 예측**
4. **컨텍스트 압축**

### 📁 파일 구조

```
src/main/ai/
├── ContextManager.ts      # 컨텍스트 관리
├── ContextRanker.ts       # 랭킹 알고리즘
├── ContextSummarizer.ts   # 요약
└── types.ts               # AI 타입

src/renderer/components/ai/
├── ContextSelector.tsx    # 컨텍스트 선택
└── TokenUsage.tsx         # 토큰 사용량
```

### 1️⃣ AI 타입 정의

**파일**: `src/renderer/types/ai.ts`

```typescript
export interface ContextItem {
  id: string;
  type: 'message' | 'file' | 'snippet';
  content: string;
  tokens: number;
  relevance: number;
  timestamp: number;
  metadata: {
    role?: 'user' | 'assistant' | 'system';
    fileType?: string;
    tags?: string[];
  };
}

export interface ContextSelection {
  items: ContextItem[];
  totalTokens: number;
  maxTokens: number;
  truncated: boolean;
}

export interface PromptTemplate {
  id: string;
  name: string;
  description: string;
  template: string;
  variables: string[];
  category: string;
}

export interface TokenUsage {
  prompt: number;
  completion: number;
  total: number;
  cost: number;
}

export interface AIModelConfig {
  id: string;
  name: string;
  provider: 'openai' | 'anthropic' | 'local';
  maxTokens: number;
  contextWindow: number;
  costPer1kTokens: {
    prompt: number;
    completion: number;
  };
}
```

### 2️⃣ Context Manager

**파일**: `src/main/ai/ContextManager.ts`

```typescript
import { ContextRanker } from './ContextRanker';
import { ContextSummarizer } from './ContextSummarizer';
import type { ContextItem, ContextSelection } from '@/renderer/types/ai';
import { encoding_for_model, type TiktokenModel } from 'tiktoken';

export class ContextManager {
  private ranker = new ContextRanker();
  private summarizer = new ContextSummarizer();
  private encoder = encoding_for_model('gpt-4' as TiktokenModel);

  async selectContext(
    query: string,
    availableItems: ContextItem[],
    maxTokens: number
  ): Promise<ContextSelection> {
    // Rank items by relevance
    const rankedItems = await this.ranker.rank(query, availableItems);

    // Select items that fit within token budget
    const selectedItems: ContextItem[] = [];
    let totalTokens = 0;
    let truncated = false;

    for (const item of rankedItems) {
      if (totalTokens + item.tokens <= maxTokens) {
        selectedItems.push(item);
        totalTokens += item.tokens;
      } else {
        truncated = true;
        break;
      }
    }

    // If we have room, try to summarize additional context
    if (truncated && totalTokens < maxTokens * 0.8) {
      const remainingItems = rankedItems.slice(selectedItems.length);
      const summary = await this.summarizer.summarize(remainingItems, maxTokens - totalTokens);

      if (summary) {
        selectedItems.push({
          id: 'summary',
          type: 'snippet',
          content: summary.content,
          tokens: summary.tokens,
          relevance: 0.5,
          timestamp: Date.now(),
          metadata: { tags: ['summary'] },
        });
        totalTokens += summary.tokens;
      }
    }

    return {
      items: selectedItems,
      totalTokens,
      maxTokens,
      truncated,
    };
  }

  countTokens(text: string): number {
    try {
      const tokens = this.encoder.encode(text);
      return tokens.length;
    } catch (error) {
      // Fallback: rough estimation
      return Math.ceil(text.length / 4);
    }
  }

  async compressContext(items: ContextItem[], targetTokens: number): Promise<ContextItem[]> {
    // Sort by relevance
    const sorted = [...items].sort((a, b) => b.relevance - a.relevance);

    // Keep high-relevance items, summarize low-relevance ones
    const highRelevance = sorted.filter(item => item.relevance > 0.7);
    const lowRelevance = sorted.filter(item => item.relevance <= 0.7);

    const compressed: ContextItem[] = [...highRelevance];
    let totalTokens = highRelevance.reduce((sum, item) => sum + item.tokens, 0);

    // Summarize low-relevance items if we need more context
    if (totalTokens < targetTokens && lowRelevance.length > 0) {
      const remainingTokens = targetTokens - totalTokens;
      const summary = await this.summarizer.summarize(lowRelevance, remainingTokens);

      if (summary) {
        compressed.push({
          id: 'compressed-summary',
          type: 'snippet',
          content: summary.content,
          tokens: summary.tokens,
          relevance: 0.5,
          timestamp: Date.now(),
          metadata: { tags: ['compressed'] },
        });
      }
    }

    return compressed;
  }

  dispose(): void {
    this.encoder.free();
  }
}

export const contextManager = new ContextManager();
```

### 3️⃣ Context Ranker

**파일**: `src/main/ai/ContextRanker.ts`

```typescript
import type { ContextItem } from '@/renderer/types/ai';
import { cosineSimilarity, generateEmbedding } from './embeddings';

export class ContextRanker {
  async rank(query: string, items: ContextItem[]): Promise<ContextItem[]> {
    // Generate query embedding
    const queryEmbedding = await generateEmbedding(query);

    // Calculate relevance scores
    const scored = await Promise.all(
      items.map(async (item) => {
        const itemEmbedding = await generateEmbedding(item.content);
        const similarity = cosineSimilarity(queryEmbedding, itemEmbedding);

        // Combine similarity with recency and type
        const recencyScore = this.calculateRecency(item.timestamp);
        const typeScore = this.getTypeScore(item.type);

        const finalScore = (
          similarity * 0.7 +
          recencyScore * 0.2 +
          typeScore * 0.1
        );

        return {
          ...item,
          relevance: finalScore,
        };
      })
    );

    // Sort by relevance (descending)
    return scored.sort((a, b) => b.relevance - a.relevance);
  }

  private calculateRecency(timestamp: number): number {
    const now = Date.now();
    const ageInHours = (now - timestamp) / (1000 * 60 * 60);

    // Exponential decay: more recent = higher score
    return Math.exp(-ageInHours / 24); // Half-life of 24 hours
  }

  private getTypeScore(type: ContextItem['type']): number {
    switch (type) {
      case 'message':
        return 1.0;
      case 'file':
        return 0.8;
      case 'snippet':
        return 0.6;
      default:
        return 0.5;
    }
  }
}

// Helper functions
async function generateEmbedding(text: string): Promise<number[]> {
  // Use OpenAI embeddings or local model
  // This is a placeholder - implement with actual embedding service
  return [];
}

function cosineSimilarity(a: number[], b: number[]): number {
  let dotProduct = 0;
  let normA = 0;
  let normB = 0;

  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }

  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

### 4️⃣ Context Summarizer

**파일**: `src/main/ai/ContextSummarizer.ts`

```typescript
import { OpenAI } from 'openai';
import type { ContextItem } from '@/renderer/types/ai';
import { contextManager } from './ContextManager';

export class ContextSummarizer {
  private client: OpenAI | null = null;

  initialize(apiKey: string): void {
    this.client = new OpenAI({ apiKey });
  }

  async summarize(
    items: ContextItem[],
    maxTokens: number
  ): Promise<{ content: string; tokens: number } | null> {
    if (!this.client || items.length === 0) return null;

    // Concatenate items
    const combined = items
      .map(item => item.content)
      .join('\n\n---\n\n');

    // Request summary
    try {
      const response = await this.client.chat.completions.create({
        model: 'gpt-3.5-turbo',
        messages: [
          {
            role: 'system',
            content: `Summarize the following context concisely in under ${maxTokens} tokens. Focus on key information relevant to the user's query.`,
          },
          {
            role: 'user',
            content: combined,
          },
        ],
        max_tokens: maxTokens,
        temperature: 0.3,
      });

      const summary = response.choices[0].message.content || '';
      const tokens = contextManager.countTokens(summary);

      return { content: summary, tokens };
    } catch (error) {
      console.error('Failed to generate summary:', error);
      return null;
    }
  }
}
```

### 5️⃣ Context Selector UI

**파일**: `src/renderer/components/ai/ContextSelector.tsx`

```typescript
import React, { useState, useEffect } from 'react';
import { Brain, FileText, MessageSquare, Sparkles } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Progress } from '@/components/ui/progress';
import type { ContextItem, ContextSelection } from '@/types/ai';

interface ContextSelectorProps {
  query: string;
  maxTokens: number;
  onSelectionChange: (selection: ContextSelection) => void;
}

export function ContextSelector({ query, maxTokens, onSelectionChange }: ContextSelectorProps) {
  const [availableItems, setAvailableItems] = useState<ContextItem[]>([]);
  const [selection, setSelection] = useState<ContextSelection | null>(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    loadAvailableContext();
  }, []);

  const loadAvailableContext = async () => {
    if (!window.electronAPI) return;

    // Load conversation history and files
    const items = await window.electronAPI.getAvailableContext();
    setAvailableItems(items);
  };

  const handleAutoSelect = async () => {
    if (!window.electronAPI) return;

    setLoading(true);

    try {
      const selected = await window.electronAPI.selectContext(query, maxTokens);
      setSelection(selected);
      onSelectionChange(selected);
    } catch (error) {
      console.error('Failed to select context:', error);
    } finally {
      setLoading(false);
    }
  };

  const tokenUsagePercent = selection
    ? (selection.totalTokens / selection.maxTokens) * 100
    : 0;

  return (
    <div className="space-y-4">
      {/* Header */}
      <div className="flex items-center justify-between">
        <h3 className="font-semibold flex items-center gap-2">
          <Brain className="h-4 w-4" />
          Smart Context Selection
        </h3>
        <Button
          size="sm"
          onClick={handleAutoSelect}
          disabled={loading || !query}
        >
          <Sparkles className="h-4 w-4 mr-2" />
          Auto Select
        </Button>
      </div>

      {/* Token Usage */}
      {selection && (
        <div className="space-y-2">
          <div className="flex items-center justify-between text-sm">
            <span>Token Usage</span>
            <span className="font-mono">
              {selection.totalTokens.toLocaleString()} / {selection.maxTokens.toLocaleString()}
            </span>
          </div>
          <Progress value={tokenUsagePercent} />
          {selection.truncated && (
            <p className="text-xs text-amber-600 dark:text-amber-400">
              Some context was truncated to fit within token limit
            </p>
          )}
        </div>
      )}

      {/* Selected Items */}
      {selection && selection.items.length > 0 && (
        <div className="space-y-2">
          <div className="text-sm font-medium">Selected Context ({selection.items.length} items)</div>
          <div className="space-y-1 max-h-60 overflow-y-auto">
            {selection.items.map(item => (
              <div
                key={item.id}
                className="p-2 border rounded text-sm flex items-start gap-2"
              >
                {item.type === 'message' && <MessageSquare className="h-4 w-4 mt-0.5 flex-shrink-0" />}
                {item.type === 'file' && <FileText className="h-4 w-4 mt-0.5 flex-shrink-0" />}
                {item.type === 'snippet' && <Sparkles className="h-4 w-4 mt-0.5 flex-shrink-0" />}

                <div className="flex-1 min-w-0">
                  <div className="truncate">{item.content.slice(0, 100)}...</div>
                  <div className="flex items-center gap-2 mt-1">
                    <Badge variant="secondary" className="text-xs">
                      {item.tokens} tokens
                    </Badge>
                    <Badge variant="outline" className="text-xs">
                      {(item.relevance * 100).toFixed(0)}% relevant
                    </Badge>
                  </div>
                </div>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 컨텍스트 랭킹 알고리즘
- [ ] 자동 요약 기능
- [ ] 토큰 카운팅
- [ ] 컨텍스트 압축
- [ ] UI 완성

### 📝 Commit Message

```
feat(ai): implement smart context selection

- Add ContextManager with ranking
- Implement ContextRanker with embeddings
- Create ContextSummarizer for compression
- Add ContextSelector UI with token tracking

Features:
- Relevance-based ranking
- Automatic summarization
- Token budget management
- Recency scoring
```

---

## Commit 98: 프롬프트 템플릿 시스템

### 📋 작업 내용

1. **템플릿 저장소**
2. **변수 치환**
3. **템플릿 라이브러리**
4. **커스텀 템플릿**

### 📁 파일 구조

```
src/main/ai/
├── TemplateManager.ts     # 템플릿 관리
└── templates/             # 기본 템플릿

src/renderer/components/ai/
├── TemplateLibrary.tsx    # 템플릿 목록
├── TemplateEditor.tsx     # 템플릿 편집
└── TemplateSelector.tsx   # 템플릿 선택
```

### 1️⃣ Template Manager

**파일**: `src/main/ai/TemplateManager.ts`

```typescript
import { app } from 'electron';
import fs from 'fs/promises';
import path from 'path';
import type { PromptTemplate } from '@/renderer/types/ai';

export class TemplateManager {
  private templatesPath: string;
  private templates: Map<string, PromptTemplate> = new Map();

  constructor() {
    this.templatesPath = path.join(app.getPath('userData'), 'templates.json');
    this.loadTemplates();
  }

  private async loadTemplates(): Promise<void> {
    try {
      const data = await fs.readFile(this.templatesPath, 'utf-8');
      const templates = JSON.parse(data) as PromptTemplate[];

      for (const template of templates) {
        this.templates.set(template.id, template);
      }
    } catch (error) {
      // File doesn't exist, load defaults
      await this.loadDefaultTemplates();
    }
  }

  private async loadDefaultTemplates(): Promise<void> {
    const defaults: PromptTemplate[] = [
      {
        id: 'code-review',
        name: 'Code Review',
        description: 'Review code for best practices and potential issues',
        template: 'Please review the following {{language}} code and provide feedback on:\n1. Code quality\n2. Potential bugs\n3. Performance improvements\n4. Best practices\n\n```{{language}}\n{{code}}\n```',
        variables: ['language', 'code'],
        category: 'development',
      },
      {
        id: 'explain-code',
        name: 'Explain Code',
        description: 'Explain what a piece of code does',
        template: 'Please explain the following {{language}} code in detail:\n\n```{{language}}\n{{code}}\n```\n\nExplain:\n1. What it does\n2. How it works\n3. Key concepts used',
        variables: ['language', 'code'],
        category: 'development',
      },
      {
        id: 'debug-help',
        name: 'Debug Help',
        description: 'Get help debugging an error',
        template: 'I\'m encountering the following error in my {{language}} code:\n\nError:\n```\n{{error}}\n```\n\nCode:\n```{{language}}\n{{code}}\n```\n\nPlease help me:\n1. Understand the error\n2. Find the root cause\n3. Suggest a fix',
        variables: ['language', 'error', 'code'],
        category: 'development',
      },
      {
        id: 'write-tests',
        name: 'Write Tests',
        description: 'Generate unit tests for code',
        template: 'Please write comprehensive unit tests for the following {{language}} code using {{testFramework}}:\n\n```{{language}}\n{{code}}\n```\n\nInclude:\n1. Happy path tests\n2. Edge cases\n3. Error handling',
        variables: ['language', 'testFramework', 'code'],
        category: 'development',
      },
      {
        id: 'optimize-code',
        name: 'Optimize Code',
        description: 'Optimize code for performance',
        template: 'Please optimize the following {{language}} code for better performance:\n\n```{{language}}\n{{code}}\n```\n\nFocus on:\n1. Time complexity\n2. Space complexity\n3. Readability\n\nProvide the optimized version with explanations.',
        variables: ['language', 'code'],
        category: 'development',
      },
      {
        id: 'summarize',
        name: 'Summarize',
        description: 'Summarize text or conversation',
        template: 'Please provide a concise summary of the following:\n\n{{content}}\n\nSummary should include:\n1. Key points\n2. Main conclusions\n3. Action items (if any)',
        variables: ['content'],
        category: 'general',
      },
      {
        id: 'translate',
        name: 'Translate',
        description: 'Translate text to another language',
        template: 'Please translate the following text from {{sourceLang}} to {{targetLang}}:\n\n{{text}}',
        variables: ['sourceLang', 'targetLang', 'text'],
        category: 'general',
      },
    ];

    for (const template of defaults) {
      this.templates.set(template.id, template);
    }

    await this.saveTemplates();
  }

  async saveTemplates(): Promise<void> {
    const templates = Array.from(this.templates.values());
    await fs.writeFile(this.templatesPath, JSON.stringify(templates, null, 2));
  }

  getTemplate(id: string): PromptTemplate | undefined {
    return this.templates.get(id);
  }

  getAllTemplates(): PromptTemplate[] {
    return Array.from(this.templates.values());
  }

  getTemplatesByCategory(category: string): PromptTemplate[] {
    return Array.from(this.templates.values()).filter(
      t => t.category === category
    );
  }

  async addTemplate(template: PromptTemplate): Promise<void> {
    this.templates.set(template.id, template);
    await this.saveTemplates();
  }

  async updateTemplate(id: string, updates: Partial<PromptTemplate>): Promise<void> {
    const template = this.templates.get(id);
    if (!template) throw new Error(`Template not found: ${id}`);

    this.templates.set(id, { ...template, ...updates });
    await this.saveTemplates();
  }

  async deleteTemplate(id: string): Promise<void> {
    this.templates.delete(id);
    await this.saveTemplates();
  }

  renderTemplate(templateId: string, variables: Record<string, string>): string {
    const template = this.getTemplate(templateId);
    if (!template) throw new Error(`Template not found: ${templateId}`);

    let result = template.template;

    // Replace variables
    for (const [key, value] of Object.entries(variables)) {
      result = result.replace(new RegExp(`{{${key}}}`, 'g'), value);
    }

    return result;
  }

  extractVariables(template: string): string[] {
    const regex = /{{(\w+)}}/g;
    const variables: string[] = [];
    let match;

    while ((match = regex.exec(template)) !== null) {
      if (!variables.includes(match[1])) {
        variables.push(match[1]);
      }
    }

    return variables;
  }
}

export const templateManager = new TemplateManager();
```

### 2️⃣ Template Library UI

**파일**: `src/renderer/components/ai/TemplateLibrary.tsx`

```typescript
import React, { useState, useEffect } from 'react';
import { FileText, Plus, Edit, Trash, Copy } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Badge } from '@/components/ui/badge';
import { TemplateEditor } from './TemplateEditor';
import type { PromptTemplate } from '@/types/ai';

export function TemplateLibrary() {
  const [templates, setTemplates] = useState<PromptTemplate[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedCategory, setSelectedCategory] = useState<string | null>(null);
  const [editingTemplate, setEditingTemplate] = useState<PromptTemplate | null>(null);
  const [showEditor, setShowEditor] = useState(false);

  useEffect(() => {
    loadTemplates();
  }, []);

  const loadTemplates = async () => {
    if (!window.electronAPI) return;

    const templates = await window.electronAPI.getAllTemplates();
    setTemplates(templates);
  };

  const handleUseTemplate = async (template: PromptTemplate) => {
    // Open template with variable inputs
    setEditingTemplate(template);
  };

  const handleDelete = async (id: string) => {
    if (!window.electronAPI) return;
    if (!confirm('Delete this template?')) return;

    await window.electronAPI.deleteTemplate(id);
    await loadTemplates();
  };

  const categories = Array.from(new Set(templates.map(t => t.category)));

  const filteredTemplates = templates.filter(t => {
    const matchesSearch = t.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
      t.description.toLowerCase().includes(searchQuery.toLowerCase());
    const matchesCategory = !selectedCategory || t.category === selectedCategory;
    return matchesSearch && matchesCategory;
  });

  return (
    <div className="space-y-4">
      {/* Header */}
      <div className="flex items-center justify-between">
        <h3 className="font-semibold flex items-center gap-2">
          <FileText className="h-4 w-4" />
          Prompt Templates
        </h3>
        <Button size="sm" onClick={() => setShowEditor(true)}>
          <Plus className="h-4 w-4 mr-2" />
          New Template
        </Button>
      </div>

      {/* Search and Filters */}
      <div className="flex gap-2">
        <Input
          placeholder="Search templates..."
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          className="flex-1"
        />
      </div>

      {/* Categories */}
      <div className="flex flex-wrap gap-2">
        <Badge
          variant={selectedCategory === null ? 'default' : 'outline'}
          className="cursor-pointer"
          onClick={() => setSelectedCategory(null)}
        >
          All
        </Badge>
        {categories.map(category => (
          <Badge
            key={category}
            variant={selectedCategory === category ? 'default' : 'outline'}
            className="cursor-pointer"
            onClick={() => setSelectedCategory(category)}
          >
            {category}
          </Badge>
        ))}
      </div>

      {/* Templates */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
        {filteredTemplates.map(template => (
          <div
            key={template.id}
            className="p-4 border rounded-lg space-y-2 hover:bg-accent transition-colors"
          >
            <div className="flex items-start justify-between">
              <div>
                <h4 className="font-medium">{template.name}</h4>
                <p className="text-sm text-muted-foreground">{template.description}</p>
              </div>
              <Badge variant="secondary">{template.category}</Badge>
            </div>

            <div className="flex flex-wrap gap-1">
              {template.variables.map(variable => (
                <Badge key={variable} variant="outline" className="text-xs">
                  {variable}
                </Badge>
              ))}
            </div>

            <div className="flex gap-2 pt-2">
              <Button
                size="sm"
                variant="default"
                onClick={() => handleUseTemplate(template)}
                className="flex-1"
              >
                <Copy className="h-3 w-3 mr-1" />
                Use
              </Button>
              <Button
                size="sm"
                variant="outline"
                onClick={() => {
                  setEditingTemplate(template);
                  setShowEditor(true);
                }}
              >
                <Edit className="h-3 w-3" />
              </Button>
              <Button
                size="sm"
                variant="outline"
                onClick={() => handleDelete(template.id)}
              >
                <Trash className="h-3 w-3" />
              </Button>
            </div>
          </div>
        ))}
      </div>

      {/* Template Editor Modal */}
      {showEditor && (
        <TemplateEditor
          template={editingTemplate}
          onSave={async (template) => {
            await loadTemplates();
            setShowEditor(false);
            setEditingTemplate(null);
          }}
          onClose={() => {
            setShowEditor(false);
            setEditingTemplate(null);
          }}
        />
      )}
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 템플릿 저장/로드
- [ ] 변수 치환
- [ ] 템플릿 라이브러리 UI
- [ ] 커스텀 템플릿 생성

### 📝 Commit Message

```
feat(ai): add prompt template system

- Implement TemplateManager with storage
- Create default template library
- Add variable substitution
- Build TemplateLibrary UI

Templates:
- Code review
- Explain code
- Debug help
- Write tests
- Optimize code
- Summarize
- Translate
```

---

## Commit 99: AI 응답 캐싱

### 📋 작업 내용

1. **응답 캐시 저장소**
2. **캐시 키 생성**
3. **캐시 무효화**
4. **캐시 통계**

### 📁 파일 구조

```
src/main/ai/
├── ResponseCache.ts       # 응답 캐싱
└── CacheStore.ts          # 캐시 저장소

src/renderer/components/ai/
└── CacheStats.tsx         # 캐시 통계
```

### 1️⃣ Response Cache

**파일**: `src/main/ai/ResponseCache.ts`

```typescript
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';
import crypto from 'crypto';

interface CachedResponse {
  key: string;
  prompt: string;
  response: string;
  model: string;
  tokens: number;
  timestamp: number;
  hitCount: number;
}

export class ResponseCache {
  private db: Database.Database;
  private ttl = 7 * 24 * 60 * 60 * 1000; // 7 days

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'ai-cache.db');
    this.db = new Database(dbPath);
    this.initialize();
  }

  private initialize(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS response_cache (
        key TEXT PRIMARY KEY,
        prompt TEXT NOT NULL,
        response TEXT NOT NULL,
        model TEXT NOT NULL,
        tokens INTEGER NOT NULL,
        timestamp INTEGER NOT NULL,
        hit_count INTEGER DEFAULT 0
      );

      CREATE INDEX IF NOT EXISTS idx_timestamp ON response_cache(timestamp);
      CREATE INDEX IF NOT EXISTS idx_model ON response_cache(model);
    `);

    // Clean up old entries
    this.cleanup();
  }

  generateKey(prompt: string, model: string, context?: string): string {
    const content = `${prompt}|${model}|${context || ''}`;
    return crypto.createHash('sha256').update(content).digest('hex');
  }

  get(key: string): CachedResponse | null {
    const row = this.db.prepare(`
      SELECT * FROM response_cache WHERE key = ?
    `).get(key) as any;

    if (!row) return null;

    // Check if expired
    if (Date.now() - row.timestamp > this.ttl) {
      this.delete(key);
      return null;
    }

    // Increment hit count
    this.db.prepare(`
      UPDATE response_cache SET hit_count = hit_count + 1 WHERE key = ?
    `).run(key);

    return {
      key: row.key,
      prompt: row.prompt,
      response: row.response,
      model: row.model,
      tokens: row.tokens,
      timestamp: row.timestamp,
      hitCount: row.hit_count + 1,
    };
  }

  set(key: string, prompt: string, response: string, model: string, tokens: number): void {
    this.db.prepare(`
      INSERT OR REPLACE INTO response_cache (key, prompt, response, model, tokens, timestamp, hit_count)
      VALUES (?, ?, ?, ?, ?, ?, 0)
    `).run(key, prompt, response, model, tokens, Date.now());
  }

  delete(key: string): void {
    this.db.prepare('DELETE FROM response_cache WHERE key = ?').run(key);
  }

  clear(): void {
    this.db.prepare('DELETE FROM response_cache').run();
  }

  cleanup(): void {
    const cutoff = Date.now() - this.ttl;
    this.db.prepare('DELETE FROM response_cache WHERE timestamp < ?').run(cutoff);
  }

  getStats(): {
    totalEntries: number;
    totalTokensSaved: number;
    avgHitCount: number;
    cacheSize: number;
  } {
    const stats = this.db.prepare(`
      SELECT
        COUNT(*) as total_entries,
        SUM(tokens * hit_count) as total_tokens_saved,
        AVG(hit_count) as avg_hit_count
      FROM response_cache
    `).get() as any;

    const dbPath = path.join(app.getPath('userData'), 'ai-cache.db');
    const dbSize = require('fs').statSync(dbPath).size;

    return {
      totalEntries: stats.total_entries || 0,
      totalTokensSaved: stats.total_tokens_saved || 0,
      avgHitCount: stats.avg_hit_count || 0,
      cacheSize: dbSize,
    };
  }

  getMostUsed(limit = 10): CachedResponse[] {
    return this.db.prepare(`
      SELECT * FROM response_cache
      ORDER BY hit_count DESC
      LIMIT ?
    `).all(limit) as CachedResponse[];
  }

  close(): void {
    this.db.close();
  }
}

export const responseCache = new ResponseCache();
```

### 2️⃣ Cache-Aware AI Client

**파일**: `src/main/ai/CachedAIClient.ts`

```typescript
import { OpenAI } from 'openai';
import { responseCache } from './ResponseCache';
import { contextManager } from './ContextManager';

export class CachedAIClient {
  private client: OpenAI;

  constructor(apiKey: string) {
    this.client = new OpenAI({ apiKey });
  }

  async chat(
    messages: Array<{ role: string; content: string }>,
    model: string,
    useCache = true
  ): Promise<{ response: string; cached: boolean; tokens: number }> {
    // Generate cache key
    const prompt = messages[messages.length - 1].content;
    const context = messages.slice(0, -1).map(m => m.content).join('\n');
    const cacheKey = responseCache.generateKey(prompt, model, context);

    // Check cache
    if (useCache) {
      const cached = responseCache.get(cacheKey);
      if (cached) {
        return {
          response: cached.response,
          cached: true,
          tokens: 0, // No tokens used
        };
      }
    }

    // Make API call
    const response = await this.client.chat.completions.create({
      model,
      messages: messages as any,
    });

    const content = response.choices[0].message.content || '';
    const tokens = response.usage?.total_tokens || 0;

    // Cache response
    if (useCache) {
      responseCache.set(cacheKey, prompt, content, model, tokens);
    }

    return {
      response: content,
      cached: false,
      tokens,
    };
  }
}
```

### ✅ 완료 기준

- [ ] 응답 캐싱 작동
- [ ] 캐시 키 생성
- [ ] TTL 관리
- [ ] 캐시 통계

### 📝 Commit Message

```
feat(ai): implement response caching

- Add ResponseCache with SQLite
- Create cache key generation
- Implement TTL and cleanup
- Add CachedAIClient wrapper

Features:
- SHA-256 cache keys
- 7-day TTL
- Hit count tracking
- Token savings statistics
```

---

## Commit 100: 스트리밍 최적화

### 📋 작업 내용

1. **청크 버퍼링**
2. **백프레셔 처리**
3. **스트림 재개**
4. **에러 복구**

### 📁 파일 구조

```
src/main/ai/
├── StreamOptimizer.ts     # 스트리밍 최적화
└── StreamBuffer.ts        # 버퍼 관리

src/renderer/components/ai/
└── StreamingIndicator.tsx # 스트리밍 상태
```

### 1️⃣ Stream Optimizer

**파일**: `src/main/ai/StreamOptimizer.ts`

```typescript
import { EventEmitter } from 'events';

interface StreamChunk {
  id: string;
  content: string;
  timestamp: number;
}

export class StreamOptimizer extends EventEmitter {
  private buffer: StreamChunk[] = [];
  private bufferSize = 10;
  private flushInterval = 50; // ms
  private flushTimer: NodeJS.Timeout | null = null;
  private paused = false;

  async processStream(
    stream: AsyncIterable<any>,
    onChunk: (content: string) => void,
    onComplete: () => void,
    onError: (error: Error) => void
  ): Promise<void> {
    try {
      for await (const chunk of stream) {
        if (this.paused) {
          await this.waitForResume();
        }

        const content = chunk.choices[0]?.delta?.content || '';
        if (content) {
          this.addToBuffer(content);

          // Flush if buffer is full
          if (this.buffer.length >= this.bufferSize) {
            this.flush(onChunk);
          }
        }
      }

      // Flush remaining
      this.flush(onChunk);
      onComplete();
    } catch (error) {
      onError(error as Error);
    }
  }

  private addToBuffer(content: string): void {
    this.buffer.push({
      id: crypto.randomUUID(),
      content,
      timestamp: Date.now(),
    });

    // Start flush timer if not running
    if (!this.flushTimer) {
      this.flushTimer = setTimeout(() => {
        this.flushTimer = null;
      }, this.flushInterval);
    }
  }

  private flush(onChunk: (content: string) => void): void {
    if (this.buffer.length === 0) return;

    const combined = this.buffer.map(chunk => chunk.content).join('');
    onChunk(combined);

    this.buffer = [];

    if (this.flushTimer) {
      clearTimeout(this.flushTimer);
      this.flushTimer = null;
    }
  }

  pause(): void {
    this.paused = true;
    this.emit('paused');
  }

  resume(): void {
    this.paused = false;
    this.emit('resumed');
  }

  private waitForResume(): Promise<void> {
    return new Promise((resolve) => {
      if (!this.paused) {
        resolve();
        return;
      }

      const handler = () => {
        this.off('resumed', handler);
        resolve();
      };

      this.on('resumed', handler);
    });
  }

  destroy(): void {
    if (this.flushTimer) {
      clearTimeout(this.flushTimer);
      this.flushTimer = null;
    }
    this.buffer = [];
    this.removeAllListeners();
  }
}
```

### ✅ 완료 기준

- [ ] 청크 버퍼링
- [ ] 백프레셔 처리
- [ ] 스트림 일시정지/재개
- [ ] 에러 복구

### 📝 Commit Message

```
feat(ai): optimize streaming performance

- Implement StreamOptimizer with buffering
- Add backpressure handling
- Support pause/resume
- Improve chunk batching

Performance:
- 10-chunk buffering
- 50ms flush interval
- Memory-efficient processing
```

---

## Commit 101: 토큰 사용량 추적

### 📋 작업 내용

1. **사용량 데이터베이스**
2. **비용 계산**
3. **사용량 차트**
4. **알림 임계값**

### 📁 파일 구조

```
src/main/ai/
├── TokenTracker.ts        # 토큰 추적
└── CostCalculator.ts      # 비용 계산

src/renderer/components/ai/
├── TokenUsageDashboard.tsx # 대시보드
└── UsageChart.tsx         # 차트
```

### 1️⃣ Token Tracker

**파일**: `src/main/ai/TokenTracker.ts`

```typescript
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';
import type { TokenUsage } from '@/renderer/types/ai';

interface UsageRecord {
  id: number;
  sessionId: string;
  model: string;
  promptTokens: number;
  completionTokens: number;
  totalTokens: number;
  cost: number;
  timestamp: number;
}

export class TokenTracker {
  private db: Database.Database;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'token-usage.db');
    this.db = new Database(dbPath);
    this.initialize();
  }

  private initialize(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS token_usage (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        session_id TEXT NOT NULL,
        model TEXT NOT NULL,
        prompt_tokens INTEGER NOT NULL,
        completion_tokens INTEGER NOT NULL,
        total_tokens INTEGER NOT NULL,
        cost REAL NOT NULL,
        timestamp INTEGER NOT NULL
      );

      CREATE INDEX IF NOT EXISTS idx_session_id ON token_usage(session_id);
      CREATE INDEX IF NOT EXISTS idx_timestamp ON token_usage(timestamp);
      CREATE INDEX IF NOT EXISTS idx_model ON token_usage(model);
    `);
  }

  recordUsage(
    sessionId: string,
    model: string,
    promptTokens: number,
    completionTokens: number,
    cost: number
  ): void {
    this.db.prepare(`
      INSERT INTO token_usage (session_id, model, prompt_tokens, completion_tokens, total_tokens, cost, timestamp)
      VALUES (?, ?, ?, ?, ?, ?, ?)
    `).run(
      sessionId,
      model,
      promptTokens,
      completionTokens,
      promptTokens + completionTokens,
      cost,
      Date.now()
    );
  }

  getSessionUsage(sessionId: string): TokenUsage {
    const result = this.db.prepare(`
      SELECT
        SUM(prompt_tokens) as prompt,
        SUM(completion_tokens) as completion,
        SUM(total_tokens) as total,
        SUM(cost) as cost
      FROM token_usage
      WHERE session_id = ?
    `).get(sessionId) as any;

    return {
      prompt: result.prompt || 0,
      completion: result.completion || 0,
      total: result.total || 0,
      cost: result.cost || 0,
    };
  }

  getTotalUsage(startDate?: number, endDate?: number): TokenUsage {
    let query = `
      SELECT
        SUM(prompt_tokens) as prompt,
        SUM(completion_tokens) as completion,
        SUM(total_tokens) as total,
        SUM(cost) as cost
      FROM token_usage
    `;

    const params: number[] = [];

    if (startDate || endDate) {
      query += ' WHERE ';
      const conditions: string[] = [];

      if (startDate) {
        conditions.push('timestamp >= ?');
        params.push(startDate);
      }

      if (endDate) {
        conditions.push('timestamp <= ?');
        params.push(endDate);
      }

      query += conditions.join(' AND ');
    }

    const result = this.db.prepare(query).get(...params) as any;

    return {
      prompt: result.prompt || 0,
      completion: result.completion || 0,
      total: result.total || 0,
      cost: result.cost || 0,
    };
  }

  getUsageByModel(): Array<{ model: string; usage: TokenUsage }> {
    const rows = this.db.prepare(`
      SELECT
        model,
        SUM(prompt_tokens) as prompt,
        SUM(completion_tokens) as completion,
        SUM(total_tokens) as total,
        SUM(cost) as cost
      FROM token_usage
      GROUP BY model
    `).all() as any[];

    return rows.map(row => ({
      model: row.model,
      usage: {
        prompt: row.prompt,
        completion: row.completion,
        total: row.total,
        cost: row.cost,
      },
    }));
  }

  getUsageTimeline(interval: 'hour' | 'day' | 'week' = 'day'): Array<{
    timestamp: number;
    usage: TokenUsage;
  }> {
    let groupBy: string;

    switch (interval) {
      case 'hour':
        groupBy = 'timestamp / (1000 * 60 * 60)';
        break;
      case 'week':
        groupBy = 'timestamp / (1000 * 60 * 60 * 24 * 7)';
        break;
      default:
        groupBy = 'timestamp / (1000 * 60 * 60 * 24)';
    }

    const rows = this.db.prepare(`
      SELECT
        ${groupBy} as period,
        SUM(prompt_tokens) as prompt,
        SUM(completion_tokens) as completion,
        SUM(total_tokens) as total,
        SUM(cost) as cost
      FROM token_usage
      GROUP BY period
      ORDER BY period ASC
    `).all() as any[];

    return rows.map(row => ({
      timestamp: row.period * (interval === 'hour' ? 3600000 : interval === 'week' ? 604800000 : 86400000),
      usage: {
        prompt: row.prompt,
        completion: row.completion,
        total: row.total,
        cost: row.cost,
      },
    }));
  }

  close(): void {
    this.db.close();
  }
}

export const tokenTracker = new TokenTracker();
```

### 2️⃣ Token Usage Dashboard

**파일**: `src/renderer/components/ai/TokenUsageDashboard.tsx`

```typescript
import React, { useState, useEffect } from 'react';
import { TrendingUp, DollarSign, Zap } from 'lucide-react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import type { TokenUsage } from '@/types/ai';

export function TokenUsageDashboard() {
  const [totalUsage, setTotalUsage] = useState<TokenUsage | null>(null);
  const [usageByModel, setUsageByModel] = useState<Array<{ model: string; usage: TokenUsage }>>([]);

  useEffect(() => {
    loadUsage();
  }, []);

  const loadUsage = async () => {
    if (!window.electronAPI) return;

    const [total, byModel] = await Promise.all([
      window.electronAPI.getTotalTokenUsage(),
      window.electronAPI.getTokenUsageByModel(),
    ]);

    setTotalUsage(total);
    setUsageByModel(byModel);
  };

  if (!totalUsage) return null;

  return (
    <div className="space-y-4">
      <h3 className="font-semibold">Token Usage</h3>

      {/* Summary Cards */}
      <div className="grid grid-cols-3 gap-4">
        <Card>
          <CardHeader className="pb-2">
            <CardTitle className="text-sm font-medium flex items-center gap-2">
              <Zap className="h-4 w-4" />
              Total Tokens
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">{totalUsage.total.toLocaleString()}</div>
            <p className="text-xs text-muted-foreground mt-1">
              {totalUsage.prompt.toLocaleString()} prompt + {totalUsage.completion.toLocaleString()} completion
            </p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="pb-2">
            <CardTitle className="text-sm font-medium flex items-center gap-2">
              <DollarSign className="h-4 w-4" />
              Total Cost
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">${totalUsage.cost.toFixed(2)}</div>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="pb-2">
            <CardTitle className="text-sm font-medium flex items-center gap-2">
              <TrendingUp className="h-4 w-4" />
              Avg per Session
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">
              {(totalUsage.total / Math.max(usageByModel.length, 1)).toFixed(0)}
            </div>
          </CardContent>
        </Card>
      </div>

      {/* Usage by Model */}
      <div className="space-y-2">
        <h4 className="text-sm font-medium">Usage by Model</h4>
        {usageByModel.map(({ model, usage }) => (
          <div key={model} className="flex items-center justify-between p-3 border rounded-lg">
            <div>
              <div className="font-medium">{model}</div>
              <div className="text-sm text-muted-foreground">
                {usage.total.toLocaleString()} tokens
              </div>
            </div>
            <div className="text-right">
              <div className="font-semibold">${usage.cost.toFixed(2)}</div>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 토큰 사용량 기록
- [ ] 비용 계산
- [ ] 대시보드 UI
- [ ] 모델별 통계

### 📝 Commit Message

```
feat(ai): add token usage tracking

- Implement TokenTracker with SQLite
- Add cost calculation
- Create usage dashboard
- Support timeline visualization

Metrics:
- Total tokens (prompt + completion)
- Cost tracking
- Usage by model
- Timeline charts
```

---

## Commit 102: Model Comparison

### 📋 작업 내용

1. **멀티 모델 실행**
2. **응답 비교 UI**
3. **품질 평가**
4. **성능 메트릭**

### 📁 파일 구조

```
src/main/ai/
└── ModelComparison.ts     # 모델 비교

src/renderer/components/ai/
├── ModelComparison.tsx    # 비교 UI
└── ComparisonResults.tsx  # 결과 표시
```

### 1️⃣ Model Comparison

**파일**: `src/main/ai/ModelComparison.ts`

```typescript
import { OpenAI } from 'openai';
import { Anthropic } from '@anthropic-ai/sdk';

interface ComparisonResult {
  model: string;
  response: string;
  tokens: number;
  latency: number;
  cost: number;
}

export class ModelComparison {
  private openai: OpenAI;
  private anthropic: Anthropic;

  constructor(openaiKey: string, anthropicKey: string) {
    this.openai = new OpenAI({ apiKey: openaiKey });
    this.anthropic = new Anthropic({ apiKey: anthropicKey });
  }

  async compare(
    prompt: string,
    models: string[]
  ): Promise<ComparisonResult[]> {
    const results = await Promise.all(
      models.map(model => this.runModel(model, prompt))
    );

    return results;
  }

  private async runModel(model: string, prompt: string): Promise<ComparisonResult> {
    const startTime = Date.now();

    try {
      if (model.startsWith('gpt-')) {
        return await this.runOpenAI(model, prompt, startTime);
      } else if (model.startsWith('claude-')) {
        return await this.runAnthropic(model, prompt, startTime);
      }

      throw new Error(`Unknown model: ${model}`);
    } catch (error) {
      return {
        model,
        response: `Error: ${(error as Error).message}`,
        tokens: 0,
        latency: Date.now() - startTime,
        cost: 0,
      };
    }
  }

  private async runOpenAI(model: string, prompt: string, startTime: number): Promise<ComparisonResult> {
    const response = await this.openai.chat.completions.create({
      model,
      messages: [{ role: 'user', content: prompt }],
    });

    const latency = Date.now() - startTime;
    const tokens = response.usage?.total_tokens || 0;
    const cost = this.calculateCost(model, response.usage?.prompt_tokens || 0, response.usage?.completion_tokens || 0);

    return {
      model,
      response: response.choices[0].message.content || '',
      tokens,
      latency,
      cost,
    };
  }

  private async runAnthropic(model: string, prompt: string, startTime: number): Promise<ComparisonResult> {
    const response = await this.anthropic.messages.create({
      model,
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    });

    const latency = Date.now() - startTime;
    const tokens = response.usage.input_tokens + response.usage.output_tokens;
    const cost = this.calculateCost(model, response.usage.input_tokens, response.usage.output_tokens);

    const content = response.content[0];
    const text = content.type === 'text' ? content.text : '';

    return {
      model,
      response: text,
      tokens,
      latency,
      cost,
    };
  }

  private calculateCost(model: string, promptTokens: number, completionTokens: number): number {
    const pricing: Record<string, { prompt: number; completion: number }> = {
      'gpt-4': { prompt: 0.03, completion: 0.06 },
      'gpt-3.5-turbo': { prompt: 0.0015, completion: 0.002 },
      'claude-3-opus': { prompt: 0.015, completion: 0.075 },
      'claude-3-sonnet': { prompt: 0.003, completion: 0.015 },
    };

    const modelPricing = pricing[model] || { prompt: 0, completion: 0 };

    return (
      (promptTokens / 1000) * modelPricing.prompt +
      (completionTokens / 1000) * modelPricing.completion
    );
  }
}
```

### 2️⃣ Model Comparison UI

**파일**: `src/renderer/components/ai/ModelComparison.tsx`

```typescript
import React, { useState } from 'react';
import { GitCompare, Play } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { Checkbox } from '@/components/ui/checkbox';
import { Label } from '@/components/ui/label';
import { ComparisonResults } from './ComparisonResults';

export function ModelComparison() {
  const [prompt, setPrompt] = useState('');
  const [selectedModels, setSelectedModels] = useState<string[]>(['gpt-4', 'claude-3-sonnet']);
  const [results, setResults] = useState<any[]>([]);
  const [loading, setLoading] = useState(false);

  const availableModels = [
    'gpt-4',
    'gpt-3.5-turbo',
    'claude-3-opus',
    'claude-3-sonnet',
  ];

  const handleCompare = async () => {
    if (!window.electronAPI || !prompt) return;

    setLoading(true);

    try {
      const results = await window.electronAPI.compareModels(prompt, selectedModels);
      setResults(results);
    } catch (error) {
      console.error('Comparison failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="space-y-4">
      <h3 className="font-semibold flex items-center gap-2">
        <GitCompare className="h-4 w-4" />
        Model Comparison
      </h3>

      {/* Prompt */}
      <div className="space-y-2">
        <Label>Prompt</Label>
        <Textarea
          placeholder="Enter a prompt to compare models..."
          value={prompt}
          onChange={(e) => setPrompt(e.target.value)}
          rows={4}
        />
      </div>

      {/* Model Selection */}
      <div className="space-y-2">
        <Label>Models to Compare</Label>
        <div className="grid grid-cols-2 gap-2">
          {availableModels.map(model => (
            <div key={model} className="flex items-center gap-2">
              <Checkbox
                id={model}
                checked={selectedModels.includes(model)}
                onCheckedChange={(checked) => {
                  if (checked) {
                    setSelectedModels([...selectedModels, model]);
                  } else {
                    setSelectedModels(selectedModels.filter(m => m !== model));
                  }
                }}
              />
              <Label htmlFor={model} className="cursor-pointer">
                {model}
              </Label>
            </div>
          ))}
        </div>
      </div>

      {/* Run Button */}
      <Button
        onClick={handleCompare}
        disabled={loading || !prompt || selectedModels.length === 0}
        className="w-full"
      >
        <Play className="h-4 w-4 mr-2" />
        {loading ? 'Comparing...' : 'Compare Models'}
      </Button>

      {/* Results */}
      {results.length > 0 && (
        <ComparisonResults results={results} />
      )}
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 멀티 모델 실행
- [ ] 응답 비교 UI
- [ ] 성능 메트릭
- [ ] 비용 비교

### 📝 Commit Message

```
feat(ai): add model comparison tool

- Implement ModelComparison engine
- Support OpenAI and Anthropic models
- Add side-by-side comparison UI
- Track latency and costs

Features:
- Parallel model execution
- Response quality comparison
- Performance metrics
- Cost analysis
```

---

## 🎯 Day 17 완료 체크리스트

### 기능 완성도
- [ ] 스마트 컨텍스트 선택
- [ ] 프롬프트 템플릿
- [ ] 응답 캐싱
- [ ] 스트리밍 최적화
- [ ] 토큰 추적
- [ ] 모델 비교

### Electron 통합
- [ ] Native AI processing
- [ ] GPU acceleration (optional)
- [ ] Resource monitoring
- [ ] Power-efficient processing

---

## 📦 Dependencies

```json
{
  "dependencies": {
    "tiktoken": "^1.0.10",
    "@anthropic-ai/sdk": "^0.9.1"
  }
}
```

---

**다음**: Day 18에서는 보안 및 엔터프라이즈 기능을 구현합니다.
