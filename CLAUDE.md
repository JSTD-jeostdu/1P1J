# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

The app is **implemented and working**. `index.html` (single file, ~1200 lines) contains the whole product: teacher flow, student flow, the assignment algorithm, and all styling. `firestore.rules` holds the deployed security rules. `1인1역_배정앱_PRD.md` is the original spec — still the source of truth for *intent*, but the code has moved past it in places, so read the code first for anything about current behavior.

## What this app is

A classroom role-assignment tool ("1인1역 배정") for Korean elementary/middle school homeroom teachers. Teachers collect students' top 1–3 role preferences via a QR code, then the app computes an assignment that maximizes class-wide satisfaction (not just pairwise conflict resolution). Teachers review, manually adjust, and finalize the result themselves — the app never auto-decides.

## Repository layout

```
index.html                 전체 앱 (CONFIG · 알고리즘 · Firebase · Vue 앱 · CSS 전부)
firestore.rules            Firestore 보안 규칙 (Firebase 콘솔에 수동 배포)
1인1역_배정앱_PRD.md        원본 기획서
CLAUDE.md                  이 파일
```

There is deliberately **no `package.json`, no bundler, no build step, no test suite**. Do not add them without asking.

### Structure inside `index.html`

| 구간 | 내용 |
|---|---|
| `<style>` | 디자인 시스템 전체 (CSS 변수 팔레트 → 컴포넌트 → 애니메이션 → 반응형/인쇄) |
| `CONFIG` | Firebase 설정, `collectionPrefix`, `rolePresets` — 튜너블은 전부 여기 |
| `computeAssignment()` | min-cost max-flow 배정 알고리즘 (순수 함수, Vue와 무관) |
| 헬퍼 | `shuffleArray` `parseRosterPaste` `generateRangeRows` `downloadCsv` `roleEmoji` |
| `initFirebase()` + `P()` 계열 | Firestore 컬렉션 참조 헬퍼 (`classesCol`, `prefsCol` 등) |
| `App.setup()` | 라우팅 · 인증 · 구독 · 교사 액션 · 학생 액션 |
| `App.template` | 교사 모드 / 학생 모드 전체 마크업 (Vue 템플릿 문자열) |
| `HeroBand` 컴포넌트 | 상단 노란 밴드 + 언덕 SVG + 마스코트 이모지 (전역 등록) |

## Running and testing

`index.html` uses `<script type="module">` with ESM imports from CDN, so **`file://` does not work** — you must serve it over HTTP:

```bash
python -m http.server 8000
```

Then open `http://localhost:8000/index.html`. `localhost` is an authorized domain in Firebase Auth by default, so Google 로그인이 그대로 동작합니다.

- **교사 화면**: `http://localhost:8000/index.html` — 해시 라우팅
  - `#/classes` 학급 목록 · `#/class/{classId}` 작업공간 · `#/class/{classId}/draft/{draftId}` 배정안 상세 · `#/class/{classId}/present` 발표용
  - `#/guide` 사용법(튜토리얼). **로그인 전에도 열려야 하므로 템플릿 분기에서 `!user` 검사보다 앞에 온다.**
- **학생 화면**: `?s={classId}` 쿼리 파라미터가 붙으면 학생 모드로 진입 (해시 라우팅 없음). 교사 화면 "지망 접수" 탭의 QR/링크가 이 URL을 만듭니다.

There are no automated tests. Verify changes by walking the flow in a browser: 학급 개설 → 명단 등록 → 역할 설정 → 접수 열기 → (학생 URL로) 지망 제출 → 배정 실행 → 확정 → 발표용 화면.

## Deployment

GitHub Pages, `main` 브랜치 루트에서 서빙. `git push` 하면 배포됩니다. 별도 빌드/CI 없음.

`firestore.rules`는 자동 배포되지 않습니다 — 규칙을 고쳤으면 Firebase 콘솔(Firestore → 규칙)에 직접 붙여넣어야 합니다.

## Mandated tech stack (do not deviate without asking)

- **Single `index.html` file, no build tooling.**
- **CDN-only dependencies:**
  - Vue 3.4.31 (global build, `<script defer>`)
  - Firebase 10.13.2 modular SDK (`firebase-app` / `firebase-auth` / `firebase-firestore`) — `initFirebase()`에서 **동적 `import()`로 지연 로딩**
  - `qrcode@1.5.3` (jsDelivr `+esm` 번들 — 이 패키지는 브라우저용 UMD 번들이 없어서 ESM으로만 불러옴)
  - Google Fonts: Jua (제목/브랜드), Noto Sans KR (본문)
- A top-level `CONFIG` object at the very top of the script for tunables.
- **Korean inline comments** throughout.
- Auth/data: Firebase Auth + Firestore. 교사는 **구글 로그인 또는 익명 로그인** 중 하나를 쓴다 (학생은 어느 쪽도 쓰지 않는다).

## Authentication

