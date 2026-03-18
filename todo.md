# todo.md — 개발 작업 목록

> **최종 수정일**: 2026-03-19
> **진행 상황**: Sprint 11~13 완료 (ST 블록 에디터 + ST 텍스트 에디터 + PLC 코드 내보내기; SFC/LD 미완료)

---

## 범례

- [ ] 미착수
- [~] 진행 중
- [x] 완료

---

## Sprint 1~5: 기반 + 에디터 + 하드웨어 통합 ✅ 완료

모든 항목 완료. 상세 내용은 CLAUDE.md 참고.

---

## Sprint 6: PLC 카드 시스템 재설계 ✅ 완료

### Phase A: 타입 시스템 + 블록 파서 ✅

- [x] `types/cards.ts` — CardStructureType, CardShape, FlowCard depth/blockId
- [x] `card-definitions.ts` — 기존 16종 structureType 추가 + 신규 8종 = 24종
- [x] `block_parser.py` — AST 파서 (ActionNode/IfNode/LoopNode/WhileNode/ParallelNode)
- [x] `condition_evaluator.py` — 센서/색상/컨베이어 조건 평가
- [x] 테스트 35개 통과

### Phase B: Flow Engine 재설계 ✅

- [x] `engine.py` — AST 재귀 실행 (_execute_node, IF/LOOP/WHILE/PARALLEL)
- [x] 연속 스캔 모드 (continuous=true)
- [x] ConditionEvaluator 통합
- [x] 테스트 10개 통과

### Phase C: 프론트엔드 블록 편집기 ✅

- [x] `flowStore.ts` — computeDepths, 자동 페어링, 블록 삭제
- [x] `CardItem.tsx` — structureType별 5종 스타일
- [x] `FlowCanvas.tsx` — 흐름길 레일, 들여쓰기
- [x] `CardPalette.tsx` — block_close/middle 비표시

### Phase D: 실시간 모니터링 + 연속 모드 ✅

- [x] `ws_server.py` — 센서 모니터 백그라운드 태스크 (500ms)
- [x] `hardwareStore.ts` — sensorIr1/Ir2 + setSensorState
- [x] `App.tsx` — sensor_state 메시지 핸들러
- [x] `Header.tsx` — 연속 모드 토글 + 센서 LED 실시간 표시

### Phase E: 미션 예제 + 마이그레이션 ✅

- [x] 미션 1: 벨트 제어 기초 (초급, 순차)
- [x] 미션 2: 센서 활용 (중급, IF/ELSE 조건 분기)
- [x] 미션 3: 택배 분류 (고급, WHILE + IF/ELSE + 카메라 + 로봇)
- [x] v1→v2 .bridge.json 마이그레이션 (depth/blockId 자동 추가)
- [x] Header 미션 로드 UI (초급/중급/고급 버튼)

**전체 테스트: 85개 통과**

---

## Sprint 6 이후: 로봇 제어 강화 ✅ 완료 (2026-03-17)

- [x] `robot_controller.py` — Lite6 전용 그리퍼 API (open/close/stop_lite6_gripper)
- [x] `robot_controller.py` — 진공 그리퍼 분기 (set_vacuum_gripper True/False)
- [x] `robot_controller.py` — get_current_position() 위치 읽기
- [x] `robot_controller.py` — gripper_stop() 그리퍼 정지
- [x] `ws_server.py` — get_robot_position 핸들러 + 카메라 IP 빈값 가드
- [x] `card_executor.py` + `block_parser.py` — ROBOT_GRIPPER_STOP 카드 추가
- [x] `card-definitions.ts` — robot_gripper_stop 추가 (25종)
- [x] `hardwareStore.ts` — robotPositions + livePosition 상태 추가
- [x] `App.tsx` — robot_position WS 메시지 수신
- [x] `PositionEditor.tsx` — 위치 교시 UI (현재 위치 읽기 → 적용 → 저장)
- [x] `ConnectionPanel.tsx` — 그리퍼 타입 선택 (표준/진공) 라디오 버튼
- [x] `ConnectionPanel.tsx` — 카메라 IP placeholder 개선 (사무실/현장 대역)

