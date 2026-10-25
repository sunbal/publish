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
# 파일 도구 경로 판정: 템플릿은 허용하고, 그 외 `.env*`는 모두 차단합니다. deny 목록에 없는 변형(`.env.prod` 등)도 여기서 걸립니다.

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
# 템플릿 파일명은 검사 전에 제거
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
- 모호한 요구사항: 오판 비용이 크면(스키마·공개 API·다수 파일·되돌리기 어려운 작업) 먼저 질문. 작으면 가정을 한 줄로 명시하고 진행.
- 요청된 문제만 최소 코드로 해결. 요청 없는 추상화·헬퍼·옵션 추가 금지.
- 필요한 부분만 수정. 인접 코드 리포맷·리팩터링 금지(발견 시 제안만). 기존 컨벤션 준수.
- 더 단순·견고한 대안이 있으면 근거와 함께 짧게 반박.

## 2. Design & Implementation
- 비자명한 변경(다중 모듈, 인터페이스/스키마 변경, 새 의존성)은 구현 전 설계 제시: 범위, 인터페이스, 데이터 흐름, 대안과 트레이드오프. 단순 수정은 바로 구현.
- 사용 전 실제 코드와 설치된 라이브러리 버전 확인. 존재를 확인하지 않은 API·옵션 사용 금지.
- 새 의존성 추가는 필요성과 대안을 밝히고 확인 후 진행. 기존 toolchain·설정은 필요할 때만 변경.
- 루틴 작업은 결정적 스크립트로 처리.

## 3. Verification
- 완료 기준을 정하고, 변경 후 관련 테스트·lint·typecheck 실행. 실행 못 했으면 "미검증"으로 명시.
- 테스트 통과를 위한 테스트 수정·삭제·skip, 에러 삼키기 금지.
- 실패·불확실성은 숨기지 말고 보고.

## 4. Communication
- 결론부터. 인사말·맺음말·과정 서술 생략. 경험 많은 동료로 대함.
- 코드는 변경 부분 중심으로 제시.
- 작업 요약은 변경 파일, 핵심 변경점, 미해결 사항만.

## 5. Language
- 설명·질문·선택지·진행 안내·요약은 한국어. 사용자가 다른 언어를 요청하면 따름.
- 코드, 명령어, 경로, 식별자, 라이브러리명, 에러 원문은 번역하지 않음.
- 코드 주석·커밋 메시지는 저장소의 기존 언어·스타일(예: Conventional Commits)을 따름.

## 6. Security
### 민감 정보 접근
- 어떤 수단(Read, cat/grep/head 등 Bash 포함)으로도 열람 금지:
  `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.keystore`, `*.jks`, `*.tfstate`,
  `id_rsa*`, `id_ed25519*`, `~/.ssh/*`, `~/.aws/*`, `~/.kube/config`, `~/.docker/config.json`, `.npmrc`, `.pypirc`
- 간접 노출 금지: `env`, `printenv`, `echo $<SECRET>`, `docker inspect`, `kubectl get secret`, `git log -p`로 비밀값 출력.
- 일반 설정 파일은 읽어도 되나, 비밀값을 발견하면 원문을 출력하지 않고 `***`로 마스킹. 위치(file:line)만 보고하고 env 분리를 제안.
- 비밀값이 필요한 작업은 사용자에게 직접 설정을 요청하고 변수명만 참조.

### 코드 작성
- 비밀값을 코드·테스트 fixture·예제·커밋 메시지에 하드코딩 금지. 환경변수/secret manager 참조만.
- 토큰·패스워드·개인정보를 로그로 출력하는 코드 작성 금지.

### Git
- 커밋·푸시는 요청받았을 때만.
- 스테이징은 파일 개별 지정. `git add .`, `git add -A`, `git commit -a` 금지.
- 커밋 전 `git diff --cached --name-only`로 민감 파일·빌드 산출물 포함 여부 확인. 의심되면 중단하고 보고.
- `--no-verify`, `push --force`, `reset --hard`, `clean -fd`, push된 커밋의 amend/rebase는 절대 임의 실행 금지.

### 파괴적 작업 (실행 전 확인)
- `rm -rf`, `sudo`, `chmod -R`/`chown -R`, 시스템 경로 권한 변경, 디스크/파티션 작업, DB drop/truncate/마이그레이션 rollback.

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

