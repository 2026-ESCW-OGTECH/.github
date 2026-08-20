# 2026ESWContest_free_TBD

## 🔖 Intro

SafeAid Kit은 **묻기 전에 먼저 말해 주는** 배낭 장착형 오프라인 오지 생존 보조 장치입니다.
인터넷이 끊긴 산악·오지에서 오프라인 지도와 GPS로 현재 위치와 되돌아갈 길을 잃지 않게 하고,
남은 일조 시간을 계산해 되돌아설 시점을 알려 주며, 사용자가 잠든 동안에도 일산화탄소를 감시합니다.

**한 줄: "묻기 전에 먼저 말하고, 사용자가 의식이 없어도 감시는 계속된다."**

## 💡 Inspiration

초보 등산객은 정보가 없어서가 아니라 **판단할 시점을 놓쳐서** 위험에 빠집니다.
해가 지는 속도, 트레일에서 벗어난 거리, 텐트 안의 일산화탄소 — 전부 알 수 있었지만 아무도 먼저 알려 주지 않았던 것들입니다.
지도 앱은 사용자가 꺼내 봐야 하고, 위성 통신기는 이미 조난된 뒤에 쓰는 물건입니다.
"**사용자가 묻기 전에, 그리고 사용자가 잠든 동안에도 대신 감시할 수 없을까?**"라는 질문에서 SafeAid Kit은 시작되었습니다.

## 📸 Overview

[시스템 전체 구성 그림]

<br>

1. 사용자가 배낭에 장치를 장착하고 전원을 켠다.
2. 부팅 시 **"이 장치는 구조 요청 수단이 아니다"** 라는 한계 고지를 건너뛸 수 없게 1회 표시한다.
3. STM32 상시 계층이 GNSS·CO·온습도·기압을 상시 기록하고, Jetson 전원은 MOSFET로 차단해 둔다.
4. 지도 엔진이 현재 위치·측위 정확도·트레일 이탈·복귀 방위를 전부 로컬에서 계산한다.
5. 임계 상황(일조 시간 부족, 트레일 이탈, CO 상승)이면 장치가 **먼저** 부저·진동·음성으로 개입한다.
6. 사용자가 물리 버튼을 눌러 음성으로 질문한다. (터치가 젖어 죽어도 버튼은 살아 있다)
7. whisper.cpp(CPU)가 받아쓰고, 키워드 게이트가 14개 시나리오 라벨 중 하나로 분류한다.
8. 생명 관련 라벨은 **LLM을 건너뛰고** 검수된 고정 카드가 TTS로 직행한다. (경로 B, 목표 2초)
9. 나머지 라벨만 Qwen2.5가 카드를 2~4줄로 다듬어 문장 단위 스트리밍으로 읽어 준다. (경로 A, 목표 3.5초)
10. 검증 실패 또는 지연 초과 시 **재시도 없이** 고정 화면으로 전환하고, 응답이 끝나면 화면은 다시 꺼진다.

## 👀 Main feature

- ### 1️⃣ 오프라인 지도와 위치 로그 역추적

  사전 반입한 지도 데이터(GraphML / OSM XML)를 Jetson 안에서 변환해 Canvas 2D로 직접 렌더링합니다.
  네트워크·타일 서버·외부 SDK가 전혀 없어야 성립하는 요구사항이라 지도 엔진을 자체 구현했습니다.
  **방위·거리·경로는 전부 코드가 계산합니다. LLM은 이 값을 만들지 않고 읽어 주기만 합니다.**

  [오프라인 지도 화면 그림]

  <br>
  <details>
    <summary>지도 · 항법 module 상세설명 ⏬</summary>

  - **map_engine**: GraphML/OSM XML 파싱, 보행로 그래프 구성, 뷰포트 타일 캐시 상한 관리.
  - **gps_service**: Air530 NMEA 파싱, fix 여부·위성 수·정확도 관리. **미수신은 추정으로 덮지 않고 회색으로 유지.**
  - **navigation_service**: 트레일 이탈 거리 판정, 복귀 방위·거리 산출, 다음 웨이포인트 안내.
  - **position_history**: 이동 경로 자동 기록, "3분 전 지점까지 방위 210°, 40 m" 형식의 역추적 응답 생성.
  - **경로 계산**: NetworkX 기반 최단 경로. 외부 라우팅 API를 호출하지 않습니다.
  </details>

