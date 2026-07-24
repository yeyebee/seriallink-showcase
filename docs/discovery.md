# Discovery — 표준 §5.1 범위 내 사설 확장

> **한 줄 요약**: KS X 3267 이 명시적으로 "자유"라고 남겨둔 슬레이브 주소 부여 방식을, 마스터가 포트별 GPIO 로 슬레이브 응답 시점을 제어하는 **per-port 자동 배정** 으로 채웠다. 표준 프레임/함수코드 100% 준수, 외부 표준 마스터·노드와도 그대로 호환.

관련 문서
- [architecture.md](architecture.md) — 시스템 전체 계층 (firmware / gateway / backend / HMI)
- [vendor-spec.md](vendor-spec.md) — KS X 3286 자동등록 흐름 (외부 vendor 노드 통합)
- [verification.md](verification.md) — 표준 시험 매트릭스와 재현 절차

---

## 1. 배경 — §5.1 이 어디까지 규정하고 어디서 자유를 주는가

KS X 3267 은 산업/시설 원격 모니터링용 시리얼 프로토콜 표준이다. 마스터(제어기) 하나가 RS-485 버스에 슬레이브 (센서/구동기) 여러 대를 물고 Modbus RTU 로 통신한다. 표준이 규정하는 것과 규정하지 않는 것을 정확히 나누는 게 이 문서의 출발점이다.

**표준이 규정하는 것** (조문 그대로 지켜야 하는 부분):

| 조 | 내용 |
|---|---|
| §4.1 | 유니캐스트 주소 범위 `1..247` (0 = 브로드캐스트, 248..255 예약) |
| §4.3.3 | 워드 인코딩 `word-LE` (같은 32비트값을 2 레지스터로 나눌 때 하위 워드가 낮은 주소) |
| §6.1 | 초기화 시퀀스 — NodeInfo(reg 1~8) 조회 → DeviceCode(reg 101~) 조회 → 이후 상태/제어 |
| 부속서 A | 디폴트 레지스터 맵 (센서 값 201~, 구동기 상태 202~, 제어 301~ …) |

**표준이 규정하지 않는 것** (§5.1 이 명시적으로 언급하는 자유):

> *"이 표준은 슬레이브 주소가 어떻게 부여되는지 규정하지 않는다. DIP 스위치, 설정 명령, 자동등록 (KS X 3286) 등 자체 방식으로 미리 지정되어 있음을 가정한다."*

이 문장이 열어놓은 세 갈래 선택지:

- **① 수동 지정** — DIP 스위치, 로터리, 시리얼 콘솔 명령. 물리 조작 필요.
- **② KS X 3286 자동등록** — 표준 정공법이지만 이 표준의 관심사는 "회사·제품 코드로부터 레지스터 맵을 계산하는 것" 이지 addr 부여 자체가 아니다. 3286 §5.2.a 도 *"각 노드의 주소는 관리자가 자체 방식으로 미리 지정되어 있음을 가정"* 이라고 못박는다.
- **③ 스캔** — 마스터가 `1..247` 을 훑어 응답 있는 addr 을 registry 에 등록. 물리적 위치와 논리 주소 매핑이 불가능하고 (같은 addr 슬레이브가 여러 개 붙는 사고에 취약) 규모가 커지면 timeout 누적 비용이 크다.

두 표준 모두 "주소는 이미 있다" 를 전제로 나머지를 정의한다. 우리 discovery 는 그 전제를 **우리 마스터 + 우리 슬레이브 세트에서 자동으로 만족시키는 편의** 다. 표준 준수 여부와는 별도 계층에 있는 이야기.

---

## 2. 문제 — 20~30 대 붙는 실 운영의 부담

시연 세트라면 슬레이브 수가 5대 미만이라 각각에 addr 을 하드코딩으로 flash 해도 부담이 없다. 그러나 실 온실/스마트팜/시설 하나가 붙기 시작하면 상황이 바뀐다.

- 같은 펌웨어 이미지를 20~30 대 슬레이브에 얹고, 각각의 addr 을 flash 시점에 개별 지정하려면 sdkconfig 만 다른 20개 빌드가 나온다.
- 현장에서 슬레이브 하나가 고장나 교체할 때, 창고에 있는 예비 슬레이브를 그 자리에 꽂으려면 **먼저 그 자리의 addr 로 flash 부터 해야 한다**. 노트북 + 케이블 필요.
- 물리적 커넥터 번호와 addr 이 매핑되어 있지 않으면 UI 는 "센서 3번" 이라는 라벨을 표시하는데 그게 실제로 몇번 커넥터에 꽂힌 건지 사람이 별도로 관리해야 한다.

