# KS X 3267 표준 적합성 검증

> 자체 개발한 마스터(온실 통합 제어기)와 슬레이브(센서/구동기) 펌웨어가 KS X 3267:2022의 **디폴트 레지스터 맵 노드** 규정을 실제로 지키는지, 실 하드웨어를 대상으로 명령 단위로 왕복 검증한 기록.

관련 문서
- `architecture.md` — 시스템 구성(마스터/슬레이브, MQTT, 프론트엔드)
- `discovery.md` — 표준 §5.1이 남긴 자유 영역에서 채택한 사설 discovery 규약
- `vendor-spec.md` — KS X 3286 자동등록 노드(비-디폴트 맵)와의 상호운용

---

## 1. 검증 개요

| 항목 | 내용 |
|---|---|
| 표준 | **KS X 3267:2022** — 스마트 온실 센서/구동기 노드와 온실 통합 제어기 간 RS485 기반 모드버스 인터페이스 |
| 검증 대상 | 자체 개발 펌웨어 — 온실 통합 제어기(master) 1대, 센서 노드(slave) 1대, 구동기 노드(slave) 1대 |
| 검증 범위 | KS X 3267 **디폴트 레지스터 맵**(부속서 A) 기반 노드의 전(全) 표준 명령 |
| 결과 요약 | 디폴트 맵 범위 표준 명령 **전수 PASS**, 검증 중 발견한 표준 비적합 **4건 전부 수정·재검증 완료** |

### 1.1 왜 "디폴트 레지스터 맵" 범위인가

KS X 3267은 §5.1에서 노드 유형을 두 갈래로 나눈다.

- **§5.1.1 자동등록 기능을 지원하는 노드** — 회사·제품 코드가 non-zero. 실제 레지스터 배치는 별도 표준 KS X 3286이 정의하는 노드 스펙 JSON에서 계산한다.
- **§5.1.2 디폴트 레지스터 맵을 사용하는 노드** — 기관코드·회사코드 모두 0. 표준 부속서 A가 정의한 고정 맵을 그대로 사용한다.

우리 슬레이브는 **후자(디폴트 맵)** 로 자기선언한다. 마스터도 부속서 A 맵으로 통신하는 범위까지 우선 완성했고, 이 문서의 검증은 그 범위 안에서 이뤄진다. KS X 3286 노드 통합은 별개 잔여로 관리한다(§7 참조).

### 1.2 검증 환경 — 최소 구성 3대

| 역할 | 보드 | 펌웨어 | 표준 ProductType (§6.1.1 표9) |
|---|---|---|---|
| master (온실 통합 제어기) | ESP32-S3 | controller | — |
| 센서 노드 (slave) | ESP32-C3 | sensor slave | 1 (센서 노드) |
| 구동기 노드 (slave) | ESP32-C3 | actuator slave | 2 (구동기 노드) |

- 물리 계층: RS485, 38400 bps, 8N1, modbus RTU, slave 주소 1
- 센서 노드 = 제어기 BUS#1, 구동기 노드 = 제어기 BUS#2 (타입별 버스 분리)
- 두 슬레이브 모두 KS X 3267 §5.1.2 **디폴트 레지스터 맵 사용 노드** — 기관코드·회사코드 = 0

---

## 2. 검증 방법론

### 2.1 대전제 — 마스터가 묻고 노드가 답한다

표준 §6의 모든 명령은 **마스터(제어기)가 요청, 슬레이브(노드)가 응답** 구조다. 슬레이브가 자발적으로 뭘 내보내는 경로는 표준에 없다.

초기에는 P4 HMI 버튼을 눌러 마스터 태스크를 유발하고 시리얼 로그를 캡처했지만, 곧 세 가지 한계에 부딪혔다.

- 임의 파라미터(예: `op=305, position=37`)를 만들 버튼이 없다
- 재현이 안 된다. 같은 순서로 다시 누를 수가 없다
- USB-Serial-JTAG가 write burst 로그를 드롭한다

### 2.2 해결 — 서버가 마스터에 명령을 주입한다

제어기는 이미 MQTT로 상위 서버와 통신한다. 여기에 **검증자가 직접 붙어서 임의의 모드버스 요청을 주입**하는 방식으로 바꿨다.

```
검증자 ──MQTT publish──▶ 온실 통합 제어기 ──RS485 modbus 요청──▶ 노드(slave)
       ◀──MQTT ack────                    ◀──RS485 modbus 응답──
```