- ### 2️⃣ 남은 일조 시간 카운트다운

  위도·경도·날짜만으로 일출·일몰·시민박명을 **오프라인 천문 계산**으로 구합니다.
  최근 보행 속도와 복귀 거리를 비교해 되돌아설 시점을 역산하고, 여유가 부족해지면 적색 경고와 음성으로 개입합니다.

  [남은 일조 시간 카운트다운 화면 그림]

  <br>
  <details>
    <summary>일조 시간 계산 상세설명 ⏬</summary>

  - **solar_service**: 태양 적위·시간각 계산 → 일출 / 일몰 / 시민박명 종료 시각. 통신·API 없이 성립합니다.
  - **잔여 시간 산출**: `civil_end - now`를 분 단위로 유지하고 상단 계기 스트립에 상시 노출합니다.
  - **회귀 판단**: 최근 이동 속도 × 복귀 거리로 소요 시간을 추정해 잔여 일조와 비교합니다.
  - **색 규율**: 여유 충분은 시안(계측 판독값), 임계 도달은 적색(즉시 행동). 앰버는 `추정` 값에만 씁니다.
  - **LLM 미관여**: 이 경로에 모델은 전혀 개입하지 않습니다.
  </details>

- ### 3️⃣ 취침 중 일산화탄소 감시와 이중 전원 계층

  밀폐된 텐트 안의 연소기구는 **수면 중에 초기 증상을 자각할 수 없다**는 점이 가장 위험합니다.
  그래서 감시 주체를 Jetson이 아니라 STM32로 내렸습니다.
  전기화학식 CO 센서를 상시 계층에 직결해, **Jetson 전원이 꺼져 있어도** 부저·진동·스트로브가 울립니다.
  경보는 Jetson 부팅을 기다리지 않습니다.

  [이중 전원 계층 블록도 그림]

  <br>
  <details>
    <summary>전력 상태(State Machine) 상세설명 ⏬</summary>

  - **S1 / 감시 (0.35 W)**
    STM32 단독. CO·온습도·기압 폴링, GNSS 로깅, Jetson 전원 차단(MOSFET OFF). 기본 상태입니다.
  - **S2 / 항법 (0.55 W)**
    이동 감지 시 GNSS 샘플링 주기를 올려 트레일 이탈을 판정합니다. 화면은 여전히 OFF.
  - **S3 / 표시 (13 W)**
    사용자가 화면을 켠 구간. Jetson 전원 인가 → Chromium 키오스크 → 글랜서블 4개 표시.
  - **S4 / 응답 (18 W)**
    음성 질의 처리 구간. STT → 안전 분기 → (필요할 때만) LLM → TTS. 응답 후 즉시 S1/S2로 복귀합니다.
  - **경보 인터록**
    CO 임계 초과는 어떤 상태에서도 최우선입니다. S1에서도 Jetson을 깨우지 않고 즉시 물리 경보를 냅니다.
  </details>

  <br>
  <details>
    <summary>센서 · 전원 구성 상세설명 ⏬</summary>

  - **CO**: 전기화학식(ZE16B-CO). MQ 시리즈는 히터 소비전력이 상시 예산의 2배라 **채택하지 않습니다.**
  - **환경**: 현재 결선은 **DHT11**(온·습도). SHT40 + BMP390(기압)은 미연결 목표 구성입니다.
  - **측위·시각**: Air530 GNSS + MMC5983MA(지자기) + IMU + DS3231 RTC.
  - **물리 출력**: IP67 압전 부저 · 진동 모터 · 고휘도 스트로브 · 물리 버튼 3개.
  - **전원**: 4S Li-ion 21700(330 Wh) + 접이식 태양광 + BQ24650 MPPT.
  </details>

