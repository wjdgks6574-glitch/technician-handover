# CLAUDE.md — 작업 규칙 (토큰 절약 우선)

이 파일은 **큰 파일을 열지 않고도 작업할 수 있도록** 핵심 사실을 하드코딩한
것이다. 아래에 답이 있으면 원본 파일을 다시 읽지 말고 이 문서를 신뢰하라.
상세 배경은 `HANDOFF.md`에 있으나, 일상 작업에는 이 문서만으로 충분하다.

## ⛔ 절대 통째로 읽지 말 것 (토큰 낭비 지점)

1. **`goapp/initial_data_v142.json`** — 402 KB / 12,457줄. 전체 읽으면 ~10만 토큰.
   - 내용: JC01 초기 데이터 **1384건** (전부 `dong="JC01"`), ID `100001~101384`.
   - 스키마는 아래 "데이터 모델" 참고. 값 확인이 필요하면 `head`/`grep`으로
     특정 줄만, 편집이 필요하면 `python3`로 스크립트 처리 (개별 Read/Edit 금지).
2. **`goapp/vendor/`** — 10.5 MB / 684개 파일. 오프라인 빌드용 의존성.
   - **grep/glob 시 반드시 제외**: `--glob '!goapp/vendor/**'` 또는 `type` 지정.
   - 이 안의 코드는 우리가 건드리지 않는다. 절대 탐색/읽기 대상 아님.

## 📁 우리가 관리하는 파일은 6개뿐 (vendor 제외)

```
HANDOFF.md                     상세 인수인계 (배경/사건사고)
CLAUDE.md                      이 파일
goapp/go.mod
goapp/initial_data_v142.json   embed 데이터 (위 경고 참고)
goapp/main_jc01_v142.go.tmp    JC01 소스 (정본)
goapp/main_jc02_v142.go.tmp    JC02 소스 (정본)
```

`main.go`는 빌드 시 생성되는 임시 파일(.gitignore됨). 소스는 `.go.tmp` 두 개다.

## 🔨 빌드 & 버전

현재 버전: **v1.4.41**

```bash
export GOPATH=$HOME/go && export PATH=$PATH:/usr/local/go/bin
cd goapp
# JC01 (JC02는 파일명만 jc02로)
cp main_jc01_v142.go.tmp main.go && gofmt -w main.go
GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc \
  go build -mod=vendor -ldflags="-H windowsgui" -o 인수인계관리_JC01_v1.4.41.exe .
rm -f main.go
```

- `-ldflags`에 **`-s -w` 넣지 말 것** (백신 오탐 원인).
- **빌드해서 전달할 때마다 버전을 올린다.** 소스 2개 HTML의 `v1.4.X` 문자열
  (각 파일 680번째 줄)과 이 문서·`HANDOFF.md`의 버전도 함께 바꾼다.
- `go vet`는 로컬(리눅스)에서 `syscall.NewLazyDLL`, `buildHTML`의 Sprintf(`%`)를
  오탐한다 — 둘 다 예전부터 있던 노이즈. `GOOS=windows` 빌드가 통과하면 정상.

## ⚠️ 두 소스는 997줄이 동일, 딱 13곳만 다르다

`main_jc01`과 `main_jc02`는 **거의 동일**하다. 한쪽을 수정하면 **반드시 다른
쪽도 같은 위치에 반영**하라 (아래 13곳 외에는 두 파일이 항상 같아야 함).
확인은 `diff main_jc01_v142.go.tmp main_jc02_v142.go.tmp` 한 줄이면 된다 —
두 파일을 각각 Read 하지 말 것.

다른 곳 (JC01 → JC02 기준 라인번호, v1.4.41 시점):
| 줄 | JC01 | JC02 |
|----|------|------|
| 307 | 주석 "JC01은 100000번대" | "JC02는 200000번대" |
| 310 | `max := 100000` | `max := 200000` |
| 312 | `r.ID < 200000` | `r.ID < 300000` |
| 329 | `const myDong = "JC01"` | `= "JC02"` (Go 쓰기권한 동) |
| 470 | `...\1동\Pending Item&지속관리` | `...\CELL_MEE_..\P10 Cell Sorter` (파츠폴더) |
| 481 | `"JC_1 Sorter 필요 Parts"` | `"JC_2 Sorter 필요 Parts2"` (파츠파일명) |
| 684 | `[JC01]` (제목) | `[JC02]` |
| 697-698 | `JC01 selected` | `JC02 selected` |
| 751 | `<option value="JC02">` | `<option value="JC02" selected>` |
| 823 | `const MY_DONG='JC01'` | `='JC02'` (JS 쓰기권한 동) |
| 831 | `memoSave('hk_memo',…)` | `memoSave('hk_memo_jc02',…)` |
| 837 | `memoLoad('hk_memo')` | `memoLoad('hk_memo_jc02')` |
| 987 | `fD.value='JC01'` | `='JC02'` |

