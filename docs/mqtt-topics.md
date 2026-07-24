# MQTT topic 스키마

마스터(게이트웨이)와 백엔드가 주고받는 모든 MQTT topic·payload 계약을 한 파일에 요약한다 — 표준(KS X 3267 / 3286) 밖 영역이므로 여기가 유일한 소스다.

관련 문서
- `architecture.md` — 큰 그림, 컴포넌트 배치
- `discovery.md` — audit event 가 어떻게 만들어지는지
- `vendor-spec.md` — `vendor_model` 필드가 붙는 근거
- `verification.md` — cmd/ack 를 활용한 표준 적합성 검증 절차

---

## 1. 개요

| 항목 | 값 |
|---|---|
| 브로커 | mosquitto (host 프로세스, backend 는 클라이언트로만 참여) |
| 접속 | `mqtt://<broker-host>:<port>` (운영 환경에서는 TLS) |
| 구독 (backend) | `#` — 라우팅은 topic parser 로 분기 |
| 구독 (master) | `smartfarm/controller/<ctrl>/cmd/#` |
| 기본 QoS | 시계열/이벤트 = **1**, cmd/ack = **0** (재시도는 상위 layer 몫) |
| retain | `esp32/ip` (master 접속 시 자기 IP, retain=1) 외에는 **모두 0** |
| outbox 상한 (master) | 64 KB — 절단 시 최신 데이터 우선, 초과분 drop |
| 최대 packet | 1 MB (backend `set_max_packet_size`) |

토픽 prefix 는 마스터 펌웨어 `uplink_cfg_t.topic_prefix` 로 주입되며, 공개 스냅샷에서는 `smartfarm/controller/<ctrl>` 형태를 표준으로 삼는다. `<ctrl>` 은 컨트롤러 논리 ID (예: `1`) 이고, 백엔드는 heartbeat 를 받으며 이 값을 `C-{MAC}` device_id 로 매핑해 둔다.

---

## 2. 토픽 트리 요약

```
smartfarm/
└── controller/
    └── <ctrl>/                             # 컨트롤러 논리 ID (heartbeat 로 device_id 에 매핑)
        ├── heartbeat                       # ↑ 마스터 → 백엔드 (5 s)
        ├── discovery                       # ↑ 슬레이브 lifecycle event
        ├── cmd/#                           # ↓ 백엔드/오퍼레이터 → 마스터
        ├── ack                             # ↑ 마스터 → 요청자 (cmd 결과)
        └── bus/<bus>/node/<addr>/
            └── snapshot                    # ↑ 슬레이브 스냅샷 (5 s, 노드마다)

esp32/ip                                    # ↑ 마스터 접속 시 자기 IP (retain=1)
```

| # | Topic | 방향 | QoS · retain | 페이로드 | 주기 |
|---|---|---|---|---|---|
| 1 | `.../heartbeat` | M → B | 1 · 0 | 컨트롤러 상태 JSON | 5 s |
| 2 | `.../bus/<bus>/node/<addr>/snapshot` | M → B | 1 · 0 | 노드 스냅샷 JSON | 5 s (노드 별) |
| 3 | `.../discovery` | M → B | 1 · 0 | audit event JSON | 이벤트 발생 시 |
| 4 | `.../cmd/#` | B → M | 0 · 0 | 명령 JSON (`cmd` 필드) | 요청 시 |
| 5 | `.../ack` | M → B | 0 · 0 | ack JSON (`id` echo) | cmd 도착 시 |
| 6 | `esp32/ip` | M → B | 1 · **1** | IP 문자열 (`192.0.2.10`) | 접속 이벤트 |

M = 마스터, B = 백엔드.

---

## 3. Topic 상세

### 3.1 `smartfarm/controller/<ctrl>/heartbeat`
컨트롤러가 살아있음을 알리는 최소 신호. 등록 슬레이브가 0개여도 무조건 발행되므로 "게이트웨이는 살아있는데 슬레이브만 안 잡히는" 상황을 즉시 구분할 수 있다.

```json
{
  "seq": 421,
  "uptime_s": 12345,
  "heap": 148320,
  "nodes": 4,
  "mac": "3C0F02CFF514",
  "chip": "esp32s3"
}
```

- 백엔드 처리: `mac` (12자 hex) → `device_id = "C-{mac}"` 로 `devices` 테이블 upsert (device_type = `smart_farm_controller`, `channel_count = nodes`).
- 사용자 필드 (`alias`, `location`, `owner`) 는 보존.
- 이 시점의 `<ctrl> ↔ device_id` 매핑은 in-memory `ctrl_map` 에 저장되어, 이후 snapshot ingest 에서 `parent_controller_id` 를 채우는 데 쓰인다.

