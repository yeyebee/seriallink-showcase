# 스택 결정 근거 (ADR)

> 각 계층을 왜 그 기술로 골랐는지, 무엇을 대안으로 놓고 저울질했는지, 그 선택이 나중에 어떤 대가로 돌아왔는지를 Architecture Decision Record (ADR) 형식으로 정리한다.

README 의 스택 요약 표는 결과만 보여준다. 이 문서는 그 결과에 도달한 과정 — 문제 배경, 후보군, 판단, 그리고 실제로 그 선택이 낳은 trade-off — 을 하나씩 남긴다. 순서는 아래에서 위로 (프로토콜 → MCU → HMI → 서버 → DB → 프론트 → 배포) 쌓아 올렸다.

각 ADR 은 아래 네 항목으로 구성된다:

- **Context** — 그 선택이 필요했던 문제 배경.
- **Options** — 진지하게 저울질한 후보들.
- **Decision** — 실제로 고른 것.
- **Consequences** — 그 선택이 낳은 결과 · trade-off · 후속 결정에 미친 영향.

특정 결정이 다른 결정을 유도한 경우 (예: DB 선택이 백엔드 async 패턴을 요구) ADR 번호로 상호 참조한다. 마지막에 참조 관계를 한 곳에 정리해 두었다.

---

## ADR-001: RS485 Modbus RTU 상 KS X 3267 표준 채택

**Context**

스마트 온실 현장은 센서 회사, 구동기 회사, 제어기 회사가 전부 다르다. 각자 자기 프로토콜을 쓰면 제어기 회사는 납품 건마다 드라이버를 새로 짜야 하고, 농가는 한 번 산 제어기에 묶인다. 이 프로젝트의 첫 번째 선택은 "우리끼리 잘 돌아가는 프로토콜을 만들 것인가, 이기종을 붙일 수 있는 표준을 지킬 것인가" 였다.

**Options**

- **자체 binary protocol** — 우리 마스터·슬레이브 조합에는 최적화 가능. 다만 타사 노드 흡수 불가.
- **OPC UA** — 산업 표준이지만 임베디드 노드 (ESP32-C3 급) 에 얹기엔 스택이 무겁다. 서버 사이드 정합성 관리 부담도 큼.
- **MQTT-only (제어기 ↔ 클라우드) + 노드는 자체 프로토콜** — 클라우드 통합은 되지만 필드 (RS485 링크) 이기종 호환이 여전히 해결 안 됨.
- **KS X 3267 (RS485 Modbus RTU)** — 국가표준. 레지스터 번호 단위까지 못 박혀 있음.

**Decision**

**KS X 3267 채택.** 부속 표준으로 3286 (자동등록 non-default map 노드), 3269 (센서 메타데이터) 도 같이 참조. 물리 계층은 표준 기본값인 9600 bps.

**Consequences**

- 이기종 상호운용 확보. 실제로 자체 slave 를 non-default map "vendor 노드" 로 재플래시하는 3286 시연 시나리오가 마스터에서 정상 동작함을 확인 (ADR-002 의 마스터 확장 참조).
- "동작한다" 와 "표준에 맞다" 가 다르다는 것을 검증 단계에서 아프게 배움 — 자체 검증 워크플로우 (docs/verification.md) 를 세우고 나서야 자체 구현의 비적합 4건 (cmd 필드 순서 역전, 응답 영역 write 미보호, 워드 인코딩, 회사코드 0xFEED) 이 드러났다.
- 표준 준수 → 인증 획득까지 확장할 여지가 열림. 대신 프로토콜 상 편의 기능 (예: 마스터 → 노드 push 알림) 은 표준이 금지 (마스터 폴링, 노드 응답 단방향) 하므로 시스템 상위층에서 구현해야 한다.
- 판정 검증 코스트가 초기 요구 수준을 상회 — "동작 확인" 이 아니라 "표준 명령 하나하나에 대한 요청/응답 바이트 대조" 를 해야 하므로, 검증자 스크립트 · 바이트 레벨 로그 · 고정값 (dummy) 응답 모드 같은 부속 도구를 함께 만들어야 했다. 이 도구들이 나중에 회귀 방지 · vendor 노드 통합 검증에도 재사용됨.

---

## ADR-002: Master MCU = ESP32-S3

**Context**

