# 국가연구개발사업 제재처분 안건 초안 작성 앱 — 스킬 명세서

> 파일: `index_수정_클로드.html`  
> 작성일: 2026-08-20  
> 환경: 단일 HTML 파일 · Vanilla JS · Gemini API (REST / SSE 스트리밍)

---

## 목차

1. [앱 개요 및 아키텍처](#1-앱-개요-및-아키텍처)
2. [데이터 모델](#2-데이터-모델)
3. [UI 상태 관리](#3-ui-상태-관리)
4. [단계별 워크플로우 (Phase 1~7)](#4-단계별-워크플로우)
5. [스킬 01 — Gemini API 연동 (SSE 스트리밍)](#5-스킬-01--gemini-api-연동)
6. [스킬 02 — Phase 3 페르소나 토론 (법률·회계)](#6-스킬-02--phase-3-페르소나-토론)
7. [스킬 03 — Phase 7 모의 처분 위원회 (5인)](#7-스킬-03--phase-7-모의-처분-위원회-5인)
8. [스킬 04 — API 키 관리](#8-스킬-04--api-키-관리)
9. [스킬 05 — 예시 데이터 (실제 제재 사례)](#9-스킬-05--예시-데이터)
10. [CSS 테마 변수 및 버블 스타일](#10-css-테마-변수-및-버블-스타일)
11. [보안 제약](#11-보안-제약)

---

## 1. 앱 개요 및 아키텍처

### 목적
혁신법(국가연구개발혁신법) 제31·32조에 근거한 연구개발비 제재처분 안건 초안을 AI 보조로 작성하는 단일 페이지 앱.

### 기술 스택
| 항목 | 내용 |
|------|------|
| 언어 | HTML5 + Vanilla JS (ES2022) |
| AI | Google Gemini API (REST, SSE streaming) |
| 스타일 | CSS 변수 기반 + Tailwind CDN |
| 폰트 | KoPub Dotum / KoPub Batang |
| 저장소 | 브라우저 `localStorage` (API 키만) |
| 상태 | `cp` (케이스패킷) + `ui` (UI 상태) 전역 객체 |

### 렌더 구조
```
.app
  ├── .masthead (네비 헤더 + 단계 스트립)
  └── .workspace (3패널 그리드: 20% | 40% | 40%)
        ├── .panel-log     ← 작업 로그 / 저장 내역
        ├── .panel-center  ← 주요 입력 / 단계별 메인 UI
        └── .panel-preview ← 미리보기 / 안건 초안 출력
```

---

## 2. 데이터 모델

### `cp` (Case Packet) — 메인 데이터 객체

```javascript
cp = {
  phase: 0,                    // 현재 단계 (0=랜딩, 1~7)
  created_at: "",
  updated_at: "",

  // Phase 1 — 기본 정보
  original_problem: "",        // 원문 위반 사실

  // Phase 2 — 위반 정보 구조화
  violation: {
    behavior: "",              // 위반 행위 요약
    frequency: "",             // 빈도 (초범/재범 등)
    law_basis: "",             // 법적 근거 조문
    evidence: []               // 증거 목록
  },

  // Phase 3 — 페르소나 검토 저장
  discussion: {
    law_review: "",            // 법률 페르소나 검토 요약
    acc_review: "",            // 회계 페르소나 검토 요약
    completed_at: ""           // 완료 시각
  },

  // Phase 4~5 — 제재 처분 산정
  sanctions: {
    recovery: {
      final_amount: 0          // 연구개발비 환수 금액 (원)
    },
    participation_restriction: {
      final: "",               // 참여제한 기간 (예: "2년")
      basis: ""                // 처분 근거 설명
    },
    surcharge: {
      applicable: false,
      base_amount: 0,
      final_amount: 0,
      formula: ""              // 산정식 (예: "위반금액 × 1배")
    },
    confirmed_from_chat: false
  },

  // Phase 5 — 최종 안건 텍스트
  draft_text: ""
}
```

---

## 3. UI 상태 관리

```javascript
ui = {
  // Phase 3 페르소나 토론
  discScript: [],        // { speaker:'lawyer'|'accountant'|'sys', text, streaming? }
  discStep: -1,
  discDone: false,
  isDiscApiLoading: false,

  // Phase 7 모의 처분 위원회
  mockDiscScript: [],    // { speaker: MOCK_COMMITTEE key | 'sys'|'usr', text, streaming? }
  mockDiscStep: -1,
  mockDiscDone: false,
  isMockApiLoading: false,
}
```

---

## 4. 단계별 워크플로우

| Phase | 이름 | 주요 동작 |
|-------|------|-----------|
| 0 | 랜딩 | API 키 상태 확인, 새 케이스 생성 |
| 1 | 위반 사실 입력 | 원문 입력 / 예시 데이터 로드 |
| 2 | 사실 구조화 | Gemini로 위반 행위·근거·증거 추출 |
| 3 | 페르소나 검토 | 법률+회계 2인 자동 토론 (스트리밍) |
| 4 | 처분 산정 | 환수금·참여제한·제재부가금 산정 |
| 5 | 안건 초안 | 최종 안건 문서 생성 및 편집 |
| 6 | 검토 (예비) | 추가 검토 단계 (확장 예정) |
| 7 | 모의 처분 위원회 | 5인 위원 AI 시뮬레이션 |

---

## 5. 스킬 01 — Gemini API 연동

### SSE 스트리밍 제너레이터

```javascript
async function* streamGeminiText(prompt, maxTokens = 1500) {
  // GEMINI_MODELS 배열을 순서대로 시도 (자동 폴백)
  // SSE 청크를 파싱하여 텍스트 조각을 yield
  // 실패 시 _cachedModel 초기화 후 다음 모델 시도
}
```

### 모델 폴백 배열

```javascript
const GEMINI_MODELS = [
  'gemini-2.5-flash',
  'gemini-2.0-flash',
  'gemini-1.5-flash'
];
let _cachedModel = null;  // 성공한 모델 캐싱
```

### 비스트리밍 (모의 토의 이전 레거시, 현재 스트리밍으로 대체됨)

```javascript
async function getGeminiResponse(prompt, maxTokens) { ... }
```

---

## 6. 스킬 02 — Phase 3 페르소나 토론

### 구성 페르소나

| 역할 | CSS 키 | 화풍 |
|------|--------|------|
| 법률 페르소나 | `lawyer` | 판례 중심, 절차 하자 지적 |
| 회계 페르소나 | `accountant` | 금액 산정, 정산 오류 검토 |
| 시스템 | `sys` | 사회자 역할 |

### 주요 함수

```javascript
// 토론 시작 — 스트리밍으로 스크립트 생성
async function startDiscussion() {
  for await (const chunk of streamGeminiText(buildDiscussionPrompt(), 1500)) {
    // 누적 텍스트 파싱 → ui.discScript 갱신 → patchDiscThread()
  }
  // 완료 후 검증: lawyer + accountant 모두 존재해야 함
  const hasLawyer   = normalized.some(m => m.speaker === 'lawyer');
  const hasAccountant = normalized.some(m => m.speaker === 'accountant');
  ui.discScript = (hasLawyer && hasAccountant)
    ? normalized
    : buildInitialDiscussionFallback();  // 둘 중 하나라도 없으면 폴백
}

// 400ms 간격 자동 진행
function autoAdvanceDisc() {
  if (ui.discDone) return;
  ui.discStep++;
  if (ui.discStep >= ui.discScript.length - 1) {
    ui.discDone = true;
    // 완료 시 cp.discussion 에 저장
    cp.discussion.law_review = ui.discScript
      .filter(m => m.speaker === 'lawyer')
      .map(m => m.text.replace(/\[제안[\s\S]*?\]/g, '').trim())
      .filter(Boolean).join('\n');
    cp.discussion.acc_review = ui.discScript
      .filter(m => m.speaker === 'accountant')
      .map(m => m.text.replace(/\[제안[\s\S]*?\]/g, '').trim())
      .filter(Boolean).join('\n');
    cp.discussion.completed_at = nowStr();
    render();
  }
  patchDiscThread();
}

// LLM 출력 파싱 (마크다운 데코레이션 처리)
// LAW_RE : /^(?:[*#\[<\d.\s]+)?\s*(?:법률|변호사|법무).*?[:\-]\s*([\s\S]*)/i
// ACC_RE : /^(?:[*#\[<\d.\s]+)?\s*(?:회계|회계사|재무).*?[:\-]\s*([\s\S]*)/i
function parseDiscussionLines(text) { ... }

// DOM 부분 업데이트 (전체 리렌더 없이)
function patchDiscThread() { ... }

// 버블 HTML 생성
function buildDiscBubblesHtml(msgs) { ... }
```

### 검토 내용 저장 표시

`renderLog()`에서 토론 완료 후 좌측 패널에 자동 표시:
- 법률 페르소나 검토 요약
- 회계 페르소나 검토 요약
- 완료 시각

---

## 7. 스킬 03 — Phase 7 모의 처분 위원회 (5인)

### 위원 구성 — `MOCK_COMMITTEE` 상수

```javascript
const MOCK_COMMITTEE = {
  'law-prog': {
    key:'law-prog', name:'이민준', fullName:'이민준 변호사',
    role:'진취적 변호사', icon:'⚖', badge:'진취', side:'left',
    profile:'판례 분석형 공격 변호사. 정부 패소 사례를 꿰고 있으며, 절차·재량 결함을 예리하게 찌른다.',
    style:'단호하고 직설적. "이건 취소 가능합니다", "절차 하자가 있어요" 식.'
  },
  'law-cons': {
    key:'law-cons', name:'김태원', fullName:'김태원 변호사',
    role:'보수적 변호사', icon:'📜', badge:'보수', side:'left',
    profile:'조문 원문·판례를 토씨 하나까지 확인하는 수비형 변호사.',
    style:'신중하고 완서. "원문을 다시 보겠습니다" 식.'
  },
  'acc-prog': {
    key:'acc-prog', name:'박서연', fullName:'박서연 회계사',
    role:'진취적 회계사', icon:'📊', badge:'진취', side:'right',
    profile:'정부 패소 사례의 금액 산정 오류 패턴 숙지. 공격형.',
    style:'예리·단도직입. "이 산식, 기초부터 다시 봐야 해요" 식.'
  },
  'acc-cons': {
    key:'acc-cons', name:'최두현', fullName:'최두현 회계사',
    role:'보수적 회계사', icon:'🔢', badge:'보수', side:'right',
    profile:'정산서를 줄 단위로 검토하는 꼼꼼한 회계사.',
    style:'천천히 단계적. "서두르지 맙시다" 식.'
  },
  'admin': {
    key:'admin', name:'정상민', fullName:'정상민 행정전문가',
    role:'행정전문가', icon:'🏛', badge:'행정', side:'left',
    profile:'혁신법·행정절차법·행정기본법·행정소송법 조문만 인용. 가장 보수적.',
    style:'공식·건조. "행정절차법 제21조 사전통지 요건 확인해야 합니다" 식. 항상 조문 번호 명시.'
  }
};

// 이름 → key 역방향 매핑
const MOCK_NAME_MAP = {};
Object.values(MOCK_COMMITTEE).forEach(p => { MOCK_NAME_MAP[p.name] = p.key; });
```

### CSS 버블 스타일 (각 위원별)

```css
/* law-prog (이민준 — 진취적 변호사) */
.chat-row.law-prog { flex-direction: row }
.chat-avatar.law-prog { background: linear-gradient(135deg, var(--danger), #c45050) }
.chat-bubble.law-prog { background: var(--danger-soft); border: 1.5px solid rgba(161,60,60,.2); border-top-left-radius: 4px }

/* law-cons (김태원 — 보수적 변호사) */
.chat-bubble.law-cons { background: #f0edf9; border-color: rgba(61,52,114,.2) }

/* acc-prog (박서연 — 진취적 회계사) */
.chat-row.acc-prog { flex-direction: row-reverse }  /* 우측 배치 */
.chat-bubble.acc-prog { background: #fff4e6; border-top-right-radius: 4px }

/* acc-cons (최두현 — 보수적 회계사) */
.chat-row.acc-cons { flex-direction: row-reverse }
.chat-bubble.acc-cons { background: var(--money-soft) }

/* admin (정상민 — 행정전문가) */
.chat-bubble.admin { background: var(--amber-soft) }
```

### 주요 함수

```javascript
// LLM 출력 파싱 — "이민준: ..." 형식 인식
function parseMockDiscussionLines(text) {
  // 각 위원 이름 정규식 자동 생성
  // 인식 못한 줄 → speaker:'sys' 로 수집
}

// 버블 HTML 빌더 — MOCK_COMMITTEE 메타 사용
function buildMockBubblesHtml(msgs) { ... }

// DOM 부분 업데이트
function patchMockDiscThread() {
  const c = document.getElementById('mockDiscThreadContainer');
  c.innerHTML = buildMockBubblesHtml(ui.mockDiscScript.slice(0, ui.mockDiscStep + 1));
  c.scrollTop = c.scrollHeight;
}

// 토의 시작 (스트리밍, 2000 maxTokens)
async function startMockDiscussion() {
  // 프롬프트: 5인 위원, 각자 고유 말투, "이민준: ..." 형식, 10~15줄
  // 완료 후 검증: 3인 이상 등장해야 함
  for await (const chunk of streamGeminiText(buildMockDiscussionPrompt(), 2000)) { ... }
}

// 위원장 발언 개입 (스트리밍, 900 maxTokens)
async function submitMockDiscIntervention() {
  // 적절한 2~3인 위원이 반응
  // 기존 스크립트 컨텍스트 포함하여 프롬프트 구성
}

// 빠른 질문 버튼 헬퍼
function submitMockWithText(text) {
  const input = document.getElementById('mockDiscIntervene');
  if (input) input.value = text;
  submitMockDiscIntervention();
}
```

### `renderPhase7` 빠른 질문 버튼 목록

| 버튼 | 전송 텍스트 |
|------|-------------|
| 절차 하자 집중 | 절차적 하자가 소송에서 문제될 가능성을 중심으로 검토해 주세요 |
| 재량권 리스크 | 재량권 일탈·남용 주장이 법원에서 인용될 가능성은 얼마나 됩니까 |
| 금액 산정 검토 | 환수금액과 제재부가금 산정 근거가 법원에서 인정받을 수 있는지 검토해 주세요 |
| 처분 시효 | 처분 시효(제32조제5항 10년) 준수 여부를 확인해 주세요 |
| 보완 서류 목록 | 보완이 필요한 증거·서류 목록을 정리해 주세요 |

---

## 8. 스킬 04 — API 키 관리

### 보안 원칙 (불변)

> **API 키는 HTML에 하드코딩하지 않는다. 브라우저 localStorage에 사용자가 직접 입력한 키만 저장한다.**

### 관련 함수

```javascript
const GEMINI_KEY_STORAGE = 'gemini_api_key';

function hasGeminiApiKey()     { return !!localStorage.getItem(GEMINI_KEY_STORAGE); }
function getGeminiApiKey()     { return localStorage.getItem(GEMINI_KEY_STORAGE) || ''; }
function saveGeminiApiKey(key) { localStorage.setItem(GEMINI_KEY_STORAGE, key.trim()); }
function removeGeminiApiKey()  { localStorage.removeItem(GEMINI_KEY_STORAGE); }

function configureGeminiApiKey() {
  const key = prompt('Gemini API 키를 입력하세요:\n(Google AI Studio에서 발급)');
  if (key && key.trim()) { saveGeminiApiKey(key); render(); }
}
function resetGeminiApiKey() {
  if (confirm('저장된 API 키를 삭제합니까?')) { removeGeminiApiKey(); render(); }
}
```

### 랜딩 페이지 API 키 상태 카드

- 키 설정 여부를 뱃지로 표시 (✓ 설정됨 / ✗ 미설정)
- [키 입력 / 변경] 버튼 상시 노출
- 키 설정 시 [키 삭제] 버튼 추가 표시

---

## 9. 스킬 05 — 예시 데이터

### 사용 함수

```javascript
function loadExampleData() { ... }
```

### 예시 사건 개요

```
연구책임자 홍길동 (한국과학기술원, 미래원천기술개발사업 RS-2023-0001)
2023.08.12(토) 22시경 일반유흥주점 '향가리'에서
연구개발비 카드로 3,500,000원 결제 → 사후 회의비 처리
```

### 예시 처분 내용

| 항목 | 내용 |
|------|------|
| 환수 금액 | 3,500,000원 |
| 참여제한 | 2년 |
| 참여제한 근거 | 혁신법 제32조제1항제3호 및 시행령 별표 — 부정행위(연구개발비 사용기준 위반), 초범·위반규모 고려 최소 기간 |
| 제재부가금 | 3,500,000원 (위반금액 × 1배) |
| 위반 빈도 | 초범 (동일 과제 내 동일 유형 위반 최초 확인) |

---

## 10. CSS 테마 변수 및 버블 스타일

### 색상 변수 체계

```css
:root {
  /* 잉크 / 배경 */
  --ink: #1b2422;       --ink-2: #414b46;
  --muted: #6f766f;     --paper: #faf9f4;     --paper-2: #f2f0e6;
  --line: #d9d6c9;      --line-strong: #b7b39f;

  /* 브랜드 */
  --navy: #1c3450;      --navy-2: #274f76;

  /* 페르소나 */
  --agent: #0f7d74;     --agent-soft: #e7f4f1;   /* AI 에이전트 */
  --law:   #4a3f82;     --law-soft:   #efecf8;   /* 법률 */
  --money: #1f6b46;     --money-soft: #e9f5ee;   /* 회계·금액 */
  --user:  #2f6298;     --user-soft:  #eaf2fa;   /* 사용자 */
  --amber: #96650f;     --amber-soft: #fbf1dd;   /* 경고·행정 */
  --danger:#a13c3c;     --danger-soft:#faecec;   /* 위험·강조 */
}
```

### 채팅 버블 공통 구조

```html
<div class="chat-row {persona-key}">
  <div class="chat-avatar {persona-key}">{icon}</div>
  <div class="chat-body {persona-key}">
    <div class="chat-name {persona-key}">{이름} <span class="badge">{역할}</span></div>
    <div class="chat-bubble {persona-key}">
      {텍스트}<span class="stream-cursor">▌</span>  <!-- 스트리밍 중만 표시 -->
    </div>
  </div>
</div>
```

---

## 11. 보안 제약

1. **API 키 하드코딩 금지** — localStorage 사용자 직접 입력만 허용
2. **XSS 방지** — 사용자 입력 표시 시 `esc()` 함수(HTML 이스케이프) 적용
3. **외부 데이터 전송 없음** — Gemini API 호출만 수행, 케이스 데이터 외부 전송 없음

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|-----------|
| 2026-08-20 | Phase 3 토론 중단 버그 수정 (hasLawyer && hasAccountant 검증) |
| 2026-08-20 | `cp.discussion` 필드 추가 — 페르소나 검토 내용 자동 저장 |
| 2026-08-20 | 랜딩 페이지 API 키 상태 카드 추가 |
| 2026-08-20 | Phase 7 모의토의 스트리밍 전환 (`getGeminiResponse` → `streamGeminiText`) |
| 2026-08-20 | `MOCK_COMMITTEE` 5인 위원 구성 (진취·보수 변호사, 진취·보수 회계사, 행정전문가) |
| 2026-08-20 | `parseMockDiscussionLines`, `buildMockBubblesHtml`, `patchMockDiscThread` 신규 |
| 2026-08-20 | `renderPhase7` — 위원 명단 로스터, 빠른 질문 버튼, `buildMockBubblesHtml` 연결 |
| 2026-08-20 | 예시 데이터 구체화 (유흥주점 카드결제 3,500,000원 실제 사례형) |
