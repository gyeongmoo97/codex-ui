# Day 16 TODO - 고급 검색 및 인덱싱 (Electron)

> **목표**: 빠른 전문 검색, 시맨틱 검색, 스마트 필터 구현

## 전체 개요

Day 16은 Codex UI에 강력한 검색 기능을 추가합니다:
- 전문 검색 인덱싱 (Full-text search)
- 시맨틱 검색 (Embedding 기반)
- 파일 내용 인덱싱
- 고급 필터 (날짜, 모델, 태그)
- 검색 히스토리
- 검색 결과 하이라이팅

**Electron 특화:**
- SQLite FTS5로 빠른 검색
- Native indexing (백그라운드 프로세스)
- Spotlight-style 검색 UI (Cmd+Shift+F)
- 파일 시스템 워치로 자동 인덱싱
- Power-efficient indexing

---

## Commit 91: 전문 검색 인덱싱

### 📋 작업 내용

1. **SQLite FTS5 설정**
2. **인덱스 구축**
3. **검색 쿼리 엔진**
4. **인덱스 업데이트**

### 📁 파일 구조

```
src/main/search/
├── SearchIndexer.ts       # 인덱싱 엔진
├── SearchQuery.ts         # 쿼리 빌더
├── types.ts               # 검색 타입
└── fts.sql               # FTS5 스키마

src/main/handlers/
└── search.ts              # 검색 IPC

src/renderer/components/search/
├── SearchBar.tsx          # 검색 바
├── SearchResults.tsx      # 결과 목록
└── SearchFilters.tsx      # 필터
```

### 1️⃣ 검색 타입 정의

**파일**: `src/renderer/types/search.ts`

```typescript
export interface SearchQuery {
  text: string;
  filters?: {
    dateRange?: {
      start: number;
      end: number;
    };
    models?: string[];
    tags?: string[];
    hasAttachments?: boolean;
    minLength?: number;
  };
  limit?: number;
  offset?: number;
}

export interface SearchResult {
  id: string;
  type: 'message' | 'session' | 'file';
  sessionId: string;
  messageId?: string;
  content: string;
  snippet: string;
  highlights: Array<{ start: number; end: number }>;
  score: number;
  timestamp: number;
  metadata: {
    model?: string;
    tags?: string[];
    fileType?: string;
  };
}

export interface IndexStats {
  totalDocuments: number;
  totalTokens: number;
  lastIndexed: number;
  indexSize: number; // bytes
}
```

### 2️⃣ SQLite FTS5 스키마

**파일**: `src/main/search/fts.sql`

```sql
-- FTS5 virtual table for messages
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
  session_id,
  message_id,
  content,
  role,
  model,
  tags,
  timestamp,
  tokenize = 'porter unicode61'
);

-- FTS5 virtual table for files
CREATE VIRTUAL TABLE IF NOT EXISTS files_fts USING fts5(
  session_id,
  file_id,
  file_name,
  file_content,
  file_type,
  timestamp,
  tokenize = 'porter unicode61'
);

-- Metadata table
CREATE TABLE IF NOT EXISTS search_metadata (
  id INTEGER PRIMARY KEY,
  last_indexed_at INTEGER NOT NULL,
  total_documents INTEGER NOT NULL,
  index_version TEXT NOT NULL
);
```

### 3️⃣ Search Indexer

**파일**: `src/main/search/SearchIndexer.ts`