마스터는 여러 역할을 동시에 진다: RS485 두 개 버스 (센서/구동기 분리 — ADR-001 의 부산물) 폴링, Wi-Fi 로 MQTT 업링크, HMI (ADR-004) 로 향하는 UART 링크, 개발 중 모니터링용 시리얼, 나중에 붙일 옵션 인터페이스까지. UART 다중 · Wi-Fi 병행 · 여유 flash/PSRAM 이 요구 사항이다.

**Options**

- **ESP32 클래식 (WROOM-32)** — 검증된 chip 이나 flash/PSRAM 여유가 뻣뻣하고 UART 수가 빠듯.
- **ESP32-S2** — Wi-Fi 는 있지만 BT 없음, 단일 core, 대체로 부족.
- **ESP32-C3** — 저가·소형이지만 UART 수·flash 여유 부족. 마스터 규모를 감당하기엔 작다.
- **ESP32-S3** — Xtensa dual core, Wi-Fi + BT, USB-Serial-JTAG, 넉넉한 flash/PSRAM, UART 여유 충분.

**Decision**

**ESP32-S3 (N8R8, 8 MB flash / 8 MB PSRAM).**

**Consequences**

- 필요한 UART 수 (RS485 2 bus + HMI + monitor) 를 여유 있게 확보. Wi-Fi/MQTT 태스크가 폴링 태스크와 경합하지 않도록 core affinity 로 분리 가능.
- 하드웨어 상 대가: 이 보드의 CP210x flash 포트는 IO0 이 미배선되어 BOOT 버튼 홀드가 필수. 자동 리셋 회로도 DTR/RTS swap 이 필요 (블로그 §12 삽질 로그 참조). 야전에서 재플래시가 불편하지만 마스터는 갱신 빈도가 낮아 감수.
- P4 링크가 UART0 을 remap 하는 구성이라 초기 부팅 로그를 잃기 쉬움 → 진단은 monitor 포트로 분리해 얻는다.
- Dual core 활용으로 MQTT publish · Wi-Fi 재접속 로직이 RS485 폴링 timing 을 방해하지 않음. 폴링은 응답 timeout 이 짧기 때문에 (~50 ms 단위) 다른 태스크가 core 를 점유하면 즉시 상호작용에 영향이 오는데, core affinity 분리로 회피.

---

## ADR-003: Slave MCU = ESP32-C3

> **Superseded by ADR-009** (2026-08) — 노드 하드웨어 회로 통합 결정으로 신규
> 노드는 ESP32-S3 로 이관. 기존 배포된 C3 노드는 legacy 로 유지, 신규는 S3.

**Context**

슬레이브 (센서/구동기 노드) 는 마스터와 반대다. 저가·소형이 우선이고, 하나의 온실에 수십 대가 붙는다. 야전에서 재플래시할 일이 잦기 때문에 별도 flasher 없이 USB 만으로 즉시 굽는 편의가 중요하다.

**Options**

- **ESP32-C3** — RISC-V single core, Wi-Fi + BLE, USB-Serial-JTAG 내장. 시장 가격 낮음.
- **ESP32-C6** — C3 후속, Wi-Fi 6 + Zigbee. 스마트팜 노드에는 과잉 사양.
- **ESP32-S2** — Wi-Fi 는 있으나 BT 없음, 소형화 이점 미미.
- **RP2040** — 저렴하지만 Wi-Fi 미탑재 (Pico W 는 별도 chip 얹은 구성).
- **STM32 계열** — 성숙한 옵션이나 별도 ST-Link/SWD flasher 가 필요해 야전 편의가 떨어짐.

**Decision**

**ESP32-C3.** 센서/구동기/vendor/dummy 등 kind 별로 `sdkconfig.defaults.*` 프로필로 빌드 분기.

**Consequences**

