# Knowledge Base Implementation Plan (Archived)

> Status: **Not implemented in the current Snailer CLI**.
>
> The current CLI provides:
> - Skills: `snailer skills ...`
> - Persistent notes: `read_notes` / `write_notes` (project root `NOTES.md`)
> - Experimental code indexer: `snailer wiki ...`
>
> This document is kept as a historical design draft. Any referenced commands like
> `snailer kb ...` / `snailer knowledge-base ...` do not exist today and should not be treated as user-facing documentation.

## Overview

Knowledge Base 기능은 사용자가 프로젝트 관련 문서, 가이드라인, API 문서 등을 저장하고 관리하여 AI 에이전트가 작업 시 참조할 수 있도록 하는 기능입니다.

## Goals

1. 로컬 파일 시스템 기반의 문서 관리
2. 인터랙티브한 CLI 인터페이스
3. 플랜별 사용량 제한
4. 효율적인 문서 검색 및 인덱싱
5. 에이전트와의 자연스러운 통합

## Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CLI Interface                            │
│  snailer knowledge-base [subcommand]                        │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│              Knowledge Base Manager                         │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │  File        │  │  Indexer     │  │  Query Engine   │  │
│  │  Manager     │  │              │  │                 │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                  Storage Layer                              │
│  ┌──────────────────┐         ┌─────────────────────────┐  │
│  │   File System    │         │   SQLite Database       │  │
│  │  (~/.snailer/kb) │         │  (metadata + index)     │  │
│  └──────────────────┘         └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
1. User adds file
   ↓
2. File Manager copies file to ~/.snailer/kb/files/{hash}
   ↓
3. Indexer extracts metadata and content
   ↓
4. Database stores metadata and search index
   ↓
5. Agent queries KB when needed
   ↓
6. Query Engine returns relevant documents
```

## Database Schema

### Tables

```sql
-- Files metadata table
CREATE TABLE kb_files (
    id TEXT PRIMARY KEY,              -- UUID
    original_name TEXT NOT NULL,      -- 원본 파일명
    file_path TEXT NOT NULL,          -- 저장된 파일 경로
    file_hash TEXT NOT NULL UNIQUE,   -- SHA256 해시
    file_size INTEGER NOT NULL,       -- 파일 크기 (bytes)
    mime_type TEXT,                   -- MIME 타입
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    tags TEXT,                        -- JSON array of tags
    description TEXT                  -- 사용자 설명
);

-- File content index (for full-text search)
CREATE VIRTUAL TABLE kb_content_fts USING fts5(
    file_id UNINDEXED,
    content,
    tokenize = 'porter unicode61'
);

-- User quota tracking
CREATE TABLE kb_quota (
    user_id TEXT PRIMARY KEY,         -- 사용자 ID (로컬에서는 'local')
    plan_tier TEXT NOT NULL,          -- starter, pro, max
    file_count INTEGER DEFAULT 0,     -- 현재 파일 개수
    total_size INTEGER DEFAULT 0,     -- 총 파일 크기
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- File access logs (for analytics)
CREATE TABLE kb_access_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_id TEXT NOT NULL,
    accessed_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    context TEXT,                     -- 어떤 컨텍스트에서 접근했는지
    FOREIGN KEY (file_id) REFERENCES kb_files(id)
);

-- Indexes
CREATE INDEX idx_files_created_at ON kb_files(created_at DESC);
CREATE INDEX idx_files_hash ON kb_files(file_hash);
CREATE INDEX idx_access_log_file_id ON kb_access_log(file_id);
```

## CLI Interface

### Main Command

```bash
snailer knowledge-base [SUBCOMMAND]
```

### Subcommands

#### 1. `add` - 파일 추가

```bash
# 인터랙티브 모드
snailer knowledge-base add

# 직접 파일 경로 지정
snailer knowledge-base add <FILE_PATH>

# 여러 파일 추가
snailer knowledge-base add <FILE1> <FILE2> <FILE3>

# 설명과 태그 추가
snailer knowledge-base add <FILE_PATH> --description "API documentation" --tags "api,docs"
```

**인터랙티브 모드 플로우:**
```
┌─────────────────────────────────────────────┐
│  Knowledge Base - Add File                  │
├─────────────────────────────────────────────┤
│  Drop file here or enter path:              │
│  > _                                        │
│                                             │
│  [Tab to autocomplete] [Esc to cancel]      │
└─────────────────────────────────────────────┘

