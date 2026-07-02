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

현재 버전: **v1.4.20**

```bash
export GOPATH=$HOME/go && export PATH=$PATH:/usr/local/go/bin
cd goapp
# JC01 (JC02는 파일명만 jc02로)
cp main_jc01_v142.go.tmp main.go && gofmt -w main.go
GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc \
  go build -mod=vendor -ldflags="-H windowsgui" -o 인수인계관리_JC01_v1.4.20.exe .
rm -f main.go
```

- `-ldflags`에 **`-s -w` 넣지 말 것** (백신 오탐 원인).
- **빌드해서 전달할 때마다 버전을 올린다.** 소스 2개 HTML의 `v1.4.X` 문자열
  (각 파일 557번째 줄)과 이 문서·`HANDOFF.md`의 버전도 함께 바꾼다.

## ⚠️ 두 소스는 997줄이 동일, 딱 13곳만 다르다

`main_jc01`과 `main_jc02`는 **거의 동일**하다. 한쪽을 수정하면 **반드시 다른
쪽도 같은 위치에 반영**하라 (아래 13곳 외에는 두 파일이 항상 같아야 함).
확인은 `diff main_jc01_v142.go.tmp main_jc02_v142.go.tmp` 한 줄이면 된다 —
두 파일을 각각 Read 하지 말 것.

다른 곳 (JC01 → JC02 기준 라인번호, v1.4.20 시점):
| 줄 | JC01 | JC02 |
|----|------|------|
| 251 | 주석 "JC01은 100000번대" | "JC02는 200000번대" |
| 254 | `max := 100000` | `max := 200000` |
| 256 | `r.ID < 200000` | `r.ID < 300000` |
| 371 | `...\1동\Pending Item&지속관리` | `...\CELL_MEE_..\P10 Cell Sorter` (파츠폴더) |
| 382 | `"JC_1 Sorter 필요 Parts"` | `"JC_2 Sorter 필요 Parts2"` (파츠파일명) |
| 557 | `[JC01]` (제목) | `[JC02]` |
| 570-571 | `JC01 selected` | `JC02 selected` |
| 624 | `<option value="JC02">` | `<option value="JC02" selected>` |
| 682 | `memoSave('hk_memo',…)` | `memoSave('hk_memo_jc02',…)` |
| 688 | `memoLoad('hk_memo')` | `memoLoad('hk_memo_jc02')` |
| 806 | `fD.value='JC01'` | `='JC02'` |
| 1007 | `... || 'JC01'` | `... || 'JC02'` |

## 🗂 데이터 모델 (Record)

```go
type Record struct {
    ID int; Date, Equip, Worker, Category, Content, Dong string
}  // json 태그: id/date/equip/worker/category/content/dong
```
- `ID`는 **정수** (문자열로 절대 바꾸지 말 것 — WebView2 바인딩 의존).
- **ID 대역 분리**: JC01 = `100000`번대(100001~199999), JC02 = `200000`번대.
  `nextID()`는 자기 동 대역 안에서만 최댓값을 찾고, `migrateIDOffsets()`가
  옛 순차 ID를 시작 시 자동 이관한다. 대역이 겹치면 안 됨.
- 저장: `\\172.23.11.175\...\records.json` (네트워크 공유, JC01/JC02 혼재,
  `dong`으로 구분) + `%APPDATA%\인수인계관리\records_local_backup.json` (폴백).
  읽기 5초 / 쓰기 3초 타임아웃(goroutine+channel).

## 🧭 Go 함수 위치 (main_jc01, 대략 동일)

| 줄 | 함수 |
|----|------|
| 19-20 | `//go:embed` + `initialDataJSON` |
| 39 `getDataDir` · 50 `getNetworkDir` | 경로 |
| 57 `readWithTimeout` · 87 `writeWithTimeout` | 타임아웃 IO |
| 143 `migrateEquipNames` · 159 `migrateIDOffsets` | 마이그레이션 |
| 178 `loadRecords` · 222 `saveRecords` | 데이터 로직 |
| 235 `saveMerged` · 253 `nextID(rs)` | 저장 시 재읽기·델타 병합 / 채번 |
| 268 `handleBind` | WebView2 바인딩 (dbGetAll/dbAdd/dbUpdate/dbDelete→saveMerged, memoSave/memoLoad) |
| 370 `openPartsFileImpl` · 402 `maximizeWindow` | |
| 408 `main` | WebView2 생성. `DataPath` 고정(getDataDir/webview2) |
| 444 `buildHTML` | UI 전체 (HTML+CSS+JS, ~600줄) |

## 🚫 회귀 방지 (HANDOFF의 과거 사건 요약 — 상세는 HANDOFF.md)

- 테이블은 `<table>` 아님, **div Flexbox** (`.frow`/`.fc-*`). 되돌리지 말 것.
- 네트워크 상태는 `dbGetAll()` 응답에 실어 보냄. **별도 상태 전용 바인딩
  추가 금지** (dbGetPath 단독 호출이 WebView2에서 hang 재발 위험).
- **저장은 `saveMerged`로 재읽기·병합** — `dbAdd/Update/Delete`는 통째 덮어쓰기
  금지. 저장 직전 네트워크 파일을 다시 읽어 내 델타만 얹는다(JC01/JC02 동시
  사용 시 상대 동 데이터 유실 방지). 전량 덮어쓰기로 되돌리지 말 것.
- 무거운 렌더링 전 `requestAnimationFrame` 한 프레임 양보 유지(배지 리페인트).
- **메인 목록은 가상 스크롤(점진적 렌더)** — `renderTable`이 `fil` 전량을
  `innerHTML`로 그리지 말 것(1만 건 렉 원인). `RCHUNK`(60)씩 `renderMore()`로
  이어붙이고 `.tw` 스크롤 하단에서 다음 묶음 로드. 전량 렌더로 되돌리지 말 것.
- **메모는 `localStorage` 금지, 로컬 파일 저장** — `SetHtml`(NavigateToString)은
  origin이 opaque라 localStorage가 재시작 시 유실된다. `memoSave/memoLoad` Go
  바인딩으로 `%APPDATA%\인수인계관리\<key>.txt`에 저장. localStorage로 되돌리지 말 것.
- 헤드리스/유닛테스트 통과 ≠ Windows 실기 정상. 큰 변경은 실기 확인 필요.
