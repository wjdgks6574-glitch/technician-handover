# CLAUDE.md — 작업 규칙 (토큰 절약 우선)

원본 파일을 다시 읽지 말고 이 문서를 신뢰하라.

## ⛔ 절대 통째로 읽지 말 것

1. **`goapp/initial_data_v142.json`** — 402 KB / 12,457줄. JC01 초기 데이터 1384건
   (전부 `dong="JC01"`), ID `100001~101384`. 확인은 `grep`/`head`, 편집은 `python3`.
2. **`goapp/vendor/`** — 10.5 MB / 684개 파일. grep/glob 시 `--glob '!goapp/vendor/**'`
   또는 `type` 지정으로 제외. 건드리지 않음.

## 📁 관리 파일 (vendor 제외)

```
HANDOFF.md                     빌드/이슈 참고
CLAUDE.md                      이 파일
goapp/go.mod
goapp/initial_data_v142.json   embed 데이터
goapp/main_jc01_v142.go.tmp    JC01 소스 (정본)
goapp/main_jc02_v142.go.tmp    JC02 소스 (정본)
```

`main.go`는 빌드 시 생성되는 임시 파일(.gitignore됨). 소스는 `.go.tmp` 두 개다.
파일명에 `_v142` 접미사가 붙은 게 유일한 정본. 다른 이름의 main_jc01.go 등은 무시.

## 🔨 빌드 & 버전

현재 버전: **v1.4.56**

```bash
export GOPATH=$HOME/go && export PATH=$PATH:/usr/local/go/bin
cd goapp
cp main_jc01_v142.go.tmp main.go && gofmt -w main.go
GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc \
  go build -mod=vendor -ldflags="-H windowsgui" -o 인수인계관리_JC01_v1.4.56.exe .
rm -f main.go
```

- `-ldflags`에 `-s -w` 넣지 말 것 (백신 오탐 원인).
- 빌드해서 전달할 때마다 버전을 올린다. 소스 2개 HTML의 `v1.4.X` 문자열
  (JC01 690번째 줄 / JC02 723번째 줄)과 이 문서·`HANDOFF.md`의 버전도 함께 바꾼다.
- **문서(CLAUDE.md/HANDOFF.md) 갱신은 버전 5개마다 한 번씩만** 몰아서 정리.
- `go vet`는 로컬(리눅스)에서 `syscall.NewLazyDLL`, `buildHTML`의 Sprintf(`%`)를
  오탐한다 — 노이즈. `GOOS=windows` 빌드가 통과하면 정상.

## ⚠️ 두 소스는 대부분 동일, 13곳 + JC02 전용 기능 1개만 다르다

확인은 `diff main_jc01_v142.go.tmp main_jc02_v142.go.tmp` 한 줄이면 된다.

**공통 13곳** (JC01 기준 라인번호, v1.4.56 시점):
| 줄 | JC01 | JC02 |
|----|------|------|
| 307 | 주석 "JC01은 100000번대" | "JC02는 200000번대" |
| 310 | `max := 100000` | `max := 200000` |
| 312 | `r.ID < 200000` | `r.ID < 300000` |
| 329 | `const myDong = "JC01"` | `= "JC02"` |
| 470 | `...\1동\Pending Item&지속관리` | `...\CELL_MEE_..\P10 Cell Sorter` |
| 481 | `"JC_1 Sorter 필요 Parts"` | `"JC_2 Sorter 필요 Parts2"` |
| 690 | `[JC01]` (제목) | `[JC02]` |
| 703-704 | `JC01 selected` | `JC02 selected` |
| 763 | `<option value="JC02">` | `<option value="JC02" selected>` |
| 835 | `const MY_DONG='JC01'` | `='JC02'` |
| 844 | `memoSave('hk_memo',…)` | `memoSave('hk_memo_jc02',…)` |
| 850 | `memoLoad('hk_memo')` | `memoLoad('hk_memo_jc02')` |
| 1009 | `fD.value='JC01'` | `='JC02'` |

**JC02 전용 기능 (v1.4.54~)**: "Part's 교체 이력" 버튼. JC02 diff의 순수 추가분
(`451a`, `470c`의 `openHistoryFileImpl`, `566a`의 `.btn-history` CSS, `716a`의
버튼, `1393a`의 `openHistoryFile()` JS). JC01엔 없다.

> PM 체크리스트 저장 키(`'hk_pm_'+MY_DONG`)와 좌/우 분할(`PM_SPLIT`, JC02만 값
> 존재하나 코드 문자열은 양쪽 동일)은 diff 아님.

## 🗂 데이터 모델 (Record)

```go
type Record struct {
    ID int; Date, Equip, Worker, Shift, Category, Content, Dong string; Flag bool
}  // json: id/date/equip/worker/shift(omitempty)/content/category/dong/flag(omitempty)
```
- `ID`는 정수 (문자열 금지 — WebView2 바인딩 의존). JC01=100000번대(100001~199999),
  JC02=200000번대. `nextID()`는 자기 동 대역 안에서만 최댓값 탐색.
- `Shift`: `"주"`/`"야"`/빈 문자열. 모달 `#fs`에서 선택. `workerCell(w,shift)`가
  근무자 이름 위에 표시.
