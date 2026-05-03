+++
title = "Sandbox 환경? (2) Codex와 Claude Code 옵션편"
date = 2026-04-03T18:20:00+09:00
draft = false
+++

이전 글에서 샌드박스가 뭔지, 왜 필요한지, OS 레벨에서는 어떻게 구현되는지 먼저 정리했다.
이번 글에서는 그 다음 단계로 들어가서 Codex와 Claude Code가 샌드박스를 실제로 어떤 옵션과 정책으로 다루는지 정리해본다.

## OpenAI Codex CLI 샌드박스

Codex는 **샌드박스**와 **승인 정책(approval policy)** 두 가지를 따로 관리한다.

### 샌드박스 모드 (`--sandbox`)

3단계다.

**1. `read-only` (기본값)**

```bash
codex exec --sandbox read-only "코드 분석해줘"
```

파일 읽기만 된다. 수정 불가, 네트워크 불가, 시스템 명령 실행 불가.

**2. `workspace-write`**

```bash
codex exec --sandbox workspace-write "이 버그 고쳐줘"
```

프로젝트 폴더 안에서 파일 읽기/쓰기 가능. 코드 수정, 새 파일 생성 가능.
근데 curl, mysql 같은 네트워크/시스템 명령은 여전히 차단된다.

`--full-auto`가 이 모드다. (`--sandbox workspace-write` + `--ask-for-approval on-request`)

**3. `danger-full-access`**

```bash
codex exec --sandbox danger-full-access "백테스트 돌려봐"
```