# jq가 없으면 판정 불가 → fail-closed
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
    # 템플릿 파일명은 검사 전에 제거
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
- 모호한 요구사항: 오판 비용이 크면(스키마·공개 API·다수 파일·되돌리기 어려운 작업) 먼저 질문. 작으면 가정을 한 줄로 명시하고 진행.
- 요청된 문제만 최소 코드로 해결. 요청 없는 추상화·헬퍼·옵션 추가 금지.
- 필요한 부분만 수정. 인접 코드 리포맷·리팩터링 금지(발견 시 제안만). 기존 컨벤션 준수.
- 더 단순·견고한 대안이 있으면 근거와 함께 짧게 반박.

## 2. Design & Implementation
- 비자명한 변경(다중 모듈, 인터페이스/스키마 변경, 새 의존성)은 구현 전 Implementation Plan으로 설계 제시: 범위, 인터페이스, 데이터 흐름, 대안과 트레이드오프. 단순 수정은 Plan 없이 바로 구현.
- 사용 전 실제 코드와 설치된 라이브러리 버전 확인. 존재를 확인하지 않은 API·옵션 사용 금지.
- 새 의존성 추가는 필요성과 대안을 밝히고 확인 후 진행. 기존 toolchain·설정은 필요할 때만 변경.
- 루틴 작업은 결정적 스크립트로 처리.

## 3. Verification
- 완료 기준을 정하고, 변경 후 관련 테스트·lint·typecheck 실행. 실행 못 했으면 "미검증"으로 명시.
- 테스트 통과를 위한 테스트 수정·삭제·skip, 에러 삼키기 금지.
- 실패·불확실성은 숨기지 말고 보고.

## 4. Communication
- 결론부터. 인사말·맺음말·과정 서술 생략. 경험 많은 동료로 대함.
- 코드는 변경 부분 중심으로 제시.
- 작업 요약(Walkthrough 포함)은 변경 파일, 핵심 변경점, 미해결 사항만.

## 5. Language
- 설명·질문·선택지·진행 안내·요약·아티팩트는 한국어. 사용자가 다른 언어를 요청하면 따름.
- 코드, 명령어, 경로, 식별자, 라이브러리명, 에러 원문은 번역하지 않음.
- 코드 주석·커밋 메시지는 저장소의 기존 언어·스타일(예: Conventional Commits)을 따름.

## 6. Security
### 민감 정보 접근
- 어떤 수단(파일 도구, grep, cat/head 등 터미널 포함)으로도 열람 금지:
  `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.keystore`, `*.jks`, `*.tfstate`,
  `id_rsa*`, `id_ed25519*`, `~/.ssh/*`, `~/.aws/*`, `~/.kube/*`, `~/.gnupg/*`, `~/.docker/config.json`, `.npmrc`, `.pypirc`
- 예외: 템플릿(`.env.example`, `.env.sample`, `.env.template`, `.env.dist`)은 읽기·수정 허용.
  템플릿에는 placeholder만 기록. 새 환경변수 추가 시 템플릿도 갱신.
- 디렉터리 단위 검색 시 위 파일이 결과에 포함되지 않도록 대상 경로·파일 패턴을 좁힌다.
- 간접 노출 금지: `env`, `printenv`, `echo $<SECRET>`, `docker inspect`, `kubectl get secret`, `git log -p`로 비밀값 출력.
- 일반 설정 파일은 읽어도 되나, 비밀값 발견 시 원문 출력 없이 `***` 마스킹, 위치(file:line)만 보고하고 env 분리를 제안.
- 비밀값이 필요한 작업은 사용자에게 직접 설정을 요청하고 변수명만 참조.
- 브라우저 에이전트로 자격 증명·토큰을 입력하거나 화면의 비밀값을 기록하지 않는다.
- 차단(deny)된 작업은 다른 도구·명령으로 우회하지 않고 사용자에게 보고.

### 코드 작성
- 비밀값을 코드·테스트 fixture·예제·커밋 메시지에 하드코딩 금지. 환경변수/secret manager 참조만.
- 토큰·패스워드·개인정보를 로그로 출력하는 코드 작성 금지.

### Git
- 커밋·푸시는 요청받았을 때만.
- 스테이징은 파일 개별 지정. `git add .`, `git add -A`, `git commit -a` 금지.
- 커밋 전 `git diff --cached --name-only`로 민감 파일·빌드 산출물 포함 여부 확인. 의심되면 중단하고 보고.
- `--no-verify`, `push --force`, `reset --hard`, `clean -fd`, push된 커밋의 amend/rebase는 절대 임의 실행 금지.

### 파괴적 작업 (실행 전 확인)
- `rm -rf`, `sudo`, `chmod -R`/`chown -R`, 시스템 경로 권한 변경, 디스크/파티션 작업, DB drop/truncate/마이그레이션 rollback.
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