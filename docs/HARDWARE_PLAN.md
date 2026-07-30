# 하드웨어 계획

> Notion `HW`, `HW-전력`, `HW-spec`, `HW-구현`, `빠때리`, `마익췍`, `HW - 통신` 기준 동기화: 2026-07-09

SafeAid Kit는 소프트웨어 안내 앱이 아니라, 실제 구급함 내부 상태를 읽고 필요한 구역을 표시하며 기구부를 제어하는 임베디드 시스템으로 시연합니다.

## 최신 기준 요약

- 메인 컴퓨팅은 `Jetson Xavier NX`를 사용합니다.
- MCU는 `STM32F401RET6`(`STM32F401RE` 계열)를 기준으로 합니다.
- 터치 화면은 `Waveshare 7inch HDMI LCD (C)`를 Jetson에 HDMI/USB로 연결합니다.
- 통신 모듈은 `SIM7670G development board`를 STM32 UART와 AT command로 제어합니다.
- 서보는 `DM-S1300MD digital metal-gear servo` 4개를 기준으로 합니다.
- LED는 `2835 SMD LED strip/pieces` 총 10개 배치를 우선 기준으로 합니다.
- 전원은 4S LiFePO4 배터리 기반 `BAT_IN`에서 `5V_COMPUTE`, `5V_LED`, `6V_SERVO`, `3V3_LOGIC`, `BAT_SIM`으로 분리합니다.
- 기구부는 3D 프린팅 판재, 카본 시트지 래핑, DF2020 프레임, 300mm 3단 댐핑 볼 레일을 사용합니다.

## 전체 구조

| 구분 | 역할 | 설계 기준 |
| --- | --- | --- |
| Jetson Xavier NX | 터치 UI, AI/검색 로직, USB 마이크/스피커, 화면 출력 | 5V 입력 안정성 최우선, LED/서보/SIM 부하와 전원 분리 |
| STM32F401RET6 | LED 스위칭, 서보 PWM, 인터럽트 입력, 배터리 전압 감지, SIM AT command 처리 | 3.3V logic rail, GPIO는 제어 신호만 출력 |
| SIM7670G 개발보드 | LTE 통신 | STM32 UART 연결, `VIN 5V~16V` 또는 안정 5V 입력 사용 |
| 7inch HDMI LCD | 사용자 UI와 터치 입력 | Jetson HDMI + USB touch, 5V 전원 |
| USB mic/speaker | 음성 입력/출력 | Jetson USB host 연결, 5V_COMPUTE 예산에 포함 |

Jetson은 SIM 모듈에 직접 AT command를 보내지 않고, STM32에 "통신 상태 조회", "송신 요청", "초기화 요청" 같은 상위 명령을 보냅니다. STM32는 SIM7670G의 AT command sequence, timeout, retry, URC 처리를 담당하고 결과만 Jetson에 전달합니다.

## 기구 구조

| 층 | 구성 | 대표 물품 | 표시/동작 |
| --- | --- | --- | --- |
| 1층 | 큰 서랍 1개, 내부 3파티션 | 붕대, 부목, 파스 등 큰 물품 | 서랍 단위 개방, 파티션별 LED |
| 2층 | 큰 서랍 1개, 내부 6파티션 | 연고, 반창고 등 중형 물품 | 서랍 단위 개방, 파티션별 LED |
| 3층 | 상부 뚜껑 접근 공간, 4~6파티션 후보 | 식염수, 지혈대 등 빠른 접근 물품 | 뚜껑 접근, 파티션별 LED 후보 |

- 1층과 2층은 수평 서랍 구조입니다.
- 3층은 수평 서랍이 아니라 상부 뚜껑을 열어 접근하는 구조입니다.
- Notion 구현안에는 서보 래치, 압축 스프링, 유연 노치로 자동 인출/재잠금을 구현하는 구조가 정리되어 있습니다.
- Notion 본문에는 랙 앤 피니언과 3단 댐핑 볼 레일을 이용한 자동 개폐 방향도 포함되어 있으므로, 실제 제작 전에는 두 방식 중 제작 난이도와 서보 부하를 비교해야 합니다.
- 현재 LED 배치 기준은 1층 3개, 2층 6개, 3층 1개로 총 10개입니다. 3층을 4~6파티션으로 확정하면 LED/센서 수와 펌웨어 `CELL_IDS`를 다시 늘려야 합니다.