After file selection:

┌─────────────────────────────────────────────┐
│  File: project-guidelines.md (12 KB)        │
├─────────────────────────────────────────────┤
│  Description (optional):                    │
│  > Project coding guidelines                │
│                                             │
│  Tags (comma-separated):                    │
│  > guidelines,coding-standards              │
│                                             │
│  [Enter to confirm] [Esc to cancel]         │
└─────────────────────────────────────────────┘
```

#### 2. `list` - 파일 목록 조회

```bash
# 전체 목록
snailer knowledge-base list

# 태그로 필터링
snailer knowledge-base list --tag api

# 검색
snailer knowledge-base list --search "authentication"

# 상세 정보 포함
snailer knowledge-base list --verbose
```

**출력 예시:**
```
📚 Knowledge Base Files (3/5 used - Starter Plan)

ID       Name                    Size    Tags              Added
─────────────────────────────────────────────────────────────────
abc123   api-docs.md            24 KB   api,docs          2 days ago
def456   guidelines.md          12 KB   guidelines        1 week ago
ghi789   architecture.md        45 KB   architecture      2 weeks ago
```

#### 3. `status` - 현재 상태 확인

```bash
snailer knowledge-base status
```

**출력 예시:**
```
📊 Knowledge Base Status

Plan: Starter
Usage: 3/5 files (60%)
Total Size: 81 KB / ∞

Recent Activity:
  • api-docs.md accessed 5 times (last: 2 hours ago)
  • guidelines.md accessed 12 times (last: 1 day ago)
  • architecture.md accessed 3 times (last: 1 week ago)

Most Used Files:
  1. guidelines.md (12 accesses)
  2. api-docs.md (5 accesses)
  3. architecture.md (3 accesses)
```

#### 4. `remove` - 파일 삭제

```bash
# ID로 삭제
snailer knowledge-base remove <FILE_ID>

# 인터랙티브 선택
snailer knowledge-base remove
```

#### 5. `show` - 파일 내용 보기

```bash
# 파일 내용 출력
snailer knowledge-base show <FILE_ID>

# 메타데이터만 보기
snailer knowledge-base show <FILE_ID> --meta
```

#### 6. `search` - 전문 검색

```bash
# 키워드 검색
snailer knowledge-base search "authentication flow"

