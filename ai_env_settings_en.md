---
일자: 2026-10-25
내용: llama.cpp, hermes, claude, gemini 환경정보
목적: AI service를 사용하기 위한 환경설정 값 정리
검증: 확인전
링크:
  - 관련된 문서의 링크 추가
---
### 1. 설정위치

| claude                                                                                    | antigravity                                                                                                  | hermes |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ------ |
| global 설정<br><br>- .claude/CLAUDE.md<br>- .claude/commands/{name}.md<br>- .claude/skills/ | global 설정<br>- .gemini/settings.json<br>- .gemini/instructions.md<br>- .gemini/config/skills/{name}/SKILL.md | N/A    |

	- 글로벌 정책을 LLM으로 정의: .claude/CLAUDE.md, .gemini/instructions.md
	- 명령어 추가: .claude/commands, .gemini/config/skills (gemini는 모두 skill로 만들어야 함)
	- settings를 이용한 강제 규칙 추가: .gemini/settings.json (or yaml)
	- google인 경우 설치된 package에 따라 .gemnini나 .antigravity를 사용해야 함.

## 2. claude 설정값 정리

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)", "Read(**/.env.local)", "Read(**/.env.*.local)",
      "Read(**/.env.development)", "Read(**/.env.production)",
      "Read(**/.env.staging)", "Read(**/.env.test)",
      "Read(**/*.pem)", "Read(**/*.key)", "Read(**/*.p12)", "Read(**/*.pfx)",
      "Read(**/*.keystore)", "Read(**/*.jks)", "Read(**/*.tfstate)",
      "Read(~/.ssh/**)", "Read(~/.aws/**)", "Read(~/.kube/**)", "Read(~/.docker/config.json)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash", "hooks": [{ "type": "command", "command": "~/.claude/hooks/guard.sh" }] },
      { "matcher": "Read|Edit|Write|Grep", "hooks": [{ "type": "command", "command": "~/.claude/hooks/file-guard.sh" }] }
    ]
  }
}
```


	
```bash

# .claude/hooks/file-guard.sh
#!/usr/bin/env bash
# File-tool path check: allow env templates, block every other .env* (incl. variants not in deny list, e.g. .env.prod).

#!/usr/bin/env bash
path=$(jq -r '.tool_input.file_path // .tool_input.path // ""')
name=${path##*/}

case "$name" in
  .env.example|.env.sample|.env.template|.env.dist|.env.*.example) exit 0 ;;
  .env|.env.*) echo "Blocked: secret env file ($name)" >&2; exit 2 ;;
esac
exit 0


# .claude/hooks/guard.sh
#!/usr/bin/env bash
cmd=$(jq -r '.tool_input.command // ""')
# Strip template filenames before matching
check=$(sed -E 's/\.env[.A-Za-z0-9_-]*\.(example|sample|template|dist)//g' <<<"$cmd")

pattern='(^|[ /"'\''])\.env|id_rsa|id_ed25519|\.pem|\.p12|printenv|^env( |$)|git push .*(-f|--force)|reset --hard|clean -[a-z]*f|git add (-A|--all|\.( |$))|commit -a|--no-verify'
if grep -Eq "$pattern" <<<"$check"; then
  echo "Blocked by guard hook: matched sensitive/destructive pattern" >&2
  exit 2
fi

```

```markdown
// .claude/CLAUDE.md

# Global Operating Rules

## 1. Working Style
- Ambiguity: if a wrong guess is costly (schema, public API, many files, irreversible), ask first. Otherwise state the assumption in one line and proceed.
- Solve only what was asked, with minimal code. No unrequested abstractions, helpers, or options.
- Touch only what is necessary. No reformatting/refactoring of adjacent code (suggest instead). Follow existing conventions.
- If a simpler, more robust alternative exists, push back briefly with rationale.

## 2. Design & Implementation
- Non-trivial changes (multi-module, interface/schema change, new dependency): present a design before coding — scope, interfaces, data flow, alternatives/trade-offs. Simple fixes: implement directly.
- Check actual code and installed library versions before use. Never use APIs/options not confirmed to exist.
- New dependency: state need and alternatives, get approval first. Change toolchain/config only when required.
- Use deterministic scripts for routine operations.

## 3. Verification
- Define done criteria. After changes, run relevant tests, lint, typecheck. If not run, say "unverified".
- Never modify, delete, or skip tests, or swallow errors, to make things pass.
- Report failures and uncertainty; never hide them.

## 4. Communication
- Lead with the answer. No greetings, sign-offs, or process narration. Treat me as an experienced peer.
- Show only changed code.
- Summaries: changed files, key changes, open issues only.

## 5. Language
- Write all explanations, questions, options, progress updates, and summaries in Korean unless I ask otherwise.
- Never translate code, commands, paths, identifiers, library names, or raw error messages.
- Code comments and commit messages follow the repo's existing language and style (e.g., Conventional Commits).

