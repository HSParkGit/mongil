# 몬길 (mongil) MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** "몬스터 길들이기 스타 다이브" 일일/주간 숙제 체크용 단일 HTML 웹앱(`index.html`) 구현 — KST 09:00 기준 자동 리셋, 다크 모드 카드형 UI, 의뢰 우선 보스 치트시트 포함.

**Architecture:** 단일 `index.html` 하나에 HTML + CSS + JS + 이모지 아이콘을 모두 인라인. 상태는 `localStorage` 키 `mongil_state` 에 저장. 시간 계산·리셋 판정 같은 순수 함수는 `?test=1` 쿼리로 접근하는 인라인 테스트 러너에서 검증 (외부 테스트 프레임워크 없음).

**Tech Stack:** Vanilla HTML5 + CSS3 (커스텀 프로퍼티 / flex) + ES2020 JavaScript. 빌드 도구·번들러·의존성 전무. 브라우저만 있으면 실행.

**Spec:** `docs/superpowers/specs/2026-04-23-mongil-design.md`

---

## 파일 구조

단일 파일:

```
/Users/baghyeonsin/RubymineProjects/mongil/
└── index.html   (신규 — 유일한 산출물)
```

`index.html` 내부 논리 섹션 (주석으로 구분):

1. `<head>` — 메타, viewport, CSS 변수 + 전체 스타일
2. `<body>` — 정적 구조(헤더, 일일 컨테이너, 주간 컨테이너, 치트시트 컨테이너). 카드 자체는 JS 렌더.
3. `<script>` 내부:
   - `// --- constants ---` DAILY / WEEKLY / CHEATSHEET 데이터
   - `// --- time ---` `computeBaselineDate`, `computeIsoWeek`, `nowKst`
   - `// --- storage ---` `loadState`, `saveState`, 기본값
   - `// --- reset ---` `applyReset`
   - `// --- render ---` DOM 갱신
   - `// --- events ---` 카드 탭, 치트시트 토글, `storage` 이벤트, setInterval
   - `// --- tests ---` `?test=1` 분기 (바깥 실행 차단)
   - `// --- bootstrap ---` 최종 진입점

단일 파일이라 "다른 파일을 만들지 말 것" — 이미지도 이모지만 사용(이모지는 텍스트라 base64 불필요).

---

## 공통 실행/검증 방법

- **앱 실행**: Finder에서 `index.html` 더블클릭 → 시스템 기본 브라우저가 `file:///...` 로 연다.
- **테스트 실행**: 같은 파일을 `file:///...mongil/index.html?test=1` 로 열고 DevTools 콘솔 확인. `PASS:` / `FAIL:` 로그 출력, 전체 결과 `ALL TESTS PASSED` 또는 `X TESTS FAILED`.
- **커밋은 생략**: 이 프로젝트는 git 저장소가 아님. 원하면 사용자가 나중에 `git init` 해서 한 번에 스냅샷.

---

## Task 1: 프로젝트 스켈레톤 + 다크 테마 + 테스트 러너 뼈대

**Files:**
- Create: `/Users/baghyeonsin/RubymineProjects/mongil/index.html`

**목적:** 빈 구조와 스타일 기반, 그리고 `?test=1` 모드 분기 뼈대만. 이후 태스크가 여기에 살을 붙인다.

- [ ] **Step 1: `index.html` 전체 작성**