- 명령 토픽: `smartfarm/controller/{MAC}/cmd/#`
- 응답 토픽: `smartfarm/controller/{MAC}/ack`
- `mb_read` → 제어기가 **FC 0x03 (Read Holding Registers)** 로 노드 조회
- `mb_write` → 제어기가 **FC 0x10 (Write Multiple Registers)** 로 노드에 제어 명령
- 예외 응답 검증(범위 밖 주소, RO 영역 write)도 같은 경로로 유발 가능

이 방식으로 검증이 **결정적이고 반복 가능**해졌다.

### 2.3 검증 파이프라인 — 프론트엔드에서 한 번에

SvelteKit 라우트 `/ksx-verify` 와 디바이스 상세 화면의 `KsxVerifyWidget` 이 검증 진입점이다.

```
프론트엔드 v* 함수 (v_n1 / v_s1 / v_sw4 …)
  └─ mbCall({ cmd, bus, addr, reg, count | values })
       └─ MQTT publish  smartfarm/controller/{MAC}/cmd
            └─ controller 펌웨어가 RS485 로 slave 에 전달
       └─ MQTT subscribe smartfarm/controller/+/ack  (5000ms 타임아웃)
            └─ id correlation → Promise resolve
```

- 성공 read: `{ id, ok:true, regs:[...] }`
- 성공 write: `{ id, ok:true }`
- 실패: `{ id, ok:false, err:"..." }` 또는 `timeout 5000ms` 예외

### 2.4 주소·워드 인코딩 — 검증에 앞서 못 박아야 하는 두 규약

**주소 변환 규칙 (§5, p.13)**
- KS X 3267 레지스터는 **1-based**, modbus 와이어 프로토콜은 **0-based**
- 와이어 시작 주소 = (KS 레지스터 번호 − 1)
- 모든 명령에서 이 변환을 로그로 확인했다

**워드 인코딩 규칙 (§4.3.3, p.~11)**
- uint32·float 다중 워드 값: **워드 내부는 big-endian, 워드 간은 little-endian**
- 예: float 28.8 = IEEE-754 `0x41E66666`
  - `reg[n]   = 0x6666`  (하위 워드)
  - `reg[n+1] = 0x41E6`  (상위 워드)

이 두 규약이 어긋나면 그 위의 모든 명령이 조용히 틀린다. 그래서 프로토콜 계열(P-*) 을 가장 먼저 검증했다.

### 2.5 바이트 레벨 로그 — 대조를 위한 최소 요건

양쪽에서 같은 트랜잭션을 서로 다른 시점으로 찍어 대조했다.

```
[master 측]
D MB_MASTER: TX  bus2→slv1  FC=10 WR_MULTI  ks_reg=503..506  wire_start=0x01F6 cnt=0x0004
                payload= c9 00 2f f4 00 00 00 00
D MB_MASTER: RX  bus2←slv1  FC=10 ok (echo start=0x01F6 cnt=0x0004)

[slave 측]
I MBS_REQ: WR  FC=0x10  ks_reg=503..506 (4W)  wire_off=0x01F6  area=Act cmd(503~506): ch1
I ACTUATOR: ch1 opid=12276 op=201 → status=201 remain=0s
```

핵심은 `ks_reg` 와 `wire_off` 를 같이 찍고, **영역 이름(`area=`)** 을 로그에 넣은 것이었다. 주소만 봐서는 그게 어느 블록의 몇 번 채널인지 매번 계산해야 한다.

---

## 3. 판정 기준 — verdict 4단계와 실패 라인 포맷

### 3.1 verdict 정의

| verdict | 의미 | UI 표시 |
|---|---|---|
| **PASS** | 요청 성공 + 응답이 기대치와 완전 일치 | 초록 |
| **WARN** | 통신 성공했으나 값이 기대와 불일치 (표준 위반이지만 응답은 있음) | 노랑 |
| **FAIL** | 통신 자체 실패 (timeout / ack err / 예외) | 빨강 + fail lines |
| **N/A** | 노드 타입 부적용 (예: 센서 노드에 구동기 명령) | 회색 |

### 3.2 판정 룰 (모든 항목 공통)

| 카테고리 | 판정 함수 | PASS 조건 | WARN 조건 | FAIL 조건 |
|---|---|---|---|---|
| **read only** (N-1, N-2, N-3, S-1) | 각 v* | read 성공 + 값이 표준과 일치 | read 성공했으나 값 불일치 | read 실패 |
| **write+echo** (N-5, SW-*, OC-*) | `actVerdict` | write ok + status read ok + opid echo 일치 + status ∈ expectStatus | opid 또는 status 불일치 | write 실패 또는 status read 실패 |
| **예외 write** (P-2.RO) | `vRoProtect` | RO 영역 write **거부됨** (mbCall 실패) | — | RO 영역 write **수용됨** (표준 비적합) |
| **예외 read** (P-2.ADDR) | `vIllegalAddrRead` | 범위 밖 reg read 에 예외 응답 | — | 정상 응답 반환 (표준 비적합) |

