# AR Viewer 설계 문서

- 작성일: 2026-10-07
- 상태: 승인됨 (구현 계획 작성 전)

## 1. 목적

3D 작업자가 완성된 AR용 모델(GLB/glTF)을 PC에서 열어 3D로 돌려보며 검토하는 데스크톱 앱.
**읽기 전용** — 원본 파일을 수정하거나 저장하지 않는다.

## 2. 범위

### 포함

- GLB / glTF 파일 열기 (툴바 버튼, 메뉴 Cmd/Ctrl+O, 드래그 앤 드롭)
- 마우스로 회전 · 줌 · 이동 (model-viewer `camera-controls`)
- 시점 리셋 (툴바 버튼, 메뉴 Cmd/Ctrl+0)
- 창 하나에 파일 하나. 여러 파일은 새 창(Cmd/Ctrl+N)으로 각각 띄운다
- macOS, Windows 지원
- 완전 오프라인 동작 (내부망 전용)

### 제외 (YAGNI)

- 모델 · 머티리얼 수정, 내보내기
- 조명 · 환경 변경 UI
- 애니메이션 재생, 모델 정보 패널, Variant 전환, 자동 회전
- 휴대폰 AR 테스트 (QR 전송 등)
- 파일 연결(더블클릭으로 열기), 최근 파일 목록
- 자동 업데이트, 코드 서명

## 3. 기술 스택

| 항목 | 선택 |
|---|---|
| 런타임 | Electron |
| 개발 · 번들 | electron-vite |
| 패키징 | electron-builder (`electron-builder.yml`) |
| UI | React + TypeScript |
| 3D 뷰어 | `@google/model-viewer` (npm, 앱에 번들) |
| 단위 테스트 | Vitest |
| E2E 테스트 | Playwright (Electron 지원) |

## 4. 고정 렌더링 설정

