# Windows 빌드 절차 (ui/modernize-step0)

이 소스는 `open-rpa/openrpa`의 하드포크다. UI 현대화 1차 변경 3건이 들어 있고,
아직 **한 번도 빌드된 적이 없다.** 이 문서의 목적은 빌드를 성공시키는 것이다.

## 0. 사전 확인 — 여기서 막히면 나머지는 의미 없다

**(1) NuGet 접근**

브라우저에서 다음 주소를 연다.

```
https://api.nuget.org/v3/index.json
```

JSON 텍스트가 보이면 통과. 차단 페이지가 뜨면 **이 PC에서는 빌드할 수 없다.**
프로젝트 42개가 NuGet 패키지 33개를 직접 참조하고, 그중 Roslyn(`Microsoft.CodeAnalysis.*`)과
`Emgu.CV`의 전이 의존성이 수백 개다. 오프라인 번들링은 현실적이지 않다.

**(2) 관리자 권한**

VS2022 설치와 구성요소 추가에 필요하다. 없으면 (1)이 통과해도 막힌다.

## 1. Visual Studio 2022

Visual Studio Installer에서:

- **워크로드**: `.NET 데스크톱 개발`
- **개별 구성 요소**에서 검색해 추가:
  - `Windows Workflow Foundation` — **기본 선택이 아니다.** 없으면
    `System.Activities.Presentation` 참조가 깨지고 WF 디자이너가 빌드되지 않는다
  - `.NET Framework 4.6.2 타게팅 팩`

## 2. 빌드

1. `OpenRPA.sln` 열기
2. 솔루션 탐색기에서 `OpenRPA` 프로젝트 → 우클릭 → **시작 프로젝트로 설정**
3. 첫 빌드 전 **복원**이 끝날 때까지 기다린다 (패키지가 많아 수 분 걸린다)
4. 빌드 → 실행

## 3. `layout.config` 삭제 — 안 하면 Output 변경이 안 보인다

`MainWindow.xaml.cs`의 `LoadLayout()`은 저장된 레이아웃 파일을 찾으면
XAML에 정의된 레이아웃을 **통째로 대체한다.** 기존에 OpenRPA를 쓴 적이 있으면
파일이 남아 있다. 실행 전에 지운다.

```
%USERPROFILE%\Documents\OpenRPA\layout.config
```

`%APPDATA%\OpenRPA\settings.json`이 존재하면 대신 이쪽:

```
%APPDATA%\OpenRPA\layout.config
```

## 4. 이번 변경 내용

| 변경 | 위치 |
|---|---|
| AvalonDock 테마 `VS2010Theme` → `MetroTheme` | `OpenRPA/MainWindow.xaml` |
| Output을 자동숨김 스트립에서 하단 도킹 패널로 (`DockHeight=180`) | `OpenRPA/MainWindow.xaml` |
| `LoadLayout()` fallback에 Output 펼치기 추가 | `OpenRPA/MainWindow.xaml.cs` |
| `FontSize=14`, 리본행·상태바 고정높이(120/25) → `Auto` | `OpenRPA/MainWindow.xaml` |

`MetroTheme`를 고른 이유: `Extended.Wpf.Toolkit 4.2.0`에 실제로 포함된 테마 어셈블리는
Metro / Aero / VS2010 셋뿐이고, 그중 가장 평평하다. `Vs2013LightTheme`는 이 버전에 없다.

## 5. 실행 후 확인할 것

1. **Metro 테마 실물** — 좌우 패널 제목줄의 회색 그라데이션과 문서 탭의 파란 띠가
   얼마나 평평해졌는지. 마음에 안 들면 `MainWindow.xaml`의 테마 한 줄만
   `GenericTheme` / `ExpressionLightTheme`로 바꿔 비교한다
2. **Output 패널** — 하단에 180px 높이로 펼쳐져 있어야 한다.
   여전히 얇은 탭이면 3번(`layout.config`)을 안 지운 것이다
3. **잘리는 곳** — 리본행을 `Auto`로 풀었지만 `<Ribbon ... Margin="0,-22,0,0">`이
   남아 있어 위쪽 22px를 계속 잘라낸다. 리본 상단이 이상하면 이 값이 원인이다
4. **탭 제목만 작게 남는지** — `OpenRPA/Views/CloseableHeader.xaml`에
   `FontSize`가 하드코딩돼 있어 상속을 받지 않는다
5. **WF 캔버스 글씨도 커졌는지** — 상속되면 같이 커진다. 의도한 변경은 아니다.
   커진 게 나으면 두고, 아니면 캔버스 쪽만 되돌린다

## 6. 빌드 실패 시

에러 전문을 그대로 복사해 둔다. 특히 아래는 원인이 정해져 있다.

- `System.Activities.Presentation`을 찾을 수 없음 → 1번의 Windows Workflow Foundation 미설치
- 패키지 복원 실패 → 0번(1)의 NuGet 차단
- `net462` 타게팅 팩 없음 → 1번의 타게팅 팩 미설치