- USB-Serial-JTAG 내장 덕분에 USB 케이블 한 개로 flash·monitor 가 끝난다. 유지보수 workflow 가 단순.
- USB-Serial-JTAG 의 대가: chip reset 시 USB 가 재enum 된다. Pyserial 기반 monitor 스크립트가 reset 시점에 연결이 끊어지므로, reader 는 자동 재연결 로직을 붙여야 한다. 또한 스크립트가 포트를 열 때 DTR/RTS 를 assert 하면 보드가 다운로드 모드로 빠지는 사례가 있어, 포트 오픈 **전에** DTR/RTS 를 False 로 명시하는 절차가 표준화되어 있다 (블로그 §12.3).
- 두 대의 C3 (센서/구동기) 외형이 같아 어느 보드에 어느 펌웨어가 올라갔는지 눈으로 구분이 안 됨 → 부팅 배너에 `type / mac / addr / bus / baud` 를 찍고, 30 초 주기 비콘으로 재확인하는 관례로 해결. Chip efuse MAC 이 유일 식별자이므로 포트 번호는 신뢰하지 않고 항상 배너를 먼저 읽는 규율을 세움.
- Single core RISC-V 라서 마스터 대비 여유가 없지만, 슬레이브의 역할은 "요청받은 주소의 값을 그대로 돌려주는 수동적 메모리 맵" 이라 CPU 부담이 낮음. 표준 (KS X 3267) 이 노드 능동 통신을 금지한 덕분에 이 사양이 정합.

---

## ADR-004: HMI = ESP32-P4 + LVGL 원시 (no builder)

**Context**

현장 HMI 는 800×1280 MIPI DSI 터치 패널에 30+ slave 카드, 실시간 값, 상태 stale 표시, 알람 리스트를 얹어야 한다. 화면은 리치하지만 인터랙션은 정형화되어 있음 — 카드 그리드 · 리스트 · 다이얼로그가 대부분.

**Options**

- **웹 (Chromium embedded)** — Rich UI 는 강력하나 P4 급에서 브라우저 스택 구동은 무겁고 부팅 지연이 큼.
- **Flutter Embedded** — 개발 편의 좋으나 배포 아티팩트 크기와 GPU 요구가 부담.
- **TouchGFX** — ST 생태계 위주. ESP 계열 통합 미흡.
- **Squareline Studio** — LVGL 위 GUI 빌더. 초기 생산성 높지만 flow 라는 부속 개념 학습 부담, 손으로 짠 코드와 생성 코드 간 drift 위험, 생성 시 diff 노이즈 큼.
- **LVGL 원시 (C API 직접)** — 러닝 커브 있지만 자유도 최대, 코드 review 명확.

**Decision**

**LVGL 원시.** Squareline 은 시연 프로토타입에 좋지만 장기 유지에는 손 코드가 낫다는 판단.

**Consequences**

- UI 는 코드로만 존재하므로 diff 리뷰가 깔끔하고 생성기 drift 없음.
- 카드 slot 관리 · 2-tier stale (15 s soft / 60 s hard) · flex 재정렬 등을 손으로 구현해야 함 (블로그 §14.6). 대신 이 로직들이 명시적이라 데이터가 안 오는 상황도 이벤트로 처리하는 UI 원칙이 코드에 눈에 보인다.
- Chart 렌더 시 720 point 가 쌓이면 IDLE0 태스크가 starve 되는 이슈가 발견 → 약 120 point cap + 다운샘플 + `lv_chart_refresh` 를 데이터 flush done 시점에만 호출하는 패턴으로 해결. LVGL 을 직접 다루었기에 원인 지점을 잡기 쉬웠음.
- HMI ↔ 마스터 링크는 UART. 표준 (KS X 3267) 은 HMI 를 다루지 않으므로 이 링크는 프로젝트 사설. ADR-002 의 마스터 UART 여유가 여기서 소진된다.
- Squareline 을 배제한 이 판단은 프로젝트 초기 시행착오의 결과이기도 함 — 초기에 빌더로 프로토타이핑을 시도했다가 손코드로 뒤엎을 때마다 diff 가 어지러웠고, 이후 요구 변경 (예: 카드 stale 색상 룰) 이 코드/생성기 양쪽에 걸치면 어디를 진짜로 고쳐야 하는지 판단이 흐려지는 문제가 반복됐다. 손코드 일원화 후 이 애매함이 사라짐.

---

## ADR-005: Backend = Rust (axum + sqlx)

**Context**

백엔드는 세 개 흐름을 동시에 처리해야 한다: (1) 마스터에서 올라오는 MQTT 스냅샷 ingest (수십 슬레이브 × 초 단위), (2) 프론트로 향하는 HTTP + WebSocket, (3) VAPID push, MediaMTX WHEP proxy 등 부수 서비스. 단일 VPS 배포이고, 재시작/장애 상황에서 fault-tolerant 동작이 중요.

**Options**