## 6. Security
### Secrets access
- Never read by any means (Read, or cat/grep/head via Bash):
  `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.keystore`, `*.jks`, `*.tfstate`,
  `id_rsa*`, `id_ed25519*`, `~/.ssh/*`, `~/.aws/*`, `~/.kube/*`, `~/.gnupg/*`, `~/.docker/config.json`, `.npmrc`, `.pypirc`
- Exception: templates (`.env.example`, `.env.sample`, `.env.template`, `.env.dist`) may be read and edited. Placeholders only; update them when adding env vars.
- No indirect exposure: never print secrets via `env`, `printenv`, `echo $<SECRET>`, `docker inspect`, `kubectl get secret`, `git log -p`.
- Config files may be read. If a secret is found, never print it: mask as `***`, report file:line only, suggest moving it to env.
- For tasks needing secrets, ask me to set them; reference variable names only.
- Never work around a denied action with another tool or command; report it.

### Code
- Never hardcode secrets in code, test fixtures, examples, or commit messages. Reference env vars/secret manager only.
- Never write code that logs tokens, passwords, or PII.

### Git
- Commit/push only when asked.
- Stage files explicitly. No `git add .`, `git add -A`, `git commit -a`.
- Before committing, check `git diff --cached --name-only` for secrets/build artifacts. If suspicious, stop and report.
- Never run `--no-verify`, `push --force`, `reset --hard`, `clean -fd`, or amend/rebase pushed commits unless explicitly asked.

### Destructive operations (confirm first)
- `rm -rf`, `sudo`, `chmod -R`/`chown -R`, system path permission changes, disk/partition ops, DB drop/truncate/migration rollback.

```


## 3. gemini (antigravity)

|항목|Claude Code|Antigravity|
|---|---|---|
|전역 규칙|`~/.claude/CLAUDE.md`|`~/.gemini/GEMINI.md`|
|hook 설정|`settings.json`의 `hooks`|`~/.gemini/config/hooks.json` (App·CLI·IDE 공통)|
|hook 차단 방식|exit 2 + stderr|stdout JSON `{"decision":"deny"}`|
|권한 규칙|glob 패턴 (`Read(**/.env)`)|경로 단위 매칭, glob 없음|

	 ~/.gemini/config/hooks/guard.sh
```bash

#!/usr/bin/env bash
# Antigravity PreToolUse guard: block secret files & destructive commands, allow env templates.
allow() { echo '{"decision":"allow"}'; exit 0; }
deny()  { jq -cn --arg r "$1" '{decision:"deny",reason:$r}'; exit 0; }

# No jq -> cannot evaluate -> fail-closed
command -v jq >/dev/null || { echo '{"decision":"deny","reason":"guard: jq not installed"}'; exit 0; }

input=$(cat)
tool=$(jq -r '.toolCall.name // ""' <<<"$input")

is_secret_path() {
  local name=${1##*/}
  case "$name" in
    .env.example|.env.sample|.env.template|.env.dist|.env.*.example) return 1 ;;
    .env|.env.*|*.pem|*.key|*.p12|*.pfx|*.keystore|*.jks|*.tfstate|id_rsa*|id_ed25519*) return 0 ;;
  esac
  case "$1" in
    */.ssh/*|*/.aws/*|*/.kube/*|*/.gnupg/*|*/.docker/config.json) return 0 ;;
  esac
  return 1
}

case "$tool" in
  view_file)
    p=$(jq -r '.toolCall.args.AbsolutePath // ""' <<<"$input") ;;
  write_to_file|replace_file_content|multi_replace_file_content)
    p=$(jq -r '.toolCall.args.TargetFile // ""' <<<"$input") ;;
  grep_search)
    p=$(jq -r '.toolCall.args.SearchPath // ""' <<<"$input") ;;
  run_command)
    cmd=$(jq -r '.toolCall.args.CommandLine // ""' <<<"$input")
    # Strip template filenames before matching
    check=$(sed -E 's/\.env[.A-Za-z0-9_-]*\.(example|sample|template|dist)//g' <<<"$cmd")
    pattern='(^|[ /"'\''])\.env|id_rsa|id_ed25519|\.pem|\.p12|\.ssh/|\.aws/|printenv|^env( |$)|git push.*( -f( |$)| --force)|reset --hard|clean -[a-z]*f|git add (-A|--all|\.( |$))|commit -a|--no-verify'
    grep -Eq "$pattern" <<<"$check" && deny "Blocked by secret-guard: sensitive/destructive command"
    allow ;;
  *) allow ;;
esac

[ -n "$p" ] && is_secret_path "$p" && deny "Blocked by secret-guard: secret file (${p##*/})"
allow
```

	~/.gemini/config/hooks.json
```bash
f=~/.gemini/config/hooks.json
[ -f "$f" ] || echo '{}' > "$f"
jq --arg cmd "$HOME/.gemini/config/hooks/guard.sh" '. + {
  "secret-guard": {
    "PreToolUse": [{
      "matcher": "view_file|write_to_file|replace_file_content|multi_replace_file_content|grep_search|run_command",
      "hooks": [{ "type": "command", "command": $cmd, "timeout": 10 }]
    }]
  }
}' "$f" > "$f.tmp" && mv "$f.tmp" "$f"
```

	~/.gemini/antigravity-cli/settings.json`의 `permissions`아래