---

## Sprint 7: 로봇 Live Control + GPIO + 카드 시각 디자인 🔄 Phase A+B 완료 / Phase C 진행 중

### Phase A: Live Control ✅ 완료

- [x] `robot_controller.py` — `jog_cartesian()` Incremental Position 방식 (Mode 0 유지)
- [x] `robot_controller.py` — `set_teaching_mode()` Mode 2 ↔ Mode 0 전환
- [x] `robot_controller.py` — `ensure_position_mode()` 플로우 실행 전 Mode 0 복원 가드
- [x] `robot_controller.py` — `apply_speed()` 속도 즉시 반영 (TCP/Joint maxacc)
- [x] `robot_controller.py` — `_move_joints()` 관절 이동 (INITIAL/ZERO 위치)
- [x] `simulator.py` — `jog_cartesian()`, `set_teaching_mode()` Mock
- [x] `ws_server.py` — `jog_robot` 핸들러 (IDLE 가드, 거리 제한 XYZ 50mm/RPY 10°)
- [x] `ws_server.py` — `set_teaching_mode` 핸들러 + E-STOP 시 자동 해제
- [x] `ws_server.py` — `set_robot_speed` 핸들러 (10~300mm/s)
- [x] `safety_manager.py` — `set_speed()` (10~300mm/s 클램프)
- [x] `messages.py` — `jog_result()`, `teaching_mode_changed()` 헬퍼
- [x] `hardwareStore.ts` — `teachingMode`, `jogStepSize`, `robotSpeed` 상태
- [x] `App.tsx` — `jog_result`, `teaching_mode_changed`, `robot_speed_changed` 핸들러
- [x] `LiveControlPanel.tsx` — XYZ/RPY ±조그 버튼 + 이동 간격 [1/2/5/10/20mm] + 속도 슬라이더 (10~300mm/s) + Teaching 토글
- [x] `ConnectionPanel.tsx` — PositionEditor + LiveControlPanel 통합 (로봇 연결 시 표시)
- [x] `Toast.tsx` — 에러 토스트 알림 (5초 자동 숨김)

### Phase B: GPIO 모니터/제어 + 카드 ✅ 완료

- [x] `robot_controller.py` — CGPIO 7개 메서드 (get/set_cgpio_digital/analog, get/set_tgpio_digital)
- [x] `simulator.py` — GPIO 상태 (CI 8DI / CO 8DO / AI 2 / TGPIO 2) + E-STOP 리셋
- [x] `ws_server.py` — `gpio_read`, `gpio_write`, `gpio_get_all`, `update_gripper_type` 핸들러
- [x] `messages.py` — `gpio_state()`, `gpio_read_result()`, `gpio_write_result()` 헬퍼
- [x] `hardwareStore.ts` — `gpioState` 상태 + `setGpioState()`
- [x] `App.tsx` — `gpio_state` 메시지 핸들러
- [x] `GpioPanel.tsx` — CI LED 표시 + CO 클릭 토글 + AI 전압 + TGPIO 토글
- [x] `HardwareStatus.tsx` — GPIO 패널 통합
- [x] `card_executor.py` — `GPIO_SET`, `GPIO_READ`, `TGPIO_SET`, `WAIT_GPIO` 핸들러
- [x] `condition_evaluator.py` — `gpio_ci0~ci7` 조건 평가
- [x] `card-definitions.ts` — GPIO 카드 4종 추가 (버튼 입력 대기, 출력 켜기, 출력 끄기, 툴 출력 설정) → 총 29종
- [x] `if_cond` 조건 파라미터에 GPIO CI0~CI3 선택지 추가

**전체 테스트: 120개 통과**

### Phase C: 카드 시각 + UI 재설계 ✅ 완료

