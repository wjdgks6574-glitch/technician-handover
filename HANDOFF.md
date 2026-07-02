# 인수인계 관리 프로그램 - 작업 인수인계 문서

## 현재 버전: v1.4.15

Go + WebView2 기반 Windows 데스크톱 앱. JC01(1동)/JC02(2동) 두 변형이 거의 동일한
소스 구조를 공유하며, 각각 별도의 .exe로 빌드됨.

---

## 파일 구성

```
goapp/
├── main_jc01_v142.go.tmp   ← JC01용 소스 (빌드 시 main.go로 복사해서 사용)
├── main_jc02_v142.go.tmp   ← JC02용 소스
├── initial_data_v142.json  ← 프로그램에 내장(embed)되는 초기 데이터 1384건
├── go.mod
└── vendor/                 ← 오프라인 빌드용 의존성 (go-webview2 등)
```

**중요**: 파일명에 `_v142` 접미사가 붙은 이유는 과거 시행착오의 흔적입니다.
한때 v1.5.0~v1.5.7까지 네트워크 동기화/휴지통 등 복잡한 기능을 추가했다가
Windows 실기(WebView2)에서 원인 불명의 "데이터 0건 표시" 버그가 반복 발생해서
v1.4.2 기준으로 전체 롤백했습니다. **지금부터는 이 `_v142` 파일들이 유일한
정본(source of truth)입니다.** 다른 이름의 main_jc01.go.tmp 등이 남아있다면
전부 폐기된 옛날 버전이니 무시하세요.

---

## 빌드 명령 (Windows용 크로스 컴파일, Linux/WSL 환경 기준)

```bash
export GOPATH=$HOME/go
export PATH=$PATH:/usr/local/go/bin
cd goapp

# JC01
cp main_jc01_v142.go.tmp main.go
gofmt -w main.go
GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc \
  go build -mod=vendor -ldflags="-H windowsgui" -o 인수인계관리_JC01_v1.4.15.exe .

# JC02
cp main_jc02_v142.go.tmp main.go
gofmt -w main.go
GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc \
  go build -mod=vendor -ldflags="-H windowsgui" -o 인수인계관리_JC02_v1.4.15.exe .
```

**주의**: `-ldflags`에 `-s -w`를 넣지 마세요. 심볼 제거(압축)가 백신 오탐의
주요 원인으로 확인되어 이후 버전부터 제거했습니다.

버전을 올릴 때는 두 소스 파일 안의 HTML에 있는 `v1.4.X` 문자열도
**반드시 함께 바꿔야 합니다** (파일명과 내부 표시 버전이 따로 놀아서
사용자가 혼란을 겪은 적이 있습니다 — v1.4.4 사건 참고).

**규칙 (v1.4.13부터)**: 코드 변경 후 실행 파일을 빌드해서 전달할 때마다
버전을 올린다. 같은 버전 번호로 다른 내용의 exe를 여러 번 전달하지 않는다.

---

## 아키텍처 요약

### 데이터 저장 (v1.4.3부터 네트워크 지원 재도입)
- **주 저장소**: `\\172.23.11.175\ETC_JC_BergerHalm\인수인계관리_DB\records.json`
  (JC01/JC02 공유 — 두 동 데이터가 한 파일에 섞여 있고 `dong` 필드로 구분)
- **로컬 백업**: `%APPDATA%\인수인계관리\records_local_backup.json`
  (네트워크 접근 실패 시 자동 폴백, 저장할 때마다 같이 갱신)
- 읽기/쓰기 모두 **5초/3초 타임아웃**을 goroutine+channel로 걸어둠
  (`readWithTimeout`, `writeWithTimeout` 함수). 이게 없으면 서버가 꺼져있을 때
  Windows SMB 타임아웃(수십 초)까지 프로그램이 멈춰 보이는 문제가 있었음.