우리는 **마스터 PCB 에 슬레이브 커넥터 5개** 가 있고, 각 커넥터 자리에 RS-485 A/B/GND 3선 + **GPIO 신호선 1선**, 총 4선을 배선했다. 이 GPIO 신호선이 discovery 의 하드웨어 근간이다.

목표를 한 줄로:

> **"물리 포트 번호 = 슬레이브 addr = UI 채널 번호"** 를 자동으로 만족시키되, 표준 프레이밍 · 함수코드는 손대지 않는다.

---

## 3. 우리 접근 — 개요

전체 그림.

```
┌──────────────────── Master (ESP32-S3) ────────────────────┐
│                                                              │
│   RS-485 UART  ────────────────────── A / B / GND ─────┐    │
│                                                          │   │
│   GPIO1  ────── discovery signal to port#1 ─────────┐   │   │
│   GPIO2  ────── discovery signal to port#2 ─────┐   │   │   │
│   GPIO3  ────── discovery signal to port#3 ─┐   │   │   │   │
│   GPIO4  ────── discovery signal to port#4  │   │   │   │   │
│   GPIO5  ────── discovery signal to port#5  │   │   │   │   │
└─────────────────────────────────────────────┼───┼───┼───┼───┘
                                              │   │   │   │
                             port#3           │   │   │   │
                           ┌────────┐         │   │   │   │
                           │ Slave  │◄────────┘   │   │   │
                           │ (C3)   │◄────────────┘   │   │
                           │ GPIO7  │◄────────────────┘   │
                           └────┬───┘◄────────────────────┘
                                └── shared RS-485 A/B/GND
```

- 모든 포트가 같은 RS-485 버스 (A/B/GND) 를 공유. 하나뿐인 시리얼 트랜시버.
- 각 포트마다 마스터에서 별도 GPIO 신호선이 나와 슬레이브 쪽 GPIO7 (input, pull-down) 에 연결.
- **Idle 상태**: 모든 마스터 GPIO = LOW → 모든 슬레이브의 GPIO7 = LOW.
- **Discovery 상태**: 마스터가 포트 하나만 GPIO HIGH → 그 포트에 꽂힌 슬레이브만 자기 GPIO7 이 HIGH 로 올라간 것을 감지.

이 GPIO 신호는 표준 트래픽 (RS-485 A/B) 위로 흐르는 것이 아니고 완전히 별도의 신호선이다. 표준 시험 장비/외부 마스터에는 이 신호가 존재하지 않으며, 존재하지 않아도 슬레이브는 정상 통신 가능 (fallback tier 로 넘어감. §5 참조).

두 개의 사설 값 — 표준 맵 밖의 예약된 자리 — 을 지정한다.

| 값 | 의미 | 표준 근거 |
|---|---|---|
| **Ephemeral addr `247`** | Discovery 중 슬레이브가 임시로 반응하는 유니캐스트 주소. 정상 배정 후엔 절대 사용 안 함. | §4.1 이 정의하는 유니캐스트 범위의 최상단 값. 지금까지 우리 배포 사례에서 슬레이브가 addr=247 을 실제 운영 주소로 갖게 배정되는 경우는 없음 (5포트 마스터라 최대 5). |
| **Sentinel reg `250`** | Discovery 중 마스터가 슬레이브에게 "너의 새 주소는 N" 이라고 write 하는 레지스터. | 부속서 A.1.2 미할당 영역 (표준 맵은 `202..292`, `501..598` 등을 지정, 250 은 어디에도 없음). Vendor-specific 사용이 허용되는 영역. |

**정상 운영 사이클에서는 addr 247 도 reg 250 도 결코 등장하지 않는다.** 배정 완료 즉시 슬레이브는 표준 맵 100% 로 동작 (센서 값 201~, 구동기 상태 202~ 등).

---

## 4. Master 측 — GPIO edge + Sentinel write

### 4.1 시퀀스

한 포트 배정 흐름.