`write+echo` 계열은 write 성공 후 **300ms 대기 → status block read** 순으로 진행한다. 이 대기는 slave 가 명령을 처리하고 status 를 채울 시간이다. 짧으면 opid echo 불일치로 WARN 이 뜬다.

### 3.3 FAIL 시 노출되는 실패 라인 포맷

```
{항목} 실패: {reason}                # timeout 5000ms / ack err / 예외 메시지
요청 bus=2 addr=1 reg=101 count=1     # 어느 PDU 가 실패했는지
ack ok=false err="crc" errno=3 ...    # ack payload 있을 때만
```

`reason` 우선순위: `mbCall` 에서 throw 된 `e.message` > ack 의 `err` > `"ack ok=false (err 미상)"`.

---

## 4. 검증 항목 — 명령별 판정

각 항목: **표준 위치(문서·페이지·레지스터) → 요청/응답 절차 → 결과**.

### 4.1 노드 정보 계열 (§6.1)

#### N-1 · 노드 정보 조회 — §6.1.1 (p.16~17, 표9)

| 항목 | 값 |
|---|---|
| 레지스터 | 1=기관코드, 2=회사코드, 3=제품타입, 4=제품코드, 5=프로토콜버전, 6=채널수, 7~8=시리얼번호(uint32) |
| 요청 | `mb_read bus addr reg=1 count=8` |
| 응답 | 8 워드 `[CertAuthority, CompanyCode, ProductType, ProductCode, ProtoVer, ChannelNumber, SerialLo, SerialHi]` |
| PASS | `ProtocolVersion == 10` AND `ChannelNumber == 노드 실제 채널수` |
| 결과 | **PASS** — 센서 노드 `ProductType=1`, 구동기 노드 `ProductType=2` 로 응답. 채널수·시리얼번호 정상. |

- `ProductType`: 1=SENSOR, 2=ACTUATOR, 3=INTEGRATED
- SerialNumber 는 §4.3.3 워드 인코딩(lo/hi)으로 32bit 재조합해야 사람이 읽는 값이 나온다

#### N-2 · 디바이스 코드 조회 — §6.1.2 (p.17~18, 표10)

| 항목 | 값 |
|---|---|
| 레지스터 | 101~(100+채널수) — 디바이스 코드 목록 (0x00=미부착) |
| 요청 | `mb_read bus addr reg=101 count=채널수` |
| PASS | read 성공 (모든 채널 코드 디코드 가능) |
| 결과 | **PASS** — 센서 노드 `[온도1, 습도2, CO2 11, 토양함수율 14]`, 구동기 노드 스위치 코드 `102` 응답. 부속서 A 디폴트 맵과 일치. |

#### N-3 · 노드 상태 조회 (레벨0) — §6.1.3.1 (p.19~20, 표11)

| 항목 | 값 |
|---|---|
| 레지스터 | 202 = 노드 상태 (uint16, 상태코드는 부속서 B.2) |
| 요청 | `mb_read bus addr reg=202 count=1` |
| 결과 | **PASS** — `reg202 = 0 (READY)` 응답 |

`actStatusLabel(status)` 매핑: 0=READY, 201=ON, 202=BUSY, 299=USER, 301=OPENING, 302=CLOSING, 399=MANUAL, 900~999=VENDOR.

#### N-5 · 노드 제어 / 제어권 변경 CONTROL — §6.1.4.1 (p.21, 표13) · 구동기/통합 노드 한정