```typescript
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';
import fs from 'fs/promises';
import type { IndexStats } from '@/renderer/types/search';

export class SearchIndexer {
  private db: Database.Database;
  private indexing = false;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'search.db');
    this.db = new Database(dbPath);
    this.initialize();
  }

  private initialize(): void {
    // Load FTS5 schema
    const schema = `
      CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
        session_id UNINDEXED,
        message_id UNINDEXED,
        content,
        role UNINDEXED,
        model UNINDEXED,
        tags UNINDEXED,
        timestamp UNINDEXED,
        tokenize = 'porter unicode61'
      );

      CREATE VIRTUAL TABLE IF NOT EXISTS files_fts USING fts5(
        session_id UNINDEXED,
        file_id UNINDEXED,
        file_name,
        file_content,
        file_type UNINDEXED,
        timestamp UNINDEXED,
        tokenize = 'porter unicode61'
      );

      CREATE TABLE IF NOT EXISTS search_metadata (
        id INTEGER PRIMARY KEY CHECK (id = 1),
        last_indexed_at INTEGER NOT NULL,
        total_documents INTEGER NOT NULL,
        index_version TEXT NOT NULL
      );
    `;

    this.db.exec(schema);

    // Initialize metadata
    const metadata = this.db.prepare('SELECT * FROM search_metadata WHERE id = 1').get();
    if (!metadata) {
      this.db.prepare(`
        INSERT INTO search_metadata (id, last_indexed_at, total_documents, index_version)
        VALUES (1, 0, 0, '1.0.0')
      `).run();
    }
  }

  async indexSession(sessionId: string, messages: any[]): Promise<void> {
    const insert = this.db.prepare(`
      INSERT INTO messages_fts (session_id, message_id, content, role, model, tags, timestamp)
      VALUES (?, ?, ?, ?, ?, ?, ?)
    `);

    const insertMany = this.db.transaction((messages: any[]) => {
      for (const msg of messages) {
        insert.run(
          sessionId,
          msg.id,
          msg.content,
          msg.role,
          msg.model || '',
          (msg.tags || []).join(' '),
          msg.timestamp
        );
      }
    });

    insertMany(messages);

    // Update metadata
    this.updateMetadata();
  }

  async indexFile(
    sessionId: string,
    fileId: string,
    fileName: string,
    content: string,
    fileType: string
  ): Promise<void> {
    this.db.prepare(`
      INSERT INTO files_fts (session_id, file_id, file_name, file_content, file_type, timestamp)
      VALUES (?, ?, ?, ?, ?, ?)
    `).run(sessionId, fileId, fileName, content, fileType, Date.now());

    this.updateMetadata();
  }

  async removeSession(sessionId: string): Promise<void> {
    this.db.prepare('DELETE FROM messages_fts WHERE session_id = ?').run(sessionId);
    this.db.prepare('DELETE FROM files_fts WHERE session_id = ?').run(sessionId);
    this.updateMetadata();
  }

  async rebuildIndex(): Promise<void> {
    if (this.indexing) {
      throw new Error('Indexing already in progress');
    }

    this.indexing = true;

    try {
      // Clear existing index
      this.db.exec('DELETE FROM messages_fts');
      this.db.exec('DELETE FROM files_fts');

      // Load all sessions
      const sessionsDir = path.join(app.getPath('userData'), 'sessions');
      const sessionFiles = await fs.readdir(sessionsDir);

      for (const file of sessionFiles) {
        if (!file.endsWith('.json')) continue;

        const sessionPath = path.join(sessionsDir, file);
        const sessionData = JSON.parse(await fs.readFile(sessionPath, 'utf-8'));

        await this.indexSession(sessionData.id, sessionData.messages || []);
      }

      this.updateMetadata();
    } finally {
      this.indexing = false;
    }
  }

  private updateMetadata(): void {
    const totalMessages = this.db.prepare('SELECT COUNT(*) as count FROM messages_fts').get() as { count: number };
    const totalFiles = this.db.prepare('SELECT COUNT(*) as count FROM files_fts').get() as { count: number };

    this.db.prepare(`
      UPDATE search_metadata
      SET last_indexed_at = ?, total_documents = ?
      WHERE id = 1
    `).run(Date.now(), totalMessages.count + totalFiles.count);
  }

  getStats(): IndexStats {
    const metadata = this.db.prepare('SELECT * FROM search_metadata WHERE id = 1').get() as any;
    const dbPath = path.join(app.getPath('userData'), 'search.db');
    const stats = require('fs').statSync(dbPath);

    return {
      totalDocuments: metadata.total_documents,
      totalTokens: 0, // TODO: Calculate
      lastIndexed: metadata.last_indexed_at,
      indexSize: stats.size,
    };
  }

  close(): void {
    this.db.close();
  }
}

export const searchIndexer = new SearchIndexer();
```

### 4️⃣ Search Query Engine

**파일**: `src/main/search/SearchQuery.ts`

```typescript
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';
import type { SearchQuery, SearchResult } from '@/renderer/types/search';

export class SearchQueryEngine {
  private db: Database.Database;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'search.db');
    this.db = new Database(dbPath);
  }

  search(query: SearchQuery): SearchResult[] {
    let sql = `
      SELECT
        session_id,
        message_id,
        content,
        role,
        model,
        tags,
        timestamp,
        rank AS score
      FROM messages_fts
      WHERE messages_fts MATCH ?
    `;

    const params: any[] = [this.buildFTSQuery(query.text)];

    // Apply filters
    if (query.filters) {
      if (query.filters.dateRange) {
        sql += ' AND timestamp >= ? AND timestamp <= ?';
        params.push(query.filters.dateRange.start, query.filters.dateRange.end);
      }

      if (query.filters.models && query.filters.models.length > 0) {
        sql += ` AND model IN (${query.filters.models.map(() => '?').join(',')})`;
        params.push(...query.filters.models);
      }

      if (query.filters.tags && query.filters.tags.length > 0) {
        const tagConditions = query.filters.tags.map(() => 'tags LIKE ?').join(' OR ');
        sql += ` AND (${tagConditions})`;
        params.push(...query.filters.tags.map(tag => `%${tag}%`));
      }
    }

    // Order and limit
    sql += ' ORDER BY score DESC';

    if (query.limit) {
      sql += ' LIMIT ?';
      params.push(query.limit);
    }

    if (query.offset) {
      sql += ' OFFSET ?';
      params.push(query.offset);
    }

    const rows = this.db.prepare(sql).all(...params) as any[];

    return rows.map(row => ({
      id: row.message_id,
      type: 'message' as const,
      sessionId: row.session_id,
      messageId: row.message_id,
      content: row.content,
      snippet: this.generateSnippet(row.content, query.text),
      highlights: this.findHighlights(row.content, query.text),
      score: Math.abs(row.score), // FTS5 rank is negative
      timestamp: row.timestamp,
      metadata: {
        model: row.model,
        tags: row.tags ? row.tags.split(' ') : [],
      },
    }));
  }

  private buildFTSQuery(text: string): string {
    // Support phrase queries with quotes
    if (text.startsWith('"') && text.endsWith('"')) {
      return text;
    }

    // Split into terms and build OR query
    const terms = text
      .toLowerCase()
      .split(/\s+/)
      .filter(term => term.length > 0)
      .map(term => `"${term}"*`); // Prefix matching

    return terms.join(' OR ');
  }

  private generateSnippet(content: string, query: string, maxLength = 200): string {
    const terms = query.toLowerCase().split(/\s+/);
    const lowerContent = content.toLowerCase();

    // Find first occurrence
    let firstIndex = -1;
    for (const term of terms) {
      const index = lowerContent.indexOf(term);
      if (index !== -1 && (firstIndex === -1 || index < firstIndex)) {
        firstIndex = index;
      }
    }

    if (firstIndex === -1) {
      return content.slice(0, maxLength) + '...';
    }

    // Extract snippet around match
    const start = Math.max(0, firstIndex - 50);
    const end = Math.min(content.length, firstIndex + maxLength);
    const snippet = content.slice(start, end);

    return (start > 0 ? '...' : '') + snippet + (end < content.length ? '...' : '');
  }

  private findHighlights(content: string, query: string): Array<{ start: number; end: number }> {
    const terms = query.toLowerCase().split(/\s+/);
    const lowerContent = content.toLowerCase();
    const highlights: Array<{ start: number; end: number }> = [];

    for (const term of terms) {
      let index = 0;
      while ((index = lowerContent.indexOf(term, index)) !== -1) {
        highlights.push({
          start: index,
          end: index + term.length,
        });
        index += term.length;
      }
    }

    return highlights.sort((a, b) => a.start - b.start);
  }

  close(): void {
    this.db.close();
  }
}

export const searchQueryEngine = new SearchQueryEngine();
```