```
                Master                                     Slave (unassigned)
                ──────                                     ──────────────────
Step 1   GPIO_N = HIGH   ──── 신호선 ─────►  GPIO7 rising edge 감지
                                                            │
Step 2   TSETTLE (50 ms) 대기                              │ ephemeral mb slave
                                                            │ 기동 (uid=247)
Step 3   FC03 read addr=247 reg 1..8   ─── RS-485 ──►     │
                                       ◄─────────────      NodeInfo 응답
                                                            (serial 포함)
Step 4   serial 추출

Step 5   FC06 write addr=247 reg 250 = N  ── RS-485 ──►   │ sentinel callback:
                                                            │ NVS에 addr=N 저장
                                                            │ mb slave 재기동
Step 6   TCOMMIT (300 ms) 대기                             │ (uid=N, 표준 맵)

Step 7   FC03 read addr=N reg 7..8       ── RS-485 ──►    │
                                       ◄─────────────      serial 응답 (verify)

Step 8   Registry 등록:                                    ◄ addr N 로 정상 운영
         (bus, addr=N, serial) + lifecycle event
         GPIO_N = LOW

  다음 포트로 반복 →
```

핵심 포인트:

- **함수코드는 FC03 (read holding) / FC06 (write single register) 만 사용** → RTU 프레임 규격 100% 준수. 외부 표준 마스터 시뮬레이터가 캡처해도 표준 위반 프레임은 나오지 않는다.
- **verify step (Step 7)** 은 배정 후 실제로 그 addr 로 통신 가능한지 + 원래 응답한 슬레이브가 맞는지 (serial 일치) 를 확인. 오배정 detection.
- **GPIO_N LOW 로 내리기** — 배정 완료 후 원상복귀. 다른 포트 배정 시 GPIO 상태 순수성 유지.

### 4.2 타이밍 상수

`master_discovery.c` 상단 정의.

| 상수 | 값 | 이유 |
|---|---|---|
| `TSETTLE_MS` | 50 ms | 슬레이브가 GPIO7 rising 감지 → ephemeral mb slave 기동에 걸리는 시간 여유 |
| `TCOMMIT_MS` | 300 ms | 슬레이브가 NVS commit + mb_slave stop + `mb_slave_start(new_addr)` 하는 데 필요. 150 ms 로 시작했다가 verify 실패 사례 (2026-07-22) 관측 후 상향 |
| Modbus timeout | 100 ms | RTU 3.5 char + 슬레이브 처리 여유 (§6.1 초기화 시퀀스 권장 timeout 범위 내) |

### 4.3 코드 진입점

`master_discovery.h` 에 세 함수만 노출된다.

- `master_discovery_init(bus_id)` — GPIO output 세팅, 모두 LOW. `SFC_DISCOVERY_ENABLED=n` 이면 no-op.
- `master_discovery_scan(bus_id, port_start, port_end, skip_known)` — 1회 스캔. `skip_known=true` 면 이미 registry에 있고 `reachable=true` 인 포트 skip.
- `master_discovery_start_rescan_task(bus_id)` — 주기적 rescan task 기동 (`SFC_DISCOVERY_RESCAN_MS > 0` 시).

Boot 시퀀스는 `init → scan(1, 5, false) → start_rescan_task` 순.

### 4.4 무응답 포트 처리

Step 3 FC03@247 timeout 시, 이 포트에 이전에 배정된 registry entry 가 있으면 `reachable=false` 로 stale mark 하고 `REMOVED` audit event 발행 (슬레이브가 다른 포트로 이동했거나 물리 제거된 경우). Poller 는 이후 이 노드 poll 생략, UI 는 회색 처리. 24 시간 stale 유지 시 registry GC 로 삭제 (`ks_registry_gc_stale`).

---

## 5. Slave 측 — 3-tier addr fallback

슬레이브 부팅 시 addr 을 어디서 가져오는가.

```
                    Boot
                     │
                     ▼
        ┌────────────────────────────┐
        │  Tier 1                    │
        │  slave_addr_load(&addr)    │
        │  (NVS namespace=ks_disc)   │
        └────────────┬───────────────┘
                     │
             hit ────┤
                     │              miss
                     │                ▼
                     │   ┌──────────────────────────────┐
                     │   │  Tier 2                      │
                     │   │  Kconfig KSNODE_SLAVE_ADDR   │
                     │   │  (컴파일 타임 상수)          │
                     │   └────┬─────────────────────────┘
                     │        │
                     │   ≠ 0 ─┤
                     │        │                    == 0
                     │        │                      ▼
                     │        │        ┌──────────────────────────┐
                     │        │        │  Tier 3                  │
                     │        │        │  discovery_wait_and_     │
                     │        │        │  assign(&addr)           │
                     │        │        │  (GPIO7 edge 대기)       │
                     │        │        └────┬─────────────────────┘
                     │        │             │ (blocking)
                     ▼        ▼             ▼
              mb_slave_start(addr) → normal operation

              if (tier != 2)
                  discovery_start_reassign_watcher();
```

