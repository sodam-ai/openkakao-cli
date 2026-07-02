# AX Hardening (v1.5.0) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden `local-send`/`ax-read` (introduced in v1.4.0) without adding new user-facing features: make chat-matching testable, stop silently dropping non-text messages from `ax-read`, and turn "Accessibility permission not granted" from a confusing generic error into a clear one.

**Architecture:** All changes live in `src/ax_send.rs`. One new pure function (`match_chat_row`) moves out of the `#[cfg(target_os = "macos")] mod imp { ... }` block to file scope so it compiles and is unit-tested on every platform, including the Linux runner that release CI's `verify` job actually uses. The other two changes (message-type placeholders, permission check) stay inside `mod imp` since they call real AX/`accessibility-sys` APIs.

**Tech Stack:** Rust, `accessibility`/`accessibility-sys` crates (macOS-only, already a dependency), existing `anyhow::Result` error convention.

## Global Constraints

- Every new function that does NOT touch `AXUIElement`/`accessibility_sys` types must live outside `#[cfg(target_os = "macos")] mod imp { ... }` in `src/ax_send.rs`, so it compiles and is tested under the Linux `verify` CI job (confirmed to run on `ubuntu-latest` — see `docs/superpowers/specs/2026-07-02-ax-hardening-design.md` § 배경). Every new function that DOES touch those types must live inside `mod imp`.
- Chat-name matching stays exact-match (`==`), no case-folding or whitespace trimming — same behavior as today, just relocated.
- No new public CLI flags, no new config fields, no new `AxMessage` fields — the message-type change reuses the existing `text: String` field with a placeholder value.
- Every step that touches code must leave `cargo build --release --bin openkakao-cli`, `cargo clippy --all-targets -- -D warnings`, `cargo fmt --check`, and `cargo test` passing on macOS before moving to the next step.
- Manual QA against real KakaoTalk (self/memo chat only, per the existing project safety convention) is required before the final commit — this cannot be automated because there's no AX tree mock.

---

## Task 1: Extract `match_chat_row` as a pure, cross-platform function with unit tests