- **Node/TypeScript (Express/Fastify)** — 개발 속도 빠름. 다만 CPU-bound 처리 (대량 ingest 파싱, 시계열 downsample) 와 장기 실행 안정성에서 노력이 필요.
- **Go (gin/chi)** — 좋은 균형. 컴파일 · 단일 바이너리 · 동시성 성숙.
- **Python FastAPI** — DX 우수하나 async 성능 상한이 낮고 배포 시 인터프리터 · 의존성 관리 부담.
- **Rust (axum + sqlx)** — 정적 타입, async, 단일 바이너리, tokio 기반 동시성, 컴파일 타임 SQL 검증. 초기 학습 비용 큼.

**Decision**

**Rust axum + sqlx + rumqttc.** 단일 프로세스 안에서 MQTT · HTTP · WS · 백그라운드 job 을 tokio 태스크로 병렬 구동.

**Consequences**

- 배포 안정성 · 메모리 안정성이 확보됨. Panic 없이 며칠 무재시작 가능한 특성이 스마트팜 백엔드 요구와 잘 맞음.
- 초기 학습 비용 존재 (lifetime, `Send + Sync`, sqlx 매크로의 컴파일 타임 스키마 검증 등). 대신 한 번 컴파일이 통과하면 런타임에서 조용히 실패하는 사례가 극히 드묾.
- 예: `sensor_data_v2.device_id` 가 smallint FK 라서 문자열 device 코드는 `devices_dim` lookup 후 bind 해야 하는데, 초기에 lookup 을 빠뜨리자 sqlx 가 타입 mismatch 로 즉시 500 을 냈다. 조용히 넘어가지 않는 성질이 디버깅 시간을 줄임.
- 대량 DELETE 시 nginx timeout 을 넘기는 경우 async job 패턴 (`tokio::spawn` + polling status endpoint) 이 필요해짐 — 이 요건의 근본 원인은 ADR-006 (TimescaleDB) 이지만, 해결이 자연스럽게 async 언어의 강점 위에 얹혔다.
- 단일 바이너리 배포 (`Dockerfile` 안에서 multi-stage build → 작은 runtime image) 라 배포 표면이 좁음. 롤백은 이전 image tag 로 container 를 다시 띄우는 것으로 즉시 (ADR-008 참조).
- MQTT ingest · HTTP · WS · 백그라운드 purge · sensor prune · VAPID push 를 한 프로세스 안에서 tokio 태스크로 병렬 구동. 프로세스 경계를 넘지 않으므로 상태 공유 (예: `AppState.purge_jobs: HashMap`) 가 그냥 `Arc<Mutex<>>` 로 끝남 — 분산 캐시나 별도 큐 서비스가 필요 없는 규모.

---

## ADR-006: DB = PostgreSQL + TimescaleDB

**Context**

센서 값은 device × metric × timestamp 의 시계열이다. 슬레이브 20~30 대 × metric 여러 개 × 초 단위 upsert 로 수백만 row 가 금방 쌓인다. 최근 데이터 read 는 잦고, 오래된 데이터는 compression 이 필수. 동시에 관계형 모델 (`devices`, `devices_dim`, `metrics_dim`, `device_audit`) 이 함께 존재해야 함.

**Options**

- **InfluxDB** — 시계열 전문. 다만 SQL 아닌 자체 쿼리 언어, 관계 조인 약함, 외부 툴 호환 (dbeaver 등) 은 별도 플러그인.
- **ClickHouse** — 대용량 OLAP 강자. 우리 규모엔 과잉이고 관계형 write 워크로드에 최적화 안 됨.
- **plain PostgreSQL** — 관계 모델 편리하나, 초 단위 sensor row 가 쌓이면 index / partition 관리를 손으로 해야 함.
- **QuestDB** — 빠르지만 생태계 · 관리 도구 성숙도 부족.
- **TimescaleDB (Postgres extension)** — Postgres 문법 그대로. Hypertable · continuous aggregate · compression 을 확장으로 제공.

**Decision**

**PostgreSQL + TimescaleDB.** `sensor_data_v2` 를 hypertable 로, 오래된 chunk 는 compression 활성화.

**Consequences**

