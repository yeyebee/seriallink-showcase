---
title: "스마트팜 KS X 3267 표준 통합 제어기를 직접 구현해 본 기록"
description: "RS485 Modbus RTU 기반 KS X 3267 표준 온실 통합 제어기와 센서/구동기 노드를 직접 구현하고 표준 적합성을 전수 검증하기까지."
pubDate: 2026-07-24
tags: ["smart-farm", "ks-x-3267", "modbus-rtu", "esp32", "rust", "sveltekit", "timescaledb"]
draft: false
---

# 스마트팜에서 사용하는 KS X 표준 프로토콜의 스마트 통합 제어기

> RS485 모드버스 기반 KS X 3267 표준 온실 통합 제어기와 센서/구동기 노드를 직접 구현하고,
> 표준 적합성을 전수 검증하기까지의 기록.

---

## 목차

1. [들어가며 — 왜 표준부터 봤나](#1-들어가며--왜-표준부터-봤나)
2. [표준 지형도 — 3267 / 3286 / 3288 / 3269](#2-표준-지형도--3267--3286--3288--3269)
3. [시스템 구성 — 무엇을 만들었나](#3-시스템-구성--무엇을-만들었나)
4. [프로토콜 기초 — PDU, 기능코드, 워드 인코딩](#4-프로토콜-기초--pdu-기능코드-워드-인코딩)
5. [레지스터 맵 — 디폴트 맵이라는 약속](#5-레지스터-맵--디폴트-맵이라는-약속)
6. [초기화 시퀀스 — 노드를 발견하고 정체를 파악하기](#6-초기화-시퀀스--노드를-발견하고-정체를-파악하기)
7. [센서 노드 구현](#7-센서-노드-구현)
8. [구동기 노드 구현 — cmd/status 블록과 OPID](#8-구동기-노드-구현--cmdstatus-블록과-opid)
9. [검증 방법론 — "눌러서 확인"에서 벗어나기](#9-검증-방법론--눌러서-확인에서-벗어나기)
10. [검증에서 깨진 것들 — 표준 비적합 4건](#10-검증에서-깨진-것들--표준-비적합-4건)
11. [표준이 그어놓은 선 — 디폴트 맵의 한계](#11-표준이-그어놓은-선--디폴트-맵의-한계)
12. [삽질 로그 — 표준과 상관없이 시간을 잡아먹은 것들](#12-삽질-로그--표준과-상관없이-시간을-잡아먹은-것들)
13. [정리와 남은 일](#13-정리와-남은-일)
14. [검증 이후 — 표준을 지키면서 편의를 얻는 법](#14-검증-이후--표준을-지키면서-편의를-얻는-법)

---

## 1. 들어가며 — 왜 표준부터 봤나

스마트팜 제어기를 만든다고 하면 보통 "센서 값 읽어서 화면에 띄우고, 릴레이 켜고 끄면 되는 거 아냐?"에서 출발한다.
그런데 온실 현장은 **센서 회사, 구동기 회사, 제어기 회사가 전부 다르다.** 각자 자기 프로토콜을 쓰면
제어기 회사는 납품 건마다 드라이버를 새로 짜야 하고, 농가는 한 번 산 제어기에 묶인다.

그래서 KS X 3267이 있다. **"온실 통합 제어기와 센서/구동기 노드는 RS485 모드버스로 이렇게 대화한다"**를
레지스터 번호 단위까지 못 박아 놓은 국가표준이다.

이 글은 그 표준을 읽고,
- 온실 통합 제어기(master) 1대
- 센서 노드(slave) 1대
- 구동기 노드(slave) 1대

를 직접 펌웨어로 구현한 뒤, **표준에 정의된 명령을 하나씩 전수 검증**한 기록이다.
"동작한다"와 "표준에 맞다"가 다른 말이라는 걸 꽤 아프게 배운 과정이기도 하다.

> 결론부터: 디폴트 레지스터 맵 범위의 표준 명령은 전수 PASS. 다만 그 과정과 직후 재검토에서 **자체 구현의 표준 비적합 4건**을
> 찾아내 고쳤다. 네 건 중 세 건은 우리끼리는 잘 동작하던 코드였다. (→ [10장](#10-검증에서-깨진-것들--표준-비적합-4건))

---

## 2. 표준 지형도 — 3267 / 3286 / 3288 / 3269

혼자 있는 표준이 아니다. 구현하다 보면 계속 옆 표준으로 넘겨진다.

| 표준 | 다루는 것 | 이 프로젝트에서의 위치 |
|---|---|---|
| **KS X 3267** | 센서/구동기 노드 ↔ 온실 통합 제어기 간 RS485 모드버스 인터페이스 | **주 검증 대상.** 레지스터 맵, 명령 체계, 인코딩 |
| **KS X 3286** | 노드/디바이스 **등록 절차 및 기술 규격** (자동등록) | 임의 레지스터 맵 노드. 레벨2 항목은 여기서만 완전 표현 가능 |
| **KS X 3288** | 온실 통합 제어기 ↔ **양액기 노드** 인터페이스 | 이후 확장 대상 |
| **KS X 3269** | 센서 값의 **범위·단위** | 3267은 "float 2워드"까지만 정의. 값의 의미는 여기 소관 |

이 구분이 중요한 이유: 검증 중에 "이 필드는 왜 디폴트 맵에 자리가 없지?" 하는 순간이 반복해서 온다.
답은 대부분 **"그건 3267이 아니라 3286 소관"** 이다. 표준이 못 만든 게 아니라 의도적으로 위임한 것이다.

---

## 3. 시스템 구성 — 무엇을 만들었나

### 3.1 역할 분담

```
                    ┌──────────────────────────┐
   MQTT / 웹 UI ───▶│  온실 통합 제어기 (MASTER) │
                    │        ESP32-S3          │
                    └───┬──────────────────┬───┘
                RS485 BUS#1          RS485 BUS#2
                        │                  │
                  ┌─────▼─────┐      ┌─────▼─────┐
                  │ 센서 노드  │      │ 구동기 노드 │
                  │ ESP32-C3  │      │ ESP32-C3  │
                  │  (SLAVE)  │      │  (SLAVE)  │
                  └───────────┘      └───────────┘
```

| 역할 | 보드 | 표준 ProductType |
|---|---|---|
| 온실 통합 제어기 (master) | ESP32-S3 | — |
| 센서 노드 (slave) | ESP32-C3 | 1 (센서 노드) |
| 구동기 노드 (slave) | ESP32-C3 | 2 (구동기 노드) |
| HMI | ESP32-P4 | (UART 링크, 표준 범위 밖) |

- 물리 계층: RS485, **9600 bps** (KS X 3267 §4.1 표준 기본), Modbus RTU, slave 주소 1 (기본, 노드별 재플래시로 변경)
- 타입별로 버스를 분리 (센서 = BUS#1, 구동기 = BUS#2)
- 두 노드 모두 **디폴트 레지스터 맵 사용 노드** (기관코드·회사코드 0)
- RS485 결선: M12 4핀 — 1=24VDC, 2=A, 3=B, 4=GND. **GND 공통은 필수** (안 잡으면 간헐적 CRC 에러로 고생한다)

### 3.2 대전제 — 반드시 마스터가 묻고 노드가 답한다

표준의 통신 모델은 단순하다.

- **제어기(MASTER)가 요청(폴링)하고, 노드(SLAVE)가 응답한다.** 노드가 먼저 말하는 경우는 없다.
- 모드버스 **직렬 회선 유니캐스트 모드**만 사용한다.
- 주소 0(브로드캐스트)은 **사용하지 않는다.**

즉 슬레이브는 **수동적인 메모리 맵**이다. 요청받은 주소의 값을 그대로 돌려줄 뿐,
마스터가 무엇을 원하는지 알지 못한다. 이 한 줄이 뒤에 나올 설계 전부를 규정한다.

---

## 4. 프로토콜 기초 — PDU, 기능코드, 워드 인코딩

### 4.1 공통 PDU

```
Address field | Function Code | Data | CRC
```

- **Function Code**: 서버가 수행할 액션의 종류. 값 범위는 1~255이며, **128~255는 예외 응답용으로 예약**되어 있다. 0은 사용하지 않는다.
- **Data**: 레지스터 주소, 다룰 항목의 수, 실제 데이터 바이트 수 등 기능 코드 수행에 필요한 부가 정보.
- PDU 프레임 최대 크기: **256 바이트**

실제로 쓰는 기능 코드는 사실상 둘이다.

| FC | 이름 | 용도 |
|---|---|---|
| `0x03` | Read Holding Registers | 모든 조회 (노드 정보, 디바이스 코드, 센서값, 구동기 상태) |
| `0x10` | Write Multiple Registers | 모든 제어 (구동기 명령, 노드 제어권 변경) |

`0x03` 요청의 Data에는 시작 주소(start address)와 레지스터 개수(quantity of registers)가 들어간다.
응답의 Data에는 **바이트 수**(레지스터 값 부분의 길이)와 **레지스터 값**(요청 시작 주소부터 요청한 개수만큼, 각 2바이트)이 들어간다.

### 4.2 1 워드 = 2 바이트 = 16 비트

여기서 첫 번째 함정이 나온다. 온도값 하나가 float(4바이트)라면 레지스터 2개에 걸친다.
**두 개 이상의 레지스터에 걸치는 값의 바이너리 인코딩 방식(word order)** 은 표준이 못 박아 놓았다.

- **한 워드 내부 바이트 순서 → 빅 엔디언(big-endian)**
- **워드 간 순서 → 리틀 엔디언(little-endian)**

말로만 보면 헷갈리니 실제 값으로 보자. 28.8℃를 보낸다고 하면:

```
28.8  →  IEEE-754 float32  →  0x41E6 6666

  워드 내부는 BE, 워드 간은 LE  →

  reg[n]     = 0x6666   ← 하위 워드가 먼저
  reg[n + 1] = 0x41E6   ← 상위 워드가 나중
```

디코딩은 이렇게 된다.

```python
# reg[n] = low word, reg[n+1] = high word
raw = (regs[n + 1] << 16) | regs[n]
value = struct.unpack('>f', struct.pack('>I', raw))[0]
```

**이걸 반대로 구현해도 자기들끼리는 완벽하게 동작한다.** 마스터와 슬레이브 코드를 같은 사람이 짰기 때문이다.
타사 장비를 붙이는 순간에야 터진다 — 표준 검증이 필요한 이유가 정확히 이것이다.

### 4.3 1-based와 0-based

- **KS X 3267의 레지스터 번호는 1-based** (레지스터 1번부터 시작)
- **모드버스 와이어 프로토콜은 0-based**

```
wire_start = (KS 레지스터 번호 − 1)
```

문서에 "레지스터 203"이라고 적혀 있으면 실제 패킷에는 `0x00CA`(202)가 실린다.
이 변환을 한 군데에만 두고(마스터의 요청 생성 지점), 그 경계를 넘나드는 코드에서는
전부 KS 번호로만 이야기하도록 정리했다. 안 그러면 오프 바이 원 지옥이 열린다.

> **팁:** 로그를 찍을 때 `ks_reg=203..205 wire_start=0x00CA` 처럼 **둘 다** 찍으면
> 검증할 때 눈으로 바로 대조된다. 실제로 이 로그 포맷 하나가 검증 속도를 크게 바꿨다.

---

## 5. 레지스터 맵 — 디폴트 맵이라는 약속

### 5.1 두 갈래

표준은 노드를 두 종류로 나눈다.

| 구분 | 레지스터 맵 | 마스터가 알아내는 방법 |
|---|---|---|
| **디폴트 레지스터 맵 사용 노드** | 표준 부속서 A에 고정 | **미리 알고 있다.** 기관코드·회사코드가 0이면 이 노드 |
| **자동등록 기능 지원 노드** | 제조사가 임의로 구성 | KS X 3286 절차로 맵을 조회해서 알아낸다 |

이번 구현은 **디폴트 레지스터 맵 노드**를 대상으로 했다. 온실 통합 제어기는
디폴트 맵의 **시작 레지스터 주소, 각 주소의 의미, 크기를 사전에 알고 있어야 한다.**
그래서 마스터에 맵 상수를 박아두고, 노드 타입만 확인되면 바로 정확한 주소를 때릴 수 있다.

### 5.2 주요 영역 (디폴트 맵 기준)

| 레지스터 | 내용 | 방향 |
|---|---|---|
| 1~8 | 노드 정보 (기관코드, 회사코드, 제품타입, 제품코드, 프로토콜버전, 채널수, 시리얼번호 uint32) | 읽기 |
| 101~(100+채널수) | 부착 디바이스 코드 목록 (0x00 = 미부착) | 읽기 |
| 202 | 노드 상태 | 읽기 |
| 203~ | **센서**: 채널당 3워드 `[값 float 2W + 상태 1W]` | 읽기 |
| 201, 203~ | **구동기**: status 블록, 채널당 4워드 `[OPID, 상태, 남은동작시간 uint32]` | 읽기 |
| 501~ | 노드 제어 (제어권 변경 등) | 쓰기 |
| 503~ | **구동기**: cmd 블록, 채널당 4워드 `[제어명령, OPID, 파라미터 uint32]` | 쓰기 |
| 567~ | **구동기**: 개폐형 cmd 블록 (채널 17번부터 = 개폐기 영역) | 쓰기 |

> **주의 — 비대칭:** status 블록은 `[OPID, 상태, ...]` 로 **OPID가 먼저**인데,
> cmd 블록은 `[제어명령, OPID, ...]` 로 **명령이 먼저**다. 표준이 대칭이 아니다.
> 이걸 "통일하는 게 깔끔하지" 하고 맞춰버리면 표준 위반이 된다. (→ [10장 #1](#비적합-1--cmd-블록-필드-순서-역전))

### 5.3 그리고 방향성 — 읽기 영역에 쓰면 안 된다

노드 정보·디바이스 코드·상태 정보는 **응답(읽기) 영역**이다. 마스터가 여기에 write를 시도하면
노드는 이를 **거부해야 한다.** 처음 구현에서는 이걸 놓쳤다. (→ [10장 #3](#비적합-3--응답-영역-write-미보호))

---

## 6. 초기화 시퀀스 — 노드를 발견하고 정체를 파악하기

### 6.1 노드 주소 스캔

가장 먼저 마스터가 노드 주소를 스캔한다. 부팅 시 1회로 끝내지 않고, **동작 중에도 주기적으로 다시 스캔**한다.
현장에서 노드가 추가/교체되기 때문이다. 이 단계에서 노드의 주소와 응답 여부를 획득한다.

### 6.2 노드 정보 조회 (레지스터 1~8)

```
요청:  FC 0x03, wire_start=0x0000 (KS reg 1), count=8
응답:  03 10 <16 bytes>          (byte count 0x10 = 16 = 8W × 2)
```

| 레지스터 | 의미 | 확인 포인트 |
|---|---|---|
| 1 | 기관코드 (CertAuthority) | 0 → 디폴트 맵 노드 |
| 2 | 회사코드 (CompanyCode) | 0 → 디폴트 맵 노드 |
| 3 | **제품타입 (ProductType)** | 1=센서, 2=구동기 |
| 4 | 제품코드 (ProductCode) | |
| 5 | 프로토콜 버전 | 10 |
| 6 | **채널 수 (ChannelNumber)** | 이후 모든 조회 길이의 기준 |
| 7~8 | 시리얼 번호 (uint32) | 워드 간 리틀 엔디언 |

**여기서 얻은 제품타입과 채널 수가 이후 모든 통신의 전제**가 된다. 타입으로 디폴트 맵의 어느 쪽(센서/구동기)을
쓸지 결정하고, 채널 수로 다음 단계의 조회 길이를 정한다.

### 6.3 디바이스 코드 획득 (레지스터 101~)

```
요청:  FC 0x03, wire_start=0x0064 (KS reg 101), count=채널수
응답:  채널 순서대로 디바이스 코드 목록. 0x00 = 미부착
```

이 단계가 이 프로토콜 설계의 핵심을 드러낸다.

> **모드버스 요청에는 device_code도 type도 실리지 않는다.**
>
> 요청 패킷은 `[addr][FC][start][quantity][CRC]` 뿐이다. "이 채널은 EC 센서다" 같은 정보는
> **요청에 실을 자리가 없다.** 슬레이브는 요청받은 주소의 메모리를 그대로 응답할 뿐이다.
>
> → 그래서 device_code는 **슬레이브가 reg 101~에 미리 박아둔 데이터**이고,
> 마스터가 이 조회로 **읽어서** 노드 구성을 파악한다.

이 사실을 이해하고 나면 검증 전략도 바뀐다. "EC 센서 전용 펌웨어"를 따로 만들 필요가 없다.
**디폴트 맵 노드 한 대가 이미 채널마다 다른 센서**로 구성돼 있으므로,
디바이스 코드 조회 한 번 + 센서값 조회 한 번으로 여러 센서 타입을 동시에 검증할 수 있다.

### 6.4 폴링 루프

정체가 파악되면 주기 폴링(5초)에 등록한다. 마스터는 센서 노드에는 값 조회를,
구동기 노드에는 상태 조회를 반복하고, 사용자 명령이 들어오면 그 사이에 제어 명령을 끼워 넣는다.

> **stale 주의:** 슬레이브의 파라미터(채널 수, 타입)를 바꾸면 마스터의 노드 레지스트리가 낡은 값을 들고 있게 된다.
> 주기 재검증(reverify)으로 자동 회복되게 만들었지만, 개발 중에는 마스터를 리셋해서 fresh scan을 도는 게 빠르다.

---

## 7. 센서 노드 구현

### 7.1 페이로드 구조

센서 채널의 페이로드는 **모든 센서 타입이 동일**하다.

```
채널당 3워드:  [ 값 (float32, 2W) ][ 상태 (uint16, 1W) ]

reg 203~205  ch1
reg 206~208  ch2
reg 209~211  ch3
    ...
```

타입별로 달라지는 것은 딱 두 가지다.
1. reg 101~의 **디바이스 코드**
2. 값의 **범위와 단위** — 그런데 이건 **KS X 3269 소관**이다.

즉 KS X 3267 관점에서 "센서 타입 전수 검증"이란
**"디바이스 코드를 정확히 보고하는가 + 값/상태 인코딩이 적합한가"** 를 타입마다 반복하는 것을 의미한다.

### 7.2 실제 응답 예

디폴트 맵 4채널 구성(온도/습도/CO2/토양함수율)의 실측:

```
reg101..104 = [1, 2, 11, 14]        ← 온도 / 습도 / CO2 / 토양함수율

ch1 reg203..205  words=[0x970a, 0x41df] → 27.95   status=0   (온도)
ch2 reg206..208  words=[0xc203, 0x4292] → 73.38   status=0   (습도)
ch3 reg209..211  words=[0x0a88, 0x4447] → 796.16  status=0   (CO2)
ch4 reg212..214  words=[0xdb26, 0x4208] → 34.21   status=0   (토양함수율)
```

4채널 모두 디바이스 코드에 **물리적으로 타당한 값**이 나왔다 — 워드 인코딩이 맞다는 강한 정황이다.
하지만 "정황"으로 PASS를 줄 수는 없다.

### 7.3 정황이 아니라 증명으로 — 고정값 모드

더미 센서 값이 사인파로 계속 변하면 바이트 단위 대조가 불가능하다.
그래서 펌웨어에 **고정값 모드**를 두고 전 채널을 28.8로 고정해 재플래시했다.

```
기대:  28.8 = IEEE-754 0x41E66666
       → 워드 인코딩 적용 시  reg[n]=0x6666(하위), reg[n+1]=0x41E6(상위)

실측:  전 채널  [0x6666, 0x41E6]   ← 일치
```

이제 "워드 내부 빅 엔디언 + 워드 간 리틀 엔디언"이 **증명**되었다.

> **검증 모드 설계 원칙:** 펌웨어에 넣을 검증용 옵션은 **device_code와 직교(orthogonal)한 것만** 필요하다.
> - 고정 값 모드 — float 인코딩 바이트 대조용
> - 고정 상태 모드 — 비정상 상태코드 파싱 검증용 (더미는 항상 READY라 못 만든다)
> - 노드 타입 / 주소 / 보레이트 / 채널 수 — 노드 자체의 정체
>
> 반면 "센서 타입별 전용 펌웨어" 같은 건 필요 없다. 디폴트 맵이 이미 다양한 구성이기 때문이다.

---

## 8. 구동기 노드 구현 — cmd/status 블록과 OPID

### 8.1 두 종류의 구동기

| 종류 | 예시 | 제어 명령 |
|---|---|---|
| **스위치형** | 팬, 펌프, 조명 | ON / OFF / 지정기간 작동 / 방향·강도 지정 작동 |
| **개폐형** | 천창, 측창, 커튼 | 열림 / 닫힘 / 멈춤 / 시간제어 열림·닫힘 / 개방 비율 / 시간 설정 |

디폴트 맵에서는 앞쪽 16채널이 스위치형, 그 뒤가 개폐형이다.
개폐기 채널을 검증하려면 노드의 채널 수를 24(스위치 16 + 개폐기 8)로 늘려 재플래시해야 했다.

### 8.2 명령 코드

| operation | 의미 | 종류 |
|---|---|---|
| `0` | STOP / OFF | 공통 |
| `201` | ON (작동 시작) | 스위치형 |
| `202` | TIMED_ON (지정기간 작동) | 스위치형 |
| `203` | DIRECTIONAL_ON (시간·방향·강도) | 스위치형, 레벨2 |
| `301` | OPEN (열림) | 개폐형 |
| `302` | CLOSE (닫힘) | 개폐형 |
| `303` | TIMED_OPEN (시간제어 열림) | 개폐형 |
| `304` | TIMED_CLOSE (시간제어 닫힘) | 개폐형 |
| `305` | SET_POSITION (개방 비율) | 개폐형, 레벨2 |
| `306` | SET_CONFIG (열림/닫힘 시간 설정) | 개폐형, 레벨2 |

### 8.3 제어의 왕복 구조

모든 구동기 제어는 **쓰고 → 읽어서 확인**하는 왕복이다.

```
① 마스터 → 노드   FC 0x10 으로 cmd 블록에 쓴다
     reg 503 = [ 제어명령(operation), 명령ID(opid), 파라미터(uint32) ]

② 노드            응답 = 시작주소 + 개수 에코 (모드버스 규격)

③ 마스터 → 노드   FC 0x03 으로 status 블록을 읽는다
     reg 203 ← [ OPID 에코, 상태(status), 남은동작시간(uint32) ]

④ 대조            보낸 opid가 그대로 돌아왔는가? 상태가 명령대로 바뀌었는가?
```

실제 캡처:

```
[SW-4 작동 시작 ON]
  송신 cmd    reg503 = [op=201, opid=7777, arg=0]
  회신 status reg203 = [opid=7777, status=201(ON), remain=0]     PASS

[SW-6 지정기간 작동 TIMED_ON]
  송신 cmd    reg503 = [op=202, opid=9301, arg=6]
  회신 status  remain 추이: t+2s=5 → t+4s=3 → t+6s=1 → t+8s=0 → 자동 OFF   PASS
```

### 8.4 OPID — 명령의 정체성

이 프로토콜에서 제일 재밌는 설계가 **OPID(명령 ID)** 다.

폴링 기반 프로토콜에는 근본적인 문제가 있다. 마스터는 계속 같은 주소에 명령을 쓴다.
그러면 노드는 **"이게 새 명령인가, 아니면 아까 그 명령을 마스터가 또 쓴 건가?"** 를 구분할 수 없다.

표준의 답: **명령마다 OPID를 바꾼다. 그리고 OPID가 바뀌는 시점이 곧 명령이 활성화되는 시점이다.**

```c
/* 노드 측 판정 — opid가 이전과 다르고 0이 아닐 때만 새 명령으로 처리 */
if (cmd_opid != prev_opid && cmd_opid != 0) {
    apply_command(op_code, cmd_opid, arg);
    prev_opid = cmd_opid;
}
```

노드는 받은 OPID를 status 블록에 그대로 에코한다. 마스터는 이 에코로
**"내 명령이 도달했고 적용됐다"** 를 확인한다. ACK 프레임이 따로 없는 폴링 세계에서의
우아한 해법이다. 실측 로그에서도 명령마다 opid가 바뀌는 게 그대로 보인다.

```
12276 → 12277 → 62087 → 62088 → 5101 → 5102 → 46980 → 46981 → 46982
```

---

## 9. 검증 방법론 — "눌러서 확인"에서 벗어나기

### 9.1 문제: HMI 버튼으로는 검증이 안 된다

처음에는 HMI 화면의 버튼을 눌러 명령을 유발하고 시리얼 로그를 캡처했다.
곧 한계가 왔다.

- 원하는 파라미터 조합(예: `op=305, position=37`)을 만들 버튼이 없다
- 재현이 안 된다. 같은 순서로 다시 누를 수가 없다
- 캡처가 불안정하다. C3의 USB-Serial-JTAG가 write burst 로그를 드롭한다

### 9.2 해결: 검증자가 서버 역할로 명령을 주입

제어기는 이미 MQTT로 상위 서버와 통신한다. 여기에 **검증자가 직접 붙어서 임의의 모드버스
요청을 주입**하는 방식으로 바꿨다.

```
검증자 ──MQTT publish──▶ 온실 통합 제어기 ──RS485 modbus 요청──▶ 노드(slave)
       ◀──MQTT ack────                    ◀──RS485 modbus 응답──
```

- `mb_read`  → 제어기가 FC 0x03으로 노드에 조회
- `mb_write` → 제어기가 FC 0x10으로 노드에 제어 명령 송신
- 검증 스크립트가 요청 생성 → 응답 파싱 → 기대값 대조를 자동으로 수행

이걸로 검증이 **결정적이고 반복 가능**해졌다. 임의의 레지스터에 임의의 값을 쓸 수 있으니
예외 응답 검증(범위 밖 주소, 읽기 전용 영역 write)도 이 경로로 전부 가능하다.

### 9.3 바이트 레벨 로그

양쪽에서 같은 트랜잭션을 서로 다른 시점으로 찍어 대조했다.

```
[master 측]
D MB_MASTER: TX  bus2→slv1  FC=10 WR_MULTI  ks_reg=515..518  wire_start=0x0202 cnt=0x0004
                payload= f4 2f c9 00 00 00 00 00
D MB_MASTER: RX  bus2←slv1  FC=10 ok (echo start=0x0202 cnt=0x0004)

[slave 측]
I MBS_REQ: WR  FC=0x10  ks_reg=503..506 (4W)  wire_off=0x01F6  area=Act cmd(503~506): ch1
I ACTUATOR: ch4 opid=12276 op=201 → status=201 remain=0s
```

`ks_reg`와 `wire_off`를 함께 찍은 것, 그리고 **영역 이름(`area=`)** 을 로그에 넣은 것이
결과적으로 가장 유용했다. 주소만 봐서는 그게 어느 블록의 몇 번 채널인지 매번 계산해야 한다.

### 9.4 검증 순서

의존 관계를 따라 아래에서 위로 올라가는 순서로 진행했다.

1. **프로토콜 기반** — 정상 흐름, 워드 인코딩, 주소 오프셋. 이후 모든 명령의 전제
2. **노드 정보 + 디바이스 코드** — 디폴트 맵 노드 1대로 채널별 다른 디바이스 코드 동시 확인
3. **노드 상태 + 센서 조회** — 채널마다 다른 센서지만 읽는 방식은 동일함을 확인
4. **센서 정밀 검증** — 고정값 모드로 바이트 대조
5. **구동기 제어 계열** — 각 제어 후 상태 조회로 opid 에코 확인
6. **레벨1 노드 상태 / 제어권**
7. **예외 응답 / 타임아웃** — 의도적으로 비정상 요청 유발
8. **노드 정체 변경 검증** — 타입·주소·채널 수를 바꿔가며 재인식 확인

---

## 10. 검증에서 깨진 것들 — 표준 비적합 4건

여기가 이 글의 본론이다. **네 건 모두 "우리끼리는 완벽하게 동작하던" 코드**였다.

### 비적합 1 — cmd 블록 필드 순서 역전

**증상:** 없었다. HMI에서 버튼을 누르면 팬이 돌았다.

**발견:** 바이트 레벨 페이로드를 표준 부속서와 한 워드씩 대조하다가 나왔다.

```
캡처된 payload:  f4 2f c9 00 00 00 00 00

  reg[0] = 0x2FF4 = 12276  ← 이게 opid
  reg[1] = 0x00C9 = 201    ← 이게 operation

표준(부속서 A.2.6):
  cmd 블록 = [ 제어명령(op) 1W, OPID 1W, 동작시간(uint32) 2W ]
  → reg[0]이 명령, reg[1]이 opid 여야 한다.
```

**원인:** 앞에서 언급한 **표준의 비대칭** 때문이다.

```
status 블록 (A.2.5) = [ OPID,     상태,  남은시간 ]   ← OPID 먼저
cmd    블록 (A.2.6) = [ 제어명령,  OPID,  동작시간 ]   ← 명령 먼저
```

구현할 때 "일관성 있게 opid를 앞에 두자"고 **양쪽을 통일**해버렸다.
그 결과 status는 우연히 표준과 맞고, cmd만 틀렸다.
그리고 마스터와 슬레이브가 **똑같이 틀렸기 때문에** 자기들끼리는 완벽하게 동작했다.

표준 준수 타사 제어기가 이 노드에 명령을 보내면? `op=201`을 쓰려 한 것이
`opid=201, op=<쓰레기>`로 해석된다. 팬은 안 돈다.

**수정:**

```c
/* 마스터·슬레이브가 같은 정의를 참조하도록 공통 헤더에 오프셋 상수를 둔다 */
#define KS_ACT_CMD_OFF_OP    0   /* A.2.6 */
#define KS_ACT_CMD_OFF_OPID  1
#define KS_ACT_CMD_OFF_ARG   2

#define KS_ACT_ST_OFF_OPID   0   /* A.2.5 — 순서가 다르다! */
#define KS_ACT_ST_OFF_STATUS 1
#define KS_ACT_ST_OFF_REMAIN 2
```

매직 넘버로 `vals[0]`, `vals[1]`을 쓰던 걸 전부 상수로 바꿨다. **표준 조항 번호를 주석에 박아서**
다음 사람이 "이거 반대 아냐?" 하고 다시 통일하지 못하게 했다.

**재검증:**

```
송신 cmd    reg503 = [op=201, opid=7777, arg=0]
회신 status reg203 = [opid=7777, status=201, remain=0]     PASS

  ← 옛 펌웨어였다면 opid=201, status=READY 로 나왔을 것.
```

> **교훈:** "일관성 있게 정리"하고 싶은 충동이 표준 위반의 주범이다.
> 표준이 비대칭이면 비대칭이 정답이다.

---

### 비적합 2 — 지정기간 명령 중 상태 코드 미반영

**증상:** TIMED_ON을 걸면 팬은 도는데 상태 조회 결과가 `0(READY)`로 나온다.

**표준:** 작동 중에는 status가 `201(ON)` / `301(OPENING)` / `302(CLOSING)` 이어야 한다.

**원인:** 상태 결정 로직이 `ON`, `OPEN`, `CLOSE` 만 분기하고
`TIMED_*`, `DIRECTIONAL_ON` 은 `default` 로 떨어져 READY가 되고 있었다.

```c
static uint16_t status_for_op(uint16_t op) {
    switch (op) {
    case OP_SW_ON:        return KS_STS_SW_ON;
    case OP_OPEN:         return KS_STS_OPENING;
    case OP_CLOSE:        return KS_STS_CLOSING;
    /* + TIMED_ON, DIRECTIONAL_ON → SW_ON
       + TIMED_OPEN → OPENING, TIMED_CLOSE → CLOSING  케이스 추가 */
    default:              return KS_STS_READY;
    }
}
```

**교훈:** `default: return READY`는 위험한 기본값이다. 새 명령을 추가할 때마다
조용히 "정상 대기 중"이라고 거짓말을 한다. 열거형 스위치에서 default를 두지 않고
컴파일러 경고에 맡기는 편이 나았다.

---

### 비적합 3 — 응답 영역 write 미보호

**증상:** 마스터가 노드의 **노드 정보 영역(reg 1)에 write를 하면 노드가 그대로 받아들였다.**

```
mb_write reg1 = 9999  →  ok 회신 + reg1이 실제로 9999로 변경됨
```

노드의 제품타입이나 시리얼 번호가 바깥에서 덮어써질 수 있다는 뜻이다.
버그 있는 마스터 하나가 노드의 정체를 영구히 망가뜨릴 수 있다.

**표준:** 노드 정보·디바이스 코드·상태 정보는 **응답(읽기) 영역**이다.

**수정:** 모드버스 영역 등록 시 접근 권한을 명시하도록 바꿨다.

| 영역 | 시작 레지스터 | 권한 |
|---|---|---|
| 노드 정보 | 1 | **RO** |
| 디바이스 코드 | 101 | **RO** |
| 구동기 status | 201 | **RO** |
| 센서 블록 | 202 | **RO** |
| 구동기 cmd | 501 | RW |

이제 RO 영역에 write 요청이 오면 모드버스 스택이 **예외 응답(0x02, illegal data address)** 으로
자동 거부한다. 노드 자신의 태스크가 상태를 갱신하는 건 이 플래그와 무관하므로 정상 동작한다.

**재검증:**

```
[구동기]
  노드정보 reg1 write     → 거부   PASS
  디바이스코드 reg101 write → 거부   PASS
  status reg203 write     → 거부   PASS
  cmd reg503 write        → 수용   PASS   ← 명령 영역은 RW가 맞다
  노드정보 / status read   → 정상   PASS

[센서]
  노드정보 reg1 write      → 거부   PASS
  센서값 reg203 write      → 거부   PASS
```

**교훈:** 모드버스 홀딩 레지스터는 이름부터가 read/write 겸용이라
**"읽기 전용"을 명시적으로 선언하지 않으면 기본값이 쓰기 가능**이다.
표준 문서의 그림에서 "응답 영역"이라고 화살표가 그려진 부분은 전부 RO여야 한다.

---

### 비적합 4 — 회사코드 0xFEED 자기선언 (default map 오인식 유발)

**증상:** 없었다. 우리 마스터는 잘 붙었다. 자체 검증도 프레임·맵 전부 표준.

**발견:** 검증 완료를 넘긴 시점에, 별도 세션에서 노드 응답을 다시 훑어보다 나왔다.
NodeInfo 응답의 **회사코드 자리에 0xFEED** 가 실려 있었다.

```
reg1 (CertAuthority)  = 0x0000   ✓
reg2 (CompanyCode)    = 0xFEED   ← ★
reg3 (ProductType)    = 1        ✓
...
```

**표준:** 부속서 A.1.1 각주 — **디폴트 레지스터 맵 노드는 기관코드·회사코드가 둘 다 0** 이어야 한다.
하나라도 non-zero 면 마스터는 이 노드를 **"KS X 3286 자동등록 대상"** 으로 판정하고
노드 스펙 파일을 요청한다. 3286 미구현 슬레이브라면 그 요청에 응답할 방법이 없다.

**원인:** 개발 초기에 회사코드 자리에 "우리 회사" 표식으로 재미 삼아 넣어둔 상수가
정리되지 않고 남아있었다. 우리 마스터는 이 값을 무시하고 부속서 A 맵으로 처리해왔기 때문에
**자체 통신에서는 아무 증상이 없었다.**

**수정:**

```
- CONFIG_KSNODE_COMPANY_CODE=0xFEED
+ CONFIG_KSNODE_COMPANY_CODE=0x0000

  Kconfig 도움말에 표준 근거 못박기:
  "KS X 3267 §5.1 + 부속서 A.1.1 각주:
     기관코드=0 AND 회사코드=0 → 디폴트 맵 노드로 판정
     하나라도 non-zero → KS X 3286 자동등록 노드로 오인식
   본 프로젝트는 §5.1.2 디폴트 맵 경로 → 회사코드는 0 유지"
```

**교훈:** 검증 도구는 **프레임 규격만 심판한다.** 응답 필드가 표준이 정의한 **의미** 에서
어긋나 있어도 프레임은 통과한다. 표준 준수는 "프레임 통과 + 필드 의미의 정확한 자기선언"
두 축이 다 맞아야 한다.

또 하나: **"우리끼리는 잘 도는 코드"** 함정이 이번에도 재현됐다. 우리 마스터가 이 값을
무시하고 정상 동작했기 때문에 무증상이었다. 남의 마스터가 물릴 때에야 실패한다.

---

## 11. 표준이 그어놓은 선 — 디폴트 맵의 한계

검증 중 반복해서 마주친 벽이 있다. 레벨2 명령들이다.

| 명령 | 검증 가능 범위 |
|---|---|
| SW-4/5/6 (ON/OFF/TIMED_ON), OC-4~8 (OPEN/CLOSE/STOP/TIMED_OPEN/TIMED_CLOSE) | **cmd → status 왕복 완전 검증** |
| SW-7 DIRECTIONAL_ON (강도), OC-9 SET_POSITION, OC-10 SET_CONFIG | 명령 전송·opid 에코는 검증됨. **status 에코 검증 불가** |

이유는 단순하다. **디폴트 맵의 구동기 status 블록은 채널당 4워드 고정**이고,
그 안에 position·ratio·state-hold-time을 담을 자리가 없다.

처음에는 "그럼 status 블록을 늘려야 하나?" 싶었다. 표준 원문을 다시 보고 답을 찾았다.

> §6.3.3.6 SET_POSITION / §6.3.3.7 SET_CONFIG 는 **"자동등록 기능을 지원하는 노드에만 해당"**

즉 이 필드들은 **KS X 3286 자동등록 노드가 임의 주소 맵으로 표현**하는 것이지,
디폴트 맵 노드가 자리를 만들어 낼 대상이 아니다.
**현재의 4워드 고정 구조가 KS X 3267 디폴트 맵의 정답이고, 늘리면 오히려 표준 위반이다.**

이걸 "미구현"이 아니라 **"표준이 다른 표준에 위임한 범위"** 로 보고 문서에 못 박아 두는 것이,
다음 사람이 같은 고민을 반복하지 않게 하는 유일한 방법이었다.

---

## 12. 삽질 로그 — 표준과 상관없이 시간을 잡아먹은 것들

표준 준수 이야기는 여기까지고, 실제로 시간을 가장 많이 먹은 건 이런 것들이었다.

### 12.1 어느 보드에 어느 펌웨어가 올라갔는지 모른다

C3 두 대(센서/구동기)가 외형도 같고 USB로 구분도 안 된다. 포트 번호는 리플래시할 때마다 바뀐다.
결국 **구동기 펌웨어가 센서 보드에 올라간 상태로 한참 검증을 진행**한 사고가 났다
(두 버스 모두 제품타입 2로 응답하는 걸 보고 발견).

**해결 — 부팅 배너와 주기 비콘:**

```
KSNODE_ID  type=SENSOR  mac=AC:A7:04:D1:8F:9C  addr=1  ch=19  baud=38400
KSID:      type=SENSOR  mac=AC:A7:04:D1:8F:9C  addr=1        (~30초 주기)
```

칩 efuse MAC이 보드별 유일 식별자다. **플래시 전에 반드시 대상 포트의 배너를 먼저 읽어
보드를 확정한다** 는 원칙을 세우고 나서 사고가 사라졌다. 포트 번호는 신뢰하지 않는다.

### 12.2 카운트다운이 안 돌던 이유

TIMED_ON을 걸면 남은 시간이 설정값에 고정된 채 줄지 않았다. ON/OFF는 멀쩡했다.

원인은 구동기 태스크가 매 루프마다 부르던 **모드버스 이벤트 확인 API가 사실 블로킹 호출**이었다는 것.
주석에는 "즉시 검사"라고 적혀 있었지만 실제로는 write 이벤트가 올 때까지 태스크를 세워두고 있었다.

- ON/OFF는 정상 — write 이벤트가 곧 태스크를 깨우니까
- TIMED의 카운트다운은 write가 없는 구간에서 돌아야 하는데, 태스크가 거기서 자고 있었다

**수정:** 그 호출을 제거했다. 1초 주기 delay가 이미 tick을 제공하고,
새 명령 감지는 **OPID 비교**로 하므로 이벤트가 애초에 필요 없었다.
쓰이지 않으면서 고장까지 나 있던 함수는 선언째로 지웠다.

```
수정 후 remain 추이:  t+2s=5 → t+4s=3 → t+6s=1 → t+8s=0  (≈1/s, 0에서 정지)
```

### 12.3 시리얼 캡처 스크립트가 보드를 다운로드 모드로 넣는다

로그 캡처용 파이썬 스크립트가 포트를 열 때 DTR/RTS를 assert 하면서
ESP32의 USB-Serial-JTAG를 **다운로드 모드로 진입**시켰다.

```
rst:0x15 (USB_UART_CHIP_RESET), boot:0x23 (DOWNLOAD)
```

포트를 열기 **전에** DTR/RTS를 명시적으로 False로 두는 걸로 해결.

### 12.4 GND

RS485는 A/B 두 선만 연결하면 될 것 같지만, **GND 공통이 없으면 간헐적으로 CRC 에러가 뜬다.**
"가끔 응답이 없다"의 원인을 프로토콜에서 찾다가 결선에서 발견하는 일이 없기를.

### 12.5 baud 불일치 (양쪽 다 우리 코드인데 놓쳤다)

마스터와 슬레이브를 같은 사람이 짜는 함정의 또 다른 얼굴이다.

- 마스터 rs485_hw.h: `baud=9600` (KS X §4.1 표준 기본)
- 슬레이브 sdkconfig.defaults: `KSNODE_BAUDRATE=38400` (성능 테스트 시절 잔여)

한동안 잘 돌던 이유는 슬레이브 firmware 를 오래 재플래시 안 했고, 마스터도 문서 주석과 달리
실제로는 38400 을 세팅해왔기 때문. 마스터 코드 정리 커밋 하나가 9600 으로 복구되는 순간
**모든 slave 가 조용해졌다.**

수정은 sdkconfig 한 줄이었는데 **"통신이 아예 안 됨"** 이라는 증상이 프로토콜 상위 문제로 오해되기 쉽다.
링크 레벨부터 의심하는 습관 — baud 는 로그에 매번 찍는다.

시판 센서를 붙일 때도 반복된다. 데이터시트에는 9600 이라고 적혀 있지만 이전 세팅이 4800 인 개체가
많다. 이런 unknown-baud 상황용 auto-scan 은 §14.4 에 정리했다.

### 12.6 값이 이상해 보이면 물리를 먼저 의심하라

토양 sensor 를 처음 붙였을 때 humidity 응답이 **0.0%** 로 나왔다. 우리 파싱 버그로 의심하고
scale 을 뒤졌다. 아무리 봐도 코드는 정상. **sensor 프로브가 공기 중이라 실제로 dry.**
물그릇에 담그자마자 100.0% 로 상승했다.

값이 예상 범위 밖일 때, **파싱 오류 → 통신 오류 → 물리 상태** 순서가 몸에 밴 습관인데
이번엔 물리부터 의심했어야 했다. Sensor 데이터는 늘 "센서가 지금 뭘 재고 있는지" 를 먼저 확인.

### 12.7 PCB 라벨과 실 GPIO 매핑이 다르면 진단이 무너진다

우리 마스터 PCB 는 슬레이브 커넥터에 IO1~IO5 라벨이 있다. Slave 를 "IO5" 커넥터에 꽂았는데
addr=2 로 배정됐다. **표시된 라벨과 실 GPIO 배선이 어긋난 상황을 의심하지 못했다** —
발견한 방법은 다음과 같다.

Arduino Uno 를 임시로 마스터 5 GPIO 에 붙이고 100Hz 로 각 pin 상태를 시리얼에 찍는 sketch 를 올렸다.
Slave 는 잠깐 빼고 Uno 만으로 관측:

```
[Uno]  IO12345=10000  active_ports=1     ← master GPIO1 만 HIGH
       IO12345=01000  active_ports=2     ← GPIO2
       ...
       IO12345=00001  active_ports=5     ← GPIO5
```

**마스터 GPIO drive 는 완벽 순차, crosstalk 0.** PCB 라벨과 실 GPIO 매핑도 정상.
문제는 **사용자가 IO5 라벨 pin header 로 인지한 자리에 실은 slave 케이블이 다른 pin 에 꽂혀있던 것.**
Uno 진단이 아니었다면 firmware 를 몇 시간 뜯었을 것이다.

교훈: **관측 도구가 있으면 물리 배선의 진실도 이진 판별로 좁혀진다.** Sketch 하나로 관측
가능한 상태를 만들면, hypothesis-driven 디버깅으로 급격히 넘어간다.

---

## 13. 정리와 남은 일

### 13.1 검증 결과

디폴트 레지스터 맵 범위의 표준 명령을 전수 검증했고, 전 항목 PASS.

| 계열 | 항목 | 결과 |
|---|---|---|
| 노드 정보 | 노드 정보 조회, 디바이스 코드 조회, 노드 상태 조회, 노드 제어권 변경 | PASS |
| 센서 | 센서 상태 정보 조회 (+ 고정값 바이트 대조) | PASS |
| 스위치형 구동기 | ON / OFF / TIMED_ON / DIRECTIONAL_ON | PASS |
| 개폐형 구동기 | OPEN / CLOSE / STOP / TIMED_OPEN / TIMED_CLOSE / SET_POSITION / SET_CONFIG | PASS |
| 프로토콜 | 정상 응답 흐름, 예외 응답, 워드 인코딩, 주소 변환, OPID 규칙 | PASS |

검증 중 그리고 직후 재검토에서 발견한 **표준 비적합 4건은 전부 수정·재검증 완료**.

### 13.2 남은 일

- **센서 19종 전수 스윕** — 센서 노드를 19채널(디바이스 코드 1~19와 1:1 대응) 구성으로
  플래시하고, 프론트엔드에 검증 위젯을 붙여 `디바이스 코드 → 센서명 + 값 + 상태` 를
  한 표로 일괄 조회할 수 있게 준비를 마쳤다. 남은 것은 실행하고 기록하는 것.
- **레벨1 / 레벨2 노드 상태** — 노드 레벨을 올린 구성에서 추가 검증
- **KS X 3286 자동등록 노드** — 레벨2 status 항목의 완전 검증은 여기서만 가능
- **KS X 3288 양액기 노드** — 다음 확장 대상

### 13.3 이 프로젝트에서 얻은 것

**1. "동작한다"는 표준 적합의 증거가 아니다.**
네 건의 비적합 중 세 건은 증상이 전혀 없었다. 마스터와 슬레이브를 같은 사람이 짜면
같은 오해를 양쪽에 심는다. 그리고 그 오해는 **자기들끼리는 완벽하게 상쇄된다.**

**2. 표준의 비대칭은 대개 의도된 것이다.**
"일관성 있게 정리하자"는 충동이 가장 위험했다. 표준이 이상해 보이면 먼저 원문을 다시 읽는다.

**3. 로그 포맷이 검증 속도를 결정한다.**
`ks_reg`와 `wire_off`를 같이 찍고, 영역 이름을 붙인 로그 한 줄이
"이 주소가 어느 블록 몇 번 채널이더라"를 매번 계산하는 시간을 통째로 없앴다.

**4. 검증은 재현 가능해야 한다.**
버튼을 눌러 확인하는 방식으로는 3건 중 한 건도 못 찾았을 것이다.
임의의 레지스터에 임의의 값을 쓸 수 있는 주입 경로를 만든 순간부터 검증이 실제로 시작됐다.

---

*이 글은 자체 개발 펌웨어의 KS X 3267:2022 적합성 검증 기록을 정리한 것이다.
표준 원문의 조항·표 번호는 검증 보고서를 참조.*

---

## 14. 검증 이후 — 표준을 지키면서 편의를 얻는 법

검증이 끝나고 나면 실제 운영에서 나오는 요구가 다시 시작된다.
"슬레이브 5대를 출고했는데, 각각 addr 을 손으로 flash 하는 건 안 하고 싶다."
"어제 여기 있던 노드가 오늘 다른 자리에 있다면 어떻게 알까."
"HMI 화면에 사라진 노드 카드가 계속 남아있다."

전부 **표준이 답을 안 주는 편의 문제** 다. 그러나 답을 만들다 표준을 부순다면
검증한 게 무의미해진다. 이 장은 **표준의 회색지대만 골라 밟은 방법들** 이다.

### 14.1 Discovery — §5.1 이 남긴 자유

KS X 3267 §5.1 은 명시적으로 말한다.

> "이 표준은 슬레이브 주소가 어떻게 부여되는지 규정하지 않는다.
>  DIP 스위치, 설정 명령, KS X 3286 자동등록 어느 방식이든 자유."

이 문장이 열어놓은 자유를 우리는 **마스터-주도 per-port GPIO discovery** 로 채웠다.

> **잠깐, 3286 이 자동 부여 안 하나?** 원문을 다시 읽어보니 3286 §5.2.a 도 이렇게 말한다.
> *"각 노드의 모드버스 주소는 관리자가 자체 방식 (딥 스위치 등) 으로 미리 지정되어 있음을 가정."*
>
> 즉 **3286 조차 addr 부여는 다루지 않는다.**  3286 의 자동화는 회사/제품 코드로부터 규격 JSON 을
> 조회해 **레지스터맵을 계산** 하는 것이지, addr 을 부여하는 것이 아니다.  두 표준 다 "addr 은
> 이미 있다" 를 전제한다.  우리의 discovery 는 그 전제를 **우리 세트에서 자동으로 만족** 시키는 편의.

**하드웨어**: 마스터 PCB 에 슬레이브 커넥터 5 포트가 있다. 각 포트에 RS485 A/B/GND 외에
**GPIO 신호선 1개** 를 추가로 뽑았다 (마스터 GPIO1~5). 슬레이브 쪽에서는 이 신호선을 GPIO7 로 받는다.

**프로토콜**: 표준 함수코드만 사용.

```
① master GPIO_N = HIGH  (다른 포트는 LOW)
② 50 ms 대기 (slave GPIO7 edge 안정)
③ FC 0x03 read  addr=247  reg 1~8    ← 임시 addr 247 은 discovery 예약
④ 응답 있음 → serial 획득
⑤ FC 0x06 write addr=247  reg 250 = N   ← sentinel: "너는 이제 addr N"
⑥ slave NVS 저장 후 addr=N 로 재기동
⑦ FC 0x03 read addr=N  reg 7~8       ← serial 재확인 (verify)
⑧ master registry (bus, addr=N, serial) 등록, port_N LOW
```

임시 addr **247** 은 KS X §4.1 유니캐스트 범위 (1~247) 상한을 discovery 예약으로 못박은 것이고,
sentinel reg **250** 은 부속서 A 미할당 영역이라 vendor-specific 사용이 허용된다.
**정상 운영 사이클에서 이 두 값은 절대 나오지 않는다** — 배정 후엔 완전히 표준 맵으로 통신.

**표준 준수 검증**: 남의 마스터가 이 slave 를 물면 어떻게 될까? 회사코드 = 0 이므로 마스터는 부속서 A
디폴트 맵으로 처리하려 한다. Slave 는 addr=N 으로 정상 응답. GPIO7 신호는 물론 안 오지만 그 상태로도
**표준 통신은 완전 정상**. 즉 discovery 는 **plug-in 확장** 이지 slave 의 표준 준수를 침범하지 않는다.

### 14.2 3-tier fallback — 검증 환경과 운영 환경의 공존

Slave 부팅 시:

```c
if (slave_addr_load_from_nvs(&addr)) {
    // tier 1: NVS 저장값 (우리 discovery 후 정상 운영)
} else if (CONFIG_KSNODE_SLAVE_ADDR != 0) {
    // tier 2: Kconfig fixed (표준 시험 / 외부 마스터 조합)
    // → GPIO7 신호 감시 자체를 켜지 않는다
} else {
    // tier 3: unassigned — GPIO7 rising edge 대기 후 discovery 프로토콜 수행
    discovery_wait_and_assign(&addr);
}
mb_slave_start(addr);
if (tier != 2) discovery_start_reassign_watcher();
```

**tier 2 (fixed) 에서 watcher 를 안 켜는 게 핵심**이다. 표준 시험 환경에서 slave 의 addr 은
tester 가 지정한 값으로 절대 안 바뀌어야 한다. GPIO7 에 우연히 신호가 들어와도 무시.

이 3-tier 는 **한 코드 트리로 두 목적 (우리 세트 + 표준 시험) 을 동시 지원** 한다.
이 원칙이 없었다면 검증용 펌웨어와 운영용 펌웨어를 분리하고 매번 다른 걸 flash 해야 했을 것이다.

### 14.3 in-place reassign — `esp_restart()` 를 왜 안 썼나

Slave 를 다른 포트로 이동하면 재배정이 필요하다. 첫 구현:

```c
// GPIO7 rising 감지 → NVS clear → esp_restart()
// 부팅 후 unassigned tier 로 진입 → 다시 discovery
```

문제: **재부팅에 200 ms 걸린다.** 마스터의 rescan 사이클이 5 포트 순차라 각 포트 사이가 500 ms 정도.
Slave 가 restart 중일 때 마스터는 이미 다음 포트로 넘어가 있고, slave 가 unassigned 진입한 시점엔
자기 포트 pin 이 이미 LOW. **다음 10 초 rescan 사이클까지 대기.**

Timing race 를 아예 소멸시키는 fix — **in-place swap**:

```c
// GPIO7 rising 감지
slave_addr_clear();                      // NVS
mb_slave_stop();                         // esp-modbus 만 stop
mb_slave_start_ephemeral();              // 임시 addr 247 로 즉시 재기동
                                          // (sensor task 등은 계속 running)

while (mb_slave_sentinel_new_addr() == 0) {
    vTaskDelay(50);                      // master 가 sentinel write 할 때까지
}
uint8_t new_addr = mb_slave_sentinel_new_addr();
slave_addr_save(new_addr);
mb_slave_stop();
mb_slave_start(new_addr);                // 새 addr 로 normal 재기동
```

Restart 없음. 20~50 ms 안에 ephemeral 진입. 마스터가 아직 포트 pin HIGH 유지 중일 때 slave 가 응답 →
같은 rescan 사이클에서 배정 완료. **재부팅의 관성이 사라지니 사용자 관점 hot-swap 이 매끄러워졌다.**

### 14.4 실 sensor 통합 — addr 자동 change 는 register 후보 순차 시도

시판 3-in-1 토양 sensor (humidity/temp/EC) 를 붙이는 워크플로우:

```
1. baud 후보 = [9600, 4800]
2. baud 별로 addr 1..50 scan (FC03 reg 0 humidity 시도)
3. 응답 발견 → 실 addr 확정
4. target_addr (Kconfig, default 1) 과 다르면 addr change 시도:
     후보 register [0x07D0, 0x0100, 0x0101, 0x0200, 0x1000] 순차 FC06 write
5. 새 addr 로 응답 확인되면 NVS 저장 → 이후 부팅 skip scan
```

Register 후보 리스트는 시판 sensor 여러 vendor 의 datasheet 를 훑어 정리한 것이다.
우리 sensor 는 **0x07D0 첫 시도로 성공**. NVS 저장 후 재부팅 시 tier1 로 skip.

**Register dump 진단**: 부팅 후 첫 read 때 reg 0..3 raw 값을 log 로 찍는다.

```
[SOIL] raw register dump [addr=1, wire_reg 0..3]:
  reg 0 = 1000  (0x03E8)  ← humidity 100.0% (0.1% scale)
  reg 1 =  259  (0x0103)  ← temperature 25.9℃ (0.1℃)
  reg 2 =   21  (0x0016)  ← EC 21 μS/cm
  reg 3 =   58  (0x003A)  ← reserved / vendor
```

Sensor 별 scale 이 datasheet 예상과 다를 수 있어 **처음 한 번 raw 를 눈으로 대조**하게 했다.
운영 자동화만 만들어놓으면 배포 초반 인수인계에서 반드시 사고가 난다.

### 14.5 Lifecycle audit — serial 기반 이력

Slave 를 addr 만으로 관리하면 하나의 관점을 놓친다.
**"어제 addr=3 이었던 slave 가 오늘 addr=5 로 나타났다면 이건 이동인가, 교체인가?"**

Master registry 에 `(bus, addr, serial=MAC 파생 uint32)` 를 저장하고, discovery 이벤트마다
**serial 로 이전 등록을 조회**한다.

```c
ks_node_t *existing = ks_registry_find_by_serial(serial_new);
if (!existing)                                           evt = ADDED;
else if (existing->bus == bus && existing->addr == N)    evt = REASSIGN;   // 같은 위치 재확인
else { evt = MOVED; from = existing->addr;               // 다른 위치 이동
       ks_registry_remove(existing); }                   // old entry cleanup
```

이 이벤트를 MQTT audit topic 으로 uplink:

```
smartfarm/controller/<mac>/discovery
  {"event":"moved","serial":"0x8D296DD8","bus":2,
   "from_port":3,"to_port":5,"addr":5,"ts_ms":...}
```

Backend (Rust + PostgreSQL) 는 이걸 `device_audit` 테이블에 timeseries 로 저장 +
WebSocket 으로 프론트엔드 브로드캐스트. SvelteKit 대시보드에 **최근 lifecycle** 위젯이
실시간 업데이트되고 `/device-audit` 페이지에 전체 timeline 이 표시된다.

**설계 원칙**: Discovery 프로토콜 (firmware) 은 표준 프레임만 사용, audit 프로토콜은
**표준의 위에 얹은 layer** 로 완전 분리. Firmware 는 이벤트 생성만, 저장·표시는 상위 layer.
이래야 표준 검증 시 audit layer 를 통째로 꺼도 firmware 는 표준에 남는다.

### 14.6 P4 HMI 카드 slot 관리 — 카드가 사라지지 않는다는 불만

HMI 는 slave 를 카드로 표시한다. 첫 구현은 `slot=1` 만으로 카드 인덱싱했더니 여러 slave 가
같은 카드[0] 를 덮어쓰는 사고 (slv1 표시가 곧바로 slv3 로 바뀜). 개선:

- **카드 pool 매핑**: `(bus, slave, ch)` triple 로 lookup, 없으면 첫 UNUSED slot 할당
- **정렬**: 새 카드 등장 시 `(bus, slave, ch)` 기준 재정렬 (LVGL flex `lv_obj_move_to_index`)
- **2-tier stale**:
  - 15 s soft — 회색 처리 (일시 disconnect 대응)
  - 60 s hard — 카드 hidden + slot 반환 (다른 slave 재사용 가능)

Hard stale 이 없으면 이전 dummy 슬레이브가 회색으로 계속 남아 clutter. Slot 반환 후에는
새 slave 등장 시 그 자리를 재활용, hidden flag 해제.

```c
static void stale_check_cb(lv_timer_t *t) {  // 2 s 주기
    for (each active card c) {
        uint32_t age = now - c->last_update_ms;
        if (age > 60000) {
            lv_obj_add_flag(c->card, LV_OBJ_FLAG_HIDDEN);
            c->mode = CARD_UNUSED;                    // slot 반환
        } else if (age > 15000) {
            // 색만 회색 계열로
        }
    }
    if (any_hidden) card_reorder(pool);               // 활성 카드만 재배치
}
```

**교훈**: 실시간 UI 는 데이터 도착 시 갱신뿐 아니라, **데이터가 안 오는 것도 이벤트로 처리**해야
사용자 관점의 "깨끗한 화면" 이 유지된다.

### 14.7 폴더 이름이 코드의 진화를 못 따라올 때

Backend 는 처음 `hello-world-dev` 라는 이름으로 시작했다가 seriallink 로 evolve 했다.
Local 은 monorepo 로 정리됐지만 VPS 폴더명은 그대로 남아있었다:

```
VPS:  <vps-repo-path>/
      <vps-repo-path>/backend/    ← 실은 seriallink Rust 코드
      <vps-repo-path>/frontend/   ← 실은 seriallink SvelteKit
Docker container: hello-world-backend-dev
Nginx: <service-host> → 127.0.0.1:8000 (hello-world-backend)
```

Nginx conf, docker container 이름, mount path 전부 옛 이름. 한 사이클 잡고 정리:

1. `docker stop / rm` (약 30 초 downtime)
2. 폴더 rename
3. `sudo sed` 로 nginx conf 안의 경로 갱신
4. Docker image `seriallink-backend:dev` 로 tag 새로 build
5. 새 이름으로 container run
6. `nginx -s reload` + smoke test

이런 정리는 늦추면 자꾸 사고를 부른다. 원격에서 "hello-world 배포 스크립트" 를 찾아 갔더니
seriallink 코드가 나와서 몇 분 헤맸다. **이름은 코드의 사실을 반영해야 한다.**

---

### 14.8 KS X 3286 자동등록 노드 통합 (마스터 확장, 구현 완료)

우리 slave 는 default map 노드 (기관/회사 = 0) 만 만들었다.  하지만 실 온실에는 타사 vendor 의
non-default map 노드 (기관/회사 non-zero + custom register 배치) 가 붙는다.  이걸 통합하는 것이
**KS X 3286 §7~§8** 이 정의한 것이고, 마스터에 이 flow 를 얹었다.

**3286 이 요구하는 flow** (제어기가 수행):

```
① NodeInfo 조회 → 회사/제품 코드 확인
② 회사/제품 코드 → 원격 저장소 or 파일에서 노드 규격 JSON 조회
③ 디바이스 코드 조회 (reg 101~200)
④ 회사/디바이스 코드 → 디바이스 규격 JSON 조회
⑤ CommSpec.read.starting-register + items → 실 register 주소 계산

예:
{ "CommSpec": {
    "KS X 3267": {
      "read":  {"starting-register": 201, "items": ["status"]},
      "write": {"starting-register": 301, "items": ["operation","opid"]}
    }
  }
}
→ reg 201=status, reg 301=operation, reg 302=opid
```

**구현**:

- 마스터: `ks_x_3286.{h,c}` — item keyword size table + spec 배열 + CommSpec 파싱 함수
- `node_registry` — 회사코드 non-zero 감지 시 `ks3286_find_node_spec(company, product)` → `ks3286_calc_map()` → `custom_map` 저장
- `poller` — `custom_map.valid` 이면 `poll_custom_map_node` 로 분기 (default map path 우회)

**시연 시나리오**: 우리 slave 하나를 vendor 노드처럼 flash.

```
1. slave 를 vendor mode 로 flash (sdkconfig.defaults.vendor):
   CONFIG_KSNODE_COMPANY_CODE=0xABCD
   CONFIG_KSNODE_PRODUCT_CODE=0x0001
   CONFIG_KSNODE_VENDOR_DEMO=y            ← reg 401 (status, opid) dummy area 등록

2. master reset. registry_scan 시 NodeInfo 응답 상 회사=0xABCD → spec table 조회 →
   "VendorX Actuator Node" 매칭 → CommSpec (read: reg 401=[status, opid]) →
   register map 계산 → custom_map 저장.

3. poller 는 이제 default map (reg 202) 대신 3286 spec 이 지정한 reg 401..402 로 poll.
```

이걸 얹으니 **"타사 표준 준수 노드가 우리 마스터에 붙어도 정상 작동"** 이 검증됐다.
편의 (§14.1 discovery) 가 우리 세트 최적화였다면, 이건 **호환성 검증의 완결편** 이다.

**아직 안 한 것**:
- 규격 JSON 원격 로드 (SPIFFS/HTTP fetch) — 현재는 컴파일 타임 embedded spec 배열
- 디바이스 규격 (§7.4) — 노드 규격만 지원
- Sensor `value` 매핑 — 지금은 status/opid/control 만

실 vendor 노드가 실제로 등장하는 시점에 확장 예정.  현 시점 코드는 **3286 §7~§8 의 골격
(spec 조회 → map 계산 → poll) 을 갖추었고**, 시연 시나리오로 flow 가 검증된다.

### 14.9 운영 편의성 — 관리 UI · async job · stale 정리

검증이 끝난 뒤 운영에 붙이면서 나온 UI/백엔드 개선들. 표준 자체와 무관하지만 실제로 슬레이브
20~30 대가 붙는 순간부터 필요해진다.

**devices 테이블에 modbus 주소 컬럼.** 지금까지 UI 는 `bus_id`(BUS#1/#2) 만 보여줬는데,
같은 버스 안 5~10개 슬레이브를 구분하려면 modbus address 가 필요했다. 스키마에 `addr smallint`
컬럼 추가, backend `ingest_node_snapshot` 이 snapshot payload 의 `slave` 값을 upsert.
Frontend `DevicesCard` 는 bus 옆에 addr 컬럼을 추가하고, alias 셀은 **inline 편집** (hover 시 연필
아이콘, 더블클릭 → input → Enter/체크 저장, Esc/X 취소). Snapshot 도착시 default alias 를
`"센서 BUS#1 슬레이브1"` 로 자동 생성하지만, 사용자가 편집하면 그 값이 유지된다 (COALESCE).

**Async job 패턴 — 대량 DELETE 는 즉시 응답 + polling.** Device 완전 삭제 (`sensor_data_v2` +
`devices` + `devices_dim` 통째) 를 sync 로 시도했더니 21 만 row 짜리 device 에서 **178초** 소요.
TimescaleDB compressed hypertable 상 partial DELETE 는 chunk 별 decompress + scan 이라 어쩔
수 없는 시간이다. Nginx 는 60s 에 504, 브라우저는 abort → backend 는 rollback. Timeout 을 300s
로 늘려도 304s 걸리는 경우가 있어 무한 우회 게임.

해결은 timeout 확장이 아니라 **패턴 변경**:

```
POST /api/devices/by-device-id/:id
  → 즉시 202 Accepted + { job_id, poll_url }
  → 실제 DELETE 는 tokio::spawn 안에서 실행
  → AppState.purge_jobs: HashMap<device_id, PurgeJob> (in-memory)

GET  /api/devices/by-device-id/:id/purge-status
  → { state: running|done|error, elapsed_ms, sensor_rows, ... }
```

Frontend `deleteDevice(id, onProgress?)` 는 202 받고 3s 간격 polling, done/error 까지 대기.
UI 는 "⏳ 진행 중... 12s 경과 (job purge-...)" 로 매초 업데이트. 페이지를 이동해도 backend task
는 살아있어서 다음 조회 시 상태 확인 가능. 21 만 row 삭제도 사용자 관점에서 부드럽게 완료된다.

**Sensor prune UI — dummy → 특정 센서 교체 후 stale metric 정리.** 슬레이브가 dummy sensor 로
부팅하면 backend 가 KS X device code 1~19 를 값 0.0 으로 다 upsert. 이후 실 sensor (예: SOIL_WATER)
로 flash 하면 SOIL_MOISTURE 만 update 되지만, 기존 dummy metric row 들은 `sensor_data_v2` 에
남아서 UI 카드로 계속 보인다 ("6.3시간 전 CO2=0.00" 이 며칠씩 남음). 이력이 남는 것 자체는
스마트팜 시스템에서 원칙적으로 valuable 하지만 UX 관점에선 clutter.

3-모드 prune API + 재사용 `SensorPrunePanel` 컴포넌트:

- **Stale**: 최근 N시간 무데이터 metric 만 삭제 (활성 metric 은 유지)
- **기간**: N일 이전 row 전부 삭제 (모든 metric)
- **전체**: 이 device 의 모든 sensor 이력 wipe

같은 SQL 함수 안에서 처리하되, `sensor_data_v2.device_id` 는 smallint FK 이라
문자열 `S-XXXXXXXX` 를 `(SELECT id FROM devices_dim WHERE device_id=$1)` 로 lookup 후 bind.
(초기 구현에서 이 lookup 을 빼먹어 500 이 났다 — sqlx 타입 강제라 조용히 실패하지 않고 에러 발생.)

이 3개는 표준과 직접 무관하지만, **10-20대 슬레이브 운영 시점에 없으면 UI 가 실사용 불가 상태**가
된다. 검증 코드와 운영 코드는 자란 이유가 다를 뿐, 결국 같은 서비스 안에 살아야 한다.

---

*§14 는 검증 이후에 만든 것들이지만, 아이러니하게도 **이것들을 만드는 과정이 표준을 다시 이해하게 했다.**
"이 편의를 어떻게 얹어야 표준을 안 부술까" 라는 질문은 결국 **§5.1 이 어디까지 자유를 주는지**,
**§4.1 이 어떤 범위를 예약했는지**, **3286 과 3267 이 서로 어디를 담당하는지** 를 원문에서 다시 확인하는 일로 이어졌다.*