샌드박스 안에서 모든 게 된다. curl로 API 호출, mysql로 DB 조회, 네트워크 접속 등.
이름에 "danger"가 붙어있긴 한데 **격리는 유지**된다. 내 실제 시스템 파일에는 영향 없다.
*(출처: [Codex CLI Reference](https://developers.openai.com/codex/cli/reference), [Sandboxing – Codex](https://developers.openai.com/codex/concepts/sandboxing))*

### 승인 정책 (`--ask-for-approval`)

샌드박스와 별개로, 명령 실행 전에 사람한테 물어볼지를 정하는 옵션이다.

| 값 | 의미 |
|----|------|
| `untrusted` | 안전한 명령(ls, cat 등)만 자동 실행. 나머지는 물어봄 |
| `on-request` | 모델이 판단해서 필요할 때만 물어봄 |
| `never` | 절대 안 물어봄. 실행 실패하면 그냥 모델한테 에러 돌려줌 |

### 핵폭탄 옵션

```bash
codex exec --dangerously-bypass-approvals-and-sandbox "알아서 해"
```

**샌드박스 끔 + 승인도 안 물어봄.** 격리 없이 내 맥북에서 직접 실행된다.
AI가 파일 삭제하면 진짜 삭제됨.

공식 문서에 "EXTREMELY DANGEROUS"라고 적혀있다. 이미 외부에서 격리된 환경(CI/CD, Docker 안)에서만 쓰라고 만든 옵션이다.
*(출처: [Agent approvals & security – Codex](https://developers.openai.com/codex/agent-approvals-security))*

### config.toml로 기본값 세팅

매번 플래그 안 치고 싶으면 `~/.codex/config.toml`에 설정해두면 된다.

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"
```

조합으로 보면:
- `sandbox_mode = "workspace-write"` + `approval_policy = "on-request"` = `--full-auto`와 같다
- `sandbox_mode = "danger-full-access"` + `approval_policy = "never"` = Full access (핵폭탄)

추가로 특정 디렉토리에 쓰기 권한을 열고 싶으면:

```toml
[sandbox_workspace_write]
writable_roots = ["/tmp/build", "~/.kube"]
```
*(출처: [Config Reference – Codex](https://developers.openai.com/codex/config-reference))*


## Claude Code 샌드박스

Claude Code는 Codex랑 구조가 좀 다르다.

### 격리 범위

Claude Code의 샌드박스는 **Bash 도구에만** 적용된다.
Read, Edit, Write 같은 내장 파일 도구는 샌드박스가 아니라 퍼미션 시스템으로 관리된다.

샌드박스가 격리하는 건 두 가지다.
- **파일시스템**: 현재 프로젝트 디렉토리 안에서만 쓰기 가능. 밖은 읽기 전용.
- **네트워크**: 프록시 서버를 통해 허용된 도메인만 접근 가능. 새 도메인은 사용자에게 물어봄.

### 샌드박스 모드

`/sandbox` 명령으로 켜고, 두 가지 모드가 있다.

- **Auto-allow**: 샌드박스 안의 명령은 자동 승인. 밖으로 나가는 건 물어봄.
- **Regular permissions**: 샌드박스 안이든 밖이든 다 물어봄.

### 탈출구: dangerouslyDisableSandbox

명령이 샌드박스 제약 때문에 실패하면 (네트워크 연결 안 됨, 특정 도구 호환 안 됨 등),
Claude가 알아서 분석하고 `dangerouslyDisableSandbox` 파라미터를 붙여서 재시도할 수 있다.
이때는 일반 퍼미션 플로우를 타니까 사용자 승인이 필요하다.

이 탈출구 자체를 막고 싶으면:

```json
{
  "sandbox": {
    "allowUnsandboxedCommands": false
  }
}
```

이러면 `dangerouslyDisableSandbox`가 완전히 무시된다.

### settings.json 설정

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": ["~/.kube", "/tmp/build"],
      "denyRead": ["~/"],
      "allowRead": ["."]
    }
  }
}
```

- `allowWrite`: 프로젝트 밖에 쓰기 허용할 경로
- `denyRead`: 읽기 차단할 경로
- `allowRead`: denyRead 안에서 다시 읽기 허용할 경로

여러 설정 스코프(managed, user, project, local)에 흩어져 있으면 전부 **merge**된다. 덮어쓰기가 아니라 합쳐진다.
*(출처: [Sandboxing - Claude Code Docs](https://code.claude.com/docs/en/sandboxing), [Anthropic 엔지니어링 블로그](https://www.anthropic.com/engineering/claude-code-sandboxing))*


## Codex vs Claude Code 비교

| | Codex CLI | Claude Code |
|------|-----------|-------------|
| 샌드박스 단계 | 3단계 (read-only, workspace-write, danger-full-access) | 2모드 (auto-allow, regular) + 탈출구 |
| 승인 정책 | 별도 옵션 (`--ask-for-approval`) | 퍼미션 시스템과 통합 |
| 적용 범위 | 모든 명령 | Bash 도구만 (Read/Edit는 퍼미션) |
| 네트워크 격리 | 샌드박스 모드에 포함 | 프록시 기반 도메인 필터링 |
| 핵폭탄 | `--dangerously-bypass-approvals-and-sandbox` | `dangerouslyDisableSandbox: true` |
| 설정 파일 | `~/.codex/config.toml` | `settings.json` (여러 스코프) |
| OS 기술 | Seatbelt (macOS), Bubblewrap (Linux) | 동일 |


## 정리

| 옵션 | 파일 읽기 | 파일 쓰기 | 시스템 명령 | 네트워크 | 격리 |
|------|:---------:|:---------:|:-----------:|:--------:|:----:|
| Codex `read-only` | O | X | X | X | O |
| Codex `workspace-write` | O | O | X | X | O |
| Codex `danger-full-access` | O | O | O | O | O |
| Codex `dangerously-bypass...` | O | O | O | O | **X** |
| Claude Code sandbox ON | O | 프로젝트만 | 승인 필요 | 허용 도메인만 | O |
| Claude Code `dangerouslyDisableSandbox` | O | O | O | O | **X** |




---

## 출처

- [Sandboxing – Codex | OpenAI Developers](https://developers.openai.com/codex/concepts/sandboxing) — Codex의 샌드박스 모드 설명
- [Command line options – Codex CLI | OpenAI Developers](https://developers.openai.com/codex/cli/reference) — `--sandbox` 옵션과 CLI 사용법
- [Agent approvals & security – Codex | OpenAI Developers](https://developers.openai.com/codex/agent-approvals-security) — approval policy와 bypass 옵션 설명
- [Config Reference – Codex | OpenAI Developers](https://developers.openai.com/codex/config-reference) — `config.toml` 설정 레퍼런스
- [Sandboxing - Claude Code Docs](https://code.claude.com/docs/en/sandboxing) — Claude Code sandbox, settings, `dangerouslyDisableSandbox` 설명
- [Making Claude Code more secure and autonomous – Anthropic](https://www.anthropic.com/engineering/claude-code-sandboxing) — Claude Code 샌드박스 설계 배경
