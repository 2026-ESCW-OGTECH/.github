# 2026-ESW-OGTECH / .github

팀 **OGTECH**의 조직 공통 저장소입니다. 작품 이름은 **SafeAid Kit**입니다.

> 2026 임베디드 소프트웨어 경진대회 자유공모 출품작 —
> 인터넷 없이 작동하는 배낭 장착형 오지 생존 보조 장치.

이 저장소에는 **실행 코드가 없습니다.** 조직 프로필과 팀 공통 문서만 둡니다.
코드는 아래 4개 저장소에 있습니다.

## 저장소 안내

| 저장소 | 무엇이 들어 있나 | 언어 |
|---|---|---|
| [OGTECH-embedded](https://github.com/2026-ESW-OGTECH/OGTECH-embedded) | STM32 상시 계층 펌웨어. CO·온습도·GNSS 센서 허브 | C (STM32 HAL) |
| [OGTECH-backend](https://github.com/2026-ESW-OGTECH/OGTECH-backend) | 안전 분기 엔진과 장치 API 서버 (`:8765`) | Python |
| [OGTECH-frontend](https://github.com/2026-ESW-OGTECH/OGTECH-frontend) | 7인치 키오스크 UI와 **오프라인 지도 엔진** (`:8780`) | Python · JavaScript |
| [OGTECH-llm](https://github.com/2026-ESW-OGTECH/OGTECH-llm) | 온디바이스 음성 파이프라인(STT→분기→LLM→TTS)과 평가 하네스 | Python |

읽는 순서를 하나만 고른다면 **OGTECH-embedded → OGTECH-frontend** 입니다.
이 작품의 핵심 주장인 "Jetson이 꺼져 있어도 감시는 계속된다"가 그 두 저장소에서 증명됩니다.

## 문서

| 문서 | 내용 |
|---|---|
| [profile/README.md](profile/README.md) | 조직 프로필 — 작품 소개, 기능, 기술 스택 |
| [PLAN.md](PLAN.md) | 개발 계획서. 목표, P0 범위, 정량 지표, 일정 |
| [docs/HARDWARE_PLAN.md](docs/HARDWARE_PLAN.md) | 부품 선정과 배선 |
| [docs/ROADMAP.md](docs/ROADMAP.md) | 주차별 일정 |
| [docs/DEMO_SCRIPT.md](docs/DEMO_SCRIPT.md) | 시연 영상 촬영 대본 |
| [docs/SUBMISSION_CHECKLIST.md](docs/SUBMISSION_CHECKLIST.md) | 제출 전 점검 목록 |
| [docs/GITHUB_OPERATIONS.md](docs/GITHUB_OPERATIONS.md) | 브랜치·PR·공개 전환 절차 |

## 시스템 구성

```text
사용자 (음성 · 물리 버튼 · 터치)
  └─ OGTECH-frontend :8780        Chromium 단일 kiosk + 프록시 + 오프라인 지도
       └─ OGTECH-backend :8765    안전 분기 · 검수된 고정 카드 · 장치 API
            ├─ 지도 · 일출몰 · 기압 추정        전부 로컬 계산 (LLM 미관여)
            ├─ OGTECH-embedded → STM32 상시 계층 (CO · 환경 · GNSS · 전원 게이팅)
            └─ OGTECH-llm      → 분류 · 대상 추출 · 카드 문장 다듬기만
```

## 안전 경계

**SafeAid Kit은 구조 요청 수단이 아닙니다.** 조난 **예방**과 **자력 탈출**만 담당하며,
사용자는 별도의 구조 요청 수단(휴대폰·PLB·위성 통신기)을 반드시 함께 지참해야 합니다.
이 한계는 부팅 시 화면에 표시되고 건너뛸 수 없습니다.

- 경로·방위·거리는 지도 엔진이 계산합니다. **LLM 출력 스키마에는 숫자 필드가 없습니다.**
- 생명 관련 응답은 LLM을 거치지 않고 검수된 고정 카드에서 음성으로 직행합니다.
- GPS 미수신을 추정 좌표로 덮지 않습니다. 마지막 확정 좌표와 경과 시간만 표시합니다.
- 기상은 예보가 아니라 기압 추세 기반 국지 추정이며 항상 `추정` 배지가 붙습니다.
- 진단·약물·침습 처치·**야생 동식물 식용 판정**은 어떤 형태로도 하지 않습니다.

## 제3자 라이선스

- 지도 데이터: © OpenStreetMap contributors, ODbL 1.0
- Qwen2.5 1.5B: Apache-2.0 / llama.cpp · whisper.cpp: MIT
- 프로젝트 자체 LICENSE와 추가 의존성 고지는 공개 전환 전 확정합니다.