### 5️⃣ Search Bar UI

**파일**: `src/renderer/components/search/SearchBar.tsx`

```typescript
import React, { useState, useEffect, useRef } from 'react';
import { Search, Filter, X } from 'lucide-react';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { SearchResults } from './SearchResults';
import { SearchFilters } from './SearchFilters';
import type { SearchQuery, SearchResult } from '@/types/search';

export function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [showFilters, setShowFilters] = useState(false);
  const [filters, setFilters] = useState<SearchQuery['filters']>({});
  const [isSearching, setIsSearching] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  // Global shortcut: Cmd+Shift+F
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.shiftKey && e.key === 'f') {
        e.preventDefault();
        inputRef.current?.focus();
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, []);

  const handleSearch = async () => {
    if (!query.trim() || !window.electronAPI) return;

    setIsSearching(true);

    try {
      const searchQuery: SearchQuery = {
        text: query,
        filters,
        limit: 50,
      };

      const results = await window.electronAPI.search(searchQuery);
      setResults(results);
    } catch (error) {
      console.error('Search failed:', error);
    } finally {
      setIsSearching(false);
    }
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      handleSearch();
    }
  };

  const handleClear = () => {
    setQuery('');
    setResults([]);
    setFilters({});
  };

  return (
    <div className="space-y-4">
      {/* Search Input */}
      <div className="flex gap-2">
        <div className="flex-1 relative">
          <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
          <Input
            ref={inputRef}
            placeholder="Search messages, files... (Cmd+Shift+F)"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            onKeyDown={handleKeyDown}
            className="pl-10 pr-10"
          />
          {query && (
            <button
              onClick={handleClear}
              className="absolute right-3 top-1/2 -translate-y-1/2 text-muted-foreground hover:text-foreground"
            >
              <X className="h-4 w-4" />
            </button>
          )}
        </div>

        <Button
          variant="outline"
          size="icon"
          onClick={() => setShowFilters(!showFilters)}
          className={showFilters ? 'bg-accent' : ''}
        >
          <Filter className="h-4 w-4" />
        </Button>

        <Button onClick={handleSearch} disabled={isSearching}>
          {isSearching ? 'Searching...' : 'Search'}
        </Button>
      </div>

      {/* Filters */}
      {showFilters && (
        <SearchFilters filters={filters} onChange={setFilters} />
      )}

      {/* Results */}
      {results.length > 0 && (
        <SearchResults results={results} query={query} />
      )}

      {/* No Results */}
      {query && results.length === 0 && !isSearching && (
        <div className="text-center py-8 text-muted-foreground">
          No results found for "{query}"
        </div>
      )}
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] SQLite FTS5 설정
- [ ] 메시지 인덱싱
- [ ] 파일 인덱싱
- [ ] 검색 쿼리 작동
- [ ] 검색 UI 완성
- [ ] 하이라이팅 작동

### 📝 Commit Message

```
feat(search): implement full-text search with FTS5

- Add SQLite FTS5 virtual tables
- Create SearchIndexer for indexing sessions
- Implement SearchQueryEngine with filters
- Add SearchBar UI with Cmd+Shift+F shortcut
- Support snippet generation and highlighting

Features:
- Porter stemming for better matching
- Prefix matching for autocomplete
- Date/model/tag filters
- Background indexing

Electron-specific:
- Native SQLite with better-sqlite3
- Global keyboard shortcuts
- Power-efficient indexing
```

---

## Commit 92: 시맨틱 검색

### 📋 작업 내용

1. **Embedding 생성**
2. **벡터 저장소**
3. **Cosine similarity 검색**
4. **하이브리드 검색**

### 📁 파일 구조

```
src/main/search/
├── EmbeddingEngine.ts     # Embedding 생성
├── VectorStore.ts         # 벡터 저장
└── SemanticSearch.ts      # 시맨틱 검색

src/renderer/components/search/
└── SemanticResults.tsx    # 시맨틱 결과
```

### 1️⃣ Embedding Engine

**파일**: `src/main/search/EmbeddingEngine.ts`

```typescript
import { OpenAI } from 'openai';

export class EmbeddingEngine {
  private client: OpenAI;
  private model = 'text-embedding-3-small';

  constructor(apiKey: string) {
    this.client = new OpenAI({ apiKey });
  }

