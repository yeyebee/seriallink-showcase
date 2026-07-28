# firmware/ — 컴포넌트 요약

KS X 3267 표준 스마트팜 통합 제어기의 펌웨어 구조 요약. 실 소스는 사설 저장소에 있고,
이 문서는 아키텍처 이해용 발췌입니다.

## 구조

```
firmware/
├── components/
│   └── ks_protocol/             # 공유 component (ks_protocol.h + ks_codec.h/.c)
├── p4/                          # HMI: JC-ESP32P4-M3 (LVGL, MIPI DSI 800×1280)
├── master/                      # Master 컨트롤러 (RS485, ESP32-S3)
├── slave-base/                  # 슬레이브 공유 source pool ★
├── slave-c3-sensor/             # ESP32-C3 sensor 노드 (slave-base symlink)
└── slave-c3-act/                # ESP32-C3 actuator 노드 (slave-base symlink)
```

### components/ks_protocol/

KS X 3267 프로토콜 정의 + Modbus RTU codec. `master/` 와 `slave-base/` 가 같은 source 참조 → drift 방지.

초기에는 두 곳에 복사본을 두었다가 주석 등이 어긋나기 시작해 (md5 불일치 발견) component 로 추출해 source-of-truth 1곳으로 통일.

### slave-base 의 역할 (symlink pool)

`slave-base/main/` 상 17개 source (`ks_protocol.h`, `ks_codec.c`, `ks_actuator_node.c` 등) 가 공유 pool.
`slave-c3-{sensor,act}/main/` 상 동일 이름 파일들은 **symlink** 로 `../../slave-base/main/*` 를 가리킴.
variant 별로 `sdkconfig.defaults` (핀맵) + `CMakeLists.txt` 만 별도 보유.

빌드 시 변형 디렉토리에서 `idf.py build` 하면 symlink 따라 slave-base 의 source 를 사용.

## Master-driven per-port discovery

Slave addr 을 컴파일 타임에 하드코딩하지 않고, **master 가 신 PCB 의 discovery GPIO (GPIO1~5)
를 순차 activate 하며 각 port 에 응답한 slave 에 addr 을 배정**. 물리 포트 번호 = slave addr =
UI CH 번호 로 자동 매핑.

- **표준 준수**: KS X 3267 §5.1 상 addr 부여 방식은 자유. §4.1 유니캐스트, §4.3.3 word-LE,
  §6.1 초기화 시퀀스, 부속서 A 디폴트 맵 모두 준수. 회사코드 = 0.
- **3-tier fallback** (슬레이브 부팅 시): (1) NVS 저장 addr > (2) Kconfig `KSNODE_SLAVE_ADDR ≠ 0` fixed > (3) unassigned + discovery 대기
- **Master toggle**: `CONFIG_SFC_DISCOVERY_ENABLED=n` 으로 discovery 완전 우회 → 하드코딩 registry 만 사용 (외부 slave 조합·마스터 표준시험 시)
- **교차 호환**: slave fixed addr flash → 외부 표준 마스터가 부속서 A 로 통신 가능

상세: [`../docs/discovery.md`](../docs/discovery.md)

## KS X 3286 vendor spec 통합

회사코드 non-zero 노드 감지 시 embedded spec table 조회 → register map 계산 → `custom_map` 경로 poll.

- `ks_x_3286.[ch]` — item keyword size table + spec 배열 (`k_specs[]` 시연용)
- `node_registry` 상 `ks3286_find_node_spec(company, product)` → `ks3286_calc_map()` → `custom_map` 저장
- `poller` 상 `custom_map.valid` 이면 `poll_custom_map_node` 로 분기

상세: [`../docs/vendor-spec.md`](../docs/vendor-spec.md)

## Actuator downstream — 외부 NPN modbus relay module (R421 계열)

Actuator slave (`slave-c3-act/`) 는 KS X 3267 채널 상태를 실 relay 로 out 하기 위해 **downstream Modbus master** 로 외부 NPN relay module 을 write. 두 축 완전 분리:

| 축 | 프로토콜 | 성격 |
|---|---|---|
| Master ↔ Slave (upstream) | **KS X 3267 표준 준수** — default map (cmd reg 503+i*4, status reg 203+i*4), 회사코드=0 | 표준 그대로. 외부 표준 마스터/시험소도 호환. |
| Slave → NPN relay module (downstream) | **R421 계열 관례 하드코딩** — FC=06, reg=ch(1-based), value 0x0100 ON / 0x0200 OFF | 완제품 내부 취급. 다른 브랜드 NPN 이면 재작성. |

**활성 조건**: `CONFIG_KSNODE_TYPE_ACTUATOR=y` + `CONFIG_KSNODE_EXT_MASTER_ENABLE=y` + `CONFIG_KSNODE_NPN_RELAY_ADDR=<addr>`. RS485-2 (g4/5/6) 로 NPN A/B 결선.

**Hook**: `ks_actuator_node.c` 상 채널 state 변경 2곳 (새 cmd + TIMED 종료) 에서 `ext_master_write_single(addr, ch, value)` 호출. NPN 응답 로그: `→ NPN addr=1 reg=0x0004 val=0x0100 (ON): ESP_OK`.

**다른 모델 대응**: FC/register/value 매핑 모델별로 다름 (일부는 FC=05 coil, ON=0xFF00). hook 부분만 재작성. Kconfig `KSNODE_NPN_MODEL` 로 select 옵션 확장 여지.

## slave-c3-sensor 의 sensor kind 분리

한 firmware 소스로 여러 종의 센서 slave (CO2 / EC / soil / dummy / vendor) 를 다루기 위해 **kind 별 `sdkconfig.defaults.<kind>` 파일 + Kconfig choice group 강제** 방식 사용.

```
slave-c3-sensor/
├─ sdkconfig.defaults           # kind-neutral 공통 base
├─ sdkconfig.defaults.dummy     # NONE — 19채널 dummy (KS X 검증용)
├─ sdkconfig.defaults.co2       # SCD41 I2C
├─ sdkconfig.defaults.ec        # SoilA RS485
├─ sdkconfig.defaults.soil      # 토양함수율 (RS485, 4800 baud)
├─ sdkconfig.defaults.vendor    # KS X 3286 vendor 시연 (회사=0xABCD/제품=0x0001)
└─ scripts/flash-slave.sh       # kind + port + addr wrapper
```

- **Kconfig choice `KSNODE_SENSOR_KIND`** 로 한 slave 는 한 kind 만 (여러 driver 동시 활성 금지).
  각 driver enable 옵션은 `def_bool y if KSNODE_SENSOR_KIND_XX` 로 자동 파생.
- **`SLAVE_ADDR` 은 script command-line arg 로 override** — 같은 kind 여러 slave 를 다른 addr 로
  flash 시 sdkconfig 파일 수정 불필요.

## 빌드 / 플래시 (참고)

### ESP-IDF 환경
```bash
source ~/esp/esp-idf/export.sh
```

### 각 컴포넌트

| 컴포넌트 | target | 명령 |
|---|---|---|
| `p4/` | esp32p4 | `cd p4 && idf.py -p <PORT> flash` |
| `master/` | esp32s3 | `cd master && idf.py -p <PORT> flash` (BOOT 홀드 필요) |
| `slave-c3-sensor/` | esp32c3 | `cd slave-c3-sensor && scripts/flash-slave.sh <kind> <PORT> [addr]` |
| `slave-c3-act/` | esp32c3 | `cd slave-c3-act && idf.py -p <PORT> flash` |

### 슬레이브 핀맵

| 변형 | 외부 모듈 핀 | controller comm 핀 |
|---|---|---|
| `slave-c3-sensor/` | IO4/5/6 | IO0/1/10 |
| `slave-c3-act/` | IO4/5/6 | IO0/1/10 |

### RS485 baud

**9600 bps** (KS X 3267 §4.1 표준 기본). master 와 모든 slave 가 일치.
초기 실험에서 38400 도 동작했으나 마진 위해 9600 채택 — 이 결정 이력은
[블로그 §12.5](../blog-post.md) 참조.