파일 전체 내용:

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, viewport-fit=cover" />
  <meta name="theme-color" content="#0f1115" />
  <title>몬길</title>
  <style>
    :root {
      --bg: #0f1115;
      --surface: #171a21;
      --surface-2: #1f232c;
      --border: #2a2f3a;
      --text: #e6e9ef;
      --text-dim: #9aa3b2;
      --accent: #7aa2ff;
      --done: #3fb950;
      --radius: 14px;
      --pad: 14px;
      --font: -apple-system, BlinkMacSystemFont, "Pretendard", "Apple SD Gothic Neo",
              system-ui, Segoe UI, Roboto, sans-serif;
    }
    * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
    html, body { margin: 0; padding: 0; background: var(--bg); color: var(--text); font-family: var(--font); }
    body { max-width: 560px; margin: 0 auto; padding: env(safe-area-inset-top) 12px 40px; }

    header.top { padding: 18px 4px 10px; display: flex; flex-direction: column; gap: 8px; }
    header.top .row { display: flex; justify-content: space-between; align-items: baseline; }
    header.top h1 { margin: 0; font-size: 22px; letter-spacing: 0.2px; }
    header.top .date { color: var(--text-dim); font-size: 13px; font-variant-numeric: tabular-nums; }
    .progress { display: flex; align-items: center; gap: 10px; color: var(--text-dim); font-size: 13px; }
    .progress .bar { flex: 1; height: 6px; background: var(--surface); border-radius: 999px; overflow: hidden; }
    .progress .bar > span { display: block; height: 100%; width: 0%; background: var(--accent); transition: width 240ms ease; }

    section.group { margin-top: 18px; }
    section.group h2 { font-size: 13px; color: var(--text-dim); margin: 0 4px 8px; font-weight: 600; letter-spacing: 0.6px; text-transform: uppercase; }

    .card {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: var(--radius);
      padding: var(--pad);
      margin-bottom: 10px;
      display: flex; gap: 12px; align-items: flex-start;
      cursor: pointer; user-select: none;
      transition: background 120ms ease, opacity 120ms ease;
    }
    .card:active { background: var(--surface-2); }
    .card.done { opacity: 0.55; }
    .card .check { width: 22px; height: 22px; border-radius: 6px; border: 2px solid var(--border); flex: 0 0 22px; margin-top: 2px; display: inline-flex; align-items: center; justify-content: center; font-size: 14px; color: transparent; }
    .card.done .check { background: var(--done); border-color: var(--done); color: #081a0a; }
    .card.done .check::before { content: "✓"; }
    .card .body { flex: 1; min-width: 0; }
    .card .label { font-size: 15px; font-weight: 600; }
    .card .note { margin-top: 3px; font-size: 12.5px; color: var(--text-dim); }

    details.cheatsheet { margin-top: 24px; background: transparent; }
    details.cheatsheet > summary { list-style: none; cursor: pointer; padding: 12px 6px; color: var(--text-dim); font-size: 13px; letter-spacing: 0.4px; text-transform: uppercase; font-weight: 600; display: flex; align-items: center; gap: 8px; }
    details.cheatsheet > summary::-webkit-details-marker { display: none; }
    details.cheatsheet > summary::before { content: "▶"; font-size: 10px; transition: transform 160ms ease; }
    details.cheatsheet[open] > summary::before { transform: rotate(90deg); }
    .boss-card { background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius); padding: 12px; margin-bottom: 8px; display: flex; gap: 10px; align-items: flex-start; }
    .boss-card .ico { font-size: 22px; flex: 0 0 28px; text-align: center; line-height: 28px; }
    .boss-card .bbody { flex: 1; min-width: 0; }
    .boss-card .bname { font-size: 14.5px; font-weight: 600; }
    .boss-card .tags { margin-top: 4px; display: flex; gap: 4px; flex-wrap: wrap; }
    .boss-card .tag { font-size: 10.5px; padding: 2px 7px; border-radius: 999px; font-weight: 600; letter-spacing: 0.3px; }
    .tag.field-miss { background: #4a1a1a; color: #ff9a9a; }
    .tag.mat        { background: #4a311a; color: #ffc38a; }
    .tag.cheatsheet { background: #16314e; color: #9ec3ff; }
    .tag.op         { background: #3a1a4a; color: #d5a7ff; }
    .tag.prime      { background: #4a421a; color: #ffe28a; }
    .tag.codex      { background: #2a2f3a; color: var(--text-dim); }
    .boss-card .bnote { margin-top: 6px; font-size: 12.5px; color: var(--text-dim); line-height: 1.4; }

    pre.testlog { white-space: pre-wrap; background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius); padding: 12px; font-size: 12px; margin: 12px 0; }
    .hidden { display: none !important; }
  </style>
</head>
<body>
  <header class="top">
    <div class="row">
      <h1>🌙 몬길</h1>
      <div class="date" id="dateLabel">—</div>
    </div>
    <div class="progress">
      <span id="progressText">0/0 완료</span>
      <div class="bar"><span id="progressBar"></span></div>
      <span id="progressPct">0%</span>
    </div>
  </header>

  <section class="group">
    <h2>일일</h2>
    <div id="dailyList"></div>
  </section>

  <section class="group">
    <h2>주간</h2>
    <div id="weeklyList"></div>
  </section>

  <details class="cheatsheet" id="cheatsheet">
    <summary>의뢰 치트시트</summary>
    <div id="cheatsheetList"></div>
  </details>

  <pre class="testlog hidden" id="testlog"></pre>

  <script>
  "use strict";

  // --- bootstrap ---
  const params = new URLSearchParams(location.search);
  const TEST_MODE = params.get("test") === "1";

  if (TEST_MODE) {
    // Task 2 이후에 채움
    document.getElementById("testlog").classList.remove("hidden");
    document.getElementById("testlog").textContent = "Test mode — no tests registered yet.";
  } else {
    // Task 3 이후에 채움
    document.getElementById("dateLabel").textContent = "(앱 부트스트랩 미구현)";
  }
  </script>
</body>
</html>
```

- [ ] **Step 2: 파일을 브라우저에서 열어 확인**

macOS에서:

```bash
open /Users/baghyeonsin/RubymineProjects/mongil/index.html
```

**Expected:** 다크 배경 페이지에 "🌙 몬길", "(앱 부트스트랩 미구현)", 빈 "일일"/"주간" 섹션, "의뢰 치트시트" 접힘 섹션이 보임. 에러 없이 로드.

- [ ] **Step 3: 테스트 모드 분기 확인**

브라우저에서 URL 끝에 `?test=1` 붙여서 새로고침.

**Expected:** 헤더 아래에 `Test mode — no tests registered yet.` 회색 박스 표시. (로직 없음 — 분기만 확인)

---

## Task 2: 시간 계산 순수 함수 (KST 기준일 + ISO 주 번호) + 테스트 러너

**Files:**
- Modify: `/Users/baghyeonsin/RubymineProjects/mongil/index.html` — `<script>` 내 `// --- bootstrap ---` 위에 `// --- time ---` 섹션 추가. `TEST_MODE` 분기를 실제 테스트 실행으로 교체.

**배경 — 알고리즘:**

- **KST 기준일**: 현재 UTC 시각에 +9시간을 더해 KST 시각을 만든 뒤, **시(hour)가 9 미만이면 "어제"**, **9 이상이면 "오늘"**. 결과는 `"YYYY-MM-DD"` 문자열. 로컬 타임존 API(`toLocaleDateString` 등) **사용 금지** — 사용자가 해외에서 접속해도 KST 고정이어야 함.
- **ISO 주 번호 (KST)**: 주간 리셋은 "월요일 09:00 KST"가 경계. 아이디어: 현재 KST 시각에서 9시간 뒤로 당겨(=리셋 기준을 00:00 경계로 정렬) 그 시각이 어느 월요일 주에 속하는지 ISO week로 계산하면, "월요일 09:00 이전은 지난 주, 이후는 이번 주"가 자연스럽게 나온다. 출력은 `"YYYY-Www"` (예: `"2026-W17"`).

- [ ] **Step 1: `// --- time ---` 섹션 + 테스트 러너 구현**

`<script>` 내부에서 `// --- bootstrap ---` 블록을 통째로 **아래로** 교체:

```js
// --- time ---
// All functions take a UTC ms epoch (Date.now() style) and return strings.
// They do NOT read clock/timezone from the environment.

function kstPartsFromUtcMs(utcMs) {
  // Shift to KST by adding 9h, then read UTC fields (which now represent KST).
  const d = new Date(utcMs + 9 * 3600 * 1000);
  return {
    y: d.getUTCFullYear(),
    m: d.getUTCMonth() + 1,
    d: d.getUTCDate(),
    h: d.getUTCHours(),
    dow: d.getUTCDay(), // 0=Sun..6=Sat
  };
}

function pad2(n) { return n < 10 ? "0" + n : "" + n; }

function computeBaselineDate(utcMs) {
  // Daily reset at KST 09:00. Subtract 9h from the KST wallclock so the
  // boundary becomes midnight, then read the calendar date.
  const kst = new Date(utcMs + 9 * 3600 * 1000);
  const effective = new Date(kst.getTime() - 9 * 3600 * 1000);
  return `${effective.getUTCFullYear()}-${pad2(effective.getUTCMonth()+1)}-${pad2(effective.getUTCDate())}`;
}

function computeIsoWeekKst(utcMs) {
  // Align boundary to Monday 09:00 KST: subtract 9h from KST wallclock,
  // then compute ISO week on the resulting "KST-shifted" date.
  const kst = new Date(utcMs + 9 * 3600 * 1000);
  const shifted = new Date(kst.getTime() - 9 * 3600 * 1000);
  // ISO week algorithm (standard): Thursday of the current week defines the year.
  const date = new Date(Date.UTC(
    shifted.getUTCFullYear(),
    shifted.getUTCMonth(),
    shifted.getUTCDate()
  ));
  const dayNum = (date.getUTCDay() + 6) % 7; // Mon=0..Sun=6
  date.setUTCDate(date.getUTCDate() - dayNum + 3); // Thursday of this week
  const firstThu = new Date(Date.UTC(date.getUTCFullYear(), 0, 4));
  const firstThuDayNum = (firstThu.getUTCDay() + 6) % 7;
  firstThu.setUTCDate(firstThu.getUTCDate() - firstThuDayNum + 3);
  const week = 1 + Math.round((date.getTime() - firstThu.getTime()) / (7 * 24 * 3600 * 1000));
  return `${date.getUTCFullYear()}-W${pad2(week)}`;
}

// --- tests ---
function runTests() {
  const log = [];
  let pass = 0, fail = 0;
  function assert(cond, msg) {
    if (cond) { pass++; log.push("PASS: " + msg); }
    else      { fail++; log.push("FAIL: " + msg); }
  }
  function assertEq(actual, expected, msg) {
    assert(actual === expected, `${msg} — expected ${JSON.stringify(expected)}, got ${JSON.stringify(actual)}`);
  }

  // Helpers — build UTC ms from KST wallclock for test clarity.
  function kst(y, m, d, h = 0, mi = 0) {
    return Date.UTC(y, m - 1, d, h - 9, mi); // subtract 9h to go from KST to UTC
  }

  // computeBaselineDate: before 09:00 KST → yesterday
  assertEq(computeBaselineDate(kst(2026, 4, 23,  8, 59)), "2026-04-22", "baseline: 08:59 KST → yesterday");
  assertEq(computeBaselineDate(kst(2026, 4, 23,  9,  0)), "2026-04-23", "baseline: 09:00 KST → today");
  assertEq(computeBaselineDate(kst(2026, 4, 23, 23, 59)), "2026-04-23", "baseline: 23:59 KST → today");
  assertEq(computeBaselineDate(kst(2026, 4, 24,  0,  0)), "2026-04-23", "baseline: 00:00 KST next day → still today's session");
  assertEq(computeBaselineDate(kst(2026, 1,  1,  0,  0)), "2025-12-31", "baseline: Jan 1 00:00 KST → last year");

  // computeIsoWeekKst: Monday 09:00 KST boundary
  // 2026-04-20 is a Monday. Before Mon 09:00 = previous week; after = new week.
  assertEq(computeIsoWeekKst(kst(2026, 4, 20,  8, 59)), "2026-W16", "iso week: Mon 08:59 KST → previous week");
  assertEq(computeIsoWeekKst(kst(2026, 4, 20,  9,  0)), "2026-W17", "iso week: Mon 09:00 KST → new week");
  assertEq(computeIsoWeekKst(kst(2026, 4, 23, 12,  0)), "2026-W17", "iso week: Thu 12:00 KST → W17");
  assertEq(computeIsoWeekKst(kst(2026, 4, 27,  8, 59)), "2026-W17", "iso week: next Mon 08:59 → still W17");

  log.push(`\n${fail === 0 ? "ALL TESTS PASSED" : fail + " TESTS FAILED"} (${pass} pass, ${fail} fail)`);
  return { log, pass, fail };
}

// --- bootstrap ---
const params = new URLSearchParams(location.search);
const TEST_MODE = params.get("test") === "1";

if (TEST_MODE) {
  const { log } = runTests();
  const el = document.getElementById("testlog");
  el.classList.remove("hidden");
  el.textContent = log.join("\n");
  console.log(log.join("\n"));
} else {
  document.getElementById("dateLabel").textContent = "(앱 부트스트랩 미구현)";
}
```

- [ ] **Step 2: 테스트 실행하여 모두 PASS 확인**

`index.html?test=1` 을 새로고침. 화면 + 콘솔에서 출력 확인.

**Expected:** 9개 PASS, 0개 FAIL, 마지막 줄 `ALL TESTS PASSED (9 pass, 0 fail)`.

- [ ] **Step 3: 일반 모드 확인**

`?test=1` 빼고 새로고침 → 헤더 "(앱 부트스트랩 미구현)" 그대로, 에러 없음.

---

## Task 3: 상태 관리 (`localStorage`) + 항목 데이터 상수 + 리셋 판정

**Files:**
- Modify: `/Users/baghyeonsin/RubymineProjects/mongil/index.html` — `<script>` 맨 위에 `// --- constants ---` 섹션 추가, `// --- time ---` 밑에 `// --- storage ---` + `// --- reset ---` 섹션 추가. `runTests()` 에 새 케이스 추가.

- [ ] **Step 1: 상수 섹션 추가 (스크립트 맨 위)**

`<script> "use strict";` 바로 아래에 삽입:

```js
// --- constants ---
const STORAGE_KEY = "mongil_state";

const DAILY_ITEMS = [
  { key: "login",     label: "로그인",                       note: "공헌도 +20" },
  { key: "quest",     label: "의뢰 5개 완수",                 note: "공헌도 +60, 별빛 25×5" },
  { key: "capture",   label: "필드 몬스터 5마리 포획",        note: "공헌도 +40" },
  { key: "reward",    label: "일일 공헌도 100점 보상 수령",   note: "별빛 50, 골드 5만 (7일 이월)" },
  { key: "dungeon",   label: "던전 10판",                     note: "열쇠 소모, 최고난도 1클 보너스" },
  { key: "fieldboss", label: "필드 보스 처치",                note: "도감·네임드 몬스터링" },
];

const WEEKLY_ITEMS = [
  { key: "rift", label: "균열 5판 + 입장권 구매", note: "월요일 09:00 리셋" },
];

const DAILY_KEYS  = DAILY_ITEMS.map(i => i.key);
const WEEKLY_KEYS = WEEKLY_ITEMS.map(i => i.key);

const CHEATSHEET = [
  { name: "스푼머거", icon: "🍴", tags: ["field-miss","mat"],
    note: "의뢰 수락 시에만 등장. 폭식의 껍데기(바람 4셋) 드랍" },
  { name: "포크머거", icon: "🍴", tags: ["field-miss","mat"],
    note: "의뢰 수락 시에만 등장. 탐식의 비계(불 2셋) 드랍" },
  { name: "아몬",     icon: "🌟", tags: ["op","prime"],
    note: "서브딜·서포트 최강. 프라임 합성 필수급" },
  { name: "스카",     icon: "⚔️", tags: ["op"],
    note: "치명타 시 보스 피해 증가. 1티어" },
  { name: "보르보그", icon: "🔥", tags: ["op"],
    note: "종결급 품종. 굴각 합성 재료" },
  { name: "몰리",     icon: "✨", tags: ["op"],
    note: "약공 시 아군 전체 공증 버프. 황금향 재료" },
  { name: "이리도커", icon: "⚡", tags: ["cheatsheet"], note: "의뢰 우선 수락 추천" },
  { name: "슬라킹",   icon: "💧", tags: ["cheatsheet"], note: "의뢰 우선 수락 추천" },
  { name: "타그락",   icon: "🗡️", tags: ["cheatsheet"], note: "의뢰 우선 수락 추천" },
  { name: "하카",     icon: "🦅", tags: ["cheatsheet"], note: "의뢰 우선 수락 추천" },
];

const TAG_LABEL = {
  "field-miss": "필드미젠",
  "mat":        "재료",
  "cheatsheet": "치트시트",
  "op":         "사기",
  "prime":      "프라임합성",
  "codex":      "도감작",
};
```

- [ ] **Step 2: `// --- storage ---` 섹션 추가 (time 밑)**

```js
// --- storage ---
function defaultState(nowMs) {
  return {
    baselineDate: computeBaselineDate(nowMs),
    weeklyBaselineWeek: computeIsoWeekKst(nowMs),
    cheatsheetOpen: false,
    checked: {},
  };
}

function loadState(nowMs) {
  let raw;
  try { raw = localStorage.getItem(STORAGE_KEY); }
  catch { return defaultState(nowMs); }
  if (!raw) return defaultState(nowMs);
  try {
    const parsed = JSON.parse(raw);
    if (!parsed || typeof parsed !== "object") return defaultState(nowMs);
    return {
      baselineDate:       typeof parsed.baselineDate === "string"       ? parsed.baselineDate       : computeBaselineDate(nowMs),
      weeklyBaselineWeek: typeof parsed.weeklyBaselineWeek === "string" ? parsed.weeklyBaselineWeek : computeIsoWeekKst(nowMs),
      cheatsheetOpen:     !!parsed.cheatsheetOpen,
      checked:            (parsed.checked && typeof parsed.checked === "object") ? { ...parsed.checked } : {},
    };
  } catch {
    return defaultState(nowMs);
  }
}

function saveState(state) {
  try { localStorage.setItem(STORAGE_KEY, JSON.stringify(state)); }
  catch { /* quota or disabled — intentionally silent; this is a personal app */ }
}
```

- [ ] **Step 3: `// --- reset ---` 섹션 추가**

```js
// --- reset ---
// Returns a NEW state with daily/weekly checks reset as needed.
function applyReset(state, nowMs) {
  const today = computeBaselineDate(nowMs);
  const week  = computeIsoWeekKst(nowMs);

  let checked = state.checked;
  let changed = false;

  if (state.baselineDate !== today) {
    const next = { ...checked };
    for (const k of DAILY_KEYS) delete next[k];
    checked = next;
    changed = true;
  }
  if (state.weeklyBaselineWeek !== week) {
    const next = { ...checked };
    for (const k of WEEKLY_KEYS) delete next[k];
    checked = next;
    changed = true;
  }

  if (!changed && state.baselineDate === today && state.weeklyBaselineWeek === week) {
    return state;
  }
  return {
    ...state,
    baselineDate: today,
    weeklyBaselineWeek: week,
    checked,
  };
}
```

- [ ] **Step 4: 리셋 테스트 케이스 추가**

`runTests()` 마지막 `log.push(\`\\n${...}\`)` 줄 바로 위에 삽입:

```js
  // applyReset
  const t1 = kst(2026, 4, 23, 10, 0);
  const t1Next = kst(2026, 4, 24, 10, 0); // next day after 09:00
  const stateA = {
    baselineDate: computeBaselineDate(t1),
    weeklyBaselineWeek: computeIsoWeekKst(t1),
    cheatsheetOpen: false,
    checked: { login: true, quest: true, rift: true },
  };
  const afterA = applyReset(stateA, t1Next);
  assertEq(afterA.baselineDate, "2026-04-24", "reset: daily date advances");
  assertEq(afterA.checked.login, undefined, "reset: login cleared next day");
  assertEq(afterA.checked.quest, undefined, "reset: quest cleared next day");
  assertEq(afterA.checked.rift, true, "reset: weekly rift preserved within same week");

  const t2Mon = kst(2026, 4, 27,  9, 0); // next Monday 09:00 → new week W18
  const afterB = applyReset(afterA, t2Mon);
  assertEq(afterB.weeklyBaselineWeek, "2026-W18", "reset: weekly week advances at Mon 09:00");
  assertEq(afterB.checked.rift, undefined, "reset: rift cleared at weekly boundary");

  const sameDay = applyReset(stateA, kst(2026, 4, 23, 20, 0));
  assert(sameDay === stateA, "reset: no-op returns same reference when nothing changed");

  // loadState fallback
  localStorage.removeItem(STORAGE_KEY);
  const fresh = loadState(t1);
  assertEq(fresh.checked && typeof fresh.checked, "object", "loadState: empty storage returns default");
  localStorage.setItem(STORAGE_KEY, "{{{not json");
  const corrupt = loadState(t1);
  assertEq(corrupt.baselineDate, computeBaselineDate(t1), "loadState: corrupt JSON → default");
  localStorage.removeItem(STORAGE_KEY);
```

- [ ] **Step 5: 테스트 전체 재실행**

`index.html?test=1` 새로고침.

**Expected:** 16개 PASS, 0개 FAIL, `ALL TESTS PASSED`.

---

## Task 4: 렌더링 (헤더/날짜/진행률 + 체크리스트 카드) + 체크 인터랙션

**Files:**
- Modify: `/Users/baghyeonsin/RubymineProjects/mongil/index.html` — `<script>` 에 `// --- render ---` / `// --- events ---` 섹션 추가, `// --- bootstrap ---` 의 일반 모드 분기를 실제 앱으로 교체.

- [ ] **Step 1: `// --- render ---` 섹션 추가 (reset 밑)**

```js
// --- render ---
let STATE = null; // mutated via setState()
let NOW_MS_FN = () => Date.now(); // swappable for tests (not used in v1)

function cardHtml(item, done) {
  return `
    <div class="card ${done ? "done" : ""}" data-key="${item.key}" role="button" aria-pressed="${done}">
      <div class="check"></div>
      <div class="body">
        <div class="label">${escapeHtml(item.label)}</div>
        <div class="note">${escapeHtml(item.note)}</div>
      </div>
    </div>`;
}

function escapeHtml(s) {
  return String(s).replace(/[&<>"']/g, c => ({
    "&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"
  }[c]));
}

function renderHeader() {
  document.getElementById("dateLabel").textContent = STATE.baselineDate;
  const total = DAILY_ITEMS.length + WEEKLY_ITEMS.length;
  const done = [...DAILY_KEYS, ...WEEKLY_KEYS].filter(k => !!STATE.checked[k]).length;
  const pct = total === 0 ? 0 : Math.round((done / total) * 100);
  document.getElementById("progressText").textContent = `${done}/${total} 완료`;
  document.getElementById("progressPct").textContent = `${pct}%`;
  document.getElementById("progressBar").style.width = pct + "%";
}

function renderLists() {
  document.getElementById("dailyList").innerHTML =
    DAILY_ITEMS.map(it => cardHtml(it, !!STATE.checked[it.key])).join("");
  document.getElementById("weeklyList").innerHTML =
    WEEKLY_ITEMS.map(it => cardHtml(it, !!STATE.checked[it.key])).join("");
}

function renderAll() {
  renderHeader();
  renderLists();
  // Cheatsheet handled in Task 5.
}

function setState(next) {
  STATE = next;
  saveState(STATE);
  renderAll();
}
```

- [ ] **Step 2: `// --- events ---` 섹션 추가**

```js
// --- events ---
function toggle(key) {
  const next = { ...STATE, checked: { ...STATE.checked } };
  if (next.checked[key]) delete next.checked[key];
  else next.checked[key] = true;
  setState(next);
}

function installCheckHandlers() {
  for (const id of ["dailyList", "weeklyList"]) {
    document.getElementById(id).addEventListener("click", (e) => {
      const card = e.target.closest(".card");
      if (!card) return;
      const key = card.getAttribute("data-key");
      if (!key) return;
      toggle(key);
    });
  }
}
```

- [ ] **Step 3: 부트스트랩 교체 — 일반 모드에서 앱 시작**

`// --- bootstrap ---` 블록의 `else { ... }` 부분만 교체:

```js
} else {
  const now = Date.now();
  STATE = applyReset(loadState(now), now);
  saveState(STATE);
  renderAll();
  installCheckHandlers();
}
```

- [ ] **Step 4: 브라우저에서 기능 확인**

`index.html` 새로고침 (쿼리 없음).

**Expected 순서대로:**
1. 헤더에 오늘(KST 기준) 날짜 표시.
2. 일일 6개 카드 + 주간 1개 카드 렌더, 진행률 `0/7 완료 0%`.
3. 아무 카드나 탭 → `done` 스타일(오펜시티 낮아지고 체크박스 녹색), 진행률 증가.
4. 새로고침 → 체크 상태 유지, 진행률 유지.
5. DevTools Application → LocalStorage → `mongil_state` 값이 현재 체크 상태를 반영.

- [ ] **Step 5: 테스트 여전히 PASS 확인**

`?test=1` 로 새로고침.

**Expected:** 16개 PASS (기능 추가로 깨진 거 없어야 함).

---

## Task 5: 치트시트 렌더 + 토글 상태 저장

**Files:**
- Modify: `/Users/baghyeonsin/RubymineProjects/mongil/index.html` — `// --- render ---` 에 `renderCheatsheet()` 추가, `renderAll()` 에서 호출, `// --- events ---` 에 toggle handler 추가.

- [ ] **Step 1: `renderCheatsheet()` 추가 (render 섹션 안)**

```js
function renderCheatsheet() {
  const el = document.getElementById("cheatsheet");
  el.open = !!STATE.cheatsheetOpen;
  const list = document.getElementById("cheatsheetList");
  list.innerHTML = CHEATSHEET.map(b => `
    <div class="boss-card">
      <div class="ico">${escapeHtml(b.icon)}</div>
      <div class="bbody">
        <div class="bname">${escapeHtml(b.name)}</div>
        <div class="tags">
          ${b.tags.map(t => `<span class="tag ${t}">${escapeHtml(TAG_LABEL[t] || t)}</span>`).join("")}
        </div>
        <div class="bnote">${escapeHtml(b.note)}</div>
      </div>
    </div>`).join("");
}
```

- [ ] **Step 2: `renderAll()` 에서 호출**

`renderAll()` 의 `// Cheatsheet handled in Task 5.` 주석을 `renderCheatsheet();` 로 교체:

```js
function renderAll() {
  renderHeader();
  renderLists();
  renderCheatsheet();
}
```

- [ ] **Step 3: 토글 상태 저장 — events 섹션에 추가**

`installCheckHandlers()` 함수 **밖**, events 섹션 하단에 추가:

```js
function installCheatsheetHandler() {
  const el = document.getElementById("cheatsheet");
  el.addEventListener("toggle", () => {
    if (!STATE) return;
    if (STATE.cheatsheetOpen === el.open) return;
    setState({ ...STATE, cheatsheetOpen: el.open });
  });
}
```

- [ ] **Step 4: 부트스트랩에서 호출 — 일반 모드 분기 갱신**

`installCheckHandlers();` 바로 아래에 `installCheatsheetHandler();` 추가:

```js
} else {
  const now = Date.now();
  STATE = applyReset(loadState(now), now);
  saveState(STATE);
  renderAll();
  installCheckHandlers();
  installCheatsheetHandler();
}
```

- [ ] **Step 5: 브라우저 확인**

`index.html` 새로고침.

**Expected:**
1. 하단에 `▶ 의뢰 치트시트` 접힘 섹션.
2. 탭하면 10개 보스 카드 펼쳐짐: 스푼머거/포크머거 2장(빨강 "필드미젠", 주황 "재료" 배지), 아몬(보라 "사기" + 노랑 "프라임합성"), 스카·보르보그·몰리(보라 "사기"), 이리도커·슬라킹·타그락·하카(파랑 "치트시트").
3. 펼친 상태에서 새로고침 → 펼침 상태 유지.
4. 접힘으로 바꾸고 새로고침 → 접힘 상태 유지.

---

## Task 6: 자동 리셋 (포커스/인터벌) + 멀티탭 동기화 + 최종 수동 QA

**Files:**
- Modify: `/Users/baghyeonsin/RubymineProjects/mongil/index.html` — events 섹션에 리스너 추가, 부트스트랩에서 호출.

- [ ] **Step 1: 자동 재평가 + 외부 storage 이벤트 핸들러 추가**

`// --- events ---` 섹션 하단에 추가:

```js
function installAutoReset() {
  function reeval() {
    if (!STATE) return;
    const now = Date.now();
    const next = applyReset(STATE, now);
    if (next !== STATE) setState(next);
  }
  // 1 min periodic
  setInterval(reeval, 60 * 1000);
  // Run immediately on tab focus / visibility regain
  window.addEventListener("focus", reeval);
  document.addEventListener("visibilitychange", () => {
    if (document.visibilityState === "visible") reeval();
  });
}

function installCrossTabSync() {
  window.addEventListener("storage", (e) => {
    if (e.key !== STORAGE_KEY) return;
    const now = Date.now();
    STATE = applyReset(loadState(now), now);
    renderAll();
  });
}
```

- [ ] **Step 2: 부트스트랩에서 호출**

일반 모드 분기에 두 줄 추가 (기존 핸들러 뒤):

```js
} else {
  const now = Date.now();
  STATE = applyReset(loadState(now), now);
  saveState(STATE);
  renderAll();
  installCheckHandlers();
  installCheatsheetHandler();
  installAutoReset();
  installCrossTabSync();
}
```

- [ ] **Step 3: 전체 테스트 한 번 더 PASS 확인**

`index.html?test=1` 새로고침 → `ALL TESTS PASSED (16 pass, 0 fail)`.

- [ ] **Step 4: 수동 QA 체크리스트**

아래를 하나씩 수행하고 표시:

- [ ] `file://` 로 `index.html` 열어 정상 로드
- [ ] 일일 6개, 주간 1개 카드 렌더
- [ ] 카드 탭 → 체크/해제 동작, 진행률 / 퍼센트 / 바 길이 실시간 갱신
- [ ] 새로고침 후 체크 상태 유지
- [ ] 치트시트 펼침/접힘 상태 유지
- [ ] 보스 10종 + 태그 배지 6색 전부 표시 확인
- [ ] 같은 파일을 다른 탭으로 하나 더 연 뒤, A 탭에서 체크 → B 탭 포커스하면 해당 카드 반영 (storage 이벤트)
- [ ] DevTools → Application → LocalStorage → `mongil_state` 에 `baselineDate` `weeklyBaselineWeek` `cheatsheetOpen` `checked` 모두 존재
- [ ] LocalStorage 에서 `mongil_state` 값을 `"{{{garbage"` 로 바꾸고 새로고침 → 에러 없이 기본값으로 초기화
- [ ] 시스템 시각을 내일 09:01 로 바꾼 뒤 새로고침 → 일일 체크가 모두 풀림, 주간 체크는 유지 (주 경계 전이라면)
- [ ] 모바일 Chrome (DevTools Device Mode — iPhone 12 Pro) 에서 카드 터치 영역 충분, 글자 가독 OK
- [ ] 에러 로그 없음 (Console 탭)

- [ ] **Step 5: (선택) 사용자가 git 관리 원하면 초기화**

사용자 선택. 원한다면:

```bash
cd /Users/baghyeonsin/RubymineProjects/mongil
git init
git add -A
git commit -m "feat: mongil MVP — daily/weekly checklist with KST 09:00 reset"
```

---

## 완성 후 배포 옵션 (사용자 결정)

MVP 동작 확인 후 고를 수 있는 배포 방식 — 구현 완료 후 사용자에게 제시:

1. **로컬만 사용** — 파일 브라우저 북마크
2. **iCloud Drive / Google Drive 동기화** — 여러 기기에서 같은 `index.html` 열기 (단, localStorage 는 브라우저별로 분리)
3. **GitHub Pages** (5분) — `git init` → GitHub 리포 푸시 → Pages 활성화 → `https://<user>.github.io/mongil/` 북마크

---

## 자체 검토 결과 (self-review)

- Spec § 1~11 모두 Task 에 매핑됨:
  - § 2 단일 파일 → Task 1
  - § 3 스토리지 / § 4 기준일 로직 → Task 2, Task 3
  - § 5 일일/주간 항목 / § 6 치트시트 → Task 3 (데이터), Task 4, Task 5 (렌더)
  - § 7 UI / 진행률 → Task 1 (스타일), Task 4 (진행률)
  - § 8 엣지케이스(손상 storage, 멀티탭, 오래 열어둠) → Task 3 (loadState fallback + 테스트), Task 6 (storage 이벤트 + interval)
  - § 10 수동 QA → Task 6 Step 4
- 플레이스홀더 없음. 각 스텝에 실제 코드/명령/기대값 포함.
- 타입 일관성: `STATE` 객체 필드(`baselineDate`, `weeklyBaselineWeek`, `cheatsheetOpen`, `checked`)가 Task 3 정의 이후 계속 동일한 이름으로 사용됨.
- 함수 이름 일관: `computeBaselineDate` / `computeIsoWeekKst` / `loadState` / `saveState` / `applyReset` / `setState` / `renderAll` 전체 흐름 일관.
