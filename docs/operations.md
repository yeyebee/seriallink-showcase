# 운영 편의성 — 검증 이후 UI 완성도

> 표준 준수 자체와는 무관하지만, 슬레이브 10~30 대가 붙기 시작한 시점부터 "UI/DB 운영이 부드럽게 굴러가는가" 를 결정한 네 가지 축의 기록.

관련 문서
- `architecture.md` — 전(全) 스택 구성
- `verification.md` — KS X 3267 디폴트 맵 노드 왕복 검증
- `discovery.md` — controller-slave heartbeat / snapshot / audit 파이프라인
- `vendor-spec.md` — KS X 3286 자동등록 노드 통합

| 축 | 문제 | 해결 |
|---|---|---|
| **§1 Async job device purge** | 21 만 row device 완전 삭제 시 178 ~ 306s → nginx timeout | POST 202 + tokio task + polling |
| **§2 Sensor prune (3-mode)** | dummy → 실 sensor 교체 시 stale metric row 가 UI 카드에 잔존 | stale / older_than / all 정리 API |
| **§3 `devices.addr` 컬럼** | 같은 버스 안 5~10 슬레이브 구분 불가 (bus_id 만 표시) | schema migration + snapshot upsert |
| **§4 Alias inline 편집** | Device 목록에서 alias 편집 위해 별도 화면 왕복 | 셀 더블클릭 → input → Enter/Esc, PATCH COALESCE |
| **§5 Alarm/Auto rule metric dropdown** | 노드가 EC 만 장착돼도 metric 이 free-form text → 오타 rule 은 fire 안 함 (silent failure) | `sensorState` 실시간 캐시 + `devCodeMetric` 매핑 → 실 sensor 만 옵션 노출 |

---

## §1. Async job device purge — nginx timeout 회피

### 1.1 문제 — 대량 DELETE 는 sync 로 감당 불가

Device 완전 삭제는 세 테이블을 한 트랜잭션으로 지운다.

- `migration_dev.sensor_data_v2` — 센서 이력 (TimescaleDB compressed hypertable)
- `migration_dev.devices` — 사용자 메타 (alias, location, group_id)
- `migration_dev.devices_dim` — ingest 가 자동 등록한 차원 테이블 (UI 목록 source)

이 중 압도적 다수는 첫 번째. 21 만 row 규모 device sync 삭제 실측:

| 시도 | 소요 | 결과 |
|---|---|---|
| nginx 기본 timeout (60s) | 60s abort | backend 트랜잭션 rollback |
| nginx timeout 300s 확장 | 178 ~ 306s | 대부분 성공, 일부 여전히 504 |

TimescaleDB compressed hypertable 상 partial DELETE 는 **chunk 별 decompress + scan** 이 필요하다. Compression 을 걷어내는 비용은 device 수/chunk 수에 비례하며, timeout 을 늘려서 근본 해결되는 성격이 아니다.

### 1.2 해결 패턴 — POST 202 + tokio task + polling

Timeout 확장이 아니라 **응답 모델 자체를 바꾼다**. 즉시 202 Accepted 반환, 실제 DELETE 는 tokio task 로 백그라운드 실행.

```
Client                                    Backend
  │                                          │
  │  ① DELETE /devices/by-device-id/S-...   │
  ├─────────────────────────────────────────►│
  │                                          ├─ tokio::spawn { … DELETE … }
  │◄─── 202 { job_id, poll_url } ────────────┤
  │                                          │
  │  ② GET /purge-status  (3s 간격)         │
  ├─────────────────────────────────────────►│
  │◄─── { state:"running", elapsed_ms:42130} │
  │           ...                            │
  │◄─── { state:"done",                      │
  │       sensor_rows:214883,                │
  │       elapsed_ms:186220 }                │
```

### 1.3 In-memory job registry + 핸들러

Key 를 `device_id` 로 잡은 이유 — **같은 device 에 대한 중복 purge 방지**. 사용자가 실수로 삭제 버튼을 두 번 눌러도 두 번째 요청은 409 로 튕겨서 pool slot 잠식을 막는다.