## 외장/프레임

| 항목 | 기준 |
| --- | --- |
| 바디 | 3D 프린팅 판재, 내부 채움 조절로 경량화, 배선로와 인서트 홀 사전 설계 |
| 스킨 | 외부 카본 시트지 래핑, 3D 프린팅 결 커버 및 시제품 마감 |
| 프레임 | DF2020, 탭가공 필수, 최소 60mm 기준 |
| 모서리 결합 | 코너블락 사용 |
| 기동부 | 300mm 3단 댐핑 볼 레일 4개 |
| 제어부 | 후면 고정 벽면에 MCU/전원부 배치 |
| 배선 | 슬림형 FFC 케이블과 3D 프린팅 내장 가이드 채널로 유격과 걸림 최소화 |
| 잠금 | 스프링 내장 매미 2개를 사용해 suitcase식 잠금 구조 구성 |

## 확정/예정 BOM

| 구분 | 부품 | 수량 | 메모 |
| --- | --- | --- | --- |
| 메인 컴퓨팅 | Jetson Xavier NX | 1 | 10W~15W 모드, 5V 입력 안정성 필요 |
| 하위 제어기 | STM32F401RET6 | 1 | 3.3V logic, PWM/UART/ADC/interrupt 담당 |
| 디스플레이 | Waveshare 7inch HDMI LCD (C) | 1 | HDMI 화면, USB touch, 5V 전원 |
| 통신 | SIM7670G development board | 1 | STM32 UART, AT command, VIN 전원 |
| 서보 | DM-S1300MD digital metal-gear servo | 4 | 6V nominal, stall 2.5A/개 |
| LED | 2835 SMD LED strip/pieces | 10개 예정 | 5V, 6W/m, 총 길이 기준으로 전류 계산 |
| 마이크 | Adafruit Mini USB Microphone Product ID 3367 | 1 | Jetson USB |
| 스피커 | Adafruit Mini External USB Stereo Speaker Product ID 3369 | 1 | Jetson USB, 2 x 2W |
| 레일 | 300mm 3단 댐핑 볼 레일 | 4 | 서랍 기동부 |
| 프레임 | DF2020 + 코너블락 | 필요 수량 | 탭가공 필수 |
| 배터리 | 4S LiFePO4 배터리팩 후보 | 1 | `BAT_IN` 10V~14.6V 기준 |
| 보호/전원 | BMS, fuse/eFuse, load switch, TVS, DC/DC converter | 필요 수량 | 각 rail 분리와 단락/과전류 보호 |
| 개발/디버그 | ST-LINK | 1 | STM32 flashing/debug |

## 전원 구조

권장 구조:

```text
4S LiFePO4 Battery / main input
  -> BMS / fuse / power switch / reverse polarity protection
  -> BAT_IN: 10V~14.6V main path
  -> 5V_COMPUTE: Jetson, LCD, USB mic/speaker
  -> 5V_LED: LED strip/pieces
  -> 6V_SERVO: DM-S1300MD servo x4
  -> 3V3_LOGIC: STM32, level shifter, low-current logic
  -> BAT_SIM: SIM7670G VIN branch with fuse/load switch/filter
```

| Rail | 연결 대상 | 권장 정격 |
| --- | --- | --- |
| `BAT_IN` | BMS, fuse, main switch 뒤 PCB 입력 | 4S LiFePO4 기준 10V~14.6V, main path 20A급 여유 |
| `5V_COMPUTE` | Jetson, LCD, USB mic/speaker | 최소 5V 6A, LCD/USB 여유 포함 5V 8A 권장 |
| `5V_LED` | LED strip/pieces | 실제 LED 길이에 따라 산정, 3m 예시 3.6A |
| `6V_SERVO` | DM-S1300MD x4 | peak 12A~15A |
| `3V3_LOGIC` | STM32, level shifter, logic | 0.5A 기본, 확장 여유 1A |
| `BAT_SIM` | SIM7670G VIN | BAT_IN 직접 분기 가능, fuse/load switch/filter 필수 |

LED 전력은 개수가 아니라 총 길이로 계산합니다.

```text
LED_power_W = 6W/m x total_LED_length_m
LED_current_A = LED_power_W / 5V
```

예시:

```text
30cm 조각 10개 = 3m
LED_power = 6W/m x 3m = 18W
LED_current = 18W / 5V = 3.6A
```