# 태그와 함께 검색
snailer knowledge-base search "API" --tag docs
```

#### 7. `init` - Knowledge Base 초기화

```bash
snailer knowledge-base init --plan starter
```

## File Structure

```
~/.snailer/
├── kb/
│   ├── files/
│   │   ├── {hash1}/
│   │   │   └── original-filename.md
│   │   ├── {hash2}/
│   │   │   └── another-file.pdf
│   │   └── ...
│   ├── kb.db                 # SQLite database
│   └── config.json           # KB configuration
├── config.toml               # Global snailer config
└── cache/
```

### Configuration (`~/.snailer/kb/config.json`)

```json
{
  "plan": "starter",
  "limits": {
    "max_files": 5,
    "max_file_size_mb": 10
  },
  "indexing": {
    "enabled": true,
    "auto_index": true
  },
  "created_at": "2025-10-29T10:00:00Z"
}
```

## Plan Tiers

| Feature | Starter | Pro | Max |
|---------|---------|-----|-----|
| Max Files | 5 | 15 | ∞ |
| Max File Size | 10 MB | 50 MB | 100 MB |
| Full-text Search | ✓ | ✓ | ✓ |
| Auto-indexing | ✓ | ✓ | ✓ |
| Access Analytics | ✗ | ✓ | ✓ |
| Cloud Sync | ✗ | ✗ | ✓ (future) |

## Implementation Phases

### Phase 1: Core Infrastructure (Week 1)

#### Tasks:
1. **Database Setup**
   - [ ] Create SQLite schema
   - [ ] Implement migration system
   - [ ] Write database helper functions

2. **File Manager**
   - [ ] Implement file copying to KB directory
   - [ ] Calculate and verify file hashes
   - [ ] Handle file deduplication

3. **Basic CLI Commands**
   - [ ] `kb init` - Initialize KB
   - [ ] `kb add <file>` - Add file (basic version)
   - [ ] `kb list` - List files

**Files to Create:**
```
src/
├── commands/
│   └── knowledge_base/
│       ├── mod.rs
│       ├── init.rs
│       ├── add.rs
│       ├── list.rs
│       ├── remove.rs
│       ├── status.rs
│       └── search.rs
├── kb/
│   ├── mod.rs
│   ├── database.rs
│   ├── file_manager.rs
│   ├── indexer.rs
│   ├── quota.rs
│   └── types.rs
```

### Phase 2: Indexing & Search (Week 2)

#### Tasks:
1. **Content Indexing**
   - [ ] Extract text from different file types (MD, TXT, PDF, etc.)
   - [ ] Build full-text search index
   - [ ] Implement relevance ranking

2. **Search Implementation**
   - [ ] `kb search <query>` command
   - [ ] Search with filters (tags, date range)
   - [ ] Fuzzy search support

3. **Quota Management**
   - [ ] Implement plan tier checks
   - [ ] File count/size validation
   - [ ] User-friendly quota error messages

**Dependencies:**
```toml
# Add to Cargo.toml
[dependencies]
rusqlite = { version = "0.31", features = ["bundled"] }
sha2 = "0.10"
mime_guess = "2.0"
walkdir = "2.4"
serde_json = "1.0"
```

### Phase 3: Interactive UI (Week 3)

#### Tasks:
1. **Interactive Add Command**
   - [ ] File path input with autocomplete
   - [ ] Drag & drop support (terminal)
   - [ ] Description and tags input

2. **Interactive Remove**
   - [ ] File selection UI
   - [ ] Confirmation prompt
   - [ ] Batch delete

3. **Status Dashboard**
   - [ ] Usage visualization
   - [ ] Access statistics
   - [ ] Recommendations

**Dependencies:**
```toml
dialoguer = "0.11"  # For interactive prompts
indicatif = "0.17"  # For progress bars
```

### Phase 4: Agent Integration (Week 4)

#### Tasks:
1. **Agent Context Provider**
   - [ ] Automatically suggest KB files for current task
   - [ ] Inject relevant KB content into agent context
   - [ ] Track which files are useful for which tasks

2. **Smart Retrieval**
   - [ ] Semantic search (optional, using embeddings)
   - [ ] Context-aware file suggestion
   - [ ] Usage pattern learning

3. **Agent Commands**
   - [ ] `@kb <query>` - Reference KB in agent conversation
   - [ ] Auto-inject KB content based on task

## Code Examples

### Adding a File

```rust
// src/kb/file_manager.rs

use anyhow::{Context, Result};
use sha2::{Digest, Sha256};
use std::fs;
use std::path::{Path, PathBuf};

pub struct FileManager {
    kb_root: PathBuf,
}

impl FileManager {
    pub fn new(kb_root: PathBuf) -> Self {
        Self { kb_root }
    }

    pub fn add_file(&self, source: &Path) -> Result<FileInfo> {
        // 1. Calculate hash
        let hash = self.calculate_hash(source)?;

        // 2. Check if file already exists
        if self.file_exists(&hash)? {
            return Err(anyhow::anyhow!("File already exists in KB"));
        }

        // 3. Copy file to KB directory
        let dest = self.kb_root.join("files").join(&hash);
        fs::create_dir_all(&dest)?;

        let filename = source.file_name()
            .context("Invalid filename")?;
        let dest_file = dest.join(filename);

        fs::copy(source, &dest_file)?;

        // 4. Get file metadata
        let metadata = fs::metadata(&dest_file)?;
        let mime_type = mime_guess::from_path(source)
            .first()
            .map(|m| m.to_string());

        Ok(FileInfo {
            hash,
            original_name: filename.to_string_lossy().to_string(),
            file_path: dest_file,
            file_size: metadata.len(),
            mime_type,
        })
    }

    fn calculate_hash(&self, path: &Path) -> Result<String> {
        let content = fs::read(path)?;
        let hash = Sha256::digest(&content);
        Ok(format!("{:x}", hash))
    }

    fn file_exists(&self, hash: &str) -> Result<bool> {
        let path = self.kb_root.join("files").join(hash);
        Ok(path.exists())
    }
}

pub struct FileInfo {
    pub hash: String,
    pub original_name: String,
    pub file_path: PathBuf,
    pub file_size: u64,
    pub mime_type: Option<String>,
}
```

### Quota Check

```rust
// src/kb/quota.rs