| 항목 | 값 |
|---|---|
| 레지스터 | 501=노드 명령 (값 = KS_CTRL_LOCAL(1) / REMOTE(2) / MANUAL(3)), 502=OPID#0 (부속서 A.2.4, p.37) |
| 요청 (모드 전환) | `mb_write reg=501 values=[<code>, opid]` |
| 대기 | 200ms |
| status | `mb_read reg=201 count=1` (OPID#0 echo) + 이후 채널 status 관찰 |
| PASS 조건 | (a) write ok + OPID echo, (b) LOCAL/MANUAL 전환 후 SW cmd → 채널 status = `SW_USER_CONTROL(299)`, OC cmd → `MANUAL_CONTROL(399)`, (c) REMOTE 복귀 후 SW/OC cmd 정상 처리 |
| 결과 | **PASS** — opid echo + 채널 status override 전수 대조 완료 |

**구현 노트 (2026-07)**: 초기엔 opid echo 만 지원 (CONTROL code 값 무시) → LOCAL/MANUAL 모드에서도 원격 SW/OC cmd 정상 처리 = 표준 §6.1.4.1 위반이었으나 발견되지 않았음 (자체 verify widget 도 opid echo 만 검사).  slave 상 `s_control_code` 상태 저장 + 채널 처리 루프 상 mode 반영 (REMOTE 아니면 status override + NPN write skip) 로 수정.

### 4.2 센서 계열 (§6.2)

#### S-1 · 센서 상태 정보 조회 — §6.2 (p.22, 표14)

| 항목 | 값 |
|---|---|
| 레지스터 | 채널당 3워드 = `[값 float(2W) + 상태 uint16(1W)]` (부속서 A.1.4) |
| 요청 | `mb_read reg=203 count=채널수×3` |
| PASS | read ok + 모든 채널 `status == 0 (READY)` |
| WARN | read ok, 1개 이상 채널이 READY 아님 |
| 결과 | **PASS** |

**정밀 검증 (바이트 대조):**

노드를 **고정값 모드**로 재플래시해 전 채널을 28.8로 강제한 뒤, 응답 레지스터를 IEEE-754 기대값과 byte 단위로 대조했다.

```
기대: 28.8 = IEEE-754 0x41E66666
   → §4.3.3 워드 인코딩 적용 시
     reg[n]   = 0x6666  (하위)
     reg[n+1] = 0x41E6  (상위)

실측: 전 채널 [0x6666, 0x41E6]  →  일치
```

**결과: PASS** — 워드 내부 big-endian + 워드 간 little-endian 적합.

### 4.3 구동기 계열 (§6.3)

구동기 제어 명령은 모두 아래 공통 파이프라인을 따른다.

```
1. opid = 16bit 난수 생성
2. mb_write reg=cmd_base+(ch-1)*4 values=[op, opid, argLo, argHi]   ← cmd block, 4 words
3. 300ms 대기
4. mb_read  reg=status_base+(ch-1)*4 count=4                         ← status block, 4 words
5. actVerdict(opid, statusRes, expectStatus)
```

- cmd 블록 순서 `[op, opid, argLo, argHi]` — 부속서 A.2.6 (p.39~41)
- status 블록 순서 `[stOpid, stStatus, remainLo, remainHi]` — 부속서 A.2.5 (p.37)
- operation 코드: 부속서 B.3 (p.43), status 코드: 부속서 B.2 (p.42)

> **주의: cmd 와 status 는 필드 순서가 다르다.** cmd 는 op 먼저, status 는 opid 먼저. 이 비대칭을 "일관성 있게 통일"하려 하면 곧바로 표준 비적합이 된다(§6 비적합 #1 참조).

#### 스위치형 구동기 SW — §6.3.4

cmd 베이스 = 503, status 베이스 = 203 (채널1 기준).

| ID | 이름 | op | arg | expectStatus | 표준 | 결과 |
|---|---|---|---|---|---|---|
| **SW-4** | ON | 201 | 0 | `[201]` | §6.3.4.1 | **PASS** |
| **SW-5** | OFF | 0 | 0 | `[0]` (READY 복귀) | §6.3.4.2 | **PASS** |
| **SW-6** | TIMED_ON `Ns` | 202 | 초 | `[201]` + `remain` 카운트다운, 0초에서 자동 OFF | §6.3.4.3 | **PASS** |
| **SW-7** | DIRECTIONAL_ON | 203 | 파라미터 | opid echo 만 (ratio status echo 자리 없음 — 디폴트맵 한계) | §6.3.4.4 | **PASS (명령 전송)** |

#### 개폐형 구동기 OC — §6.3.3

cmd 베이스 = 567, status 베이스 = 267 (개폐기1 기준). 구동기 노드 채널수를 24로 설정(스위치16 + 개폐기8)해 개폐기 채널을 활성화한 뒤 검증했다.

| ID | 이름 | op | arg | expectStatus | 표준 | 결과 |
|---|---|---|---|---|---|---|
| **OC-4** | OPEN | 301 | 0 | `[301]` | §6.3.3.1 | **PASS** |
| **OC-5** | CLOSE | 302 | 0 | `[302]` | §6.3.3.2 | **PASS** |
| **OC-6** | STOP | 0 | 0 | `[0]` | §6.3.3.3 | **PASS** |
| **OC-7** | TIMED_OPEN `Ns` | 303 | 초 | `[301]` + `remain` | §6.3.3.4 | **PASS** |
| **OC-8** | TIMED_CLOSE `Ns` | 304 | 초 | `[302]` + `remain` | §6.3.3.5 | **PASS** |
| **OC-9** | SET_POSITION `p%` | 305 | 위치 | opid echo 만 | §6.3.3.6 | **PASS (명령 전송)** |
| **OC-10** | SET_CONFIG | 306 | 설정값 | opid echo 만 | §6.3.3.7 | **PASS (명령 전송)** |

### 4.4 프로토콜/예외 계열 (§4)

#### P-1 정상 응답 흐름 (§4.2)
요청 → 응답 정상 왕복. 전 명령 공통 확인. **PASS.**

#### P-2.ADDR · 범위 밖 reg read 예외 (§4.3.1.2.2)

| 항목 | 값 |
|---|---|
| 요청 | `mb_read reg=9000 count=1` (표준 명세 밖 주소) |
| PASS | ack `ok:false` (`err="…"` 로 예외 사유 반환) |
| FAIL | 정상 응답 반환 (0 채워 리턴 등) — 표준 비적합 |
| 결과 | **PASS** — 노드가 modbus 예외 PDU 회신, 제어기가 예외로 인식 |

#### P-2.RO · 응답영역 RO write 거부 (§4.3.1.2.2)

| 항목 | 값 |
|---|---|
| 요청 | `mb_write reg=1 values=[0]` (NodeInfo 영역, RO) |
| PASS | mbCall 실패 (slave 가 write 거부) — reason 을 PASS 라인에 노출 |
| FAIL | write 성공 (표준 비적합) |
| 결과 | **PASS** |

**"성공하면 실패" 논리 반전** — RO 영역이 write 거부돼야 표준 준수다.

#### P-4 워드 인코딩 (§4.3.3)
uint32(남은동작시간)·float(센서값) 워드 인코딩 byte 대조. **PASS.**

#### P-5 1-based ↔ 0-based (§5)
모든 요청에서 와이어 시작주소 = (KS 레지스터 − 1) 확인. **PASS.**

#### P-6 opid 규칙 (§6.1.4)
제어 명령마다 opid 변경, opid 변경 시점 = 명령 활성화. **PASS.**

### 4.5 카탈로그 매트릭스 — 노드 타입별 적용

| ID | 그룹 | SENSOR (1) | ACTUATOR (2) | INTEGRATED (3) |
|---|---|:---:|:---:|:---:|
| N-1 노드 정보 | 공통 | O | O | O |
| N-2 디바이스 코드 | 공통 | O | O | O |
| N-3 노드 상태 | 공통 | O | O | O |
| N-5 CONTROL | 공통 | N/A | O | O |
| S-1 (종합) | 센서 | O | N/A | O |
| S-1.ch (채널별) | 센서 | O | N/A | O |
| SW-4~7 | 구동기 | N/A | O | O |
| OC-4~10 | 구동기 | N/A | O | O |
| P-2.RO / P-2.ADDR | 예외 | O | O | O |

---

## 5. 최종 결과

디폴트 레지스터 맵 범위 표준 명령을 **전수 PASS.**

- 노드 정보 계열 N-1 ~ N-5 → 전부 PASS
- 센서 계열 S-1 (바이트 레벨 정밀 대조 포함) → PASS
- 스위치형 구동기 SW-4 ~ SW-7 → 전부 PASS (SW-7 은 표준상 opid-echo-only 한계 명시)
- 개폐형 구동기 OC-4 ~ OC-10 → 전부 PASS (OC-9/10 동일 한계)
- 프로토콜/예외 P-1 · P-2.ADDR · P-2.RO · P-4 · P-5 · P-6 → 전부 PASS

한 노드가 표준에 완전히 적합하려면 아래를 모두 PASS 해야 한다.

**SENSOR 노드:** N-1 (Ver=10, ch 일치) + N-2 + N-3 + S-1 종합 + 모든 채널 S-1.ch + P-2.RO + P-2.ADDR

**ACTUATOR 노드:** N-1 + N-2 + N-3 + N-5 + 각 채널 SW-4/5/6 + SW-7(opid-only) + 각 채널 OC-4~8 + OC-9/10(opid-only) + P-2.RO + P-2.ADDR

**INTEGRATED 노드:** 위 둘의 합집합.

---

## 6. 검증에서 깨진 것들 — 표준 비적합 4건

네 건 모두 "우리끼리는 완벽하게 동작하던" 코드였다. 표준 준수 검증은 **자기끼리의 왕복 성공이 아니라, 남의 마스터를 붙였을 때에도 성립하는가**를 묻는 도구다.

### 비적합 #1 — cmd 블록 필드 순서 역전

**증상:** 없었다. HMI 버튼을 누르면 팬이 돌았다.

**발견:** 바이트 레벨 payload 를 표준 부속서와 한 워드씩 대조하다가 나왔다.

```
캡처된 payload:  f4 2f c9 00 00 00 00 00

  reg[0] = 0x2FF4 = 12276  ← 이게 opid
  reg[1] = 0x00C9 = 201    ← 이게 operation

표준(부속서 A.2.6):
  cmd 블록 = [ 제어명령(op) 1W, OPID 1W, 동작시간(uint32) 2W ]
  → reg[0]이 명령, reg[1]이 opid 여야 한다.
```

**원인:** 표준의 비대칭 때문이다.

```
status 블록 (A.2.5) = [ OPID,     상태,  남은시간 ]   ← OPID 먼저
cmd    블록 (A.2.6) = [ 제어명령,  OPID,  동작시간 ]   ← 명령 먼저
```

구현할 때 "일관성 있게 opid 를 앞에 두자"고 양쪽을 통일해버렸다. status 는 우연히 표준과 맞고 cmd 만 틀렸다. 그리고 마스터와 슬레이브가 **똑같이 틀렸기 때문에** 자기들끼리는 완벽하게 동작했다. 표준 준수 타사 제어기가 이 노드에 명령을 보내면 `op=201` 을 쓰려 한 것이 `opid=201, op=<쓰레기>` 로 해석된다.

**수정:** 매직 넘버를 없애고 표준 조항 번호를 주석에 박은 오프셋 상수로 통일.

```c
/* 마스터·슬레이브가 같은 정의를 참조하도록 공통 헤더에 오프셋 상수를 둔다 */
#define KS_ACT_CMD_OFF_OP    0   /* A.2.6 */
#define KS_ACT_CMD_OFF_OPID  1
#define KS_ACT_CMD_OFF_ARG   2

#define KS_ACT_ST_OFF_OPID   0   /* A.2.5 — 순서가 다르다! */
#define KS_ACT_ST_OFF_STATUS 1
#define KS_ACT_ST_OFF_REMAIN 2
```

**재검증:**

```
송신 cmd    reg503 = [op=201, opid=7777, arg=0]
회신 status reg203 = [opid=7777, status=201, remain=0]     PASS

  ← 옛 펌웨어였다면 opid=201, status=READY 로 나왔을 것.
```

**교훈:** "일관성 있게 정리"하고 싶은 충동이 표준 위반의 주범이다. 표준이 비대칭이면 비대칭이 정답이다.

### 비적합 #2 — 지정기간 명령 중 상태 코드 미반영

**증상:** TIMED_ON 을 걸면 팬은 도는데 상태 조회 결과가 `0(READY)` 로 나온다.

**표준:** 부속서 B.2 (p.42) — 작동 중에는 status 가 `201(ON)` / `301(OPENING)` / `302(CLOSING)` 이어야 한다.

**원인:** 상태 결정 로직이 `ON`, `OPEN`, `CLOSE` 만 분기하고 `TIMED_*`, `DIRECTIONAL_ON` 은 `default` 로 떨어져 READY 가 되고 있었다.

```c
static uint16_t status_for_op(uint16_t op) {
    switch (op) {
    case OP_SW_ON:  return KS_STS_SW_ON;
    case OP_OPEN:   return KS_STS_OPENING;
    case OP_CLOSE:  return KS_STS_CLOSING;
    /* + TIMED_ON, DIRECTIONAL_ON → SW_ON
       + TIMED_OPEN → OPENING, TIMED_CLOSE → CLOSING  케이스 추가 */
    default:        return KS_STS_READY;
    }
}
```

**교훈:** `default: return READY` 는 위험한 기본값이다. 새 명령을 추가할 때마다 조용히 "정상 대기 중" 이라고 거짓말을 한다. 열거형 스위치에서 default 를 두지 않고 컴파일러 경고에 맡기는 편이 나았다.

### 비적합 #3 — 응답 영역 write 미보호

**증상:** 마스터가 노드의 **노드 정보 영역(reg 1)에 write 를 하면 노드가 그대로 받아들였다.**

```
mb_write reg1 = 9999  →  ok 회신 + reg1 이 실제로 9999 로 변경됨
```

노드의 제품타입이나 시리얼 번호가 바깥에서 덮어써질 수 있다는 뜻이다. 버그 있는 마스터 하나가 노드의 정체를 영구히 망가뜨릴 수 있다.

**표준:** §5.2 그림14 (p.14) — 노드 정보·디바이스 코드·상태 정보는 **응답(읽기) 영역**.

**수정:** 모드버스 영역 등록 시 접근 권한을 명시하도록 바꿨다.

| 영역 | 시작 레지스터 | 권한 |
|---|---|---|
| 노드 정보 | 1 | **RO** |
| 디바이스 코드 | 101 | **RO** |
| 구동기 status | 201 | **RO** |
| 센서 블록 | 202 | **RO** |
| 구동기 cmd | 501 | RW |

이제 RO 영역에 write 요청이 오면 모드버스 스택이 **예외 응답(0x02, illegal data address)** 으로 자동 거부한다. 노드 자신의 태스크가 상태를 갱신하는 건 이 플래그와 무관하므로 정상 동작한다.

**재검증:**

```
[구동기]
  노드정보 reg1 write        → 거부   PASS
  디바이스코드 reg101 write   → 거부   PASS
  status reg203 write        → 거부   PASS
  cmd reg503 write           → 수용   PASS   ← 명령 영역은 RW 가 맞다
  노드정보 / status read     → 정상   PASS

[센서]
  노드정보 reg1 write        → 거부   PASS
  센서값 reg203 write        → 거부   PASS
```

**교훈:** 모드버스 홀딩 레지스터는 이름부터가 read/write 겸용이라 **"읽기 전용" 을 명시적으로 선언하지 않으면 기본값이 쓰기 가능**이다. 표준 문서의 그림에서 "응답 영역" 이라고 화살표가 그려진 부분은 전부 RO여야 한다.

### 비적합 #4 — 회사코드 0xFEED 자기선언 (default map 오인식 유발)

**증상:** 없었다. 우리 마스터는 잘 붙었다. 자체 검증도 프레임·맵 전부 표준.

**발견:** 위 3건의 검증이 완료된 뒤 별도 세션에서 노드 응답을 다시 훑어보다 나왔다. NodeInfo 응답의 **회사코드 자리에 0xFEED** 가 실려 있었다.

```
reg1 (CertAuthority)  = 0x0000    OK
reg2 (CompanyCode)    = 0xFEED    ← 문제
reg3 (ProductType)    = 1         OK
...
```

**표준:** 부속서 A.1.1 각주 — **디폴트 레지스터 맵 노드는 기관코드·회사코드가 둘 다 0** 이어야 한다. 하나라도 non-zero 면 마스터는 이 노드를 **"KS X 3286 자동등록 대상"** 으로 판정하고 노드 스펙 파일을 요청한다. 3286 미구현 슬레이브라면 그 요청에 응답할 방법이 없다.

**원인:** 개발 초기에 회사코드 자리에 "우리 회사" 표식으로 재미 삼아 넣어둔 상수가 정리되지 않고 남아있었다. 우리 마스터는 이 값을 무시하고 부속서 A 맵으로 처리해왔기 때문에 **자체 통신에서는 아무 증상이 없었다.**

**수정:**

```diff
- CONFIG_KSNODE_COMPANY_CODE=0xFEED
+ CONFIG_KSNODE_COMPANY_CODE=0x0000
```

Kconfig 도움말에 표준 근거를 못 박아 다음 사람이 다시 돌리지 못하게 했다.

```
KS X 3267 §5.1 + 부속서 A.1.1 각주:
  기관코드=0 AND 회사코드=0 → 디폴트 맵 노드로 판정
  하나라도 non-zero → KS X 3286 자동등록 노드로 오인식
본 프로젝트는 §5.1.2 디폴트 맵 경로 → 회사코드는 0 유지
```

**교훈:** 검증 도구는 **프레임 규격만 심판한다.** 응답 필드가 표준이 정의한 **의미** 에서 어긋나 있어도 프레임은 통과한다. 표준 준수는 "프레임 통과 + 필드 의미의 정확한 자기선언" 두 축이 다 맞아야 한다. 그리고 "우리끼리는 잘 도는 코드" 함정이 여기서 다시 재현됐다.

### 공통 흐름 — 비적합 발견에서 재검증까지

네 건 모두 아래 흐름을 밟았다.

```
1. 검증 스크립트가 WARN/FAIL 을 낸다
     (혹은 #4처럼 별도 세션에서 응답 field 를 다시 훑다 발견)
2. 표준 원문 확인 — 조항·페이지·표 번호를 찍는다
3. 펌웨어 수정 — 매직 넘버 대신 표준 조항 주석을 박은 상수로
4. 재플래시 후 검증 재실행 — PASS 라인 확인
5. 회귀 방지 — 검증 위젯의 정기 실행 대상에 등록
```

---

## 7. 검증 한계와 잔여

### 7.1 잔여 #1 — 센서 타입 전수 스윕 (S-1 확장)

본 문서의 S-1 검증은 디폴트 맵의 **4개 채널(온도·습도·CO2·토양함수율)** 로 수행했다. 표준 부속서 A.1.2 가 정의하는 **19개 센서 타입 전체**를 한 노드가 채널별로 '제시' 하며 페이로드를 전수 대조하는 작업이 남아 있다.

**준비 완료:**
- 센서 노드를 **19채널 구성으로 플래시** — ch1~19 가 디바이스 코드 1~19(온도~누적유량)와 1:1 대응
- 프론트엔드 슬레이브 상세 페이지에 **KsxVerifyWidget "종합" 버튼** 추가 — ch1~19 의 `디바이스 코드 → 센서명 + 값 + 상태` 를 표 하나로 일괄 조회
- 남은 것은 위젯에서 일괄 조회를 실행하고 19종 페이로드를 표준과 대조·기록하는 작업뿐

**범위 주의:** KS X 3267 범위에서 모든 센서 타입은 **동일한 페이로드 구조**(채널당 3워드: float 값 + uint16 상태)를 사용한다. 타입별 차이는 ① reg101+ 의 디바이스 코드 ② 값의 범위·단위인데, **값 범위·단위는 KS X 3269 별도 표준** 소관이다. 따라서 3267 기준 전수 스윕은 "N-2 디바이스 코드 정확 보고 + S-1 값/상태 인코딩 적합" 을 센서 타입마다 반복 검증하는 것을 의미한다.

### 7.2 잔여 #2 — 레벨1/2 노드 상태 계열

디폴트 맵은 레벨0~1 기준이다. 레벨1 전용 항목(§6.1.3.2)은 sdkconfig 레벨 변경 후 추가 검증 가능하다. 레벨2 전용 항목(SW-7 강도, OC-9 SET_POSITION, OC-10 SET_CONFIG 의 status echo)은 **디폴트 맵 구조에 자리가 없다.** 표준 원문상 이들은 "자동등록기능 지원 노드"(KS X 3286 소관)에만 해당하므로, 현재 디폴트 맵 노드의 4워드 고정 구조가 KS X 3267 의 정답이고 늘리면 오히려 표준 위반이다. 완전 검증은 KS X 3286 노드 별도 구현 시 가능하다.

### 7.3 잔여 #3 — KS X 3288 양액기 표준

KS X 3288 은 양액기 노드용 별개 표준이다. 프로토콜 골격(RS485/modbus RTU, 워드 인코딩, cmd/status 블록)은 3267 을 계승하지만 **부속서 레지스터 맵과 명령 코드가 다르다.** 현 시점에서 3288 슬레이브는 구현 범위 밖이며, 3267 검증 프레임을 그대로 활용해 3288 매트릭스로 확장하는 것이 다음 단계다.

---

## 8. 결론

- KS X 3267:2022 **디폴트 레지스터 맵(부속서 A) 기반 노드의 표준 명령** (노드정보 N-1~5, 센서 S-1, 스위치형 구동기 SW-4~7, 개폐형 구동기 OC-4~10, 프로토콜 P-1·P-2·P-4·P-5·P-6) 을 전수 검증, **전 항목 PASS.**
- 검증 중 발견한 표준 비적합 **4건을 모두 수정하고 재검증으로 적합 확인.**
- 디폴트 맵 구조상 한계 항목(레벨2 position/ratio) 은 표준 자체가 별도 표준(KS X 3286)에 위임한 범위로, 현재 구현이 KS X 3267 디폴트 맵 기준으로는 적합하다.
- **현 시점에서 자체 개발 펌웨어는 KS X 3267:2022 디폴트 레지스터 맵 범위의 RS485 모드버스 인터페이스 표준에 적합하다.**

검증 프로세스 자체를 프론트엔드 위젯에 심어두었기 때문에, 이후 펌웨어를 수정할 때마다 회귀 검증이 클릭 한 번으로 가능하다. 잔여 항목(센서 19종 전수 스윕, 레벨1 노드 상태, KS X 3288 확장) 도 같은 프레임 위에서 진행할 계획이다.
