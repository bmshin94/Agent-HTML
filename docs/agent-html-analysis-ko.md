# Agent-HTML 분석 정리 (한국어)

- 원본(업스트림) 저장소: https://github.com/Sayhi-bzb/Agent-HTML (조사 시점 2026-09 기준 ⭐ 538 · 포크 19)
- 내 포크 저장소: https://github.com/bmshin94/Agent-HTML
- npm 패키지: https://www.npmjs.com/package/agent-html (`agent-html` 0.3.0)
- 라이선스: Apache-2.0 (단, `_archive/apps/agent-html-app`은 독점 라이선스라 사용하면 안 돼)

---

## 1. 전수조사: 이게 뭐고, 언제 쓰고, 나한테 어떤 도움이 되나

### 한 줄 요약
> AI 에이전트(Codex, Claude Code 등)가 답변을 **마크다운 글** 대신 **React로 만든 인터랙티브 웹 화면(대시보드, 리포트, 칸반, 차트)** 으로 내놓게 해 주고,
> 그 화면을 **브라우저에서 보고 → 특정 부분을 클릭해서 → 그 부분만 AI에게 고쳐 달라고** 할 수 있게 해 주는 로컬 작업 공간(Canvas)이야.
> 슬로건은 "You don't need a chat UI but a canvas with AI".

### 핵심 개념
| 개념 | 설명 |
| --- | --- |
| Artifact(아티팩트) | AI가 만든 결과물 한 편. `agent-html/artifacts/*.artifact.tsx` |
| Block(블록) | 아티팩트 안의 구역(예: "PR 개요", "리스크 맵"). id가 있어서 콕 집어 수정 요청 가능 |
| Canvas Host | `npx agent-html dev`로 뜨는 로컬 웹 뷰어. 아티팩트 목록, 미리보기, 검증 오류 표시, 블록 hover → 프롬프트 입력창, 테마 적용 |
| Validate | `agent-html validate`: AI가 규칙을 어겼는지 자동 검사(금지 import, 인라인 스타일 등). 오류 = 무조건 고쳐야 함 |
| Infinite Canvas | `Canvas`/`Node` JSX로 Figma처럼 무한 캔버스 위에 React 카드를 배치 (React Flow 기반) |
| Theme Preset | claude-plus, hex, manga, mimi, pixel-quest, whatsapp 6종 테마 |
| AHTML Desktop | Tauri(Rust)로 만든 데스크톱 앱. 프로젝트 선택 + Canvas 실행 관리 |

### 폴더 구조 (약 1,342개 파일, 코드 약 12.5만 줄)
```
Agent-HTML/
├── agent-html/            ← ⭐ AI가 작업하는 "작업실" (npx agent-html init 하면 이게 생김)
│   ├── AGENTS.md          AI가 반드시 지킬 규칙
│   ├── TASTE.md           디자인 감각 가이드
│   ├── artifacts/         예제 6개: code-review-room, health-report-decoder, linux-do-community-case,
│   │                      nasa-artemis-ii, nyc-taxi-sketchbook, tokyo-three-speeds
│   ├── canvases/          무한 캔버스 예제(operations.canvas.tsx) + .layout(좌표 자동 저장)
│   ├── components/ui/     shadcn/ui 부품 36종 (button, card, table, tabs, dialog…)
│   ├── components/chart/  차트 10종 (area, bar, line, pie, radar, sankey, heatmap, network…)
│   ├── components/        kanban, map, timeline, data-table, code-block 등
│   ├── hooks/ lib/        재사용 훅/유틸
│   ├── styles/ theme/     CSS 토큰, 테마 프리셋
│   └── index/             자동 생성된 API/스타일 목차 (AI용 지도)
├── packages/
│   ├── kernel/            규칙·검증·의존성 카탈로그 (프레임워크 무관한 "헌법")
│   ├── react/             @agent-html/react: Artifact, Block, Canvas, Node, defineArtifact
│   └── cli/               npm "agent-html": init / validate / dev / demo-build
│       ├── src/dev-server/  Vite 서버, 아티팩트 탐색, codex-bridge(Codex 연결)
│       └── src/host/        브라우저 뷰어 UI(오버레이, 프롬프트, 스레드, 테마 편집기)
├── apps/
│   ├── desktop/           AHTML 데스크톱 앱 (Tauri + Rust)
│   └── docs/              문서 사이트 (fumadocs + React Router)
├── taste/                 디자인/에이전트 사용성 철학 문서
├── docs/diary/            AI 작업 자세 메모
├── _archive/              옛날 .ahtml DSL 버전 (참고용, 사용 금지)
├── .claude/ .agents/      개발용 스킬: shadcn, ui-ux-pro-max, wrangler, gitnexus
└── .github/workflows/     Cloudflare Pages 배포, npm 배포
```

