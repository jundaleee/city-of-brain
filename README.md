# City of Brain

자기주도학습 워크벤치의 상용화 버전. 자매 프로젝트 [Sorganizer(dshs-organizer)](https://github.com/jundaleee/dshs-organizer)에서 갈라져 나온 공개 서비스로, 단일 HTML 파일(`index.html`)로 동작하는 바닐라 JS SPA — 프레임워크 없음, 빌드 스텝 없음. Supabase 인증, 클라우드 동기화, AdSense 광고가 붙은 실제 배포 서비스라는 점이 dshs-organizer와 다름.

> 이 문서는 다음에 이 코드를 이어받을 AI 에이전트를 위해 쓰였음. 배포 방식, 데이터 모델, 이번 세션의 대폭 축소/개편 배경을 최대한 상세히 적어둠.

## 배포

**https://cityofbrain.org** (커스텀 도메인, `CNAME` 파일로 GitHub Pages에 연결) 로 서비스 중. `main` 브랜치에 푸시하면 자동 배포됨.

- 로그인은 Supabase(이메일/비밀번호, Google OAuth, 익명 로그인 모두 지원) — **로그인은 선택**이지 게이트가 아님. 비로그인 상태에서도 `localStorage`만으로 전 기능이 동작하고, 로그인하면 `app_data` 테이블에 계정별로 클라우드 동기화됨(`pushCloudData`/`pullCloudData`, 600ms 디바운스). `persist()`는 항상 로컬에 먼저 쓰고, 세션이 있을 때만 클라우드에 얹는 식이라 오프라인에서도 안전.
- 광고: 사이드바에 AdSense 슬롯(`ca-pub-3167268253626557`, 실제 퍼블리셔 ID). 페이지를 다시 그릴 때마다 `window.adsbygoogle`을 다시 push해야 광고가 갱신됨 — `render()` 안에 이 로직이 있으니 렌더 시스템을 건드릴 때 실수로 지우지 말 것.
- 과목 박스는 고정 로스터가 아니라 **사용자가 직접 이름 지어 추가**함(`state.data.boxes`, `submitAddBox()`).

## 이번 세션: 중간고사 벼락치기용 대폭 축소 개편

사용자가 중간고사 주간을 앞두고 "과제/아카이브/도시 탭 다 없애고, 무지개색 없애고 블루·화이트·블랙·글래스만 써라, 그리고 과목별로 이번 주에 끝낼 것들을 적어두면 끝낸 만큼 음식(길쭉한 모양만)을 먹어치우는 걸 만들어라"고 명시적으로 요청한 결과. **이전 세션에 한 번 "무채색 전체 변환"을 과하게 해석해서 사용자에게 되돌리라는 지적을 받은 적이 있음** — 이번엔 지시받은 범위(탭 3개 삭제 + 색상 제약 + 신규 기능 1개)를 넘어서지 않으려 했음.

### 제거된 것

- **과제(Assignments) 탭 전체**: `renderAssignmentsPage`, `assignmentRowHtml`, `addAssignment`/`toggleAssignment`/`deleteAssignment`/`saveAssignmentEdit`, 우선순위(`PRIORITY_LABELS`/`PRIORITY_RANK`), D-day 역산 타임라인(`renderDdayTimeline` — 이름이 비슷해 헷갈리기 쉬운데 **과제 마감일 타임라인이지 디데이 카드와는 다른 기능**이었음, 디데이 카드는 그대로 생존), `state.data.assignments` 데이터 자체. 스톱워치 연결 대상에서도 `type:'assignment'` 분기 제거.
- **아카이브 탭**: `renderArchivePage`, `computeArchiveStats`, `exportArchiveLog`, `downloadTextFile`, `RATING_LABELS`. **단, `archiveLog` 원자료와 `logArchiveEvent()`/`findAnyCompletedItem()`은 그대로 남겨둠** — 완료 피드백 대화창과 "마지막 완료 취소"(`undoLastCompletion`)가 이 원자료를 그대로 쓰기 때문에, 탭(뷰)만 지우고 데이터 파이프라인은 건드리지 않음.
- **도시 탭 + 3D 씬 전체**: `CITY3D` 네임스페이스와 그 아래 전부(씬 빌드/루프/레이캐스팅/툴팁/카메라 트윈 등), `renderCityPage`/`renderCityGrowthBar`/`cityGrowth`/`renderCityInspectModal`/`renderGraduationArchive`/`collectCityCitizens` 등 시각화 데이터 함수, 홈 화면 우측의 "살아있는 도시 미리보기"(`home-split`/`home-city`, `#city3dSlotHome`)까지 전부 제거. `isWideScreen()`과 그걸 쓰던 resize 리스너도 2단 레이아웃이 사라지면서 같이 삭제. vendored three.js(`assets/vendor/three-city.min.js`)와 GLB/텍스처 에셋(`assets/city/`)도 저장소에서 삭제(2.3MB 절약) — `git rm`으로 지웠으니 필요하면 git history에서 복구 가능.
- **색상**: 8개 accent 변수(`--red/--green/--orange/--purple/--pink/--indigo/--teal`)를 전부 삭제하고 `--blue` 하나만 남김. 과목 박스 팔레트(`BOX_PALETTE`)도 무지개 로테이션 대신 블루/화이트 글로우 3종 로테이션으로 축소. 사이드바 브랜드 마크도 7색 conic-gradient → 블루→화이트 linear-gradient로. 남은 구글 로그인 버튼의 "G" 로고 4색(`#4285F4` 등)은 **구글 브랜드 가이드라인상 건드리면 안 되는 부분**이라 그대로 둠.

**사고 당시 발견한 실수**: 할 일 페이지 관련 정리를 orphan 코드 삭제로 착각해서 `itemStaleDays`/`isEditingTask`/`taskEditRowHtml`/`taskRowHtml`/`setTaskAddType`/`addTaskUnified`/`renderTasksPage`를 통째로 같이 지웠던 적이 있음 — git의 원본 커밋과 `function 이름(` 목록을 diff해서 전부 찾아 복구함. **대량 삭제(sed로 통짜 라인 범위 지우기)를 할 땐 그 범위 안을 전부 눈으로 읽고 나서 지울 것** — 함수 경계를 추정만 하고 지우면 이런 사고가 또 난다.

### 새로 추가된 것: "이번주" 탭 — 과목별 간식 게이지

사용자 요구사항 원문 요약: "계획을 세운다기보다 뭘 끝내야 되는지 다 적어놓고 끝낸 만큼 게이지가 차는 것". 음식에 비유해서 끝낸 만큼 그 과목의 간식을 먹어치우는 형태로, **길쭉해서 길이로 늘어나는 것처럼 보이는 모양의 스프라이트만** 쓰라고 명시함(둥근 과일 등은 쓰지 말 것).

- **에셋**: 사용자가 첨부한 Kenney `pixel-platformer-food-expansion` 팩(18×18px 타일시트, CC0)에서 가로로 긴 9종만 잘라냈음 — 초콜릿 바, 딸기 바, 그래엄 크래커 바, 치즈 바, 소시지, 글레이즈드 소시지, 서브 샌드위치, 클럽 샌드위치, 콘도그. 전부 base64 PNG로 `FOOD_SPRITES` 상수에 인라인 임베드(원본 타일시트 전체를 합쳐도 5KB 미만이라 별도 에셋 파일을 안 만들고 그대로 박아넣음 — 이 앱의 "단일 HTML 파일" 원칙 유지). 음식 이미지 자체는 원래 색 그대로 둠(블루/화이트/블랙 제약은 UI 크롬에만 적용, 음식 일러스트는 예외).
- **데이터 모델**: `state.data.examWeek = {subjects:[{id, name, food, items:[{id, text, done}]}]}`. `food`는 `FOOD_ORDER` 배열에서 과목 추가 순서대로 로테이션 배정(`FOOD_ORDER[subjects.length % FOOD_ORDER.length]`).
- **초기 설정 흐름**: "+ 과목 추가" → 과목 이름 입력 + 텍스트에어리어에 이번 주에 끝낼 것들을 줄바꿈으로 구분해서 한 번에 입력 → 제출하면 그 줄 수만큼 `items` 배열이 바로 생기고, 블록 수 = 그 항목 개수(사용자가 명시한 "각 과목마다 간식의 기본값 세팅은 항목 개수만큼의 블록" 그대로 구현). 이후에도 과목 카드 하단의 인라인 입력으로 항목을 더 추가할 수 있고, 블록 수는 `items.length`를 매번 다시 세서 그리므로 자동으로 늘어남(별도 동기화 로직 불필요 — 늘 원자료에서 다시 계산).
- **"먹어치우기" 렌더링** (`.food-bar`): 배경 이미지 슬라이싱 같은 복잡한 짓 안 하고, 세 레이어를 겹치는 방식으로 단순화함:
  1. `.food-bar-img` — 음식 스프라이트를 바 전체 너비로 늘려서(`background-size:100% 100%`) 꽉 채움. `image-rendering:pixelated`로 픽셀아트 느낌 유지.
  2. `.food-bar-blocks` — `repeating-linear-gradient`로 `var(--n)`(항목 개수) 등분한 얇은 구분선을 올려서 "블록"이 보이게 함.
  3. `.food-bar-eaten` — 완료 비율(`done/total*100%`)만큼 왼쪽부터 `var(--bg)`(어두운 배경색)로 덮어서 그만큼 "먹힌" 것처럼 보이게 함. 폭이 바뀔 때 `transition:width`로 부드럽게 줄어듦.
  이 방식은 각 블록이 정확히 몇 번째 항목에 대응하는지는 추적하지 않고(먹힌 개수만 왼쪽부터 누적) — 사용자가 순서 상관없이 아무 항목이나 체크해도 "진행률"이라는 의미는 그대로 전달됨.
- **다른 탭과의 관계**: 기존 "할 일"(`renderTasksPage`, 과목 박스별 수업 과제/자율학습)과는 **완전히 별개의 독립된 기능**임 — 데이터도 안 섞이고 서로 참조하지 않음. 사용자가 "과제/아카이브/도시 탭만 지워라"고 했지 할 일 탭을 이 기능으로 대체하라곤 안 했으므로, 둘 다 나란히 살려둠(내비게이션: 홈 → 할 일 → 이번주 → 설정).

### 남은 탭/기능 (전부 생존, 이번 세션에 구조 변경 없음 — 색상만 블루로 정리)

홈(사용자 정의 과목 박스 + 온보딩 가이드 + 디데이 카드 + 스톱워치/뽀모도로), 할 일(수업 과제/자율학습 통합 입력 + 길게 눌러 휴지통 드래그 삭제), 완료 피드백 대화창(4단계 자기평가 + 자유 서술, `archiveLog`에 기록), 완료 실행취소(`undoLastCompletion`), 알림 벨(디데이 D-7만 — 과제 마감 알림은 과제 탭과 함께 제거됨), Supabase 인증/클라우드 동기화, AdSense, 보고서 템플릿(`getTemplates()`).

## 코드 구조

`index.html` 하나(`<style>` + `<script>`, IIFE 하나로 감쌈). 대략 순서:

1. `state` 객체 + `buildSkeleton()` (데이터 모델: `tasks`, `completed`, `ddays`, `boxes`, `archiveLog`, `examWeek`)
2. Supabase 클라이언트 초기화 + 인증 흐름 + 클라우드 동기화(`pushCloudData`/`pullCloudData`)
3. `localStorage` 저장/로드 (`loadData`/`persist`)
4. 아이콘(`icon(name)`), `NAV_ITEMS`(홈/할일/이번주), 렌더 시스템(`render()`, `SCROLL_CHAIN=['home','tasks','exam']`, 사이드바/바텀내브)
5. 홈(`renderHome`) / 할 일(`renderTasksPage`) 페이지 렌더 함수
6. 완료 피드백 대화창, 아카이브 로그(`logArchiveEvent`), 스톱워치/뽀모도로
7. **이번주 탭** — `FOOD_SPRITES`/`FOOD_ORDER` 상수, `addExamSubject`/`deleteExamSubject`/`addExamItem`/`toggleExamItem`/`deleteExamItem`, `renderExamPage`/`examSubjectCardHtml`
8. 이벤트 위임 디스패처(`document.body`에서 `data-action` 기반 `switch`)
9. `init()` — 로컬/클라우드 데이터 로드 후 `render()`

**렌더 패턴**: 상태가 바뀌면 `render()`가 `#app.innerHTML`을 통째로 새로 씀. 오버레이(모달/알림/피드백독)만 여닫을 땐 `renderOverlaysOnly()`로 가볍게 처리.

## 데이터 구조 (localStorage 로컬 + Supabase `app_data` 테이블 클라우드)

```json
{
  "tasks": [ { "id": "", "text": "", "boxKey": null, "corner": "review|self", "createdAt": 0 } ],
  "completed": [ { "id": "", "text": "", "boxKey": null, "corner": "review|self", "completedAt": 0,
                    "recallRating": "AGAIN|HARD|GOOD|EASY", "feedbackNote": "", "feedbackAt": 0 } ],
  "ddays": [ { "id": "", "name": "", "date": "YYYY-MM-DD", "hidden": false } ],
  "boxes": [ { "key": "", "label": "", "qc": "var(--blue)" } ],
  "archiveLog": [ { "id": "", "ts": 0, "createdAt": 0, "subject": "", "subjectKey": "", "kind": "review|self",
                     "text": "", "rating": "AGAIN|HARD|GOOD|EASY", "note": "" } ],
  "examWeek": { "subjects": [ { "id": "", "name": "", "food": "choc|straw|graham|cheese|sausage|glazed|sub|club|corndog",
                                 "items": [ { "id": "", "text": "", "done": false } ] } ] }
}
```

`boxes`는 사용자가 런타임에 추가/삭제하는 배열 — `boxKey`는 그 배열의 `key` 값 또는 `null`(미분류). `examWeek`는 `tasks`/`boxes`와 전혀 연결되지 않는 독립 데이터.

## 검증 방법

- **문법 체크**: `node -e "new Function(require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/g).map(s=>s.replace(/<\/?script>/g,'')).join('\n'))"`
- **함수 목록 diff** (대량 삭제 후 필수): `git show HEAD:index.html | grep -oE "function [a-zA-Z0-9_]+\(" | sed 's/function //;s/(//' | sort -u` 를 수정 전/후로 각각 뽑아서 `comm -23`으로 비교 — "지우려던 것"과 "실제로 사라진 것"이 정확히 일치하는지 반드시 확인할 것(이번 세션에 이걸 안 하고 넘어갔다가 Tasks 페이지 함수를 같이 날려먹었음).
- **로컬 서빙 + Playwright**: `python3 -m http.server <port>` 로 띄운 뒤 `NODE_PATH=/opt/node22/lib/node_modules node <script>.js`.

## 남은 작업 / 다음 단계 후보

- 이번주 탭에 과목/항목 수정(이름 변경, 항목 텍스트 수정) UI는 아직 없음 — 삭제 후 재입력만 가능.
- "이번주" 데이터를 주차 단위로 보관하거나 리셋하는 기능 없음 — 다음 시험 기간엔 사용자가 기존 과목을 직접 지우고 새로 추가해야 함.
- AI 기능을 상용 버전에 어떻게 들여올지: 사용자별 API 키 입력 방식은 다중 사용자 서비스에 안 맞음. 서버 프록시부터 설계해야 함(이번 세션과 무관하게 여전히 미해결).
