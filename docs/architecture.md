# 아키텍처

KS X 3267 (RS485 Modbus RTU) 기반 스마트팜 게이트웨이 · HMI · 백엔드 · 프런트엔드 전(全) 스택의 큰 그림.

> 이 문서는 원본 사설 저장소 (`seriallink-web`) 의 공개용 스냅샷입니다.
> 운영 인프라의 호스트/도메인/자격증명 등 민감 값은 익명 플레이스홀더로 대체하고,
> 표준 스택 (KS X 3267 / 3286 / 3269) 만을 서술합니다. 사설로 병행 운영되던
> 초기 세대 (레거시) 스택은 이 공개본의 범위 밖입니다.

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [큰 그림 아키텍처](#2-큰-그림-아키텍처)
3. [표준 스택 (KS X 3267 / 3286 / 3269)](#3-표준-스택-ks-x-3267--3286--3269)
4. [컴포넌트 상세](#4-컴포넌트-상세)
5. [데이터 모델](#5-데이터-모델)
6. [KS X 3267 검증 상태](#6-ks-x-3267-검증-상태)
7. [KS X 3286 vendor spec 통합](#7-ks-x-3286-vendor-spec-통합)
8. [디스커버리 · 라이프사이클 감사](#8-디스커버리--라이프사이클-감사)
9. [배포](#9-배포)
10. [부록: 진행 상태 요약](#10-부록-진행-상태-요약)

---

## 1. 프로젝트 개요

**seriallink** 는 노지·시설 원예 규모의 스마트팜을 대상으로 하는
**KS X 3267 준수 게이트웨이 · 원격제어 · 데이터 파이프라인** 프로젝트입니다.
"현장의 RS485 센서/구동기 한 세트" 를 "클라우드의 대시보드/알림 한 화면" 으로
잇는 것이 목표이며, 그 사이 계층 전부를 하나의 monorepo 로 관리합니다.

### 1.1 구성

시스템은 다음 다섯 계층으로 구성됩니다.

| 계층 | 역할 | 주요 기술 |
|---|---|---|
| **Master** (게이트웨이) | RS485 폴링, MQTT 업링크, 슬레이브 디스커버리·감사 | ESP32-S3, ESP-IDF, FreeRTOS |
| **Slave** (센서/구동기) | KS X 3267 표준 노드 (soil / ec / co2 / dummy 등, 그리고 구동기) | ESP32-C3, ESP-IDF |
| **HMI** (현장 터미널) | 로컬 카드 UI, 30+ 슬롯, 2-tier stale 관리 | JC-ESP32-P4-M3, LVGL |
| **Backend** (서버) | MQTT ingest, 시계열 저장, WebSocket, VAPID push | Rust (axum), TimescaleDB, mosquitto |
| **Frontend** (웹) | 실시간 카드, 감사·설정 UI, 표준 검증 위젯 | SvelteKit (static) |

### 1.2 설계 원칙

- **표준 우선.** 신규 노드 프로토콜은 KS X 3267 default map 을 기본으로 하고,
  비표준 벤더 노드는 KS X 3286 자동등록 절차로 편입한다. 임의 주소·필드 확장은 지양한다.
- **버스는 라벨로 통일.** PCB · 펌웨어 · MQTT · UI 어느 계층에서든 물리 커넥터의
  버스 번호가 동일하게 보이도록 관례를 고정 (자세한 관례는 부록 참조).
- **디스커버리는 마스터가 주도.** 슬레이브는 자기 주소를 강제하지 않으며,
  마스터의 sentinel 스캔 + 3-tier fallback 으로 배정된다.
- **관측 가능성 우선.** 모든 슬레이브 lifecycle 이벤트 (등장/이동/사라짐/재배정) 는
  `device_audit` 로 영속화되며 UI 에서 시각적으로 확인 가능하다.
- **압축된 시계열.** 활성 chunk 만 read-write, 오래된 chunk 는 TimescaleDB 압축.
  Partial delete 는 async 백그라운드 잡으로 처리.

### 1.3 리포지토리 레이아웃 (개념)

```
seriallink/
├─ firmware/
│  ├─ master/            ESP32-S3 게이트웨이 (RS485 + MQTT)
│  ├─ slave-c3-sensor/   soil / ec / co2 / vendor / dummy variant
│  ├─ slave-c3-actuator/ 구동기 노드
│  └─ hmi-p4/            JC-ESP32-P4-M3 + LVGL
├─ server/
│  ├─ backend/           Rust (axum + tokio) MQTT ingest + REST + WS
│  ├─ frontend/          SvelteKit static build
│  └─ nginx-system-sites/ 리버스 프록시 conf (monorepo source-of-truth)
├─ scripts/              배포 · 스캔 · 검증 helper
└─ docs/                 본 문서 포함, 표준 검증 보고서 등
```

---

## 2. 큰 그림 아키텍처

```
┌────────────────────────────────────────────────────────────────────┐
│                        Cloud / 사용자                               │
│                                                                     │
│   ┌───────────────┐              ┌──────────────────┐               │
│   │  Web 브라우저 │              │  모바일 (APK)    │               │
│   └───────┬───────┘              └────────┬─────────┘               │
└───────────┼───────────────────────────────┼─────────────────────────┘
            │ HTTPS                          │ HTTPS
            ▼                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                     VPS  (<vps-host>)                              │
│                                                                     │
│   ┌────────────────────────────────────────────────────────────┐   │
│   │  nginx  (reverse proxy · WebSocket upgrade · /wiki 정적)  │   │
│   │  <service-host>  →  127.0.0.1:8000                         │   │
│   └────────────┬───────────────────────────────────────────────┘   │
│                │                                                    │
│                ▼                                                    │
│   ┌────────────────────────────────────────────────────────────┐   │
│   │  seriallink-backend-dev  (Docker · host network)           │   │
│   │  ┌──────────────────────────────────────────────────────┐  │   │
│   │  │  Rust axum                                            │  │   │
│   │  │  ├─ MQTT ingest (rumqttc, TLS)                        │  │   │
│   │  │  ├─ REST + WebSocket (tokio-tungstenite)              │  │   │
│   │  │  ├─ VAPID push (Web Push)                             │  │   │
│   │  │  ├─ async purge worker · sensor prune                 │  │   │
│   │  │  └─ static SvelteKit build 서빙                       │  │   │
│   │  └──────────────────────────────────────────────────────┘  │   │
│   └────────────────────────┬───────────────────────────────────┘   │
│                            │                                        │
│   ┌────────────────────────▼───────────────────────────────────┐   │
│   │  PostgreSQL 14 + TimescaleDB                               │   │
│   │  ├─ migration_dev.sensor_data_v2  (hypertable, 압축)       │   │
│   │  ├─ migration_dev.devices          (사용자 메타)           │   │
│   │  ├─ migration_dev.devices_dim      (자동등록 dim)          │   │
│   │  ├─ migration_dev.metrics_dim      (metric key ↔ id)       │   │
│   │  └─ migration_dev.device_audit     (lifecycle 이벤트)      │   │
│   │                                                             │   │
│   │  mosquitto MQTT broker  (TLS, ACL)                          │   │
│   └────────────────────────┬───────────────────────────────────┘   │
└────────────────────────────┼───────────────────────────────────────┘
                             │ MQTT (TLS)
                             ▼
┌────────────────────────────────────────────────────────────────────┐
│                        현장 (Field site)                           │
│                                                                     │
│   ┌──────────────────────────────────────────────────────────┐    │
│   │  HMI                                                      │    │
│   │  JC-ESP32-P4-M3  (LVGL, 800×1280)                         │    │
│   │  ├─ 30+ card slot, 2-tier stale (hard/soft)               │    │
│   │  └─ UART link ─────────────────────────┐                  │    │
│   └────────────────────────────────────────┼──────────────────┘    │
│                                            │                        │
│   ┌────────────────────────────────────────▼──────────────────┐    │
│   │  Master  (ESP32-S3, N8R8, 8MB flash)                      │    │
│   │  ├─ Wi-Fi + MQTT (TLS)                                    │    │
│   │  ├─ Discovery: GPIO per-port + sentinel reg 250           │    │
│   │  ├─ Lifecycle audit (added / moved / removed / reassign)  │    │
│   │  └─ 2 × RS485 bus  (BUS#1, BUS#2)                         │    │
│   └──────────┬────────────────────────────┬───────────────────┘    │
│              │ RS485 9600 bps 8N1          │                         │
│              │ KS X 3267 Modbus RTU        │                         │
│              ▼                              ▼                         │
│    ┌───────────────────┐         ┌───────────────────┐              │
│    │ BUS#1             │         │ BUS#2             │              │
│    │ addr 1..N         │         │ addr 1..M         │              │
│    │ ┌──────┐ ┌──────┐ │         │ ┌──────┐ ┌──────┐ │              │
│    │ │ soil │ │ co2  │ │         │ │ act  │ │vendor│ │              │
│    │ └──────┘ └──────┘ │         │ └──────┘ └──────┘ │              │
│    └───────────────────┘         └───────────────────┘              │
│                                                                     │
│    Slave  ESP32-C3 (sensor variant: soil / ec / co2 / dummy)        │
│           ESP32-C3 (actuator variant)                               │
│           Vendor node (KS X 3286 자동등록, non-default map)         │
└────────────────────────────────────────────────────────────────────┘
```

- `<vps-host>`, `<service-host>` 는 익명화된 플레이스홀더입니다.
- Backend/frontend 는 동일 컨테이너·동일 프로세스에서 서빙됩니다 (SvelteKit static build 를 axum 이 static route 로).
- MQTT broker (mosquitto) 는 host 상의 별도 프로세스이며 backend 는 클라이언트로만 참여합니다.

---

## 3. 표준 스택 (KS X 3267 / 3286 / 3269)

이 프로젝트가 준수하는 세 개의 KS X 스마트팜 표준입니다.

| 표준 | 범위 | 이 프로젝트에서의 위치 |
|---|---|---|
| **KS X 3267** | RS485 Modbus RTU 인터페이스. 센서/구동기/제어기 사이 전송 계층 및 default register map. | **핵심 프로토콜.** Master ↔ Slave 통신 전 구간. |
| **KS X 3286** | 자동등록 노드. 3267 default map 에 없는 벤더 노드가 자기 spec 을 광고·등록하는 절차. | 벤더/비표준 노드 편입 경로. Master 확장 완료. |
| **KS X 3269** | 센서 메타데이터. unit, range, precision 등의 명명 규약. | metric key 명명·UI 라벨링 참조. |

### 3.1 프로토콜 요약 (KS X 3267)

- **물리 계층**: RS485 half-duplex, 9600 bps, 8N1 (본 프로젝트 기준값. 표준은 다중 속도 허용).
- **링크 계층**: Modbus RTU. Function code 0x03 (Read Holding), 0x06 (Write Single), 0x10 (Write Multiple).
- **응용 계층**:
  - 노드 종류별 default register map 이 규정됨 (센서 종별로 상이).
  - 구동기는 cmd 블록 · status 블록의 PDU 필드 순서가 규정됨.
  - S-1 (센서 값), N-1..N-5 (노드 메타), SW/OC 계열 (구동기 스위치/카운터).
- **주소 규약**: 1..247. 브로드캐스트 0. 본 프로젝트는 마스터 주도 디스커버리로
  동적 배정하되 표준 §5.1 범위 내에서 동작.

각 명령의 판정 기준과 PDU 필드 순서 등의 상세는 별도 문서로 분리되어 있습니다
(`verification.md` 참조).

### 3.2 표준 · 사설 확장의 경계

디스커버리 (마스터가 슬레이브 addr 을 배정하고 사용자 alias 를 관리하는 절차) 는
표준 §5.1 의 초기화·주소 배정 흐름 안에 포함시켰으며, 표준 밖 확장은
`docs/discovery.md` 에 별도 명시합니다. 표준 부합/불부합 여부가 애매한 지점은
검증 보고서 (`docs/verification.md`) 에서 명령별로 판정합니다.

---

## 4. 컴포넌트 상세

### 4.1 Master (ESP32-S3)

RS485 물리 버스 2 채널을 각각 폴링하고, 결과를 MQTT 로 업링크하는 게이트웨이입니다.

| 항목 | 값 |
|---|---|
| MCU | ESP32-S3 (N8R8, 8MB flash, 8MB PSRAM) |
| 프레임워크 | ESP-IDF (FreeRTOS) |
| RS485 | 2 × UART (BUS#1: UART2, BUS#2: UART1), 9600 bps, DE/RE 결선 |
| Wi-Fi | 2.4GHz STA (WPA2), 자동 재접속 |
| MQTT | TLS, keepalive · reconnect 관리 |
| HMI 링크 | UART0 remap → JC-ESP32-P4-M3 |
| 부트/플래시 | CP210x, BOOT 버튼 홀드 필수 (IO0 미배선), DTR/RTS swap |
| 주요 태스크 | poll_bus × 2, mqtt_up, discovery, audit, hmi_bridge |

**책임 (Responsibilities):**

1. **폴링.** BUS#1 / BUS#2 상의 등록된 슬레이브에 대해 KS X 3267 read 명령을
   주기적으로 발행하고 결과를 정규화된 metric 형태로 만든다.
2. **디스커버리.** 새 슬레이브를 감지하기 위해 GPIO per-port 시퀀스 + sentinel reg 250
   스캔을 수행한다. 3-tier addr fallback (요청 addr → 후보 addr → 스캔 배정).
3. **감사.** 슬레이브의 lifecycle 이벤트 (added / moved / removed / reassign) 를
   생성하여 MQTT 로 backend 로 전송.
4. **업링크.** 정규화된 sensor snapshot, actuator status, audit event 를
   MQTT topic 규약에 맞춰 발행. QoS · payload schema 는 backend 와 계약.
5. **HMI 브리지.** UART0 remap 을 통해 P4 HMI 로 현재 카드 상태를 미러링.

**KS X 3286 통합.** vendor node 를 감지하면 마스터는 자기가 알지 못하는 register map 을
NVS 캐시 (또는 backend 에서 fetch) 를 통해 확보하고 정상 slave 처럼 취급한다.
상세는 `docs/vendor-spec.md`.

### 4.2 Slave-c3 sensor

ESP32-C3 기반 표준 센서 노드. 하나의 코드베이스에서 sdkconfig 로 종별을 스위치합니다.

| Variant | 대상 metric (KS X 3267 device code) |
|---|---|
| `soil` | 토양 온도 · 함수율 (soil_moisture) |
| `ec` | 토양 EC (device code 12) |
| `ph` | 토양 pH (device code 16) |
| `co2` | 대기 CO2, 온·습도 |
| `vendor` | KS X 3286 자동등록 (spec 광고) |
| `dummy` | 하드웨어 없이 값 생성 (검증·CI 용) |

- **flash**: `flash-slave.sh` 로 variant · target addr · bus 를 인자로 전달하여
  sdkconfig defaults 를 스위치·빌드·flash 한 뒤 확인 절차까지 자동화.
- **원격 addr 변경**: master 를 경유한 KS X 3267 write 명령으로 slave 의 addr 을
  런타임 변경 가능. 표준 범위 밖 임의 주소 변경은 허용하지 않음.
- **link**: RS485 half-duplex, 마스터의 폴링에 응답만 함 (unsolicited push 없음).

### 4.3 Slave-c3 actuator

밸브 · 릴레이 · 모터 driver 등 구동기 노드. sensor 와 동일 SoC (ESP32-C3) 를 씁니다.

- **cmd 블록**: master 가 write 하는 채널별 명령 (open/close/duty/level 등).
- **status 블록**: master 가 read 하는 실제 상태 · 카운터.
- 두 블록의 PDU 필드 순서는 KS X 3267 규정을 따르며, 초기 구현에서 상이했던
  필드 순서를 표준에 맞춰 수정한 이력이 있음 (검증 보고서의 "비적합 #1 fix").

### 4.4 HMI (JC-ESP32-P4-M3, LVGL)

현장에 설치되는 8" 급 터미널. 800×1280 RGB IPS + capacitive touch.

- **UI 프레임워크**: LVGL 9.x. 별도 웹뷰 없음 (네이티브 렌더).
- **카드 슬롯**: 30 + 슬롯. 각 슬롯은 KS X 3267 device code 별 카드 타입에 매핑.
- **2-tier stale 관리**:
  - **soft stale**: 응답 지연으로 최근 값이 오래됨 → 시각 표시만 (회색톤).
  - **hard stale**: 임계 시간 초과 → 카드 자체를 회수하고 다른 노드에 재할당 가능.
- **차트 성능**: `lv_chart` point cap ~ 120, 다운샘플, `done()` 에서만 refresh
  (IDLE0 starve 를 피하기 위한 P4 특화 튜닝).
- **링크**: 마스터와 UART bridge. HMI 는 값을 생성하지 않고 미러링·조작만.

#### 4.4.1 rapid click 파이프라인 (씹힘/회색화 근절)

사용자가 카드 12 개를 rapid 하게 누르면 두 증상이 났습니다: (1) 일부 클릭이
슬레이브까지 안 감 (씹힘), (2) 카드가 회색으로 렌더 (soft stale). 원인 3 개,
fix 는 파이프라인 재설계:

| 층 | 원인 | fix |
|---|---|---|
| HMI 터치 이벤트 | LVGL dispatch 안 UART TX **blocking** → 다음 touch 대기 | UART tx buffer 를 0 → 4 KiB (async write) |
| HMI optimistic UI | pending window 2 s < snap 주기 5 s → 만료 후 stale snap 이 UI 롤백 | pending window 2 s → 6 s |
| Master `link_p4` | rx_task 안 `mb_master_write_multiple` + poll + snap 이 250 ms/click blocking → 12 click 3 s 점유 → RX buffer overflow line drop, snap gap 17 s+ | **2 단 offload**: rx→cmd_queue→cmd_task(write)→kick_queue→kick_task(30 ms drain dedup + poll + snap) |

**최종 흐름**: `HMI UART → rx_task(parse+enqueue, µs) → cmd_queue → cmd_task
(순차 write) → kick_queue → kick_task(dedup+poll+snap)`. rx_task 는 항상
µs 단위 nonblocking → RX line drop 완전 소멸.

**Stack overflow 사고**: 초기 kick_task stack 3 KiB 로 잡았다가
`send_node_snapshot` (buf[768] + JSON build) 실행 시 panic → reboot loop →
click 12 개 중 6 개만 도달 (재부팅 dead-zone 사이 drop). `addr2line
vApplicationStackOverflowHook` 로 5 분 만에 원인 확정 → stack 6 KiB 로 확장.

**partial-fail 안전 gate 3 건** (HMI): (i) `update_actuator` 상 c non-null
확인 즉시 `last_update_ms` 선갱신 (subfield NULL 이어도 stale timer fresh),
(ii) `stale_check` 상 partial 카드는 render skip, (iii) `build_card` 상
subfield alloc 실패 시 이미 만든 obj destroy + memset rollback (좀비 카드
청소).

#### 4.4.2 표준 준수 강화 (2026-07 심사 대비 감사 8건)

파이프라인 리팩터 후 KS X 3267 심사 관점 gap 감사에서 8건 발견·수정:

| # | 심각도 | 항목 | fix |
|---|---|---|---|
| F-1 | CRITICAL | Master snapshot JSON buffer 트루케이션 → 24ch 노드 후반 채널 (SW#13~16, OC#5~8) 이 HMI UI 카드 자체 미노출 (header 130B + entry 60B × ch, 768B 로는 10~12ch 만 담김) | buffer 768 → 2048 B 확대 (HMI 파서 라인 상한 2400 B 이미 수용) |
| F-2 | CRITICAL | Slave CONTROL(§6.1.4.1) 처리 불완전 — code 값 (LOCAL/REMOTE/MANUAL) 무시하고 OPID 만 echo → LOCAL 모드에서도 원격 SW 처리 = 표준 위반 | `s_control_code` 상태 저장 + 채널 status override (SW→299, OC→399) + NPN write skip |
| F-3 | MEDIUM | 외부 NPN relay downstream 실패 시 slave 는 status = SW_ON 유지 → 상태 거짓 반영 (UI "ON" 인데 실 relay OFF) | 실패 시 status = 900 (VENDOR_MIN, "NPN_UNREACHABLE") + node_status = ERROR |
| F-4 | MEDIUM | MQTT `mb_write` handler 는 sync + kick 미호출 → verify widget 이 status 반영을 5 s poll 까지 대기 → flakiness | mb_write/mb_write1 성공 시 50 ms delay + `poller_kick_one` (HMI 경로와 반영 latency 동조) |
| F-5 | MEDIUM | HMI 카드 UI 로 발행 가능 op = 4종 (SW_OFF/ON, OC_OPEN/CLOSE) 만.  TIMED / DIRECTIONAL / SET_POSITION / SET_CONFIG UI 부재 | 나머지 op 코드는 verify widget → backend MQTT `mb_write` 로 전수 검증 (문서 명시) |
| F-6 | MEDIUM | Master cmd_queue depth 32 silent drop → 심사 rapid burst (24ch × 2 = 48 cmd) 초과분 drop, HMI optimistic UI 는 "발행됨" 표시 → 실 wire 미전송 = status 불일치 | depth 128 + full 시 200 ms blocking backpressure (drop 대신) |
| F-7 | MINOR | `status_for_op` 미지원 op → KS_STS_READY 조용히 회귀. 심사원 이상값(op=999) 테스트 시 판독 곤란 | `default: KS_STS_ERROR` (§B.2) |
| F-8 | MINOR | Discovery sentinel reg 250 이 sensor 디폴트 맵 상 ch17 value LO 위치임을 문서 명시 | 검증 보고서 §4.1 에 "ephemeral discovery 모드 전용, fixed addr 모드에선 표준대로 사용" 명시 |

**감사 접근**: 우리 자체 verify widget (opid echo 만 검사) 이 놓치는 gap 을
심사원 관점 (LOCAL 모드 SW cmd 응답, 24ch 조작 시연, NPN 물리 실패 등) 으로
재감사.  가장 중요한 F-2 는 "자체 PASS 였지만 표준 위반" 사례 — verify
로직 자체의 한계를 문서화해 두는 게 재발 방지.

#### 4.4.3 외부 표준 slave 자동 호환 (bi-directional interop)

우리 스택은 **두 방향 호환** 을 목표로 한다:
- 우리 sensor slave → 외부 표준 마스터에 연결
- 외부 표준 slave → 우리 마스터에 연결 (자동 UI 대응)

두 번째 방향의 자동 대응 범위를 감사·강화한 결과:

| 항목 | 이전 | 강화 후 |
|---|---|---|
| Sensor devcode 라벨/단위 | 19종 중 10종만 (온도/습도/이슬점/유량/일사/CO2/EC/토양수분/pH/지온) | **§A.1.2 19종 전수** (감우/강우량/풍속/풍향/전압/광양자/토양장력/무게/누적유량 추가) + 양액기 (§A.2.8) |
| Status 코드 표기 | 4종 (READY/ON/OPENING/CLOSING) 만, 나머지는 `st=N` raw | **§B.1~B.5 전수** — ERROR/BUSY/V_ERR/I_ERR/T_ERR/FUSE_ERR/REPLACE/CALIB/CHECK/TIMED_ON/USER_CTRL/MANUAL/PREPARE/SUPPLY/STOP + vendor 900~999 |
| Registry scan 상한 | hardcode addr 1~16 | Kconfig `SFC_REGISTRY_SCAN_MAX` 노출 (default 16, range 1~247). 표준 시험 시 상한 addr 검증 대응 |
| 회사코드=0 default map slave | 자동 대응 (registry probe → devcode 매핑 → UI 카드 자동 생성) | 그대로. 회사코드 non-zero (vendor) + KS X 3286 spec 미제공은 스코프 밖 (§6.5) |

즉 회사코드=0 표준 slave 를 우리 마스터에 붙이면 값 스케일링 (IEEE-754 f32 word-LE) / SW/OC 카드 자동 생성 / 라벨·단위·status 정상 표기가 자동으로 완료된다.

#### 4.4.4 물리 스위치 vs HMI (표준 옵션)

KS X 3267 §6.1.4.1 CONTROL 은 **mode 존재만 요구** (LOCAL/REMOTE/MANUAL). 물리 스위치 자체는 강제 X.  심사는 `mb_write reg=501 values=[<code>, opid]` → 노드가 status 를 SW_USER_CONTROL(299) / MANUAL_CONTROL(399) 로 override 하는지만 검증.

**실용 관점**: 기존 물리 스위치의 3가지 가치를 우리 상황에 대입:

| 가치 | 우리 상황 |
|---|---|
| 통신 장애 fallback | ⚠ 불필요 — HMI 가 HMI 로컬 카드 UI.  네트워크/backend 다운돼도 HMI ↔ master ↔ slave UART/RS485 로 그대로 조작 |
| 긴급 정지 | ⚠ 낮음 — HMI 화면 조작 반응 시간 = 물리 버튼과 큰 차이 X |
| 심사 MANUAL 시연 | 🟨 모호 — verify widget 로 `mode=3` 발행 시뮬레이션 대체 가능 |

**결론**: 심사 지침이 "물리 조작 시연" 을 명시 요구하지 않으면 skip. 명시 시엔 slave 노드에 e-stop 급 물리 버튼 1개 정도 추가 (표준상 slave 위치).  현재 구성은 논리 mode 지원만으로 §6.1.4.1 충족.

#### 4.4.5 OC (개폐형) 완료 신호 없는 slave 의 default hold

Simple `KS_OP_OP_OPEN(301) / CLOSE(302)` 는 표준 §B.3 상 "위치 도달 = READY 전환" 이지만, 우리 NPN downstream 은 SW 채널만 매핑 (16 relay), OC 채널 (idx 17~24) 은 물리 hw 없음 → 완료 신호도 없음 → HMI 상 CLOSING/OPENING 무한 지속.

Fix: `KSNODE_OC_DEFAULT_HOLD_SEC` Kconfig (default 3 s, range 1~60).  Simple OPEN/CLOSE 발행 시 remain 을 이 값으로 세팅 → tick 카운트다운 후 자동 READY.  TIMED_OPEN/CLOSE (303/304) 는 arg 그대로, SET_CONFIG (306) 는 미구현 (fallback 없이 default 사용).

#### 4.4.6 HMI 카드 스크롤 방지

LVGL v9 default 는 모든 컨테이너 `SCROLLABLE=ON`.  카드 dense grid 상 label 폭이 card 폭 초과 시 자동 좌우 스크롤 발생.  `build_card` 상 card + val_row + btn_row 에 `LV_OBJ_FLAG_SCROLLABLE` 제거 로 방지.

#### 4.4.7 SW 카드 dual-state 카드 자체 버튼화

SW 카드 조작을 카드 자체 click 으로 통합 — 기존 `ON/OFF` 두 버튼 → 카드 자체가 toggle button (`last_status == SW_ON` 이면 OFF 발행, 그 외 ON 발행).  OC 카드는 반복 OPEN/CLOSE (진행 중 재발행) 시나리오 지원 위해 기존 `OPEN/CLOSE` 두 버튼 유지.  optimistic UI (pending window + 즉시 색 반전) 는 두 경로 공통.

#### 4.4.8 HMI landscape rotation + 우측 사이드바 탭

HMI 를 가로형 (rotation=1, 물리 800×1280 → 논리 1280×800) 으로 전환하고 우측 사이드바 탭 (모니터링 / 제어) 로 재구성.  단계별 flash 검증:

- rotation=1 flip (LVGL v9 PPA 하드웨어 가속 SW rotation).  주의: LVGL task stack 은 7168→12288 확장 필요 (PPA 콜스택이 크게 소모, `Stack protection fault` 실측 후 fix).
- 상단 공통 헤더 (탭 무관): 로고 + Smart Farm + controller ID + 노드 meta + heartbeat (uptime/heap) + 시각 (T+HH:MM:SS, SNTP 미연결 시 uptime placeholder) + 터치 디버그 + NET 버튼.
- 모니터링 탭: 센서 zone (위) + 차트 zone (가운데) + 알림 placeholder (하단 fixed 80px).  좌우 분할 실험 후 세로 스택이 UX 상 나음 (좌우 분할 시 build_zone 상 `lv_pct(100)` width 세팅과 grid layout 간 순환 참조로 카드 납작 문제).
- 제어 탭: 구동기 zone (기존 SW/OC grid).
- 탭 스위칭: `LV_OBJ_FLAG_HIDDEN` 토글만 (obj destroy 없음).

### 4.5 Rust backend (axum)

Docker container 하나로 통합된 백엔드. host network 로 동작합니다.

**모듈 구성 (개념):**

```
backend/
├─ ingest/          MQTT (rumqttc) → 정규화 → DB insert
├─ api/             axum route: REST + WebSocket + static
├─ push/            VAPID Web Push (알림)
├─ discovery/       audit event ingest, addr/alias 관리
├─ purge/           async partial-delete job (chunk decompression 부담 회피)
├─ prune/           sensor prune (오래된 원시 row 정리)
├─ ksx/             KS X 검증 helper (verify widget backend)
└─ config/          .env 스키마 (자격증명은 배포 시 주입, repo 밖)
```

**주요 특성:**

- 단일 프로세스, 단일 컨테이너, host network → 마이크로서비스가 아님. 배포 단순화.
- **async purge**: TimescaleDB 압축 chunk 는 partial delete 시 chunk 별 decompression
  비용이 큼 (실측 21만 row ~180s). 사용자 요청을 job 으로 접수하고
  진행 상태를 별도 API 로 노출하는 패턴 채택.
- **sensor prune**: 활성 chunk 는 유지하되 오래된 원시 row 는 주기 정리.
  Continuous aggregate 로 대체 가능한 metric 은 대체 후 원본 축소.
- **환경변수 스키마**: DB URL, MQTT broker URL/TLS, COOKIE_KEY, VAPID public/private,
  push_subject 등. **본 공개본은 스키마만 언급하며 실제 값은 포함하지 않습니다.**

### 4.6 SvelteKit frontend

SvelteKit static adapter 로 빌드된 SPA. Rust backend 가 static route 로 서빙.

- **카드 대시보드**: 30 + 카드 유형. 실시간 값은 WebSocket 으로 push, 과거 값은
  REST 로 fetch. 그래프는 client-side 렌더.
- **KsxVerify 위젯**: 대상 device 를 골라 KS X 3267 명령별 판정을 즉석에서 실행하고
  통과/실패 · 판정 근거를 표기.
- **device-audit 페이지**: master 가 발행한 lifecycle 이벤트를 시간순으로 조회.
  added / moved / removed / reassign 을 필터.
- **addr / alias UI**: 사용자가 슬레이브에 사람이 읽는 이름을 부여하고, 표준 범위 내
  addr 변경을 요청할 수 있는 폼.
- **static 배포**: 빌드 산출물을 backend 가 정적 서빙. CDN 없음.

---

## 5. 데이터 모델

### 5.1 `sensor_data_v2` (hypertable)

시계열 원본 저장소. TimescaleDB hypertable.

```
CREATE TABLE migration_dev.sensor_data_v2 (
    device_id  smallint     REFERENCES devices_dim(id),
    metric_id  smallint     REFERENCES metrics_dim(id),
    ts         timestamptz  NOT NULL,
    value      real         NOT NULL
);

CREATE INDEX ON migration_dev.sensor_data_v2
    (device_id, metric_id, ts DESC);

SELECT create_hypertable('migration_dev.sensor_data_v2', 'ts',
                         chunk_time_interval => interval '7 days');
```

- **압축**: 오래된 chunk 는 TimescaleDB 컬럼 압축 적용. 활성 chunk 만 read-write.
- **Partial delete**: 압축 chunk 를 부분 삭제하려면 chunk decompression 이 발생하며
  비용이 큼 → **async purge worker** 로 처리 (§4.5).
- **Metric key 표준화**: 초기 구현의 `soil_ec` → `ec`, `soil_ph` → `ph`,
  `soil_humidity` → `soil_moisture` 로 리네임. KS X device code 12/16/14 는
  본질적으로 토양 전용이 아니므로 프리픽스 제거.

### 5.2 `devices` (사용자 메타)

디바이스에 대한 사람 관점의 메타데이터.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | smallint | PK |
| `device_id` | text | 자연 키 (자동등록 dim 과 매칭) |
| `alias` | text | 사용자 부여 이름 |
| `location` | text | 위치 태그 |
| `owner_user_id` | int | 소유 사용자 |
| `group_id` | int | 그룹 |
| `addr` | smallint | 현재 RS485 addr (참고용 · UI 필터) |

### 5.3 `devices_dim` (자동등록 dim)

Snapshot ingest 시 upsert 되는 dim. 목록의 source of truth 입니다.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | smallint | PK (compact key, sensor_data_v2 FK 대상) |
| `device_id` | text | 자연 키 |
| `first_seen` | timestamptz | 최초 관측 |
| `last_seen` | timestamptz | 최근 관측 |

`devices` (사용자 메타) 와 `devices_dim` (자동등록) 은 서로 다른 관심사이므로
별도 테이블로 유지하며 `device_id` (text) 로 조인합니다.

### 5.4 `metrics_dim`

Metric key (text) ↔ metric id (smallint) 매핑. `sensor_data_v2.metric_id` FK.
새 metric 은 ingest 시 upsert.

### 5.5 `device_audit` (lifecycle 이벤트)

Master 의 `discovery_audit` 서브시스템이 발행하는 이벤트를 영속화.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | bigserial | PK |
| `ts` | timestamptz | 이벤트 발생 시각 (master 관점) |
| `device_id` | text | 대상 슬레이브 |
| `event` | text | `added` / `moved` / `removed` / `reassign` |
| `from_bus` | smallint | 이동/재배정 전 버스 |
| `to_bus` | smallint | 이동/재배정 후 버스 |
| `from_addr` | smallint | 이전 addr |
| `to_addr` | smallint | 이후 addr |
| `payload` | jsonb | 부가 (스캔 근거 등) |

UI 는 시간순 timeline 으로 표시. 자세한 discovery 정책과 이벤트 생성 규칙은
`docs/discovery.md` 참조.

### 5.6 데이터 흐름 요약

```
   Slave                Master              MQTT              Backend                DB
     │                    │                   │                  │                    │
     │ ─ RS485 read ────▶ │                   │                  │                    │
     │ ◀── response ───── │                   │                  │                    │
     │                    │ ─ snapshot ─────▶ │ ─ subscribe ───▶ │ ─ ingest ───────▶  │
     │                    │                   │                  │  normalize         │
     │                    │                   │                  │  upsert dim        │
     │                    │                   │                  │  insert sample     │
     │                    │                   │                  │                    │
     │                    │ ─ audit event ──▶ │ ─────────────▶  │ ─ insert audit ─▶  │
     │                    │                   │                  │                    │
     │                    │                   │                  │ ─ WS broadcast ─▶ Frontend
     │                    │                   │                  │ ─ push (VAPID) ─▶ 브라우저
```

---

## 6. KS X 3267 검증 상태

**결론: 전수 PASS.** 발견된 4 건의 비적합은 모두 fix 완료. 상세는
`docs/verification.md` (검증 보고서 · 명령별 판정 기준 · 워크플로우) 참조.

### 6.1 검증 범위

| 항목 | 상태 |
|---|---|
| N-1 ~ N-5 (노드 메타 read) | PASS |
| S-1 (센서 값 read) | PASS |
| SW / OC 계열 (구동기 스위치 · 카운터) | PASS |
| 구동기 cmd / status PDU 필드 순서 | PASS (비적합 #1 fix 후) |
| Default register map 준수 | PASS |
| 주소 배정 · 초기화 흐름 | PASS (사설 discovery 는 §5.1 범위 내) |

### 6.2 발견 · 수정 이력 (요약)

| # | 항목 | 조치 |
|---|---|---|
| 1 | 구동기 cmd/status 블록 PDU 필드 순서 상이 | 표준 순서에 맞춰 재배열 |
| 2 | Metric key 명명 (토양 전용 프리픽스) | `soil_ec` → `ec` 등 리네임 |
| 3 | Sensor kind 전환 시 addr 잔존 | flash-slave.sh 절차에 재검증 단계 추가 |
| 4 | 3-tier addr fallback 경계값 | 경계 조건 명시 · 회귀 테스트 추가 |

### 6.3 검증 자동화

- Frontend 의 **KsxVerify 위젯** 은 대상 device 를 선택하여 각 명령을 즉석에서 실행하고
  판정 근거 (원 PDU · 기대값 · 실제값) 를 표시.
- Backend 의 `ksx/` 모듈이 판정 로직 (문서화된 판정 기준을 코드화) 을 담당.

---

## 7. KS X 3286 vendor spec 통합

벤더 · 비표준 노드를 표준 파이프라인에 편입시키는 경로입니다.

### 7.1 문제

KS X 3267 default register map 은 대표적 센서/구동기만을 다루므로,
default map 에 없는 벤더 노드는 그대로는 폴링·표시가 불가능합니다.
KS X 3286 은 이런 노드가 **자기 register map (spec) 을 광고** 하도록 규정합니다.

### 7.2 접근

- Master 가 KS X 3286 vendor 노드를 감지하면 spec 을 확보해야 함.
- **원천**: (a) SPIFFS 로컬 캐시, (b) backend 에서 HTTP fetch, (c) NVS 캐시.
- 확보된 spec 에 따라 마스터는 자기 폴링 스케줄러에 vendor 노드를 정상 slave 로 편입.

### 7.3 상태

- Master 측 통합: **완료** (spec 소비 · 폴링).
- Backend 측 spec fetch 경로: **설계 완료 · 구현 진행**.
- SPIFFS · NVS 캐시 정책, spec 스키마 정의, 갱신 절차 등의 상세는
  `docs/vendor-spec.md` 로 분리.

---

## 8. 디스커버리 · 라이프사이클 감사

새 슬레이브가 라인에 물렸을 때, 마스터가 그것을 감지 · 배정 · 등록하는 절차와
이후의 이동/사라짐을 감시하는 절차입니다.

### 8.1 디스커버리 삼단 (요약)

```
     ┌──────────────────────────┐
     │  Trigger:                │
     │   ├─ 사용자 요청          │
     │   ├─ GPIO edge (per-port)│
     │   └─ 주기 스캔            │
     └────────────┬─────────────┘
                  │
                  ▼
     ┌──────────────────────────┐
     │  Step 1: Sentinel probe  │
     │   reg 250 을 broadcast   │
     │   or 후보 addr 로 read   │
     └────────────┬─────────────┘
                  │
                  ▼
     ┌──────────────────────────┐
     │  Step 2: 3-tier fallback │
     │   ①  요청 addr → 존재?    │
     │   ②  후보 addr → 존재?    │
     │   ③  스캔 → 자유 addr 배정│
     └────────────┬─────────────┘
                  │
                  ▼
     ┌──────────────────────────┐
     │  Step 3: audit event     │
     │   added / moved / reassign│
     │   → MQTT → backend       │
     └──────────────────────────┘
```

### 8.2 GPIO per-port 신호

마스터의 각 RS485 포트에는 커넥터 삽입 감지를 위한 GPIO 가 결선되어 있어,
사용자가 새 슬레이브를 물리적으로 꽂았을 때 즉시 디스커버리를 트리거할 수 있습니다.
이는 표준 요구사항이 아닌 UX 개선을 위한 사설 확장이며 `docs/discovery.md` 에 명시.

### 8.3 감사 이벤트

디스커버리 결과는 `device_audit` (§5.5) 에 영속화되어 UI 에서 timeline 으로 조회
가능합니다. 이벤트 종류와 각각의 판정 규칙, 재배정 정책 등은 `docs/discovery.md` 참조.

---

## 9. 배포

### 9.1 컨테이너

- **컨테이너**: `seriallink-backend-dev`
- **이미지**: `seriallink-backend:dev`
- **네트워크**: host network (backend 가 mosquitto/nginx 와 loopback 통신)
- **런타임 마운트**:
  - `.env` (자격증명, repo 밖)
  - TLS 인증서 (mqtt · web)
  - 정적 자원 (SvelteKit build)

### 9.2 배포 스크립트

`scripts/deploy-server.sh` 는 다음을 자동화합니다.

1. 로컬 빌드 (frontend static, backend 는 컨테이너 안에서 cargo build).
2. `rsync` 로 소스 → VPS 로 동기화.
3. VPS 에서 `docker build`.
4. 기존 컨테이너 stop/rm → 새 컨테이너 run.
5. Smoke test (health endpoint, MQTT connect 확인).

> 스크립트 자체는 자격증명 하드코딩 위험이 있어 `.gitignore` 로 배제되며,
> 공개 저장소에는 포함되지 않습니다. `.env` 스키마와 배포 흐름만 문서화합니다.

### 9.3 nginx

- monorepo 내 `server/nginx-system-sites/` 가 **source of truth**.
- VPS 의 `/etc/nginx/sites-enabled/` 는 monorepo 파일로 symlink.
- 주요 역할:
  - HTTPS termination.
  - WebSocket upgrade 프록시.
  - `/wiki` 등 정적 자원.
  - `<service-host>` → `127.0.0.1:8000` 리버스 프록시.

### 9.4 관측

- MQTT broker log · backend log · nginx log 는 host 상 표준 위치.
- 애플리케이션 메트릭 (ingest rate, ws client count 등) 은 backend 의 REST endpoint 로
  노출 (인증 필요).

### 9.5 백업

- PostgreSQL 은 host 상에서 실행되며 별도 백업 스케줄에 따라 dump.
- TimescaleDB 압축 chunk 는 dump 크기에 영향을 주지만 별도 export 절차는 표준.
- 자세한 백업 · 복원 절차는 이 공개 문서의 범위 밖 (사설 운영 문서 참조).

---

## 10. 부록: 진행 상태 요약

### 10.1 컴포넌트별 상태

| 컴포넌트 | 상태 |
|---|---|
| Master (ESP32-S3): RS485 9600 bps, KS X 3267 준수, KS X 3286 vendor spec 통합 | 완료 |
| Master discovery: GPIO per-port + sentinel reg 250, 3-tier addr fallback | 완료 |
| Master lifecycle audit (added / moved / removed / reassign) | 완료 |
| Slave-c3 sensor: soil / ec / co2 / vendor / dummy variant | 완료 |
| Slave-c3 actuator | 완료 |
| HMI P4: LVGL card slot 관리 (2-tier stale, hard/soft) | 완료 |
| Backend: MQTT ingest, TimescaleDB, WebSocket, VAPID push, async purge, sensor prune | 완료 |
| Frontend: 30+ 카드, KsxVerify 위젯, device-audit, addr/alias UI | 완료 |
| KS X 3267 전수 검증 (default map 범위, 비적합 4 건 fix) | 완료 |
| KS X 3286 vendor spec fetch (SPIFFS/HTTP) | 진행 중 (설계 완료) |
| KS X 3288 양액기 | 미시작 |

### 10.2 버스 · 포트 라벨 관례

모든 계층 (PCB · 펌웨어 · MQTT · UI) 에서 다음과 같이 통일합니다.

| 라벨 | PCB 커넥터 | 마스터 UART | GPIO (S3) | MQTT 필드 |
|---|---|---|---|---|
| **BUS#1** | 좌측 커넥터 | UART2 | TX=15 · RX=14 · DE/RE=6 | `bus: 1` |
| **BUS#2** | 우측 커넥터 | UART1 | TX=17 · RX=16 · DE/RE=7 | `bus: 2` |

라벨 반전 시 사고 이력이 있으므로 (모든 계층에서 동일 번호가 물리 커넥터를 가리키도록)
새 하드웨어 · 새 펌웨어 도입 시 이 관례를 최우선 확인합니다.

### 10.3 관련 문서

이 아키텍처 문서 외에 다음 하위 문서가 세부를 다룹니다.

| 문서 | 다루는 것 |
|---|---|
| `docs/verification.md` | KS X 3267 명령별 판정 기준, 검증 워크플로우, 발견 · 수정 이력 |
| `docs/discovery.md` | 디스커버리 삼단, GPIO per-port, 3-tier addr fallback, audit 이벤트 상세 |
| `docs/vendor-spec.md` | KS X 3286 spec fetch 설계, SPIFFS/NVS 캐시 정책 |

### 10.4 용어

| 용어 | 의미 |
|---|---|
| **Node** | KS X 3267 의 노드 (센서 · 구동기 · 제어기). |
| **Master** | 본 프로젝트의 게이트웨이 (ESP32-S3). KS X 3267 의 제어기 역할. |
| **Slave** | 본 프로젝트의 센서 · 구동기 노드 (ESP32-C3). |
| **HMI** | 현장 터미널 (JC-ESP32-P4-M3, LVGL). |
| **BUS** | 물리 RS485 커넥터 하나. 마스터는 BUS#1, BUS#2 두 채널 보유. |
| **Snapshot** | 마스터가 폴링 후 정규화하여 MQTT 로 발행하는 값 집합. |
| **Audit event** | 슬레이브 lifecycle 이벤트 (added / moved / removed / reassign). |
| **Hypertable** | TimescaleDB 의 시계열 파티션 테이블. |
| **Compressed chunk** | Hypertable 의 오래된 파티션에 컬럼 압축이 적용된 상태. |
| **Async purge** | 압축 chunk 의 partial delete 를 백그라운드 job 으로 수행하는 패턴. |

---

*본 문서는 공개용 스냅샷입니다. 운영 파라미터 · 자격증명 · 사설 호스트 정보는
의도적으로 제외되었으며, 재현을 위한 지침이 아니라 아키텍처 이해를 위한 참고 자료입니다.*