  async generateEmbedding(text: string): Promise<number[]> {
    const response = await this.client.embeddings.create({
      model: this.model,
      input: text,
      encoding_format: 'float',
    });

    return response.data[0].embedding;
  }

  async generateEmbeddings(texts: string[]): Promise<number[][]> {
    const response = await this.client.embeddings.create({
      model: this.model,
      input: texts,
      encoding_format: 'float',
    });

    return response.data.map(d => d.embedding);
  }

  cosineSimilarity(a: number[], b: number[]): number {
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
}
```

### 2️⃣ Vector Store

**파일**: `src/main/search/VectorStore.ts`

```typescript
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';

interface VectorDocument {
  id: string;
  sessionId: string;
  messageId: string;
  content: string;
  embedding: number[];
  timestamp: number;
}

export class VectorStore {
  private db: Database.Database;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'vectors.db');
    this.db = new Database(dbPath);
    this.initialize();
  }

  private initialize(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS vectors (
        id TEXT PRIMARY KEY,
        session_id TEXT NOT NULL,
        message_id TEXT NOT NULL,
        content TEXT NOT NULL,
        embedding BLOB NOT NULL,
        timestamp INTEGER NOT NULL
      );

      CREATE INDEX IF NOT EXISTS idx_session_id ON vectors(session_id);
      CREATE INDEX IF NOT EXISTS idx_timestamp ON vectors(timestamp);
    `);
  }

  async addDocument(doc: VectorDocument): Promise<void> {
    // Convert embedding to blob
    const embeddingBuffer = Buffer.from(
      new Float32Array(doc.embedding).buffer
    );

    this.db.prepare(`
      INSERT OR REPLACE INTO vectors (id, session_id, message_id, content, embedding, timestamp)
      VALUES (?, ?, ?, ?, ?, ?)
    `).run(
      doc.id,
      doc.sessionId,
      doc.messageId,
      doc.content,
      embeddingBuffer,
      doc.timestamp
    );
  }

  async search(
    queryEmbedding: number[],
    limit = 10,
    threshold = 0.7
  ): Promise<Array<VectorDocument & { similarity: number }>> {
    const rows = this.db.prepare('SELECT * FROM vectors').all() as any[];

    const results = rows
      .map(row => {
        // Convert blob back to array
        const embedding = Array.from(
          new Float32Array(row.embedding.buffer)
        );

        const similarity = this.cosineSimilarity(queryEmbedding, embedding);

        return {
          id: row.id,
          sessionId: row.session_id,
          messageId: row.message_id,
          content: row.content,
          embedding,
          timestamp: row.timestamp,
          similarity,
        };
      })
      .filter(result => result.similarity >= threshold)
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, limit);

    return results;
  }

  private cosineSimilarity(a: number[], b: number[]): number {
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

  async removeSession(sessionId: string): Promise<void> {
    this.db.prepare('DELETE FROM vectors WHERE session_id = ?').run(sessionId);
  }

  close(): void {
    this.db.close();
  }
}

export const vectorStore = new VectorStore();
```

### 3️⃣ Semantic Search

**파일**: `src/main/search/SemanticSearch.ts`

```typescript
import { EmbeddingEngine } from './EmbeddingEngine';
import { VectorStore, vectorStore } from './VectorStore';
import { searchQueryEngine } from './SearchQuery';
import type { SearchQuery, SearchResult } from '@/renderer/types/search';

export class SemanticSearch {
  private embeddingEngine: EmbeddingEngine | null = null;

  async initialize(apiKey: string): Promise<void> {
    this.embeddingEngine = new EmbeddingEngine(apiKey);
  }

  async search(query: SearchQuery): Promise<SearchResult[]> {
    if (!this.embeddingEngine) {
      throw new Error('Semantic search not initialized');
    }

    // Generate query embedding
    const queryEmbedding = await this.embeddingEngine.generateEmbedding(query.text);

    // Search vectors
    const vectorResults = await vectorStore.search(
      queryEmbedding,
      query.limit || 20,
      0.7 // Similarity threshold
    );

    return vectorResults.map(result => ({
      id: result.messageId,
      type: 'message' as const,
      sessionId: result.sessionId,
      messageId: result.messageId,
      content: result.content,
      snippet: result.content.slice(0, 200),
      highlights: [],
      score: result.similarity,
      timestamp: result.timestamp,
      metadata: {},
    }));
  }

  async hybridSearch(query: SearchQuery): Promise<SearchResult[]> {
    // Run both FTS and semantic search
    const [ftsResults, semanticResults] = await Promise.all([
      searchQueryEngine.search(query),
      this.search(query),
    ]);

    // Merge and re-rank
    const merged = new Map<string, SearchResult>();

    // Add FTS results with boosted score
    for (const result of ftsResults) {
      merged.set(result.id, {
        ...result,
        score: result.score * 2, // Boost exact matches
      });
    }

    // Add semantic results
    for (const result of semanticResults) {
      const existing = merged.get(result.id);
      if (existing) {
        // Combine scores
        existing.score = Math.max(existing.score, result.score);
      } else {
        merged.set(result.id, result);
      }
    }

    // Sort by combined score
    return Array.from(merged.values())
      .sort((a, b) => b.score - a.score)
      .slice(0, query.limit || 50);
  }

  async indexMessage(
    sessionId: string,
    messageId: string,
    content: string
  ): Promise<void> {
    if (!this.embeddingEngine) return;

    const embedding = await this.embeddingEngine.generateEmbedding(content);

    await vectorStore.addDocument({
      id: `${sessionId}-${messageId}`,
      sessionId,
      messageId,
      content,
      embedding,
      timestamp: Date.now(),
    });
  }
}

export const semanticSearch = new SemanticSearch();
```

### ✅ 완료 기준

- [ ] Embedding 생성 작동
- [ ] 벡터 저장소 구현
- [ ] 코사인 유사도 검색
- [ ] 하이브리드 검색 구현

### 📝 Commit Message

```
feat(search): add semantic search with embeddings

- Implement EmbeddingEngine with OpenAI
- Create VectorStore with SQLite
- Add cosine similarity search
- Implement hybrid search (FTS + semantic)

Features:
- text-embedding-3-small model
- Float32 vector storage
- Similarity threshold filtering
- Score combination for hybrid results
```

---

## Commit 93: 파일 내용 인덱싱

### 📋 작업 내용

1. **텍스트 추출 (PDF, DOCX, TXT)**
2. **이미지 OCR**
3. **코드 파일 인덱싱**
4. **파일 워처**

### 📁 파일 구조

```
src/main/search/
├── FileIndexer.ts         # 파일 인덱싱
├── TextExtractor.ts       # 텍스트 추출
└── FileWatcher.ts         # 파일 변경 감지
```

### 1️⃣ File Indexer

**파일**: `src/main/search/FileIndexer.ts`

```typescript
import { searchIndexer } from './SearchIndexer';
import { TextExtractor } from './TextExtractor';
import path from 'path';
import fs from 'fs/promises';

export class FileIndexer {
  private textExtractor = new TextExtractor();

  async indexFile(
    sessionId: string,
    fileId: string,
    filePath: string
  ): Promise<void> {
    const ext = path.extname(filePath).toLowerCase();
    const fileName = path.basename(filePath);

    let content = '';

    try {
      // Extract text based on file type
      if (['.txt', '.md', '.js', '.ts', '.tsx', '.jsx', '.py', '.java', '.go', '.rs'].includes(ext)) {
        content = await fs.readFile(filePath, 'utf-8');
      } else if (ext === '.pdf') {
        content = await this.textExtractor.extractFromPDF(filePath);
      } else if (['.doc', '.docx'].includes(ext)) {
        content = await this.textExtractor.extractFromDocx(filePath);
      } else if (['.png', '.jpg', '.jpeg'].includes(ext)) {
        content = await this.textExtractor.extractFromImage(filePath);
      }

      // Index the file
      await searchIndexer.indexFile(
        sessionId,
        fileId,
        fileName,
        content,
        ext
      );
    } catch (error) {
      console.error(`Failed to index file ${filePath}:`, error);
    }
  }

  async indexDirectory(sessionId: string, dirPath: string): Promise<void> {
    const entries = await fs.readdir(dirPath, { withFileTypes: true });

    for (const entry of entries) {
      const fullPath = path.join(dirPath, entry.name);

      if (entry.isFile()) {
        await this.indexFile(sessionId, entry.name, fullPath);
      } else if (entry.isDirectory()) {
        await this.indexDirectory(sessionId, fullPath);
      }
    }
  }
}

export const fileIndexer = new FileIndexer();
```

### 2️⃣ Text Extractor

**파일**: `src/main/search/TextExtractor.ts`

```typescript
import * as pdfjsLib from 'pdfjs-dist';
import { createWorker } from 'tesseract.js';
import mammoth from 'mammoth';
import fs from 'fs/promises';

export class TextExtractor {
  async extractFromPDF(filePath: string): Promise<string> {
    const data = await fs.readFile(filePath);
    const pdf = await pdfjsLib.getDocument({ data }).promise;

    let text = '';

    for (let i = 1; i <= pdf.numPages; i++) {
      const page = await pdf.getPage(i);
      const content = await page.getTextContent();

      const pageText = content.items
        .map((item: any) => item.str)
        .join(' ');

      text += pageText + '\n';
    }

    return text;
  }

  async extractFromDocx(filePath: string): Promise<string> {
    const buffer = await fs.readFile(filePath);
    const result = await mammoth.extractRawText({ buffer });
    return result.value;
  }

  async extractFromImage(filePath: string): Promise<string> {
    const worker = await createWorker('eng');

    try {
      const { data } = await worker.recognize(filePath);
      return data.text;
    } finally {
      await worker.terminate();
    }
  }
}
```

### 3️⃣ File Watcher

**파일**: `src/main/search/FileWatcher.ts`

```typescript
import chokidar from 'chokidar';
import { fileIndexer } from './FileIndexer';
import { app } from 'electron';
import path from 'path';

export class FileWatcher {
  private watcher: chokidar.FSWatcher | null = null;

  start(sessionId: string): void {
    const sessionsDir = path.join(app.getPath('userData'), 'sessions', sessionId, 'files');

    this.watcher = chokidar.watch(sessionsDir, {
      persistent: true,
      ignoreInitial: false,
      awaitWriteFinish: {
        stabilityThreshold: 2000,
        pollInterval: 100,
      },
    });

    this.watcher
      .on('add', async (filePath) => {
        const fileId = path.basename(filePath);
        await fileIndexer.indexFile(sessionId, fileId, filePath);
      })
      .on('change', async (filePath) => {
        const fileId = path.basename(filePath);
        await fileIndexer.indexFile(sessionId, fileId, filePath);
      });
  }

  stop(): void {
    if (this.watcher) {
      this.watcher.close();
      this.watcher = null;
    }
  }
}

export const fileWatcher = new FileWatcher();
```

### ✅ 완료 기준

- [ ] PDF 텍스트 추출
- [ ] DOCX 텍스트 추출
- [ ] 이미지 OCR
- [ ] 코드 파일 인덱싱
- [ ] 파일 워처 작동

### 📝 Commit Message

```
feat(search): add file content indexing

- Implement TextExtractor for PDF/DOCX/images
- Add FileIndexer for various file types
- Create FileWatcher for auto-indexing
- Support OCR with Tesseract.js

Supported formats:
- Text files (txt, md, code files)
- PDF documents
- Word documents
- Images (OCR)
```

---

## Commit 94: 고급 필터

### 📋 작업 내용

1. **날짜 범위 필터**
2. **모델 필터**
3. **태그 필터**
4. **파일 타입 필터**

### 📁 파일 구조

```
src/renderer/components/search/
├── SearchFilters.tsx      # 필터 UI
└── FilterBadges.tsx       # Active filters
```

### 1️⃣ Search Filters UI

**파일**: `src/renderer/components/search/SearchFilters.tsx`

```typescript
import React from 'react';
import { Calendar, Tag, FileType } from 'lucide-react';
import { Label } from '@/components/ui/label';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Badge } from '@/components/ui/badge';
import type { SearchQuery } from '@/types/search';

interface SearchFiltersProps {
  filters: SearchQuery['filters'];
  onChange: (filters: SearchQuery['filters']) => void;
}

export function SearchFilters({ filters, onChange }: SearchFiltersProps) {
  const handleDateRangeChange = (start: string, end: string) => {
    onChange({
      ...filters,
      dateRange: {
        start: new Date(start).getTime(),
        end: new Date(end).getTime(),
      },
    });
  };

  const handleModelChange = (model: string) => {
    const models = filters?.models || [];
    const newModels = models.includes(model)
      ? models.filter(m => m !== model)
      : [...models, model];

    onChange({
      ...filters,
      models: newModels,
    });
  };

  const handleTagChange = (tag: string) => {
    const tags = filters?.tags || [];
    const newTags = tags.includes(tag)
      ? tags.filter(t => t !== tag)
      : [...tags, tag];

    onChange({
      ...filters,
      tags: newTags,
    });
  };

  return (
    <div className="space-y-4 p-4 border rounded-lg">
      <h4 className="font-semibold">Filters</h4>

      {/* Date Range */}
      <div className="space-y-2">
        <Label className="flex items-center gap-2">
          <Calendar className="h-4 w-4" />
          Date Range
        </Label>
        <div className="flex gap-2">
          <Input
            type="date"
            onChange={(e) => {
              const end = filters?.dateRange?.end || Date.now();
              handleDateRangeChange(e.target.value, new Date(end).toISOString());
            }}
          />
          <Input
            type="date"
            onChange={(e) => {
              const start = filters?.dateRange?.start || 0;
              handleDateRangeChange(new Date(start).toISOString(), e.target.value);
            }}
          />
        </div>
      </div>

      {/* Models */}
      <div className="space-y-2">
        <Label>AI Models</Label>
        <div className="flex flex-wrap gap-2">
          {['gpt-4', 'gpt-3.5-turbo', 'claude-3-opus', 'claude-3-sonnet'].map(model => (
            <Badge
              key={model}
              variant={filters?.models?.includes(model) ? 'default' : 'outline'}
              className="cursor-pointer"
              onClick={() => handleModelChange(model)}
            >
              {model}
            </Badge>
          ))}
        </div>
      </div>

      {/* Tags */}
      <div className="space-y-2">
        <Label className="flex items-center gap-2">
          <Tag className="h-4 w-4" />
          Tags
        </Label>
        <Input
          placeholder="Add tag..."
          onKeyDown={(e) => {
            if (e.key === 'Enter') {
              const value = e.currentTarget.value.trim();
              if (value) {
                handleTagChange(value);
                e.currentTarget.value = '';
              }
            }
          }}
        />
        {filters?.tags && filters.tags.length > 0 && (
          <div className="flex flex-wrap gap-2">
            {filters.tags.map(tag => (
              <Badge
                key={tag}
                variant="secondary"
                className="cursor-pointer"
                onClick={() => handleTagChange(tag)}
              >
                {tag} ×
              </Badge>
            ))}
          </div>
        )}
      </div>

      {/* File Type */}
      <div className="space-y-2">
        <Label className="flex items-center gap-2">
          <FileType className="h-4 w-4" />
          Has Attachments
        </Label>
        <Select
          value={filters?.hasAttachments ? 'yes' : 'no'}
          onValueChange={(v) => onChange({
            ...filters,
            hasAttachments: v === 'yes',
          })}
        >
          <SelectTrigger>
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="yes">Yes</SelectItem>
            <SelectItem value="no">No</SelectItem>
          </SelectContent>
        </Select>
      </div>
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 날짜 범위 필터
- [ ] 모델 필터
- [ ] 태그 필터
- [ ] 파일 타입 필터

### 📝 Commit Message

```
feat(search): add advanced search filters

- Implement date range filtering
- Add model selection filter
- Create tag-based filtering
- Support file type filtering

UI:
- Interactive filter badges
- Date picker inputs
- Tag autocomplete
```

---

## Commit 95: 검색 히스토리

### 📋 작업 내용

1. **검색 기록 저장**
2. **자동완성**
3. **인기 검색어**
4. **검색 통계**

### 📁 파일 구조

```
src/main/search/
└── SearchHistory.ts       # 검색 기록

src/renderer/components/search/
├── SearchAutocomplete.tsx # 자동완성
└── PopularSearches.tsx    # 인기 검색어
```

### 1️⃣ Search History

**파일**: `src/main/search/SearchHistory.ts`

```typescript
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';

interface SearchHistoryEntry {
  id: number;
  query: string;
  timestamp: number;
  resultsCount: number;
}

export class SearchHistory {
  private db: Database.Database;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'search.db');
    this.db = new Database(dbPath);
    this.initialize();
  }

  private initialize(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS search_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        query TEXT NOT NULL,
        timestamp INTEGER NOT NULL,
        results_count INTEGER NOT NULL
      );

      CREATE INDEX IF NOT EXISTS idx_timestamp ON search_history(timestamp);
      CREATE INDEX IF NOT EXISTS idx_query ON search_history(query);
    `);
  }

  addEntry(query: string, resultsCount: number): void {
    this.db.prepare(`
      INSERT INTO search_history (query, timestamp, results_count)
      VALUES (?, ?, ?)
    `).run(query, Date.now(), resultsCount);
  }

  getRecentSearches(limit = 10): SearchHistoryEntry[] {
    return this.db.prepare(`
      SELECT DISTINCT query, MAX(timestamp) as timestamp, results_count
      FROM search_history
      GROUP BY query
      ORDER BY timestamp DESC
      LIMIT ?
    `).all(limit) as SearchHistoryEntry[];
  }

  getPopularSearches(limit = 10): Array<{ query: string; count: number }> {
    return this.db.prepare(`
      SELECT query, COUNT(*) as count
      FROM search_history
      WHERE timestamp > ?
      GROUP BY query
      ORDER BY count DESC
      LIMIT ?
    `).all(Date.now() - 7 * 24 * 60 * 60 * 1000, limit) as Array<{ query: string; count: number }>;
  }

  getSuggestions(prefix: string, limit = 5): string[] {
    const results = this.db.prepare(`
      SELECT DISTINCT query
      FROM search_history
      WHERE query LIKE ?
      ORDER BY timestamp DESC
      LIMIT ?
    `).all(`${prefix}%`, limit) as Array<{ query: string }>;

    return results.map(r => r.query);
  }

  clearHistory(): void {
    this.db.prepare('DELETE FROM search_history').run();
  }

  close(): void {
    this.db.close();
  }
}

export const searchHistory = new SearchHistory();
```

