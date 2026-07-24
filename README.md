# SerialLink — KS X 3267 스마트팜 통합 제어기

> 스마트 온실용 **KS X 3267** (RS485 Modbus RTU) 표준 통합 제어기의 마스터·슬레이브·HMI·백엔드
> 를 직접 구현하고 **표준 적합성을 전수 검증**한 기록의 공개 스냅샷입니다.

**개발 프로젝트**. 이 저장소는 실 개발 저장소(사설) 의 아키텍처·설계·블로그 원고를
독자가 참조할 수 있도록 정리한 공개 스냅샷이며, 실 소스와 운영 좌표는 포함하지 않습니다.

---

## 무엇을 만들었나

**하드웨어 — 3-tier 구성**

- **Master (ESP32-S3)** — RS485 마스터. KS X 3267 프로토콜, 슬레이브 폴링, MQTT 업링크, HMI 링크.
- **Slave (ESP32-C3)** — 센서·구동기 노드. 표준 default map 준수. 센서 kind (CO2 / EC / soil / dummy / vendor) 별 sdkconfig 프로필.
- **HMI (JC-ESP32P4-M3)** — LVGL 기반 800×1280 터치 UI, UART 링크로 master 와 연결.

**서버 — Rust + SvelteKit**

- **Backend**: Axum + sqlx + TimescaleDB + MQTT ingest + VAPID push
- **Frontend**: SvelteKit static build (30+ cards, 실시간 WS, KsxVerify widget)

**표준 스택**

- **KS X 3267**: RS485 Modbus RTU 인터페이스 (2022). 센서·구동기·제어기 프로토콜.
- **KS X 3286**: 자동등록 노드 (vendor spec 기반 non-default map). 마스터 통합 완료.
- **KS X 3269**: 센서 메타데이터 (unit, range). 참조.

---

## 검증 결과

- 디폴트 레지스터 맵 범위 표준 명령 **전수 PASS**
- 검증 과정과 직후 재검토에서 **자체 구현의 표준 비적합 4건 발견·수정 완료**
  - #1 cmd 블록 필드 순서 역전
  - #2 응답 영역 write 미보호
  - #3 워드 인코딩
  - #4 회사코드 0xFEED (검증 완료를 넘긴 시점 발견)
- **KS X 3286 시연 시나리오** 정상 동작 (vendor spec 노드 통합)

상세: [docs/verification.md](docs/verification.md) · [docs/vendor-spec.md](docs/vendor-spec.md)

---

## 저장소 구성

```
seriallink-showcase/
├── README.md                # ← 지금 이 파일
├── LICENSE                  # MIT
├── blog-post.md             # 블로그 원고 (전 과정 서사 · 14장 · 약 1200 lines)
├── docs/
│   ├── architecture.md      # 시스템 전체 아키텍처
│   ├── verification.md      # KS X 3267 전수 검증 방법론·결과·비적합 4건
│   ├── discovery.md         # 사설 확장 §5.1 — per-port GPIO + 3-tier addr fallback
│   ├── vendor-spec.md       # KS X 3286 자동등록 노드 통합
│   ├── operations.md        # 검증 이후 운영 편의성 (async purge · sensor prune · addr · alias)
│   ├── stack-decisions.md   # ADR (스택 선택 근거 — Rust/Timescale/SvelteKit/LVGL 등)
│   └── mqtt-topics.md       # MQTT topic 스키마 + payload 예시 + backend ingest 흐름
└── firmware/
    └── README.md            # 펌웨어 컴포넌트 요약 (master/slave/p4)
```

---

## 어디부터 읽으면 좋을까

**블로그 서사부터** — [blog-post.md](blog-post.md) 는 왜 표준부터 봤는지 → 프로토콜 기초 →
센서/구동기 구현 → 검증에서 깨진 것들 → 삽질 로그 → 검증 이후 편의성 강화까지 시간순서로
읽히도록 씀. 특정 부분만 궁금하면 아래 문서로 바로:

| 관심사 | 문서 |
|---|---|
| 전체 그림 (하드웨어 + 서버 + 데이터) | [docs/architecture.md](docs/architecture.md) |
| KS X 3267 준수 어떻게 검증했나 | [docs/verification.md](docs/verification.md) |
| 슬레이브 자동 배정 (사설 확장) | [docs/discovery.md](docs/discovery.md) |
| KS X 3286 vendor 노드 통합 | [docs/vendor-spec.md](docs/vendor-spec.md) |
| 검증 이후 운영 편의성 UI | [docs/operations.md](docs/operations.md) |
| 스택 선택 근거 (ADR) | [docs/stack-decisions.md](docs/stack-decisions.md) |
| MQTT topic 스키마·payload | [docs/mqtt-topics.md](docs/mqtt-topics.md) |
| 펌웨어 컴포넌트 구조 · 빌드 | [firmware/README.md](firmware/README.md) |

---

## 스택 결정 요약

| 계층 | 선택 | 이유 |
|---|---|---|
| RS485 프로토콜 | KS X 3267 Modbus RTU (9600 bps) | 표준 준수 → 이기종 노드 상호운용 (KS X 3286 vendor 노드까지) |
| Master MCU | ESP32-S3 | 여유 flash·PSRAM + Wi-Fi + UART 다중 (RS485 2 bus + HMI + monitor) |
| Slave MCU | ESP32-C3 | 저가·소형·USB-Serial 내장 (야전 flash 수월) + i2c/UART 필요 요건 충족 |
| HMI | ESP32-P4 + LVGL | 800×1280 MIPI DSI 지원 · LVGL 안정 · UART 로 master 와 링크 |
| Backend | Rust (axum) + sqlx | 단일 바이너리·async·정적 타입·MQTT+WS+HTTP 동시 · fault-tolerant |
| DB | PostgreSQL + TimescaleDB | 센서 시계열 hypertable + compression + continuous aggregate |
| Frontend | SvelteKit (adapter-static) | 정적 파일 배포 (nginx 서빙) + reactive · 소규모 팀 유지 부담 최소 |
| 배포 | Docker (host network) + nginx | reverse proxy · TLS · static + api + WS + MediaMTX WHEP 통합 |

---

## 이 저장소의 성격

이 저장소는 **참조·인용 목적**입니다. Clone 해서 그대로 돌리는 튜토리얼이 아닙니다:

- 실 firmware 빌드는 별도 사설 저장소에 있음 (독자가 재현하려면 실 하드웨어 + KS X 인증 노드 필요)
- 서버 코드는 발췌만 (실 배포 좌표·자격증명·secret 은 포함하지 않음)
- 데이터베이스/MQTT topic 예시는 스키마 수준만

원본 사설 저장소는 활발히 개발 중이며, 이 저장소는 **블로그 발행 시점의 스냅샷**입니다. 이후
갱신이 있으면 태그 (`v1.x-blog`) 로 별도 스냅샷을 남깁니다.

---

## 인용

블로그·논문·기술문서에서 이 저장소를 인용하려면 아래처럼:

```
SerialLink — KS X 3267 스마트팜 통합 제어기 (2026).
https://github.com/yeyebee/seriallink-showcase (tag: v1.0-blog)
```

---

## 라이선스

[MIT](LICENSE). 개발 프로젝트.