> 참고: PM 체크리스트 저장 키는 `'hk_pm_'+MY_DONG`으로 **양쪽 파일 동일**(변수라 diff 아님).

## 🗂 데이터 모델 (Record)

```go
type Record struct {
    ID int; Date, Equip, Worker, Shift, Category, Content, Dong string; Flag bool
}  // json: id/date/equip/worker/shift(omitempty)/content/category/dong/flag(omitempty)
```
- **Shift(근무조, v1.4.37)**: `"주"`/`"야"`/빈 문자열. 새 항목 모달의 구분 밑
  칸(`#fs`)에서 선택. `omitempty`라 옛 레코드엔 없어도 하위호환. 메인 목록에서
  근무자 칸 이름 위에 한 줄로 표시(`workerCell(w,shift)`의 `.wk-shift`).
- `ID`는 **정수** (문자열로 절대 바꾸지 말 것 — WebView2 바인딩 의존).
- **정렬(메인 목록, v1.4.39 기준)**: `flag`(최상단) → 날짜 내림 → **근무조 우선순위**
  (`SHIFT_ORDER`: 야>주, 빈 값은 맨 뒤) → **구분 우선순위**
  (`CAT_ORDER`: 전달사항>Classification>기자재관리>설비이슈, 그 외인 감소활동은
  맨 뒤) → **설비 호기 순서**(`EQUIP_BY_DONG` 나열 순) → id 내림(안정성 타이브레이커).
  `flag`는 행별 ⚑ 버튼으로 토글, `dbSetFlag(id,flag)`가 메모리 갱신 후
  백그라운드로 DB에 저장(공유).
- **ID 대역 분리**: JC01 = `100000`번대(100001~199999), JC02 = `200000`번대.
  `nextID()`는 자기 동 대역 안에서만 최댓값을 찾고, `migrateIDOffsets()`가
  옛 순차 ID를 시작 시 자동 이관한다. 대역이 겹치면 안 됨.
- 저장: `\\172.23.11.175\...\records.json` (네트워크 공유, JC01/JC02 혼재,
  `dong`으로 구분) + `%APPDATA%\인수인계관리\records_local_backup.json` (폴백).
  읽기 5초 / 쓰기 3초 타임아웃(goroutine+channel).
- **저장은 백그라운드 (v1.4.25)**: 추가/수정/삭제/플래그는 메모리(`records`)만
  즉시 바꾸고 반환 → `requestSave()`가 백그라운드 flusher에 신호 → `flushOnce()`가
  네트워크에 저장. UI 스레드가 네트워크 IO에 안 막혀 렉이 사라짐. 아래 회귀 방지 참고.

## 🧭 Go 함수 위치 (main_jc01, 대략 동일)

| 줄 | 함수 |
|----|------|
| 20 | `//go:embed` + `initialDataJSON` |
| 44 `recMu`(뮤텍스) · 49 `saveSignal`(chan) | records 보호 / 저장 신호 |
| 51 `getDataDir` · 62 `getNetworkDir` | 경로 |
| 69 `readWithTimeout` · 99 `writeWithTimeout` | 타임아웃 IO |
| 155 `migrateEquipNames` · 171 `migrateIDOffsets` | 마이그레이션 |
| 190 `loadRecords` · 234 `saveRecords` | 데이터 로직 |
| 244 `requestSave` · 254 `startFlusher` · 269 `flushOnce` | 백그라운드 저장(내 동=메모리·상대 동=네트워크 병합) |
| 309 `nextID(rs)` · 329 `myDong`(상수) | 채번 / 쓰기권한 동 |
| 331 `handleBind` | WebView2 바인딩 (dbGetAll/dbAdd/dbUpdate/dbDelete/dbSetFlag→requestSave, memoSave/memoLoad). 소유권은 lock 안 루프에서 검사. dbAdd/dbUpdate 시그니처에 `shift` 인자 포함(v1.4.37) |
| 469 `openPartsFileImpl` · 501 `maximizeWindow` | |
| 507 `main` | loadRecords→startFlusher→WebView2. 종료 시 flushOnce로 마지막 저장 |
| 549 `buildHTML` | UI 전체 (HTML+CSS+JS, ~600줄) |