### 2️⃣ Search Autocomplete

**파일**: `src/renderer/components/search/SearchAutocomplete.tsx`

```typescript
import React, { useState, useEffect } from 'react';
import { Clock, TrendingUp } from 'lucide-react';

interface SearchAutocompleteProps {
  query: string;
  onSelect: (query: string) => void;
}

export function SearchAutocomplete({ query, onSelect }: SearchAutocompleteProps) {
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [popular, setPopular] = useState<Array<{ query: string; count: number }>>([]);

  useEffect(() => {
    const loadSuggestions = async () => {
      if (!window.electronAPI) return;

      if (query.length > 0) {
        const results = await window.electronAPI.getSearchSuggestions(query);
        setSuggestions(results);
      } else {
        const results = await window.electronAPI.getPopularSearches();
        setPopular(results);
      }
    };

    loadSuggestions();
  }, [query]);

  if (query.length === 0 && popular.length > 0) {
    return (
      <div className="absolute top-full mt-1 w-full bg-popover border rounded-lg shadow-lg z-50">
        <div className="p-2">
          <div className="text-xs text-muted-foreground mb-2 flex items-center gap-1">
            <TrendingUp className="h-3 w-3" />
            Popular Searches
          </div>
          {popular.map(item => (
            <button
              key={item.query}
              onClick={() => onSelect(item.query)}
              className="w-full text-left px-2 py-1.5 hover:bg-accent rounded flex items-center justify-between"
            >
              <span>{item.query}</span>
              <span className="text-xs text-muted-foreground">{item.count}</span>
            </button>
          ))}
        </div>
      </div>
    );
  }

  if (suggestions.length === 0) return null;

  return (
    <div className="absolute top-full mt-1 w-full bg-popover border rounded-lg shadow-lg z-50">
      <div className="p-2">
        {suggestions.map(suggestion => (
          <button
            key={suggestion}
            onClick={() => onSelect(suggestion)}
            className="w-full text-left px-2 py-1.5 hover:bg-accent rounded flex items-center gap-2"
          >
            <Clock className="h-3 w-3 text-muted-foreground" />
            <span>{suggestion}</span>
          </button>
        ))}
      </div>
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 검색 기록 저장
- [ ] 자동완성 작동
- [ ] 인기 검색어 표시
- [ ] 검색 통계

### 📝 Commit Message

```
feat(search): add search history and autocomplete