**완료:**
- [x] `themeStore.ts` — 다크/라이트 테마 토글 (localStorage 영속, `dark` class 제어)
- [x] `tailwind.config.js` — `darkMode: 'class'` 추가
- [x] `Header.tsx` — 팩토스튜디오 브랜딩 + ☀️/🌙 테마 토글 버튼
- [x] `MainLayout.tsx` — 팩토블록 FACTO Block (2×2 SVG 아이콘 + 한/영 병기)
- [x] `FlowCanvas.tsx` — 팩토조립소 FACTO Build Station
- [x] `index.html` + `src/main/index.ts` — 타이틀 "팩토스튜디오 — FACTO Studio"
- [x] 전체 컴포넌트 화이트/다크 테마 대응 (`slate-X dark:gray-X` 패턴)
- [x] `CardItem.tsx` — 컴팩트 가로형 카드 (~44px 높이, 3px 좌측 색상 바)
- [x] `CardItem.tsx` — 파라미터 요약 배지 (카드 오른쪽, 컬러 배경)
- [x] `CardPalette.tsx` — 동일 가로형 카드 디자인 적용
- [x] `FlowCanvas.tsx` — 블록 범위 세로 연결선 (BlockZone 컬러 그라디언트)
- [x] `FlowCanvas.tsx` — `BlockSectionArea` (블록 섹션 내부 드롭 정확도 수정)
- [x] `FlowCanvas.tsx` + `CardPalette.tsx` + `App.tsx` — 캔버스 카드 삭제: 팔레트 드롭 or 하단 휴지통(TrashZone)
- [x] `App.tsx` — 병렬 레인 드롭 + 블록 섹션 드롭 + 삭제 드롭 통합 처리

**추가 완료 (2026-03-18):**
- [x] 다이아몬드(◆) 형태 조건 카드 (IF) — CSS clip-path 좌측 V-노치 + 좌측 ◆ 절대 배치
- [x] 순환(↻) 형태 반복 카드 (LOOP/WHILE) — 좌측 그라디언트 바 + ↺ 뱃지 + 왼쪽 둥근 모서리
- [x] 포크(⑃) 형태 병렬 카드 CSS 강화 — ⑃ 뱃지 표시
- [x] CardPalette.tsx — 동일 shape 스타일 적용 (팔레트 카드 = 캔버스 카드 외관 일치)

**추가 완료 (2026-03-18):**
- [x] 조건 참/거짓 실행 뱃지 (✓ 참 / ✗ 거짓) — `condition_result` WS 메시지 + `conditionResults` 스토어
- [x] 반복 진행 카운터 (n/N 표시) — `loop_progress` WS 메시지 + `loopProgress` 스토어

**완료:**
- [x] SVG 커스텀 아이콘 17종 (CardIcon.tsx) — Sprint 9에서 구현

---

## Sprint 8: 텍스트/음성/사운드 카드 + 그래픽 모니터링 ✅ 완료

### Phase A: 텍스트 표시 카드 ✅

- [x] `card-definitions.ts` — `TEXT_DISPLAY` 카드 (text, size 파라미터)
- [x] `card_executor.py` — `TEXT_DISPLAY` 핸들러 → WS 브로드캐스트
- [x] `messages.py` — `text_display()` 헬퍼
- [x] `App.tsx` — `text_display` 메시지 핸들러
- [x] `MessageBoard.tsx` — 우측 패널, 최대 50개 로그, 자동 스크롤, 지우기 버튼
- [x] `websocket-client.ts` — StrictMode 이중 연결 버그 수정

### Phase B: 음성 안내 카드 (TTS) ✅

- [x] `card-definitions.ts` — `SPEAK` 카드 (text, speed 파라미터)
- [x] `card_executor.py` — `SPEAK` 핸들러 → WS 브로드캐스트
- [x] `App.tsx` — `speak` 핸들러 → Web Speech API (ko-KR, cancel+setTimeout 레이스 컨디션 처리)
- [x] E_STOP에서만 TTS 중단 (IDLE 전환 시 중단 안 함)