### 3.2 `smartfarm/controller/<ctrl>/bus/<bus>/node/<addr>/snapshot`
슬레이브 하나 분량의 노드 정보 + 최신 센서·구동기 값. `bus_id` 별로 같은 `addr` 가 공존할 수 있으므로 토픽에 `bus/<bus>` 를 명시한다.

```json
{
  "bus": 2, "slave": 3,
  "type": 1, "code": 4, "company": 1234, "proto": 1, "ch": 4,
  "serial": 3555045960,
  "reachable": true,
  "errors": 0,
  "vendor_model": "ACME-TH20-4",
  "node":  { "st": 0, "opid": 12, "ctrl": 0 },
  "sensors": [
    { "slot": 0, "dev": 1,  "inst": 1, "v": 24.7,   "st": 0 },
    { "slot": 1, "dev": 2,  "inst": 1, "v": 61.3,   "st": 0 },
    { "slot": 2, "dev": 14, "inst": 1, "v": 0.412,  "st": 0 }
  ],
  "actuators": [
    { "idx": 0, "dev": 202, "opid": 45, "st": 12, "remain": 180 }
  ],
  "nutrient": { "st": 0, "area": 1, "alert": 0, "opid": 0, "remain": 0 }
}
```

- 필수 키: `bus`, `slave`, `type`, `serial`. 나머지는 노드 종류에 따라 optional.
- `type` = KS X product_type (1=sensor, 2=actuator, 3=integrated).
- `dev` = KS X 3267 §A.1.2 device code (예: 1=temperature, 2=humidity, 11=co2, 14=soil_moisture).
- 백엔드는 `serial` → `device_id = "S-{serial:08X}"` 로 슬레이브 자체를 device 로 upsert 하고, `sensors[]` 중 `st == 0` (READY) 인 것만 metric 화하여 `sensor_data_v2` 하이퍼테이블에 insert.

### 3.3 `smartfarm/controller/<ctrl>/discovery`
`master_discovery` 가 slave 라이프사이클 판정을 내릴 때마다 발행. 자세한 배정 로직은 `discovery.md` 참고.

```json
{ "event": "added",    "serial": "0x8d23ac5c", "bus": 2, "addr": 1, "from_port": 0, "to_port": 1, "ts_ms": 12345 }
{ "event": "moved",    "serial": "0x8d23ac5c", "bus": 2, "addr": 5, "from_port": 3, "to_port": 5, "ts_ms": 45678 }
{ "event": "removed",  "serial": "0x8d23ac5c", "bus": 2, "addr": 3, "from_port": 0, "to_port": 0, "ts_ms": 45900 }
{ "event": "reassign", "serial": "0x8d23ac5c", "bus": 2, "addr": 3, "from_port": 0, "to_port": 3, "ts_ms": 46200 }
```

- 백엔드는 `migration_dev.device_audit` 에 INSERT 하고, WebSocket 으로 `{type:"discovery_audit", ...}` 를 브로드캐스트한다 (UI 배지·토스트 갱신).

### 3.4 `smartfarm/controller/<ctrl>/cmd/#`
백엔드/오퍼레이터 → 마스터 명령 채널. **subtopic 은 자유** (예: `.../cmd/mb_write`) — 마스터는 payload 의 `cmd` 필드로만 분기한다. `id` 를 넣어 두면 ack 에 그대로 echo 되므로 프런트엔드가 correlation 용으로 쓴다.

지원 `cmd`:

| cmd | 의미 | 필수 필드 |
|---|---|---|
| `ping` | 헬스체크 | — |
| `mb_read` | Holding registers (FC 03) 읽기 | `bus`, `addr`, `reg`, `count` |
| `mb_write` | Multiple registers (FC 10) 쓰기 | `bus`, `addr`, `reg`, `values[]` |
| `mb_write1` | Single register (FC 06) 쓰기 | `bus`, `addr`, `reg`, `value` |
| `mb_scan` | 주소 range 노드 정보 스캔 | `bus`, `from`, `to` |
| `chart_real` | P4 HMI 차트에 실 데이터 push | `device`, `hours`, `bucket_s` |
| `chart_demo` | P4 HMI 차트에 dummy 시계열 push | `points` |

`bus` 기본값은 1, 알 수 없는 `cmd` 는 `{"ack":"unknown","id":"..."}` 로 회신된다.

### 3.5 `smartfarm/controller/<ctrl>/ack`
모든 cmd 결과가 이 하나의 토픽으로 돌아온다 — cmd 서브트리 밖에 두어야 마스터가 자기 응답을 자기 구독으로 되받는 루프를 막을 수 있다.