- ### 4️⃣ 제한된 LLM과 즉시 폴백

  LLM은 판단 주체가 아니라 **정해진 계약 안에서 텍스트만 다루는 부품**입니다.
  경로·방위·거리는 **출력 스키마에 숫자 필드 자체가 없어** 환각할 자리가 없습니다.
  생명 관련 질문은 모델에 도달하기 전에 키워드 게이트가 잡아 검수된 고정 카드로 보냅니다.

  [응답 경로 A/B 분기 그림]

  <br>
  <details>
    <summary>LLM 역할 3가지 상세설명 ⏬</summary>

  - **1) 시나리오 분류**: 사용자 발화 → 14개 라벨 중 **1개**. JSON을 생성하지 않고 라벨만 냅니다.
    (`lost, route, daylight, weather, shelter, warmth, water, food, sleep_safety, injury, wildlife, gear, refuse, unknown`)
  - **2) 질의 대상 추출**: `target` × `ask`. **값이 아니라 "무엇을 묻는지"만** 뽑습니다.
  - **3) 카드 맞춤 문장**: 선택된 고정 카드 + 장치 상태 → 2~4줄, 약 96 토큰 상한.
  - **생성 설정**: `temperature = 0`. 취향이 아니라 리허설 20회 동일 출력을 보장하기 위한 재현성 요구입니다.
  - **구조 강제**: JSON Schema 제약(llama.cpp GBNF). 문법 실패가 구조적으로 0이므로 **재시도 단계가 없습니다.**
  </details>

  <br>
  <details>
    <summary>응답 경로 2개 상세설명 ⏬</summary>

  ```text
  경로 B (LLM 우회, 목표 2.0 s 이내)
    lost / daylight / warmth / sleep_safety / injury / refuse
    → 키워드 게이트 → 검수된 고정 카드 → TTS 직행

  경로 A (LLM 다듬기, 목표 3.5 s 이내)
    route / weather / water / food / shelter / wildlife / gear
    → 분류 → 고정 카드 → LLM 2~4줄 → 문장 단위 스트리밍 TTS
  ```

  - **경로 B가 더 빠르고 동시에 더 안전합니다.** 생명 관련 응답에서 모델을 빼는 것은 성능 손해가 아닙니다.
  - **폴백**: 검증 실패 또는 2초 초과 시 재시도 없이 고정 화면으로 전환합니다.
  - **모호하면 키워드가 결정하지 않습니다.** 두 라벨이 동시에 잡히면 LLM 분류로 강등합니다.
    단, `refuse` 키워드가 있으면 다른 매칭을 무시하고 무조건 `refuse`입니다. (안전 편향)
  </details>

  <br>
  <details>
    <summary>STT(whisper.cpp) 실행 구성 상세설명 ⏬</summary>

  Jetson 통합 메모리 환경에서는 74M 파라미터 모델의 **커널 실행 오버헤드가 연산량을 압도**해 CPU가 유리합니다.
  반대로 1.5B LLM은 GPU가 유리합니다. 그래서 실행 타깃을 작업 크기별로 나눴습니다. (STT → CPU, LLM → GPU, TTS → CPU)

  - **`-ac 450`이 지연의 대부분을 설명합니다.** whisper는 입력이 5초여도 인코더를 30초 멜 윈도로 돌립니다.
    이 패딩 연산을 잘라 **7,720 ms → 1,494 ms** `[실측]`.
  - **`-ng`는 선택이 아니라 필수**입니다. 빼면 91 MiB cudaMalloc 실패로 SIGSEGV `[실측]`.
  - **beam search 기각**: 지연 +33%에 출력이 한 글자도 바뀌지 않았습니다 `[실측]`.
  - **판정은 중앙값이 아니라 최댓값**으로 봅니다. 데모 조건이 연속 20회라 한 번의 이상치가 곧 실패입니다.
  </details>