- Implement SearchHistory storage
- Add autocomplete suggestions
- Show popular searches
- Track search statistics

Features:
- Recent searches
- Query suggestions
- Trending searches (last 7 days)
- Clear history option
```

---

## Commit 96: 검색 결과 하이라이팅

### 📋 작업 내용

1. **하이라이트 컴포넌트**
2. **스니펫 생성**
3. **컨텍스트 표시**
4. **결과 프리뷰**

### 📁 파일 구조

```
src/renderer/components/search/
├── SearchResults.tsx      # 결과 목록
├── HighlightedText.tsx    # 하이라이트
└── ResultPreview.tsx      # 프리뷰
```

### 1️⃣ Highlighted Text

**파일**: `src/renderer/components/search/HighlightedText.tsx`

```typescript
import React from 'react';

interface HighlightedTextProps {
  text: string;
  highlights: Array<{ start: number; end: number }>;
  className?: string;
}

export function HighlightedText({ text, highlights, className = '' }: HighlightedTextProps) {
  if (highlights.length === 0) {
    return <span className={className}>{text}</span>;
  }

  // Sort highlights by start position
  const sortedHighlights = [...highlights].sort((a, b) => a.start - b.start);

  const parts: React.ReactNode[] = [];
  let lastIndex = 0;

  for (const highlight of sortedHighlights) {
    // Add text before highlight
    if (highlight.start > lastIndex) {
      parts.push(
        <span key={`text-${lastIndex}`}>
          {text.slice(lastIndex, highlight.start)}
        </span>
      );
    }

    // Add highlighted text
    parts.push(
      <mark
        key={`highlight-${highlight.start}`}
        className="bg-yellow-200 dark:bg-yellow-900 font-semibold"
      >
        {text.slice(highlight.start, highlight.end)}
      </mark>
    );

    lastIndex = highlight.end;
  }

  // Add remaining text
  if (lastIndex < text.length) {
    parts.push(
      <span key={`text-${lastIndex}`}>
        {text.slice(lastIndex)}
      </span>
    );
  }

  return <span className={className}>{parts}</span>;
}
```

### 2️⃣ Search Results

**파일**: `src/renderer/components/search/SearchResults.tsx`

```typescript
import React, { useState } from 'react';
import { MessageSquare, FileText, ExternalLink } from 'lucide-react';
import { HighlightedText } from './HighlightedText';
import { ResultPreview } from './ResultPreview';
import { formatDistanceToNow } from 'date-fns';
import type { SearchResult } from '@/types/search';