```json
{ "ack": "mb_write", "id": "op-4711", "ok": true,
  "addr": 3, "reg": 503, "n": 4 }

{ "ack": "mb_read", "id": "op-4712", "ok": true,
  "addr": 3, "reg": 0, "regs": [1, 2, 1234, 5678, 0, 0, 0, 0] }

{ "ack": "mb_write", "id": "op-4713", "ok": false,
  "addr": 3, "reg": 503, "n": 4, "err": "ESP_ERR_TIMEOUT" }
```

### 3.6 `esp32/ip` (레거시 dev 편의)
마스터가 접속 즉시 자기 IP 를 문자열로 publish (retain=1). 백엔드 dev 편의용이며 시계열 파이프라인과 무관.

---

## 4. Backend ingest 흐름

```
mosquitto ──► rumqttc EventLoop ──► topic parser
                                     │
                                     ├── heartbeat  ─► ingest_controller_heartbeat
                                     │                  └─ devices upsert (C-{MAC})
                                     │                  └─ ctrl_map <ctrl> → C-{MAC}
                                     │
                                     ├── snapshot   ─► ingest_node_snapshot
                                     │                  └─ devices upsert (S-{serial})
                                     │                  └─ sensor_data_v2 insert
                                     │                  └─ alarm + auto-control 평가
                                     │
                                     └── discovery  ─► ingest_discovery_audit
                                                        └─ device_audit insert
                                                        └─ WebSocket broadcast
```

- topic 판정은 `is_ks_snapshot_topic` / `is_controller_heartbeat_topic` / `is_discovery_audit_topic` 세 함수로 시작 — 형식이 어긋나면 조용히 drop.
- snapshot ingest 시 `st != 0` (READY 아님) 인 센서는 스킵.
- `parent_controller_id` 는 이전에 heartbeat 가 매핑을 채워둔 경우에만 설정, 없으면 다음 스냅샷에서 채워진다.

---

## 5. KS X 3286 `vendor_model`

snapshot payload 에 `vendor_model` (문자열) 이 붙어 있으면, 백엔드는 그 값을 슬레이브 device 의 default `alias` 로 사용한다 (`ACME-TH20-4` 등 사람이 읽기 좋은 모델명). 사용자가 UI 에서 alias 를 이미 지정했다면 보존된다 (`COALESCE(devices.alias, EXCLUDED.alias)`).

- 필드는 마스터가 `ks3286_find_node_spec(company, code)` 로 model_name 을 찾은 경우에만 실린다.
- 없으면 백엔드는 `"센서 BUS#{bus} 슬레이브{slave}"` 형태의 로컬 default alias 를 생성.

`company` / `code` / `vendor_model` 매칭 규칙 전체는 `vendor-spec.md` 참고.

---

## 6. 폴링·발행 주기

| 채널 | 기본 주기 | 결정 위치 |
|---|---|---|
| 컨트롤러 heartbeat | 5 s | 마스터 `uplink_task` |
| 노드 snapshot | 5 s (등록 노드마다) | 마스터 poller · uplink |
| discovery event | 이벤트 발생 시 (bursty) | `master_discovery` 판정 순간 |
| cmd/ack | 요청 시 | 백엔드 / 오퍼레이터 |

주기는 펌웨어 config 로 조정 가능하며, 백엔드는 주기에 의존하지 않는 upsert 만 수행한다.

---

## 7. 참고 — `mosquitto_pub` 로 KS X 검증

표준 적합성 검증 시 `verification.md` 는 마스터 UART 로그를 기다리지 않고 `mosquitto_pub` 로 직접 `mb_write` 를 쏘는 경로를 권장한다. 예시:

```bash
# 슬레이브(BUS 2, addr 3) 채널 1 스위치 ON — KS X op = 201, opid = 42
mosquitto_pub -h <broker-host> -p <port> \
  -t 'smartfarm/controller/1/cmd/mb_write' \
  -m '{"id":"verify-01","cmd":"mb_write","bus":2,"addr":3,"reg":503,"values":[201,42,0,0]}'

# ack 구독 (다른 터미널)
mosquitto_sub -h <broker-host> -p <port> \
  -t 'smartfarm/controller/1/ack' -v
```

- KS X 표준 cmd block 순서는 `values[0]=op_code`, `values[1]=opid`, `values[2..3]=arg(u32 LE word)`.
- `opid` 는 이전 명령의 `opid` 와 달라야 슬레이브가 동일 명령 두 번을 silently drop 하지 않는다 (표준 §A.2 semantics).
- 백엔드 REST 를 통하는 경우 `POST /api/mqtt/publish` 로 같은 payload 를 전송할 수 있다 (내부적으로 위와 동일한 publish 를 수행).