```rust
struct AppState {
    // ...
    purge_jobs: Arc<RwLock<HashMap<String, PurgeJob>>>,   // Key = device_id
}

#[derive(Clone, serde::Serialize)]
struct PurgeJob {
    job_id:      String,
    device_id:   String,
    state:       String,           // "running" | "done" | "error"
    started_at:  DateTime<Utc>,
    finished_at: Option<DateTime<Utc>>,
    elapsed_ms:  Option<u64>,
    sensor_rows: Option<u64>,
    devices:     Option<u64>,
    devices_dim: Option<u64>,
    error:       Option<String>,
}
```

핸들러는 admin 권한 체크 → 중복 running 체크 (409) → dim 존재 사전 확인 (spawn 낭비 방지) → job insert → 즉시 202 반환. 실제 DELETE 는 `tokio::spawn` 안에서.

```rust
tokio::spawn(async move {
    let t0 = Instant::now();
    let mut tx = pool_bg.begin().await?;
    sqlx::query("SET LOCAL timescaledb.max_tuples_decompressed_per_dml_transaction TO 0")
        .execute(&mut *tx).await?;
    let a = sqlx::query(
        "DELETE FROM migration_dev.sensor_data_v2
         WHERE device_id = (SELECT id FROM migration_dev.devices_dim WHERE device_id = $1)")
        .bind(&did_bg).execute(&mut *tx).await?.rows_affected();
    let b = sqlx::query("DELETE FROM migration_dev.devices WHERE device_id = $1")
        .bind(&did_bg).execute(&mut *tx).await?.rows_affected();
    let c = sqlx::query("DELETE FROM migration_dev.devices_dim WHERE device_id = $1")
        .bind(&did_bg).execute(&mut *tx).await?.rows_affected();
    tx.commit().await?;
    // ... jobs.write() 로 state=done, elapsed_ms, (a,b,c) 반영 ...
});
```

핵심 두 가지.

- **`SET LOCAL timescaledb.max_tuples_decompressed_per_dml_transaction TO 0`** — Timescale 기본값은 decompression 상한이 있어 대량 DELETE 가 중간에 튕긴다. `LOCAL` 로 해당 트랜잭션에서만 unlimited.
- 세 DELETE 는 **한 트랜잭션**. sensor 만 지우고 devices_dim 이 남으면 다음 snapshot 이 다시 자동 등록해서 유령 device 가 살아난다.

### 1.4 Frontend — `deleteDevice(id, onProgress?)`

3 초 간격 polling, 30 분 max. `onProgress` 콜백으로 `"⏳ 진행 중... 12s 경과 (job purge-S-...)"` 를 매초 업데이트. 페이지를 이동해도 backend task 는 살아있어서 다음 조회 시 상태 확인 가능.

```typescript
export async function deleteDevice(
  deviceId: string,
  onProgress?: (s: PurgeStatus & { elapsed_s: number }) => void,
): Promise<PurgeStatus> {
  const r = await af(`${API_BASE}/devices/by-device-id/${encodeURIComponent(deviceId)}`,
                     { method: 'DELETE' });
  if (r.status !== 202 && !r.ok) throw new Error(`HTTP ${r.status}`);

  const t0 = Date.now();
  while (Date.now() - t0 < 30 * 60 * 1000) {
    await new Promise(res => setTimeout(res, 3000));
    const sr = await af(`${API_BASE}/devices/by-device-id/${encodeURIComponent(deviceId)}/purge-status`);
    if (!sr.ok) continue;                       // 일시적 실패는 다음 poll 로 재시도
    const st = await sr.json() as PurgeStatus;
    onProgress?.({ ...st, elapsed_s: Math.round((Date.now() - t0) / 1000) });
    if (st.state === 'done')  { await loadDevices(); return st; }
    if (st.state === 'error') return st;
  }
  throw new Error('30분 초과 — backend log 확인 필요');
}
```

---