[modelviewer.dev/editor](https://modelviewer.dev/editor/) 기본값과 동일하게 고정한다. 사용자가 변경할 수 없다.

```html
<model-viewer
  camera-controls
  environment-image="neutral"
  exposure="1"
  shadow-intensity="1"
  shadow-softness="1"
  tone-mapping="neutral">
</model-viewer>
```

## 5. 아키텍처

```
┌─ Main 프로세스 ─────────────────────────────────────┐
│ • 창 관리: 새 창(Cmd/Ctrl+N), 창마다 독립 상태        │
│ • 앱 메뉴: 열기(Cmd/Ctrl+O), 새 창, 리셋(Cmd/Ctrl+0) │
│ • 파일 열기 대화상자 (.glb/.gltf 필터)                │
│ • 커스텀 프로토콜 `model://` — 로컬 파일 제공         │
│ • 외부 네트워크 요청 전면 차단                        │
└──────────────┬─────────────────────────────────────┘
               │ IPC
┌─ Preload ────┴─────────────────────────────────────┐
│ window.api = {                                      │
│   openFileDialog(): Promise<string | null>          │
│   getPathForFile(file: File): string                │
│   allowDirectory(filePath: string): Promise<void>   │
│   onMenuOpen(cb), onMenuReset(cb)                   │
│ }                                                   │
└──────────────┬─────────────────────────────────────┘
┌─ Renderer (React) ─────────────────────────────────┐
│ <App>                                               │
│  ├─ <Toolbar>     [열기] [리셋] ········ 파일명      │
│  ├─ <EmptyState>  드래그 앤 드롭 안내 + 파일 선택     │
│  └─ <Viewer>      <model-viewer> + 오류/경고 표시    │
└─────────────────────────────────────────────────────┘
```

### 폴더 구조

```
ar-viewer/
├─ electron-builder.yml
├─ electron.vite.config.ts
├─ resources/decoders/      # draco, basis(ktx2), meshopt 디코더 (오프라인용)
├─ src/
│  ├─ main/      index.ts, window.ts, menu.ts, protocol.ts, offline.ts
│  ├─ preload/   index.ts
│  └─ renderer/  index.html, main.tsx, App.tsx, components/{Toolbar,EmptyState,Viewer}.tsx
└─ test/
   ├─ unit/
   ├─ e2e/
   └─ fixtures/              # Khronos glTF 샘플
```

### 보안 설정

- `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`
- 렌더러는 `model://` 프로토콜로만 로컬 파일에 접근한다

## 6. 동작 흐름

### 파일 열기

```
경로 획득 (대화상자 / 드래그 앤 드롭 → webUtils.getPathForFile)
  → 확장자 검사 (.glb / .gltf, 대소문자 무시)
     ├ 실패 → "GLB/glTF 파일만 열 수 있습니다" 안내, 기존 모델 유지
     └ 통과 → main에 해당 파일의 폴더를 허용 폴더로 등록
            → <model-viewer src="model://<인코딩된 절대경로>">
            → 로딩 바 표시 → load 이벤트 시 렌더 완료
```

- 모델이 열린 창에서 새 파일을 열면 확인 없이 그 창의 모델을 **교체**한다.
- 창 제목과 툴바 오른쪽에 파일명을 표시한다. 마우스를 올리면 전체 경로를 툴팁으로 보여준다.

### `model://` 프로토콜

- glTF가 `.bin`과 텍스처를 상대 경로로 참조하므로, blob URL이 아니라 실제 경로를 기반으로 한 URL을 사용한다.
- 허용 폴더는 앱 전체에서 하나의 집합으로 관리한다. 어느 창에서든 파일을 열면 그 파일의 폴더가 추가된다. (`protocol.handle`은 요청한 창을 알 수 없어서 창별 관리는 하지 않는다)
- 요청 경로를 정규화한 뒤 허용 폴더 하위인지 검사한다. 하위가 아니면(`../` 우회 포함) 403을 반환한다.
- 파일이 없으면 404를 반환하고, 해당 경로를 "누락 파일" 목록에 기록한다. 렌더러는 모델의 `load` 또는 `error` 이벤트 뒤에 이 목록을 가져가서(가져가면 비워짐) 경고로 표시한다. 새 파일을 열 때도 목록을 비운다.

### 리셋

- 모델을 처음 불러왔을 때의 카메라 시점(`camera-orbit`, `camera-target`, `field-of-view` 기본값)으로 되돌린다.
- 단축키는 Cmd/Ctrl+0이다. Cmd/Ctrl+R은 Electron 기본 새로고침과 겹쳐서 쓰지 않는다.

### 창

- Cmd/Ctrl+N: 빈 창을 새로 연다. 기존 창에서 비켜난 위치에 띄우고, 창마다 독립 상태를 가진다.
- macOS: 마지막 창을 닫아도 앱은 유지된다. Dock을 클릭하면 새 창이 열린다.
- Windows: 마지막 창을 닫으면 앱이 종료된다.

## 7. 오프라인 보장

- model-viewer는 기본적으로 Draco / KTX2 / Meshopt 디코더를 gstatic CDN에서 내려받는다. 이 파일들을 `resources/decoders/`에 포함하고 경로를 지정한다.
  ```ts
  ModelViewerElement.dracoDecoderLocation   = '<앱 내부>/draco/'
  ModelViewerElement.ktx2TranscoderLocation = '<앱 내부>/basis/'
  ModelViewerElement.meshoptDecoderLocation = '<앱 내부>/meshopt_decoder.js'
  ```
- 디코더 파일은 설치된 `three` 패키지(model-viewer가 의존)의 `examples/jsm/libs/`에서 `npm install` 시(postinstall) 복사해 버전을 일치시킨다.
- Meshopt 디코더는 model-viewer 번들에 이미 들어 있다. 하지만 `meshoptDecoderLocation`이 지정돼야 활성화되므로, 빈 로컬 스크립트(`meshopt-ready.js`)를 지정한다.
- 설정은 `self.ModelViewerElement` 전역 객체로 하고, `@google/model-viewer`를 import하기 전에 실행한다.
- CSP로 `connect-src`, `script-src` 등을 `'self'`와 `model:`로 제한한다.
- main의 `session.webRequest.onBeforeRequest`에서 `http(s)://` 요청을 모두 취소하고 로그를 남긴다.
- `neutral` 환경맵은 코드로 생성되므로 네트워크가 필요 없다.
- 네트워크는 빌드할 때(npm, Electron 바이너리 다운로드)만 필요하다. 빌드는 인터넷이 되는 PC에서 하고, 설치 파일만 내부망으로 전달한다.

## 8. 오류 처리

| 상황 | 동작 |
|---|---|
| 지원하지 않는 확장자 | "GLB/glTF 파일만 열 수 있습니다". 기존 모델 유지 |
| 파싱 실패 (`error` 이벤트) | "모델을 불러오지 못했습니다: 파일명" + 상세 메시지. 툴바는 계속 사용 가능 |
| `.bin` · 텍스처 누락 (404) | 가능한 만큼 표시하고 "누락된 파일: …" 경고 표시 |
| 허용 폴더 밖 참조 (403) | 차단하고 누락 파일 경고에 포함 |
| 큰 파일 | model-viewer 로딩 바 표시 |

## 9. 테스트

- **단위 (Vitest)**
  - 확장자 검사
  - 경로 ↔ `model://` URL 변환 (공백, 한글, Windows 드라이브 문자 경로 포함)
  - 허용 폴더 검사 (`../` 및 인코딩된 `%2E%2E` 우회)
  - 리셋 시 카메라 기본값 적용
- **E2E (Playwright + Electron)**
  - 앱 실행 → 빈 화면 표시
  - GLB 열기 → `load` 이벤트 → 창 제목에 파일명 표시
  - glTF + bin + 텍스처 열기 → 누락 경고 없음
  - 리셋 → 카메라가 초기값으로 복귀
  - 새 창 → 독립 상태
  - Draco 압축 GLB 열기 → 정상 로드, **외부 요청 0건**
- **픽스처**: Khronos glTF Sample Assets 중 일반 GLB 1개, glTF(분리형) 1개, Draco 압축 1개

## 10. 배포

- `npm run build:mac` → `.dmg` (arm64 + x64)
- `npm run build:win` → NSIS `.exe` (x64)
- 정식 코드 서명 없음(사내 배포). macOS는 Apple Silicon 실행을 위해 ad-hoc 서명(`identity: "-"`)만 한다. 설치 안내 문서에 아래 내용을 적는다.
  - macOS 15 이상: 처음 실행하면 차단됨 → 시스템 설정 → 개인정보 보호 및 보안 → "그래도 열기". 또는 터미널에서 `xattr -dr com.apple.quarantine "/Applications/AR Viewer.app"`
  - Windows: SmartScreen → "추가 정보" → "실행"
- Windows 설치 파일은 Windows PC에서 빌드하는 것을 권장한다.
