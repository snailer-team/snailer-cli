# Snailer Tool System

> 🔧 **목적**: Snailer Agent가 사용하는 도구 시스템의 구조와 확장 방법을 설명합니다.

## 📚 목차

1. [개요](#개요)
2. [도구 구조](#도구-구조)
3. [내장 도구 목록](#내장-도구-목록)
4. [도구 실행 흐름](#도구-실행-흐름)
5. [새로운 도구 추가하기](#새로운-도구-추가하기)
6. [모범 사례](#모범-사례)

---

## 개요

Snailer의 도구 시스템은 AI 에이전트가 **파일 시스템, Git, 셸 명령** 등과 상호작용할 수 있게 해주는 핵심 컴포넌트입니다.

### 핵심 원칙

1. **선언적 정의**: JSON Schema로 도구의 입력을 명확하게 정의
2. **타입 안전성**: Rust의 타입 시스템을 활용한 안전한 실행
3. **에러 핸들링**: 모든 도구는 Result<String>을 반환하여 에러 전파
4. **AI 친화적**: AI가 이해하기 쉬운 설명과 명확한 파라미터

---

## 도구 구조

### 1. Tool 정의

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tool {
    pub name: String,           // 도구 이름 (예: "read_file")
    pub description: String,    // 도구 설명 (AI가 읽음)
    pub input_schema: Value,    // JSON Schema 형식의 입력 스키마
}
```

### 2. ToolUse (도구 호출)

AI가 도구를 호출할 때 사용하는 구조:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolUse {
    pub id: String,             // 고유 ID (AI가 생성)
    pub tool_type: String,      // "tool_use" (고정값)
    pub name: String,           // 도구 이름
    pub input: Value,           // 입력 파라미터 (JSON)
}
```

### 3. ToolResult (도구 결과)

도구 실행 후 AI에게 반환하는 구조:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub tool_use_id: String,    // ToolUse의 ID와 매칭
    pub result_type: String,    // "tool_result" (고정값)
    pub content: String,        // 결과 내용
    pub is_error: Option<bool>, // 에러 여부 (true면 에러)
    pub name: String,           // 도구 이름 (일부 모델 API에서 필수)
}
```

### 4. ToolRegistry

모든 도구를 관리하는 레지스트리:

```rust
pub struct ToolRegistry {
    pub project_path: PathBuf,  // 작업 디렉토리
    tools: Vec<Tool>,           // 사용 가능한 도구 목록
}

impl ToolRegistry {
    // 새 레지스트리 생성
    pub fn new(project_path: PathBuf) -> Self

    // 사용 가능한 도구 목록 반환
    pub fn get_tools(&self) -> Vec<Tool>

    // 도구 실행
    pub fn execute_tool(&self, tool_use: &ToolUse) -> ToolResult
}
```

---

## 내장 도구 목록

이 문서는 **공개 레포** 기준의 개념 설명이며, 실제 구현은 private repo의 `src/tools.rs`가 권위 있는 소스입니다.

### 🔍 Discovery Tools (탐색)

#### read_file

파일 내용을 읽습니다. (선택적으로 라인 범위)

```json
{ "path": "src/main.rs", "start": 10, "end": 50 }
```

#### view_file

파일 내용을 “보기 전용”으로 읽습니다. `str_replace`용 정확한 스니펫 복사에 유용합니다.

```json
{ "path": "src/main.rs", "start": 10, "end": 50 }
```

#### search_repo

ripgrep으로 코드 검색합니다. (`.gitignore` 존중)

```json
{ "query": "async fn", "file_pattern": "*.rs" }
```

#### find_files

파일명 패턴으로 파일을 찾습니다.

```json
{ "pattern": "*.toml" }
```

#### vscode_open_file (GUI-only)

VS Code 같은 GUI 에디터가 연결된 환경에서 파일을 열도록 “요청”합니다. (daemon 클라이언트가 처리)

```json
{ "path": "src/main.rs", "selection": { "startLine": 10, "endLine": 50 } }
```

---

### ✏️ Editing Tools (편집)

#### edit_file

`old_text`를 `new_text`로 교체합니다. (작은 변경에 안전/신뢰성 높음)

```json
{
  "file_path": "src/main.rs",
  "old_text": "println!(\"Hello\");",
  "new_text": "println!(\"Hello, World!\");"
}
```

#### str_replace

파일에서 `old`의 **첫 번째** 등장만 `new`로 교체합니다. (`view_file`로 복사한 스니펫을 사용 권장)

```json
{ "path": "src/main.rs", "old": "foo()", "new": "bar()" }
```

#### write_file

새 파일 생성 또는 기존 파일 덮어쓰기입니다.

```json
{ "path": "src/new_module.rs", "content": "pub fn hello() {}" }
```

#### delete_file

파일을 삭제합니다.

```json
{ "path": "temp_file.txt" }
```

---

### 🖥️ Command Execution Tools (명령 실행)

#### bash_run (recommended)

빌드/테스트/린트 같은 커맨드를 실행합니다. **전체 로그는 파일로 저장**하고, 응답에는 요약만 포함해 컨텍스트 폭발을 줄입니다.

```json
{ "cmd": ["cargo", "test"], "timeout_sec": 120 }
```

#### bash_log

이전 `bash_run`의 전체 로그를 ID로 조회합니다. (가능하면 `tail`/`head`로 범위를 제한)

```json
{ "cmd_id": "bash_20250107_143022_a1b2c3d4", "stream": "stderr", "tail": 200 }
```

#### bash_history

최근 실행한 bash 커맨드 히스토리를 요약합니다.

```json
{ "last_n": 10 }
```

#### run_cmd (deprecated)

이전 방식의 커맨드 실행 도구입니다. 신규 작업에서는 `bash_run`을 사용하세요.

```json
{ "cmd": "npm test", "timeout_sec": 120, "detached": false }
```

---

### 🧠 Project Memory / Skills

#### read_notes / write_notes

프로젝트 루트의 `NOTES.md`를 통해 “지속 메모”를 읽고/씁니다.

```json
{ "section": "Architecture" }
```

```json
{ "section": "Decisions", "content": "We chose X because Y.", "append": true }
```

#### use_skill

사전 정의된 “스킬(워크플로우)”을 실행합니다.

```json
{ "skill_id": "skill-installer", "reason": "Need to install a curated skill", "inputs": { "user_request": "install skill X" } }
```

#### start_appgen_wizard

요청이 짧거나 모호할 때, TUI로 간단한 다지선다 질문을 통해 요구사항을 명확히 합니다.

```json
{ "reason": "Need clarification", "flow_hint": "focus on auth and data model" }
```

#### disable_repair_guard

Repair Guard가 정당한 작업을 막는 경우, 근거와 함께 강제로 비활성화합니다.

```json
{ "reason": "Need to edit files outside allowed set for this fix." }
```

## 도구 실행 흐름

### 전체 흐름

```
┌──────────────────────────────────────────────────────────────┐
│ 1. AI가 Tool Use 생성                                         │
│    {                                                          │
│      "type": "tool_use",                                      │
│      "id": "toolu_123",                                       │
│      "name": "read_file",                                     │
│      "input": {"path": "src/main.rs"}                         │
│    }                                                          │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. ToolRegistry::execute_tool() 호출                          │
│    - 도구 이름으로 라우팅                                      │
│    - match tool_use.name { ... }                              │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. 개별 도구 함수 실행                                         │
│    fn tool_read_file(&self, input: &Value) -> Result<String> │
│    {                                                          │
│      // 1. 파라미터 파싱                                       │
│      let path = input["path"].as_str()?;                      │
│                                                               │
│      // 2. 검증                                               │
│      if !path.exists() {                                      │
│        return Err(anyhow!("File not found"));                 │
│      }                                                        │
│                                                               │
│      // 3. 실행                                               │
│      let content = fs::read_to_string(path)?;                 │
│                                                               │
│      // 4. 결과 반환                                          │
│      Ok(content)                                              │
│    }                                                          │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. ToolResult 생성                                            │
│    {                                                          │
│      "type": "tool_result",                                   │
│      "tool_use_id": "toolu_123",                              │
│      "content": "fn main() { ... }",                          │
│      "is_error": false                                        │
│    }                                                          │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. AI에게 결과 반환                                           │
│    - conversation_history에 추가                              │
│    - AI가 결과를 보고 다음 행동 결정                           │
└──────────────────────────────────────────────────────────────┘
```

### 에러 처리

```rust
pub fn execute_tool(&self, tool_use: &ToolUse) -> ToolResult {
    let result_content = match tool_use.name.as_str() {
        "read_file" => self.tool_read_file(&tool_use.input),
        "write_file" => self.tool_write_file(&tool_use.input),
        _ => Err(anyhow!("Unknown tool: {}", tool_use.name)),
    };

    match result_content {
        Ok(content) => ToolResult {
            tool_use_id: tool_use.id.clone(),
            result_type: "tool_result".to_string(),
            content,
            is_error: None,  // 성공
        },
        Err(e) => ToolResult {
            tool_use_id: tool_use.id.clone(),
            result_type: "tool_result".to_string(),
            content: format!("Error: {}", e),
            is_error: Some(true),  // 에러 표시
        },
    }
}
```

**특징**:
- 모든 에러를 캐치하여 AI에게 전달
- AI가 에러를 보고 재시도 또는 대안 선택
- 시스템 크래시 방지

---

## 새로운 도구 추가하기

### Step 1: 도구 정의 추가

`define_tools()` 함수에 새 도구 추가:

```rust
fn define_tools() -> Vec<Tool> {
    vec![
        // ... 기존 도구들 ...

        // 새 도구: HTTP 요청
        Tool {
            name: "http_request".to_string(),
            description: "Make HTTP GET/POST request to external API".to_string(),
            input_schema: json!({
                "type": "object",
                "required": ["url"],
                "properties": {
                    "url": {
                        "type": "string",
                        "description": "The URL to request"
                    },
                    "method": {
                        "type": "string",
                        "description": "HTTP method (GET/POST)",
                        "enum": ["GET", "POST"]
                    },
                    "body": {
                        "type": "string",
                        "description": "Request body (for POST)"
                    }
                }
            }),
        },
    ]
}
```

### Step 2: 도구 구현

개별 도구 함수 구현:

```rust
fn tool_http_request(&self, input: &Value) -> Result<String> {
    // 1. 파라미터 파싱
    let url = input["url"]
        .as_str()
        .ok_or_else(|| anyhow!("Missing 'url' parameter"))?;

    let method = input["method"]
        .as_str()
        .unwrap_or("GET");

    // 2. 검증
    if !url.starts_with("http://") && !url.starts_with("https://") {
        return Err(anyhow!("Invalid URL scheme"));
    }

    // 3. 실행
    let response = match method {
        "GET" => reqwest::blocking::get(url)?,
        "POST" => {
            let body = input["body"].as_str().unwrap_or("");
            reqwest::blocking::Client::new()
                .post(url)
                .body(body.to_string())
                .send()?
        }
        _ => return Err(anyhow!("Unsupported method: {}", method)),
    };

    // 4. 결과 반환
    let status = response.status();
    let body = response.text()?;

    Ok(format!("Status: {}\n\n{}", status, body))
}
```

### Step 3: 라우팅 추가

`execute_tool()` 함수에 라우팅 추가:

```rust
pub fn execute_tool(&self, tool_use: &ToolUse) -> ToolResult {
    let result_content = match tool_use.name.as_str() {
        "read_file" => self.tool_read_file(&tool_use.input),
        "write_file" => self.tool_write_file(&tool_use.input),
        "http_request" => self.tool_http_request(&tool_use.input),  // ← 추가
        _ => Err(anyhow!("Unknown tool: {}", tool_use.name)),
    };

    // ... 에러 처리 ...
}
```

### Step 4: 테스트 작성

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_http_request_tool() {
        let registry = ToolRegistry::new(PathBuf::from("."));

        let tool_use = ToolUse {
            id: "test_123".to_string(),
            tool_type: "tool_use".to_string(),
            name: "http_request".to_string(),
            input: json!({
                "url": "https://api.github.com",
                "method": "GET"
            }),
        };

        let result = registry.execute_tool(&tool_use);
        assert!(result.is_error.is_none());
        assert!(result.content.contains("Status:"));
    }
}
```

---

## 모범 사례

### ✅ Do's (권장)

1. **명확한 설명 작성**
```rust
Tool {
    name: "search_repo".to_string(),
    description: "Search for files by name pattern. Use glob patterns like *.rs or Config*.json".to_string(),
    // ↑ AI가 언제 사용할지 명확히 알 수 있음
}
```

2. **타입 검증**
```rust
fn tool_example(&self, input: &Value) -> Result<String> {
    // ✅ Good: 명시적 에러 메시지
    let path = input["path"]
        .as_str()
        .ok_or_else(|| anyhow!("Missing required parameter 'path'"))?;

    // ❌ Bad: panic 발생 가능
    // let path = input["path"].as_str().unwrap();
}
```

3. **경로 검증**
```rust
fn tool_read_file(&self, input: &Value) -> Result<String> {
    let path = input["path"].as_str()?;
    let full_path = self.project_path.join(path);

    // ✅ 프로젝트 경로 밖으로 벗어나지 않는지 검증
    if !full_path.starts_with(&self.project_path) {
        return Err(anyhow!("Path escapes project directory"));
    }

    fs::read_to_string(full_path)
}
```

4. **타임아웃 설정**
```rust
fn tool_bash_run(&self, input: &Value) -> Result<String> {
    let timeout_sec = input["timeout_sec"]
        .as_u64()
        .unwrap_or(120);  // ✅ 기본값 120초

    // 타임아웃 적용 로직
}
```

### ❌ Don'ts (비권장)

1. **무제한 리소스 사용**
```rust
// ❌ Bad: 파일 크기 제한 없음
fn tool_read_file(&self, input: &Value) -> Result<String> {
    let content = fs::read_to_string(path)?;  // 100GB 파일도 읽음
    Ok(content)
}

// ✅ Good: 크기 제한
fn tool_read_file(&self, input: &Value) -> Result<String> {
    let metadata = fs::metadata(path)?;
    if metadata.len() > 10_000_000 {  // 10MB 제한
        return Err(anyhow!("File too large"));
    }
    let content = fs::read_to_string(path)?;
    Ok(content)
}
```

2. **민감한 정보 노출**
```rust
// ❌ Bad: 에러에 민감한 정보 포함
Err(anyhow!("Failed to read /home/user/.env: {}", e))

// ✅ Good: 일반적인 에러 메시지
Err(anyhow!("Failed to read file: permission denied"))
```

3. **위험한 명령 필터링 없음**
```rust
// ❌ Bad: 모든 명령 허용
fn tool_run_cmd(&self, input: &Value) -> Result<String> {
    let cmd = input["cmd"].as_str()?;
    Command::new("sh").arg("-c").arg(cmd).output()?;
}

// ✅ Good: 위험한 명령 차단
fn tool_run_cmd(&self, input: &Value) -> Result<String> {
    let cmd = input["cmd"].as_str()?;

    // 위험한 명령 차단
    if cmd.contains("rm -rf /") {
        return Err(anyhow!("Dangerous command blocked"));
    }

    Command::new("sh").arg("-c").arg(cmd).output()?;
}
```

---

## 성능 최적화

### 1. 결과 크기 제한

```rust
fn tool_search_repo(&self, input: &Value) -> Result<String> {
    let output = Command::new("rg")
        .args(["--max-count", "100"])  // ✅ 최대 100개 결과
        .output()?;

    Ok(String::from_utf8_lossy(&output.stdout).to_string())
}
```

### 2. 캐싱

```rust
use std::collections::HashMap;
use std::sync::Mutex;

lazy_static! {
    static ref FILE_CACHE: Mutex<HashMap<String, String>> = Mutex::new(HashMap::new());
}

fn tool_read_file(&self, input: &Value) -> Result<String> {
    let path = input["path"].as_str()?;

    // 캐시 확인
    if let Some(cached) = FILE_CACHE.lock().unwrap().get(path) {
        return Ok(cached.clone());
    }

    // 파일 읽기
    let content = fs::read_to_string(path)?;

    // 캐시 저장
    FILE_CACHE.lock().unwrap().insert(path.to_string(), content.clone());

    Ok(content)
}
```

---

## 보안 고려사항

### 1. 경로 탐색 공격 방지

```rust
fn validate_path(&self, path: &str) -> Result<PathBuf> {
    let full_path = self.project_path.join(path).canonicalize()?;

    if !full_path.starts_with(&self.project_path) {
        return Err(anyhow!("Path traversal detected"));
    }

    Ok(full_path)
}
```

### 2. 명령 인젝션 방지

```rust
fn tool_run_cmd(&self, input: &Value) -> Result<String> {
    let cmd = input["cmd"].as_str()?;

    // 허용된 명령만 실행
    let allowed_commands = ["cargo", "npm", "git"];
    let bin = cmd.split_whitespace().next().unwrap_or("");

    if !allowed_commands.contains(&bin) {
        return Err(anyhow!("Command not allowed: {}", bin));
    }

    Command::new("sh").arg("-c").arg(cmd).output()?;
}
```

### 3. 리소스 제한

```rust
use std::time::Duration;

fn tool_with_timeout(&self, input: &Value) -> Result<String> {
    let timeout = Duration::from_secs(30);

    let result = std::panic::catch_unwind(|| {
        // 실제 작업
    });

    result.map_err(|_| anyhow!("Operation timed out"))?
}
```

---

## 디버깅

### 로깅 추가

```rust
fn tool_read_file(&self, input: &Value) -> Result<String> {
    let path = input["path"].as_str()?;

    log::debug!("Reading file: {}", path);

    let content = fs::read_to_string(path)?;

    log::debug!("File read successfully: {} bytes", content.len());

    Ok(content)
}
```

### 실행 시간 측정

```rust
use std::time::Instant;

fn tool_search_repo(&self, input: &Value) -> Result<String> {
    let start = Instant::now();

    let result = /* ... 실제 작업 ... */;

    log::info!("Search completed in {:?}", start.elapsed());

    result
}
```

---

## 다음 단계

- [에이전트 아키텍처](./AGENT_ARCHITECTURE.md)
- [ACE 컨텍스트 관리](./ACE_SYSTEM.md)
- [기여 가이드](./CONTRIBUTING.md)

---

## 라이선스

MIT License