교사는 두 가지 방식으로 시작할 수 있고, 둘 다 uid 를 발급받으므로 `ownerUid` 기반 소유 판정은 동일하게 동작한다.

- **구글 로그인** (`signIn`) — 기기를 옮겨 다녀도 같은 학급을 볼 수 있다.
- **익명 로그인** (`signInAnon`) — 로그인 없이 즉시 시작. **Firebase 콘솔 → Authentication → Sign-in method 에서 "익명"을 켜 두어야 동작한다.** 꺼져 있으면 `auth/operation-not-allowed` 를 잡아 안내 토스트를 띄운다.
- **계정 연결** (`linkGoogle` → `linkWithPopup`) — 익명 계정을 구글 계정에 붙인다. uid 가 그대로 유지되므로 만들어 둔 학급이 전부 따라온다. 이미 쓰이고 있는 구글 계정이면 `auth/credential-already-in-use` 가 나므로 안내만 하고 실패시킨다.

익명 계정은 브라우저 저장소에 묶여 있어 **방문 기록을 지우면 복구할 방법이 없다.** 그래서 UI 네 군데에서 이 사실을 알린다 — 시작 화면 안내문, 학급 목록 상단 경고 배너, 상단바 [구글 계정 연결] 버튼, 로그아웃 전 확인 모달. **이 안전장치를 지우지 말 것.**

학생은 로그인하지 않는다. 익명 로그인을 붙였다고 해서 학생 화면에서 자동 로그인을 시도해서는 안 된다.

## Firebase project reuse — important constraint

**Do not create a new Firebase project.** This app rides on the existing "조회 도우미 (Johoe)" project (`johoe-9am`) to stay within the free-tier project-count limit. Isolation is by collection prefix:

- 이 앱의 모든 컬렉션은 `roleAssign_` 접두어를 쓴다 (`CONFIG.collectionPrefix`). 코드에서 컬렉션 경로를 직접 문자열로 쓰지 말고 `classesCol()` / `studentsCol()` 같은 헬퍼를 쓸 것.
- 보안 규칙은 `roleAssign_` 경로에만 좁게 적용하고, 기존 앱 데이터 접근 범위를 넓히지 않는다.

## Data model

```
roleAssign_classes/{classId}
  ├─ ownerUid, className, submissionOpen, createdAt/updatedAt
  ├─ students/{studentNumber}     — number(학번, 문서 ID와 동일), name
  ├─ roles/{roleId}               — name, capacity
  ├─ preferences/{studentNumber}  — choices: [roleId, ...] (1~3개), submittedAt
  └─ drafts/{draftId}             — assignments: {studentNumber: roleId|null},
                                     stats: {first, second, third, unassigned},
                                     isFinal, createdAt
```

Multi-tenant: 각 학급 데이터는 전부 `classId` 아래에 있고 `ownerUid`로 스코프됩니다. 교사끼리 서로의 학급을 볼 수 없습니다.

## Write policy

- **Field-level merge writes only — never overwrite whole documents.** 여러 학생이 동시에 제출하므로 문서 통째 덮어쓰기는 유실을 부릅니다.
- 학생 제출은 각자 독립 문서(`preferences/{학번}`)이므로 서로 충돌하지 않습니다.

## Security rules (`firestore.rules`)

- `roleAssign_classes/{classId}`: `ownerUid == request.auth.uid` 일 때만 읽기/쓰기.
- `preferences/{studentNumber}`: 학생 제출 쓰기 허용. 단 **자기 학번 문서에만**, **`submissionOpen == true`일 때만**, **명단에 있는 학번만**, 그리고 **지망 1~3개**일 때만.
  - 예전에는 `request.auth == null` (비로그인) 조건이 붙어 있었으나 뺐다. 교사용 익명 로그인이 생기면서 학생 기기에도 auth 세션이 남아 있을 수 있어, 그 학생이 영영 제출하지 못하는 일이 생기기 때문이다. 원래도 비로그인 쓰기를 허용하던 경로라 보안 수준은 달라지지 않는다.
- 학생은 다른 학생의 지망도, 배정안(`drafts`)도 절대 읽을 수 없습니다.

> **함정:** 학생 화면 코드에서 `preferences` 를 읽으면 안 됩니다. 규칙상 교사만 읽을 수 있어서 학생 기기에서는 `permission-denied` 가 납니다. 그런데 **교사가 자기 링크로 미리보기할 때는 `isOwner()` 가 참이라 성공**하기 때문에, 개발 중에는 멀쩡해 보이고 실제 학생만 막히는 형태로 숨습니다. 실제로 `stuLookup()` 이 이전 지망을 미리 채우려고 이 문서를 읽다가 학생 전원이 번호 입력 화면에서 더 나아가지 못한 적이 있습니다. 부득이 읽어야 한다면 반드시 개별 `try/catch` 로 삼키고, 실패해도 흐름이 이어지게 하세요.

## Assignment algorithm (`computeAssignment`)

