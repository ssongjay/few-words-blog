+++
title = "Sandbox 환경? (1) 개념편"
date = 2026-04-03T18:10:00+09:00
draft = false
+++

## 시작

개발을 하던중 Claude Code에서 자꾸 백테스트를 부정확하고 대충 검증하는 것 같았다 그래서 Codex(ChatGPT)한테 돌려보게 시키라 명령했다 
Claude Code to Codex: "백테스트 돌려봐"라고 시켰더니 어디선가 막혔다.

알고 보니 기본값이 `read-only`라서 curl 호출 자체가 차단돼있었다.

`danger-full-access`로 바꾸면 샌드박스 안에서 curl로 로컬 서버 호출도 되고 DB 조회도 된다.
근데 여기서 codex 옵션중에 danger-full-access와 다른 --dangerously-bypass-approvals-and-sandbox라는게 있다그랬다.
후자 옵션은 codex 실행할 때 봤던 옵션인데 항상 sandbox라는게 권한 옵션에 있어서 궁금해졌다.

AI 에이전트한테 "분석만 해"가 아니라 "실행해봐"를 시키려면 샌드박스 레벨을 올려야 한다.
AI 코딩 에이전트(Codex, Claude Code 등)한테 "이거 실행해봐"라고 시켰는데
"권한이 없어서 못 합니다" 하고 돌아온 적 있으면, 그게 sandbox 때문이라고 하더라


## 샌드박스가 뭔데

### 사전적 의미

Sandbox = Sand(모래) + Box(상자).
어린이 놀이터에 있는 그 모래밭이다.

아이들이 모래밭 안에서 뭘 만들든 부수든, 밖에 있는 진짜 세상에는 아무 영향이 없다.
이 비유가 그대로 컴퓨터 용어로 넘어왔다.

참고로 영어권에서는 1570년대부터 쓰인 단어라고 한다. 처음에는 "모래를 뿌리는 도구"라는 뜻이었고, 1680년대에 "모래를 담는 상자"로 의미가 바뀌었다고.
*(출처: [Etymonline](https://www.etymonline.com/word/sandbox))*

### 개발/보안에서의 의미

컴퓨팅에서 샌드박스는 **프로그램을 격리된 환경에서 실행하는 보안 메커니즘**이다.

생각보다 오래된 개념인데,
정확한 시점은 불분명하지만 적어도 1990년대에는 보안 격리 메커니즘으로 개념이 잡혔다.

핵심은 간단하다.
- 프로그램에게 **시스템 자원의 일부만** 제한적으로 줌
- 그 안에서 뭘 하든 **밖의 실제 시스템에는 영향 없음**
- 문제가 생기면 샌드박스를 날리면 끝

가장 익숙한 예시가 크롬 브라우저인데,
탭 하나하나가 사실 샌드박스 안에서 돌아가고 있다.
어떤 웹사이트가 악성 코드를 실행해도 다른 탭이나 내 컴퓨터까지 영향을 못 미치는 이유가 이거다.
*(출처: [Sandbox (computer security) - Wikipedia](https://en.wikipedia.org/wiki/Sandbox_(computer_security)))*

### AI 에이전트에서

AI 코딩 에이전트가 내 컴퓨터에서 명령어를 실행할 때,
진짜 내 시스템에서 직접 돌리면 위험하다.
AI가 실수로 `rm -rf /` 같은 걸 날려도 막을 방법이 없으니까.

그래서 **격리된 가상 환경(컨테이너)** 안에서 실행하게 한다.
내 프로젝트 폴더를 복사해서 가상 환경에 넣고, 거기서만 놀게 하는 거다.

- **샌드박스 ON** = 놀이터 모래밭 안에서만 놀아라. 밖에 나가지 마.
- **샌드박스 OFF** = 집 전체에서 자유롭게 해.


## 실제 구현: OS별로 다르다

샌드박스라는 개념은 같은데, 실제로 격리하는 방법은 OS마다 다르다.

| OS | 기술 | 설명 |
|----|------|------|
| macOS | **Seatbelt** | macOS 내장 샌드박스 프레임워크. 별도 설치 없이 바로 동작한다. |
| Linux / WSL2 | **Bubblewrap (bwrap)** | 파일시스템 네임스페이스 격리 + Seccomp 시스콜 필터링. `sudo apt install bubblewrap` 필요. |
| Windows | **Restricted Token** | Windows 네이티브 보안 토큰 기반 격리. PowerShell에서 동작. |

Codex도 Claude Code도 둘 다 이 OS 네이티브 기술을 쓴다.
macOS면 Seatbelt, Linux면 Bubblewrap. 직접 Docker 컨테이너를 띄우는 게 아니라 OS 레벨에서 프로세스 자체를 격리하는 방식이다.
*(출처: [Sandboxing – Codex | OpenAI](https://developers.openai.com/codex/concepts/sandboxing), [Sandboxing - Claude Code Docs](https://code.claude.com/docs/en/sandboxing))*

---

## 정리

여기까지가 샌드박스 자체가 뭔지에 대한 얘기다.
다음 글에서는 Codex와 Claude Code가 이 샌드박스를 실제로 어떻게 다루는지, 옵션은 어떻게 나뉘는지 따로 정리해보려 한다.

---

## 출처

- [sandbox - Etymonline](https://www.etymonline.com/word/sandbox) — sandbox 단어의 어원
- [Sandbox (computer security) - Wikipedia](https://en.wikipedia.org/wiki/Sandbox_(computer_security)) — 샌드박스 개념 일반
- [Sandboxing – Codex | OpenAI Developers](https://developers.openai.com/codex/concepts/sandboxing) — Codex 샌드박스 개요와 OS별 동작 설명
- [Sandboxing - Claude Code Docs](https://code.claude.com/docs/en/sandboxing) — Claude Code 샌드박스 개요와 OS별 동작 설명