- ### 5️⃣ 서버 및 디스플레이

  #### 1. 서버 (Python, 표준 라이브러리)

  Jetson에 네트워크가 없으므로 **외부 프레임워크·CDN·런타임 의존을 두지 않았습니다.**
  backend는 `8765`, frontend 프록시는 `8780`을 쓰고, Chromium은 `http://127.0.0.1:8780/` 단일 kiosk 창으로 뜹니다.

  [백엔드 모듈 구성 그림]

  <br>
  <details>
    <summary>백엔드 module 상세설명 ⏬</summary>

  - **safeaid_core**: 시나리오 정의, 검수된 고정 카드, 안전 분기, 한계 고지(`DISCLAIMER`) 관리.
  - **API(Handler)**: `ThreadingHTTPServer` 기반 REST 엔드포인트. 상태 조회·명령 수신.
  - **hardware**: STM32와 UART로 명령/ACK를 교환하고 부저·진동·스트로브·LED를 제어.
  - **map / solar / navigation**: 지도·일출몰·항법 계산. **전부 로컬 계산이며 네트워크를 타지 않습니다.**
  - **inventory**: 장비 점검 목록과 런타임 상태 보관(`runtime/`, 커밋하지 않음).
  </details>

  #### 2. Device UI (Web)

  [키오스크 UI 화면 그림]

  <br>
  <details>
    <summary>UI 규칙 상세설명 ⏬</summary>

  1024×600을 7인치에 넣으면 **169.5 PPI, 1 mm = 6.675 px**입니다. 이 화면에서 px는 거짓말을 합니다.

  - **터치 타깃 바닥은 80 px (12.0 mm)**. 흔히 쓰는 56 px는 8.4 mm라 장갑 조작 기준에 미달입니다.
  - **본문 텍스트는 20 px (3.0 mm)** 아래로 내리지 않습니다.
  - **화면은 기본 OFF**입니다. 전력 예산 때문이며, 음성이 1차 출력이고 화면은 2차입니다.
  - **글랜서블 4개**: 화면을 켠 순간 1초 안에 읽혀야 합니다 —
    `현재 좌표 상태` / `남은 일조 시간` / `배터리 잔여 일수` / `트레일 이탈 여부`.
  - **야간 모드는 적색 단색**. 백색광은 암순응을 파괴하고 전력도 더 씁니다.
  - **물리 버튼 3개 필수**: 전원 / 체크포인트 저장 / 음성 질문. 터치가 죽어도 P0는 살아야 합니다.
  </details>

  <br>
  <details>
    <summary>색 규율 상세설명 ⏬</summary>

  | 색 | 의미 | 사용처 |
  | --- | --- | --- |
  | 적색 | 경고 — 즉시 행동 | CO 경보, 일조 시간 부족, 하드 실패 |
  | 앰버 | 주의 — 미검증·성능저하 | DEMO 배지, **기상 추정값**, 대기열 |
  | 녹색 | **실제 센서로 확인됨** | GPS fix, LIVE 계측, 검수 배지 |
  | 시안 | 계측 판독값 | 시각, 좌표, 거리, 카운트 |
  | 회색 | **데이터 없음** | GPS 미수신, 경로 데이터 없음 |

  **회색이 녹색을 빌려 쓰면 안 됩니다.** "모름"이 "정상"처럼 보이는 순간 이 작품의 전제가 무너집니다.
  </details>

## Environment

### Embedded

<img src="https://img.shields.io/badge/STM32H7A3ZI--Q-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white">
<img src="https://img.shields.io/badge/Arduino-008184?style=for-the-badge&logo=arduino&logoColor=white">
<img src="https://img.shields.io/badge/C/C++-00599C?style=for-the-badge&logo=cplusplus&logoColor=white">
<img src="https://img.shields.io/badge/Air530%20GNSS-4B8BBE?style=for-the-badge&logoColor=white">
<img src="https://img.shields.io/badge/ZE16B--CO-8B1E1E?style=for-the-badge&logoColor=white">

### Backend

<img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white">
<img src="https://img.shields.io/badge/Jetson%20Xavier%20NX-76B900?style=for-the-badge&logo=nvidia&logoColor=white">
<img src="https://img.shields.io/badge/http.server-306998?style=for-the-badge&logo=python&logoColor=white">
<img src="https://img.shields.io/badge/NetworkX-2C3E50?style=for-the-badge&logoColor=white">
<img src="https://img.shields.io/badge/pySerial-1F6FEB?style=for-the-badge&logoColor=white">
<img src="https://img.shields.io/badge/SQLite%20FTS5-003B57?style=for-the-badge&logo=sqlite&logoColor=white">