**Files:**
- Modify: `src/ax_send.rs` (add before line 23's `#[cfg(target_os = "macos")]`, and rewire `open_chat_row` at line 180)

**Interfaces:**
- Produces: `enum ChatMatch { Found(usize), NotFound, Ambiguous(usize) }` and `fn match_chat_row(row_names: &[Option<String>], target: &str) -> ChatMatch`, both at `ax_send` module scope (not inside `imp`), both `pub(crate)` (only `imp::open_chat_row` calls them, but they need to be visible from inside `mod imp`).

- [ ] **Step 1: Write the failing tests**

Add this new `#[cfg(test)] mod match_tests` block at the very top of `src/ax_send.rs`, immediately after the module doc comment (after line 21, before line 23's `#[cfg(target_os = "macos")]`). This references `ChatMatch`/`match_chat_row`, which don't exist yet, so it won't compile — that's step 2's job to confirm.

```rust
#[derive(Debug, PartialEq, Eq)]
pub(crate) enum ChatMatch {
    Found(usize),
    NotFound,
    Ambiguous(usize),
}

pub(crate) fn match_chat_row(row_names: &[Option<String>], target: &str) -> ChatMatch {
    todo!()
}

#[cfg(test)]
mod match_tests {
    use super::*;

    #[test]
    fn empty_list_is_not_found() {
        assert_eq!(match_chat_row(&[], "Alice"), ChatMatch::NotFound);
    }

    #[test]
    fn single_exact_match_is_found() {
        let names = [Some("Alice".to_string())];
        assert_eq!(match_chat_row(&names, "Alice"), ChatMatch::Found(0));
    }

    #[test]
    fn substring_does_not_match() {
        // "Alice" must not match a group chat named "Alice & Bob".
        let names = [Some("Alice & Bob".to_string())];
        assert_eq!(match_chat_row(&names, "Alice"), ChatMatch::NotFound);
    }

    #[test]
    fn exact_match_among_non_matching_rows() {
        let names = [
            Some("Alice & Bob".to_string()),
            Some("Alice".to_string()),
            Some("Carol".to_string()),
        ];
        assert_eq!(match_chat_row(&names, "Alice"), ChatMatch::Found(1));
    }

    #[test]
    fn duplicate_names_are_ambiguous() {
        let names = [Some("Alice".to_string()), Some("Alice".to_string())];
        assert_eq!(match_chat_row(&names, "Alice"), ChatMatch::Ambiguous(2));
    }

    #[test]
    fn unreadable_rows_are_ignored_not_matched() {
        // A row whose name AX couldn't read (None) must never match, and
        // must not affect matching of the other rows.
        let names = [None, Some("Alice".to_string()), None];
        assert_eq!(match_chat_row(&names, "Alice"), ChatMatch::Found(1));
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cargo test --lib match_tests 2>&1 | tail -20`
Expected: compile error, since `match_chat_row` is `todo!()` — actually it will compile (todo! panics at runtime, not a compile error) and the tests will FAIL with `not yet implemented`. Confirm you see 6 failing tests, not a compile error and not passes.

- [ ] **Step 3: Implement `match_chat_row`**

Replace the `todo!()` body from Step 1 with:

```rust
pub(crate) fn match_chat_row(row_names: &[Option<String>], target: &str) -> ChatMatch {
    let mut matches = row_names
        .iter()
        .enumerate()
        .filter(|(_, name)| name.as_deref() == Some(target));

    match (matches.next(), matches.next()) {
        (None, _) => ChatMatch::NotFound,
        (Some((idx, _)), None) => ChatMatch::Found(idx),
        (Some(_), Some(_)) => {
            let count = row_names
                .iter()
                .filter(|name| name.as_deref() == Some(target))
                .count();
            ChatMatch::Ambiguous(count)
        }
    }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cargo test --lib match_tests 2>&1 | tail -20`
Expected: `test result: ok. 6 passed; 0 failed`

- [ ] **Step 5: Rewire `open_chat_row` to use `match_chat_row`**

In `src/ax_send.rs`, inside `mod imp`, find `fn open_chat_row` (currently around line 180-234). Replace the row-collection-and-matching block (currently lines 189-219: from `let mut rows = Vec::new();` through the `let row = match matches.as_slice() { ... };` block) with:

```rust
        let mut rows = Vec::new();
        find_descendants_by_role(table, "AXRow", &mut rows);

        // Match on the row's first AXStaticText (the chat name) exactly, not
        // a substring — e.g. "Alice" must not accidentally match an
        // "Alice & Bob" group chat. If more than one row has the exact same
        // display name, refuse to guess rather than silently picking one
        // (there is no chat-id to disambiguate with — see
        // SafetyConfig::allowed_send_chats). The matching decision itself is
        // `super::match_chat_row`, a pure function tested outside this
        // macOS-only module.
        let row_names: Vec<Option<String>> = rows
            .iter()
            .map(|row| {
                let mut texts = Vec::new();
                find_descendants_by_role(row, "AXStaticText", &mut texts);
                texts.first().and_then(value_as_string)
            })
            .collect();

        let row = match super::match_chat_row(&row_names, chat_display_name) {
            super::ChatMatch::NotFound => {
                return Err(anyhow!(
                    "chat '{chat_display_name}' not found in visible/loaded chat list"
                ))
            }
            super::ChatMatch::Found(idx) => &rows[idx],
            super::ChatMatch::Ambiguous(count) => {
                return Err(anyhow!(
                    "chat name '{chat_display_name}' matches {count} chats in the visible list — ambiguous, refusing to guess"
                ))
            }
        };
```

This is a straight behavioral no-op refactor — same exact-match logic, same error messages, same ambiguity handling, just delegating the decision to the tested pure function.

- [ ] **Step 6: Run the full local verification suite**

Run: `cargo build --release --bin openkakao-cli 2>&1 | tail -20 && cargo clippy --all-targets -- -D warnings 2>&1 | tail -20 && cargo fmt --check 2>&1 | tail -20 && cargo test 2>&1 | tail -20`
Expected: all four commands succeed with no errors/warnings, including the 6 new `match_tests` cases and the pre-existing 3 tests in `imp::tests`.

- [ ] **Step 7: Manual QA — real chat send still works**

Run against the real KakaoTalk app, targeting only the self/memo chat already configured in `~/.config/openkakao/config.toml`'s `safety.allowed_send_chats` (do not target any other chat):

```bash
./target/release/openkakao-cli local-send "<your allowlisted chat name>" "AX hardening task 1 QA" -y --json
```

Expected: `{"chat_name": "...", "status": "sent"}` and the message is visible in KakaoTalk. This confirms the refactor didn't change real-world matching behavior.

- [ ] **Step 8: Commit**

```bash
git add src/ax_send.rs
git commit -m "$(cat <<'EOF'
refactor(ax_send): extract chat-matching into a pure, tested function

match_chat_row now lives outside the macOS-only imp module, so it
compiles and is unit-tested on any platform -- including the Linux
runner release CI's verify job actually uses. Pure behavioral
no-op: open_chat_row's exact-match / ambiguity-refusal logic is
unchanged, just delegated to a testable function instead of being
inlined in the AX-tree-walking code.
EOF
)"
```

---

## Task 2: `ax-read` shows placeholders for non-text messages instead of dropping them

**Files:**
- Modify: `src/ax_send.rs`, `fn read_visible_messages` (inside `mod imp`, currently lines 291-315)

**Interfaces:**
- Consumes: `find_descendants_by_role` (existing, Task 1 doesn't change its signature), `value_as_string`, `attr_as_string` (existing).
- Produces: no signature change to `read_visible_messages(window: &AXUIElement) -> Vec<AxMessage>` or `AxMessage` — same shape, `text` field now sometimes holds a placeholder string instead of the row being dropped entirely.

- [ ] **Step 1: Replace `read_visible_messages`'s row-to-message logic**

In `src/ax_send.rs` inside `mod imp`, replace the current `read_visible_messages` function body (the `rows.iter().filter_map(...)` closure) with a version that falls back to image/file detection before giving up on a row:

```rust
    /// Scrape every message bubble currently rendered in a chat window's
    /// message list, in on-screen (chronological) order. A row with an
    /// `AXTextArea` is a text message; a row with no `AXTextArea` but an
    /// `AXImage` descendant becomes the placeholder "[사진]"; a row with a
    /// share-labeled `AXButton` ("공유") becomes "[파일]". Rows matching none
    /// of these (date separators, system notices) are skipped, same as
    /// before.
    fn read_visible_messages(window: &AXUIElement) -> Vec<AxMessage> {
        let mut tables = Vec::new();
        find_descendants_by_role(window, "AXTable", &mut tables);
        let Some(table) = tables.first() else {
            return Vec::new();
        };
        let mut rows = Vec::new();
        find_descendants_by_role(table, "AXRow", &mut rows);

        rows.iter()
            .filter_map(|row| {
                let text = message_row_text(row)?;

                let mut static_texts = Vec::new();
                find_descendants_by_role(row, "AXStaticText", &mut static_texts);
                let time = static_texts
                    .first()
                    .and_then(|t| attr_as_string(t, "AXHelp").or_else(|| value_as_string(t)));

                Some(AxMessage { time, text })
            })
            .collect()
    }

    /// Classify one message row into displayable text: the row's own
    /// `AXTextArea` value if present, else a placeholder if the row looks
    /// like an image or file share, else `None` (not a real message row).
    fn message_row_text(row: &AXUIElement) -> Option<String> {
        let mut text_areas = Vec::new();
        find_descendants_by_role(row, "AXTextArea", &mut text_areas);
        if let Some(text) = text_areas.first().and_then(value_as_string) {
            return Some(text);
        }

        let mut images = Vec::new();
        find_descendants_by_role(row, "AXImage", &mut images);
        if !images.is_empty() {
            return Some("[사진]".to_string());
        }

        let mut buttons = Vec::new();
        find_descendants_by_role(row, "AXButton", &mut buttons);
        if buttons
            .iter()
            .any(|b| attr_as_string(b, "AXDescription").as_deref() == Some("공유"))
        {
            return Some("[파일]".to_string());
        }

        None
    }
```

- [ ] **Step 2: Run the full local verification suite**

Run: `cargo build --release --bin openkakao-cli 2>&1 | tail -20 && cargo clippy --all-targets -- -D warnings 2>&1 | tail -20 && cargo fmt --check 2>&1 | tail -20 && cargo test 2>&1 | tail -20`
Expected: all pass. No test changes needed here — `message_row_text` calls real `AXUIElement` methods, so it lives inside `mod imp` and (per the codebase's existing convention for AX-tree logic, documented in the spec's 테스트 계획 section) is verified manually, not with `#[cfg(test)]` unit tests.

- [ ] **Step 3: Manual QA — send yourself a photo and confirm the placeholder appears**

Using the KakaoTalk app directly (not the CLI), send a photo to your own allowlisted memo chat. Then run:

```bash
./target/release/openkakao-cli ax-read "<your allowlisted chat name>" -n 5 --json
```

Expected: the JSON `messages` array includes an entry with `"text": "[사진]"` for the photo you just sent, in the correct chronological position relative to any surrounding text messages (i.e. it's no longer silently missing).

- [ ] **Step 4: Commit**

```bash
git add src/ax_send.rs
git commit -m "$(cat <<'EOF'
feat(ax-read): show [사진]/[파일] placeholders instead of dropping non-text messages

read_visible_messages previously skipped any row without an
AXTextArea, silently deleting photos/files from ax-read's output and
leaving gaps in the conversation order. message_row_text now falls
back to detecting an AXImage descendant ("[사진]") or a share-labeled
AXButton ("[파일]") before giving up on a row. No AxMessage schema
change -- placeholders reuse the existing text field.
EOF
)"
```

---

## Task 3: Detect missing Accessibility permission before attempting AX calls

**Files:**
- Modify: `src/ax_send.rs`, `mod imp` (add new function; call it from `read_via_ax` and `send_via_ax`)

**Interfaces:**
- Consumes: `accessibility_sys::AXIsProcessTrusted` (already available — the crate is already a dependency; confirmed present in `accessibility-sys 0.2.0`'s `src/ui_element.rs`).
- Produces: `fn ensure_ax_permission() -> Result<()>` inside `mod imp`, called at the top of both `pub fn read_via_ax` and `pub fn send_via_ax`, right after `find_kakaotalk_pid()?`.

- [ ] **Step 1: Add the `AXIsProcessTrusted` import**

In `src/ax_send.rs`, inside `mod imp`'s `use` block (currently lines 26-37), add:

```rust
    use accessibility_sys::AXIsProcessTrusted;
```

- [ ] **Step 2: Add `ensure_ax_permission`**

Add this function inside `mod imp`, directly after `find_kakaotalk_pid` (after line 64's closing `}`):

```rust
    /// Check the calling process has been granted Accessibility permission
    /// before touching the AX tree at all. Without this, every AXUIElement
    /// call below just silently fails or returns empty results, which
    /// previously surfaced as a confusing "chat not found" error with no
    /// hint that the real cause was a missing permission grant.
    fn ensure_ax_permission() -> Result<()> {
        if unsafe { AXIsProcessTrusted() } {
            Ok(())
        } else {
            Err(anyhow!(
                "Accessibility permission is not granted to this terminal app.\n\
                 Open System Settings → Privacy & Security → Accessibility,\n\
                 and enable it for your terminal (Terminal.app, iTerm2, etc.),\n\
                 then re-run this command."
            ))
        }
    }
```

- [ ] **Step 3: Call it from both entry points**

In `pub fn read_via_ax` (currently starting line 347), change:

```rust
    pub fn read_via_ax(chat_display_name: &str, count: usize) -> Result<Vec<AxMessage>> {
        let pid = find_kakaotalk_pid()?;
        let app = AXUIElement::application(pid);
```

to:

```rust
    pub fn read_via_ax(chat_display_name: &str, count: usize) -> Result<Vec<AxMessage>> {
        let pid = find_kakaotalk_pid()?;
        ensure_ax_permission()?;
        let app = AXUIElement::application(pid);
```

In `pub fn send_via_ax` (currently starting line 381), make the identical change:

```rust
    pub fn send_via_ax(chat_display_name: &str, message: &str) -> Result<()> {
        let pid = find_kakaotalk_pid()?;
        ensure_ax_permission()?;
        let app = AXUIElement::application(pid);
```

- [ ] **Step 4: Run the full local verification suite**

Run: `cargo build --release --bin openkakao-cli 2>&1 | tail -20 && cargo clippy --all-targets -- -D warnings 2>&1 | tail -20 && cargo fmt --check 2>&1 | tail -20 && cargo test 2>&1 | tail -20`
Expected: all pass. `unsafe { AXIsProcessTrusted() }` must not trigger a clippy `unsafe` lint under `-D warnings` — if it does, wrap it in `#[allow(unused_unsafe)]` is NOT the fix; instead confirm the function is genuinely `unsafe fn` in `accessibility-sys` (it is, per the `extern "C"` block) so the `unsafe {}` block is required and correct, and clippy's default lint set does not flag correctly-scoped `unsafe` blocks by default.

- [ ] **Step 5: Manual QA — confirm the error message when permission is revoked**

On the development Mac: open System Settings → Privacy & Security → Accessibility, and turn OFF the toggle for your terminal app. Then run:

```bash
./target/release/openkakao-cli ax-read "<your allowlisted chat name>" -n 5
```

Expected: the command fails immediately with the new "Accessibility permission is not granted..." message (not a "chat not found" error, not a hang). Then turn the toggle back ON in System Settings, and re-run the same command to confirm it works normally again:

```bash
./target/release/openkakao-cli ax-read "<your allowlisted chat name>" -n 5
```

Expected: normal message output, no error.

- [ ] **Step 6: Commit**

```bash
git add src/ax_send.rs
git commit -m "$(cat <<'EOF'
feat(ax_send): detect missing Accessibility permission up front

Without Accessibility permission, every AXUIElement call silently
returns empty/failed results, which previously surfaced through
local-send/ax-read as a confusing "chat not found" error -- no hint
that the real cause was a missing permission grant. ensure_ax_permission
checks AXIsProcessTrusted() right after finding KakaoTalk's pid, in
both read_via_ax and send_via_ax, and fails fast with instructions to
enable the terminal app in System Settings.
EOF
)"
```

---

## Task 4: Release v1.5.0

**Files:**
- Modify: `Cargo.toml` (version), `Cargo.lock` (via rebuild), `CHANGELOG.md`

**Interfaces:**
- Consumes: nothing new — this task packages Tasks 1-3's already-committed, already-tested changes into a release, following the exact same process used for v1.4.0-v1.4.4 earlier in this project's history.

- [ ] **Step 1: Bump the version**

In `Cargo.toml`, change:

```toml
version = "1.4.4"
```

to:

```toml
version = "1.5.0"
```

- [ ] **Step 2: Rebuild to sync `Cargo.lock`**

Run: `cargo build --release --bin openkakao-cli 2>&1 | tail -5`
Expected: `Compiling openkakao-cli v1.5.0 (...)`, `Finished release profile`. Confirm `Cargo.lock` changed: `git diff --stat Cargo.lock` should show 1 changed line.

- [ ] **Step 3: Add the CHANGELOG entry**

In `CHANGELOG.md`, insert this new section directly after the `## [Unreleased]` line and before the next `## [1.4.4]` entry:

```markdown
## [1.5.0] - <today's date, YYYY-MM-DD>

### Changed
- `local-send`/`ax-read`'s chat-matching logic (exact-match, ambiguity refusal) is now a pure, unit-tested function (`match_chat_row` in `src/ax_send.rs`), verified on every platform release CI runs on rather than only informally on a macOS dev machine.

### Added
- `ax-read` no longer silently drops photo/file messages from its output — rows with no text but a detected image or file-share now appear as `"[사진]"`/`"[파일]"` placeholders instead of leaving a gap in the conversation order.
- `local-send`/`ax-read` now detect a missing Accessibility permission grant up front and fail with a clear "enable it in System Settings → Privacy & Security → Accessibility" message, instead of a confusing "chat not found" error.
```

- [ ] **Step 4: Run the full local verification suite one more time**

Run: `cargo build --release --bin openkakao-cli 2>&1 | tail -10 && cargo clippy --all-targets -- -D warnings 2>&1 | tail -10 && cargo fmt --check 2>&1 | tail -10 && cargo test 2>&1 | tail -10`
Expected: all pass.

- [ ] **Step 5: Commit on a branch and open a PR**

```bash
git checkout -b release/v1.5.0-ax-hardening
git add Cargo.toml Cargo.lock CHANGELOG.md
git commit -m "$(cat <<'EOF'
chore: release v1.5.0

Claude-Session: https://claude.ai/code/session_01DCmATCnDVgwRM3RuVzqN5d
EOF
)"
git push -u origin release/v1.5.0-ax-hardening
gh pr create --title "release: v1.5.0 (AX hardening)" --body "$(cat <<'EOF'
## Summary
- Chat-matching logic extracted into a pure, cross-platform-tested function (match_chat_row).
- ax-read shows [사진]/[파일] placeholders instead of silently dropping non-text messages.
- local-send/ax-read detect a missing Accessibility permission grant and fail with a clear message instead of a confusing "chat not found" error.
- See docs/superpowers/specs/2026-07-02-ax-hardening-design.md for the full design.

## Test plan
- [x] cargo build/clippy/fmt/test all pass
- [x] Manual QA: real send to the allowlisted self/memo chat still works after the matching refactor
- [x] Manual QA: sending a photo to the self/memo chat and confirming ax-read shows "[사진]" in the right position
- [x] Manual QA: revoking and re-granting Accessibility permission and confirming the new error message / normal recovery

Claude-Session: https://claude.ai/code/session_01DCmATCnDVgwRM3RuVzqN5d
EOF
)"
```

- [ ] **Step 6: Merge, sync main, tag, and confirm the release workflow succeeds**

```bash
gh pr merge --squash --delete-branch=true
git checkout main
git pull origin main
git tag -a v1.5.0 -m "v1.5.0"
git push origin v1.5.0
```

Then watch the release workflow (mirroring how v1.4.0-v1.4.4 were verified earlier in this project):

```bash
gh run list --workflow=openkakao-cli-release.yml --limit 1 --json databaseId,status
# substitute the printed databaseId below
gh run watch <databaseId> --exit-status
```

Expected: workflow conclusion `success`. Then confirm the release actually published (not just the workflow succeeding — v1.4.0-v1.4.2 taught this project that a green run and a published release aren't the same fact to assume):

```bash
gh release list --limit 3
```

Expected: `v1.5.0` appears as `Latest`. If the workflow fails, do NOT re-tag `v1.5.0` again — delete the tag (`git push --delete origin v1.5.0 && git tag -d v1.5.0`), fix the root cause on `main` in a new small commit, bump to the next patch version, and re-tag with the new version (this project's established pattern from the v1.4.x hotfix chain).

---

## Self-Review Notes

- **Spec coverage:** §1 (pure matching function) → Task 1. §2 (message-type placeholders) → Task 2. §3 (permission detection) → Task 3. §4 (local-db unchanged) → correctly has no task, per spec's explicit "코드 변경 없음" instruction.
- **Placeholder scan:** no TBD/TODO markers remain outside Task 1 Step 1's intentional single `todo!()`, which Step 2 immediately exercises and Step 3 replaces.
- **Type consistency:** `ChatMatch`/`match_chat_row` signature is identical from Task 1 Step 1 (test-writing) through Step 3 (implementation) through Step 5 (call site). `AxMessage`'s shape is unchanged across Task 2 (still `{ time: Option<String>, text: String }`). `ensure_ax_permission() -> Result<()>` is called with `?` the same way at both Task 3 call sites.
