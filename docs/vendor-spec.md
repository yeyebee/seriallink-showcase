# KS X 3286 자동등록 노드 통합

우리 마스터에 **타사(vendor) 표준 준수 노드**가 붙어도 정상 작동하도록, KS X 3286 이 정의한
"NodeInfo → 회사/제품코드 → 노드 규격(CommSpec) 조회 → register map 계산 → poll" 5-단계 flow 를
펌웨어에 얹은 기록.

---

## 목차

- [1. 배경 — 왜 3286 이 필요한가](#1-배경--왜-3286-이-필요한가)
- [2. 표준이 요구하는 5단계 flow](#2-표준이-요구하는-5단계-flow)
- [3. 우리 구현 — spec table + map 계산](#3-우리-구현--spec-table--map-계산)
- [4. Registry 통합 — 회사코드 non-zero 감지](#4-registry-통합--회사코드-non-zero-감지)
- [5. Poller 통합 — custom_map 경로 분기](#5-poller-통합--custom_map-경로-분기)
- [6. Slave 쪽 vendor 시연 프로필](#6-slave-쪽-vendor-시연-프로필)
- [7. 시연 시나리오 (end-to-end)](#7-시연-시나리오-end-to-end)
- [8. 미구현 & 향후 설계 — 원격 spec fetch](#8-미구현--향후-설계--원격-spec-fetch)
- [9. 왜 얹었나 — 호환성 검증의 완결편](#9-왜-얹었나--호환성-검증의-완결편)
- [10. 관련 문서](#10-관련-문서)

---

## 1. 배경 — 왜 3286 이 필요한가

KS X 3267 스마트팜 시리즈는 두 가지 노드 프로필을 상정한다:

| 구분 | 회사/기관 코드 | Register 배치 | 마스터가 알 수 있는 방법 |
|---|---|---|---|
| **default map 노드** | `0` / `0` | 부속서 A 고정 배치 (센서 202~, 구동기 201~) | 표준에 박혀 있으므로 즉시 알고 있음 |
| **자동등록(vendor) 노드** | non-zero | 벤더가 임의로 배치 (예: reg 401, 501, …) | 매번 다르므로 **규격 문서를 조회해서 알아내야 함** |

우리 slave 는 default map 만 만들어 왔다 — 회사코드 `0`, 부속서 A 배치. 시연·검증에는 충분하다.
하지만 실 온실에는 다른 회사가 만든 non-default 노드가 함께 붙는다. 이걸 **컴파일 시점에 미리
알지 못한 채로도 부팅 후 동작하게** 만드는 것이 KS X 3286 §7~§8 의 목적이다.

> KS X 3286 은 **규격 저장 방법을 규정하지 않는다** — "원격 저장소 또는 파일 복사" 라고만 한다.
> 즉 spec 배포 채널(HTTP, SPIFFS, 컴파일 embed …)은 구현 자유.

우리는 **컴파일 타임 embedded spec table** 로 골격을 세우고, 실 vendor 노드 도입 시점에 원격
fetch 로 확장한다 (§8 설계).

---

## 2. 표준이 요구하는 5단계 flow

제어기(마스터) 관점:

```
① NodeInfo 조회 (reg 1~8)
     → cert_authority, company_code, product_type, product_code, …
     → 회사/기관 코드 non-zero 면 이 노드는 3286 대상

② 회사/제품 코드로 노드 규격(CommSpec) 조회
     로컬 embedded table (본 구현) 또는 원격 저장소/파일 (향후 §8)
     CommSpec 예:
     { read:  {starting-register: 401, items: [status, opid]},
       write: {starting-register: 501, items: [operation, opid, time]} }

③ 디바이스 코드 조회 (reg 101 ~ 100+channel_number)  ← default map path 와 공용

④ (선택) 회사/디바이스 코드로 디바이스 규격 조회  ← 노드 규격만 구현, 디바이스 규격은 후속

⑤ CommSpec.starting-register + items → 실 register 주소 계산
     starting=401, items=[status(1W), opid(1W)]
       → read map: {401:status, 402:opid}
     starting=501, items=[operation(1W), opid(1W), time(2W)]
       → write map: {501:operation, 502:opid, 503-504:time}
```

핵심은 **③ 이후의 register 주소를 코드에 하드코딩하지 않고 spec 으로부터 산출**한다는 점.
같은 마스터 펌웨어가 회사/제품이 다른 vendor 노드들 각자의 배치를 그대로 수용한다.

---

## 3. 우리 구현 — spec table + map 계산

### 3.1 파일 구성

```
firmware/master/main/
  ks_x_3286.h    ← item keyword enum + spec struct + API
  ks_x_3286.c    ← item size 매핑 + embedded spec table + calc_map()
```

### 3.2 Item keyword (§9.2 표 13~19)

KS X 3286 은 각 register 슬롯의 **의미(status, opid, operation, time, value …)** 를 keyword 로
정의하고, keyword 마다 word size 를 규정한다. 우리는 시연 범위(노드 상태 + 센서/구동기 상태 +
제어)만 우선 정의:

```c
typedef enum {
    KS3286_ITEM_NONE = 0,
    /* 노드 상태 (표 13) */
    KS3286_ITEM_STATUS, KS3286_ITEM_OPID, KS3286_ITEM_CONTROL,        /* 각 1W */
    /* 센서 상태 (표 14) */
    KS3286_ITEM_VALUE,                                                 /* float, 2W */
    /* 구동기 상태 (표 15, 16) */
    KS3286_ITEM_STATE_HOLD_TIME, KS3286_ITEM_REMAIN_TIME,              /* 2W */
    KS3286_ITEM_OPENTIME, KS3286_ITEM_CLOSETIME,                       /* 2W */
    KS3286_ITEM_POSITION, KS3286_ITEM_RATIO,                           /* 1W */
    /* 제어 명령 (표 17, 18) */
    KS3286_ITEM_OPERATION,                                             /* 1W */
    KS3286_ITEM_TIME, KS3286_ITEM_HOLD_TIME,                           /* 2W */
    KS3286_ITEM_MAX,
} ks3286_item_t;
```

Item → word size 는 단일 `k_item_meta[]` 테이블로 관리한다 (13개 keyword × `{item, words, name}`
triple). `ks3286_item_words()` / `ks3286_item_name()` 이 이 테이블을 lookup.

### 3.3 노드 규격 (CommSpec) 스펙 struct

```c
typedef struct {
    uint16_t         starting_register;   /* 1-based (KS X 3267 규약) */
    uint8_t          n_items;
    ks3286_item_t    items[KS3286_MAX_ITEMS];
} ks3286_block_t;

typedef struct {
    uint16_t         company_code;
    uint16_t         product_code;
    const char      *model_name;          /* 로그·UI 표시용 */
    ks3286_block_t   read;                /* 상태 조회 블록 */
    ks3286_block_t   write;               /* 제어 블록 (없으면 n_items=0) */
} ks3286_node_spec_t;
```

### 3.4 Embedded spec table (시연용)

```c
static const ks3286_node_spec_t k_specs[] = {
    {
        .company_code = 0xABCD,
        .product_code = 0x0001,
        .model_name   = "VendorX Actuator Node",
        .read  = {
            .starting_register = 401,
            .n_items = 2,
            .items = { KS3286_ITEM_STATUS, KS3286_ITEM_OPID },
        },
        .write = {
            .starting_register = 501,
            .n_items = 3,
            .items = { KS3286_ITEM_OPERATION, KS3286_ITEM_OPID, KS3286_ITEM_TIME },
        },
    },
    /* 실 vendor 추가는 여기 append. 회사코드는 인증기관 발급 후 사용. */
};
```

### 3.5 Spec → register map 계산

`ks3286_calc_map()` 이 §8 을 그대로 반영: **starting_register 부터 items 순차, 각 item 의 word
size 만큼 주소 누적**.

```c
static void calc_block(const ks3286_block_t *b, ks3286_field_t *out, uint8_t *n_out)
{
    uint16_t addr = b->starting_register;
    uint8_t  n = 0;
    for (uint8_t i = 0; i < b->n_items && n < KS3286_MAX_FIELDS_PER_BLOCK; i++) {
        uint8_t w = ks3286_item_words(b->items[i]);
        if (w == 0) continue;
        out[n].item     = b->items[i];
        out[n].reg_addr = addr;
        out[n].words    = w;
        addr += w;
        n++;
    }
    *n_out = n;
}

esp_err_t ks3286_calc_map(const ks3286_node_spec_t *spec, ks3286_map_t *out)
{
    if (!spec || !out) return ESP_ERR_INVALID_ARG;
    memset(out, 0, sizeof(*out));
    calc_block(&spec->read,  out->read,  &out->n_read);
    calc_block(&spec->write, out->write, &out->n_write);
    out->valid = true;
    return ESP_OK;
}
```

계산 결과 예 — `starting=501, items=[operation(1W), opid(1W), time(2W)]`:

```
addr=501 → field{reg=501, item=operation, words=1}
addr=502 → field{reg=502, item=opid,      words=1}
addr=503 → field{reg=503, item=time,      words=2}   (time 2W → 다음 addr=505)
```

---

## 4. Registry 통합 — 회사코드 non-zero 감지

`node_registry` 는 각 (bus, addr) 노드 엔트리를 캐시한다. 3286 통합의 진입점은 **NodeInfo 상
회사코드 non-zero 감지 → embedded spec table 조회 → map 계산 → entry 에 저장** 이다.

`ks_node_t` 에 map 필드 하나 추가 (기존 필드는 …):

```c
typedef struct {
    /* … bus_id, slave_addr, reachable, info, device_codes[], last_ok_tick, … */

    /* KS X 3286 자동등록 노드 map — 회사코드 non-zero 이고 spec 등록된 경우만 valid.
     * default-map 노드는 map.valid=false 로 두고 poller 가 부속서 A 로 처리. */
    ks3286_map_t custom_map;
} ks_node_t;
```

`ks_registry_refresh()` 는 NodeInfo 를 read 한 뒤 아래 조건 분기를 추가한다:

```c
node->custom_map.valid = false;
if (node->info.company_code != 0) {
    const ks3286_node_spec_t *spec =
        ks3286_find_node_spec(node->info.company_code, node->info.product_code);
    if (spec) {
        ks3286_calc_map(spec, &node->custom_map);
        ESP_LOGI(TAG, "bus%u slave %u: KS X 3286 spec loaded (%s)",
                 node->bus_id, node->slave_addr, spec->model_name);
    } else {
        ESP_LOGW(TAG, "bus%u slave %u: company=0x%04X product=0x%04X — spec 미등록, poll skip",
                 node->bus_id, node->slave_addr,
                 node->info.company_code, node->info.product_code);
    }
}
```

세 갈래로 나뉜다:

| 조건 | 결과 |
|---|---|
| `company_code == 0` | default map 노드 → `custom_map.valid=false` 유지, poller 가 §A.1/A.2 로 처리 |
| `company_code != 0` + spec **hit** | `custom_map` 채움, poller 가 §5 경로로 분기 |
| `company_code != 0` + spec **miss** | 경고 로그 + `custom_map.valid=false` — poll 은 skip (미매핑) |

세 번째 갈래(회사코드는 알겠지만 spec 이 로컬에 없는 경우)가 §8 원격 fetch 를 넣어야 하는 지점.

---

## 5. Poller 통합 — custom_map 경로 분기

Poller 는 매 사이클 registry 를 순회하며 각 노드를 read 한다. 3286 통합 전에는 product_type
switch 하나 뿐이었다:

```c
switch (node->info.product_type) {
    case KS_PRODTYPE_SENSOR_NODE:     err = poll_sensor_node(node, &scratch);   break;
    case KS_PRODTYPE_ACTUATOR_NODE:   err = poll_actuator_node(node, &scratch); break;
    case KS_PRODTYPE_INTEGRATED_NODE: err = poll_nutrient_node(node, &scratch); break;
    default:                          /* skip */                                break;
}
```

3286 통합 후: `custom_map.valid` 를 **먼저** 검사해 그것이 참이면 그 map 으로 read.

```c
if (node->custom_map.valid) {
    err = poll_custom_map_node(node, &scratch);
} else switch (node->info.product_type) {
    case KS_PRODTYPE_SENSOR_NODE:     err = poll_sensor_node(node, &scratch);   break;
    case KS_PRODTYPE_ACTUATOR_NODE:   err = poll_actuator_node(node, &scratch); break;
    case KS_PRODTYPE_INTEGRATED_NODE: err = poll_nutrient_node(node, &scratch); break;
    default:                          /* skip */                                break;
}
```

`poll_custom_map_node` 는 spec 의 read block 을 **연속된 한 트랜잭션**으로 read 한 뒤 item 별
offset 으로 dispatch. 핵심 부분만:

```c
static esp_err_t poll_custom_map_node(ks_node_t *node, ks_node_snapshot_t *out)
{
    const ks3286_map_t *m = &node->custom_map;
    if (m->n_read == 0) return ESP_ERR_INVALID_STATE;

    /* Read block 은 continuous — 첫 field addr 부터 마지막 (addr+words) 까지 한 번에 read */
    uint16_t start = m->read[0].reg_addr;
    uint16_t last  = m->read[m->n_read-1].reg_addr + m->read[m->n_read-1].words;
    uint16_t cnt   = last - start;

    uint16_t buf[32];
    esp_err_t err = mb_master_read_holding(node->bus_id, node->slave_addr,
                                           start, cnt, buf);
    if (err != ESP_OK) return err;

    for (uint8_t i = 0; i < m->n_read; i++) {
        uint16_t off = m->read[i].reg_addr - start;
        switch (m->read[i].item) {
            case KS3286_ITEM_STATUS:  out->node.status = buf[off]; break;
            case KS3286_ITEM_OPID:    out->node.opid = buf[off];
                                      out->node.has_opid = true; break;
            case KS3286_ITEM_CONTROL: out->node.control = buf[off];
                                      out->node.has_control = true; break;
            default: break;
        }
    }
    return ESP_OK;
}
```

전체 dispatch 흐름:

```
Poller cycle
  └─ for each node in registry:
       ├─ node.custom_map.valid ?
       │    ├─ true  → poll_custom_map_node  (KS X 3286 경로)
       │    │          ├─ spec.read block 을 한 번에 read
       │    │          └─ item 별로 status/opid/control 매핑
       │    │
       │    └─ false → product_type switch  (KS X 3267 부속서 A 경로)
       │               ├─ SENSOR    → poll_sensor_node   (reg 202~)
       │               ├─ ACTUATOR  → poll_actuator_node (reg 201~)
       │               └─ INTEGRATED→ poll_nutrient_node (reg 201~ + reg 301~)
       │
       └─ 결과를 cache slot 에 저장 → uplink 로 전송
```

---

## 6. Slave 쪽 vendor 시연 프로필

실 vendor 노드가 없는 상태에서 end-to-end flow 를 검증하려면 우리 slave 하나를 "vendor 인 척"
하도록 flash 해야 한다. `sdkconfig.defaults.vendor` 프로필이 이 역할이다:

```
# ── Sensor kind: VENDOR (KS X 3286 자동등록 노드 시연) ──
CONFIG_KSNODE_SENSOR_KIND_NONE=y
CONFIG_KSNODE_CHANNEL_NUMBER=1
CONFIG_KSNODE_TYPE_ACTUATOR=y

# 회사·제품 코드 (마스터 spec table 매칭)
CONFIG_KSNODE_COMPANY_CODE=0xABCD
CONFIG_KSNODE_PRODUCT_CODE=0x0001

# Vendor 시연 활성
CONFIG_KSNODE_VENDOR_DEMO=y
CONFIG_KSNODE_VENDOR_READ_REG=0x0191    # = 401 decimal
```

`CONFIG_KSNODE_VENDOR_DEMO` 가 켜지면 slave-base 의 `mb_slave.c` 가 지정된 register 에 dummy
read area 하나를 추가 등록한다:

```c
#if CONFIG_KSNODE_VENDOR_DEMO
    static uint16_t s_vendor_read[2] = { 0x0000, 0x0001 };  /* status=READY, opid=1 */
    ESP_ERROR_CHECK(add_area((uint16_t)CONFIG_KSNODE_VENDOR_READ_REG,
                             s_vendor_read, sizeof(s_vendor_read), MB_ACCESS_RO));
    ESP_LOGI(TAG, "KS X 3286 vendor demo: reg 0x%04X (%u) 등록 [status, opid]",
             CONFIG_KSNODE_VENDOR_READ_REG, CONFIG_KSNODE_VENDOR_READ_REG);
#endif
```

이 slave 를 마스터가 read 하면:
- NodeInfo: `company=0xABCD, product=0x0001` 응답 → 마스터가 3286 경로로 진입
- reg 401 read: `[0x0000, 0x0001]` = `status=READY, opid=1` 로 해석

시연 후엔 다른 kind (co2/ec/soil/dummy) 로 flash-slave 스크립트를 다시 돌리면 default map 으로
원복된다.

---

## 7. 시연 시나리오 (end-to-end)

```
[준비]
  1. slave-c3-sensor 를 vendor 프로필로 flash:
     $ ./flash-slave.sh vendor <port>
     → CONFIG_KSNODE_COMPANY_CODE=0xABCD
       CONFIG_KSNODE_PRODUCT_CODE=0x0001
       CONFIG_KSNODE_VENDOR_DEMO=y
     → reg 401 (status, opid) 를 dummy area 로 등록

  2. master 리셋 (RST 버튼 or `esptool.py --port … run`)

[관찰 — 마스터 로그]
  I REGISTRY: bus1 slave 5: type=2 code=1 ver=1 ch=1 serial=… (VendorX Actuator Node)
  I KS3286  : map for VendorX Actuator Node (0xABCD/0x0001):
  I KS3286  :   read  reg 401 = status  (1W)   read  reg 402 = opid (1W)
  I KS3286  :   write reg 501 = operation(1W)  write reg 502 = opid (1W)
  I KS3286  :   write reg 503 = time    (2W)
  I REGISTRY: bus1 slave 5: KS X 3286 spec loaded (VendorX Actuator Node)
  I POLLER  : bus1 slave 5 custom-map poll: reg 401..402 (2W)
  I POLLER  : bus1 slave 5 status=0x0000 opid=1

[검증 포인트]
  - default map (reg 202) 이 아니라 spec 이 지정한 reg 401 을 read 하는가?  ✓
  - status/opid 가 slave 가 등록한 dummy 값과 일치하는가?  ✓
  - 같은 sensor kind 로 재flash 시 custom_map.valid=false 로 원복되는가?  ✓
```

이 flow 가 통과하면 **회사코드만 알고 있으면 (그리고 spec 이 등록되어 있으면) 마스터 코드
변경 없이 새 vendor 노드가 붙는다** 는 것이 검증된 셈이다.

---

## 8. 미구현 & 향후 설계 — 원격 spec fetch

현 구현의 embedded spec table 은 시연 규모에서만 유효하다. 실 온실은 vendor 가 계속 늘어나므로
매번 firmware rebuild + flash 는 불가능. **원격 spec 배포 채널**이 필요하다.

### 8.1 세 가지 접근안 비교

| 안 | 흐름 | 장점 | 단점 |
|---|---|---|---|
| **A. HTTP + NVS 캐시** | 부팅 시 embedded fallback → NVS 캐시 로드 → 네트워크 준비 후 backend fetch → 병합/저장 | 파티션 변경 불필요, 백엔드 중앙 관리, 오프라인 대응 | JSON 파서 + backend endpoint 필요, race 처리 |
| **B. SPIFFS 파일** | 파티션 테이블에 spiffs 추가 → `/spiffs/ks3286_specs.json` 파싱 | 대용량 spec 가능 | 파티션 변경 = 전체 flash erase (NVS 손실), 업데이트 채널이 결국 별도 필요 |
| **C. NVS blob only** | embedded 유지 + NVS 에 추가 blob | 최소 변경 | NVS 24KB 제약, 확장성 낮음 |

**A 안 채택.** 이유:
- 파티션 테이블 그대로 → 마스터 재flash 시 NVS erase 리스크 없음
- backend 가 이미 있는 인프라(스마트팜 API 서버) 옆에 endpoint 하나 추가하는 정도의 확장
- NVS 캐시가 부족해지면 partition subtype `nvs` 를 16KB→64KB 로 늘려 대응 (전체 파티션 재배치
  없이 nvs region 만 재구성)

### 8.2 A 안 상세 흐름

```
Master boot
  ├─[1] embedded k_specs[] 로 즉시 동작 (fallback, 항상 존재)
  ├─[2] NVS "ks3286_cache" 로드 → dynamic table 병합 (이전 fetch 성공분)
  ├─[3] Wi-Fi got_ip 이벤트 → ks3286_fetch_task spawn
  ├─[4] GET /api/ks3286/specs → cJSON 파싱 → add → NVS save
  └─[5] 미등록 회사코드 발견 시 on-demand:
         GET /api/ks3286/specs?company=0xABCD&product=0x0001
```

### 8.3 Backend 스키마 초안

```sql
CREATE TABLE ks3286_specs (
  id            SERIAL PRIMARY KEY,
  company_code  INTEGER NOT NULL,
  product_code  INTEGER NOT NULL,
  model_name    TEXT    NOT NULL,
  spec_json     JSONB   NOT NULL,       -- {read: {...}, write: {...}}
  version       INTEGER NOT NULL DEFAULT 1,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (company_code, product_code)
);
```

Endpoints: `GET /specs` (전체 boot fetch), `GET /specs?company=&product=` (on-demand),
`POST /specs` / `DELETE /specs/:id` (admin).

JSON payload:

```json
{
  "company_code": 43981, "product_code": 1,
  "model_name": "VendorX Actuator Node",
  "read":  { "starting_register": 401, "items": ["status", "opid"] },
  "write": { "starting_register": 501, "items": ["operation", "opid", "time"] }
}
```

### 8.4 Firmware 구조 변경 (예정)

- `ks_x_3286.h/.c` — `ks3286_add_spec()` + `k_dynamic_specs[16]`, NVS load/save, string→item 역매핑
- `ks_x_3286.c` — `find_node_spec` 을 embedded → dynamic 순서로 검색
- `ks3286_fetch.c` (신규) — 네트워크 준비 후 HTTP GET → 파싱 → add → NVS save

### 8.5 다른 미구현 항목

- **디바이스 규격 (§7.4)** — 노드 규격만 지원. 디바이스별 sensor value/unit 매핑이 필요한 vendor
  노드가 등장하면 추가.
- **Sensor `value` 매핑** — `poll_custom_map_node` 는 status/opid/control 만 dispatch.
  `KS3286_ITEM_VALUE` 는 enum 에 있으나 poll path 미연결. vendor 센서 실물 도입 시 확장.
- **Spec 무결성 검증** — 해시/서명. Backend 신뢰 전제로 우선 skip.

### 8.6 오늘 스코프에서 제외한 이유

실 vendor 노드가 없어 end-to-end 검증 불가 (시연 spec 은 embedded 로 이미 동작). Backend 쪽
CRUD UI + admin 워크플로우도 함께 필요 — 별개 작업 사이클. 실 vendor 도입 결정 시 §8 대로 진행.

---

## 9. 왜 얹었나 — 호환성 검증의 완결편

우리 세트 (default map slave + 자체 마스터) 만 검증했다면 그건 "우리 것끼리는 잘 붙는다" 밖에
증명 못한다. 표준을 채택했다는 얘기는 결국 **표준 준수 타사 노드가 붙어도 정상 작동** 해야 성립.

컴파일 타임 embedded table 이 실무 배포엔 부족하다. 그러나 §7~§8 이 요구하는 **flow 자체**
(NodeInfo → spec 조회 → map 계산 → poll 분기) 를 갖췄고 시연 시나리오로 end-to-end 통과가
확인됐다. 원격 fetch 는 이 flow 위에 얹는 배포 채널일 뿐이라 골격을 바꾸지 않는다. 즉 현재
코드는 **"3286 §7~§8 골격 완료 + 배포 채널만 남긴 상태"** 이다.

---

## 10. 관련 문서

- [architecture.md](./architecture.md) — 마스터·slave·backend 3-tier 전체 구조
- [discovery.md](./discovery.md) — bus scan + registry lifecycle (3286 진입점 상위 컨텍스트)
- [verification.md](./verification.md) — KS X 표준 준수 검증 방법론 (vendor 시연은 검증의 일부)
