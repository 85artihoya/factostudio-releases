# CLAUDE.md — Bridge Program Development Guide

## 프로젝트 개요
**자동화 브릿지 프로그램**: UFactory Lite 6 로봇팔 + 스텝모터 컨베이어 + ESP32-CAM 비전 + 센서를 통합 제어하는 교육용 자동화 플랫폼.

- **Phase 1** (우선): PLC 카드 기반 UI → 초등 고학년 대상
- **Phase 2** (확장): IEC 61131-3 ST(구조화 텍스트) 기반 블록 코딩 → 중학생 이상 대상

## 기술 스택
- **Desktop**: Electron 39 + electron-vite 5.0 (메인 프로세스에서 Python 자식 프로세스 관리)
- **Frontend**: React 19 + TypeScript 5.9 + Tailwind CSS 3.4
- **Drag & Drop**: @dnd-kit/core 6.3 + @dnd-kit/sortable 10.0
- **State**: Zustand 5
- **Backend (Flow Engine)**: Python 3.11+ (asyncio + websockets 13.x)
- **IPC**: WebSocket (ws://localhost:8765)
- **Robot SDK**: xArm-Python-SDK (TCP/IP, 선택적 — HAS_XARM 플래그)
- **Serial**: pyserial (ESP32 USB 통신, 115200bps, 선택적 — HAS_SERIAL 플래그)
- **Camera**: ESP32-CAM MJPEG stream (HTTP) + aiohttp + OpenCV (색상 분석)
- **Build**: Vite 7.2, electron-builder (macOS/Windows/Linux)
- **Block Coding (Phase 2)**: IEC 61131-3 ST 기반 커스텀 블록 에디터 + Monaco Editor (ST 코드 뷰어)

## 프로젝트 구조 (실제)
```
bridge-program/
├── package.json
├── electron.vite.config.ts      # electron-vite 빌드 설정
├── tailwind.config.js           # Tailwind CSS (카드 카테고리 커스텀 색상)
├── tsconfig.json                # 루트 TS (node/web 참조)
├── tsconfig.node.json           # Main/Preload TS 설정
├── tsconfig.web.json            # Renderer TS 설정
├── src/
│   ├── main/                    # Electron 메인 프로세스
│   │   ├── index.ts             # 윈도우 생성, IPC, Python spawn ✅
│   │   └── python-manager.ts    # Python 프로세스 spawn/kill ✅
│   ├── preload/                 # IPC 브릿지
│   │   ├── index.ts             # openFile/saveFile/exportSTFile API 노출 ✅
│   │   └── index.d.ts           # BridgeAPI 타입 정의 ✅
│   └── renderer/src/            # React 프론트엔드
│       ├── App.tsx              # DndContext + WebSocket 리스너 ✅
│       ├── main.tsx             # React DOM 엔트리 ✅
│       ├── assets/
│       │   └── main.css         # Tailwind 베이스 ✅
│       ├── types/
│       │   └── cards.ts         # CardDefinition, FlowCard, BridgeFile 타입 ✅
│       ├── stores/
│       │   ├── flowStore.ts     # 카드 플로우 상태 (Zustand) ✅
│       │   ├── hardwareStore.ts # 장비 연결 + 엔진 상태 + GPIO + LiveControl + 메시지 보드 ✅
│       │   ├── themeStore.ts    # 다크/라이트 테마 (localStorage 영속) ✅
│       │   └── settingsStore.ts   # 단계별 해금 + PLC 뱃지 모드 + editorMode ('card'|'st') (Zustand persist) ✅
│       ├── utils/
│       │   ├── websocket-client.ts  # 싱글턴 WS + 자동 재연결 ✅
│       │   ├── card-definitions.ts  # 47종 카드 정의 데이터 + CARD_ID_MIGRATION ✅
│       │   ├── audio-player.ts      # Web Audio API 효과음 + BGM 합성기 ✅
│       │   ├── st-code-generator.ts # FlowCard[] → ST 코드 + lineMap (cardUid → 라인번호) ✅
│       │   ├── st-parser.ts         # ST 코드 → FlowCard[] 역방향 파서 (47종 패턴) ✅
│       │   ├── monaco-iec-st.ts     # Monaco IEC ST 언어 정의 (Monarch 토크나이저, 34키워드 호버힌트) ✅
│       │   └── st-exporter.ts       # PLC 코드 내보내기 (iec_st/codesys/tia_scl 3종 포맷) ✅
│       └── components/
│           ├── layout/
│           │   ├── MainLayout.tsx    # 3패널 레이아웃 (다크 테마, ST 에디터 조건부 렌더링) ✅
│           │   ├── Header.tsx        # 실행/일시정지/E-STOP/저장/불러오기 + 해금 레벨 선택 + PLC 뱃지 토글 + [블록]/[ST 코드] 탭 ✅
│           │   ├── StatusBar.tsx     # 하단 연결 상태 표시 ✅
│           │   └── Toast.tsx        # 에러 토스트 알림 (5초 자동 숨김) ✅
│           ├── st-editor/
│           │   ├── STFBPalette.tsx   # ST 라이브러리 팔레트 (6그룹: 컨베이어/로봇/센서/비전/타이머/GPIO) ✅
│           │   ├── STBlockCanvas.tsx # ST 블록 캔버스 (카드 + ST 키워드 뱃지, depth 들여쓰기, 활성 하이라이트) ✅
│           │   └── STEditorPanel.tsx # ST 에디터 전체 패널 (좌: 팔레트+예제, 우: 블록캔버스+Monaco) ✅
│           ├── card-editor/
│           │   ├── CardPalette.tsx   # 왼쪽 카드 팔레트 (드래그, 해금 레벨, hwTag/plcTerm 뱃지) ✅
│           │   ├── FlowCanvas.tsx    # 중앙 플로우 에디터 (드롭+정렬+자동스크롤) ✅
│           │   ├── CardItem.tsx      # 개별 카드 (실시간 하이라이트, hwTag/plcTerm 뱃지) ✅
│           │   ├── CardParamEditor.tsx # 파라미터 입력 (number/select) ✅
│           │   └── CardIcon.tsx         # SVG 커스텀 아이콘 17종 (컨베이어/로봇/센서/카메라/GPIO/흐름) ✅
│           ├── camera/
│           │   ├── CameraViewer.tsx  # MJPEG 스트림 뷰어 + 에러/재시도 ✅
│           │   └── DetectionOverlay.tsx # 색상 감지 결과 오버레이 (3초 자동 숨김) ✅
│           ├── monitor/
│           │   ├── HardwareStatus.tsx     # 실행 상태 + 진행률 바 + 현재 카드 + GPIO ✅
│           │   ├── ConnectionPanel.tsx    # 로봇/ESP32/카메라 연결 관리 + 그리퍼 선택 ✅
│           │   ├── PositionEditor.tsx     # 로봇 위치 교시 (HOME/INITIAL/ZERO/A/B/C) ✅
│           │   ├── LiveControlPanel.tsx   # 조그 버튼(XYZ/RPY ±) + Teaching 토글 + 속도 슬라이더 ✅
│           │   ├── GpioPanel.tsx          # GPIO 모니터 (CI/CO/AI/TGPIO 토글) ✅
│           │   ├── MessageBoard.tsx       # 텍스트 표시 카드 로그 (최대 50개, 자동 스크롤) ✅
│           │   ├── BgmUpload.tsx          # MP3/WAV/OGG 업로드 + AudioBuffer 미리듣기 ✅
│           │   └── SystemIllustration.tsx # SVG 시스템 일러스트 (벨트/로봇/센서/카메라 실시간) ✅
│           └── safety/              # 미사용 (E-STOP은 Header에 통합)
│               └── EStopButton.tsx
├── python/                      # Python 백엔드
│   ├── main.py                  # 엔트리포인트 (BRIDGE_READY 시그널) ✅
│   ├── requirements.txt         # websockets, xArm-SDK(선택), pyserial(선택)
│   ├── venv/                    # Python 가상환경
│   ├── flow_engine/
│   │   ├── engine.py            # Flow Engine 코어 (순차실행/루프/일시정지) ✅
│   │   ├── card_executor.py     # 47종 카드 + 변수/카운터 런타임 + GPIO pin 파라미터 파싱 ✅
│   │   ├── condition_evaluator.py # 조건 평가기 (센서/색상/컨베이어/GPIO) ✅
│   │   ├── block_parser.py      # AST 블록 파서 (LoopNNode, SubDefineNode 포함) ✅
│   │   └── state_machine.py     # 6상태 전이 관리 ✅
│   ├── hardware/
│   │   ├── robot_controller.py  # Lite 6 xArm SDK 래핑 (async, 그리퍼 2종, 위치 교시, 조그, GPIO) ✅
│   │   ├── esp32_controller.py  # ESP32 시리얼 통신 (async) ✅
│   │   ├── camera_controller.py # ESP32-CAM HTTP 캡처 + OpenCV 색상분석 ✅
│   │   └── safety_manager.py    # 교육모드 + 위치검증 + E-STOP 통합 ✅
│   ├── protocol/
│   │   ├── ws_server.py         # WebSocket 서버 (전체 통합 허브) ✅
│   │   └── messages.py          # 메시지 헬퍼 함수 ✅
│   ├── simulation/
│   │   └── simulator.py         # 하드웨어 Mock (전 장비+GPIO 시뮬레이션) ✅
│   └── tests/                   # pytest 테스트
│       ├── conftest.py
│       ├── test_flow_engine.py  # StateMachine + FlowEngine 테스트 ✅
│       ├── test_simulator.py    # 시뮬레이터 전체 커버리지 ✅
│       ├── test_safety.py       # SafetyManager 위치검증 + 속도제어 ✅
│       ├── test_camera_controller.py # 카메라 캡처+색상분석 ✅
│       ├── test_gpio.py         # GPIO 시뮬레이터 + 메시지 (16 tests) ✅
│       └── test_gpio_cards.py   # GPIO 카드 실행 + 그리퍼 전환 (12 tests) ✅
├── firmware/
│   └── conveyor_controller/
│       └── conveyor_controller.ino  # ESP32 스텝모터+센서 펌웨어 ✅
└── resources/
```

## 핵심 설계 원칙

### 1. 카드 시스템 — "흐름길 카드"
모든 자동화 동작은 **카드(Card)**로 추상화한다. PLC 래더 다이어그램의 개념을 아이 친화적으로 변환.

**카드 구조 분류:**

| structureType | 형태(shape) | 역할 | 예시 |
|---------------|------------|------|------|
| `action` | rect(사각형) | 단일 동작 실행 | 벨트 출발, 로봇 이동 |
| `block_open` | diamond(다이아몬드)/loop(순환)/fork(포크) | 블록 시작 | 만약~이면, 반복하기, 동시에 시작 |
| `block_middle` | bar(구분선) | 블록 내 분기 | 아니면, 작업 나누기 |
| `block_close` | bar(축소 바) | 블록 종료 | 만약~끝, 반복 끝 |
| `marker` | pill(원형) | 흐름 표시 | 시작, 정지 |

**흐름길 시각화:** 좌측 세로 레일 + 카드 연결선 + 블록 들여쓰기(depth × 24px)

```typescript
// types/cards.ts
export type CardStructureType = 'action' | 'block_open' | 'block_middle' | 'block_close' | 'marker'
export type CardShape = 'rect' | 'diamond' | 'loop' | 'fork' | 'bar' | 'pill'

export type PaletteCategory = 'flow_control'|'input'|'output'|'condition'|'timer_counter'|'variable'|'subroutine'
export type HwTag = 'conveyor'|'sensor'|'robot'|'camera'|'gpio'

export interface CardDefinition {
  id: string;
  category: 'basic'|'conveyor'|'sensor'|'robot'|'vision'|'gpio'; // HW 카테고리 (하위 호환)
  paletteCategoryId: PaletteCategory;  // 팔레트 표시 카테고리
  icon: string;
  name: string;
  description: string;
  color: string;
  params?: CardParamDef[];
  command: string;
  structureType: CardStructureType;
  shape: CardShape;
  blockPair?: string;
  blockMiddle?: string;
  hwTags?: HwTag[];       // 하드웨어 태그 뱃지
  unlockLevel: 1|2|3|4;   // 해금 단계
  plcTerm?: string;        // PLC 뱃지 모드 표시 용어 (TON, CTU, %Q 등)
}

export interface UnlockProfile {
  level: 1|2|3|4;
  customUnlocked?: PaletteCategory[];
}

export interface FlowCard {
  uid: string;
  definitionId: string;
  params: Record<string, any>;
  depth: number;             // 블록 중첩 깊이 (0 = 최상위)
  blockId?: string;          // 같은 블록 open/middle/close 연결
}
```

### 2. WebSocket 프로토콜 (UI ↔ Python)
포트: `ws://localhost:8765`, JSON 메시지.

| 방향 | type | payload |
|------|------|---------|
| UI→PY | `ping` | `{}` |
| UI→PY | `execute_flow` | `{cards: FlowCard[], continuous: boolean}` |
| UI→PY | `pause` | `{}` |
| UI→PY | `resume` | `{}` |
| UI→PY | `reset` | `{}` |
| UI→PY | `e_stop` | `{}` |
| UI→PY | `connect_hw` | `{target, ip?, port?, gripper_type?: 'gripper'\|'vacuum'}` |
| UI→PY | `disconnect_hw` | `{target: 'robot'\|'esp32'\|'camera'}` |
| UI→PY | `list_ports` | `{}` |
| UI→PY | `update_position` | `{name: string, position: RobotPosition}` |
| UI→PY | `get_robot_position` | `{}` |
| UI→PY | `jog_robot` | `{axis, direction, distance}` |
| UI→PY | `set_teaching_mode` | `{enable: boolean}` |
| UI→PY | `set_robot_speed` | `{speed: number}` |
| UI→PY | `update_gripper_type` | `{gripper_type: 'gripper'\|'vacuum'}` |
| UI→PY | `gpio_read` | `{port_type, ionum}` |
| UI→PY | `gpio_write` | `{port_type, ionum, value}` |
| UI→PY | `gpio_get_all` | `{}` |
| PY→UI | `pong` | `{}` |
| PY→UI | `state_update` | `{state, step, currentCardUid}` |
| PY→UI | `hw_status` | `{robot: string, esp32: string, camera: string}` |
| PY→UI | `detection` | `{color: string, confidence: number}` |
| PY→UI | `error` | `{message: string, code: string}` |
| PY→UI | `card_complete` | `{cardUid: string, result: {}}` |
| PY→UI | `serial_ports` | `{ports: [{device, description, hwid}]}` |
| PY→UI | `sensor_state` | `{ir1: boolean, ir2: boolean}` |
| PY→UI | `robot_position` | `{status, x, y, z, roll, pitch, yaw}` |
| PY→UI | `position_update` | `{name, ok, message}` |
| PY→UI | `jog_result` | `{status, position?}` |
| PY→UI | `teaching_mode_changed` | `{enabled: boolean}` |
| PY→UI | `robot_speed_changed` | `{speed: number}` |
| PY→UI | `gpio_state` | `{ci[], co[], ai[], tgpio[]}` |
| PY→UI | `gpio_read_result` | `{status, port, value}` |
| PY→UI | `gpio_write_result` | `{status, port, value}` |

### 3. Serial 프로토콜 (Python ↔ ESP32)
115200bps, JSON + `\n` 구분, 타임아웃 3초, 재시도 2회.

요청: `{"cmd": "CONV_START", "params": {}}\n`
응답: `{"status": "OK"}\n`
이벤트: `{"event": "sensor", "sensor": "ir1", "value": true}\n`

### 4. 상태 머신
```
IDLE → (execute) → RUNNING → (card done) → RUNNING → ... → IDLE
                  → WAITING (센서 대기)
                  → ERROR (오류 시 모든 장비 정지)
         (pause) → PAUSED → (resume) → RUNNING
         (e_stop) → E_STOP (하드 컷)
```

### 5. 카메라
ESP32-CAM MJPEG 스트림: `<img src="http://{cameraIp}:81/stream" />`
캡처: `fetch("http://{cameraIp}/capture")` → JPEG blob
색상 분석: Python 백엔드에서 OpenCV 처리 후 WebSocket으로 결과 전달.

### 6. 안전
- 교육 모드: 로봇 속도 10~300mm/s (기본 100, 슬라이더 조절), 가속도 비례 자동 조정, 충돌 감도 3/5
- 동작 범위: 사전 정의된 위치(HOME/INITIAL/ZERO/A/B/C) + workspace_limits 검증
- 조그 제한: XYZ max 50mm/step, RPY max 10°/step, IDLE에서만 허용
- E-STOP: UI 버튼 → WebSocket → Python → 모든 장비 정지 + Teaching 모드 자동 해제 + GPIO 출력 리셋
- 실행 전 확인 대화상자 필수

### 6a. 그리퍼 (Lite 6)
UI에서 그리퍼 타입 선택 (연결 중에도 전환 가능):

| 타입 | 집기 (pick) | 놓기 (place) | 정지 (stop) |
|------|------------|-------------|------------|
| `gripper` (표준) | `close_lite6_gripper()` | `open_lite6_gripper()` | `stop_lite6_gripper()` |
| `vacuum` (진공) | `set_vacuum_gripper(True)` | `set_vacuum_gripper(False)` | `set_vacuum_gripper(False)` |

### 6b. 네트워크 구성
| 장비 | IP 대역 | 연결 방식 | 비고 |
|------|---------|----------|------|
| 노트북 ↔ Lite 6 | `192.168.1.xxx` | LAN 직결 | xArm SDK TCP/IP |
| 노트북 ↔ ESP32-CAM | `192.168.1.xxx` 또는 `192.168.10.xx` | Wi-Fi | 사무실/현장 대역 다름 |
| 노트북 ↔ ESP32 (컨베이어) | USB Serial | 유선 | pyserial 115200bps |

- 현장: LAN(로봇) + Wi-Fi(ESP-CAM) 동시 사용, macOS 자동 라우팅
- 사무실: Wi-Fi만 (ESP-CAM 192.168.1.xxx로 접속)

### 6c. 로봇 위치 교시
- UI: ConnectionPanel → PositionEditor (로봇 연결 시 표시)
- 사전 정의 위치: HOME, INITIAL, ZERO, A, B, C
  - **INITIAL**: X:200 Y:200 Z:200 R:180 P:0 W:0, J[0,9.9,31.8,0,21.9,0] (관절 이동)
  - **ZERO**: X:87 Y:0 Z:154.2 R:180 P:0 W:0, J[0,0,0,0,0,0] (관절 이동)
- 교시 방법: Teaching 모드(Mode 2)로 로봇 이동 → "현재 위치 읽기" → 저장
- joints 필드가 있는 위치는 `set_servo_angle()`로 관절 이동, 없으면 `set_position()`으로 Cartesian 이동
- `get_robot_position` → `arm.get_position()` → UI에 좌표 표시
- `update_position` → safety_manager 위치 갱신
- 위치 저장: hardwareStore (런타임) + .bridge.json (파일)

### 7. 전원 설계 (실제 환경)
메인 전원: **민웰 RS-100-24** SMPS (24V 4.5A, 108W)에서 전체 시스템 통합 공급.

**24V 직접 공급 경로:**
- RS-100-24 → TB6600 → 스텝모터 (컨베이어) — 피크 ~2A (48W)

**5V 변환 경로 (스텝다운 모듈 24V→5V, LM2596 3A급):**
- → ESP32 DevKit V1 (~250mA)
- → ESP32-CAM (~350mA, WiFi+카메라)
- → E18-D80NK 적외선 센서 x1 (~30mA)

총 소비: 피크 ~53W / RS-100-24 용량 108W = 49% (충분한 여유)

**Lite 6 로봇팔은 별도 AC 전원** (자체 어댑터). RS-100-24와 공유하지 않음.

**E18-D80NK 센서 배선:**
- 갈색 = VCC(5V), 파랑 = GND, 검정 = OUT(NPN, 감지 시 LOW)
- OUT → ESP32 GPIO 22 (INPUT_PULLUP)
- 반드시 5V 전원 사용 (3.3V 불가)

**ESP32-CAM UART 연결:**
- ESP32 메인 GPIO 16 (TX2) → ESP32-CAM RX
- ESP32 메인 GPIO 17 (RX2) → ESP32-CAM TX

**ESP32-CAM + ESP32 메인보드가 같은 스텝다운에서 5V를 받으므로 GND 자동 공유** → UART 통신용 별도 GND 불필요

## 개발 순서 (Sprint by Sprint)

### Sprint 1: 기반 구축 ✅ 완료
1. electron-vite + Electron 39 + React 19 + TypeScript 5.9 셋업 ✅
2. Tailwind CSS 3.4 설정 (카드 카테고리 커스텀 색상 포함) ✅
3. Python WebSocket 서버 (`python/protocol/ws_server.py`) — asyncio + websockets 13.x ✅
4. `src/main/python-manager.ts` — Python 프로세스 spawn/kill + BRIDGE_READY 감지 ✅
5. `src/renderer/src/utils/websocket-client.ts` — 싱글턴 WS + 자동 재연결 (지수 백오프) ✅
6. `src/renderer/src/components/layout/MainLayout.tsx` — 3패널 다크 테마 레이아웃 ✅
7. ping/pong 동작 확인 ✅

### Sprint 2: 카드 에디터 ✅ 완료
1. `types/cards.ts` — CardDefinition, FlowCard, BridgeFile, RobotPosition 타입 ✅
2. `utils/card-definitions.ts` — 5카테고리 16종 카드 정의 ✅
3. `stores/flowStore.ts` — Zustand (add/insert/remove/move/select/update/clear) ✅
4. `components/card-editor/CardPalette.tsx` — 카테고리별 드래그 가능한 카드 목록 ✅
5. `components/card-editor/CardItem.tsx` — 카드 UI + 파라미터 편집 + useSortable ✅
6. `components/card-editor/FlowCanvas.tsx` — @dnd-kit 드래그&드롭 + 정렬 ✅
7. Header.tsx에서 .bridge.json 저장/불러오기 (Electron IPC 파일 다이얼로그) ✅
8. `components/card-editor/CardParamEditor.tsx` — number/select 파라미터 입력 ✅

### Sprint 3: 컨베이어 연동 ✅ 완료
1. `firmware/conveyor_controller/conveyor_controller.ino` — ESP32 펌웨어 v5.1 ✅
   - TB6600 스텝모터: ESP32 하드웨어 타이머 ISR로 정밀 펄스 생성 (AccelStepper 미사용)
   - 핀: STEP=GPIO 18, DIR=GPIO 19(HIGH 고정), ENA=GPIO 21
   - E18-D80NK IR 센서 (GPIO 22, 디바운스 200ms)
   - ESP32-CAM UART2 통신 (TX=GPIO 16, RX=GPIO 17)
   - 5단계 속도 (500~1600µs 간격), 기본 Lv1 (1600µs = 625 steps/s)
   - 센서 감지 시 자동 정지, 해제 시 자동 재시작 (userStarted 플래그)
   - 모터: 42BYG45040-24 (NEMA17, 1.8°, 1.68A)
   - TB6600 DIP: SW1=ON SW2=ON (16마이크로스텝), SW4=ON SW5=OFF SW6=OFF (2.0A)
2. `firmware/esp32_cam/esp32_cam_firmware.ino` — ESP32-CAM 펌웨어 ✅
   - HTTP :80 /capture, /status + :81 /stream (MJPEG)
   - Serial UART: ready/status(30초 heartbeat)/capture_done 이벤트
   - Wi-Fi 자동 재연결, PSRAM 자동 해상도 선택
3. `python/hardware/esp32_controller.py` — pyserial async 통신 + CAM 이벤트 ✅
   - CAM_EVENTS 처리 (cam_ready/cam_status/cam_capture_done/cam_disconnected)
   - wait_for_sensor, read_sensor, emergency_stop, cam_capture, cam_status
4. `python/flow_engine/engine.py` — 순차실행 + LOOP/IF/일시정지/E-STOP ✅
5. `python/flow_engine/card_executor.py` — 16종 카드 시뮬/실제 디스패치 ✅
6. `python/flow_engine/state_machine.py` — 6상태 전이 관리 ✅
7. `python/simulation/simulator.py` — 전 장비 Mock (랜덤 타이밍/결과) ✅

### Sprint 4: 로봇팔 연동 ✅ 완료
1. `python/hardware/robot_controller.py` — xArm SDK 래핑 (HAS_XARM 플래그, async executor) ✅
2. `python/hardware/safety_manager.py` — 교육모드(100mm/s), 위치검증, e_stop_all ✅
3. 사전 정의 위치 관리 (HOME/A/B/C) + workspace_limits ✅
4. ws_server.py — connect_hw/disconnect_hw/update_position 핸들러 ✅
5. E-STOP: Header.tsx 버튼 → WS → safety.e_stop_all() → 로봇+ESP32+시뮬 동시 정지 ✅
6. **참고**: EStopButton.tsx 별도 컴포넌트 대신 Header.tsx에 E-STOP 통합 ✅

### Sprint 5: 카메라 + 통합 ✅ 완료
**Python 백엔드 완료:**
1. `python/hardware/camera_controller.py` — aiohttp 캡처 + OpenCV HSV 색상분석 ✅
2. 비전 카드 연동 (card_executor: COLOR_DETECT/CAPTURE 핸들러) ✅
3. ws_server.py — 카메라 connect/disconnect + detection 브로드캐스트 ✅
4. `python/tests/test_camera_controller.py` — 캡처/색상분석 테스트 ✅

**프론트엔드 완료:**
5. `components/camera/CameraViewer.tsx` — MJPEG 스트림 뷰어 (3초 자동 재시도 + key 리마운트) ✅
6. `components/camera/DetectionOverlay.tsx` — 색상 감지 오버레이 (3초 자동 숨김) ✅
7. `components/monitor/HardwareStatus.tsx` — 실행 상태 + 진행률 바 + 현재 카드 ✅
8. `components/monitor/ConnectionPanel.tsx` — 로봇/ESP32(자동 포트 감지)/카메라 연결 관리 ✅
9. `CardItem.tsx` 강화 — 실행 중 ▶ 깜빡임, 완료 ✓, 실행 중 편집 잠금 ✅
10. `FlowCanvas.tsx` 강화 — 현재 카드 자동 스크롤, 실행 중 삭제 잠금 ✅
11. `MainLayout.tsx` — 우측 패널에 카메라뷰 + 모니터 통합 ✅

**하드웨어 통합 완료:**
12. ws_server.py — ESP32 실제 연결/해제 구현 + list_ports 핸들러 ✅
13. Electron webSecurity:false — MJPEG 스트림 보안 정책 해제 ✅
14. hardwareStore — cameraIp 상태 추가 (카메라 뷰어 연동) ✅
15. 컨베이어 모터 하드웨어 검증 완료 (배선/DIP/속도 테스트) ✅

### Sprint 6: PLC 카드 시스템 재설계 ✅ 완료
**Phase A** ✅: 타입 시스템 (CardStructureType/CardShape) + block_parser.py (AST 파서) + condition_evaluator.py (조건 평가기) — 35 tests
**Phase B** ✅: Flow Engine v2 (AST 재귀 실행, IF/ELSE/LOOP/WHILE/PARALLEL, 연속 스캔 모드) — 10 tests
**Phase C** ✅: flowStore (computeDepths, 자동 페어링, 블록 삭제) + CardItem (structureType별 5종 스타일) + FlowCanvas (흐름길 레일, 들여쓰기) + CardPalette (block_close/middle 비표시)
**Phase D** ✅: 센서 모니터 백그라운드 태스크 + Header 연속 모드 토글 + 센서 LED 실시간 표시
**Phase E** ✅: 미션 예제 3종 (초급/중급/고급) + v1→v2 마이그레이션 + 미션 UI

**전체 테스트**: 85개 통과

**이번 세션 추가 완료 (2026-03-17):**
- robot_controller.py — Lite6 전용 그리퍼 API (open/close/stop) + vacuum 분기 ✅
- robot_controller.py — get_current_position() 위치 읽기 ✅
- ws_server.py — get_robot_position 핸들러 + 카메라 IP 빈값 가드 ✅
- card_executor.py / block_parser.py — ROBOT_GRIPPER_STOP 카드 ✅
- card-definitions.ts — 그리퍼 정지 카드 (25종) ✅
- hardwareStore.ts — robotPositions + livePosition 상태 ✅
- App.tsx — robot_position WS 수신 ✅
- PositionEditor.tsx — 위치 교시 UI (현재 위치 읽기 → 적용 → 저장) ✅
- ConnectionPanel.tsx — 그리퍼 타입 선택 (표준/진공) + PositionEditor 통합 ✅

**카드 디자인 미완성 (Sprint 7에서 진행):**
- 다이아몬드/순환/포크 형태 카드 실제 렌더링 ⬜
- SVG 커스텀 일러스트 (이모지 대체) ⬜
- 카테고리별 색상 배경 ⬜
- 블록 범위 괄호 연결선 ⬜
- 실행 상태 조건 참/거짓 뱃지 ⬜

### Sprint 7: 로봇 Live Control + GPIO ✅ 완료 (Phase A+B)

**Phase A — Live Control ✅:**
1. `robot_controller.py` — `jog_cartesian()` Incremental Position 방식 (Mode 0 유지) ✅
2. `robot_controller.py` — `set_teaching_mode()` (Mode 2 ↔ 0), `ensure_position_mode()` ✅
3. `robot_controller.py` — `apply_speed()` xArm TCP/Joint maxacc 즉시 반영 ✅
4. `robot_controller.py` — `_move_joints()` 관절 이동 (INITIAL/ZERO) ✅
5. `ws_server.py` — jog_robot, set_teaching_mode, set_robot_speed 핸들러 ✅
6. `LiveControlPanel.tsx` — 조그 버튼 (XYZ/RPY ±), 이동 간격 선택, 속도 슬라이더 (10~300mm/s), Teaching 토글 ✅
7. `PositionEditor.tsx` — HOME/INITIAL/ZERO/A/B/C 6개 위치 표시 ✅
8. `Toast.tsx` — 에러 토스트 알림 (5초 자동 숨김) ✅
9. `safety_manager.py` — `set_speed()` (10~300mm/s 클램프) ✅

**Phase B — GPIO 모니터/제어 + 카드 ✅:**
1. `robot_controller.py` — CGPIO/TGPIO 읽기/쓰기 7개 메서드 ✅
2. `simulator.py` — GPIO 상태 (CI 8DI/CO 8DO/AI 2/TGPIO 2) + E-STOP 리셋 ✅
3. `ws_server.py` — gpio_read, gpio_write, gpio_get_all, update_gripper_type 핸들러 ✅
4. `GpioPanel.tsx` — CI LED + CO 토글 + AI 전압 + TGPIO 토글 ✅
5. `card_executor.py` — GPIO_SET, GPIO_READ, TGPIO_SET, WAIT_GPIO 핸들러 ✅
6. `condition_evaluator.py` — gpio_ci0~ci7 조건 평가 ✅
7. `card-definitions.ts` — GPIO 카드 4종 (버튼 입력 대기, 출력 켜기, 출력 끄기, 툴 출력 설정) ✅
8. `if_cond` 조건에 GPIO CI0~CI3 추가 ✅

**전체 테스트**: 120개 통과

**Lite 6 GPIO 사양:**
| 종류 | 핀 | 방향 | 수량 | 전압 | SDK 메서드 |
|------|-----|------|------|------|-----------|
| CGPIO Digital | CI0~CI7 | 입력 | 8 | 24V | `get_cgpio_digital(ionum)` |
| CGPIO Digital | CO0~CO7 | 출력 | 8 | 24V (NPN) | `set_cgpio_digital(ionum, val)` |
| CGPIO Analog | AI0~AI1 | 입력 | 2 | 0~5V | `get_cgpio_analog(ionum)` |
| CGPIO Analog | AO0~AO1 | 출력 | 2 | 0~5V | `set_cgpio_analog(ionum, val)` |
| TGPIO Digital | IO0, IO1 | I/O | 2 | 5V | `get/set_tgpio_digital(ionum)` |

### Sprint 7 Phase C: 카드 시각 디자인 ✅ 완료 (Sprint 9에서 마무리)
- 다이아몬드(◆) 형태 IF 카드 CSS clipPath 렌더링 ✅
- 순환(↻) 형태 LOOP 카드 둥근 모서리 + 그라디언트 ✅
- 포크(⑃) 형태 PARALLEL 카드 뱃지 ✅
- ✓참/✗거짓 조건 실행 뱃지 (condition_result WS) ✅
- n/N 반복 진행 카운터 (loop_progress WS) ✅
- SVG 커스텀 아이콘 17종 (CardIcon.tsx) ✅ — Sprint 9에서 구현
- 블록 범위 괄호 연결선 (BlockZone 좌측 세로선 + ┌┘ 모서리) ✅ — Sprint 9에서 구현

### Sprint 8: 텍스트/음성/사운드 카드 + 그래픽 모니터링 ✅ 완료

**Phase A — 텍스트 표시 카드 + 메시지 보드 ✅:**
- `TEXT_DISPLAY` 카드 (text, size: 소/중/대 파라미터) ✅
- `MessageBoard.tsx` — 우측 패널, 최대 50개 로그 쌓기, 자동 스크롤, 지우기 버튼 ✅
- `hardwareStore` — `messageBoardItems`, `addBoardMessage()`, `clearBoard()` ✅
- `websocket-client.ts` — StrictMode 이중 연결 버그 수정 (이벤트 핸들러 null 초기화) ✅

**Phase B — 음성 안내 카드 (TTS) ✅:**
- `SPEAK` 카드 (text, speed 파라미터) ✅
- Web Speech API (`window.speechSynthesis`) ko-KR, cancel+setTimeout 레이스 컨디션 처리 ✅
- `App.tsx` — E_STOP에서만 TTS 중단 (IDLE 전환 시 중단 안 함) ✅

**Phase C — 배경음/효과음 ✅:**
- `PLAY_SOUND` 카드 (효과음 5종: beep/success/error/alert/done) ✅
- `BGM_START` / `BGM_STOP` 카드 (calm/upbeat/custom 트랙) ✅
- `audio-player.ts` — Web Audio API 전용 (AudioContext, OscillatorNode, BufferSourceNode) ✅
- `BgmUpload.tsx` — MP3/WAV/OGG 업로드 → `decodeAudioData()` → AudioBuffer 루프 재생 ✅
- `Header.tsx` — E-STOP / Reset 버튼에서 WS 라운드트립 없이 즉시 BGM/TTS 중단 ✅
- `main/index.ts` — `autoplay-policy: no-user-gesture-required` 설정 ✅

**Phase D — 그래픽 모니터링 뷰 ✅:**
- `SystemIllustration.tsx` — SVG 시스템 일러스트 (중앙 패널 하단 분할) ✅
  - 컨베이어 벨트: 실행 중 파란 대시 스크롤 CSS 애니메이션
  - IR 센서: `sensorIr1` 반응형 LED 펄스 + 감지 빔
  - 카메라: 연결 상태 반응형 + FOV 빔 표시
  - 로봇팔: `livePosition` → 2축 근사 IK로 엔드 이펙터 위치 갱신
  - 엔진 상태 뱃지 (실행 중/긴급 정지 등) + 하드웨어 연결 뱃지
  - 헤더 클릭으로 접기/펼치기 토글

**카드 수: 29종 → 34종** (TEXT_DISPLAY, SPEAK, PLAY_SOUND, BGM_START, BGM_STOP 추가)

### Sprint 9: 팔레트 이중 분류 체계 + 단계별 해금 + 47종 확장 ✅ 완료

**Phase A — 타입 + 카드 정의 + 팔레트 ✅:**
- `types/cards.ts` — PaletteCategory(7개), HwTag, UnlockProfile, PaletteCategoryInfo 추가. BridgeFile v1.3 (unlockProfile 필드) ✅
- `card-definitions.ts` — 47종 카드, 7개 팔레트 카테고리, CARD_ID_MIGRATION 맵, getCardsByPaletteCategory() ✅
- `settingsStore.ts` — Zustand persist, unlockLevel 1~4, customUnlocked, plcBadgeMode, isCategoryUnlocked() ✅
- `CardPalette.tsx` — 해금 레벨별 잠금/해금, hwTag 뱃지, plcTerm 뱃지 ✅
- `Header.tsx` — 해금 레벨 select 드롭다운 + PLC 뱃지 토글 버튼 추가 ✅
- `Header.tsx` — migrateCards(): CARD_ID_MIGRATION 리네이밍 + unlockProfile 적용 ✅

**Phase B — CardItem 뱃지 ✅:**
- `CardItem.tsx` — hwTag 뱃지 + plcTerm 뱃지 (plcBadgeMode ON 시) ✅

**Phase C — 카드 시각 디자인 완성 ✅:**
- `CardIcon.tsx` — SVG 커스텀 아이콘 17종 (CARD_SVG_ICONS 맵, 이모지 폴백) ✅
- `FlowCanvas.tsx` BlockZone 리팩터 — 통일된 좌측 세로선 + ┌/└ 모서리 괄호 ✅
- `CardItem.tsx`, `CardPalette.tsx` — SVG 아이콘 연동 ✅

**Phase D — Python 백엔드 ✅:**
- `block_parser.py` — LoopNNode, SubDefineNode 추가 (loop_n/end_loop_n, sub_define/sub_define_end) ✅
- `engine.py` — SubReturnException, _subroutines dict, _execute_loop_n(), _execute_sub_define(), SUB_CALL/SUB_RETURN 처리 ✅
- `card_executor.py` — variables dict, TIMER_ON/COUNTER_UP/COUNTER_RESET/VAR_DECLARE/VAR_SET/VAR_GET/VAR_CALC 핸들러 ✅
- `card_executor.py` — GPIO pin 파라미터 파싱 개선: 'CI0'/'CO0'/'IO1' 형식 직접 지원 ✅
- `card-definitions.ts` — gpio_wait_input 커맨드 GPIO_READ→WAIT_GPIO 수정 ✅

**단계별 해금 정책:**
| 레벨 | 차시 | 해금 카테고리 |
|------|------|--------------|
| 1 | 1~4차시 | ① 흐름 제어, ② 입력, ③ 출력 |
| 2 | 5~8차시 | + ④ 조건/분기, ⑤ 타이머/카운터 |
| 3 | 9~12차시 | + ⑥ 변수/데이터 |
| 4 | 13~16차시 | + ⑦ 서브루틴 (전체) |

**전체 테스트**: 120개 통과

### Sprint 10: 전체 통합 테스트
- Phase 1 전체 + Sprint 8~9 기능 End-to-End
- 해금 정책 검증
- 2시간 연속 동작 안정성 테스트

### Sprint 11: ST 블록 에디터 — Phase 2a ✅ 완료

**학습 경로:**
```
Phase 1: 카드 모드 (초등)  →  Phase 2a: ST 블록 모드 (중등)  →  Phase 2b: ST 텍스트 모드 (고등)  →  현장: 실제 PLC
```

1. `src/renderer/src/utils/st-code-generator.ts` (신규) — FlowCard[] → ST 코드 + lineMap (cardUid → 라인번호) ✅
2. `src/renderer/src/utils/monaco-iec-st.ts` (신규) — Monaco IEC ST 커스텀 언어 (Monarch 토크나이저, 라이트/다크 테마, 34키워드 호버 힌트) ✅
3. `src/renderer/src/components/st-editor/STFBPalette.tsx` (신규) — ST 라이브러리 팔레트 (6그룹: 컨베이어/로봇/센서/비전/타이머/GPIO) ✅
4. `src/renderer/src/components/st-editor/STBlockCanvas.tsx` (신규) — 카드 + ST 키워드 뱃지, depth 들여쓰기, 활성 하이라이트 ✅
5. `src/renderer/src/components/st-editor/STEditorPanel.tsx` (신규) — 좌: STFBPalette + ST 예제, 우: 블록캔버스 + Monaco 에디터 ✅
6. `settingsStore.ts` (수정) — `editorMode: 'card' | 'st'` 상태 (기본값 'card'), `setEditorMode` 액션 추가 ✅
7. `types/cards.ts` (수정) — BridgeFile 버전 1.4, `mode: 'card' | 'st'` ✅
8. `Header.tsx` (수정) — [블록] / [ST 코드] 모드 탭 버튼, 저장/불러오기 모드 처리 ✅
9. `MainLayout.tsx` (수정) — `editorMode === 'st'` 시 STEditorPanel 조건부 렌더링 ✅
10. 양방향 하이라이트: 카드 클릭 → Monaco revealLine; Monaco 커서 위치 → 블록 하이라이트 ✅

### Sprint 12: ST 텍스트 에디터 — Phase 2b ✅ 완료

1. `src/renderer/src/utils/st-parser.ts` (신규) — ST → FlowCard[] 역방향 파서 (`parseSTCode` 함수) ✅
   - 섹션 상태 머신: preamble | function_block | var | body | done
   - BlockStack 클래스 — open/close/middle 블록 페어링
   - `tryMultiline()`: 5종 멀티라인 카드 시그니처 (delay, timer_on, counter_up, counter_reset, gpio_wait_input)
   - `matchSingle()`: 47종 카드 패턴 (우선순위 순)
   - REVERSE_COND 맵 — 조건 역방향 매핑
2. `monaco-iec-st.ts` (수정) — KO_HOVER 맵 (34종 한글 키워드 설명), `registerHoverProvider` 추가 ✅
3. Monaco 에디터 `readOnly: false`, 600ms 디바운스 parse-on-edit, `monacoEditingRef` 루프 방지 플래그 ✅
4. 파싱 경고 뱃지: "⚠ N줄 미인식" (호버 툴팁), "파싱 중…" 애니메이션 뱃지 ✅
5. ST 예제 파일 4종 (`src/renderer/public/missions/`) ✅
   - `st_example1_belt.st` — 벨트 속도 제어
   - `st_example2_sensor.st` — 센서 + IF/ELSE 로봇 이동
   - `st_example3_counter.st` — FOR 루프 5회 반복
   - `st_example4_sorting.st` — FUNCTION_BLOCK SortItem + WHILE TRUE 색상 분류

### Sprint 13: 확장 IEC 뷰 — Phase 2c ✅ PLC 내보내기 완료 (SFC/LD 미완료)

1. `src/renderer/src/utils/st-exporter.ts` (신규) — 3종 PLC 내보내기 포맷 ✅
   - `iec_st`: 순수 IEC 61131-3 ST (.st) — 범용
   - `codesys`: CODESYS V3.5 ST (.st) — CODESYS V3.5.14+ / OpenPLC Runtime 호환
   - `tia_scl`: Siemens TIA Portal SCL (.scl) — PROGRAM→ORGANIZATION_BLOCK, 단/쌍따옴표 변환
2. `src/main/index.ts` (수정) — `dialog:exportSTFile` IPC 핸들러 (.st/.scl 파일 저장 다이얼로그) ✅
3. `src/preload/index.ts` / `index.d.ts` (수정) — `exportSTFile(data, defaultName, ext)` API 추가 ✅
4. `STEditorPanel.tsx` — "PLC 내보내기 ▾" 드롭다운 버튼 (3종 포맷) ✅
5. SFC(순차 기능 차트) 뷰 — 미완료 ⬜
6. LD(래더 다이어그램) 뷰 — 미완료 ⬜

**하드웨어 확장성:**
- ST 문법은 하드웨어 독립적 (IEC 국제 표준)
- 로봇/컨베이어/센서를 다른 제품으로 교체해도 ST 코드 동일
- `CardExecutor` 하드웨어 드라이버 레이어만 교체하면 됨

## 카드 정의 — 현재 구현 47종 (Sprint 9 기준)

> 7개 프로그래밍 개념 카테고리 (paletteCategoryId) + 하드웨어 태그 뱃지 (hwTags) 이중 분류 체계.
> 카드 ID 리네이밍 완료: wait_gpio→gpio_wait_input, gpio_on→gpio_output_on, gpio_off→gpio_output_off, tgpio_set→gpio_tool_set

### ① 흐름 제어 (9종) — unlockLevel 1

| id | name | command | shape | structureType |
|----|------|---------|-------|---------------|
| start | 시작 | FLOW_START | pill | marker |
| stop | 끝 | FLOW_STOP | pill | marker |
| loop | 반복하기 | LOOP | loop | block_open |
| end_loop | 반복하기 끝 | END_LOOP | bar | block_close |
| while_cond | ~하는 동안 | WHILE | loop | block_open |
| end_while | 반복 끝 | END_WHILE | bar | block_close |
| parallel_start | 동시에 시작 | PARALLEL_START | fork | block_open |
| parallel_branch | 작업 나누기 | PARALLEL_BRANCH | bar | block_middle |
| parallel_end | 모두 완료되면 | PARALLEL_END | bar | block_close |

### ② 입력 (4종) — unlockLevel 1

| id | name | command | hwTags | structureType |
|----|------|---------|--------|---------------|
| wait_sensor | IR 감지 대기 | WAIT_SENSOR | sensor | action |
| color_detect | 색깔 확인 | COLOR_DETECT | camera | action |
| capture | 카메라 촬영 | CAPTURE | camera | action |
| gpio_wait_input | 버튼 입력 대기 | WAIT_GPIO | gpio | action |

### ③ 출력 (16종) — unlockLevel 1

| id | name | command | hwTags | structureType |
|----|------|---------|--------|---------------|
| conv_start | 컨베이어 켜기 | CONV_START | conveyor | action |
| conv_stop | 컨베이어 끄기 | CONV_STOP | conveyor | action |
| conv_speed | 컨베이어 속도 | CONV_SPEED | conveyor | action |
| robot_home | 로봇 홈으로 | ROBOT_HOME | robot | action |
| robot_move | 로봇 이동 | ROBOT_MOVE | robot | action |
| robot_pick | 물건 집기 | ROBOT_PICK | robot | action |
| robot_place | 물건 놓기 | ROBOT_PLACE | robot | action |
| robot_gripper_stop | 그리퍼 정지 | ROBOT_GRIPPER_STOP | robot | action |
| gpio_output_on | 출력 켜기 | GPIO_SET | gpio | action |
| gpio_output_off | 출력 끄기 | GPIO_SET | gpio | action |
| gpio_tool_set | 툴 출력 설정 | TGPIO_SET | gpio | action |
| text_display | 메시지 보내기 | TEXT_DISPLAY | — | action |
| speak | 말하기 | SPEAK | — | action |
| play_sound | 효과음 | PLAY_SOUND | — | action |
| bgm_start | 배경음 시작 | BGM_START | — | action |
| bgm_stop | 배경음 정지 | BGM_STOP | — | action |

### ④ 조건/분기 (4종) — unlockLevel 2

| id | name | command | shape | structureType |
|----|------|---------|-------|---------------|
| if_cond | 만약~이면 | IF_CONDITION | diamond | block_open |
| else | 아니면 | ELSE | bar | block_middle |
| end_if | 만약~끝 | END_IF | bar | block_close |
| if_sensor | 물체가 있으면 | IF_SENSOR | diamond | block_open |

### ⑤ 타이머/카운터 (6종) — unlockLevel 2

| id | name | command | plcTerm | structureType |
|----|------|---------|---------|---------------|
| delay | N초 기다리기 | DELAY | TON | action |
| timer_on | N초 후 실행 | TIMER_ON | TOF | action |
| counter_up | 횟수 세기 | COUNTER_UP | CTU | action |
| counter_reset | 카운터 초기화 | COUNTER_RESET | RES | action |
| loop_n | 카운터 반복 | LOOP_N | FOR | block_open |
| end_loop_n | 카운터 반복 끝 | END_LOOP_N | — | block_close |

### ⑥ 변수/데이터 (4종) — unlockLevel 3

| id | name | command | structureType |
|----|------|---------|---------------|
| var_declare | 변수 만들기 | VAR_DECLARE | action |
| var_set | 값 저장하기 | VAR_SET | action |
| var_get | 값 불러오기 | VAR_GET | action |
| var_calc | 계산하기 | VAR_CALC | action |

### ⑦ 서브루틴 (4종) — unlockLevel 4

| id | name | command | plcTerm | structureType |
|----|------|---------|---------|---------------|
| sub_define | 서브루틴 정의 | SUB_DEFINE | FB | block_open |
| sub_define_end | 서브루틴 끝 | SUB_DEFINE_END | — | block_close |
| sub_call | 서브루틴 호출 | SUB_CALL | FB호출 | action |
| sub_return | 값 돌려주기 | SUB_RETURN | — | action |

## 저장 파일 포맷 (.bridge.json)
```json
{
  "version": "1.4",
  "name": "택배 분류 자동화",
  "mode": "card",
  "cards": [
    {"uid": "abc123", "definitionId": "start", "params": {}},
    {"uid": "def456", "definitionId": "conv_start", "params": {}}
  ],
  "robotPositions": {
    "A": {"x": 200, "y": 0, "z": 150, "roll": 180, "pitch": 0, "yaw": 0},
    "B": {"x": 200, "y": 150, "z": 150, "roll": 180, "pitch": 0, "yaw": 0},
    "C": {"x": 200, "y": -150, "z": 150, "roll": 180, "pitch": 0, "yaw": 0}
  },
  "settings": {
    "robotIp": "192.168.1.100",
    "esp32Port": "COM3",
    "cameraIp": "192.168.1.101"
  },
  "unlockProfile": {"level": 4}
}
```

## 주의사항
- 모든 UI 텍스트는 한글 (영어 코드명은 내부에서만 사용)
- 초등 고학년 대상이므로 용어를 쉽게: "컨베이어 시작" → "벨트 출발"
- 안전이 최우선: E-STOP은 항상 동작해야 하며, 교육 모드 속도 제한 필수
- 시뮬레이션 모드로 하드웨어 없이도 개발/테스트 가능해야 함
- Phase 2(IEC 61131-3 ST 블록 코딩) 확장을 고려해 Flow Engine은 카드 배열을 입력으로 받는 범용 구조로 설계
- ST 함수 블록은 기존 29종 카드 커맨드와 1:1 매핑되므로 Engine 변경 없이 ST 지원 가능