```json
{
  "permissions": {
    "deny": [
      "read_file(/Users/YOU/.ssh)",
      "read_file(/Users/YOU/.aws)",
      "read_file(/Users/YOU/.kube)",
      "read_file(/Users/YOU/.gnupg)",
      "read_file(/Users/YOU/.docker/config.json)",
      "write_file(.git/)",
      "command(sudo)",
      "command(rm -rf)",
      "command(git push --force)",
      "command(git push -f)",
      "command(git reset --hard)",
      "command(git clean)",
      "command(printenv)"
    ]
  }
}
```

	- ~/.gemini/GEMINI.md
```markdown
# Global Operating Rules

## 1. Working Style
- Ambiguity: if a wrong guess is costly (schema, public API, many files, irreversible), ask first. Otherwise state the assumption in one line and proceed.
- Solve only what was asked, with minimal code. No unrequested abstractions, helpers, or options.
- Touch only what is necessary. No reformatting/refactoring of adjacent code (suggest instead). Follow existing conventions.
- If a simpler, more robust alternative exists, push back briefly with rationale.

## 2. Design & Implementation
- Non-trivial changes (multi-module, interface/schema change, new dependency): present an Implementation Plan before coding — scope, interfaces, data flow, alternatives/trade-offs. Simple fixes: implement directly, no Plan.
- Check actual code and installed library versions before use. Never use APIs/options not confirmed to exist.
- New dependency: state need and alternatives, get approval first. Change toolchain/config only when required.
- Use deterministic scripts for routine operations.

## 3. Verification
- Define done criteria. After changes, run relevant tests, lint, typecheck. If not run, say "unverified".
- Never modify, delete, or skip tests, or swallow errors, to make things pass.
- Report failures and uncertainty; never hide them.

## 4. Communication
- Lead with the answer. No greetings, sign-offs, or process narration. Treat me as an experienced peer.
- Show only changed code.
- Summaries (including Walkthrough): changed files, key changes, open issues only.

## 5. Language
- Write all explanations, questions, options, progress updates, summaries, and artifacts in Korean unless I ask otherwise.
- Never translate code, commands, paths, identifiers, library names, or raw error messages.
- Code comments and commit messages follow the repo's existing language and style (e.g., Conventional Commits).

## 6. Security
### Secrets access
- Never read by any means (file tools, grep, or cat/head in terminal):
  `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.keystore`, `*.jks`, `*.tfstate`,
  `id_rsa*`, `id_ed25519*`, `~/.ssh/*`, `~/.aws/*`, `~/.kube/*`, `~/.gnupg/*`, `~/.docker/config.json`, `.npmrc`, `.pypirc`
- Exception: templates (`.env.example`, `.env.sample`, `.env.template`, `.env.dist`) may be read and edited. Placeholders only; update them when adding env vars.
- For directory-wide searches, narrow the path/file pattern so the files above are excluded.
- No indirect exposure: never print secrets via `env`, `printenv`, `echo $<SECRET>`, `docker inspect`, `kubectl get secret`, `git log -p`.
- Config files may be read. If a secret is found, never print it: mask as `***`, report file:line only, suggest moving it to env.
- For tasks needing secrets, ask me to set them; reference variable names only.
- Never enter credentials/tokens via the browser agent or record secrets shown on screen.
- Never work around a denied action with another tool or command; report it.

### Code
- Never hardcode secrets in code, test fixtures, examples, or commit messages. Reference env vars/secret manager only.
- Never write code that logs tokens, passwords, or PII.

### Git
- Commit/push only when asked.
- Stage files explicitly. No `git add .`, `git add -A`, `git commit -a`.
- Before committing, check `git diff --cached --name-only` for secrets/build artifacts. If suspicious, stop and report.
- Never run `--no-verify`, `push --force`, `reset --hard`, `clean -fd`, or amend/rebase pushed commits unless explicitly asked.

### Destructive operations (confirm first)
- `rm -rf`, `sudo`, `chmod -R`/`chown -R`, system path permission changes, disk/partition ops, DB drop/truncate/migration rollback.
```


	- git
```bash
# 0) jq 확인 (macOS: brew install jq)
command -v jq

# 1) guard 스크립트: 위 1번 내용을 저장
mkdir -p ~/.gemini/config/hooks
$EDITOR ~/.gemini/config/hooks/guard.sh
chmod +x ~/.gemini/config/hooks/guard.sh

# 2) hooks.json 병합: 위 2번 명령 실행

# 3) GEMINI.md: 위 5번 내용을 ~/.gemini/GEMINI.md에 저장 (기존 파일이 있으면 뒤에 추가)

# 4) 권한 규칙: CLI는 ~/.gemini/antigravity-cli/settings.json에 병합, App/IDE는 Settings에서 추가

# 5) Antigravity 재시작
```