- 정렬(메인 목록): `flag` → 날짜 내림 → 근무조(`SHIFT_ORDER=['주','야']`) →
  구분(`CAT_ORDER=['전달사항','Classification','기자재관리','설비이슈']`, 그 외 맨 뒤) →
  설비 호기(`EQUIP_BY_DONG` 순) → id 내림.
- 저장: `\\172.23.11.175\...\records.json` (JC01/JC02 공유, `dong`으로 구분) +
  `%APPDATA%\인수인계관리\records_local_backup.json` (폴백). 읽기 5초/쓰기 3초 타임아웃.
- 저장은 백그라운드: 바인딩은 `recMu` 잠그고 메모리만 변경 후 `requestSave()` →
  `startFlusher`의 단일 goroutine이 `flushOnce`로 직렬 저장(상대 동=네트워크 최신본,
  내 동=메모리 정본으로 병합).
- 설비명: `ATW#21/22/31/32/41/42/51/52` → `#21~#52`로 통합(`ATW#61`만 예외).
  `migrateEquipNames()`가 시작 시 자동 변환.
- 새 항목 모달 기본값: 동=현재 필터, 날짜=오늘, 근무자/구분=같은 동 ID 최댓값 레코드
  값 따라감, 근무조=기본값 없음.

## 🧭 Go 함수 위치 (main_jc01, 대략 동일)

| 줄 | 함수 |
|----|------|
| 20 | `//go:embed` + `initialDataJSON` |
| 44 `recMu` · 49 `saveSignal` | records 보호 / 저장 신호 |
| 51 `getDataDir` · 62 `getNetworkDir` | 경로 |
| 69 `readWithTimeout` · 99 `writeWithTimeout` | 타임아웃 IO |
| 155 `migrateEquipNames` · 171 `migrateIDOffsets` | 마이그레이션 |
| 190 `loadRecords` · 234 `saveRecords` | 데이터 로직 |
| 244 `requestSave` · 254 `startFlusher` · 269 `flushOnce` | 백그라운드 저장 |
| 309 `nextID(rs)` · 329 `myDong` | 채번 / 쓰기권한 동 |
| 331 `handleBind` | WebView2 바인딩 (dbGetAll/dbAdd/dbUpdate/dbDelete/dbSetFlag, memoSave/memoLoad) |
| 469 `openPartsFileImpl` · 501 `maximizeWindow` | |
| 507 `main` | loadRecords→startFlusher→WebView2 |
| 549 `buildHTML` | UI 전체 (HTML+CSS+JS) |

## 🚫 현재 구현 상태 (한 줄 요약, 이유·이력 생략)

- 테이블: `<table>` 아님, div Flexbox (`.frow`/`.fc-*`).
- 네트워크 상태: `dbGetAll()` 응답에 포함. 별도 상태 바인딩 추가 금지(hang 위험).
- 무거운 렌더링 전 `requestAnimationFrame` 1프레임 양보.
- 메인 목록: 가상 스크롤(`RCHUNK=60`, `renderMore()`). 전량 렌더 금지.
- 메모/PM: `memoSave/memoLoad`로 `%APPDATA%\인수인계관리\<key>.txt` 저장. localStorage 금지.
- 동별 쓰기 권한: `myDong`/`MY_DONG`만 쓰기 가능, 상대 동은 서버에서도 거부.
- 상대 동 행: `.frow.foreign` 배경만 회색, 글자색은 검정 유지.
- 전체보기 닫기 버튼: `.vfclose` 클래스로 찾을 것(`closest('[style]')` 금지).
- 달력 CSS 선언 순서: `.sat`/`.sun` → `.rng`(기간) → `.td2`(오늘) → `.sd`(선택, 항상 이김) → 근무조 배경 → `.foreign`.
- 메인 목록 내용칸(`.fc-ct .pv`): 높이 제한 없음(세로로 무한정 늘어남). `max-height`/`overflow:hidden` 넣지 말 것.
- 자정 갱신: `scheduleMidnight()`가 매일 00:00:02에 `renderCal` 재호출.
- 날짜 필터: `selDates`(Set), `save()` 후에도 유지.
- 기간 필터: `.cal-range`의 `#rs`/`#re`(텍스트, `26/06/24` 형식) + `#rsd`/`#red`(아이콘
  date input). 내부 상태(`rangeStart/End`)는 `YYYY-MM-DD`.
- 메모: `.right` 칼럼 달력 밑, `flex:1`, `min-height:260px`.
- 구분 목록 변경 시 3곳(`#fC`,`#fc`,`.c<이름>`) 동시 수정.
- PM 체크리스트: `.pm-col`(폭 auto) 안 `.pm-grid`(flex, 스크롤 1개)에 `.pm-half`
  2개(좌/우 독립 2칸 그리드 `max-content 210px`, `renderPmTable`이 L/R 문자열 조립).
  좌/우 설비 배분은 `PM_SPLIT[MY_DONG]`(JC02만 명시 배열: 좌=공통·#1~7·#31·#32·#99,
  우=#41·#51·#52·#60·#61·#71·#96)이 있으면 그대로, 없으면 목록 절반씩 자동 분할.
- 근무자 셀: `workerCell(w,shift)`로 여러 줄+폰트 축소 렌더.
- 근무조 필드: `dbAdd/dbUpdate(date,equip,worker,shift,category,content,dong)` 인자 순서 고정.
- 근무조 행 배경: `shift-day`/`shift-night`, CSS 순서 `.sel`→`shift-*`→`.foreign`.