## §2. Sensor prune — dummy → 실 센서 교체 후 stale metric 정리

### 2.1 문제 — dummy 가 남긴 유령 metric

슬레이브가 dummy sensor firmware 로 부팅하면 backend 는 KS X device code 1~19 를 값 0.0 으로 다 upsert 한다 (표준 §6.1.2). 이후 실 sensor (예: SOIL_WATER) 로 flash 하면 그 metric 만 update 되지만, 기존 dummy metric row 들은 `sensor_data_v2` 에 남아 UI 카드에 계속 보인다 — **"6.3 시간 전 CO2 = 0.00"** 이 며칠씩 잔존. 이력이 남는 것 자체는 원칙적으로 valuable 하지만 UX 관점에서는 clutter.

### 2.2 3-mode API

| Mode | 삭제 대상 | 기본 |
|---|---|---|
| `stale` | 최근 N 시간 무데이터인 metric 의 이 device row 전체 (활성 metric 유지) | `hours = 3` |
| `older_than` | N 일 이전 row 전부 (모든 metric) | `days = 7` |
| `all` | 이 device 의 모든 sensor 이력 wipe (dim/devices row 는 유지) | — |

```rust
#[derive(Deserialize)]
struct SensorPruneBody {
    mode:  String,        // "stale" | "older_than" | "all"
    hours: Option<i64>,   // stale 기준 (default 3)
    days:  Option<i64>,   // older_than 기준 (default 7)
}
```

### 2.3 dim.id lookup — 초기 500 원인

`sensor_data_v2.device_id` 는 **smallint FK** (devices_dim.id 참조). 문자열 `S-093307FC` 를 이 컬럼에 그대로 bind 하면 sqlx 타입 강제라 조용히 실패하지 않고 500 이 튄다. 초기 구현에서 이 lookup 을 빼먹었다가 즉시 걸림.

수정판은 **문자열 → dim.id 를 별도 SELECT 로 미리 뽑고**, 이어지는 DELETE 는 `i16` 로 bind. subquery 로 우회하는 것보다 실패시 404 명시 응답이 가능해서 이 방식이 낫다.

```rust
// sensor_data_v2.device_id 는 smallint (devices_dim.id FK).
// 문자열 device_id 를 dim.id 로 lookup — 명시적 fetch 후 bind.
let dim_id: (i16,) = sqlx::query_as(
    "SELECT id FROM migration_dev.devices_dim WHERE device_id = $1")
    .bind(&device_id).fetch_one(&pool).await?;

match body.mode.as_str() {
    "stale" => {
        let hours = body.hours.unwrap_or(3).clamp(1, 24*30);
        // 이 device 상 최근 hours 시간 안에 row 있는 metric_id 를 keep,
        // 그 외 metric_id 의 이 device row 모두 삭제.
        let sql = "\
            WITH keep AS ( \
                SELECT DISTINCT metric_id FROM migration_dev.sensor_data_v2 \
                 WHERE device_id = $1 AND ts >= NOW() - ($2::text || ' hours')::interval \
            ) \
            DELETE FROM migration_dev.sensor_data_v2 \
             WHERE device_id = $1 AND metric_id NOT IN (SELECT metric_id FROM keep) \
             RETURNING metric_id";
        sqlx::query(sql).bind(dim_id.0).bind(hours.to_string()).fetch_all(&pool).await
    }
    "older_than" => { /* days 기준 삭제, RETURNING metric_id */ }
    "all"        => { /* device 전체 wipe,   RETURNING metric_id */ }
    _ => return bad_request("mode must be one of: stale | older_than | all"),
}
```

`RETURNING metric_id` 로 어떤 metric 이 정리됐는지 응답에 실어서 프런트가 UI 카드 refresh 판단에 활용.

### 2.4 재사용 컴포넌트 — `SensorPrunePanel.svelte`