- Postgres 툴체인 (dbeaver, psql, sqlx, migration 툴) 그대로 사용 가능. 관계형 스키마 (`devices_dim` FK 등) 와 시계열이 한 DB 에 자연스럽게 공존.
- Compressed chunk 상 partial DELETE 가 매우 느림 (~21 만 row 에 약 178 초). 원인은 chunk 단위 decompress + scan. Nginx 60 s / 300 s timeout 을 넘어가고, 브라우저 abort → backend rollback 이라는 무한 우회 게임이 벌어짐.
- 이 문제가 ADR-005 의 async job 패턴을 낳은 직접 원인이다: sync DELETE 를 buttress 하는 대신 endpoint 를 즉시 `202 Accepted + job_id` 로 답하고 실제 삭제는 `tokio::spawn` 안에서 진행, 프론트는 status endpoint 를 polling. Timeout 을 늘리는 대신 pattern 을 바꾼 사례.
- Metric key 표준화 (예: `soil_ec` → `ec`) 같은 스키마 정리는 SQL migration 으로 안전하게 반영. KS X device code 대응이 명확해지면서 UI 카드와 DB 값의 의미가 어긋나던 잔여 debt 를 정리.
- Continuous aggregate 는 아직 도입 시점이 아님 — 현 규모에서는 raw hypertable + BRIN 유사 인덱스 조합이 충분히 빠르다. 데이터가 더 쌓이면 hourly / daily rollup 을 continuous aggregate 로 얹어 UI 쿼리를 갈아 끼우는 정도로 확장 가능.
- Metric 통합 축소 API (sensor prune) 는 SQL 함수 하나로 3-모드 (stale / 기간 / 전체) 를 처리 — Postgres 함수를 자산으로 가져갈 수 있는 점이 장기적으로는 개발 속도 이점.

---

## ADR-007: Frontend = SvelteKit (adapter-static)

**Context**

프론트는 실시간 카드 30+ 개, WebSocket 스트리밍, KsxVerify widget, device-audit 목록, addr/alias inline 편집 등 SPA 성격. 서버 사이드 렌더링 요구는 없다 (인증 뒤 관리 화면). 배포는 nginx 정적 서빙이면 충분.

**Options**

- **Next.js (React)** — 프레임워크 성숙, 커뮤니티 크지만 SSR 이 우리에겐 dead weight, 번들 크기 부담.
- **Nuxt (Vue)** — 위와 비슷한 이유.
- **plain React SPA (Vite)** — 자유롭지만 라우팅 · 데이터 로딩 · form 처리 등을 손으로 얹어야 함.
- **SvelteKit** — reactive 문법이 콤팩트 (변수 = 상태), 파일 기반 라우팅, adapter-static 으로 순수 정적 빌드 가능. TypeScript 기본 지원.

**Decision**

**SvelteKit + adapter-static.** 빌드 산출물을 nginx 로 서빙, API 는 리버스 프록시로 backend 에 연결.

**Consequences**

- 소규모 팀 (혹은 1인) 유지 부담이 낮음. Reactive 문 (`$: derived`, store) 이 카드 그리드 + WS 스트리밍 조합에 잘 맞음.
- 정적 빌드라 배포는 rsync 로 끝. nginx cache 만 신경 쓰면 롤백도 단순.
- TypeScript 타입 안정성 확보. 컴포넌트 간 prop 계약이 명시적이라 카드 리팩터 시 breakage 를 컴파일 시점에 잡음.
- Alias inline 편집, 3-mode sensor prune panel 같은 UX 세부는 컴포넌트 재사용 (`SensorPrunePanel`) 으로 감당 — 프레임워크가 강제하지 않지만 SvelteKit 의 콤팩트한 컴포넌트 스타일이 이런 재사용을 자연스럽게 유도.
- 30+ 카드가 초 단위 WS 스트림으로 갱신되는 상황에서도 reactivity 오버헤드가 체감되지 않음 (Svelte 컴파일 시점 반응성). React 였다면 `useMemo` / `React.memo` 튜닝을 손으로 얹었을 자리들이 그냥 안 필요.
- 개발 프로젝트라 인력이 팀 규모 프레임워크 (Next 등) 의 관례를 따라야 할 필요가 없음 — 관례 학습보다 문제 해결에 시간을 쓰는 방향으로 프레임워크 선택이 정합.

---

## ADR-008: 배포 = Docker (host network) + nginx

**Context**

단일 VPS 위에 (a) Rust backend, (b) MediaMTX (WHEP), (c) PostgreSQL/TimescaleDB, (d) mosquitto MQTT broker, (e) nginx 를 함께 돌린다. 팀 규모와 서비스 개수 대비 오케스트레이션 오버헤드는 최소화하고 싶다.

**Options**