세 tier 의 존재 이유:

| Tier | 트리거 조건 | 용도 |
|---|---|---|
| 1 (NVS) | NVS 에 addr 저장돼있음 | 우리 discovery 로 배정 완료 후 정상 운영. 전원 사이클 후 재사용 |
| 2 (Kconfig) | NVS 없음 + `KSNODE_SLAVE_ADDR != 0` | **표준 시험 · 외부 마스터 조합.** 이 슬레이브를 외부 표준 마스터 tester 에 물릴 때, 항상 그 addr 로 응답해야 함 (변경 금지) |
| 3 (Discovery) | NVS 없음 + `KSNODE_SLAVE_ADDR == 0` | 우리 마스터 discovery 대기. Blocking — GPIO7 rising 감지 후 sentinel write 받을 때까지 mb_slave 는 unassigned |

### 5.1 Tier 2 에서 watcher 를 안 켜는 게 핵심

`if (tier != 2) discovery_start_reassign_watcher();` — 표준 시험 tier 에서는 GPIO7 신호를 아예 감시하지 않는다. 외부 테스터가 슬레이브 addr 을 어떤 값으로 지정했든, 우연히 GPIO7 에 신호가 들어와도 addr 이 바뀌지 않아야 하기 때문이다.

이 3-tier 원칙은 **하나의 코드 트리로 두 목적 (우리 세트 편의 + 표준 시험) 을 동시 지원** 하는 근간. 이게 없으면 검증용 펌웨어와 운영용 펌웨어를 분리 flash 해야 한다.

### 5.2 코드 진입점

- `slave_addr_store.h` — `slave_addr_load/save/clear` (NVS namespace `ks_disc`, key `slave_addr`).
- `discovery.h` — `discovery_wait_and_assign(&addr)` (Tier 3 진입점, blocking) 과 `discovery_start_reassign_watcher()` (normal mode 진입 후 호출).

---

## 6. In-place reassign — restart 없이 mb_slave swap

슬레이브를 다른 포트로 재배선했을 때 (또는 사용자가 강제 재배정을 원할 때) 이 슬레이브의 addr 은 새 포트 번호로 바뀌어야 한다.

### 6.1 첫 시도 — esp_restart() 접근

```c
// GPIO7 rising 감지 → NVS clear → esp_restart()
// 부팅 후 tier 3 로 진입 → discovery 재수행
```

문제: **재부팅에 200 ms 걸린다.** 마스터의 5-port 순차 스캔이 각 포트 사이 500 ms 정도. 슬레이브가 restart 중일 때 마스터는 이미 다음 포트로 넘어가 있고, 슬레이브가 tier 3 진입한 시점엔 자기 포트 GPIO 가 이미 LOW → 다음 rescan cycle (10 초) 까지 대기.

### 6.2 In-place swap

```
GPIO7 rising edge 감지 (in normal mode)
  │
  ▼
slave_addr_clear()               ← NVS 삭제
  │
  ▼
mb_slave_stop()                  ← 현재 uid=N 인 mb slave 종료
  │
  ▼
mb_slave_start_ephemeral()       ← uid=247, sentinel 영역만 등록
  │                                 (센서/구동기 task 는 계속 running)
  ▼
sentinel reg 250 poll
  │  (50 ms 주기, master 가 FC06 write 할 때까지 대기)
  ▼
value = 새 addr 수신
  │
  ▼
slave_addr_save(new)             ← NVS 저장
  │
  ▼
mb_slave_stop()
  │
  ▼
mb_slave_start(new)              ← 새 addr 로 normal 재기동
  │
  ▼
계속 running, 다음 falling edge 대기
```

**Restart 없음.** 20~50 ms 안에 ephemeral 진입. 마스터가 아직 포트 GPIO HIGH 유지 중일 때 (Step 6 TCOMMIT 대기 구간) 슬레이브가 응답 → 같은 rescan 사이클에서 배정 완료.