반 전체에서 1지망을 받은 학생 수를 최대화하는 것이 목표입니다 (짝짓기 충돌 해소가 아님).

- **min-cost max-flow** (SPFA 기반 successive shortest path). source → 학생 → 역할 → sink.
- 비용: 1지망 0, 2지망 10, 3지망 100. **인접 등급 간 격차가 의도적으로 큽니다** — 1지망 1명을 희생해 2지망 여러 명을 얻는 교환을 막아, 상위 지망 인원이 사전식(lexicographic)으로 먼저 최대화됩니다. 이 값을 좁히면 그 보장이 깨집니다.
- 동점 처리: 매 실행마다 학생 입력 순서를 셔플하므로, 총비용이 같은 최적해가 여러 개일 때 무작위로 하나가 뽑힙니다. **같은 입력이라도 실행마다 결과가 달라지는 것은 버그가 아니라 의도된 동작입니다.**
- 전부 클라이언트에서 실행되며 30명 규모에서 즉시 끝납니다.
- 3지망 안에서 자리를 못 잡은 학생과 아예 미제출한 학생은 **미배정으로 남깁니다** — 남은 자리에 자동으로 채워 넣지 않습니다. 별도 "미배정" 목록으로 보여주고 교사가 손으로 배정합니다.
- 실행할 때마다 새 배정안(안 1, 안 2, …)이 통계와 함께 저장되어 비교할 수 있습니다.

## Key product rules to preserve

- **Results are teacher-only.** 학생은 앱에서 배정 결과를 볼 수 없습니다 (의도적 — 시차 공개로 인한 교실 혼란 방지). 요청받지 않은 학생용 결과 화면을 만들지 마세요.
- 역할 정원 총합이 학생 수보다 많아도 됩니다(빈자리로 남음). 단 **정원 총합 < 학생 수**일 때는 경고를 띄우고, 경고에서 정원 수정 화면으로 바로 갈 수 있어야 합니다.
- 확정된 배정안(`isFinal`)은 잠깁니다. 다시 편집하려면 "확정 해제"라는 별도 액션이 필요합니다.
- 학생 로그인은 번호만 입력(비밀번호 없음)이지만, 명단에는 번호와 이름을 함께 저장해야 합니다 — 학생이 번호를 맞게 넣었는지 이름으로 스스로 확인하는 단계가 있기 때문입니다.
- 학생 화면에서는 "학번"이 아니라 **"번호"**라고 부릅니다 (학생에게 학번은 5자리 학교 학적번호로 읽히기 때문). 교사 화면·CSV는 "학번" 유지.
- `#/guide` 사용법 페이지는 이 앱의 온보딩입니다. 플로우나 용어를 바꾸면 여기 설명도 같이 고쳐야 합니다 (교사 7단계 · 학생 3단계 · FAQ 6개).

## Design system

디자인 방향은 **"통통 튀는 귀여운 교실 앱"** — Knotted 같은 파스텔 베이커리 브랜딩에서 따왔습니다. 새 UI를 붙일 때 이 언어를 유지하세요.

- 팔레트는 전부 `:root` CSS 변수입니다. 하드코딩된 색을 새로 넣지 말고 변수를 쓰거나 변수를 추가하세요. 메인은 연두/초록(`--green`), 포인트는 노랑(`--butter`)과 핑크(`--pink`).
- 화면 전체를 감싸는 초록 프레임(`body::after`), 상단 노란 히어로 밴드 + 언덕 일러스트(`HeroBand`)가 브랜드 시그니처입니다.
- 컴포넌트는 **두툼한 테두리 + 아래로 떨어지는 단색 그림자**(`box-shadow: 0 Npx 0 색`)로 장난감 같은 입체감을 냅니다. 버튼은 누르면 그림자가 사라지며 내려앉습니다.
- 모서리는 크게 (카드 24px, 버튼/칩 999px).
- 애니메이션은 `pop-in` / `float-y` / `wiggle` 세 가지를 재사용합니다. 새로 만들기 전에 있는 걸 먼저 쓰세요. **`prefers-reduced-motion`에서 전부 꺼지도록 유지해야 합니다.**
- `roleEmoji(name)`가 역할 이름 키워드로 이모지를 골라줍니다 (급식→🍚, 청소→🧹 …). 매칭 실패 시 🌟. 역할을 표시하는 새 화면을 만들면 이걸 쓰세요.
- **마스코트와 아이콘은 이모지로 씁니다.** 손으로 그린 SVG 캐릭터를 넣지 마세요 — 한 번 시도했다가 걷어냈습니다. 배경 언덕처럼 이모지로 대체할 수 없는 것만 SVG로 그립니다.
- 사용법 페이지는 `.step-card`(번호 원 + 제목 + 설명 + `.tip`)와 `.faq`를 씁니다. 학생이 하는 일은 `.for-student`를 붙여 파란색 계열로 구분합니다.