### Record 구조체
```go
type Record struct {
    ID       int    `json:"id"`
    Date     string `json:"date"`
    Equip    string `json:"equip"`
    Worker   string `json:"worker"`
    Category string `json:"category"`
    Content  string `json:"content"`
    Dong     string `json:"dong"`
}
```
ID는 **정수**입니다. v1.5.x 때 문자열 ID(`legacy-405` 형식)로 바꿨다가
전부 롤백하면서 다시 정수로 돌아왔습니다. 이후 절대 문자열로 바꾸지 마세요
(WebView2 바인딩에서 `dbUpdate(id int, ...)`, `dbDelete(id int)` 시그니처가
이 타입에 의존합니다).

**ID 대역 분리 (충돌 방지)**: JC01과 JC02는 `records.json`을 공유하는데,
네트워크가 끊긴 상태에서 각자 로컬 백업 기준으로 `nextID()`를 계산하다가
재접속 시 두 동이 같은 ID를 만들어낼 수 있었습니다. 이를 막기 위해 동별로
ID 대역을 나눴습니다.
- JC01: `100000`번대 (100001~199999)
- JC02: `200000`번대 (200001~299999)

`nextID()`는 자기 동의 대역 안에서만 최댓값을 찾으므로, 두 exe가 오프라인
상태로 각각 새 레코드를 추가해도 절대 겹치지 않습니다. `migrateIDOffsets()`
함수가 프로그램 시작 시 옛 방식(순차 정수, 대역 밖 ID)으로 저장된 데이터를
만나면 자동으로 새 대역으로 옮겨줍니다 (`migrateEquipNames()`와 같은 패턴).
`initial_data_v142.json`의 ID도 전부 `100000`을 더해서 이 규칙에 맞췄습니다.

이 대역 분리 때문에, "새 항목 모달 기본값"에서 근무자/구분을 채울 때
전체 레코드 중 ID 최댓값을 쓰면 안 됩니다 (JC02 ID가 항상 JC01보다 크므로).
반드시 같은 동으로 먼저 필터링한 뒤 그 안에서 최댓값을 찾아야 합니다
(`openModal()`의 `dongRecords` 참고).

### 테이블 렌더링: Flexbox 기반 (⚠️ table 태그 아님)
과거 `<table>` + `table-layout:fixed` + `<colgroup>`으로 컬럼 폭을 고정하려
했으나, **소스 코드는 맞는데도 실제 Windows WebView2에서 컬럼 폭이 깨지는
현상**이 재현되어 (헤더 테스트/시뮬레이션에서는 정상, 실기에서만 실패)
`<div>` + CSS Flexbox 구조로 완전히 교체했습니다 (v1.4.9).
`.frow`(행), `.fc-dong/.fc-d/.fc-e/.fc-w/.fc-c`(고정폭 칸),
`.fc-ct`(내용 칸, `flex:1`로 남는 공간 전부 차지) 구조를 유지하세요.

### 네트워크 상태 배지
- 초록/빨강 점(`#netbadge`)이 상단바에 있음
- **주의**: 별도의 상태 조회 함수(`dbGetPath` 단독 호출)를 만들었다가
  WebView2에서 그 호출만 무한 대기(hang)하는 현상이 있었습니다. 원인 불명.
  지금은 이미 정상 동작하는 `dbGetAll()` 응답 안에 네트워크 상태
  (`status`, `network`, `error` 필드)를 함께 실어 보내는 방식으로 우회했고,
  이게 안정적으로 동작합니다. **별도의 상태 전용 바인딩 호출을 새로 추가하지
  마세요** — 같은 hang이 재발할 가능성이 있습니다.
- 배지 색이 즉시 안 바뀌고 클릭해야 바뀌는 버그도 있었는데, 원인은
  "1384건 렌더링"처럼 무거운 동기 작업이 색상 변경 직후 바로 실행되면서
  브라우저가 리페인트할 틈을 안 준 것이었습니다. `applyNetInfo()` 호출 직후
  `await new Promise(r=>requestAnimationFrame(r))`로 한 프레임 양보한 뒤
  무거운 렌더링을 시작하도록 순서를 맞췄습니다 (v1.4.8).