## 🚫 회귀 방지 (HANDOFF의 과거 사건 요약 — 상세는 HANDOFF.md)

- 테이블은 `<table>` 아님, **div Flexbox** (`.frow`/`.fc-*`). 되돌리지 말 것.
- 네트워크 상태는 `dbGetAll()` 응답에 실어 보냄. **별도 상태 전용 바인딩
  추가 금지** (dbGetPath 단독 호출이 WebView2에서 hang 재발 위험).
- **저장은 백그라운드 + 동별 병합 (v1.4.25, 예전 saveMerged 대체)** — 바인딩
  (dbAdd/Update/Delete/SetFlag)은 `recMu` 잠그고 메모리만 바꾸고 `requestSave()`
  후 즉시 반환(네트워크 IO로 UI 막지 말 것 — 1만 건 렉 원인). 실제 저장은
  `startFlusher`의 단일 goroutine이 `flushOnce`를 **직렬로** 실행: 네트워크를 다시
  읽어 **상대 동은 네트워크 최신본, 내 동은 메모리(정본)** 로 합쳐 쓴다(한 동은
  한 PC만 쓰므로 성립). `saveSignal`(버퍼1)로 연속 변경은 coalesce. 종료 시
  `main`이 `flushOnce` 한 번 더 호출해 막 누른 변경 유실 방지. **다시 UI 스레드에서
  동기 저장(saveMerged 방식)으로 되돌리지 말 것.** `records`는 항상 `recMu` 아래에서.
- 무거운 렌더링 전 `requestAnimationFrame` 한 프레임 양보 유지(배지 리페인트).
- **메인 목록은 가상 스크롤(점진적 렌더)** — `renderTable`이 `fil` 전량을
  `innerHTML`로 그리지 말 것(1만 건 렉 원인). `RCHUNK`(60)씩 `renderMore()`로
  이어붙이고 `.tw` 스크롤 하단에서 다음 묶음 로드. 전량 렌더로 되돌리지 말 것.
- **메모는 `localStorage` 금지, 로컬 파일 저장** — `SetHtml`(NavigateToString)은
  origin이 opaque라 localStorage가 재시작 시 유실된다. `memoSave/memoLoad` Go
  바인딩으로 `%APPDATA%\인수인계관리\<key>.txt`에 저장. localStorage로 되돌리지 말 것.
- **동별 쓰기 권한 제한 (v1.4.22)** — 자기 동(`myDong`/`MY_DONG`) 레코드만
  추가/수정/삭제/플래그 가능. 상대 동은 읽기 전용. Go의 dbAdd/Update/Delete/
  SetFlag가 상대 동이면 거부(UI 우회해도), UI는 상대 동 행의 ⚑·수정·삭제를
  숨김(`.foreign`). dbAdd는 dong을 항상 myDong으로 강제. (v1.4.25에서 별도
  `isOwnRecord` 함수는 없애고, 각 바인딩의 `recMu` 잠근 루프가 `Dong==myDong`
  일 때만 변경 → 소유권 검사가 변경과 원자적. 상대 동 id는 매칭이 안 돼 거부됨.)
- **상대 동 행 글자색 (v1.4.23)** — `.frow.foreign`은 배경만 살짝 회색(`#fafbfc`),
  글자색은 검은색 그대로 유지. 예전엔 내용 글자도 회색(`#a0aec0`)으로 흐리게
  했으나 가독성 이슈로 제거함. 다시 흐리게 만들지 말 것.
- **내용 전체보기 X 버튼 (v1.4.24)** — `viewFull`의 닫기 버튼은 `this.closest('[style]')`로
  찾지 말 것: 버튼 자신도 `style` 속성이 있어 자기 자신이 매칭되어 지워지고 창은
  안 닫힌다. `.vfclose` 클래스로 찾아 `d.remove()`(오버레이 자체)를 호출해야 함.
- **달력 요일/오늘 색 (v1.4.26)** — `renderCal`에서 날짜별 `getDay()`로 토요일은
  `.sat`(파랑 글자), 일요일은 `.sun`(빨강 글자) 클래스를 붙인다. 오늘(`.td2`)은
  파란 배경+흰 글자로 요일색보다 우선(CSS에서 `.td2`를 `.sat`/`.sun`보다 뒤에
  선언해 우선순위 확보). **선택된 날짜(`.sd`)는 노란 배경(`#f6e05e`)+갈색 글자
  (`#744210`)로 오늘(파랑)과 구별(v1.4.35)** — `.sd`가 `.td2`보다 CSS에서 먼저
  선언돼 있어, 오늘 날짜를 선택하면 `.td2`가 이겨 파란색 유지(의도된 동작).