- **bare metal + systemd** — 가장 단순. 다만 backend 의 Rust 빌드 산출물이 시스템 lib 의존성을 갖는 경우 배포마다 조율 필요.
- **docker-compose** — 여러 컨테이너 orchestration 편의. 단, Postgres/mosquitto 는 기존 bare metal 운영 자산과 이미 물려 있어 컨테이너화 시 데이터 이관 리스크.
- **k3s / Kubernetes** — 완전 오버킬. 관리 대상 리소스 · 학습 비용이 이 스케일에 맞지 않음.
- **Docker (host network) 부분 도입** — backend 만 컨테이너로. DB · broker 는 bare metal 유지. Nginx 도 시스템 서비스로.

**Decision**

**Backend 만 Docker (host network mode).** Postgres · TimescaleDB · mosquitto 는 bare metal. Nginx 는 시스템 서비스 (`/etc/nginx/`) 이되 conf 는 monorepo source-of-truth 를 유지하고 VPS 는 symlink.

**Consequences**

- Backend 재배포 흐름: 로컬 → rsync → 원격 `docker build` → container replace + smoke test. 롤백은 이전 image tag 재기동으로 즉시.
- Host network mode 라서 backend 는 `127.0.0.1:8000` 를 그대로 bind, nginx 는 loopback 으로 프록시. 별도 bridge 네트워크 관리 없음.
- Secret (DB URL, MQTT 자격, VAPID 키 등) 은 배포 스크립트가 heredoc 으로 env 를 주입하고, 스크립트 자체는 `.gitignore`. 저장소 안에 secret 은 어디에도 커밋되지 않음.
- Nginx conf 를 monorepo 에 두고 symlink 로 시스템에 반영하는 방식은 conf 변경 이력을 git 이력으로 관리하게 해 준다. `nginx -t` 후 `reload` 는 스크립트화.
- 이름과 코드의 사실 불일치를 정리하는 사이클 (예: 폴더/컨테이너/이미지 rename) 은 downtime 30 초 안팎으로 감당 가능 — Docker 를 부분 도입한 덕분에 정리 스텝이 script 로 표현 가능.
- MediaMTX WHEP 프록시도 nginx 아래로 통합 (동일 origin) 하여 브라우저 CORS · 인증 조합을 단순화. Backend 와는 별개 컨테이너 · 별개 포트지만, 외부 노출은 nginx location 규칙 하나로 통일.
- 부분 Docker 는 이관 리스크와 오케스트레이션 오버헤드 사이의 실용적 타협 — DB · broker 를 컨테이너로 옮기는 순간 데이터 볼륨 · 백업 · restore 절차를 다시 짜야 하는데, 그 이득이 이 규모에선 없다. 필요 시점 (예: 인스턴스 증설) 에 다시 검토할 여지는 남겨 둠.

---

## ADR-009: 노드 하드웨어 통합 = 마스터 회로 재사용 (S3)

*2026-08. ADR-003 (Slave MCU = C3) 을 supersede.*

**Context**

노드 하드웨어 (센서 캐리어 PCB) 는 그동안 마스터와 별개 회로로 취급했다. 마스터 = ESP32-S3 (ADR-002), 슬레이브 = ESP32-C3 (ADR-003). 그런데 실측 조사 상 몇 가지 사실이 확정됐다:

- 마스터 PCB 상 RS485 트랜시버 · 바이어스 회로 · UART pin 배치는 슬레이브 role 로도 무결하게 동작 (raw slave test 10/10, esp-modbus slave 9/10).
- 슬레이브 fw 의 핵심 로직 (Modbus RTU raw parser · discovery FSM · 외부 sensor UART master) 은 SoC-independent 순수 C — arduino-esp32 로도 재이식이 자연스러웠음.
- 노드 PCB 를 마스터 회로로 재사용하면 BOM/검증/재고 부담이 절반이 되고, chip 계열 통합으로 flash 도구 · 개발 workflow 단일화 이득.

**Options**

- **분리 유지 (C3 슬레이브 별개 회로)** — 소형·저가 이점은 있으나, 이번 조사 상 마스터 회로가 이미 슬레이브 사양을 초과 만족. 재고 이점보다 두 계열 유지 부담이 큼.
- **슬레이브 회로 = 마스터 회로 재사용 (S3)** — 하드웨어 트랜시버/pin 배치 이미 검증, fw port 는 sdkconfig pin/target 조정 수준. 회로 · fw · 도구 통합.
- **완전 신설 (다른 SoC 로 재출발)** — 검증 비용 원상 복귀. 지금까지 축적된 pin 배치 · 트랜시버 회로 · fw 로직 무효.

