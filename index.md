# 팩토스튜디오 (FACTO Studio)

**산업용 로봇팔 + 컨베이어 + 카메라 비전을 PLC 카드 방식으로 직관적으로 제어하는 교육용 자동화 체험 플랫폼**

---

## 프로젝트 개요

팩토스튜디오는 UFactory Lite 6 로봇팔, 스텝모터 컨베이어, ESP32-CAM 비전, 적외선 센서를 통합 제어하는 교육용 자동화 플랫폼입니다.

| 구분 | 내용 |
|------|------|
| **플랫폼** | Electron 39 + React 19 + Python 3.11 |
| **통신** | WebSocket (ws://localhost:8765) |
| **로봇** | UFactory Lite 6 (xArm-Python-SDK) |
| **컨베이어** | NEMA17 스텝모터 + TB6600 드라이버 + ESP32 |
| **비전** | ESP32-CAM MJPEG 스트림 + OpenCV 색상 분석 |
| **코딩 환경** | Phase 1: PLC 카드 UI / Phase 2: IEC 61131-3 ST |

---

## 학습 경로

```
Phase 1 (블록 모드)                     Phase 2 (ST 코드 모드)
────────────────────────────────        ──────────────────────────────────
1~4차시:  순차 실행 (입력+출력)  ──▶    Phase 2a: ST 블록 에디터
5~8차시:  조건/분기 + 타이머     ──▶    Phase 2b: ST 텍스트 에디터
9~12차시: 변수/데이터            ──▶    Phase 2c: PLC 코드 내보내기
13~16차시: 서브루틴 (FB 개념)    ──▶    실제 PLC (Codesys, TIA Portal)
```

---

## 주요 기능

### Phase 1 — PLC 블록 에디터
- **47종 카드** (흐름 제어 / 입력 / 출력 / 조건 / 타이머 / 변수 / 서브루틴)
- **단계별 해금**: 수업 차시에 맞게 교사가 카드 범위 설정
- **PLC 뱃지 모드**: TON, CTU, %Q 등 실제 PLC 용어 표시 토글
- **시뮬레이션 모드**: 하드웨어 없이도 플로우 테스트 가능
- **실시간 모니터링**: 시스템 일러스트 + GPIO 패널 + 메시지 보드

### Phase 2 — IEC 61131-3 ST 에디터
- **ST 코드 자동 생성**: 블록 카드 → ST 코드 실시간 변환
- **양방향 하이라이트**: 블록 클릭 ↔ ST 코드 라인 연동
- **직접 편집**: Monaco Editor (구문 하이라이팅 + 한글 키워드 힌트)
- **역방향 파싱**: ST 코드 수정 → 블록 자동 업데이트
- **PLC 코드 내보내기**: IEC ST / CODESYS V3 / TIA Portal SCL 3종

---

## 문서 구성

| 문서 | 설명 |
|------|------|
| [PRD](prd.md) | 제품 요구사항 정의서 |
| [기술 명세서](tech.md) | 아키텍처 및 구현 상세 |
| [개발 가이드](claude.md) | 코드베이스 설계 원칙 및 Sprint 진행 이력 |
| [작업 목록](todo.md) | Sprint별 완료/진행/예정 항목 |

---

## 빠른 시작

```bash
# 의존성 설치
cd bridge-program
npm install

# Python 가상환경 설정
cd python
python -m venv venv
source venv/bin/activate   # macOS/Linux
pip install -r requirements.txt

# 개발 서버 실행
cd ..
npm run dev
```

---

## 하드웨어 구성

```
민웰 RS-100-24 SMPS (24V/108W)
├── TB6600 → NEMA17 스텝모터 (컨베이어)
└── LM2596 스텝다운 (24V→5V)
    ├── ESP32 DevKit V1 (컨베이어 제어 + 센서)
    ├── ESP32-CAM (비전)
    └── E18-D80NK IR 센서

UFactory Lite 6 (별도 AC 전원)
```

---

*팩토스튜디오 — Sprint 13 완료 (2026-03-19)*