### Frontend

<img src="https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white">
<img src="https://img.shields.io/badge/CSS3-1572B6?style=for-the-badge&logo=css3&logoColor=white">
<img src="https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=111827">
<img src="https://img.shields.io/badge/Canvas%202D-FF6F00?style=for-the-badge&logoColor=white">
<img src="https://img.shields.io/badge/Chromium%20Kiosk-4285F4?style=for-the-badge&logo=googlechrome&logoColor=white">

### LLM / Voice

<img src="https://img.shields.io/badge/Qwen2.5%201.5B%20Q4__K__M-6D28D9?style=for-the-badge&logoColor=white">
<img src="https://img.shields.io/badge/llama.cpp-000000?style=for-the-badge&logoColor=white">
<img src="https://img.shields.io/badge/whisper.cpp-1A7F64?style=for-the-badge&logoColor=white">
<img src="https://img.shields.io/badge/JSON%20Schema%20GBNF-B45309?style=for-the-badge&logoColor=white">

## 🗂 Architecture

[레이어 구조 그림]

```text
사용자 (음성 · 물리 버튼 · 터치)
  └─ OGTECH-frontend :8780          Chromium 단일 kiosk + 프록시
       └─ OGTECH-backend :8765      안전 분기 · 검수된 고정 카드 · 장치 API
            ├─ 지도 엔진 · 일출몰 · Zambretti   전부 로컬 계산 (LLM 미관여)
            ├─ OGTECH-embedded → STM32 상시 계층 (CO · 환경 · GNSS · Jetson 전원 게이팅)
            └─ OGTECH-llm      → 분류 · 대상 추출 · 카드 문장 다듬기만
```

## File Architecture

```
2026ESWContest_자유공모_TBD_SafeAidKit_파일구조
.
├─ .github/                       조직 프로필과 공통 문서
│  ├─ profile/
│  │  └─ README.md
│  ├─ PLAN.md                     개발 계획서
│  └─ docs/
│     ├─ HARDWARE_PLAN.md         부품 선정과 배선
│     ├─ ROADMAP.md               주차별 일정
│     ├─ DEMO_SCRIPT.md           시연 영상 대본
│     ├─ SUBMISSION_CHECKLIST.md  제출 전 점검
│     └─ GITHUB_OPERATIONS.md     브랜치 · PR 절차
│
├─ OGTECH-embedded/
│  └─ Core/
│     ├─ Inc/                     드라이버 인터페이스 4개
│     └─ Src/                     air530_gps · dht11 · ze16b_co · sensor_app · main
│
├─ OGTECH-backend/
│  ├─ app.py                      HTTP 서버 (:8765)
│  ├─ safeaid_core.py             시나리오 · 고정 카드 · 안전 분기
│  ├─ hardware.py                 STM32 UART · 부저 · 진동 · LED
│  ├─ inventory.py                장비 점검 목록
│  └─ requirements.txt
│
├─ OGTECH-frontend/
│  ├─ server.py                   키오스크 서버 (:8780)
│  ├─ MAP/
│  │  ├─ map_engine.py            GraphML · OSM XML → 보행로 그래프
│  │  ├─ gps_service.py           Air530 NMEA 파싱 · fix 판정
│  │  ├─ navigation_service.py    트레일 이탈 · 복귀 방위/거리
│  │  ├─ position_history.py      위치 로그 · 체크포인트 역추적
│  │  ├─ solar_service.py         일출 · 일몰 · 시민박명
│  │  ├─ jetson/                  systemd 유닛 · 전원 게이팅
│  │  ├─ TEST_images/             화면 캡처 (1024×600)
│  │  └─ 시연용/                  ★ 현재 키오스크 UI (video.html)
│  └─ tests/
│
└─ OGTECH-llm/
   ├─ config/                     프롬프트 · 키워드 규칙
   ├─ harness/                    프롬프트 조립 · 스키마 제약
   ├─ eval/                       14 라벨 분류 · refuse 누출 평가
   ├─ runner/
   ├─ results/
   └─ docs2/                      근거 · 실측 기록
```