### 동작 흐름
1. `npx agent-html init` → 작업실(`agent-html/`) 생성
2. `npx agent-html dev` → 브라우저에 Canvas Host 실행
3. AI에게 "agent-html/artifacts에 대시보드 만들어줘" → AI가 `.tsx` 파일 작성
4. Host가 자동으로 화면 렌더링 + 규칙 위반 검사
5. 화면의 블록에 마우스 올려서 "이 차트 막대그래프로 바꿔줘" 입력
   → Host가 블록 id, 파일 경로, 사용자가 조작한 상태(필터 값 등)를 압축(TOON 포맷)해서 Codex에 전달
   → AI가 그 블록만 수정 → 화면 즉시 갱신

### 언제 쓰나
- AI 답변이 글보다 **화면**이 나은 경우: 운영 대시보드, 데이터 리포트, 비교표, 로드맵, 칸반, 코드리뷰 요약, 건강검진 결과 해석, 여행 계획, 제품 목업
- 결과물을 **파일로 남기고 계속 고쳐 가야** 하는 경우 (채팅은 흘러가지만 파일은 남음)

### 나(React/PHP 개발자)에게 도움 되는 점
- 보고서·대시보드를 AI로 빠르게 뽑고, 부분 수정 루프로 다듬기
- 클라이언트용 시안/목업 제작 시간 단축
- "AI가 규칙을 지키게 만드는 법"(AGENTS.md, 검증기, 자동 목차) 학습 교재
- shadcn/ui + Vite + React 19 + Tailwind 4 최신 스택 레퍼런스

---

## 2. 더 쉽게 설명

- **비유**: 지금까지 AI는 "편지(글)"로만 답했어. Agent-HTML은 AI에게 **"화이트보드 + 레고 블록 상자"** 를 주는 거야.
  - 레고 상자 = 미리 준비된 버튼·표·차트·칸반 부품
  - 화이트보드 = 브라우저 화면(Canvas Host)
  - 조립 설명서 = AGENTS.md / TASTE.md
  - 검사관 = validate (설명서 어기면 빨간불)
- **사용 장면**: "이번 달 매출 대시보드 만들어줘" → AI가 레고로 조립 → 화면에서 보다가 차트 부분만 콕 찍어 "월별로 바꿔줘" → 그 부분만 다시 조립.
- **장점 3가지**: ① 글보다 한눈에 보임 ② 파일로 남아서 계속 수정 가능 ③ 원하는 부분만 정확히 수정 요청

---

## 3. Q&A

### 설치 및 사용법
- 준비물: Node.js 22 이상, 파일을 편집할 수 있는 AI 에이전트(Claude Code, Codex 등)
- 일반 사용:
  ```bash
  npm install agent-html
  npx agent-html init      # agent-html/ 작업실 생성
  npx agent-html dev       # Canvas Host 실행 (브라우저에서 확인)
  npx agent-html validate  # 규칙 검사
  ```
  AI에게 요청 예시:
  ```text
  Build a dashboard artifact in agent-html/artifacts using Agent-HTML Canvas.
  Read agent-html/README.md and agent-html/AGENTS.md first.
  ```
- 이 저장소 자체를 개발할 때: `npm install` → `npm run dev` (Codex 파이프라인) / `npm run example:dev` (Codex 없이 미리보기) / `npm run test`, `npm run typecheck`, `npm run lint`
- 데스크톱 앱: `npm run desktop:dev` (Rust + Tauri 빌드 도구 필요)

### 플러그인? 스킬? MCP?
- **셋 다 아님.** 정확히는 **npm CLI 도구 + React 라이브러리 + 로컬 개발 서버(뷰어)** 로 된 "AI 작업 공간 프레임워크".
- MCP 서버 없음 (오히려 아티팩트 코드에서 MCP 호출을 금지함).
- Claude Code 플러그인 아님. 대신 AGENTS.md/README 규칙 파일만 읽으면 어떤 에이전트든 사용 가능.
- `.claude/skills`, `.agents/skills`에 있는 스킬(shadcn, ui-ux-pro-max, wrangler, gitnexus)은 이 저장소를 개발할 때 쓰는 보조 스킬이고, 예전 버전의 agent-html 스킬은 `_archive`로 은퇴.
- 블록 프롬프트 기능은 **Codex CLI(`codex app-server`)와 직접 연동**됨. `AGENT_HTML_CODEX_COMMAND` 환경변수로 실행 명령 교체 가능.

### API 토큰 필요?
- **Agent-HTML 자체는 API 토큰 불필요.** 외부 AI API를 직접 호출하지 않음.
- 다만 함께 쓰는 AI 에이전트는 각자 인증 필요: Claude Code(구독 로그인 또는 API 키), Codex(ChatGPT 로그인 또는 OpenAI API 키).
- `--pipeline example`로 띄우면 AI 없이도 예제 화면 구경 가능.
- 저장소의 Cloudflare 토큰은 원작자 사이트 배포용이라 나랑 무관.