### Phase C: 배경음/효과음 ✅

- [x] `card-definitions.ts` — `PLAY_SOUND` / `BGM_START` / `BGM_STOP` 카드
- [x] `card_executor.py` — 사운드 카드 WS 브로드캐스트
- [x] `audio-player.ts` — Web Audio API 전용 (AudioContext, OscillatorNode, BufferSourceNode)
- [x] `BgmUpload.tsx` — MP3/WAV/OGG → `decodeAudioData()` → AudioBuffer 루프 재생
- [x] `Header.tsx` — E-STOP/Reset에서 즉시 BGM/TTS 중단
- [x] `main/index.ts` — `autoplay-policy: no-user-gesture-required`

### Phase D: 그래픽 모니터링 뷰 ✅

- [x] `SystemIllustration.tsx` — SVG 시스템 일러스트 (중앙 패널 하단 분할)
- [x] 컨베이어 벨트 CSS 애니메이션, IR 센서 LED, 카메라 FOV 빔, 로봇팔 2축 IK
- [x] 엔진 상태 뱃지 + 하드웨어 연결 뱃지, 헤더 클릭 접기/펼치기

---

## Sprint 9: 팔레트 이중 분류 체계 + 단계별 해금 ✅ 완료

- [x] `card-definitions.ts` — 47종 카드, 7개 팔레트 카테고리, CARD_ID_MIGRATION 맵
- [x] `CardPalette.tsx` — 해금 레벨별 잠금/해금, hwTag 뱃지, plcTerm 뱃지
- [x] `CardItem.tsx` — hwTag 뱃지 + plcTerm 뱃지 (plcBadgeMode ON 시)
- [x] `settingsStore.ts` — unlockLevel 1~4, customUnlocked, plcBadgeMode, isCategoryUnlocked()
- [x] `Header.tsx` — 해금 레벨 select 드롭다운 + PLC 뱃지 토글 버튼
- [x] `CardIcon.tsx` — SVG 커스텀 아이콘 17종 (CARD_SVG_ICONS 맵)
- [x] `FlowCanvas.tsx` BlockZone 리팩터 — 통일된 좌측 세로선 + ┌/└ 모서리 괄호
- [x] `block_parser.py` — LoopNNode, SubDefineNode 추가
- [x] `card_executor.py` — TIMER_ON/COUNTER_UP/RESET/VAR_*/SUB_* 핸들러
- [x] `.bridge.json` v1.3 마이그레이션 (unlockProfile + 카드 ID 리네이밍)

---

## Sprint 10: 전체 통합 테스트 ⬜

- [ ] Phase 1 카드 전체 미션 4종 완주
- [ ] 해금 정책 각 차시 프로파일 검증
- [ ] 하드웨어 실제 연동 (로봇+컨베이어+카메라+센서)
- [ ] E-STOP 반응시간 < 500ms (10회 반복)
- [ ] TTS/사운드 동시 재생
- [ ] 그래픽 모니터링 뷰 실시간 동기화
- [ ] 2시간 연속 동작 안정성 테스트

---

## Sprint 11: ST 블록 에디터 — Phase 2a ✅ 완료

- [x] `st-code-generator.ts` — FlowCard[] → ST 코드 + lineMap (cardUid → 라인번호)
- [x] `monaco-iec-st.ts` — Monaco IEC ST 커스텀 언어 (Monarch 토크나이저, 라이트/다크 테마)
- [x] `STFBPalette.tsx` — ST 라이브러리 팔레트 (6그룹: 컨베이어/로봇/센서/비전/타이머/GPIO)
- [x] `STBlockCanvas.tsx` — 블록 캔버스 (ST 키워드 뱃지, depth 들여쓰기, 활성 하이라이트)
- [x] `STEditorPanel.tsx` — ST 에디터 전체 패널 (좌: 팔레트+예제, 우: 블록캔버스+Monaco)
- [x] `settingsStore.ts` — `editorMode: 'card' | 'st'` 상태 + `setEditorMode` 액션
- [x] `types/cards.ts` — BridgeFile v1.4, `mode: 'card' | 'st'`
- [x] `Header.tsx` — [블록] / [ST 코드] 모드 탭 버튼, 저장/불러오기 모드 처리
- [x] `MainLayout.tsx` — editorMode === 'st' 시 STEditorPanel 조건부 렌더링
- [x] 양방향 하이라이트 — 카드 클릭 → Monaco revealLine; Monaco 커서 → 블록 하이라이트