### 설비명
- `ATW#21/22/31/32/41/42/51/52` → `#21~#52`로 통합됨 (v1.4.12).
  `ATW#61`만 예외로 원래 이름 유지.
- `migrateEquipNames()` 함수가 프로그램 시작 시 자동으로 기존 저장 파일의
  옛 이름을 새 이름으로 변환합니다. 설비명을 또 바꿀 일이 있으면 이 함수와
  `equipRenameMap`을 참고해서 같은 패턴으로 처리하세요.
- 필터는 다중선택 가능 (`selectedEquips` Set, 체크박스 드롭다운, v1.4.11).
  `EQUIP_BY_DONG` 딕셔너리가 실제 데이터에 있는 모든 설비명을 포함하는지
  주기적으로 대조 확인 필요 (v1.4.11 때 ATW#21~52가 목록에서 통째로
  빠져있던 걸 뒤늦게 발견한 적 있음 — `initial_data_v142.json`을 까보고
  실제 존재하는 `equip` 값과 드롭다운 목록을 비교하는 습관 들이세요).

### 새 항목 모달 기본값 (v1.4.10)
- 동: 현재 화면 필터의 동
- 날짜: 오늘
- 근무자/구분: **ID가 가장 큰(가장 최근 추가된) 레코드**의 값을 따라감
  (날짜 필드 기준이 아님 — 날짜는 사용자가 소급 입력할 수 있어서 신뢰 불가)

### 메모장 (v1.4.4)
- 테이블과 달력 사이에 위치, `localStorage` 기반 (서버 저장 아님, PC별 개인 메모)
- JC01/JC02는 저장 키가 다름 (`hk_memo` vs `hk_memo_jc02`) — 같은 PC에
  두 exe를 다 설치해도 메모가 안 섞임
- **WebView2 데이터 폴더 고정 (v1.4.14)**: `localStorage`는 WebView2 사용자
  데이터 폴더 안에 저장되는데, `DataPath`를 지정하지 않으면 라이브러리가
  기본값으로 `%AppData%\<exe 파일명>`을 쓴다. 버전 올릴 때 exe 파일명이
  바뀌면 폴더도 바뀌어 **메모가 매번 초기화된 것처럼 보이는 문제**가 있었다.
  `main()`에서 `DataPath: filepath.Join(getDataDir(), "webview2")`로 버전과
  무관하게 고정해 해결했다 (이후 버전 올려도 메모 유지). 단, 이 수정을 처음
  배포하는 순간엔 옛 폴더의 메모가 새 폴더로 이관되지 않아 한 번은 비워진다.

---

## 알려진 미해결 이슈

1. **백신 오탐**: `-s -w` 제거로 완화했지만 근본 해결(코드 서명)은 안 됨.
   사내 IT에 화이트리스트 요청하는 게 가장 현실적.
2. **JC02용 embed 데이터 없음**: `initial_data_v142.json`은 전부 JC01 데이터
   (1384건, dong="JC01"). JC02는 처음 실행하면 빈 상태로 시작함
   (정상 — 아직 2동 과거 데이터를 확보한 적이 없음).

---

## 디버깅 시 원칙 (과거 시행착오에서 얻은 교훈)

- **브라우저 시뮬레이션/Go 유닛테스트가 통과해도 실제 Windows WebView2에서
  실패하는 사례가 여러 번 있었습니다** (컬럼 폭, dbGetPath hang, 배지
  리페인트 타이밍 등). 중요한 변경은 반드시 실제 exe를 사용자가 실행해서
  확인받은 뒤 다음 단계로 넘어가세요. 헤드리스 브라우저 테스트는
  "명백한 로직 오류를 미리 거르는 용도"로만 신뢰하고, "Windows 실기에서
  똑같이 동작할 것"이라는 보장으로 여기지 마세요.
- 새 기능을 추가할 때는 **작게 나눠서 하나씩** 검증받는 게 낫습니다.
  한 번에 네트워크+휴지통+다중선택+새로고침을 다 넣었다가 문제 원인을
  못 찾아 전체 롤백한 전례가 있습니다 (v1.5.0 사건).