## 서보 보호 기준

- 서보 V+는 STM32/Nucleo 보드에서 절대 공급하지 않습니다.
- STM32는 PWM 신호만 출력합니다.
- `6V_SERVO` 입력에는 fuse 또는 eFuse/load switch를 둡니다.
- `6V_SERVO` rail에는 rail 전압에 맞는 단방향 TVS diode를 검토합니다.
- servo rail 근처에는 `2200uF~4700uF` bulk capacitor와 `0.1uF` ceramic capacitor를 둡니다.
- 각 servo connector 근처에는 `470uF~1000uF` capacitor와 `100nF` ceramic capacitor를 둡니다.
- STM32 PWM line에는 `100~330ohm` series resistor와 `10k` pulldown을 둡니다.
- servo power return과 STM32 signal GND는 star point에서 합류시킵니다.
- 대기 전력 절감을 위해 servo rail은 high-side load switch 또는 eFuse로 필요할 때만 켜는 구조를 권장합니다.

## 앱/펌웨어 연동

현재 앱은 `smartaid-kit/hardware.py`의 `KitController`로 모의 모드와 STM32 모드를 같은 API로 다룹니다.

환경변수:

```powershell
$env:SAFEAID_KIT_MODE="stm32"
$env:SAFEAID_STM32_PORT="COM5"
python app.py
```

현재 Serial 명령:

```text
OPEN_LAYER 2
SET_CELL_LED 2-3
CLOSE_ALL
READ_STOCK
GET_BATTERY
```

추가 예정 명령:

```text
GET_COMMS_STATUS
SIM_SEND <payload>
SIM_RESET
```

SIM7670G 제어 명령은 STM32 펌웨어에서 AT command sequence가 안정화된 뒤 추가합니다.

## Bring-up 순서

1. Jetson만 전원 연결 후 5V 입력 안정성을 확인합니다.
2. HDMI LCD와 USB touch를 연결해 해상도와 touch 동작을 확인합니다.
3. USB microphone/speaker의 audio input/output을 확인합니다.
4. STM32를 3.3V logic rail에서 별도 bring-up합니다.
5. Jetson-STM32 UART 통신을 확인합니다.
6. STM32-SIM7670G UART 연결 후 기본 AT command 응답을 확인합니다.
7. SIM7670G 초기화, network 상태 조회, timeout/retry/URC 처리를 확인합니다.
8. 서보 1개를 6V rail에서 먼저 테스트합니다.
9. 서보 4개 연결 후 동시 구동과 전압 강하를 확인합니다.
10. LED를 current-limited supply에서 먼저 테스트합니다.
11. SIM7670G를 전체 시스템 전원 구성에 포함해 5V/BAT_SIM rail의 순간 전류 대응을 확인합니다.

## 안전 경계

- 일반의약품은 보관과 위치 안내 대상입니다.
- 앱은 "있습니다/없습니다", "위치는 n층 n-n구역입니다", "열어 드릴까요?"까지만 말합니다.
- 복용량, 약 추천, 처치 추천, 치료 결과 판단은 생성하지 않습니다.
- 유통기한이 지난 물품은 사용 지시 없이 교체 필요만 표시합니다.
- 재고 센서가 검증하는 것은 "물품 정체성"이 아니라 "해당 슬롯/구역의 상태"입니다. RFID, QR, 비전 검증을 추가하지 않는 한 UI 문구는 "올바른 물품"보다 "올바른 구역"으로 표현합니다.

## 미확정 항목

- 3층 파티션 수: Notion 본문에는 4~6개 후보가 있고, 현재 전력/LED 계산은 3층 LED 1개 기준입니다.
- 서랍 구동 방식: 서보 래치 + 스프링 방식과 랙 앤 피니언 방식 중 실제 제작 난이도/서보 부하를 비교해야 합니다.
- LED 실제 총 길이: `5V_LED` 정격은 최종 길이 확정 후 다시 계산해야 합니다.
- 배터리와 DC/DC converter 실제 모델: 데이터시트 확인 후 `BAT_IN`, `5V_COMPUTE`, `5V_LED`, `6V_SERVO` 정격을 확정해야 합니다.
- SIM7670G 최종 전원 경로: 개발보드의 `VIN` 입력과 `VBAT` 입력을 동시에 공급하지 않도록 실제 보드 회로를 확인해야 합니다.
