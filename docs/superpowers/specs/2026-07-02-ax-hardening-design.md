# AX 자동화 견고성 강화 (v1.5.0)

## 배경

v1.4.4에서 `local-send`/`ax-read`가 macOS Accessibility API 기반으로 처음 도입됐다. 서버 로그인이나 로컬 SQLCipher DB에 의존하지 않고 실제 메시지 전송·읽기를 지원하는 유일한 경로가 됐다. 이번 릴리즈는 신규 기능 추가가 아니라, 이 경로를 프로덕션에서 더 안정적으로 쓸 수 있게 만드는 데 집중한다.

v1.4.3/v1.4.4 릴리즈 과정에서 release CI의 `verify` 잡(`cargo fmt`/`clippy`/`test`)이 `ubuntu-latest`에서 돈다는 사실이 드러났다 — `accessibility`/`core-graphics` 같은 macOS 전용 크레이트는 `[target.'cfg(target_os = "macos")'.dependencies]`로 격리해야 하고, `ax_send.rs`도 `#[cfg(target_os = "macos")] mod imp { ... }` + `#[cfg(not(target_os = "macos"))]` 스텁 구조로 나뉘어 있다. 이번 작업에서 새 코드를 추가할 때도 이 경계를 지켜야 한다(§ 1 참고).

브레인스토밍에서 확정한 스코프:

1. 채팅 매칭 로직을 순수 함수로 분리하고 단위 테스트로 검증 가능하게 만든다.
2. `ax-read`가 텍스트가 아닌 메시지(사진/파일)를 조용히 건너뛰지 않고 플레이스홀더로 표시한다.
3. Accessibility 권한이 없을 때 모호한 에러 대신 명확한 안내를 낸다.
4. `local-chats`/`local-read`/`local-search`(로컬 DB 경로)는 현행 유지 — 코드 변경 없음, 이미 v1.4.0에서 경고 문서화를 마쳤으므로 이번 스코프에서는 손대지 않는다.

## 목표가 아닌 것

- `local-send`/`ax-read`에 새로운 사용자 대면 기능(스크롤 기반 히스토리 페이지네이션, 발신자 이름 추출, 재시도/타임아웃 정책 커스터마이징)을 추가하지 않는다 — 브레인스토밍에서 명시적으로 제외됨.
- 로컬 DB 키 유도 공식을 재조사하거나 고치지 않는다.
- 서버 로그인(LOCO/REST) 경로에는 손대지 않는다.

## 1. 채팅 매칭 로직 분리 + 단위 테스트

### 현재 구조

`src/ax_send.rs`의 `open_chat_row`가 AX 트리 순회(행 목록 얻기)와 매칭 판단(정확 일치 탐색, 중복 시 거부)을 한 함수 안에서 함께 수행한다. AX 호출 없이는 이 매칭 로직만 따로 검증할 방법이 없다.

### 변경

행 이름 목록을 받아 매칭 결과를 반환하는 순수 함수를 새로 뺀다:

```rust
enum ChatMatch {
    Found(usize),   // 유일하게 일치하는 행의 인덱스
    NotFound,
    Ambiguous(usize), // 일치하는 행 개수
}

fn match_chat_row(row_names: &[Option<String>], target: &str) -> ChatMatch
```

- `row_names`는 각 행의 (있다면) 이름 텍스트. `None`은 이름을 못 읽은 행(스킵 대상).
- 매칭은 정확히 일치(`==`)만 인정, 대소문자/공백 트리밍 없음 — 기존 동작과 동일하게 유지(카카오톡 채팅 이름은 사용자가 통제할 수 없는 값이므로 정규화하면 오히려 오탐 위험).
- `open_chat_row`는 AX로 각 행의 이름을 뽑아 `Vec<Option<String>>`을 만든 뒤 `match_chat_row`를 호출하고, 결과에 따라 기존과 동일한 에러 메시지를 만들거나 해당 인덱스의 실제 `AXUIElement` 행을 선택한다.
- **중요**: `ChatMatch`와 `match_chat_row`는 `accessibility`/`AXUIElement` 타입을 전혀 참조하지 않으므로, `#[cfg(target_os = "macos")]` `imp` 모듈 **바깥**(파일 최상단, cfg 게이트 없이)에 둔다. release CI의 `verify` 잡은 Linux 러너에서 돈다(v1.4.3/v1.4.4에서 확인된 사실) — 이 함수를 `imp` 안에 두면 Linux에서 아예 컴파일되지 않아 테스트가 실행되지 않는다. 순수 함수로 분리하는 목적 자체가 "AX 호출 없이도, 즉 어떤 플랫폼에서도 검증 가능하게" 만드는 것이므로 cfg 게이트 밖에 둬야 목적이 성립한다.

### 테스트 케이스

- 빈 목록 → `NotFound`
- 이름 하나만 있고 정확히 일치 → `Found(0)`
- 여러 행 중 하나만 정확히 일치(다른 행은 부분 문자열만 겹침, 예: `"Alice"` vs `"Alice & Bob"`) → `Found`가 해당 인덱스를 가리킴, 부분 일치 행은 무시됨
- 동일한 이름을 가진 행이 2개 → `Ambiguous(2)`
- 이름을 못 읽은 행(`None`)이 섞여 있어도 나머지 매칭에 영향 없음