use anyhow::Result;

#[derive(Debug, Clone)]
pub enum PlanTier {
    Starter,
    Pro,
    Max,
}

impl PlanTier {
    pub fn max_files(&self) -> Option<usize> {
        match self {
            PlanTier::Starter => Some(5),
            PlanTier::Pro => Some(15),
            PlanTier::Max => None, // Unlimited
        }
    }

    pub fn max_file_size_bytes(&self) -> u64 {
        match self {
            PlanTier::Starter => 10 * 1024 * 1024,  // 10 MB
            PlanTier::Pro => 50 * 1024 * 1024,      // 50 MB
            PlanTier::Max => 100 * 1024 * 1024,     // 100 MB
        }
    }
}

pub struct QuotaManager {
    db: Database,
}

impl QuotaManager {
    pub fn can_add_file(&self, plan: &PlanTier, file_size: u64) -> Result<bool> {
        let current_count = self.db.get_file_count()?;

        // Check file count limit
        if let Some(max_files) = plan.max_files() {
            if current_count >= max_files {
                return Ok(false);
            }
        }

        // Check file size limit
        if file_size > plan.max_file_size_bytes() {
            return Ok(false);
        }

        Ok(true)
    }

    pub fn get_quota_info(&self, plan: &PlanTier) -> Result<QuotaInfo> {
        let file_count = self.db.get_file_count()?;
        let total_size = self.db.get_total_size()?;

        Ok(QuotaInfo {
            plan: plan.clone(),
            current_files: file_count,
            max_files: plan.max_files(),
            total_size,
        })
    }
}

pub struct QuotaInfo {
    pub plan: PlanTier,
    pub current_files: usize,
    pub max_files: Option<usize>,
    pub total_size: u64,
}
```

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add_file() {
        // Test file addition
    }

    #[test]
    fn test_quota_limits() {
        // Test quota enforcement
    }

    #[test]
    fn test_search() {
        // Test search functionality
    }
}
```

### Integration Tests

```bash
# Test complete workflow
snailer kb init --plan starter
snailer kb add test-file.md
snailer kb list
snailer kb search "test"
snailer kb remove <id>
```

## Future Enhancements (Post-MVP)

1. **Cloud Sync** (Max plan)
   - S3/R2 integration
   - Multi-device sync
   - Version history

2. **Advanced Search**
   - Semantic search with embeddings
   - Cross-file references
   - Auto-tagging with AI

3. **Collaborative Features**
   - Team knowledge bases
   - Shared files
   - Access control

4. **Smart Features**
   - Auto-categorization
   - Duplicate detection
   - Content summarization

## Success Metrics

- [ ] Users can add/remove files successfully
- [ ] Search returns relevant results in < 100ms
- [ ] Plan limits are enforced correctly
- [ ] Agent can successfully use KB files
- [ ] 90%+ test coverage for core functions

## Timeline

| Week | Phase | Deliverable |
|------|-------|-------------|
| 1 | Core Infrastructure | Basic add/list/remove commands working |
| 2 | Indexing & Search | Full-text search operational |
| 3 | Interactive UI | Polished interactive commands |
| 4 | Agent Integration | KB integrated with agent workflow |

## Dependencies

```toml
[dependencies]
rusqlite = { version = "0.31", features = ["bundled"] }
sha2 = "0.10"
mime_guess = "2.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
anyhow = "1.0"
thiserror = "1.0"
dialoguer = "0.11"
indicatif = "0.17"
uuid = { version = "1.6", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
```

## Questions to Resolve

1. **File Type Support**: Which file types should we support initially?
   - Priority: MD, TXT, JSON, YAML, PDF
   - Future: DOCX, HTML, Code files

2. **Search Algorithm**: Should we use simple FTS5 or add semantic search?
   - Start with FTS5 (simpler, faster)
   - Add embeddings later if needed

3. **Agent Integration**: How should agent discover relevant KB files?
   - Keyword matching from task description
   - User can manually reference with `@kb`
   - Auto-suggest based on file access patterns

## Conclusion

This plan provides a comprehensive roadmap for implementing the Knowledge Base feature in Snailer. The phased approach ensures we build a solid foundation before adding advanced features, while the modular architecture allows for easy extension in the future.