- **날짜 필터 (v1.4.27, v1.4.36에서 다중 선택으로 확장)** — 달력 밑 상세 패널
  (`det-card`/`renderDetail`)을 없애고, 달력 날짜 클릭(`selDate2`)은 메인 목록을
  그 날짜(들)만 보이게 하는 **필터**다. 상태는 `selDates`(Set, 여러 날짜 누적 가능) —
  클릭할 때마다 그 날짜만 토글(있으면 제거, 없으면 추가). `applyFilter`가
  `selDates.size>0 && !selDates.has(r.date)`로 거른다. `cntbar`의 파란 칩(선택 3개
  이하면 날짜 나열, 많으면 "N일 선택")의 ✕(`clearDateFilter`)로 전체 해제.
  `rowClick`은 하이라이트만(상세 패널 부활 금지). 저장(`save`) 후엔 `selDates.clear()`로
  필터 풀어 새 항목이 보이게.
- **메모 위치 (v1.4.28)** — 메모 카드(`.memo-card`)는 예전엔 표와 달력 사이 별도
  칼럼이었으나, 이제 `.right`(달력) 칼럼 안 **달력 밑**으로 옮김. `.memo-card`는
  `flex:1`로 달력 아래 남은 높이를 채운다. 표(`.left`)가 그만큼 넓어짐.
- **구분(카테고리) 목록 (v1.4.29 기준: 설비이슈/전달사항/기자재관리/Classification/감소활동)** —
  추가·변경 시 **3곳을 모두** 고쳐야 함: ① 메인 필터 `<select id="fC">` ② 새항목 모달
  `<select id="fc">` ③ 배지 색 CSS `.c<이름>`(이름에 공백 없이). 하나만 빠지면 필터/입력/
  색 중 하나가 어긋난다.
- **설비별 PM 체크리스트 (v1.4.31, 상시 표시 표)** — 메인 목록과 달력 **사이**의
  독립 칼럼 `.pm-col`. 4칸 그리드(`.pm-grid`: 설비|내용|설비|내용)로, 이 동의 필터
  설비(`EQUIP_BY_DONG[MY_DONG]`)를 2개씩 배치(왼쪽 절반=좌측 쌍, 오른쪽 절반=우측
  쌍). 내용칸은 `contenteditable` div(`.pm-cell.pm-text`), 입력마다 `savePmCell`이
  `pmData[설비]=innerText` 후 저장. 메모처럼 로컬 파일 저장이라 껐다 켜도 유지.
  키 `'hk_pm_'+MY_DONG`(동별 파일), 값 `{설비:내용}` JSON(`pmData`). `renderPmTable`이
  시작 시 표를 그린다(그 뒤엔 셀 편집만, 재렌더 없음 → 포커스 유지). `localStorage`로
  되돌리지 말 것(메모와 동일 — opaque origin 유실). v1.4.30의 드롭다운 방식은 폐기.
  **내용 칸 폭 15% 축소 (v1.4.40, v1.4.41에서 배치 버그 수정)** —
  `grid-template-columns`가 `max-content .85fr max-content .85fr .3fr`. 내용
  칸(`1fr`이던 것)을 `.85fr` 두 개로 줄이고, 남는 `.3fr`을 5번째 트랙으로 둬서
  오른쪽 여백으로 흡수시켰다. **주의**: 트랙만 5개로 늘리면 CSS Grid의 auto-flow가
  칸을 4개가 아닌 5개 단위로 채우면서 매 논리적 행마다 칸이 하나씩 밀리는 버그가
  난다(v1.4.40에서 실제로 발생 — 사용자 스크린샷으로 발견). `renderPmTable`이
  DOM에 4개씩(설비|내용|설비|내용) 순서로 셀을 넣는 것과 grid-template-columns의
  트랙 수(5)가 안 맞아서 생기는 문제이므로, 5번째 트랙은 "폭만 있고 절대 채워지지
  않는 트랙"으로 강제해야 한다 — `.pm-grid>*:nth-child(4n+1..4n)`으로 매 셀에
  `grid-column:1~4`를 명시해 auto-flow가 5번째 칸을 절대 쓰지 않게 고정했다
  (v1.4.41). `fr`은 상대값이라 다른 `fr` 트랙이 없으면 계수를 줄여도 그대로 꽉
  채우므로, 폭을 줄이려면 이렇게 트랙을 추가하는 방식이 맞다 — 다만 **트랙을
  추가할 때마다 반드시 `nth-child`로 열을 명시 고정할 것** (안 그러면 이 버그가
  재발한다).
  **`.pm-col` 고정폭으로 전환, 메인 목록에 여백 양보 (v1.4.41)** — 예전엔
  `.pm-col{flex:1;min-width:360px}`로 `.left`(메인 목록)와 똑같이 늘어나서, 내용
  칸을 줄여도 그 여백이 PM 칸 안에서만 남고 메인 목록은 넓어지지 않았다.
  `.pm-col{flex:0 0 360px;width:360px}`로 바꿔 더 이상 늘어나지 않게 고정 —
  PM 체크리스트는 항상 달력(`.right`) 바로 왼쪽에 필요한 만큼의 폭만 차지하고,
  창을 넓히거나 내용 칸 폭을 줄여서 생기는 여유 공간은 전부 `flex:1`인
  `.left`(메인 목록/인수인계 내용)가 가져간다. `.pm-col`을 다시 `flex:1`로
  되돌리지 말 것 — 메인 목록이 좁아지는 예전 문제로 되돌아간다.
