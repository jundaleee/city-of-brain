# City of Brain

자기주도학습 워크벤치의 상용화 버전. 자매 프로젝트 [Sorganizer(dshs-organizer)](https://github.com/jundaleee/dshs-organizer)의 개인용 프로토타입에서 갈라져 나온 공개 서비스로, 단일 HTML 파일(`index.html`)로 동작하는 바닐라 JS SPA — 프레임워크 없음, 빌드 스텝 없음. Supabase 인증, 클라우드 동기화, AdSense 광고가 붙은 실제 배포 서비스라는 점이 dshs-organizer와 다름.

> 이 문서는 다음에 이 코드를 이어받을 AI 에이전트를 위해 쓰였음. dshs-organizer와의 관계, 이번 대규모 포팅 세션에서 옮겨온 기능, 의도적으로 갈라진 부분, 데이터 모델을 최대한 상세히 적어둠.

## 배포

**https://cityofbrain.org** (커스텀 도메인, `CNAME` 파일로 GitHub Pages에 연결) 로 서비스 중. `main` 브랜치에 푸시하면 자동 배포됨.

- 로그인은 Supabase(이메일/비밀번호, Google OAuth, 익명 로그인 모두 지원) — **로그인은 선택**이지 게이트가 아님. 비로그인 상태에서도 `localStorage`만으로 전 기능이 동작하고, 로그인하면 `app_data` 테이블에 계정별로 클라우드 동기화됨(`pushCloudData`/`pullCloudData`, 600ms 디바운스). `persist()`는 항상 로컬에 먼저 쓰고, 세션이 있을 때만 클라우드에 얹는 식이라 오프라인에서도 안전.
- 광고: 사이드바에 AdSense 슬롯(`ca-pub-3167268253626557`, 실제 퍼블리셔 ID). 페이지를 다시 그릴 때마다 `window.adsbygoogle`을 다시 push해야 광고가 갱신됨 — `render()` 안에 이 로직이 있으니 렌더 시스템을 건드릴 때 실수로 지우지 말 것.
- 과목 박스는 dshs-organizer처럼 고정 로스터가 아니라 **사용자가 직접 이름 지어 추가**함(`state.data.boxes`, `submitAddBox()`) — 학교/학년마다 과목 구성이 다른 여러 사용자를 상대해야 하니 하드코딩 로스터를 쓸 수 없음.

## dshs-organizer와의 관계

이 저장소는 dshs-organizer를 다음 두 단계로 단순화한 뒤 상용 기능(인증/동기화/광고/도메인)을 얹어서 시작함:

1. 선생님(Teacher) 탭·시간표·Classroom 동기화·자료 탭 전체 제거 — 학교마다 선생님/시간표 구조가 다르니 범용 서비스엔 안 맞음.
2. **AI 기능 전부 제거**(망각의 시민 생성, 자연어 자동분류, AI 주간 리포트) — 사용자별 Anthropic API 키를 직접 입력받는 dshs-organizer 방식은 다중 사용자 상용 서비스에 안 맞고(키 관리/과금/백엔드 프록시 문제), 이번 세션에서도 **의도적으로 배제**함. 나중에 붙이려면 서버 프록시(사용자가 자기 키를 안 넣어도 되는 구조)부터 설계해야 함.

이후 이 저장소만의 길을 감: 계정 시스템, 클라우드 동기화, 사용자 정의 과목 박스, 광고, 자체 3D 도시(차/사람 메타포 — dshs-organizer의 차/집 메타포와 다름), 온보딩 가이드(`renderOnboardGuide()`, 박스가 하나도 없을 때 표시), 보고서 템플릿(`getTemplates()`/`DEFAULT_TEMPLATES` — 물리/화학 보고서 양식, C++ 기본 세팅 등 미리 채워둔 텍스트 스니펫. AI 리포트와는 무관한 별개 기능).

## 이번 세션: dshs-organizer 대규모 기능 포팅 (AI 제외, 타워디펜스 없음)

dshs-organizer 쪽에서 먼저 검증된 비-AI 기능들을 이 저장소에 한 번에 이식함. 포팅 원칙: **city-of-brain이 독자적으로 개발한 부분(인증, 동기화, 사용자 정의 박스, 광고, 차/사람 3D 메타포, 온보딩 가이드, 보고서 템플릿)은 그대로 두고 덮어쓰지 않음.**

- **진짜 스크롤 체인 내비게이션**: `SCROLL_CHAIN = ['home','tasks','assignments','archive','city']` 다섯 섹션을 한 번에 이어붙여서 `render()`가 그리고, 네이티브 브라우저 스크롤 + `updateActiveSectionFromScroll()`(IntersectionObserver 대신 `getBoundingClientRect().top` 직접 스캔)로 스크롤스파이 처리. 자세한 배경·겪었던 버그(듀얼 슬롯 `city3dSlot`/`city3dSlotHome` id 충돌 등)는 dshs-organizer README의 "스크롤 내비게이션" 절 참고 — 이 저장소도 같은 원인으로 같은 버그가 생길 뻔해서 `pickCityHostSlot()`으로 사전에 막아둠.
- **아카이브 탭 + `archiveLog`**: 완료할 때마다 append-only 배열(`state.data.archiveLog`)에 `{id, ts, createdAt, subject, subjectKey, kind, text}`를 기록(`logArchiveEvent()`). `computeArchiveStats()`가 매번 이 원자료에서 통계를 다시 계산(과목별 현황, 최근 완료 기록). **dshs-organizer의 캘리브레이션/강점-약점 판정은 옮기지 않음** — 그건 망각의 시민(AI) 항목이 간격을 두고 반복 노출되는 구조가 있어야 계산되는데, AI를 뺐으므로 그 데이터 소스 자체가 없음.
- **막지 않는 완료 피드백 대화창**(`.feedback-dock`): 완료 시 뜨는 4단계 자기평가(거의 기억 안 남/가물가물했음/잘 기억남/완벽했음) + 자유 서술. `state.modal`이 아니라 `state.feedbackTargetId`로만 생사가 결정돼서 다른 모달을 열어도 안 사라짐. **등급 버튼 클릭은 선택만 하고 제출 안 함** — 별도 "제출" 버튼을 눌러야 확정(`selectFeedbackRating()` vs `submitCompletionFeedback()` 2단계 분리). dshs-organizer에서 실사용 중 "등급 누르자마자 창이 닫혀서 쓰던 텍스트가 날아갔다"는 버그를 겪고 고친 결과물이라, 처음부터 고쳐진 버전으로 이식함.
- **완료 실행취소(1단계)**: `lastCompletion`에 가장 최근 완료 1건의 스냅샷을 담아뒀다가 `undoLastCompletion()`으로 되돌림. 알림 벨의 "마지막 완료 취소" 섹션에 노출. `archiveLog`에서도 해당 항목을 지움 — append-only 원칙에 대한 의도적이고 유일한 예외(오조작 교정 목적).
- **알림 벨**: 우상단 버튼이 설정 대신 알림 벨로 동작(`renderTopbarMini`, `computeNotifications()`). 디데이 D-7, 과제 마감 3일/1일 전에만 좁게 알림. **AI 주간 리포트 버튼은 없음** — dshs-organizer엔 있지만 AI 기능이라 이식 대상에서 제외.
- **뽀모도로 모드**: 스톱워치 카드 안에서 `swMode`(`'stopwatch'`|`'pomodoro'`)로 전환. 25분 공부/5분 휴식 고정 한 주기, 설정 옵션 없음. 항목 연결(드래그)은 두 모드가 공유.
- **스톱워치 항목 연결 드롭다운 제거**: 예전엔 `<select id="swTargetSelect">`로 항목을 골라 연결했는데, 다크 테마에서 흰 배경으로 뜨는 네이티브 select 팝업이 어색해서 제거하고 **드래그 연결 하나로 통일**.
- **할 일 추가 행 통합**: "수업 과제"/"자율학습" 두 섹션에 따로 있던 추가 입력 행을, 탭 토글(`taskAddType`) 하나 + 피커/입력/버튼 한 줄로 합침(`addTaskUnified()`). 데이터 모델 자체는 그대로(`corner: 'review'|'self'`) — 입력 UI만 하나로 합친 것.
- **홈 할 일 카드 길게 눌러서 삭제**: `.box-task-row`를 550ms 이상 누르고 있으면 우하단에 휴지통이 뜸(`#taskTrashZone`). 순수 `pointerdown`+타이머로 휴지통 노출만 게이팅하고, 실제 삭제는 기존 네이티브 HTML5 드래그(`dragstart`/`dragover`/`drop`)에 얹음 — 그래서 빠르게 휙 끌어 스톱워치 카드에 놓는 기존 연결 동작은 그대로 유지됨.
- **도시 성장 바**: 도시 탭 맨 위에 `renderCityGrowthBar()` — `archiveLog.length` 기준으로 12개 완료마다 1레벨(최대 10), 진행률 바 + "OO개 더 하면 한 단계 자람" 문구. **dshs-organizer처럼 3D 씬의 녹지 밀도를 실제로 바꾸는 데까지는 안 감** — 이 저장소는 이미 독자적인 차/사람 3D 씬(`cityLandmarkLevel()`이 완료 개수로 랜드마크 건물을 키우는 로직)이 있어서, 그 씬을 다시 설계하지 않고 순수 UI 레이어(진행률 바)만 가볍게 얹음.
- **"이번 주 소홀한 과목" 경고 제거**(`renderBalanceWarning`/`weekBoxCompletionCounts`/`.balance-warn`): dshs-organizer에서 이미 제거하기로 한 결정을 그대로 반영. 오렌지 경고색 텍스트로 압박하는 톤이 "성장" 프레이밍과 안 맞는다고 판단함.
- `color-scheme: dark`를 `:root`에 추가 — 네이티브 `<select>` 팝업·날짜 picker가 라이트 배경으로 뜨던 문제 해결(이 앱은 다크 모드 전용).

**명시적으로 이식하지 않은 것**: AI 기반 기능 전부(망각의 시민, 자연어 자동분류, 주간 리포트), SM-2 간격반복 스케줄러, 캘리브레이션 지표(둘 다 AI 재노출 없이는 의미가 없음), 타워 디펜스류 게임 메커니즘(사용자가 명시적으로 불필요하다고 확인함).

## 코드 구조

`index.html` 하나(`<style>` + `<script>`, IIFE 하나로 감쌈). 대략 순서:

1. `state` 객체 + `buildSkeleton()` (데이터 모델: `tasks`, `completed`, `assignments`, `ddays`, `boxes`, `archiveLog`)
2. Supabase 클라이언트 초기화 + 인증 흐름(이메일/비밀번호, Google OAuth, 익명 로그인) + 클라우드 동기화(`pushCloudData`/`pullCloudData`)
3. `localStorage` 저장/로드 (`loadData`/`persist`)
4. 아이콘(`icon(name)`), `NAV_ITEMS`, 렌더 시스템(`render()`, 스크롤 체인, 사이드바/바텀내브)
5. 홈(`renderHome`, 사용자 정의 과목 박스 + 온보딩 가이드) / 할 일 / 과제 / 아카이브 / 도시 페이지 렌더 함수
6. 도시(City) 3D 씬 (`CITY3D` 네임스페이스, vendored three.js + GLB 에셋, 차/사람 메타포)
7. 완료 피드백 대화창, 아카이브 로그, 스톱워치/뽀모도로
8. 이벤트 위임 디스패처(`document.body`에서 `data-action` 기반 `switch`)
9. `init()` — 로컬/클라우드 데이터 로드 후 `render()`

**렌더 패턴**은 dshs-organizer와 동일: 상태가 바뀌면 `render()`가 `#app.innerHTML`을 통째로 새로 씀. 오버레이(모달/알림/피드백독)만 여닫을 땐 `renderOverlaysOnly()`로 가볍게 처리(본문 데이터가 바뀌는 액션엔 쓰면 안 됨).

## 데이터 구조 (localStorage 로컬 + Supabase `app_data` 테이블 클라우드)

```json
{
  "tasks": [ { "id": "", "text": "", "boxKey": null, "corner": "review|self", "createdAt": 0 } ],
  "completed": [ { "id": "", "text": "", "boxKey": null, "corner": "review|self", "completedAt": 0,
                    "recallRating": "AGAIN|HARD|GOOD|EASY", "feedbackNote": "", "feedbackAt": 0 } ],
  "assignments": [ { "id": "", "title": "", "dueDate": "YYYY-MM-DD", "priority": "high|mid|low", "done": false } ],
  "ddays": [ { "id": "", "name": "", "date": "YYYY-MM-DD", "hidden": false } ],
  "boxes": [ { "key": "", "label": "", "qc": "var(--blue)" } ],
  "archiveLog": [ { "id": "", "ts": 0, "createdAt": 0, "subject": "", "subjectKey": "", "kind": "review|self",
                     "text": "", "rating": "AGAIN|HARD|GOOD|EASY", "note": "" } ]
}
```

`boxes`는 dshs-organizer처럼 고정 상수가 아니라 사용자가 런타임에 추가/삭제하는 배열 — `boxKey`는 그 배열의 `key` 값 또는 `null`(미분류).

## 검증 방법

- **문법 체크**: `node -e "new Function(require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/g).map(s=>s.replace(/<\/?script>/g,'')).join('\n'))"`
- **로컬 서빙 + Playwright**: `python3 -m http.server <port>` 로 띄운 뒤 `NODE_PATH=/opt/node22/lib/node_modules node <script>.js`, `chromium.launch({executablePath:'/opt/pw-browsers/chromium', args:['--use-gl=angle','--use-angle=swiftshader','--enable-unsafe-swiftshader']})` (3D 도시 화면에 WebGL 필요, SwiftShader 소프트웨어 렌더러 사용).
- 스크롤 관련 기능은 `page.mouse.wheel()`로 실제 휠 스크롤을 흉내내야 "진짜 스크롤인지" 검증됨. 드래그 관련 기능(스톱워치 연결, 휴지통 삭제)은 헤드리스 Chromium에서 `mouse.move/down/up`만으로는 네이티브 HTML5 `dragstart`/`drop` 이벤트가 안 뜨는 경우가 있어서, `new DataTransfer()` + 수동 `DragEvent` 디스패치로 로직을 검증함.

## 남은 작업 / 다음 단계 후보

- **AI 기능을 상용 버전에 어떻게 들여올지**: 사용자별 API 키 입력 방식은 다중 사용자 서비스에 안 맞음. 서버 프록시(자체 백엔드가 Anthropic API를 대신 호출하고 사용량을 계정별로 제한/과금)를 먼저 설계해야 함.
- **도시 3D 성장 연동 심화**: 지금은 성장 바(UI)와 랜드마크 레벨(`cityLandmarkLevel()`, `completed.length` 기준)이 서로 다른 소스를 씀 — 나중에 하나로 통합하거나, dshs-organizer처럼 녹지 밀도까지 성장에 연동할지는 3D 씬을 더 들여다보고 결정할 것.
- 두 저장소(dshs-organizer/city-of-brain)가 계속 나란히 발전 중이므로, 한쪽에 새 비-AI 기능이 생기면 이쪽에도 포팅할지 검토할 것 — 단, 이 저장소만의 독자 기능(인증/동기화/사용자 정의 박스/광고/차·사람 메타포)은 절대 덮어쓰지 말 것.