interface SearchResultsProps {
  results: SearchResult[];
  query: string;
}

export function SearchResults({ results, query }: SearchResultsProps) {
  const [selectedResult, setSelectedResult] = useState<SearchResult | null>(null);

  const handleResultClick = async (result: SearchResult) => {
    setSelectedResult(result);

    // Navigate to session/message
    if (window.electronAPI) {
      await window.electronAPI.navigateToMessage(result.sessionId, result.messageId);
    }
  };

  return (
    <div className="space-y-2">
      <div className="text-sm text-muted-foreground">
        {results.length} result{results.length !== 1 ? 's' : ''}
      </div>

      {results.map(result => (
        <button
          key={result.id}
          onClick={() => handleResultClick(result)}
          className="w-full text-left p-3 border rounded-lg hover:bg-accent transition-colors"
        >
          {/* Header */}
          <div className="flex items-start gap-2 mb-2">
            {result.type === 'message' ? (
              <MessageSquare className="h-4 w-4 mt-0.5 text-muted-foreground flex-shrink-0" />
            ) : (
              <FileText className="h-4 w-4 mt-0.5 text-muted-foreground flex-shrink-0" />
            )}

            <div className="flex-1 min-w-0">
              {/* Snippet with highlights */}
              <div className="text-sm mb-1">
                <HighlightedText
                  text={result.snippet}
                  highlights={result.highlights}
                />
              </div>

              {/* Metadata */}
              <div className="flex items-center gap-3 text-xs text-muted-foreground">
                <span>
                  {formatDistanceToNow(result.timestamp, { addSuffix: true })}
                </span>

                {result.metadata.model && (
                  <span className="px-1.5 py-0.5 bg-secondary rounded">
                    {result.metadata.model}
                  </span>
                )}

                {result.metadata.tags && result.metadata.tags.length > 0 && (
                  <span>
                    {result.metadata.tags.map(tag => (
                      <span key={tag} className="px-1.5 py-0.5 bg-secondary rounded mr-1">
                        #{tag}
                      </span>
                    ))}
                  </span>
                )}

                <span className="ml-auto">
                  Score: {result.score.toFixed(2)}
                </span>
              </div>
            </div>

            <ExternalLink className="h-4 w-4 text-muted-foreground flex-shrink-0" />
          </div>
        </button>
      ))}

      {/* Preview Modal */}
      {selectedResult && (
        <ResultPreview
          result={selectedResult}
          onClose={() => setSelectedResult(null)}
        />
      )}
    </div>
  );
}
```

### ✅ 완료 기준

- [ ] 하이라이트 렌더링
- [ ] 스니펫 표시
- [ ] 메타데이터 표시
- [ ] 결과 프리뷰

### 📝 Commit Message

```
feat(search): add search result highlighting

- Implement HighlightedText component
- Add SearchResults with metadata
- Create ResultPreview modal
- Support navigation to messages

UI:
- Yellow highlights for matches
- Contextual snippets
- Metadata badges (model, tags)
- Relevance scores
```

---

## 🎯 Day 16 완료 체크리스트

### 기능 완성도
- [ ] FTS5 전문 검색
- [ ] 시맨틱 검색
- [ ] 파일 인덱싱
- [ ] 고급 필터
- [ ] 검색 히스토리
- [ ] 결과 하이라이팅

### Electron 통합
- [ ] SQLite 최적화
- [ ] 백그라운드 인덱싱
- [ ] Global shortcuts
- [ ] 파일 워처

---

## 📦 Dependencies

```json
{
  "dependencies": {
    "better-sqlite3": "^9.2.2",
    "pdfjs-dist": "^3.11.174",
    "tesseract.js": "^5.0.3",
    "mammoth": "^1.6.0",
    "chokidar": "^3.5.3"
  }
}
```

---

**다음**: Day 17에서는 AI 향상 기능을 구현합니다.