**Decision**

**신 노드 chip = ESP32-S3, 마스터 PCB 회로 재사용.** 슬레이브 fw 는 별도 프로젝트 (`slave-s3-sensor`) 로 clone, sdkconfig 상 target/pin 조정. mb_slave.c · discovery.c · ec_sensor.c 등 순수 C 로직은 재컴파일만.

Kind 별 sdkconfig 프로필 (`sdkconfig.defaults.{co2,ec,soil,vendor}`) 은 그대로 상속 — 배포 시 kind 하나만 선택 후 flash.

**Consequences**

- **PCB 회로 단일화** — 마스터/슬레이브가 동일 회로. 신 노드 PCB 는 마스터 회로도 발췌 + 슬레이브 특성 (예: sensor 커넥터, I2C 헤더) 만 추가. BOM · 조립 · 검증 workflow 단순화.
- **Fw 로직 두 계열 → 한 계열 (SoC-agnostic 유지)** — 핵심 로직은 SoC-independent 순수 C 로 유지, kind/pin 만 sdkconfig 분기. 향후 slave-c3-sensor 폴더는 slave-s3-sensor 로 통합 rename 가능 (legacy 배포 유지 목적으로 당분간 분리).
- **Flash · 개발 도구 통합** — S3 는 USB-Serial-JTAG 내장 (C3 와 동일 workflow 유지). flash-slave 스크립트 · idf.py workflow 그대로. `--chip esp32s3` 인자만 조정.
- **기존 배포 C3 노드는 legacy 유지** — 신규 노드부터 S3. 마이그레이션은 자연 감가상각 방식.
- **하드웨어 배치 실측 확인 사항** — 노드 fw 는 마스터 fw 상 P4 HMI 통신 pin (UART0 remap) 을 I2C 로 재사용 가능 (fw layer 상 pin 용도만 다름). PCB 배선은 그대로 두어도 fw 상 재구성.
- **`slave-c3-sensor` 상 발견한 fw 계열 공통 잔재 버그** (예: Kconfig 옵션 참조는 있으나 등록 누락 되어 있어 sdkconfig 상 명시해도 silent no-op 하던 것) 는 이 통합 계기로 slave-base Kconfig 에 정리 반영.

---

## ADR 간 상호 참조 요약

- **ADR-001 (KS X 3267)** → 마스터 확장 (KS X 3286 vendor 노드) 이 ADR-002 의 마스터 여유 사양에서 실현.
- **ADR-002 (S3 마스터)** ↔ **ADR-004 (P4 HMI)** — UART 링크 요구가 S3 UART 수 여유를 근거로 성립.
- **ADR-002 (S3 마스터)** → **ADR-009 (노드 통합)** — 마스터 회로 무결성 실측이 노드 재사용의 근거.
- **ADR-003 (C3 슬레이브)** *(Superseded by ADR-009)* — 야전 flash 편의를 위해 USB-Serial-JTAG 채택, 그 관례는 S3 로 이관 후에도 그대로 유지 (S3 도 USB-Serial-JTAG 내장).
- **ADR-005 (Rust)** ← **ADR-006 (TimescaleDB)** — Compressed chunk DELETE 지연이 async job 패턴을 요구, tokio task 로 자연스럽게 흡수.
- **ADR-007 (SvelteKit static)** → **ADR-008 (Docker + nginx)** — 정적 산출물을 nginx 로 서빙, backend 만 컨테이너화하는 배포 모델의 절반을 담당.
- **ADR-008 (부분 Docker)** — 기존 bare metal 자산 (Postgres/mosquitto) 을 그대로 두고 backend 만 옮긴 pragmatic 선택. 오케스트레이션 대신 스크립트 + git 이력으로 관리.
- **ADR-009 (노드 통합)** — 마스터 PCB 검증이 노드 회로 결정의 근거로 pull-through. Chip 통합으로 fw workflow · flash 도구 · 재고가 단일화.

---

*이 문서는 블로그 발행 시점의 스냅샷이다. 이후 결정이 뒤집힌 항목이 생기면 개별 ADR 을 "Superseded by ADR-NNN" 으로 표시하고 새 ADR 을 붙이는 방식으로 이력을 남긴다.*
