# Windows에서 실행하기

이 소스는 `open-rpa/openrpa`의 하드포크다. 빌드는 GitHub Actions의 Windows 러너가
대신 하므로, **실행하는 PC에는 Visual Studio도 NuGet 접근도 관리자 권한도 필요 없다.**
받아서 실행만 하면 된다.

## 1. 빌드 결과물 받기

GitHub에 접근 가능한 기계(Mac)에서:

```
https://github.com/oortcraft/openrpa/actions
```

가장 최근의 성공한 `build-windows` 실행을 열고 아래쪽 **Artifacts**에서
`openrpa-dist`를 내려받는다 (약 109MB). 그걸 Google Drive에 올려 Windows에서 받는다.

## 2. 실행

압축을 풀면 **`net462\`** 하위 폴더가 있고 그 안에 `OpenRPA.exe`가 있다.
(SDK 형식 프로젝트가 출력 경로 뒤에 타겟 프레임워크를 붙인다.)

```
net462\OpenRPA.exe
```

.NET Framework 4.8은 Windows 10/11에 기본 포함이라 런타임을 따로 깔 필요는 없다.

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

설정과 워크플로는 이 폴더를 기존 설치본(MSI)과 공유한다. 기존 워크플로가 그대로
보이는 건 그래서다.

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

**스크린샷을 찍어 둔다.** 다음 결정(리본을 걷어내고 사이드바로 갈지)의 유일한 근거다.

## 부록 — CI가 하는 일

`.github/workflows/build-windows.yml`. `ui/**` 브랜치에 push하면 돈다.
빌드를 세우기까지 걸렸던 것들:

- `LiteDB`, `Open3270`, `vb5250`가 서브모듈이라 `submodules: recursive`가 필요했다
- `OpenRPA.Interfaces` / `OpenRPA.Net` / `OpenRPA.WorkItems.Activities`가 `net46`을
  노리는데 러너에 4.6 타게팅 팩이 없었다. `Microsoft.NETFramework.ReferenceAssemblies`는
  WPF가 XAML 컴파일용으로 만드는 `*_wpftmp.csproj` 임시 프로젝트에서 참조 경로가
  유실돼 소용없었고, `net462`로 리타게팅해 해결했다
- `OpenRPA.SAP` / `OpenRPA.SAPBridge`는 SAP GUI가 깔린 기계에만 있는 COM interop을
  요구한다. 워크플로가 빌드 직전 솔루션에서 제거한다
- 일부 프로젝트가 빌드 후 `nuget.exe push`로 nuget.org에 배포를 시도한다. 업스트림
  작성자의 배포 설비다. `-p:GeneratePackageOnBuild=false`로 끈다

## 부록 — 로컬에서 직접 빌드하려면

CI를 쓰면 필요 없지만, Windows에서 직접 빌드하려면 VS2022에 워크로드
`.NET 데스크톱 개발`과 개별 구성 요소 `Windows Workflow Foundation`(기본 선택이 아니다),
그리고 `.NET Framework 4.6.2 타게팅 팩`이 있어야 한다. NuGet 복원을 위해
`https://api.nuget.org/v3/index.json` 접근도 필요하다.