### 왜 깃허브에서 유명할까?
- "채팅 UI 말고 캔버스"라는 명확한 메시지 — Claude Artifacts / ChatGPT Canvas로 대중화된 흐름을 **로컬·오픈소스**로 구현
- README의 GIF(칸반, 차트, 테마, 블록 프롬프트)가 시각적으로 강렬함
- shadcn/ui, Codex, AGENTS.md 등 요즘 핫한 키워드 조합
- 에이전트 친화 설계(규칙 파일, 자동 검증, 자동 목차)가 잘 되어 있어 개발자들이 참고용으로 스타
- 중국 커뮤니티 linux.do에서 확산 (README 중국어판 존재)
- 참고: ⭐ 538 수준이라 "초대형"은 아니고 **떠오르는 중견 프로젝트**

### 로컬 에이전트 구축에 도움 될까?
- **"두뇌"가 아니라 "출력 화면(UI 층)"으로 도움 됨.**
- 로컬 LLM(Ollama 등) 기반 에이전트를 만들면, 결과를 `agent-html/artifacts`에 `.tsx`로 쓰게 하고 `agent-html validate`를 자동 피드백 루프로 사용 가능
- 블록 프롬프트 연동은 Codex 전용 → Codex의 로컬 모델 모드를 쓰거나, `codex-bridge.mjs`를 참고해 내 에이전트용 브리지를 만들면 됨
- 참고할 설계: 규칙 파일 라우팅, 검증 커널, 사용자 조작 상태 압축 전달(TOON)

### 수익화 아이디어? → 4장 참고

### React나 PHP로 만들 수 있어?
- **React: 가능 (이미 React 그 자체).** Next.js나 Vite로 비슷한 뷰어·에디터를 만들거나, 이 패키지를 그대로 가져다 확장 가능.
- **PHP: 핵심 부분은 어렵고, 주변부는 가능.**
  - Canvas Host는 Vite + React 실시간 빌드에 의존해서 PHP로 그대로 옮기기 어려움
  - 대신 `agent-html demo-build`로 만든 정적 결과물을 PHP(Laravel) 사이트에 올려서 보여주기, 회원·결제·데이터 API를 PHP 백엔드로 제공하는 조합은 충분히 가능
  - 추천 조합: **화면/에디터 = React, 회원·결제·관리자 = PHP(Laravel)**

---

## 4. 수익화 아이디어 상세

> 라이선스: Apache-2.0이라 상업적 이용·수정 가능. 저작권/라이선스 고지만 유지. `_archive/apps/agent-html-app`(독점)은 절대 사용 금지.

| # | 아이디어 | 대상 | 방법 | 가격 예시 | 난이도 |
| --- | --- | --- | --- | --- | --- |
| 1 | AI 리포트/대시보드 제작 대행 | 소상공인, 스타트업, 마케팅팀 | 엑셀·CSV 받아서 AI로 대시보드 생성 → 정적 빌드 납품 | 건당 30~100만 원, 월 유지보수 10~30만 원 | 낮음 |
| 2 | 템플릿·테마 마켓 | 개발자, 기획자 | 업종별 아티팩트 템플릿 + 한국형 테마 판매 (크몽, Gumroad) | 템플릿 1~5만 원, 번들 10~20만 원 | 낮음 |
| 3 | 한국형 SaaS | 비개발자 팀 | 웹에서 데이터 업로드 → AI가 대시보드 생성 → 링크 공유. React(에디터) + Laravel(회원·결제) | 월 1~5만 원 구독 | 높음 |
| 4 | 사내 AI 대시보드 구축 컨설팅 | 중소기업 IT팀 | 사내 서버에 설치, 사내 데이터 연결, 직원 교육 | 프로젝트당 300~1,000만 원 | 중간 |
| 5 | 교육/콘텐츠 | 개발자, 직장인 | "AI로 대시보드 만들기" 강의, 유튜브, 전자책 | 강의 5~20만 원 | 낮음 |
| 6 | 업종 특화 리포트 | 병원, 부동산, 쇼핑몰 | 예제의 건강검진 해석기처럼 업종별 결과를 보기 좋게 해석해 주는 서비스 | 건당 과금 또는 월 구독 | 중간 |

### 추천 순서
1. **바로 시작**: 2번(템플릿) + 5번(콘텐츠)로 포트폴리오와 인지도 쌓기
2. **수입 만들기**: 1번(제작 대행)으로 실제 고객 확보, 반복되는 요구 파악
3. **확장**: 반복 요구를 제품화해서 3번(SaaS) 또는 6번(업종 특화)

### MVP 단계 (1번 기준)
1. 예제 6개를 참고해 업종별 샘플 대시보드 3개 제작 (매출, 재고, 마케팅)
2. `agent-html demo-build`로 정적 사이트 빌드 → Cloudflare Pages 등에 무료 배포해 포트폴리오화
3. 크몽/숨고 등에 서비스 등록
4. 고객 데이터 → AI 생성 → 블록 단위 수정 → 납품 흐름 표준화

### 리스크
- AI 모델 비용(구독/API)은 내 부담 → 가격에 반영
- 고객 데이터 보안: 로컬 작업, 민감 정보 마스킹
- 원작 프로젝트가 초기 단계라 구조 변경 가능 → 버전 고정(`agent-html@0.3.0`)
- Claude Artifacts, ChatGPT Canvas 등 대기업 기능과 경쟁 → **"한국어·업종 특화·납품형"** 으로 차별화
