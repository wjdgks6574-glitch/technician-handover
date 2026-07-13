# 인수인계 관리 프로그램 - 작업 인수인계 문서

## 현재 버전: v1.4.61

Go + WebView2 기반 Windows 데스크톱 앱. JC01(1동)/JC02(2동) 두 변형이 거의 동일한
소스 구조를 공유하며, 각각 별도의 .exe로 빌드됨. 상세 규칙/구현 상태는 `CLAUDE.md` 참고.

## 파일 구성

```
goapp/
├── main_jc01_v142.go.tmp   ← JC01용 소스 (빌드 시 main.go로 복사해서 사용)
├── main_jc02_v142.go.tmp   ← JC02용 소스
├── initial_data_v142.json  ← 프로그램에 내장(embed)되는 초기 데이터 1384건
├── go.mod
└── vendor/                 ← 오프라인 빌드용 의존성
```

`_v142` 파일 두 개가 유일한 정본. 다른 이름의 main_jc01.go 등이 있으면 무시.

## 빌드 명령

```bash
export GOPATH=$HOME/go
export PATH=$PATH:/usr/local/go/bin
cd goapp

cp main_jc01_v142.go.tmp main.go && gofmt -w main.go
GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc \
  go build -mod=vendor -ldflags="-H windowsgui" -o 인수인계관리_JC01_v1.4.61.exe .

cp main_jc02_v142.go.tmp main.go && gofmt -w main.go
GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc \
  go build -mod=vendor -ldflags="-H windowsgui" -o 인수인계관리_JC02_v1.4.61.exe .
```

`-ldflags`에 `-s -w` 넣지 말 것. 버전 올릴 때 두 소스 HTML의 `v1.4.X` 문자열도 함께 변경.

## 알려진 미해결 이슈

1. **백신 오탐**: `-s -w` 제거로 완화했지만 근본 해결(코드 서명)은 안 됨.
2. **JC02용 embed 데이터 없음**: `initial_data_v142.json`은 전부 JC01 데이터.
   JC02는 처음 실행하면 빈 상태로 시작(정상).

## 디버깅 시 원칙

- 브라우저 시뮬레이션/Go 유닛테스트가 통과해도 실제 Windows WebView2에서 실패하는
  사례가 있었음. 중요한 변경은 실제 exe로 확인받을 것.
- 새 기능은 작게 나눠서 하나씩 검증받는 게 낫다.
