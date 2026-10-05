# City of Brain

자기주도학습 워크벤치의 상용화 버전. 자매 프로젝트 [Sorganizer(dshs-organizer)](https://github.com/jundaleee/dshs-organizer)에서 갈라져 나온 공개 서비스로, 단일 HTML 파일(`index.html`)로 동작하는 바닐라 JS SPA — 프레임워크 없음, 빌드 스텝 없음. Supabase 인증, 클라우드 동기화, AdSense 광고가 붙은 실제 배포 서비스라는 점이 dshs-organizer와 다름.

> 이 문서는 다음에 이 코드를 이어받을 AI 에이전트를 위해 쓰였음. 배포 방식, 데이터 모델, 앱이 지금 이 모양이 된 배경을 최대한 상세히 적어둠.

## 배포

**https://cityofbrain.org** (커스텀 도메인, `CNAME` 파일로 GitHub Pages에 연결) 로 서비스 중. `main` 브랜치에 푸시하면 자동 배포됨.

- 로그인은 Supabase(이메일/비밀번호, Google OAuth, 익명 로그인 모두 지원) — **로그인은 선택**이지 게이트가 아님. 비로그인 상태에서도 `localStorage`만으로 전 기능이 동작하고, 로그인하면 `app_data` 테이블에 계정별로 클라우드 동기화됨(`pushCloudData`/`pullCloudData`, 600ms 디바운스). `persist()`는 항상 로컬에 먼저 쓰고, 세션이 있을 때만 클라우드에 얹는 식이라 오프라인에서도 안전.
- 광고: 상단에 AdSense 배너 슬롯(`ca-pub-3167268253626557`, 실제 퍼블리셔 ID). 페이지를 다시 그릴 때마다 `window.adsbygoogle`을 다시 push해야 광고가 갱신됨 — `render()` 안에 이 로직이 있으니 렌더 시스템을 건드릴 때 실수로 지우지 말 것.

## 지금 이 앱은 기능이 하나뿐임 — "이번주" 주문서/간식 게이지

**이 앱은 원래 홈(과목 박스) / 할 일 / 과제 / 아카이브 / 도시(3D 씬) 등 여러 탭을 가진 범용 학습 워크벤치였다.** 두 차례에 걸친 사용자 요청으로 지금은 **"이번주" 기능 하나만 남기고 나머지 전부를 삭제**한 상태:

1차 축소(중간고사 벼락치기 준비): 과제/아카이브/도시 탭 삭제, 무지개 색상 팔레트를 블루 하나로 축소, "이번주" 과목별 간식 게이지 기능 신설.
2차 축소(이번 세션): 사용자가 "영수증 빼고 다른 기능 다 지워"라고 명시적으로 요청 — **홈, 할 일, 디데이, 스톱워치/뽀모도로, 완료 피드백 대화창, 알림 벨, 보고서 템플릿, 과목 박스, 좌측 사이드바/상단 다중 탭 내비게이션을 전부 삭제**하고 "이번주" 영수증 카드 하나만 남김. 같은 요청에서 글래스(블러) 효과도 전부 폐기(CSS 변수 `--blur`를 아예 삭제해서 `backdrop-filter`가 전부 무효화되게 함)하고, 배경을 밝고 따뜻한 크림색(`#FFF8EE`)으로 바꾸고, 왼쪽 탭도 없앴음.

**다음에 또 기능을 줄이거나 늘려달라는 요청이 오면, 지금 이 설명이 아니라 실제로 남아있는 코드를 기준으로 판단할 것** — 이 섹션은 역사 기록이지 현재 스펙이 아님.

### 대량 삭제 작업 방법 (이번 세션에 실제로 쓴 절차)

2178줄짜리 파일에서 대부분을 지워야 했던 작업이라, 과거(1차 축소) 세션에 `sed`로 통짜 라인 범위를 지우다 서로 다른 기능이 물리적으로 섞여 있던 걸 못 보고 Tasks 페이지 함수까지 같이 날려먹은 사고가 있었음. 이번엔 그 교훈을 반영해서:

1. 먼저 `grep -n "^  function "` 등으로 전체 함수 목록과 줄 번호를 뽑아 지도를 만들고, 각 함수를 실제로 `Read`로 읽어서 "어느 기능에 속하는지", "다른 살아남는 기능이 이 함수를 참조하는지"를 전부 확인한 뒤에 삭제 범위를 확정함(예: `.task-del` CSS 클래스는 Tasks 페이지용이었지만 이번주 카드의 삭제 버튼도 재사용하고 있어서 규칙 자체는 살리고 Tasks 전용 규칙만 골라 지움).
2. 삭제는 Python 스크립트로, "시작 마커 문자열"과 "끝 마커 문자열"(둘 다 파일에서 유일해야 함, 스크립트가 `assert`로 검증)을 지정해 그 사이를 통째로 잘라내는 방식을 씀 — 하드코딩된 줄 번호를 쓰지 않아서 이전 삭제가 뒤쪽 줄 번호를 밀어내도 안전함.
3. 삭제 후 `grep -c`로 지웠다고 생각한 함수/변수 이름이 **호출부에도** 안 남아있는지 전수 조사(대시보드/디스패처/키다운 리스너/설정 모달처럼 여러 함수를 한 군데서 호출하는 코드는 함수 본체를 지워도 호출부가 그대로 남아 `ReferenceError`를 내기 쉬움).
4. CSS도 똑같이: HTML 템플릿 문자열에 쓰인 모든 `class="..."` / `classList.add(...)` 를 정규식으로 긁어서 CSS에 정의가 없는 클래스가 있는지, 반대로 CSS에만 있고 아무 데서도 안 쓰는 선택자가 있는지 교차 검증함.
5. 마지막으로 Playwright로 실제 브라우저에서 전체 플로우(과목 추가 → 항목 추가/완료/삭제 → 설정 저장 → 새로고침 후 데이터 유지 확인)를 돌려 콘솔 에러가 없는지 확인.

**앞으로 또 대량 삭제를 할 땐 이 5단계를 그대로 따를 것.**

## "이번주" 기능 상세

사용자 요구사항 원문 요약: 과목마다 "레시피 종이"(영수증/주문서)를 하나씩 받았다고 생각하고, 이번 주에 끝낼 항목을 적어두면 끝낸 만큼 그 과목의 간식(길쭉한 음식 스프라이트)을 먹어치우는 형태로 진행률을 보여줌. 핫도그/소시지/사탕/솜사탕처럼 먹음직스러운 것부터 우선 배정하고, 귀여운 폰트(Jua)를 쓰고, 진짜 식당에서 주문 영수증을 하나씩 해결하는 느낌을 내라는 지시를 반영함.

- **에셋**: 사용자가 첨부한 Kenney `pixel-platformer-food-expansion` 팩(18×18px 타일시트, CC0)에서 가로로 긴 스프라이트만 잘라냄 — 콘도그, 글레이즈드 소시지, 소시지, 솜사탕, 클럽 샌드위치, 서브 샌드위치, 초콜릿 바, 딸기 바, 치즈 바, 그래엄 크래커 바(총 10종). 전부 base64 PNG로 `FOOD_SPRITES` 상수에 인라인 임베드(합쳐도 몇 KB 수준이라 별도 에셋 파일 없이 그대로 박아넣음 — "단일 HTML 파일" 원칙 유지).
- **배정 순서**: `FOOD_ORDER = ['corndog','glazed','sausage','cottoncandy','club','sub','choc','straw','cheese','graham']` — 과목을 추가한 순서대로 이 배열을 로테이션 배정(`FOOD_ORDER[subjects.length % FOOD_ORDER.length]`). 먹음직스러운 것(콘도그/소시지류/솜사탕)이 먼저 나오도록 사용자가 명시적으로 순서를 지정함.
- **데이터 모델**: `state.data = { examWeek: { subjects: [ { id, name, food, items:[{id, text, done}] } ] } }`. 이게 **이 앱이 저장하는 데이터의 전부**임(로컬 `localStorage` + 로그인 시 Supabase `app_data` 테이블).
- **가게 이름("간판")**: `localStorage`의 `cob-shop-name` 키에 저장(`getShopName`/`setShopName`). 설정 모달의 "가게 이름" 항목에서 사용자가 자유롭게 바꿀 수 있고, 상단 바와 각 주문서 카드의 가게명 줄에 그대로 반영됨. 비어있으면 "이름 없는 분식집"으로 표시.
- **"먹어치우기" 렌더링** (`.food-bar`): 세 레이어를 겹치는 방식:
  1. `.food-bar-img` — 음식 스프라이트를 바 전체 너비로 늘려서(`background-size:100% 100%`) 꽉 채움.
  2. `.food-bar-blocks` — `repeating-linear-gradient`로 `var(--n)`(항목 개수) 등분한 구분선.
  3. `.food-bar-eaten` — 완료 비율만큼 **오른쪽부터** `var(--bg)`로 덮어서 먹힌 것처럼 보이게 함. **왼쪽이 아니라 오른쪽부터 가려야 함** — 콘도그/솜사탕 같은 스프라이트는 막대/손잡이가 스프라이트 왼쪽에 그려져 있어서, 왼쪽부터 가리면 "막대부터 먹는" 꼴이 돼서 어색함(실제로 이 문제로 사용자에게 지적받고 고침). 오른쪽(음식이 있는 쪽)부터 먹고 막대가 마지막까지 남게 하는 게 맞음.
- **체크 연출("쾌감")**: 항목을 완료로 체크하면(`toggleExamItemWithFx`) `.food-bar`에 `just-ate` 클래스(짧은 찌그러짐 애니메이션)를 붙이고 "냠!"/"아삭!" 같은 글자가 위로 떠오르며 사라지는 `.bite-pop`을 띄움. `render()`가 매번 DOM을 통째로 새로 그리는 구조라 이 클래스들은 **`toggleExamItem()`이 끝난 뒤 새로 그려진 DOM에 직접 붙였다가 `animationend`에 떼는 방식**으로 처리함(박스 드래그 시절의 `drop-bounce` 패턴과 동일) — `render()` 자체에 애니메이션 상태를 집어넣으려 하지 말 것, 매 렌더마다 재생돼서 지저분해짐.
- **주문 완료 연출**: 한 과목의 항목을 전부 완료하면(`done===n && n>0`) 카드에 `order-done` 클래스가 붙어 초록색 톤 오버레이가 깔리고, 녹색 테두리의 "완료!" 도장(`.order-stamp`)이 표시됨. 이 상태 자체는 `examSubjectCardHtml()`이 데이터에서 매번 그대로 계산해서 그리므로 새로고침해도 유지됨. 방금 막 완료된 **그 순간의 "쾅" 하는 도장 애니메이션**(`stamp-pop`)만 위와 같은 방식으로 1회성으로 붙였다 뗌.
- **식당 분위기**: 페이지 맨 위에 `renderKitchenScene()`이 그리는 작은 주방 창 배너 — 김이 올라오는 애니메이션(`kitchen-steam`)과 팔을 젓는 요리사 실루엣(`kitchen-arm`, 순수 SVG 도형, 외부 에셋 없음)을 넣어 "누가 음식을 만들고 있는" 느낌을 줌. Kenney 에셋 팩에는 캐릭터 스프라이트가 없어서(음식 전용 확장팩) 인라인 SVG로 직접 그림.
- **글꼴**: Google Fonts의 `Jua`(둥근 느낌의 한글 폰트)를 `--sans` 폰트 스택 맨 앞에 둠. 샌드박스 환경에서는 `fonts.googleapis.com` 접근이 인증서 오류로 막혀 있어 실제 로드 여부를 스크린샷으로 확인하지 못했음 — 실서비스(GitHub Pages)에서는 문제없이 로드돼야 함.

## 코드 구조

`index.html` 하나(`<style>` + `<script>`, IIFE 하나로 감쌈). 대략 순서:

1. Supabase 클라이언트 초기화 + 인증 흐름(`initAuth`/`handleAuthChange`/`signInGoogle`/`signOut`) + 클라우드 동기화(`pushCloudData`/`pullCloudData`)
2. AdSense 배너(`renderAdSlot`)
3. `localStorage` 저장/로드(`loadData`/`persist`), 가게 이름(`getShopName`/`setShopName`)
4. `state` 객체(`{modal, examAddOpen, data:buildSkeleton()}`) + `buildSkeleton()`(데이터 모델: `examWeek`만 있음)
5. 아이콘(`icon(name)` — `settings`/`close`/`trash` 세 개만 남음), 렌더 시스템(`render()` — 탭도 스크롤 체인도 없이 `#app`에 `renderExamPage()` 결과만 통째로 다시 씀), `renderTopNav()`(가게 이름 + 설정 버튼뿐)
6. 설정 모달(`renderSettingsModal`), 개인정보처리방침 모달(`renderPrivacyModal`)
7. **"이번주" 기능** — `FOOD_SPRITES`/`FOOD_ORDER` 상수, `addExamSubject`/`deleteExamSubject`/`addExamItem`/`toggleExamItem`/`deleteExamItem`, `examSubjectCardHtml`/`renderExamPage`/`renderKitchenScene`
8. 내보내기/가져오기(`exportData`/`importDataFromFile`)
9. 체크 연출 래퍼(`toggleExamItemWithFx`) + 이벤트 위임 디스패처(`document.body`에서 `data-action` 기반 `switch`, 이제 10개 케이스뿐)
10. `init()` — 로컬/클라우드 데이터 로드 후 `render()`

**렌더 패턴**: 상태가 바뀌면 `render()`가 `#app.innerHTML`을 포함해 화면 전체를 통째로 새로 씀 — 화면이 하나뿐이라 예전의 "오버레이만 가볍게 다시 그리기"(`renderOverlaysOnly`) 같은 최적화는 더 이상 필요 없어서 삭제함.

## 데이터 구조 (localStorage 로컬 + Supabase `app_data` 테이블 클라우드)

```json
{
  "examWeek": {
    "subjects": [
      {
        "id": "",
        "name": "",
        "food": "corndog|glazed|sausage|cottoncandy|club|sub|choc|straw|cheese|graham",
        "items": [ { "id": "", "text": "", "done": false } ]
      }
    ]
  }
}
```

가게 이름(`shopName`)은 데이터 동기화 대상이 아니라 기기별 `localStorage`(`cob-shop-name` 키)에만 저장됨 — 계정을 옮겨도 안 따라감(간판은 기기/브라우저에 거는 거라는 취지).

## 검증 방법

- **문법 체크**: `node -e "new Function(require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/g).map(s=>s.replace(/<\/?script>/g,'')).join('\n'))"`
- **대량 삭제 후 필수 교차검증**: 위 "대량 삭제 작업 방법" 섹션의 1~4단계(함수 지도 작성 → 마커 기반 스크립트 삭제 → 호출부 grep 전수조사 → CSS 클래스 교차검증) 그대로 반복할 것.
- **로컬 서빙 + Playwright**: `python3 -m http.server <port>` 로 띄운 뒤 `NODE_PATH=/opt/node22/lib/node_modules node <script>.js`로 과목/항목 CRUD, 설정 저장, 새로고침 후 데이터 유지, 콘솔 에러 유무를 확인.

## 남은 작업 / 다음 단계 후보

- "이번주" 탭에 과목/항목 수정(이름 변경, 항목 텍스트 수정) UI는 아직 없음 — 삭제 후 재입력만 가능.
- "이번주" 데이터를 주차 단위로 보관하거나 리셋하는 기능 없음 — 다음 시험 기간엔 사용자가 기존 과목을 직접 지우고 새로 추가해야 함.
- 완료된 주문서(`order-done`)를 자동으로 접어두거나 목록 하단으로 옮기는 기능은 없음 — 계속 같은 자리에 도장만 찍힌 채로 남아있음.