| 저장소 | 역할 |
| ---- | ---- |
| [OGTECH-frontend](https://github.com/2026-ESCW-OGTECH/OGTECH-frontend) | 7인치 키오스크 UI, 오프라인 지도 렌더링, backend 프록시 |
| [OGTECH-backend](https://github.com/2026-ESCW-OGTECH/OGTECH-backend) | 안전 분기, 검수된 고정 카드, 지도·시간 엔진, 하드웨어 API |
| [OGTECH-embedded](https://github.com/2026-ESCW-OGTECH/OGTECH-embedded) | STM32 상시 전원 관리자 · 센서 허브 · GNSS 로거 펌웨어 |
| [OGTECH-llm](https://github.com/2026-ESCW-OGTECH/OGTECH-llm) | LLM 하네스, 14 라벨 평가, STT/LLM 기준선 측정 |

## Video

[하드웨어 실물 사진]

[시연 영상 썸네일 그림]

<br>

시연 영상은 아래 8개 장면을 **실제 하드웨어에서 연속 20회 성공** 기준으로 촬영합니다.

| # | 장면 | 검증 방법 |
| --- | --- | --- |
| 1 | 오프라인 부팅 + 자가진단 | 네트워크 케이블을 뽑은 상태로 전원 인가 |
| 2 | 오프라인 지도에 현재 위치 + **정확도(±m) 동시 표기** | 실외 실측 |
| 3 | 트레일 이탈 경고 → 진동 + 음성 | 등록 트레일에서 임계 거리 이탈 |
| 4 | 위치 로그 역추적 — "3분 전 지점까지 방위 210°, 40 m" | 실외 왕복 |
| 5 | 남은 일조 시간 카운트다운 → 임계 시 적색 경고 | 시각 조작 후 재현 |
| 6 | **CO 경보 — Jetson 전원을 끈 상태에서 STM32 단독 동작** | 전원 케이블 분리 후 CO 주입 |
| 7 | 음성 질문 → 고정 카드 → 음성 응답 (경로 B, 2초 이내) | 스톱워치 |
| 8 | LLM 강제 종료 후 고정 카드 폴백 + 앱 복구 | `kill -9` |

**6번이 이 작품의 핵심 장면입니다.** 컷 편집 없이 연속 촬영으로 담습니다.

## ⚠️ 안전 경계

**SafeAid Kit은 구조 요청 수단이 아닙니다.** 조난 **예방**과 **자력 탈출**만 담당합니다.
사용자는 별도의 구조 요청 수단(휴대폰, PLB, 위성 통신기)을 반드시 함께 지참해야 하며,
이 한계는 부팅 시 화면에 표시되고 건너뛸 수 없습니다.

- 실족·추락은 예방하지 못합니다.
- 기상 표시는 예보가 아니라 기압 추세 기반 **국지 추정**이며 항상 `추정` 배지가 붙습니다.
- LLM은 경로·방위·거리·처치 절차를 생성하지 않습니다. 출력 스키마에 숫자 필드가 없습니다.
- 진단, 약물명·용량, 침습 처치, **야생 동식물의 식용 가능 여부**는 어떤 형태로도 판정하지 않습니다.
- GPS 미수신을 위치 추정으로 덮지 않습니다. 마지막 확정 좌표와 경과 시간만 표시합니다.
- 모의 값이 하나라도 섞이면 화면의 `DEMO` 배지를 숨기지 않습니다.

## 📄 제3자 라이선스

- 지도 데이터: © OpenStreetMap contributors, ODbL 1.0
- Qwen2.5 1.5B: Apache-2.0
- llama.cpp / whisper.cpp: MIT
- 프로젝트 자체 LICENSE와 추가 의존성 고지는 공개 전환 전 확정합니다.

## Team Member

<br>

| 팀원 | 역할 |
| ---- | ---- |
| **이준형(팀장)** | 시스템 설계 / LLM 하네스 · 안전 계약 / 기획 |
| **TBD** | STM32 상시 계층 / 센서 허브 · 전원 회로 / 기구 |
| **TBD** | 오프라인 지도 엔진 / 키오스크 UI / 시연 영상 |