### 6.3 부팅 loop 방지 — falling edge 부터 대기

방금 배정받은 직후엔 GPIO7 이 여전히 HIGH (마스터가 verify 후 LOW 로 내리기 전 gap). Reassign watcher 가 rising edge 만 보고 있으면 배정 직후 즉시 재배정이 다시 트리거되어 loop 에 빠진다.

```c
static void wait_falling(int PIN) {
    if (gpio_get_level(PIN) == 1) {
        while (gpio_get_level(PIN) == 1) {
            vTaskDelay(pdMS_TO_TICKS(50));
        }
    }
}
```

Watcher task 진입 시 GPIO7 이 이미 HIGH 면 falling edge 부터 대기 → 그 다음 rising 만 재배정 트리거로 인식.

### 6.4 Rescan 주기

마스터의 rescan 주기는 60 s 로 시작했다가 → **10 s** 로 단축 (2026-07-22). 슬레이브 재배정을 in-place 로 바꾸면서 timing race 는 소멸했지만, 라이브 중 물리 이동 후 첫 감지까지 사용자 관점 최대 대기 시간을 줄이기 위함.

---

## 7. Lifecycle audit — serial 기반 이력

Addr 만으로 슬레이브를 관리하면 놓치는 관점이 있다.

> **"어제 addr=3 이었던 슬레이브가 오늘 addr=5 로 나타났다. 이건 물리 이동인가, 교체인가?"**

이걸 구분하려면 addr 이 아닌 **불변 식별자** — serial (MAC 파생 uint32) — 로 slave 를 tracking 해야 한다.

### 7.1 이벤트 판정

```c
// master_discovery.c 의 try_assign_one() 후반부

ks_node_t *existing = ks_registry_find_by_serial(serial_orig);
if (!existing) {
    evt = DISC_EVT_ADDED;                    /* 새 serial */
} else if (existing->bus_id == bus_id &&
           existing->slave_addr == port_n) {
    evt = DISC_EVT_REASSIGN;                 /* 같은 위치 재확인 */
} else {
    evt = DISC_EVT_MOVED;                    /* 다른 위치로 이동 */
    from_port = existing->slave_addr;
    ks_registry_remove(existing);            /* old entry cleanup */
}
```

FC03@247 timeout 로 stale mark 될 때는 `DISC_EVT_REMOVED` 발행.

### 7.2 네 이벤트

`discovery_audit.h` 상 `disc_event_t`: `ADDED` (새 serial), `MOVED` (기존 serial, 다른 port), `REMOVED` (배정된 port 무응답), `REASSIGN` (같은 위치 재확인).

### 7.3 발행 경로

```
try_assign_one() 판정
       │
       ▼
discovery_audit_emit(&ev)
       │
       ├─── esp_log (AUDIT 태그) → serial 콘솔
       │
       └─── s_uplink_cb (uplink 모듈이 등록)
             │
             ▼
       MQTT publish → 게이트웨이 (backend)
             │
             ▼
       backend: device_audit 테이블 insert
             │
             ▼
       WebSocket broadcast → frontend
             │
             ▼
       /device-audit 페이지 timeline 업데이트
```

MQTT payload 예시.

```json
{
  "event": "moved",
  "serial": "0x8D296DD8",
  "bus": 2,
  "from_port": 3,
  "to_port": 5,
  "addr": 5,
  "ts_ms": 1721782800123
}

{
  "event": "added",
  "serial": "0x093307FC",
  "bus": 1,
  "port": 2,
  "addr": 2,
  "ts_ms": 1721782900456
}
```

### 7.4 저장·표시 layer 분리

**설계 원칙**: Discovery 프로토콜 (firmware) 은 표준 프레임만 사용. Audit 프로토콜은 표준 위에 얹은 **완전 별개 layer**.

- Firmware: 이벤트 생성만.
- MQTT topic (`<prefix>/discovery`): 이벤트 전달.
- Backend `device_audit` 테이블: timeseries 저장.
- Frontend `/device-audit` 페이지 + 대시보드 위젯: 표시.

이 분리 덕에 표준 검증 시 audit layer 를 통째로 꺼도 firmware 는 표준에 남는다.

### 7.5 시나리오별 최종 동작

**시나리오 1 — 슬레이브 A 를 port#3 → port#5 재배선 (전원 유지):**