같은 UI 를 두 곳에서 쓴다 — 대시보드의 `LatestSensorCard` 와 개별 device focus 인 `DeviceFocusCard`. 그래서 컴포넌트를 하나 만들고 `<slot name="extra-action" />` 으로 확장 가능하게 뒀다. `runDevicePurge` 는 §1 의 `deleteDevice()` 를 그대로 재사용 — 3-모드 prune 과 device 완전 삭제가 같은 panel 안에서 자연스럽게 이어진다.

---

## §3. `devices.addr` 컬럼 — bus 내 slave 주소 UI 표시

### 3.1 문제 — 같은 버스 안 5~10 슬레이브 구분 불가

기존 UI 는 `bus_id` (BUS#1 / BUS#2) 만 보여줬다. 같은 버스 안 5~10 대 슬레이브가 붙는 순간부터 어느 게 어느 건지 구분이 안 된다. Modbus RTU 는 원래 slave address 로 구분하는 프로토콜인데 UI 에 그 필드가 없었다.

### 3.2 Schema migration

```sql
-- devices.addr 컬럼 추가.
--   slave 노드의 modbus 주소 (bus 내 slave address).
--   snapshot payload 의 "slave" 필드로부터 upsert.
--
-- Idempotent (IF NOT EXISTS).
ALTER TABLE migration_dev.devices
  ADD COLUMN IF NOT EXISTS addr SMALLINT;
```

### 3.3 `ingest_node_snapshot` — snapshot.slave → devices.addr upsert

MQTT snapshot payload 의 `slave` 필드를 그대로 upsert 한다. Controller / standalone device 는 slave 개념이 없으므로 null.

```rust
async fn ingest_node_snapshot(state: &AppState, topic: &str, payload: &[u8]) {
    // ... serial → device_id ("S-XXXXXXXX"), ctrl_map → parent_controller_id lookup ...

    let snap_bus = json.get("bus")  .and_then(|v| v.as_u64()).map(|v| v as i16);
    let snap_pt  = json.get("type") .and_then(|v| v.as_u64()).map(|v| v as i16);
    let snap_ch  = json.get("ch")   .and_then(|v| v.as_u64()).map(|v| v as i16);
    let snap_slv = json.get("slave").and_then(|v| v.as_u64()).map(|v| v as i16);

    // default alias: "센서 BUS#1 슬레이브1" / "구동기 BUS#2 슬레이브3" ...
    let default_alias = format!("{} BUS#{} 슬레이브{}",
        match snap_pt.unwrap_or(0) { 1 => "센서", 2 => "구동기", 3 => "복합", _ => "노드" },
        snap_bus.unwrap_or(0), snap_slv.unwrap_or(0));

    sqlx::query(r#"
        INSERT INTO migration_dev.devices
            (device_id, device_type, alias, bus_id, channel_count, ks_product_type,
             parent_controller_id, addr, last_seen_at, online)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW(), true)
        ON CONFLICT (device_id) DO UPDATE SET
          device_type          = EXCLUDED.device_type,
          alias                = COALESCE(migration_dev.devices.alias, EXCLUDED.alias),
          bus_id               = EXCLUDED.bus_id,
          channel_count        = EXCLUDED.channel_count,
          ks_product_type      = EXCLUDED.ks_product_type,
          parent_controller_id = EXCLUDED.parent_controller_id,
          addr                 = EXCLUDED.addr,
          last_seen_at         = NOW(),
          online               = true
    "#).bind(&device_id).bind(auto_type).bind(&default_alias)
       .bind(snap_bus).bind(snap_ch).bind(snap_pt).bind(&parent).bind(snap_slv)
       .execute(pool).await.ok();
}
```

두 가지 upsert 원칙.

- **Hardware fact 는 EXCLUDED 우선**: `device_type`, `bus_id`, `channel_count`, `ks_product_type`, `parent_controller_id`, `addr` — snapshot 이 최신 진실이므로 매번 덮어씀.
- **사용자 편집값은 보존**: `alias` 는 `COALESCE(migration_dev.devices.alias, EXCLUDED.alias)` — 이미 편집된 alias 는 유지, 안 채웠으면 default 로.

### 3.4 Frontend — `Device` 타입 + DevicesCard 컬럼

`dev_list` / `dev_get` 응답에 `addr` 필드가 추가되고 frontend 타입도 확장. DevicesCard 테이블은 `bus` 옆에 `addr` 컬럼을 추가.

```typescript
export interface Device {
  id: number; device_id: string; device_type: string;
  alias: string | null;
  bus_id: number | null;
  addr: number | null;              // slave modbus 주소 (1~247). controller/일반 device 는 null.
  channel_count: number | null;
  ks_product_type: number | null;   // 1=sensor, 2=actuator, 3=integrated
  parent_controller_id: string | null;
}
```

```svelte
<td class:auto={d.bus_id != null}>{d.bus_id ?? '—'}</td>
<td class:auto={d.addr   != null}>{d.addr   ?? '—'}</td>
<td class:auto={d.channel_count != null}>{d.channel_count ?? '—'}</td>
```

`.auto` 클래스는 "snapshot 이 자동으로 채운 값" 을 파란 톤으로 시각 구분 — 사용자가 손댄 값과 하드웨어에서 들어온 값이 눈으로 구분된다.

---

## §4. Alias inline 편집 — 셀 클릭 → input → Enter/Esc

### 4.1 Backend `dev_update` — COALESCE 로 부분 업데이트

Alias 는 자주 바꾸는 값이다 (default `"센서 BUS#1 슬레이브1"` → 실 위치 `"1번 하우스 A동"`). 매번 상세 화면 왕복 없이 목록에서 바로 편집할 수 있어야 한다.

PATCH 는 부분 업데이트라서 body 에 안 담긴 필드는 `null` 로 들어온다. COALESCE 로 **null → 기존값 유지** 를 보장. 이걸 안 하면 alias 만 편집한 요청이 location/group_id 를 지워버림.

```rust
async fn dev_update(/* State, jar, Path(id), Json(body): DeviceUpdate */) -> impl IntoResponse {
    // ownership 체크 (admin 아니면 owner 만 편집 가능) ...

    // COALESCE 패턴: NULL 인 필드는 기존값 유지
    sqlx::query(r#"
        UPDATE migration_dev.devices SET
          alias    = COALESCE($2, alias),
          location = COALESCE($3, location),
          group_id = COALESCE($4, group_id)
        WHERE id = $1
    "#).bind(id).bind(body.alias).bind(body.location).bind(body.group_id)
       .execute(&pool).await
}
```

### 4.2 Frontend `updateDevice(id, patch)` + editing 상태

DevicesCard 는 `editingId: number | null` 로 편집 셀 1 개만 활성, `editValue` 는 임시 buffer. `updateDevice` 는 PATCH → 성공 시 `loadDevices()` refresh.

```svelte
<script lang="ts">
  let editingId: number | null = null;
  let editValue = '';

  function startEdit(d: Device) { editingId = d.id; editValue = d.alias ?? ''; }
  function cancelEdit()         { editingId = null; editValue = ''; }

  async function saveEdit(d: Device) {
    const newAlias = editValue.trim();
    // 빈 값이면 저장 취소 (실수 방지). 명시적 null 원하면 별도 버튼 필요.
    if (!newAlias || newAlias === d.alias) { cancelEdit(); return; }
    try { await updateDevice(d.id, { alias: newAlias }); }
    finally { cancelEdit(); }
  }

  function onEditKey(e: KeyboardEvent, d: Device) {
    if      (e.key === 'Enter')  { e.preventDefault(); saveEdit(d); }
    else if (e.key === 'Escape') { e.preventDefault(); cancelEdit(); }
  }
</script>

<td class="alias-cell">
  {#if editingId === d.id}
    <input bind:value={editValue} on:keydown={(e) => onEditKey(e, d)}
           on:blur={() => saveEdit(d)} autofocus />
    <button on:click={() => saveEdit(d)}><Check size={12} /></button>
    <button on:click={cancelEdit}><X size={12} /></button>
  {:else}
    <span on:dblclick={() => startEdit(d)}>{d.alias ?? '—'}</span>
    <button class="alias-btn subtle" on:click={() => startEdit(d)}><Pencil size={11} /></button>
  {/if}
</td>
```

UX 세부.

- **더블클릭 → 편집 진입** — 단일 클릭은 device 상세 이동 등 다른 용도로 남김.
- **hover 시 연필 아이콘 fade-in** — `.alias-btn.subtle { opacity: 0 }` + `.alias-cell:hover .alias-btn.subtle { opacity: 0.6 }`.
- **Enter 저장 / Esc 취소 / blur 자동 저장** — 모달 없이 마우스만으로도, 키보드만으로도 완결.
- **빈 문자열은 저장 취소로 처리** — 실수로 지우고 Enter 눌러 default alias 로 되돌아가는 것 방지.

---

## §5. Alarm/Auto rule metric dropdown — 실 sensor devcode 자동 필터

### 5.1 문제 — free-form metric text 는 silent failure 유발

Alarm rule form 이 `<input type="text" placeholder="metric">` 이면 사용자가 아무 문자열이나 입력할 수 있다.  예: 노드가 EC 만 장착돼 있는데 `temperature` 로 rule 저장 → rule 은 만들어지고 UI 상 나열되나 실 ingest sample metric 은 `ec` → op 비교 miss → rule 이 **영원히 fire 안 함**.  사용자는 "왜 알람이 안 오지?" 로 시간 소모, 로그도 조용.

### 5.2 3-layer 해결

**Layer 1 — `sensorState` 실시간 캐시** (신규 store, `realtime.ts`).  `actuatorState` 와 대칭.  MQTT snapshot 상 `sensors[]` 배열이 있으면 `device_id → { bus, slave, sensors: [{slot, dev, v, st}], updated_at }` 로 세팅.  매 snap (~5 s) 마다 갱신.

**Layer 2 — `devCodeMetric` 매핑** (`modbus.ts`).  KS X 3267 §A.1.2 19종 sensor devcode → backend metric key (`ks_devcode_to_metric` 와 sync).  drift 방지 위해 두 파일 동시 수정 관행.

**Layer 3 — Form dropdown**.  `selDev` 변경 시:
```
$: metricOpts = $sensorState.get(selDev.device_id)?.sensors
    .map(s => devCodeMetric(s.dev))
    .filter(m => m != null)
    .map(m => ({ metric: m, label: `${devCodeLabel(...)} (${m})` }));
```
드롭다운 노출 (label 은 한글+key 예: "EC (ec)").  device 전환 시 `form.metric` 자동 재선택 (이전 device metric 이 새 device 옵션에 없으면 첫 항목).  snap 미도착 시 (device 등록 직후) 는 fallback text input.

### 5.3 왜 backend endpoint 로 안 하고 실시간 캐시로

`GET /api/devices/:id/metrics` (DB 상 `SELECT DISTINCT metric FROM sensor_data_v2 ...`) 도 옵션이나:
- 방금 등록한 device 는 sample 이 아직 없어 empty → form 이 비어있음
- Rule 작성 = device 처음 붙이는 순간에 자주 함 → snap 이 오히려 더 fresh
- 이미 `actuatorState` snap-driven pattern 이 성숙 — 대칭성 확보

## 마무리

이 다섯 축은 KS X 3267 표준 준수와 직접 관계가 없다. 그런데 검증을 마친 뒤 실 운영에 붙이는 순간부터 **없으면 UI 가 실사용 불가 상태** 가 된다 — 슬레이브 삭제 시도 → 504 → rollback → 유령 device 계속 잔존, dummy 이력이 UI 카드에 며칠씩 남아 실 데이터를 가림, 같은 버스 안 슬레이브들이 addr 없이 alias 만으로 구분 불가, alias 편집을 위해 매번 상세 화면으로 왕복, alarm rule 은 오타 metric 으로 조용히 실패.  표준 준수가 "부수지 않는 조건" 이라면, 이 다섯 축은 "실제로 굴러가는 조건" 이다.
