# tech.md — 기술 명세서 (Technical Specification)

> **문서 버전**: 3.2
> **최종 수정일**: 2026-03-19
> **관련 문서**: [CLAUDE.md](./CLAUDE.md) (설계 가이드), [PRD.md](./PRD.md) (제품 요구사항)
> **상태**: Sprint 11~13 완료 (ST 블록 에디터 + ST 텍스트 에디터 + PLC 코드 내보내기)

---

## 목차

1. [개발 환경 설정](#1-개발-환경-설정)
2. [프로젝트 초기화](#2-프로젝트-초기화)
3. [디렉토리 구조 상세](#3-디렉토리-구조-상세)
4. [Electron 메인 프로세스](#4-electron-메인-프로세스)
5. [React 프론트엔드](#5-react-프론트엔드)
6. [상태 관리 (Zustand)](#6-상태-관리-zustand)
7. [WebSocket 통신 계층](#7-websocket-통신-계층)
8. [Python 백엔드](#8-python-백엔드)
9. [Flow Engine 상세](#9-flow-engine-상세)
10. [하드웨어 컨트롤러](#10-하드웨어-컨트롤러)
11. [시뮬레이션 모드](#11-시뮬레이션-모드)
12. [ESP32 펌웨어](#12-esp32-펌웨어)
13. [빌드 및 패키징](#13-빌드-및-패키징)
14. [테스트 전략](#14-테스트-전략)
15. [오류 처리 정책](#15-오류-처리-정책)

---

## 1. 개발 환경 설정

### 필수 소프트웨어

| 도구 | 버전 | 용도 |
|------|------|------|
| Node.js | 18 LTS 이상 | Electron + React 빌드 |
| npm | 9+ (Node.js 번들) | 패키지 관리 |
| Python | 3.11+ | 백엔드 Flow Engine |
| pip | 최신 | Python 패키지 관리 |
| Git | 최신 | 버전 관리 |
| Arduino IDE 또는 PlatformIO | 최신 | ESP32 펌웨어 업로드 |

### Node.js 의존성

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@dnd-kit/core": "^6.1.0",
    "@dnd-kit/sortable": "^8.0.0",
    "@dnd-kit/utilities": "^3.2.2",
    "zustand": "^4.5.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "electron": "^33.0.0",
    "electron-builder": "^24.0.0",
    "typescript": "^5.3.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0",
    "vite": "^5.0.0",
    "@vitejs/plugin-react": "^4.2.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@types/uuid": "^9.0.0"
  }
}
```

### Python 의존성 (`python/requirements.txt`)

```
websockets>=12.0
pyserial>=3.5
opencv-python>=4.8.0
numpy>=1.24.0
xarm-python-sdk>=1.13.0
```

### 환경 변수 / 설정

프로그램은 환경 변수 대신 `.bridge.json` 파일과 런타임 설정 UI를 사용한다. 별도 `.env` 파일 불필요.

---

## 2. 프로젝트 초기화

### 프론트엔드 셋업

```bash
# 프로젝트 루트에서
npm init -y
npm install react react-dom @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities zustand uuid
npm install -D electron electron-builder typescript tailwindcss postcss autoprefixer vite @vitejs/plugin-react @types/react @types/react-dom @types/uuid

# Tailwind 초기화
npx tailwindcss init -p
```

### `tsconfig.json` 핵심 설정

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "outDir": "./dist",
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["src/**/*", "electron/**/*"]
}
```

### `tailwind.config.js`

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ['./src/**/*.{ts,tsx}', './index.html'],
  theme: {
    extend: {
      colors: {
        // 카드 카테고리 색상
        'card-basic': '#3B82F6',      // 파랑
        'card-conveyor': '#22C55E',   // 초록
        'card-sensor': '#EAB308',     // 노랑
        'card-vision': '#A855F7',     // 보라
        'card-robot': '#EF4444',      // 빨강
        // 상태 색상
        'status-connected': '#22C55E',
        'status-error': '#EF4444',
        'status-idle': '#9CA3AF',
        // E-STOP
        'estop': '#DC2626',
      },
      fontSize: {
        'card': '16px',     // 카드 텍스트 최소
        'body': '14px',     // 본문 최소
      },
      minWidth: {
        'click': '44px',    // 최소 클릭 영역
      },
      minHeight: {
        'click': '44px',
      },
    },
  },
  plugins: [],
};
```

### Python 환경 셋업

```bash
cd python
python -m venv venv
source venv/bin/activate   # macOS/Linux
# venv\Scripts\activate    # Windows

pip install -r requirements.txt
```

---

## 3. 디렉토리 구조 상세

```
bridge-program/
├── package.json
├── tsconfig.json
├── tailwind.config.js
├── postcss.config.js
├── vite.config.ts               # Vite 빌드 설정
├── index.html                   # Electron renderer 진입점
│
├── electron/                    # === Electron 메인 프로세스 ===
│   ├── main.ts                  # BrowserWindow 생성, 앱 생명주기
│   ├── preload.ts               # contextBridge IPC 노출
│   └── python-manager.ts        # Python 자식 프로세스 관리
│
├── src/                         # === React 프론트엔드 ===
│   ├── App.tsx                  # 루트 컴포넌트
│   ├── main.tsx                 # ReactDOM.createRoot
│   ├── index.css                # Tailwind @import
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Header.tsx       # 상단바: 프로젝트명 + 실행 버튼 + E-STOP
│   │   │   ├── StatusBar.tsx    # 하단 상태바: 장비 연결 상태
│   │   │   └── MainLayout.tsx   # 3패널 구조 (좌/중/우)
│   │   │
│   │   ├── card-editor/
│   │   │   ├── CardPalette.tsx  # 왼쪽 패널: 카테고리별 카드 목록
│   │   │   ├── FlowCanvas.tsx   # 중앙 패널: 드래그&드롭 캔버스
│   │   │   ├── CardItem.tsx     # 개별 카드 렌더링 (팔레트/캔버스 공용)
│   │   │   └── CardParamEditor.tsx  # 파라미터 편집 인라인/팝업
│   │   │
│   │   ├── camera/
│   │   │   ├── CameraViewer.tsx     # MJPEG <img> 스트림
│   │   │   └── DetectionOverlay.tsx # 색상 감지 결과 오버레이
│   │   │
│   │   ├── monitor/
│   │   │   ├── HardwareStatus.tsx   # 장비 상태 표시
│   │   │   ├── ConnectionPanel.tsx  # 연결 설정 + 그리퍼 선택
│   │   │   └── PositionEditor.tsx   # 로봇 위치 교시 (HOME/A/B/C)
│   │   │
│   │   └── safety/
│   │       └── EStopButton.tsx      # 비상정지 버튼
│   │
│   ├── stores/
│   │   ├── flowStore.ts         # 카드 플로우 + 실행 상태
│   │   ├── hardwareStore.ts     # 장비 연결 상태
│   │   └── settingsStore.ts     # IP, 포트, 모드 설정
│   │
│   ├── types/
│   │   ├── cards.ts             # CardDefinition, FlowCard, CardParamDef
│   │   ├── hardware.ts          # HardwareStatus, ConnectionState
│   │   └── protocol.ts          # UIMessage, PYMessage 타입
│   │
│   └── utils/
│       ├── websocket-client.ts  # WebSocket 연결/메시지 핸들러
│       └── card-definitions.ts  # 25종 카드 정의 데이터
│
├── python/                      # === Python 백엔드 ===
│   ├── requirements.txt
│   ├── main.py                  # 엔트리포인트 (서버 시작)
│   │
│   ├── flow_engine/
│   │   ├── __init__.py
│   │   ├── engine.py            # FlowEngine 클래스 (카드 순차 실행)
│   │   ├── card_executor.py     # 카드 타입별 실행 디스패치
│   │   └── state_machine.py     # EngineState 열거형 + 전이 로직
│   │
│   ├── hardware/
│   │   ├── __init__.py
│   │   ├── robot_controller.py  # RobotController (xArm SDK 래핑)
│   │   ├── esp32_controller.py  # ESP32Controller (pyserial)
│   │   ├── camera_controller.py # CameraController (HTTP 캡처 + OpenCV)
│   │   └── safety_manager.py    # SafetyManager (E-STOP, 제한)
│   │
│   ├── protocol/
│   │   ├── __init__.py
│   │   ├── ws_server.py         # WebSocketServer 클래스
│   │   └── messages.py          # 메시지 생성/파싱 유틸
│   │
│   └── simulation/
│       ├── __init__.py
│       └── simulator.py         # SimulatedHardware (Mock)
│
├── firmware/                    # === ESP32 펌웨어 ===
│   ├── conveyor_controller/
│   │   ├── conveyor_controller.ino   # 메인 스케치
│   │   ├── motor.h / motor.cpp       # TB6600 제어
│   │   ├── sensor.h / sensor.cpp     # E18-D80NK 읽기
│   │   └── protocol.h / protocol.cpp # JSON Serial 프로토콜
│   └── esp32_cam/
│       └── (기본 CameraWebServer 스케치 기반)
│
└── docs/                        # === 문서 ===
    ├── PRD.md
    ├── tech.md                  # (이 문서)
    └── todo.md
```

---

## 4. Electron 메인 프로세스

### `electron/main.ts`

```typescript
// 핵심 책임:
// 1. BrowserWindow 생성 (preload 스크립트 지정)
// 2. Python 프로세스 시작/종료 (python-manager)
// 3. 앱 종료 시 cleanup

import { app, BrowserWindow } from 'electron';
import { PythonManager } from './python-manager';

let mainWindow: BrowserWindow | null = null;
let pythonManager: PythonManager | null = null;

app.whenReady().then(async () => {
  // Python 백엔드 시작
  pythonManager = new PythonManager();
  await pythonManager.start();

  // 메인 윈도우
  mainWindow = new BrowserWindow({
    width: 1400,
    height: 900,
    minWidth: 1200,
    minHeight: 700,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
    },
  });

  // dev: Vite dev server / prod: 빌드된 HTML
  if (process.env.NODE_ENV === 'development') {
    mainWindow.loadURL('http://localhost:5173');
  } else {
    mainWindow.loadFile(path.join(__dirname, '../dist/index.html'));
  }
});

// 모든 창 닫히면 Python 종료 후 앱 종료
app.on('window-all-closed', async () => {
  await pythonManager?.stop();
  app.quit();
});
```

### `electron/python-manager.ts`

```typescript
// 핵심 책임:
// 1. Python 자식 프로세스 spawn (python/main.py)
// 2. stdout/stderr 로깅
// 3. 프로세스 상태 감시 (비정상 종료 감지)
// 4. 앱 종료 시 확실한 kill

import { spawn, ChildProcess } from 'child_process';

export class PythonManager {
  private process: ChildProcess | null = null;
  private pythonPath: string;  // 'python3' 또는 venv 경로

  async start(): Promise<void> {
    // python/main.py 실행
    // WebSocket 서버 준비 완료까지 대기 (stdout에서 "ready" 메시지)
  }

  async stop(): Promise<void> {
    // SIGTERM → 3초 대기 → SIGKILL
    // 포트 8765 해제 확인
  }

  isRunning(): boolean { /* ... */ }
}
```

### `electron/preload.ts`

```typescript
// contextBridge로 renderer에 최소한의 API만 노출
// WebSocket은 renderer에서 직접 연결하므로 IPC는 최소화

import { contextBridge } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // 파일 저장/불러오기 다이얼로그
  saveFile: (data: string) => ipcRenderer.invoke('save-file', data),
  openFile: () => ipcRenderer.invoke('open-file'),
  // PLC 코드 내보내기 (Sprint 13)
  exportSTFile: (data: string, defaultName: string, ext: string) =>
    ipcRenderer.invoke('dialog:exportSTFile', data, defaultName, ext),
});
```

---

## 5. React 프론트엔드

### 컴포넌트 계층 구조

```
App.tsx
└── MainLayout.tsx
    ├── Header.tsx
    │   ├── 프로젝트명 / 파일명
    │   ├── 실행 제어 버튼 (▶ ⏸ ⏹)
    │   └── EStopButton.tsx
    ├── (좌) CardPalette.tsx
    │   └── CardItem.tsx (반복, draggable)
    ├── (중) FlowCanvas.tsx
    │   ├── CardItem.tsx (반복, sortable)
    │   └── CardParamEditor.tsx (선택된 카드)
    ├── (우) 사이드 패널
    │   ├── CameraViewer.tsx
    │   │   └── DetectionOverlay.tsx
    │   ├── HardwareStatus.tsx
    │   └── ConnectionPanel.tsx
    └── StatusBar.tsx
```

### 주요 컴포넌트 설계

#### `CardPalette.tsx`
- 카테고리별 아코디언 UI (접기/펼치기)
- `card-definitions.ts`에서 정의를 읽어 렌더링
- 각 카드는 `@dnd-kit`의 `useDraggable` 적용
- 카드를 **복제(clone)**하여 캔버스에 드롭 (원본은 팔레트에 유지)

#### `FlowCanvas.tsx`
- `@dnd-kit/sortable`로 카드 순서 변경
- 팔레트에서 드롭 시 새 `FlowCard` 생성 (`uuid` 할당)
- 카드 간 연결선은 CSS border/pseudo-element로 처리
- 빈 상태: "왼쪽에서 카드를 끌어다 놓으세요!" 안내 표시
- 현재 실행 중인 카드: `ring-2 ring-blue-500 animate-pulse` 스타일

#### `CardItem.tsx`
- props로 `mode: 'palette' | 'canvas'` 구분
- palette 모드: 드래그 소스, 클릭 시 설명 툴팁
- canvas 모드: 정렬 가능, 클릭 시 파라미터 편집기 표시, 삭제 버튼
- 카테고리 색상 왼쪽 보더로 표시

#### `EStopButton.tsx`
- 화면 우측 상단 고정 위치
- `bg-estop` 빨간 원형, 크기 60px 이상
- 클릭 시 즉시 WebSocket `e_stop` 전송 (확인 대화상자 없음)
- 한번 눌리면 비활성화 상태로 전환 + 복구 안내 표시

#### `CameraViewer.tsx`
- `<img src="http://{ip}:81/stream" />` 태그로 MJPEG 표시
- 연결 실패 시 placeholder 이미지 + "카메라 연결 필요" 텍스트
- `DetectionOverlay`: 색상 감지 결과를 카메라 뷰 위에 오버레이

---

## 6. 상태 관리 (Zustand)

### `stores/flowStore.ts`

```typescript
import { create } from 'zustand';
import { FlowCard } from '@/types/cards';

type EngineState = 'IDLE' | 'RUNNING' | 'PAUSED' | 'WAITING' | 'ERROR' | 'E_STOP';

interface FlowStore {
  // === 상태 ===
  cards: FlowCard[];             // 캔버스의 카드 목록
  engineState: EngineState;      // 현재 실행 상태
  currentStep: number;           // 현재 실행 중인 카드 인덱스 (-1: 미실행)
  selectedCardUid: string | null; // 파라미터 편집 대상

  // === 카드 조작 ===
  addCard: (definitionId: string, index?: number) => void;
  removeCard: (uid: string) => void;
  moveCard: (fromIndex: number, toIndex: number) => void;
  updateCardParams: (uid: string, params: Record<string, any>) => void;
  selectCard: (uid: string | null) => void;
  clearFlow: () => void;

  // === 실행 상태 ===
  setEngineState: (state: EngineState) => void;
  setCurrentStep: (step: number) => void;

  // === 저장/불러오기 ===
  exportFlow: () => object;        // .bridge.json 데이터 생성
  importFlow: (data: object) => void; // .bridge.json 데이터 로드
}
```

### `stores/themeStore.ts`

```typescript
// 다크/라이트 테마 — localStorage 영속 + 즉시 class 적용 (FOUC 방지)
export type Theme = 'dark' | 'light'

interface ThemeState {
  theme: Theme
  toggleTheme: () => void
}

// 초기화: 저장된 값 읽어 <html> 클래스에 즉시 반영
const saved = (localStorage.getItem('facto-theme') as Theme) || 'dark'
if (saved === 'dark') document.documentElement.classList.add('dark')

export const useThemeStore = create<ThemeState>((set) => ({
  theme: saved,
  toggleTheme: () => set((s) => {
    const next: Theme = s.theme === 'dark' ? 'light' : 'dark'
    localStorage.setItem('facto-theme', next)
    document.documentElement.classList[next === 'dark' ? 'add' : 'remove']('dark')
    return { theme: next }
  })
}))
```

### `stores/hardwareStore.ts`

```typescript
type ConnectionStatus = 'disconnected' | 'connecting' | 'connected' | 'error';

interface HardwareStore {
  // === 연결 상태 ===
  robot: HwConnectionStatus;  // 'connected' | 'disconnected' | 'simulation' | 'error'
  esp32: HwConnectionStatus;
  camera: HwConnectionStatus;
  cameraIp: string;

  // === 엔진 상태 ===
  engineState: string;
  currentCardUid: string;
  sensorIr1: boolean;
  sensorIr2: boolean;

  // === 로봇 Live Control ===
  teachingMode: boolean;
  jogStepSize: number;          // 1/2/5/10/20 mm
  robotSpeed: number;           // 10~300 mm/s
  livePosition: RobotPosition | null;
  robotPositions: Record<string, RobotPosition>;

  // === GPIO ===
  gpioState: GpioState | null;  // { ci[], co[], ai[], tgpio[] }
}
```

### `stores/settingsStore.ts`

```typescript
interface SettingsStore {
  robotIp: string;         // 기본 '192.168.1.100'
  esp32Port: string;       // 기본 '' (수동 선택)
  cameraIp: string;        // 기본 '192.168.1.101'
  wsPort: number;          // 기본 8765

  // Sprint 9: 단계별 해금
  unlockLevel: 1 | 2 | 3 | 4;
  plcBadgeMode: boolean;
  isCategoryUnlocked: (cat: PaletteCategory) => boolean;

  // Sprint 11: 에디터 모드 전환
  editorMode: 'card' | 'st';  // 기본 'card'
  setEditorMode: (mode: 'card' | 'st') => void;

  setRobotIp: (ip: string) => void;
  setESP32Port: (port: string) => void;
  setCameraIp: (ip: string) => void;
}
```

---

## 7. WebSocket 통신 계층

### `src/utils/websocket-client.ts`

```typescript
// 싱글턴 WebSocket 클라이언트
// 연결 관리 + 메시지 라우팅

class WebSocketClient {
  private ws: WebSocket | null = null;
  private url: string = 'ws://localhost:8765';
  private reconnectInterval: number = 3000;  // 3초
  private maxReconnectAttempts: number = 10;
  private handlers: Map<string, (payload: any) => void> = new Map();

  connect(): void {
    // WebSocket 연결
    // onopen: 연결 상태 업데이트
    // onmessage: type별 핸들러 디스패치
    // onclose: 자동 재연결 시도
    // onerror: 오류 로깅
  }

  send(type: string, payload: object): void {
    // JSON.stringify({ type, payload }) 전송
    // 연결 안 됐으면 큐에 저장
  }

  on(type: string, handler: (payload: any) => void): void {
    // 메시지 타입별 핸들러 등록
  }

  off(type: string): void {
    // 핸들러 제거
  }

  disconnect(): void {
    // 연결 해제 (재연결 중단)
  }
}

export const wsClient = new WebSocketClient();
```

### 메시지 흐름 예시: 플로우 실행

```
[사용자] 실행 버튼 클릭
    │
    ▼
[React] 확인 대화상자 표시 → 사용자 "실행" 클릭
    │
    ▼
[flowStore] engineState → 'RUNNING'
    │
    ▼
[wsClient] send('execute_flow', { cards: flowStore.cards })
    │
    ▼ WebSocket
    │
[Python ws_server] 수신 → flow_engine.execute(cards)
    │
    ▼ (각 카드 실행 시)
[Python] send('state_update', { state: 'RUNNING', step: 0 })
[Python] send('card_complete', { cardUid: '...', result: {} })
    │
    ▼ WebSocket
    │
[wsClient] on('state_update') → flowStore.setCurrentStep(0)
[wsClient] on('card_complete') → 다음 카드 하이라이트
    │
    ▼ (전체 완료 시)
[Python] send('state_update', { state: 'IDLE', step: -1 })
    │
    ▼
[flowStore] engineState → 'IDLE'
```

### 메시지 타입 정의 (`src/types/protocol.ts`)

```typescript
// === UI → Python ===
export type UIMessageType =
  | 'execute_flow'
  | 'pause'
  | 'resume'
  | 'e_stop'
  | 'connect_hw'       // {target, ip?, port?, gripper_type?}
  | 'disconnect_hw'
  | 'list_ports'
  | 'get_robot_position'
  | 'update_position'; // {name, position: RobotPosition}

export interface UIMessage {
  type: UIMessageType;
  payload: Record<string, any>;
}

// === Python → UI ===
export type PYMessageType =
  | 'state_update'
  | 'hw_status'
  | 'detection'
  | 'error'
  | 'card_complete'
  | 'robot_position'   // {x,y,z,roll,pitch,yaw}
  | 'serial_ports'     // {ports: [{device, description, hwid}]}
  | 'sensor_state'     // {ir1, ir2}
  | 'position_update'; // {name, ok, message}

export interface PYMessage {
  type: PYMessageType;
  payload: Record<string, any>;
}

// 상세 페이로드 타입
export interface StateUpdatePayload {
  state: 'IDLE' | 'RUNNING' | 'PAUSED' | 'WAITING' | 'ERROR' | 'E_STOP';
  step: number;
}

export interface HWStatusPayload {
  robot: 'connected' | 'disconnected' | 'error';
  esp32: 'connected' | 'disconnected' | 'error';
  camera: 'connected' | 'disconnected' | 'error';
}

export interface DetectionPayload {
  color: string;          // 'red' | 'blue' | 'green' | 'yellow' | 'unknown'
  confidence: number;     // 0.0 ~ 1.0
}

export interface ErrorPayload {
  message: string;        // 한글 오류 메시지
  code: string;           // 'HW_ROBOT_DISCONNECT', 'HW_SERIAL_TIMEOUT' 등
}

export interface CardCompletePayload {
  cardUid: string;
  result: Record<string, any>;
}
```

---

## 8. Python 백엔드

### `python/main.py` — 엔트리포인트

```python
"""
메인 엔트리포인트.
WebSocket 서버를 시작하고, Flow Engine과 하드웨어 컨트롤러를 초기화한다.
Electron의 python-manager.ts가 이 파일을 spawn한다.
"""
import asyncio
from protocol.ws_server import WebSocketServer
from flow_engine.engine import FlowEngine
from hardware.safety_manager import SafetyManager
from simulation.simulator import SimulatedHardware

async def main():
    safety = SafetyManager()
    engine = FlowEngine(safety_manager=safety)
    server = WebSocketServer(engine=engine, port=8765)

    print("BRIDGE_READY", flush=True)  # Electron이 이 문자열로 준비 완료 감지
    await server.start()

if __name__ == '__main__':
    asyncio.run(main())
```

### `python/protocol/ws_server.py` — WebSocket 서버

```python
"""
asyncio + websockets 기반 WebSocket 서버.
포트 8765에서 리슨, JSON 메시지 송수신.
"""
import json
import asyncio
import websockets

class WebSocketServer:
    def __init__(self, engine, port=8765):
        self.engine = engine
        self.port = port
        self.clients = set()

    async def start(self):
        async with websockets.serve(self.handler, 'localhost', self.port):
            await asyncio.Future()  # run forever

    async def handler(self, websocket):
        self.clients.add(websocket)
        try:
            async for raw in websocket:
                msg = json.loads(raw)
                await self.dispatch(msg, websocket)
        finally:
            self.clients.discard(websocket)

    async def dispatch(self, msg: dict, websocket):
        msg_type = msg.get('type')
        payload = msg.get('payload', {})

        if msg_type == 'execute_flow':
            await self.engine.execute(payload['cards'])
        elif msg_type == 'pause':
            await self.engine.pause()
        elif msg_type == 'resume':
            await self.engine.resume()
        elif msg_type == 'e_stop':
            await self.engine.emergency_stop()
        elif msg_type == 'connect_hw':
            await self.engine.connect_hardware(payload['target'])

    async def broadcast(self, msg_type: str, payload: dict):
        """모든 클라이언트에 메시지 전송"""
        data = json.dumps({'type': msg_type, 'payload': payload})
        for ws in self.clients:
            await ws.send(data)
```

### `python/protocol/messages.py` — 메시지 유틸

```python
"""메시지 생성 헬퍼"""

def state_update(state: str, step: int) -> dict:
    return {'type': 'state_update', 'payload': {'state': state, 'step': step}}

def hw_status(robot: str, esp32: str, camera: str) -> dict:
    return {'type': 'hw_status', 'payload': {'robot': robot, 'esp32': esp32, 'camera': camera}}

def detection(color: str, confidence: float) -> dict:
    return {'type': 'detection', 'payload': {'color': color, 'confidence': confidence}}

def error(message: str, code: str) -> dict:
    return {'type': 'error', 'payload': {'message': message, 'code': code}}

def card_complete(card_uid: str, result: dict = None) -> dict:
    return {'type': 'card_complete', 'payload': {'cardUid': card_uid, 'result': result or {}}}
```

---

## 9. Flow Engine 상세

### `python/flow_engine/state_machine.py`

```python
from enum import Enum

class EngineState(Enum):
    IDLE = 'IDLE'
    RUNNING = 'RUNNING'
    PAUSED = 'PAUSED'
    WAITING = 'WAITING'   # 센서 대기
    ERROR = 'ERROR'
    E_STOP = 'E_STOP'

# 허용되는 상태 전이
TRANSITIONS = {
    EngineState.IDLE:    [EngineState.RUNNING],
    EngineState.RUNNING: [EngineState.RUNNING, EngineState.PAUSED, EngineState.WAITING, EngineState.ERROR, EngineState.IDLE, EngineState.E_STOP],
    EngineState.PAUSED:  [EngineState.RUNNING, EngineState.IDLE, EngineState.E_STOP],
    EngineState.WAITING: [EngineState.RUNNING, EngineState.ERROR, EngineState.E_STOP],
    EngineState.ERROR:   [EngineState.IDLE, EngineState.E_STOP],
    EngineState.E_STOP:  [EngineState.IDLE],
}
```

### `python/flow_engine/engine.py`

```python
"""
Flow Engine 코어.
FlowCard[] 배열을 받아 순차 실행한다.
"""

class FlowEngine:
    def __init__(self, safety_manager, broadcast_fn=None):
        self.state = EngineState.IDLE
        self.safety = safety_manager
        self.broadcast = broadcast_fn  # ws_server.broadcast
        self.current_step = -1
        self.cards = []
        self._pause_event = asyncio.Event()
        self._pause_event.set()  # 초기: 일시정지 아님

    async def execute(self, cards: list[dict]):
        """플로우 실행 시작"""
        self.cards = cards
        self.state = EngineState.RUNNING
        self.current_step = 0

        for i, card in enumerate(cards):
            # 일시정지 체크
            await self._pause_event.wait()

            # E-STOP 체크
            if self.state == EngineState.E_STOP:
                break

            self.current_step = i
            await self.broadcast('state_update', {'state': 'RUNNING', 'step': i})

            # 카드 실행
            result = await CardExecutor.execute(card, self)
            await self.broadcast('card_complete', {'cardUid': card['uid'], 'result': result})

        # 완료
        self.state = EngineState.IDLE
        self.current_step = -1
        await self.broadcast('state_update', {'state': 'IDLE', 'step': -1})

    async def pause(self):
        self.state = EngineState.PAUSED
        self._pause_event.clear()

    async def resume(self):
        self.state = EngineState.RUNNING
        self._pause_event.set()

    async def emergency_stop(self):
        """E-STOP: 모든 장비 즉시 정지"""
        self.state = EngineState.E_STOP
        self._pause_event.set()  # 일시정지 해제 (루프 탈출)
        await self.safety.emergency_stop_all()
        await self.broadcast('state_update', {'state': 'E_STOP', 'step': self.current_step})
```

### `python/flow_engine/card_executor.py`

```python
"""
카드 타입별 실행 로직 디스패치.
각 카드의 command 필드에 따라 해당 하드웨어 컨트롤러를 호출한다.
"""

class CardExecutor:
    @staticmethod
    async def execute(card: dict, engine) -> dict:
        command = card.get('command', '')
        params = card.get('params', {})

        match command:
            # --- 기본 ---
            case 'FLOW_START':
                return {}
            case 'FLOW_STOP':
                return {}
            case 'DELAY':
                seconds = params.get('seconds', 1)
                await asyncio.sleep(seconds)
                return {'delayed': seconds}
            case 'LOOP':
                count = params.get('count', 1)
                # loop 내부 카드 반복 실행 (별도 구현)
                return {'loopCount': count}

            # --- 컨베이어 ---
            case 'CONV_START':
                await engine.esp32.send_command('CONV_START')
                return {}
            case 'CONV_STOP':
                await engine.esp32.send_command('CONV_STOP')
                return {}
            case 'CONV_SPEED':
                level = params.get('level', 3)
                await engine.esp32.send_command('CONV_SPEED', {'level': level})
                return {'level': level}

            # --- 센서 ---
            case 'WAIT_SENSOR':
                timeout = params.get('timeout', 30)
                engine.state = EngineState.WAITING
                result = await engine.esp32.wait_for_sensor(timeout)
                engine.state = EngineState.RUNNING
                return result
            case 'IF_SENSOR':
                sensor = params.get('sensor', 'ir1')
                value = await engine.esp32.read_sensor(sensor)
                return {'sensor': sensor, 'value': value}

            # --- 비전 ---
            case 'CAPTURE':
                image = await engine.camera.capture()
                return {'captured': True}
            case 'COLOR_DETECT':
                color, confidence = await engine.camera.detect_color()
                await engine.broadcast('detection', {'color': color, 'confidence': confidence})
                return {'color': color, 'confidence': confidence}

            # --- 로봇 ---
            case 'ROBOT_HOME':
                await engine.robot.go_home()
                return {}
            case 'ROBOT_MOVE':
                position = params.get('position', 'A')
                await engine.robot.move_to(position)
                return {'position': position}
            case 'ROBOT_PICK':
                await engine.robot.pick()
                return {}
            case 'ROBOT_PLACE':
                await engine.robot.place()
                return {}

            case _:
                raise ValueError(f"Unknown command: {command}")
```

---

## 10. 하드웨어 컨트롤러

### `python/hardware/robot_controller.py`

```python
"""
UFactory Lite 6 로봇팔 컨트롤러.
xArm-Python-SDK를 래핑하여 교육 모드 제한을 적용한다.
"""
from xarm.wrapper import XArmAPI

# 사전 정의 위치
POSITIONS = {
    'A': {'x': 200, 'y': 0,    'z': 150, 'roll': 180, 'pitch': 0, 'yaw': 0},
    'B': {'x': 200, 'y': 150,  'z': 150, 'roll': 180, 'pitch': 0, 'yaw': 0},
    'C': {'x': 200, 'y': -150, 'z': 150, 'roll': 180, 'pitch': 0, 'yaw': 0},
}

# 교육 모드 제한
MAX_SPEED = 100        # mm/s
MAX_ACCELERATION = 500 # mm/s²
COLLISION_SENSITIVITY = 3  # 1~5 (3: 민감)

class RobotController:
    def __init__(self):
        self.arm: XArmAPI | None = None
        self.connected = False

    async def connect(self, ip: str):
        self.arm = XArmAPI(ip)
        self.arm.motion_enable(enable=True)
        self.arm.set_mode(0)
        self.arm.set_state(state=0)
        # 교육 모드 제한 적용
        self.arm.set_tcp_maxacc(MAX_ACCELERATION)
        self.arm.set_collision_sensitivity(COLLISION_SENSITIVITY)
        self.connected = True

    async def move_to(self, position_name: str):
        """사전 정의 위치로만 이동 허용"""
        pos = POSITIONS.get(position_name)
        if not pos:
            raise ValueError(f"허용되지 않는 위치: {position_name}")
        self.arm.set_position(
            x=pos['x'], y=pos['y'], z=pos['z'],
            roll=pos['roll'], pitch=pos['pitch'], yaw=pos['yaw'],
            speed=MAX_SPEED, wait=True
        )

    async def go_home(self):
        self.arm.move_gohome(speed=MAX_SPEED, wait=True)

    # 그리퍼: gripper_type에 따라 분기 ("gripper" | "vacuum")
    async def pick(self):
        if self.gripper_type == "vacuum":
            self.arm.set_vacuum_gripper(True, wait=False)
        else:
            self.arm.close_lite6_gripper()  # Lite6 내장 그리퍼

    async def place(self):
        if self.gripper_type == "vacuum":
            self.arm.set_vacuum_gripper(False, wait=False)
        else:
            self.arm.open_lite6_gripper()

    async def gripper_stop(self):
        if self.gripper_type == "vacuum":
            self.arm.set_vacuum_gripper(False, wait=False)
        else:
            self.arm.stop_lite6_gripper()

    async def get_current_position(self) -> dict:
        """현재 TCP 위치 읽기 (위치 교시용)"""
        code, pos = self.arm.get_position()
        return {"x": pos[0], "y": pos[1], "z": pos[2],
                "roll": pos[3], "pitch": pos[4], "yaw": pos[5]}

    async def emergency_stop(self):
        if self.arm:
            self.arm.emergency_stop()

    async def disconnect(self):
        if self.arm:
            self.arm.disconnect()
            self.connected = False
```

### 10a. 로봇 Live Control (Sprint 7에서 구현 예정)

```python
# xArm 제어 모드
# Mode 0: Position (표준 위치 이동) — 현재 카드 실행 시 사용
# Mode 2: Teaching (자유 이동) — 손으로 잡고 교시
# Mode 4: Joint Velocity — 관절별 조그
# Mode 5: Cartesian Velocity — XYZ/RPY 조그 (추천)

# 카테시안 조그 예시 (Mode 5)
async def jog_cartesian(self, direction: list[float], speed: float = 30):
    """XYZ/RPY 방향별 조그 이동. direction=[vx,vy,vz,vr,vp,vy]"""
    self.arm.set_mode(5)
    self.arm.set_state(0)
    self.arm.vc_set_cartesian_velocity(
        speeds=[d * speed for d in direction],
        is_tool_coord=0,
        duration=0.1  # 버튼 놓으면 자동 정지
    )

# Teaching 모드 토글
async def set_teaching_mode(self, enable: bool):
    if enable:
        self.arm.set_mode(2)  # 자유 이동
    else:
        self.arm.set_mode(0)  # Position 모드 복귀
    self.arm.set_state(0)

# 카드 실행 시 Mode 0으로 복귀 필수
async def ensure_position_mode(self):
    self.arm.set_mode(0)
    self.arm.set_state(0)
```

### 10b. GPIO 제어 (Sprint 7에서 구현 예정)

```python
# === Controller GPIO (CGPIO) ===
# CI0~CI7: 디지털 입력 (24V), 약한 풀업 (미연결 시 HIGH)
# CO0~CO7: 디지털 출력 (24V, NPN 오픈 드레인 → 외부 풀업 필요)
# AI0~AI1: 아날로그 입력 (0~5V)
# AO0~AO1: 아날로그 출력 (0~5V)

# 디지털 입력 읽기
code, value = arm.get_cgpio_digital(ionum)  # ionum: 0~7

# 디지털 출력 쓰기
code = arm.set_cgpio_digital(ionum, value)  # value: 0 or 1

# 아날로그 입력 읽기
code, value = arm.get_cgpio_analog(ionum)  # ionum: 0 or 1

# === Tool GPIO (TGPIO) ===
# IO0, IO1: 디지털 I/O (5V)
code, value = arm.get_tgpio_digital(ionum)  # ionum: 0 or 1
code = arm.set_tgpio_digital(ionum, value)

# === 출력 기능 설정 (특수 기능) ===
arm.set_cgpio_digital_output_function(ionum, fun)
# fun: 0=일반, 1=E-STOP, 2=동작중, 11=오류, 13=충돌, 14=교시중
```

### `python/hardware/esp32_controller.py`

```python
"""
ESP32 컨베이어 컨트롤러.
pyserial로 USB Serial 통신. JSON + 개행 프로토콜.
타임아웃 3초, 재시도 2회.
"""
import json
import serial
import asyncio

BAUD_RATE = 115200
TIMEOUT = 3     # 초
MAX_RETRIES = 2

class ESP32Controller:
    def __init__(self):
        self.ser: serial.Serial | None = None
        self.connected = False

    async def connect(self, port: str):
        self.ser = serial.Serial(port, BAUD_RATE, timeout=TIMEOUT)
        self.connected = True

    async def send_command(self, cmd: str, params: dict = None) -> dict:
        """명령 전송 + 응답 수신 (재시도 포함)"""
        message = json.dumps({'cmd': cmd, 'params': params or {}}) + '\n'

        for attempt in range(MAX_RETRIES + 1):
            self.ser.write(message.encode())
            response_line = self.ser.readline().decode().strip()

            if response_line:
                response = json.loads(response_line)
                if response.get('status') == 'OK':
                    return response
                else:
                    raise RuntimeError(f"ESP32 오류: {response}")

            if attempt < MAX_RETRIES:
                await asyncio.sleep(0.5)  # 재시도 전 대기

        raise TimeoutError(f"ESP32 응답 없음 (cmd: {cmd})")

    async def wait_for_sensor(self, timeout: int = 30) -> dict:
        """센서 이벤트 수신 대기"""
        self.ser.timeout = timeout
        line = self.ser.readline().decode().strip()
        self.ser.timeout = TIMEOUT  # 복원

        if line:
            event = json.loads(line)
            return event
        raise TimeoutError("센서 감지 타임아웃")

    async def emergency_stop(self):
        if self.ser and self.ser.is_open:
            await self.send_command('CONV_STOP')

    async def disconnect(self):
        if self.ser and self.ser.is_open:
            self.ser.close()
            self.connected = False
```

### `python/hardware/camera_controller.py`

```python
"""
ESP32-CAM 컨트롤러.
HTTP로 캡처 요청, OpenCV로 색상 분석.
MJPEG 스트림은 프론트엔드에서 직접 접근.
"""
import cv2
import numpy as np
import urllib.request

class CameraController:
    def __init__(self):
        self.ip: str = ''
        self.connected = False

    async def connect(self, ip: str):
        self.ip = ip
        # 연결 확인: 캡처 한 번 시도
        try:
            await self.capture()
            self.connected = True
        except Exception:
            raise ConnectionError(f"카메라 연결 실패: {ip}")

    async def capture(self) -> np.ndarray:
        """JPEG 캡처 → OpenCV 이미지"""
        url = f"http://{self.ip}/capture"
        resp = urllib.request.urlopen(url, timeout=5)
        img_array = np.frombuffer(resp.read(), dtype=np.uint8)
        image = cv2.imdecode(img_array, cv2.IMREAD_COLOR)
        return image

    async def detect_color(self) -> tuple[str, float]:
        """캡처 → HSV 변환 → 지배색 판별"""
        image = await self.capture()
        hsv = cv2.cvtColor(image, cv2.COLOR_BGR2HSV)

        # 색상 범위 정의 (HSV)
        color_ranges = {
            'red':    [((0, 100, 100), (10, 255, 255)),
                       ((160, 100, 100), (180, 255, 255))],
            'blue':   [((100, 100, 100), (130, 255, 255))],
            'green':  [((40, 100, 100), (80, 255, 255))],
            'yellow': [((20, 100, 100), (40, 255, 255))],
        }

        best_color = 'unknown'
        best_ratio = 0.0
        total_pixels = hsv.shape[0] * hsv.shape[1]

        for color, ranges in color_ranges.items():
            mask = None
            for lower, upper in ranges:
                m = cv2.inRange(hsv, np.array(lower), np.array(upper))
                mask = m if mask is None else cv2.bitwise_or(mask, m)
            ratio = cv2.countNonZero(mask) / total_pixels
            if ratio > best_ratio:
                best_ratio = ratio
                best_color = color

        return best_color, round(best_ratio, 3)

    def get_stream_url(self) -> str:
        return f"http://{self.ip}:81/stream"
```

### `python/hardware/safety_manager.py`

```python
"""
안전 관리자.
E-STOP 처리, 교육 모드 제한 적용.
"""

class SafetyManager:
    def __init__(self):
        self.robot = None
        self.esp32 = None
        self.camera = None

    def register_hardware(self, robot, esp32, camera):
        self.robot = robot
        self.esp32 = esp32
        self.camera = camera

    async def emergency_stop_all(self):
        """모든 장비 즉시 정지 — E-STOP 핵심 로직"""
        errors = []

        if self.robot and self.robot.connected:
            try:
                await self.robot.emergency_stop()
            except Exception as e:
                errors.append(f"로봇 정지 실패: {e}")

        if self.esp32 and self.esp32.connected:
            try:
                await self.esp32.emergency_stop()
            except Exception as e:
                errors.append(f"ESP32 정지 실패: {e}")

        if errors:
            raise RuntimeError(" | ".join(errors))
```

---

## 11. 시뮬레이션 모드

### `python/simulation/simulator.py`

```python
"""
하드웨어 Mock.
실제 장비 없이 플로우 로직을 테스트할 수 있게 한다.
각 명령을 시간 지연으로 대체하고 가상 데이터를 반환한다.
"""
import asyncio
import random

class SimulatedRobot:
    connected = True

    async def connect(self, ip): pass
    async def go_home(self): await asyncio.sleep(1.0)
    async def move_to(self, pos): await asyncio.sleep(1.5)
    async def pick(self): await asyncio.sleep(0.5)
    async def place(self): await asyncio.sleep(0.5)
    async def emergency_stop(self): pass
    async def disconnect(self): pass

class SimulatedESP32:
    connected = True

    async def connect(self, port): pass
    async def send_command(self, cmd, params=None):
        await asyncio.sleep(0.3)
        return {'status': 'OK'}
    async def wait_for_sensor(self, timeout=30):
        delay = random.uniform(1, 3)
        await asyncio.sleep(delay)
        return {'event': 'sensor', 'sensor': 'ir1', 'value': True}
    async def read_sensor(self, sensor):
        return random.choice([True, False])
    async def emergency_stop(self): pass
    async def disconnect(self): pass

class SimulatedCamera:
    connected = True

    async def connect(self, ip): pass
    async def capture(self):
        await asyncio.sleep(0.2)
        return None  # 시뮬레이션에서는 이미지 불필요
    async def detect_color(self):
        await asyncio.sleep(0.5)
        color = random.choice(['red', 'blue', 'green', 'yellow'])
        confidence = round(random.uniform(0.7, 0.99), 3)
        return color, confidence
    def get_stream_url(self):
        return ''
```

---

## 12. ESP32 펌웨어

### `firmware/conveyor_controller/conveyor_controller.ino`

```
핵심 구조:
- setup(): Serial 115200bps 초기화, GPIO 핀 설정
- loop(): Serial에서 JSON 명령 수신 → 파싱 → 실행 → 응답
- 센서 인터럽트: IR 센서 감지 시 이벤트 JSON 전송

핀 배치:
- GPIO 18: TB6600 STEP/PUL (스텝 펄스, 하드웨어 타이머 ISR)
- GPIO 19: TB6600 DIR (방향, setup에서 HIGH 고정)
- GPIO 21: TB6600 ENA (활성화, LOW=활성)
- GPIO 22: E18-D80NK IR 센서 (INPUT_PULLUP, 감지=LOW)
- GPIO 16: UART2 RX ← ESP32-CAM TX
- GPIO 17: UART2 TX → ESP32-CAM RX
- TB6600 PUL-/DIR-/ENA- → ESP32 GND (공유 필수)

모터: 42BYG45040-24 (NEMA17, 1.8°, 1.68A)
TB6600 DIP: SW1=ON SW2=ON (16마이크로스텝, 3200 steps/rev)
            SW4=ON SW5=OFF SW6=OFF (2.0A)
기본 속도: Lv1 = 1600µs 간격 = 625 steps/s = 11.7 RPM

모터 제어: ESP32 하드웨어 타이머 ISR (loop 독립 정밀 펄스)
  - AccelStepper 미사용 (loop 오버헤드로 타이밍 불규칙 발생)
  - DIR 핀 setup에서 1회 HIGH 고정, 이후 코드에서 변경 없음

명령 처리:
- CONV_START → ENA LOW, 타이머 시작 (userStarted 플래그)
- CONV_STOP  → 타이머 정지, ENA HIGH
- CONV_SPEED → 타이머 간격 변경 (level 1~5)
- E_STOP     → 타이머 정지, ENA HIGH, 이벤트 전송
- READ_SENSOR → 센서 즉시 읽기
- CAM_CAPTURE/CAM_STATUS → Serial2로 ESP32-CAM에 릴레이

센서 이벤트 (디바운스 200ms):
- 상태 변화 시에만 → {"event":"sensor","sensor":"ir1","value":true}\n

CAM 이벤트 릴레이 (Serial2 → Serial0):
- cam_ready, cam_status, cam_capture_done, cam_disconnected (60초 타임아웃)
```

### 핀 배선 요약

| ESP32 GPIO | 연결 대상 | 방향 | 설명 |
|------------|-----------|------|------|
| GPIO 18 | TB6600 PUL+ | OUTPUT | 스텝 펄스 (하드웨어 타이머 ISR) |
| GPIO 19 | TB6600 DIR+ | OUTPUT | 회전 방향 (HIGH 고정) |
| GPIO 21 | TB6600 ENA+ | OUTPUT | 모터 활성화 (LOW=활성) |
| GPIO 22 | E18-D80NK OUT | INPUT_PULLUP | IR 센서 (감지=LOW, 디바운스 200ms) |
| GPIO 16 | ESP32-CAM TX | UART2 RX | 카메라 이벤트 수신 |
| GPIO 17 | ESP32-CAM RX | UART2 TX | 카메라 명령 전송 |
| 5V | E18-D80NK VCC (갈색) | POWER | 센서 전원 |
| GND | E18-D80NK GND (파랑) | POWER | 센서 접지 |
| GND | TB6600 PUL-/DIR-/ENA- | POWER | ★ 드라이버 GND 공유 필수 |
| TX/RX | USB-Serial (PC) | UART0 | Python 통신 (JSON + 상태 출력) |

---

## 12a. Sprint 8 기술 구현 사양

### 12a-1. 텍스트 표시 카드 (TEXT_DISPLAY)

```python
# python/flow_engine/card_executor.py — TEXT_DISPLAY 핸들러
case 'TEXT_DISPLAY':
    text = params.get('text', '')
    size = params.get('size', 'medium')  # 'small' | 'medium' | 'large'
    await engine.broadcast('text_display', {'text': text, 'size': size})
    return {'text': text}
```

```typescript
// App.tsx — text_display 핸들러
wsClient.on('text_display', (payload) => {
  addMessageLog({ text: payload.text as string, size: payload.size as string })
})
```

- 메시지 보드 컴포넌트: 우측 패널 하단, 최대 20줄 유지 (오래된 항목 자동 삭제)
- 연속 실행 모드에서는 새 스캔 시작 시 로그 초기화 여부 선택 가능

### 12a-2. TTS 카드 (SPEAK) — Web Speech API

```typescript
// App.tsx — speak 핸들러
wsClient.on('speak', (payload) => {
  const utter = new SpeechSynthesisUtterance(payload.text as string)
  utter.lang = 'ko-KR'
  utter.rate = { slow: 0.7, normal: 1.0, fast: 1.4 }[payload.speed as string] ?? 1.0
  window.speechSynthesis.speak(utter)
})
```

- Python 불필요: 카드 실행 → WS 브로드캐스트 → 프론트엔드 재생
- 브라우저 내장 API, 인터넷 연결 불필요
- `SpeechSynthesisUtterance` — ko-KR 음성 우선, 없으면 기본 음성 사용

### 12a-3. 사운드 카드 (PLAY_SOUND / BGM) — HTML5 Audio API

```typescript
// src/renderer/src/utils/audio-manager.ts
const SOUNDS: Record<string, HTMLAudioElement> = {
  beep:    new Audio('/sounds/beep.mp3'),
  success: new Audio('/sounds/success.mp3'),
  error:   new Audio('/sounds/error.mp3'),
  start:   new Audio('/sounds/start.mp3'),
  end:     new Audio('/sounds/end.mp3'),
}
const BGM: Record<string, HTMLAudioElement> = {
  work:  new Audio('/sounds/bgm_work.mp3'),
  chill: new Audio('/sounds/bgm_chill.mp3'),
}
// bgm은 loop=true, 동시에 하나만 재생
```

- 사운드 파일: `src/renderer/public/sounds/` → Vite 빌드 시 번들
- `PLAY_SOUND` → 효과음 1회 재생
- `BGM_START` → 배경음 loop 재생, `BGM_STOP` → 정지
- E-STOP 시 `bgmAudio.pause()` 자동 호출

### 12a-4. 그래픽 모니터링 뷰 — SVG + CSS 애니메이션

```
레이아웃:
중앙 패널
├── 팩토조립소 (FlowCanvas)     flex-1 (상단, 스크롤 가능)
└── 그래픽 모니터링 뷰          h-[180px] (하단, 고정 높이)
    └── SVG 장면 (1:3 비율)
        ├── 로봇팔 SVG (현재 위치 반영)
        ├── 컨베이어 벨트 SVG (CSS rotate 애니메이션)
        ├── IR 센서 원 (hardwareStore.sensorIr1/Ir2 → fill 색상)
        └── 카메라 아이콘 (연결 상태)
```

```typescript
// 컨베이어 회전 애니메이션 예시
// CSS: @keyframes belt-move { from { stroke-dashoffset: 0 } to { stroke-dashoffset: -20 } }
// isRunning ? 'animate-belt' : '' (Tailwind 커스텀 애니메이션)
```

- Zustand 상태 연동: `engineState`, `sensorIr1/Ir2`, `gpioState`, `livePosition`
- 순수 SVG + Tailwind CSS 애니메이션 — 외부 라이브러리 불필요

---

## 13. 빌드 및 패키징

### 개발 모드

```bash
# 터미널 1: Vite dev server (React)
npm run dev

# 터미널 2: Electron (메인 프로세스)
npm run electron:dev

# 터미널 3: Python 백엔드 (선택 — Electron이 자동 시작할 수도 있음)
cd python && python main.py
```

### `vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  base: './',  // Electron 파일 로딩용 상대 경로
  build: {
    outDir: 'dist',
  },
});
```

### 프로덕션 빌드

```bash
# 1. React 빌드
npm run build

# 2. Electron 패키징
npm run electron:build
# electron-builder가 dist/ 폴더를 번들링
# 출력: release/ 디렉토리에 .dmg (macOS) 또는 .exe (Windows)
```

### `package.json` 스크립트 (예시)

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "electron:dev": "electron .",
    "electron:build": "electron-builder",
    "python:dev": "cd python && python main.py"
  },
  "build": {
    "appId": "com.bridge.automation",
    "productName": "자동화 브릿지",
    "files": ["dist/**/*", "electron/**/*", "python/**/*"],
    "extraResources": [
      { "from": "python", "to": "python", "filter": ["**/*", "!venv/**"] }
    ],
    "mac": { "target": "dmg" },
    "win": { "target": "nsis" }
  }
}
```

### Python 번들링 전략

프로덕션 배포 시 Python 런타임 포함 방법:
1. **pyinstaller**: `python/main.py`를 단일 바이너리로 패키징 → Electron `extraResources`에 포함
2. **시스템 Python 의존**: 사용자 PC에 Python 3.11+ 설치 요구 (교육 환경에서는 교사가 사전 설치)

Sprint 1에서는 방법 2(시스템 Python)로 개발하고, 추후 배포 시 방법 1(pyinstaller)로 전환한다.

---

## 14. 테스트 전략

### 프론트엔드 테스트

| 범위 | 도구 | 대상 |
|------|------|------|
| 단위 테스트 | Vitest | Zustand 스토어 로직, 유틸 함수 |
| 컴포넌트 테스트 | Vitest + Testing Library | CardItem, CardPalette 렌더링 |
| E2E 테스트 | Playwright | 카드 드래그&드롭, 플로우 실행 시나리오 |

### 백엔드 테스트

| 범위 | 도구 | 대상 |
|------|------|------|
| 단위 테스트 | pytest + pytest-asyncio | FlowEngine, CardExecutor, StateMachine |
| 통합 테스트 | pytest | WebSocket 서버 + 시뮬레이션 모드 |
| 하드웨어 테스트 | 수동 | 실제 장비 연결 후 동작 확인 |

### 테스트 시나리오

```
[TC-01] 기본 플로우 실행
  1. 시뮬레이션 모드 활성화
  2. 카드 배치: 시작 → 벨트 출발 → 기다리기(2초) → 벨트 정지
  3. 실행 → 확인 대화상자 → "실행"
  4. 검증: 각 카드 순차 실행, state_update 메시지, 완료 후 IDLE

[TC-02] E-STOP 동작
  1. 플로우 실행 중 E-STOP 클릭
  2. 검증: 500ms 이내 상태 E_STOP 전이, 모든 장비 정지

[TC-03] 일시정지/재개
  1. 플로우 실행 중 일시정지
  2. 검증: 현재 카드 완료 후 PAUSED 상태
  3. 재개 → 다음 카드부터 RUNNING

[TC-04] 저장/불러오기
  1. 카드 3장 배치 + 파라미터 설정
  2. 저장 → .bridge.json 생성 확인
  3. 새 세션에서 불러오기 → 동일 플로우 복원 확인

[TC-05] 하드웨어 연결/해제
  1. 로봇 IP 입력 → 연결 → 상태 초록
  2. 잘못된 IP → 연결 실패 → 상태 빨강 + 오류 메시지
  3. 연결 해제 → 상태 회색
```

---

## 15. 오류 처리 정책

### 오류 코드 체계

| 코드 | 설명 | UI 메시지 |
|------|------|-----------|
| `HW_ROBOT_DISCONNECT` | 로봇팔 연결 끊김 | "로봇팔 연결이 끊어졌어요. 연결 상태를 확인해주세요." |
| `HW_ROBOT_COLLISION` | 로봇팔 충돌 감지 | "로봇팔이 무언가에 부딪혔어요. 주변을 확인해주세요." |
| `HW_SERIAL_TIMEOUT` | ESP32 응답 없음 | "컨베이어가 응답하지 않아요. USB 케이블을 확인해주세요." |
| `HW_SERIAL_ERROR` | ESP32 통신 오류 | "컨베이어 통신에 문제가 생겼어요." |
| `HW_CAMERA_FAIL` | 카메라 연결 실패 | "카메라에 연결할 수 없어요. WiFi 연결을 확인해주세요." |
| `ENGINE_INVALID_CARD` | 알 수 없는 카드 명령 | "알 수 없는 카드예요. 플로우를 확인해주세요." |
| `ENGINE_INVALID_PARAM` | 파라미터 범위 초과 | "잘못된 설정값이에요. 다시 확인해주세요." |
| `WS_DISCONNECT` | WebSocket 연결 끊김 | "프로그램 연결이 끊어졌어요. 잠시 후 다시 시도합니다." |

### 오류 처리 원칙

1. **하드웨어 오류 → 안전 정지**: 통신 실패 시 모든 장비 정지 (fail-safe)
2. **사용자 친화적 메시지**: 기술 용어 대신 원인 + 해결 방법을 한글로 안내
3. **로깅**: Python 백엔드에서 `logging` 모듈로 상세 로그 기록 (디버깅용)
4. **복구 가능**: 오류 후 IDLE 상태로 복귀하여 재실행 가능

---

## 13. Phase 2 — IEC 61131-3 ST 블록 코딩 기술 사양 ✅ Sprint 11~13 완료

### 13-1. 설계 방향

Phase 2는 Google Blockly 대신 **IEC 61131-3 ST(구조화 텍스트)** 기반의 PLC 교육용 블록 코딩 에디터를 구축한다.

**학습 경로:**
```
Phase 1 카드 모드 → Phase 2a ST 블록 모드 → Phase 2b ST 텍스트 모드 → 실제 PLC (Codesys/TIA Portal)
```

**핵심 원칙:**
- ST 문법은 IEC 국제 표준을 준수하되, 교육용으로 한글 힌트와 시각적 블록을 제공
- 기존 Flow Engine(`FlowCard[]` 입력)을 그대로 재사용 — ST → FlowCard[] 변환기만 추가
- 하드웨어 독립적: ST 코드는 하드웨어 드라이버 레이어와 분리

### 13-2. 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    Phase 2 프론트엔드                   │
│  ┌─────────────────┐    ┌──────────────────────────┐  │
│  │ ST 블록 에디터    │    │ ST 코드 뷰어/편집기       │  │
│  │ (dnd-kit 확장)   │ ↔  │ (Monaco Editor)          │  │
│  └────────┬────────┘    └──────────┬───────────────┘  │
│           │                        │                   │
│           ▼                        ▼                   │
│  ┌─────────────────────────────────────────────────┐  │
│  │        ST ↔ FlowCard[] 변환기 (TypeScript)       │  │
│  │  st-to-flow.ts: ST AST → FlowCard[]             │  │
│  │  flow-to-st.ts: FlowCard[] → ST 코드 문자열      │  │
│  └────────────────────────┬────────────────────────┘  │
└───────────────────────────┼────────────────────────────┘
                            │ WebSocket (기존)
                            ▼
              ┌─────────────────────────────┐
              │  Flow Engine (변경 없음)     │
              │  CardExecutor (변경 없음)    │
              │  HW Drivers (변경 없음)      │
              └─────────────────────────────┘
```

### 13-3. ST 함수 블록 라이브러리

Phase 1의 29종 카드 커맨드가 ST 함수 블록으로 1:1 매핑된다.

```pascal
(* ── 프로그램 구조 ── *)
PROGRAM 택배_분류
VAR
  감지색상 : STRING;
  반복횟수 : INT := 3;
  센서값   : BOOL;
END_VAR

(* ── 컨베이어 ── *)
Conveyor_Start();                    (* CONV_START *)
Conveyor_Stop();                     (* CONV_STOP *)
Conveyor_Speed(level := 3);          (* CONV_SPEED *)

(* ── 센서 / 비전 ── *)
Wait_Sensor(timeout := 10);          (* WAIT_SENSOR *)
Color_Detect();                      (* COLOR_DETECT *)
Capture();                           (* CAPTURE *)

(* ── 로봇 ── *)
Robot_Home();                        (* ROBOT_HOME *)
Robot_Move(position := 'A');         (* ROBOT_MOVE *)
Robot_Pick();                        (* ROBOT_PICK *)
Robot_Place();                       (* ROBOT_PLACE *)
Gripper_Stop();                      (* ROBOT_GRIPPER_STOP *)

(* ── GPIO ── *)
GPIO_Set(pin := 'CO0', value := TRUE);     (* GPIO_SET *)
GPIO_Wait(pin := 'CI0', value := TRUE, timeout := 5);  (* WAIT_GPIO *)
TGPIO_Set(pin := 0, value := TRUE);        (* TGPIO_SET *)

(* ── 제어 구조 — Phase 1 블록 카드와 1:1 대응 ── *)
IF 감지색상 = 'red' THEN             (* if_cond / end_if *)
  Robot_Move(position := 'A');
ELSE
  Robot_Move(position := 'C');
END_IF;

FOR i := 1 TO 반복횟수 DO            (* loop / end_loop *)
  Conveyor_Start();
  Wait_Sensor(timeout := 10);
  Conveyor_Stop();
END_FOR;

WHILE 센서값 DO                      (* while_cond / end_while *)
  Wait_Sensor(timeout := 5);
END_WHILE;

(* ── 추가 ST 표준 요소 (Phase 1에 없는 것) ── *)
Delay(seconds := 2.5);               (* DELAY — 실수 지원 *)
timer1(IN := TRUE, PT := T#5s);      (* TON 타이머 *)
counter1(CU := sensor_pulse);         (* CTU 카운터 *)

END_PROGRAM
```

### 13-4. ST ↔ FlowCard[] 변환기 구현 (Sprint 11~12)

**FlowCard[] → ST (`st-code-generator.ts`, Sprint 11 구현):**

```typescript
// FlowCard[] → ST 코드 문자열 + lineMap 생성
// lineMap: { [cardUid]: lineNumber } — 양방향 하이라이트용
// 1. FlowCard.definitionId → CARD_TO_ST 매핑 테이블로 ST 라인 생성
// 2. depth → 들여쓰기 (depth * 2 공백)
// 3. block_open/block_close structureType → IF/FOR/WHILE 구조 자동 생성
// 4. PROGRAM Main + VAR 섹션 + END_PROGRAM 래핑

export function generateSTCode(cards: FlowCard[]): { code: string; lineMap: Record<string, number> }
```

**ST → FlowCard[] (`st-parser.ts`, Sprint 12 구현):**

```typescript
// 섹션 상태 머신: preamble | function_block | var | body | done
// BlockStack 클래스: open/close/middle 블록 blockId 페어링
// tryMultiline(): 멀티라인 카드 시그니처 5종 (파라미터가 여러 줄에 걸치는 경우)
// matchSingle(): 47종 카드 패턴 정규식 (우선순위 순으로 매칭)
// REVERSE_COND 맵: 'color_result = RED' → if_cond 조건 역방향 변환

export function parseSTCode(stCode: string): {
  cards: FlowCard[];
  warnings: Array<{ line: number; text: string }>;
}
```

### 13-5. Monaco Editor 설정

```typescript
// ST 언어 정의 (Monaco custom language)
const ST_LANGUAGE_DEF = {
  keywords: [
    'PROGRAM', 'END_PROGRAM', 'VAR', 'END_VAR',
    'IF', 'THEN', 'ELSE', 'ELSIF', 'END_IF',
    'FOR', 'TO', 'BY', 'DO', 'END_FOR',
    'WHILE', 'END_WHILE', 'REPEAT', 'UNTIL', 'END_REPEAT',
    'CASE', 'OF', 'END_CASE',
    'FUNCTION_BLOCK', 'END_FUNCTION_BLOCK',
    'TRUE', 'FALSE', 'AND', 'OR', 'NOT', 'XOR',
  ],
  typeKeywords: [
    'BOOL', 'INT', 'REAL', 'STRING', 'TIME', 'DINT', 'LINT',
  ],
  // 자동완성: ST 함수 블록 라이브러리 시그니처 제공
  completionItems: [
    { label: 'Conveyor_Start', detail: '컨베이어 출발' },
    { label: 'Robot_Move', detail: '로봇 이동', insertText: "Robot_Move(position := '${1:A}');" },
    // ...
  ],
}
```

### 13-6. 하드웨어 확장성

ST 코드 레이어와 하드웨어 드라이버 레이어가 완전히 분리되어 있으므로:

| 확장 시나리오 | ST 코드 변경 | 드라이버 변경 |
|--------------|-------------|-------------|
| 로봇 교체 (Lite 6 → Dobot MG400) | 없음 | `robot_controller.py` 교체 |
| 컨베이어 교체 (ESP32 → 산업용 PLC I/O) | 없음 | `esp32_controller.py` 교체 |
| 카메라 교체 (ESP32-CAM → USB 카메라) | 없음 | `camera_controller.py` 교체 |
| 산업용 PLC 키트 연결 (S7-1200) | ST 코드 복사-이식 가능 | 불필요 (PLC IDE에서 직접 실행) |
| 아두이노/라즈베리파이 키트 | 함수 블록 추가 | GPIO 드라이버 추가 |

### 13-7. Phase 2 실제 구현 파일 목록 (Sprint 11~13 완료)

```
src/renderer/src/
├── components/
│   └── st-editor/
│       ├── STFBPalette.tsx          # ST 라이브러리 팔레트 (6그룹) ✅
│       ├── STBlockCanvas.tsx        # 블록 캔버스 (ST 키워드 뱃지, 들여쓰기) ✅
│       └── STEditorPanel.tsx        # ST 에디터 전체 패널 (좌/우 분할) ✅
├── utils/
│   ├── st-code-generator.ts        # FlowCard[] → ST 코드 + lineMap ✅
│   ├── st-parser.ts                # ST → FlowCard[] 역방향 파서 (47종) ✅
│   ├── monaco-iec-st.ts            # Monaco IEC ST 언어 정의 + 한글 호버 힌트 ✅
│   └── st-exporter.ts              # PLC 코드 내보내기 (iec_st/codesys/tia_scl) ✅
src/main/
│   └── index.ts                    # dialog:exportSTFile IPC 핸들러 추가 ✅
src/preload/
│   ├── index.ts                    # exportSTFile API 노출 ✅
│   └── index.d.ts                  # BridgeAPI.exportSTFile 타입 정의 ✅
src/renderer/public/missions/
│   ├── st_example1_belt.st         # ST 예제: 벨트 속도 제어 ✅
│   ├── st_example2_sensor.st       # ST 예제: 센서 + IF/ELSE ✅
│   ├── st_example3_counter.st      # ST 예제: FOR 루프 ✅
│   └── st_example4_sorting.st      # ST 예제: FUNCTION_BLOCK 색상 분류 ✅
```

### 13-8. PLC 내보내기 기술 사양 (Sprint 13 구현)

```typescript
// st-exporter.ts — 3종 포맷 내보내기
type PLCExportFormat = 'iec_st' | 'codesys' | 'tia_scl'

export function exportForPLC(stCode: string, format: PLCExportFormat): string

// iec_st: 순수 IEC 61131-3 ST, PROGRAM/END_PROGRAM 구조 그대로 유지
// codesys: CODESYS V3.5 헤더 주석 추가, OpenPLC Runtime 호환
// tia_scl: PROGRAM → ORGANIZATION_BLOCK, 단따옴표 → 쌍따옴표, (* *) → // 주석

// Electron IPC
// main/index.ts: ipcMain.handle('dialog:exportSTFile', ...)
//   → dialog.showSaveDialog({ filters: [{extensions: [ext]}] })
//   → fs.writeFile(filePath, data)

// preload/index.ts:
//   exportSTFile: (data, defaultName, ext) => ipcRenderer.invoke('dialog:exportSTFile', ...)
```

---

> **문서 끝** — 이 기술 명세서는 CLAUDE.md와 PRD.md를 기반으로 구현 수준의 상세 사양을 정리한 문서입니다.