```
t=0    Master rescan (10 s cycle) → port#3 HIGH
       → 슬레이브 A 무반응 (이제 이 포트 아님)
       → registry.(bus, 3).reachable = false
       → audit: REMOVED(serial=A, port=3)

t+~1s  Master → port#5 HIGH
       → 슬레이브 A GPIO7 rising 감지 → in-place reassign
       → NVS clear → ephemeral (uid=247)
       → master: FC03@247 응답 → FC06 write reg 250 = 5
       → 슬레이브 A: NVS save(5) → mb_slave_start(5)
       → master: verify FC03@5 → registry.find_by_serial(A)
       → 이전 (bus, 3) entry 는 방금 stale 처리됐지만
         serial 이 동일하므로 MOVED 로 판정
       → audit: MOVED(serial=A, from=3, to=5)

UI:    addr=3 카드 사라짐, addr=5 카드 신규 (from_port 표시 옵션)
```

**시나리오 2 — 슬레이브 A 제거 → 슬레이브 B (fresh) 를 port#3 에 삽입:**

```
t=0    슬레이브 A 물리 제거 → poller timeout 누적
       → 다음 rescan 에서 port#3 무응답 → REMOVED(A)

t+~10s Master → port#3 HIGH
       → 슬레이브 B (unassigned tier) 응답
       → 배정 성공, registry.find_by_serial(B) = NULL
       → ADDED(serial=B, port=3, addr=3)

이후:  슬레이브 A 를 다른 포트에 재삽입해도 그 포트 activate 시
       GPIO7 rising → in-place reassign → 새 포트 번호로 재배정.
       addr 충돌 없음 (물리 포트 = addr 이 유일 매핑이므로).
```

---

## 8. 검증 환경과 운영 환경의 공존

우리 세트를 표준 시험 (외부 시뮬레이터/tester 상대) 에 걸어야 할 때가 있다. 그때도 같은 코드 트리, 같은 이미지 계열이어야 유지 부담이 없다.

### 8.1 Master 모드 (Kconfig)

| 모드 | Kconfig | 동작 | 용도 |
|---|---|---|---|
| DISCOVERY_ENABLED | `CONFIG_SFC_DISCOVERY_ENABLED=y` (default) | Boot 시 GPIO 스캔·배정 + periodic rescan | 우리 세트 |
| DISCOVERY_DISABLED | `CONFIG_SFC_DISCOVERY_ENABLED=n` | 하드코딩 registry (sdkconfig 상 addr 목록) 만 사용 | 표준 시험, 외부 슬레이브 조합 |

DISABLED 모드에서도 §6.1 시퀀스 / 프레임 규격 100% 유지 → 외부 슬레이브 시뮬레이터로 마스터 표준 시험 가능.

### 8.2 Slave 모드 (3-tier)

이미 §5 에서 다룬 대로.

- Tier 2 (Kconfig 고정) 로 flash → 외부 표준 마스터 상대 시험 가능. GPIO7 watcher 는 아예 안 켜지므로 우리 마스터 GPIO 신호에 오염되지 않음.
- Tier 3 (unassigned) 로 flash → 우리 마스터 discovery 대상.

### 8.3 시험 매트릭스 (요약)

Discovery 와 직접 얽힌 케이스만 아래 요약. 전체 매트릭스는 [verification.md](verification.md) 참조.