## Sprint 12: ST 텍스트 에디터 — Phase 2b ✅ 완료

- [x] `st-parser.ts` — ST → FlowCard[] 역방향 파서 (`parseSTCode`, 47종 패턴)
  - [x] 섹션 상태 머신 (preamble/function_block/var/body/done)
  - [x] BlockStack 클래스 — open/close/middle 블록 blockId 페어링
  - [x] `tryMultiline()` — 5종 멀티라인 카드 시그니처
  - [x] REVERSE_COND 맵 — 조건 역방향 매핑
- [x] `monaco-iec-st.ts` 수정 — KO_HOVER 맵 (34종 한글 키워드 설명), `registerHoverProvider`
- [x] Monaco `readOnly: false`, 600ms 디바운스 parse-on-edit, `monacoEditingRef` 루프 방지
- [x] 파싱 경고 뱃지 — "⚠ N줄 미인식" + 호버 툴팁, "파싱 중…" 애니메이션
- [x] ST 예제 파일 4종 (`src/renderer/public/missions/`)
  - [x] `st_example1_belt.st` — 벨트 속도 제어
  - [x] `st_example2_sensor.st` — 센서 + IF/ELSE 로봇 이동
  - [x] `st_example3_counter.st` — FOR 루프 5회 반복
  - [x] `st_example4_sorting.st` — FUNCTION_BLOCK SortItem + WHILE TRUE 색상 분류

## Sprint 13: 확장 IEC 뷰 — Phase 2c ✅ 부분 완료

- [x] `st-exporter.ts` — PLC 코드 내보내기 (iec_st / codesys / tia_scl 3종 포맷)
- [x] `src/main/index.ts` — `dialog:exportSTFile` IPC 핸들러 (.st/.scl 파일 저장 다이얼로그)
- [x] `src/preload/index.ts` / `index.d.ts` — `exportSTFile(data, defaultName, ext)` API 추가
- [x] `STEditorPanel.tsx` — "PLC 내보내기 ▾" 드롭다운 버튼 (3종 포맷 선택)
- [ ] SFC(순차 기능 차트) 뷰 — 상태 전이도 시각화
- [ ] LD(래더 다이어그램) 뷰 — 읽기 전용 변환

---

## 공통 작업

### 문서

- [x] CLAUDE.md — 설계 가이드 (2026-03-19 갱신: Sprint 11~13 완료 반영, 프로젝트 구조 + BridgeFile v1.4)
- [x] PRD.md — 제품 요구사항 (2026-03-19 갱신: v3.4 — Sprint 11~13 완료, FR-16~21 업데이트)
- [x] tech.md — 기술 명세서 (2026-03-19 갱신: v3.2 — Section 13 실제 구현 반영, exportSTFile IPC)
- [x] todo.md — 개발 작업 목록 (2026-03-19 갱신: Sprint 8~9 ✅, Sprint 11~13 실제 완료 항목 반영)

### 코드 품질

- [ ] ESLint + Prettier 설정
- [ ] Python linter 설정
- [ ] Git 저장소 초기화 + `.gitignore`

### 배포 준비

- [ ] electron-builder 설정
- [ ] Python pyinstaller 번들링 테스트
- [ ] 설치 가이드 문서 작성

---

> **문서 끝** — 작업 완료 시 체크박스를 [x]로 업데이트합니다.