- **근무자 셀 여러 줄 (v1.4.33)** — `rowHTML`의 근무자 칸은 `workerCell(r.worker,r.shift)`로
  렌더. 쉼표로 구분된 근무자를 `<br>`로 나눠 여러 줄로 보이고, 가장 긴 이름
  글자수에 따라 폰트를 12→8px로 줄여 근무자 칸(`.fc-w`)에 맞춘다. `<br>`가 flex에서
  안 먹으므로 내부 블록 `.wk` div로 감쌈. 헤더의 "근무자"는 그대로. (v1.4.37에서
  `shift` 인자 추가 — 아래 근무조 항목 참고.)
- **메인 목록 칸 폭 축소 (v1.4.35)** — 동/날짜/설비/근무자 칸 폭과 가로 패딩을 글자
  크기에 맞게 줄임: `.fc-dong`44/`.fc-d`58/`.fc-e`56/`.fc-w`68px(+패딩 3~4px). 설비는
  최장값 `ATW#61`(JC01)이 기준이라 더 못 줄임(`.eb` 패딩도 7→5). 남는 폭은 내용
  칸(`.fc-ct`, flex:1)이 가져감. 더 줄이면 실기에서 글자 잘릴 수 있으니 주의.
- **근무조(Shift) 필드 (v1.4.37)** — `dbAdd`/`dbUpdate` 바인딩 시그니처가
  `(date,equip,worker,shift,category,content,dong)`로 **worker와 category 사이에
  shift가 추가**됐다. JS `save()`의 인자 순서와 반드시 맞춰야 함(순서 바뀌면
  값이 엉뚱한 필드로 들어감). 모달의 `#fs` select(값 "주"/"야"/""), 목록에서는
  `workerCell(w,shift)`가 근무자 이름 위에 한 줄로 표시(`.wk-shift`).
- **근무조별 행 배경색 (v1.4.38)** — `rowHTML`이 `r.shift`에 따라 행에
  `shift-day`(주, 옅은 노랑 `#fef9c3`) 또는 `shift-night`(야, 옅은 하늘색 `#dbeafe`)
  클래스를 붙인다. CSS 선언 순서가 `.sel`→`shift-*`→`.foreign` 순이라, **상대 동
  회색(`.foreign`)이 근무조 색보다 항상 우선**한다(동 구분이 더 중요). 반대로
  근무조 색은 hover/선택(`.sel`) 파랑보다 우선 표시된다. 색을 더 진하게/다르게
  바꿀 땐 이 우선순위(선언 순서)를 유지할 것.
- **정렬에 근무조 추가 (v1.4.39)** — `applyFilter`의 `fil.sort`에서 날짜 다음,
  구분(`CAT_ORDER`) 이전에 근무조 타이브레이커를 넣음: `SHIFT_ORDER=['야','주']`,
  `shiftRank(s)`가 인덱스 반환(빈 값/기타는 맨 뒤). **야간이 주간보다 먼저** 온다
  (사용자 요청 순서 — 착각해서 반대로 바꾸지 말 것).