| # | 조합 | 세팅 | 검증 목표 |
|---|---|---|---|
| 1 | 우리 master + 우리 slave 5대 (unassigned) | slave tier 3 | Boot scan 으로 5대 모두 배정 |
| 2 | 부분 장착 (1, 3, 5 만) | 위와 동일 | port 2, 4 skip 후 1/3/5 배정 |
| 3 | 라이브 hot-plug | 위와 동일 | 첫 rescan cycle 내 배정 (≤10 s) |
| 4 | 재배선 (port#3 → port#5) | 위와 동일 | In-place reassign, audit MOVED 발행 |
| 5 | 전원 사이클 | 배정 완료 상태 | slave NVS addr 재사용, master REASSIGN |
| 6 | 외부 표준 master + 우리 slave tier 2 | slave `KSNODE_SLAVE_ADDR=1..247` | 표준 부속서 A 맵으로 통신 |
| 7 | 우리 master DISABLED + 외부 표준 slave | master `SFC_DISCOVERY_ENABLED=n` | §6.1 시퀀스로 외부 slave 통신 |

---

## 9. 왜 표준 §5.1 범위 내에 있는가

이 절이 이 문서의 핵심 정당화다.

### 9.1 프레이밍 · 함수코드 · 워드 순서

Discovery 시퀀스 전 구간에서:

- **함수코드**: FC03 (read holding), FC06 (write single register). 둘 다 §4 에서 정의된 표준 함수코드.
- **프레임 구조**: `[addr | fc | payload | CRC16]` — Modbus RTU 표준 그대로.
- **워드 인코딩**: 마스터가 FC06 으로 sentinel reg 250 에 쓰는 값 (`port_n`) 은 single register (16-bit) 이므로 word-LE 무관. Serial u32 read (FC03 reg 7~8) 는 §4.3.3 word-LE 준수 (하위 워드가 낮은 주소).

RS-485 wire 상에서 캡처한 프레임을 표준 마스터 tester 에 던져도 "unknown function code" 나 "malformed frame" 은 나오지 않는다.

### 9.2 사설 값은 표준이 명시적으로 남긴 자리에

- `addr = 247` — §4.1 이 명시하는 유니캐스트 상한. 유효 범위 내.
- `reg = 250` — 부속서 A.1.2 미할당 영역 (표준 맵은 202~292, 501~598 등을 지정, 250 은 어디에도 없음). Vendor-specific 사용이 명시적으로 허용되는 영역.

### 9.3 회사코드 = 0

우리 슬레이브는 NodeInfo (reg 1~8) 응답 시 `company_code = 0` 를 선언 (default map self-declare). 표준 §6.1 흐름을 따르는 외부 마스터는 이 응답을 보고 "이 노드는 부속서 A 디폴트 맵을 따르는 노드" 로 판단하고 표준 맵대로 (센서 값 201~ 등) 통신을 이어간다.

우리 discovery 는 이 회사코드 표기를 건드리지 않는다.

### 9.4 GPIO 신호는 표준 트래픽 밖

Discovery 신호선 (마스터 GPIO ↔ 슬레이브 GPIO7) 은 RS-485 A/B 와는 **완전히 별개의 물리 신호**다. 표준이 규정하는 "modbus RTU on RS-485" 위에 어떤 프레임도 얹지 않으며, 표준 시험 장비 관점에서는 존재조차 하지 않는 신호.

이 신호가 없다고 해서 슬레이브가 표준 통신을 못 하는 것도 아니다 — Tier 2 (Kconfig 고정) 로 flash 하면 GPIO7 이 완전히 무연결 상태여도 정상 동작. 외부 마스터가 표준 시퀀스로 문제없이 물릴 수 있다.

### 9.5 §5.1 이 열어놓은 "자체 방식"

§5.1 의 문구는 "DIP 스위치, 설정 명령, KS X 3286 자동등록 **등** 자체 방식" — "등" 이 열거를 예시로 만든다. 표준이 규정하지 않는 자유 구간에서 프레이밍·함수코드·맵 규칙을 지키면서 편의 layer 를 추가하는 것은 §5.1 취지에 부합한다.

---

## 10. 함께 볼 문서

- **[architecture.md](architecture.md)** — 시스템 전체 계층 (firmware/gateway/backend/HMI) 상에서 discovery 가 어디에 위치하는지.
- **[vendor-spec.md](vendor-spec.md)** — 회사코드 non-zero 인 외부 vendor 노드를 마스터가 통합하는 KS X 3286 자동등록 흐름. Discovery 는 "우리 세트 내 편의", vendor-spec 은 "외부 노드 호환성 완결" 로 상보 관계.
- **[verification.md](verification.md)** — 표준 시험 매트릭스 (구도 A/B/C) 와 재현 절차. §8.3 요약 매트릭스의 전체판.

## 11. 미포함 · 후속 항목

- 슬레이브 물리 위치 표시 LED (`addr` 만큼 blink 등, 우선순위 낮음)
- 재배정 UI 버튼 (P4 HMI 에 "포트 N 재검색" 버튼 - 지금은 물리 재배선만)
- Discovery 신호선 (GPIO) 을 표준 3286 자동등록으로 대체하는 방향 — 현재는 우리 슬레이브만 대상, 외부 vendor 슬레이브도 자동 addr 부여받으려면 3286 §5.2.a 확장 논의 필요