## 2. ax-read 메시지 타입 구분

### 현재 동작

`read_visible_messages`가 각 행에서 `AXTextArea` 값을 찾고, 없으면 그 행 전체를 건너뛴다(`filter_map`에서 `?`로 조기 반환). 사진, 파일, 이모티콘처럼 텍스트가 없는 메시지는 결과에서 통째로 사라진다 — 대화 순서상 구멍이 생긴다.

### 변경

`AxMessage`에 필드를 하나 추가한다:

```rust
pub struct AxMessage {
    pub time: Option<String>,
    pub text: String,
}
```

`text`의 의미를 확장한다: `AXTextArea`가 있으면 그 값을 그대로 쓰고, 없으면 행 안의 다른 자식을 검사해서 최소 분류한 플레이스홀더 문자열을 채운다.

- 행에 `AXImage` 자손이 있으면 → `"[사진]"`
- 행에 `AXButton`이면서 `AXDescription`이 `"공유"`를 포함하면 → `"[파일]"`
- 위 조건에 해당하는 자식이 하나도 없으면(날짜 구분선, 시스템 알림 등 진짜 메시지가 아닌 행) → 기존처럼 결과에서 제외

우선순위: `AXTextArea` > `AXImage` > `AXButton(공유)` > 제외. (한 행에 여러 자식이 섞여 있을 가능성은 낮지만, 텍스트가 있으면 항상 텍스트를 우선한다.)

`AxMessage`에 별도의 `message_type` enum 필드를 추가하지 않고 `text`에 플레이스홀더를 넣는 이유: 현재 소비자(`ax_read.rs`의 사람이 읽는 출력과 JSON 출력)가 전부 `text` 하나만 사용하고 있고, 플레이스홀더 방식이 스키마 변경 없이 더 단순하다. 나중에 실제 구분이 더 필요해지면 그때 필드를 추가한다(YAGNI).

## 3. Accessibility 권한 미승인 감지

### 현재 동작

권한이 없는 상태에서 `AXUIElement::application(pid)`로 만든 핸들에 대한 모든 호출(`.windows()`, `.children()` 등)이 그냥 빈 값이나 에러를 반환한다. 그 결과 사용자에게는 "chat not found" 같은 엉뚱한 에러만 보인다 — 원인이 권한 문제라는 걸 알 방법이 없다.

### 변경

`imp` 모듈에 `accessibility-sys`의 `AXIsProcessTrusted()`를 감싼 체크를 추가하고, `find_kakaotalk_pid()` 성공 직후 — AX 트리를 처음 만지기 직전 — 호출한다:

```rust
fn ensure_ax_permission() -> Result<()> {
    if unsafe { accessibility_sys::AXIsProcessTrusted() } {
        Ok(())
    } else {
        Err(anyhow!(
            "Accessibility 권한이 없어 KakaoTalk을 조작할 수 없습니다.\n\
             시스템 설정 → 개인정보 보호 및 보안 → 손쉬운 사용에서\n\
             이 터미널 앱(Terminal/iTerm2/...)을 켜주세요."
        ))
    }
}
```

`send_via_ax`/`read_via_ax` 양쪽 진입점에서 `find_kakaotalk_pid()` 다음, `open_chat_row` 이전에 호출한다. 이미 권한이 있는 정상 케이스에서는 API 호출 하나 추가되는 것뿐이라 성능에 영향 없음.

## 영향받는 파일

- `src/ax_send.rs` — 세 변경 모두 여기 (imp 모듈 내부)
- `src/commands/ax_read.rs` — 없음 (AxMessage 필드 의미만 바뀌고 구조는 그대로라 이 파일은 무변경)
- `CHANGELOG.md`, `Cargo.toml` — 릴리즈 시 버전 bump

## 테스트 계획

- `match_chat_row` 순수 함수: 위에 나열한 5개 케이스를 `#[cfg(test)]`로 추가. cfg 게이트 밖에 위치하므로 macOS 개발 환경뿐 아니라 release CI의 Linux `verify` 잡에서도 실행된다 — v1.4.3/v1.4.4 사태(§ 배경 참고)로 이 크레이트의 `cargo test`/`clippy`가 Linux에서도 반드시 통과해야 한다는 게 이미 확인됐다.
- `ensure_ax_permission`: 실제 AX 승인 여부에 의존하므로 자동 테스트 불가 — 수동 QA(권한 끈 상태에서 `local-send` 실행해 에러 메시지 확인)
- 메시지 타입 플레이스홀더: 실제 카카오톡에서 사진/파일을 하나씩 보내고 `ax-read`로 확인하는 수동 QA. AX 트리 모킹이 없으므로 자동 테스트는 만들지 않는다(기존 컨벤션과 동일 — `ax_send.rs`의 다른 AX 관련 로직도 수동 QA로 검증해왔다).
