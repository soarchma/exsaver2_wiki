# EXSAVER 2.0 Edge Gateway (CMG) 연동 규격서 - [Part 2] 페이로드 명세서

**문서 버전**: v1.0.0-draft.11

[⬅️ 통합 안내서 및 변경 이력으로 돌아가기](index.md)

## 1. 데이터 전송 및 제어 페이로드 규격

본 시스템은 네트워크 대역폭 효율화 및 시스템 안정성 확보를 위해 **하이브리드 페이로드 구조**를 채택합니다.

1. 수집 빈도가 높은 **데이터(Telemetry)**는 데이터 크기 최소화를 위해 **약어(Short Key)**를 사용합니다.
2. 제어 명령(Command) 및 이벤트(Event)는 가독성 및 식별 오류 방지를 위해 **전체 단어(Full Word)**를 사용합니다.

<br>

### 1.1. LCU 전력 데이터 (Telemetry: CMG ➔ Server)

- **Topic**: `exsaver/cmg/{cmg_id}/lcu/{lcu_id}/data`
- **QoS**: `0`
- **구조**: LCU 공통 환경 데이터(전압, 주파수, 온도, 습도)와 하위 채널별 데이터 배열(`ch`)을 분리하여 중복 트래픽을 최소화한 구조.
- **참고(Vrms, Freq)**: 하드웨어 센서는 채널별로 전압과 주파수를 개별 계측하지만, 페이로드 최적화를 위해 **Meter #1의 Vrms와 Freq를 해당 LCU의 대표값**으로 사용하여 공통 데이터 객체에 담아 전송합니다.

**[데이터 페이로드 예시 (6채널 LCU 기준)]**
_(주의: 물리적 채널 수에 맞추어 `ch` 배열 내에 모든 채널의 객체가 누락 없이 포함되어야 합니다.)_

```json
{
  "ts": 1773306000,
  "v": 220.1,
  "hz": 59.9,
  "pt": 42.5,
  "h": 35.2,
  "ch": [
    {
      "n": 1,
      "s": 1,
      "w": 150.5,
      "a": 0.68,
      "pf": 0.95,
      "sw": 0.0,
      "cw": 10500.5,
      "cs": 120.0
    },
    {
      "n": 2,
      "s": 0,
      "w": 0.0,
      "a": 0.0,
      "pf": 0.0,
      "sw": 15.5,
      "cw": 5000.0,
      "cs": 450.5
    },
    {
      "n": 3,
      "s": 1,
      "w": 45.0,
      "a": 0.21,
      "pf": 0.97,
      "sw": 0.0,
      "cw": 2100.0,
      "cs": 10.0
    },
    {
      "n": 4,
      "s": 1,
      "w": 12.3,
      "a": 0.06,
      "pf": 0.88,
      "sw": 0.0,
      "cw": 800.5,
      "cs": 55.2
    },
    {
      "n": 5,
      "s": 0,
      "w": 0.0,
      "a": 0.0,
      "pf": 0.0,
      "sw": 3.2,
      "cw": 150.0,
      "cs": 5.0
    },
    {
      "n": 6,
      "s": 0,
      "w": 0.0,
      "a": 0.0,
      "pf": 0.0,
      "sw": 0.0,
      "cw": 0.0,
      "cs": 0.0
    }
  ]
}
```

**수집 데이터 약어 사전 (Short Key Dictionary)**

**[LCU 공통 데이터]**
| 약어 (Key) | 원본 명칭 (Full Word) | 데이터 타입 | 설명 및 단위 |
| :--- | :--- | :--- | :--- |
| `ts` | timestamp | Integer (`time_t`) | 측정 시간 (Unix Epoch Time, 초 단위 정수형) |
| `v` | voltage_v | Float | LCU 입력단 전압 (V) |
| `hz` | frequency_hz | Float | LCU 입력단 교류 주파수 (Hz) |
| `pt` | pcb_temp_c | Float | LCU 기판 내부 온도 (화재 예방 및 수명 모니터링용, ℃) |
| `h` | pcb_humidity | Float | LCU 기판 내부 습도 (%) |
| `ch` | channels | Array | 하위 채널별 개별 전력 데이터 배열 (최대 6개) |

**[채널별 개별 데이터 (`ch` 배열 내부)]**
| 약어 (Key) | 원본 명칭 (Full Word) | 데이터 타입 | 설명 및 단위 |
| :--- | :--- | :--- | :--- |
| `n` | ch_no | Integer | LCU 내 물리적 채널 번호 (4채널: 1-4, 6채널: 1-6) |
| `s` | status | Integer | 릴레이 현재 물리적 상태 (`1` = ON, `0` = OFF) |
| `w` | active_power | Float | 해당 채널의 유효 전력 (W) |
| `a` | current_a | Float | 해당 채널의 전류 (A) |
| `pf` | power_factor | Float | 해당 채널 부하의 역률 (0.0 ~ 1.0) |
| `sw` | saved_w | Float | 현재 차단(OFF) 상태로 인해 실시간 절감 중인 순시 대기전력 (W) |
| `cw` | cumulative_power_wh | Double | 누적 사용 전력량 (Wh, 소수점 1자리) |
| `cs` | cumulative_saved_wh | Double | 누적 절감 전력량 (Wh, 소수점 1자리, 가상 데이터) |

> ⚠️ **[중요: 펌웨어 개발 시 센서 원시값(Raw) 스케일 변환 가이드]**
> 서버의 하드웨어 종속성을 제거하기 위해, CMG는 JSON 직렬화 전 센서 레지스터의 원시값을 아래 규칙에 따라 **실제 물리량 스케일로 변환(`Double/Float` 캐스팅)**하여 전송해야 합니다.
>
> - **`* 0.1` 곱하기 적용 (반드시 64-bit `double` 사용)**:
>   - 누적 전력량(`cw`, `cs`)은 원시값이 `UINT32_T`이므로, 누적값이 커질 경우 32-bit `float` 연산 시 정밀도 손실(Precision Loss)로 데이터가 증가하지 않는 버그가 발생합니다. 반드시 `double` 형으로 캐스팅하여 연산 후 전송하십시오.
>   - 온도(`pt`), 습도(`h`) 역시 0.1을 곱하여 전송합니다.
> - **`* 0.01` 곱하기 적용**:
>   - 전압(`v`), 주파수(`hz`) _(예: Vrms 원시값 22010 ➔ `220.1` V)_
> - **`* 0.001` 곱하기 적용**:
>   - 전류(`a`), 역률(`pf`) _(예: Irms 원시값 680 ➔ `0.68` A)_
> - **변환 없이 1:1 매핑 (원시값 = 실제값)**:
>   - **유효 전력(`w`)**: 센서 스펙상 0.001 kW 단위이므로, 원시값 `1`이 정확히 `1 W`와 동일합니다. 추가 스케일 연산 없이 해당 정수를 그대로 Float로 담아 보내시면 됩니다.

<br>

### 1.2. 전체 제어 명령어 및 응답 리스트 (Command & Event)

- **명령(Command) 토픽 (Server ➔ CMG)**: `exsaver/cmg/{cmg_id}/cmd`
- **응답(Event/Ack) 토픽 (CMG ➔ Server)**: `exsaver/cmg/{cmg_id}/cmd_ack`

<br>

**[전체 제어 명령어 (Command) 요약표]**
| 카테고리 | `cmd_type` | 목적 및 설명 | Target |
| :---------------------------- | :-------------------- | :------------------------------------------------------------ | :------- |
| **1. 제어 및 모니터링** | `RELAY_CONTROL` | 특정 LCU의 단일/복수 릴레이 채널 전원 ON/OFF 핀포인트 제어 | LCU |
| | `GROUP_CONTROL` | 특정 논리 그룹(Local Group) 단위 일괄 ON/OFF 제어 (신규) | CMG |
| | `POLL_DATA` | 1분 주기 보고를 대기하지 않고 즉시 최신 전력 데이터 수집 요청 | LCU |
| | `POLL_ALL_DATA` | CMG 산하의 모든 LCU 최신 전력 데이터 즉시 일괄 수집 요청 | CMG |
| **2. 네트워크 및 프로비저닝** | `PROVISION` | 미할당 기기(LCU/RTU)에 NMK를 주입하여 사설 망(AVLN) 결속 | LCU, RTU |
| | `UNPROVISION` | 기기 망 이탈 및 공장초기화(NMK 초기화) 상태 전환 | LCU, RTU |
| | `RESTORE_NETWORK` | 교체된 신규 CMG에 기존 통신망 암호(NMK) 주입 | CMG |
| | `GET_NETWORK_INFO` | CMG 산하 사설망(NMK)에 정상 연결된 하위 기기 리스트 조회 | CMG |
| | `SCAN_UNPROVISIONED` | 현재 공용망(Public Network)에 대기 중인 미등록 기기 목록 스캔 | CMG |
| **3. 환경 설정 및 정책** | `SYNC_TIME` | 장비 내장 RTC 시간 동기화 (오프라인 스케줄 작동용) | ALL |
| | `SET_CHANNEL_CFG` | 채널별 하드웨어 잠금 기능 및 대기전력 차단 설정 | LCU |
| | `GET_CHANNEL_CFG` | 현재 기기에 적용된 대기전력 및 채널 설정값 조회 | LCU |
| | `MANAGE_GROUP` | 논리 그룹(Local Group) 1개 단위 생성/수정/삭제 (증분 동기화) | CMG |
| | `MANAGE_SCHEDULE` | 하이브리드 스케줄 1건 단위 생성/수정/삭제 (증분 동기화) | CMG |
| | `REQUEST_SYNC_DIGEST` | 서버 주도 CMG 그룹/스케줄 해시(Hash) 무결성 상태 점검 요청 | CMG |
| **4. RTU 전용 제어** | `SET_RTU_TARGET_LCU` | RTU 관할 하위 LCU MAC 리스트 할당 | RTU |
| | `GET_RTU_TARGET_LCU` | 현재 RTU에 할당된 관할 LCU 리스트 조회 | RTU |
| | `SET_RTU_MAP_LAYOUT` | RTU 배경 도면 이미지 URL 및 구역(터치 영역) 트리거 매핑 설정 | RTU |
| | `GET_RTU_MAP_LAYOUT` | RTU 현재 도면 화면 및 구역 트리거 설정 상태 조회 | RTU |
| **5. 시스템 유지보수** | `GET_DEVICE_INFO` | 펌웨어 버전, 하드웨어 리비전, Uptime, IP 주소 조회 | ALL |
| | `SET_BANK_SWAP` | LCU 뱅크 결선 논리 스왑 설정 (휴먼 에러 대응) | LCU |
| | `REBOOT` | 장비 원격 강제 재부팅 | ALL |
| | `FIRMWARE_UPDATE` | 펌웨어 업데이트 수행 명령 (URL 포함) | ALL |

**[응답 정책 및 cmd_id 처리 규칙]**

1. **트랜잭션 식별자 (Transaction ID)**: MQTT는 비동기 프로토콜이므로, 서버가 발행한 요청(Command)과 CMG가 회신하는 결과(Event)의 짝을 맞추기 위해 `cmd_id`를 사용합니다. 이는 특정 `cmd_type`과 세트로 묶이는 값이 아니며, 단순 순차 번호(Sequence Number)도 아닌 서버가 매번 새롭게 발행하는 고유 식별자입니다. **(최대 64 Byte 문자열)**
2. **Echo-back 원칙**: CMG는 `cmd_id`의 내부 포맷을 해석하거나 검증할 필요가 없습니다. 수신한 `cmd_id` 값을 메모리에 임시 보관했다가, 응답(ACK) 메시지를 발행할 때 원본 그대로(Bypass) 포함하여 회신하기만 하면 됩니다.
3. **비동기 순서 및 중복 수신 방어 (Idempotency)**: 여러 명령이 동시에 수신될 경우 처리 완료 순서대로 독립적으로 응답하면 되며, 순서를 강제할 필요가 없습니다. 단, 네트워크 불안정으로 인해 **이미 처리된 동일한 `cmd_id`가 중복 수신될 경우**, CMG는 하드웨어 중복 제어를 생략하고 기존 처리 결과(성공 ACK)만 재발송해야 합니다.

**[상태 코드 (status) 정의]**
| `status` 값 | 의미 및 조건 |
| :--- | :--- |
| `COMPLETED` | 명령 정상 수행. **(부분 성공 포함)** 제어 결과 또는 조회(GET) 시 `data` 객체 포함. |
| `FAILED` | 통신 불가, 파라미터 오류 등 수행 실패. (`error_code` 및 `message` 필수) |
| `IN_PROGRESS`| `FIRMWARE_UPDATE`, `SCAN_UNPROVISIONED` 등 완료까지 장시간이 소요되는 작업의 진행 상태.<br><br>**[장기 작업 처리 가이드]** CMG는 명령 수신 직후 즉시 동일한 `cmd_id`와 함께 `IN_PROGRESS`를 1차 응답하여 서버의 타임아웃을 방지해야 합니다. 이후 실제 작업이 최종 완료되면 다시 동일한 `cmd_id`로 `COMPLETED` (또는 `FAILED`) 응답을 2차 전송해야 합니다. |

**[공통 실패 응답 (FAILED) 페이로드 규격]**
명령 수행 완전 실패 시 아래 포맷으로 회신하며, 하단 개별 명령어 명세에서는 편의상 실패 예시를 생략합니다.

```json
{
  "cmd_id": "req-xxxx",
  "target_id": "lcu_001122334402",
  "status": "FAILED",
  "message": "Error description here",
  "error_code": "ERR_SPECIFIC_CODE"
}
```

---

#### 카테고리 1: 제어 및 모니터링 (Control & Monitor)

---

**[참고] 제어 action (상태 제어 명령) 3원화 규격**
`RELAY_CONTROL`, `GROUP_CONTROL`, `MANAGE_SCHEDULE` 등 릴레이를 제어하는 모든 명령어의 `action` 필드는 다음 3가지 중 하나를 가집니다.

- `ON`: 릴레이 즉시 켬 + 대기전력 차단 모니터링 영구 비활성화 (출근/근무 모드)
- `OFF`: 릴레이 즉시 끔 + 대기전력 차단 모니터링 비활성화 (퇴근 완전 소등)
- `STANDBY`: 릴레이 현재 켜진 상태 유지 + **대기전력 차단 모니터링 활성화** (퇴근 감시 모드. 설정된 전력/시간 조건 도달 시 자동 OFF 수행)

---

##### 1) RELAY_CONTROL (단일 기기 릴레이 핀포인트 제어)

특정 LCU의 단일/복수 릴레이 채널 전원 및 모니터링 상태를 제어합니다. (채널 배열 최대 6개)

- **부분 성공 및 안전장치 규칙**: 채널의 모드가 `FORCE_ON` 또는 `FORCE_OFF`로 잠겨 있어 제어를 거부한 경우, 해당 채널의 `status`는 실제 유지된 물리 상태(ON/OFF)를 반환하고 제어 실패 사유(`reason`)를 명시하여 부분 성공(`COMPLETED`)으로 응답합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-1001",
  "cmd_type": "RELAY_CONTROL",
  "target_id": "lcu_001122334402",
  "channels": [
    { "ch_no": 1, "action": "OFF" },
    { "ch_no": 2, "action": "STANDBY" } // 💡 모니터링 활성화 명령
  ]
}

// [응답] CMG ➔ Server (Topic: .../cmd_ack)
{
  "cmd_id": "req-1001",
  "target_id": "lcu_001122334402",
  "status": "COMPLETED",
  "data": {
    "channels": [
      // 1번 채널이 FORCE_ON 모드여서 OFF 명령을 펌웨어가 거부(Bypass)한 예시
      { "ch_no": 1, "status": "ON", "reason": "LOCKED_FORCE_ON" },
      { "ch_no": 2, "status": "ON" }
    ]
  },
  "message": "Success"
}
```

##### 2) GROUP_CONTROL (논리 그룹 일괄 제어)

사전에 동기화된 `group_id`를 전송하여 다중 기기를 즉시 제어합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-grp-ctrl-001",
  "cmd_type": "GROUP_CONTROL",
  "target_id": "cmg_001122334401",
  "group_id": "grp-local-light-01",
  "action": "STANDBY" // 💡 해당 그룹 기기 전체 대기전력 모니터링 가동
}

// [응답] CMG ➔ Server (Topic: .../cmd_ack)
{
  "cmd_id": "req-grp-ctrl-001",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "Group control executed",
  "data": {
    "group_id": "grp-local-light-01",
    "results": [
      { "lcu_id": "lcu_001122334402", "ch_no": 1, "status": "ON" },
      { "lcu_id": "lcu_001122334402", "ch_no": 2, "status": "OFF", "reason": "LOCKED_FORCE_OFF" }
    ]
  }
}
```

##### 3) POLL_DATA (특정 기기 즉시 데이터 수집)

1분 주기 보고를 대기하지 않고 즉시 최신 전력 데이터를 수집합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-1002",
  "cmd_type": "POLL_DATA",
  "target_id": "lcu_001122334402"
}

// [응답] CMG ➔ Server
// (주의: 계측 데이터 자체는 이 응답에 포함하지 않으며, 이 응답 직후 `data` 토픽으로 해당 기기의 계측 데이터를 강제 발행합니다.)
{
  "cmd_id": "req-1002",
  "target_id": "lcu_001122334402",
  "status": "COMPLETED",
  "message": "Telemetry data will be published immediately via data topic"
}
```

##### 4) POLL_ALL_DATA (전체 즉시 데이터 수집)

CMG 관할 하에 있는 모든 LCU의 최신 전력 데이터를 즉시 수집합니다. 서버 초기화 및 대시보드 일괄 동기화에 사용됩니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-1003",
  "cmd_type": "POLL_ALL_DATA",
  "target_id": "cmg_001122334401"
}

// [응답] CMG ➔ Server
// (주의: CMG는 응답 전송 후 브로커 과부하를 방지하기 위해 10~50ms 지연을 두고 순차적으로 `data` 토픽을 발행해야 합니다.)
{
  "cmd_id": "req-1003",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "All LCU telemetry data will be published sequentially"
}
```

---

#### 카테고리 2: 네트워크 및 장비 프로비저닝 (Network & Provision)

##### 1) PROVISION (기기 프로비저닝)

미할당 기기(LCU/RTU)에 NMK를 주입하여 사설 망(AVLN)에 결속시킵니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "prov-001",
  "cmd_type": "PROVISION",
  "target_id": "lcu_001122334455",
  "dak_key": "1234567890abcdef",
  "nmk_key": "Arsenal2026"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "prov-001",
  "target_id": "lcu_001122334455",
  "status": "COMPLETED",
  "message": "Provisioning successful"
}
```

##### 2) UNPROVISION (기기 초기화)

기기를 망에서 이탈시키고 공장초기화(NMK 초기화) 상태로 전환합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-2010",
  "cmd_type": "UNPROVISION",
  "target_id": "lcu_001122334455"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-2010",
  "target_id": "lcu_001122334455",
  "status": "COMPLETED",
  "message": "Device unprovisioned"
}
```

##### 3) RESTORE_NETWORK (망 복구)

교체된 신규 CMG에 기존 통신망 암호(NMK)를 주입하여 하위 기기 제어를 복구합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "restore-001",
  "cmd_type": "RESTORE_NETWORK",
  "target_id": "cmg_001122334401",
  "nmk_key": "Arsenal2026",
  "child_ids": [
    "lcu_001122334402",
    "lcu_001122334403"
  ]
}

// [응답] CMG ➔ Server
{
  "cmd_id": "restore-001",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "Network restored"
}
```

##### 4) GET_NETWORK_INFO (네트워크 정보 조회)

CMG 산하 사설망(NMK)에 정상 연결된 하위 기기 리스트를 조회합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-2001",
  "cmd_type": "GET_NETWORK_INFO",
  "target_id": "cmg_001122334401"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-2001",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "data": {
    "connected_devices": [
      "lcu_001122334402",
      "rtu_001122334405"
    ]
  },
  "message": "Success"
}
```

##### 5) SCAN_UNPROVISIONED (미등록 기기 스캔)

현재 공용망(Public Network)에 대기 중인 미등록 기기 목록을 스캔합니다. (장기 소요 작업)

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-2002",
  "cmd_type": "SCAN_UNPROVISIONED",
  "target_id": "cmg_001122334401"
}

// [최종 응답] CMG ➔ Server
{
  "cmd_id": "req-2002",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "data": {
    "unprovisioned_devices": [
      { "mac": "00:11:22:33:44:aa", "device_type": "LCU", "rssi": -45 },
      { "mac": "00:11:22:33:44:bb", "device_type": "RTU", "rssi": -60 }
    ]
  }
}
```

---

#### 카테고리 3: 환경 설정 및 통합 제어 정책 (Config & Universal Control)

##### 1) SYNC_TIME (시간 동기화)

장비 내장 RTC 시간을 서버 시간으로 동기화합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-3002",
  "cmd_type": "SYNC_TIME",
  "target_id": "cmg_001122334401",
  "timestamp": 1773336600
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-3002",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "Time synchronized"
}
```

##### 2) SET_CHANNEL_CFG (채널별 동작 모드 및 대기전력 설정)

기존 `SET_STANDBY_CFG`를 대체하는 마스터 설정 명령입니다. LCU 각 채널의 하드웨어 안전장치(동작 모드)와 대기전력 차단 기준값을 동시에 설정합니다.

- **동작 모드(`mode`) 정의**:
  - `NORMAL`: 일반 모드. 모든 원격 제어 및 STANDBY 감시 수행 정상 동작.
  - `FORCE_ON`: 안전 모드(비상등, 서버 등). 모든 `OFF` / `STANDBY` 명령 무시 및 항시 통전.
  - `FORCE_OFF`: 차단 모드(미결선, 수리중). 모든 `ON` / `STANDBY` 명령 무시 및 항시 단전.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-3001",
  "cmd_type": "SET_CHANNEL_CFG",
  "target_id": "lcu_001122334402",
  "channels": [
    {
      "ch_no": 1,
      "mode": "FORCE_ON",   // 💡 절대 꺼지지 않음
      "threshold_w": 0.0,
      "delay_sec": 0
    },
    {
      "ch_no": 2,
      "mode": "NORMAL",     // 💡 일반 PC 자리 (대기전력 5W 5분 지속 시 차단)
      "threshold_w": 5.0,
      "delay_sec": 300
    }
  ]
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-3001",
  "target_id": "lcu_001122334402",
  "status": "COMPLETED",
  "message": "Channel configuration applied"
}
```

##### 3) GET_CHANNEL_CFG (채널별 설정 조회)

현재 기기에 적용된 채널별 동작 모드 및 대기전력 설정값을 조회합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-3011",
  "cmd_type": "GET_CHANNEL_CFG",
  "target_id": "lcu_001122334402"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-3011",
  "target_id": "lcu_001122334402",
  "status": "COMPLETED",
  "data": {
    "channels": [
      { "ch_no": 1, "mode": "FORCE_ON", "threshold_w": 0.0, "delay_sec": 0 },
      { "ch_no": 2, "mode": "NORMAL", "threshold_w": 5.0, "delay_sec": 300 }
    ]
  },
  "message": "Success"
}
```

##### 4) MANAGE_GROUP (통합 논리 그룹 증분 동기화)

특정 논리 그룹 1개에 대한 생성/수정/삭제 작업을 수행합니다. (트래픽 최적화를 위해 전체 배열 전송을 피함)

```json
// [요청] Server ➔ CMG : 생성 또는 타겟 수정 (UPSERT)
{
  "cmd_id": "req-grp-add-001",
  "cmd_type": "MANAGE_GROUP",
  "target_id": "cmg_001122334401",
  "action": "UPSERT",
  "group_data": {
    "group_id": "grp-local-light-01",
    "updated_ts": 1773335000,
    "hash": "a1b2c3d4e5f6...", // 무결성 대조용 서버 해시값
    "targets": [
      { "lcu_id": "lcu_001122334402", "ch_no": 1 },
      { "lcu_id": "lcu_001122334402", "ch_no": 2 }
    ]
  }
}

// [요청] Server ➔ CMG : 완전 삭제 (DELETE)
{
  "cmd_id": "req-grp-del-001",
  "cmd_type": "MANAGE_GROUP",
  "target_id": "cmg_001122334401",
  "action": "DELETE",
  "group_id": "grp-local-light-01"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-grp-add-001",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "Group successfully managed"
}
```

##### 5) MANAGE_SCHEDULE (하이브리드 스케줄 증분 동기화)

오프라인 스케줄 1건 단위의 생성/수정/삭제 작업을 수행합니다. `actions` 배열 내에서 사전 동기화된 그룹(`GROUP`)을 호출하거나 단일 기기(`DEVICE`)를 직접 타겟팅할 수 있으며, `action` 값에 `STANDBY`를 적용하여 주/야간 모니터링 자동화가 가능합니다.

```json
// [요청] Server ➔ CMG : 스케줄 생성 또는 수정 (UPSERT)
{
  "cmd_id": "req-sch-add-001",
  "cmd_type": "MANAGE_SCHEDULE",
  "target_id": "cmg_001122334401",
  "action": "UPSERT",
  "schedule_data": {
    "schedule_id": "sch-001",
    "updated_ts": 1773335500,
    "hash": "b2c3d4e5f6g7...",
    "time": "18:00",
    "days": ["MON", "TUE", "WED", "THU", "FRI"],
    "actions": [
      // [유형 1] 그룹 제어: 18시 정각에 릴레이를 끄는 게 아니라, 대기전력 감시(STANDBY) 모드로 전환
      { "target_type": "GROUP", "target_id": "grp-local-light-01", "action": "STANDBY" },
      // [유형 2] 단일 기기 직접 제어: 특정 기기는 예외적으로 즉시 완전 소등(OFF)
      { "target_type": "DEVICE", "lcu_id": "lcu_001122334405", "ch_no": 4, "action": "OFF" }
    ]
  }
}

// [요청] Server ➔ CMG : 스케줄 삭제 (DELETE)
{
  "cmd_id": "req-sch-del-001",
  "cmd_type": "MANAGE_SCHEDULE",
  "target_id": "cmg_001122334401",
  "action": "DELETE",
  "schedule_id": "sch-001" // 삭제 시에는 actions 배열 등 기타 정보 불필요
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-sch-add-001",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "Schedule successfully managed"
}
```

##### 6) REQUEST_SYNC_DIGEST (상태 무결성 점검 요청)

서버 대시보드나 주기적 스케줄러가 CMG의 데이터 무결성 상태를 점검하기 위해 해시 요약본(Digest) 전송을 요청합니다. CMG는 수신 즉시 `SYNC_DIGEST` 이벤트 메시지로 응답해야 합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-check-001",
  "cmd_type": "REQUEST_SYNC_DIGEST",
  "target_id": "cmg_001122334401"
}

// [응답] CMG ➔ Server (Topic: .../cmd_ack)
// * 주의: 실제 해시 데이터는 4.3항의 `SYNC_DIGEST` Event 토픽으로 발행되며, 이 응답은 명령 수신 성공 여부만 반환합니다.
{
  "cmd_id": "req-check-001",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "Digest event will be published immediately"
}
```

---

#### 카테고리 4: RTU 전용 제어 (RTU Specific)

기존의 개별 채널 매핑(`SET_RTU_GROUPS`) 명령어는 폐기되었으며, 모든 RTU 터치 액션은 카테고리 3의 `MANAGE_GROUP`으로 동기화된 통합 그룹(`group_id`)을 호출(Trigger)하는 방식으로 단순화되었습니다.

##### 1) SET_RTU_TARGET_LCU (하위 LCU 할당)

RTU 관할 하위 LCU MAC 리스트를 할당합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-4001",
  "cmd_type": "SET_RTU_TARGET_LCU",
  "target_id": "rtu_001122334405",
  "lcu_ids": [
    "lcu_001122334402",
    "lcu_001122334403"
  ]
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-4001",
  "target_id": "rtu_001122334405",
  "status": "COMPLETED",
  "message": "Target LCU list applied"
}
```

##### 2) GET_RTU_TARGET_LCU (하위 LCU 조회)

현재 RTU에 할당된 관할 LCU 리스트를 조회합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-4011",
  "cmd_type": "GET_RTU_TARGET_LCU",
  "target_id": "rtu_001122334405"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-4011",
  "target_id": "rtu_001122334405",
  "status": "COMPLETED",
  "data": {
    "lcu_ids": [
      "lcu_001122334402",
      "lcu_001122334403"
    ]
  },
  "message": "Success"
}
```

##### 3) SET_RTU_MAP_LAYOUT (도면 및 터치 구역 트리거 설정)

RTU 배경 도면 이미지 URL과 구역(터치 영역)별 대상 논리 그룹 트리거를 매핑합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-4003",
  "cmd_type": "SET_RTU_MAP_LAYOUT",
  "target_id": "rtu_001122334405",
  "bg_image_url": "[http://exsaver.net/assets/bg/floor2.png](http://exsaver.net/assets/bg/floor2.png)",
  "touch_zones": [
    {
      "zone_id": "zone-01",
      "target_group_id": "grp-local-light-01", // CMG 내부에 저장된 로컬 통합 그룹 ID
      "rect": { "x": 100, "y": 250, "w": 300, "h": 200 }
    }
  ]
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-4003",
  "target_id": "rtu_001122334405",
  "status": "COMPLETED",
  "message": "Map layout applied"
}
```

##### 4) GET_RTU_MAP_LAYOUT (도면 설정 조회)

RTU 현재 도면 화면 및 구역 트리거 설정 상태를 조회합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-4013",
  "cmd_type": "GET_RTU_MAP_LAYOUT",
  "target_id": "rtu_001122334405"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-4013",
  "target_id": "rtu_001122334405",
  "status": "COMPLETED",
  "data": {
    "bg_image_url": "[http://exsaver.net/assets/bg/floor2.png](http://exsaver.net/assets/bg/floor2.png)",
    "touch_zones": [
      {
        "zone_id": "zone-01",
        "target_group_id": "grp-local-light-01",
        "rect": { "x": 100, "y": 250, "w": 300, "h": 200 }
      }
    ]
  },
  "message": "Success"
}
```

---

#### 카테고리 5: 시스템 유지보수 (System & Maintenance)

##### 1) GET_DEVICE_INFO (장비 정보 조회)

LCU, RTU, 그리고 CMG 자체를 포함한 모든 하드웨어 장비의 상태 및 유지보수 정보를 조회하는 **공용(Universal) 명령어**입니다. 조회 대상 기기(`target_id`)의 특성에 따라 반환되는 데이터 필드가 기기에 맞게 최적화되어 응답됩니다.

> ⚠️ **[참고: 필드 규격 협의 필요]**<br>
> 아래 명시된 각 기기별 `data` 객체 내부의 세부 필드(예: `uptime_sec`, `hw_revision` 등) 구성 및 명칭은 예시이며, 향후 하드웨어/펌웨어 파트너사와의 기술 미팅을 통해 실제 구현 가능한 항목들로 최종 협의하여 확정할 예정입니다.

**[예시 A] 타겟이 LCU일 경우의 응답**

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-5001",
  "cmd_type": "GET_DEVICE_INFO",
  "target_id": "lcu_001122334402"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-5001",
  "target_id": "lcu_001122334402",
  "status": "COMPLETED",
  "data": {
    "device_type": "LCU",
    "fw_version": "v1.2.4",
    "hw_revision": "rev.B",
    "uptime_sec": 86400,
    "plc_mac": "00:11:22:33:44:02",
    "is_bank_swapped": true // 💡 6채널 LCU 전용: 현재 뱅크 스왑 설정 상태
  },
  "message": "Success"
}
```

**[예시 B] 타겟이 RTU일 경우의 응답**

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-5002",
  "cmd_type": "GET_DEVICE_INFO",
  "target_id": "rtu_001122334405"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-5002",
  "target_id": "rtu_001122334405",
  "status": "COMPLETED",
  "data": {
    "device_type": "RTU",     // 💡 기기 타입 명시
    "fw_version": "v2.0.1",
    "hw_revision": "rev.A",
    "uptime_sec": 120500,
    "plc_mac": "00:11:22:33:44:05"
    // RTU는 릴레이가 없으므로 is_bank_swapped 필드 생략
  },
  "message": "Success"
}
```

**[예시 C] 타겟이 CMG일 경우의 응답**

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-5003",
  "cmd_type": "GET_DEVICE_INFO",
  "target_id": "cmg_001122334401"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-5003",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "data": {
    "device_type": "CMG",
    "fw_version": "v3.1.0",
    "hw_revision": "rev.C",
    "uptime_sec": 2592000,
    "ip_address": "192.168.1.100", // 💡 상위 네트워크(Ethernet/Wi-Fi) 정보 포함
    "mac_address": "00:11:22:33:44:01"
  },
  "message": "Success"
}
```

##### 2) SET_BANK_SWAP (LCU 뱅크 결선 논리 스왑)

현장 시공 시 6채널 LCU의 2개 라인(1,2,3 채널 뱅크와 4,5,6 채널 뱅크) 커넥터 결선이 물리적으로 뒤바뀌었을 때, 재시공 없이 펌웨어 단에서 논리/물리 채널을 교차 매핑(1↔4, 2↔5, 3↔6) 하도록 설정합니다.

- **동작 원칙**: 이 설정이 활성화(`is_bank_swapped: true`)되면, LCU는 수신되는 제어 명령뿐만 아니라 상위로 보내는 전력량(`data`) 및 물리 버튼 조작 이벤트(`STATE_CHANGE`)의 채널 번호도 모두 논리적으로 치환하여 보고해야 합니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-5004",
  "cmd_type": "SET_BANK_SWAP",
  "target_id": "lcu_001122334402",
  "is_bank_swapped": true // 💡 통일된 명칭 사용
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-5004",
  "target_id": "lcu_001122334402",
  "status": "COMPLETED",
  "message": "Bank swap configuration applied"
}
```

##### 3) REBOOT (원격 강제 재부팅)

장비를 원격으로 강제 재부팅합니다. (반드시 응답 메시지를 우선 발행한 후 펌웨어 제어 로직을 수행합니다.)

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-5002",
  "cmd_type": "REBOOT",
  "target_id": "cmg_001122334401"
}

// [응답] CMG ➔ Server
{
  "cmd_id": "req-5002",
  "target_id": "cmg_001122334401",
  "status": "COMPLETED",
  "message": "Rebooting system..."
}
```

##### 4) FIRMWARE_UPDATE (펌웨어 업데이트)

장기 소요 작업에 대한 IN_PROGRESS 1차 응답 및 COMPLETED 2차 응답의 예시입니다.

```json
// [요청] Server ➔ CMG
{
  "cmd_id": "req-5003",
  "cmd_type": "FIRMWARE_UPDATE",
  "target_id": "lcu_001122334402",
  "download_url": "[http://exsaver.net/firmware/lcu_v1.3.0.bin](http://exsaver.net/firmware/lcu_v1.3.0.bin)",
  "target_version": "v1.3.0",
  "checksum": "a1b2c3d4e5f6e7f8..."
}

// [1차 응답 - 수신 직후 발행] CMG ➔ Server
{
  "cmd_id": "req-5003",
  "target_id": "lcu_001122334402",
  "status": "IN_PROGRESS",
  "data": {
    "progress_percent": 0
  },
  "message": "Downloading firmware..."
}

// [2차 최종 응답 - 작업 완료 후 발행] CMG ➔ Server
{
  "cmd_id": "req-5003",
  "target_id": "lcu_001122334402",
  "status": "COMPLETED",
  "message": "Firmware update successful"
}
```

<br>

#### 4.3. 자발적 상태 변화 및 실행 결과 보고 (Autonomous Event)

기기 로컬의 물리적 조작이나 내장 스케줄에 의해 상태가 변했을 때, 혹은 현장 터치 패널(RTU)을 통해 설정 데이터가 변경되었을 때 즉시 서버로 전송하여 상위 시스템과의 동기화를 수행합니다. 트래픽 최적화를 위해 이벤트 성격에 따라 분리하여 보고합니다.

---

**[참고] trigger_type (발생 원인) ENUM 정의**
이벤트 발생 시 제어의 주체와 범위를 명확히 하기 위해, 단일 기기 타겟과 다중 기기 타겟을 구분하여 다음 중 하나의 값을 사용합니다.

**A. 단일 기기 제어 원인 (Single-Device Triggers)**
주로 `STATE_CHANGE` 이벤트와 함께 사용됩니다.

- `MANUAL`: LCU 본체 물리 스위치 수동 조작
- `AUTO_CUTOFF`: 대기전력 차단 조건 충족으로 인한 자동 제어 (절감량 집계용)
- `RTU_SINGLE_TOUCH`: 현장 RTU(터치 패널)를 통한 개별 기기 핀포인트 조작
- `SERVER_RELAY_CMD`: 서버 원격 개별 명령(`RELAY_CONTROL` 등) 실행 결과
- `SENSOR_SINGLE`: 외부 센서(재실, 조도 등) 연동에 의한 단일 기기 자동 제어

**B. 다중 기기 일괄 제어 원인 (Multi-Device Triggers)**
주로 `GROUP_EXECUTION_REPORT` 요약 이벤트와 함께 사용됩니다.

- `RTU_GROUP_TOUCH`: 현장 RTU를 통한 구역(논리 그룹) 일괄 조작
- `SCHEDULE`: CMG 내장 스케줄러(시간 도래)에 의한 일괄 자동 실행
- `SERVER_GROUP_CMD`: 서버 원격 논리 그룹 일괄 명령(`GROUP_CONTROL`) 실행 결과

---

##### 1) STATE_CHANGE (단일 기기 상태 변화 보고)

본체 물리 스위치 강제 조작, RTU 개별 기기 조작, 단일 대기전력 자동 차단 등 개별 LCU 단위에서 발생한 소규모 상태 변화를 즉시 보고합니다.

- **Topic**: `exsaver/cmg/{cmg_id}/event/state_change`

```json
{
  "event_type": "STATE_CHANGE",
  "target_id": "cmg_001122334401",
  "trigger_type": "RTU_SINGLE_TOUCH", // 단일 기기 제어 원인 명시
  "trigger_id": "rtu_001122334405", // 조작이 발생한 원천 RTU ID (MANUAL, AUTO_CUTOFF 등일 경우 생략 가능)
  "lcu_id": "lcu_001122334402",
  "channels": [
    { "ch_no": 1, "action": "OFF" },
    { "ch_no": 2, "action": "ON" }
  ],
  "ts": 1773331500
}
```

##### 2) GROUP_EXECUTION_REPORT (다중 기기 일괄 제어 결과 요약 보고)

RTU 터치 구역(Zone) 조작이나 내장 스케줄에 의해 논리 그룹 단위의 대규모 제어가 발생했을 때 발행합니다. 수백 개의 전체 기기 리스트를 생략하고, **성공/실패 요약(Summary)과 제어에 실패한 타겟 배열(Exception)만 전송**하여 네트워크 부하를 극소화합니다.

- **Topic**: `exsaver/cmg/{cmg_id}/event/execution_report`

```json
{
  "event_type": "GROUP_EXECUTION_REPORT",
  "target_id": "cmg_001122334401",
  "trigger_type": "SCHEDULE", // 다중 기기 일괄 제어 원인 명시
  "trigger_id": "sch-001", // 원인이 된 스케줄 ID 또는 RTU Zone ID
  "executed_group_id": "grp-local-light-01",
  "action": "OFF",
  "ts": 1773332000,
  "summary": {
    "total_targets": 100,
    "success_count": 98,
    "fail_count": 2
  },
  // 💡 실패한 기기가 없다면 빈 배열 [] 전송. 서버는 빈 배열 수신 시 100% 성공으로 간주.
  "failed_targets": [
    { "lcu_id": "lcu_001122334455", "ch_no": 1, "reason": "OFFLINE" },
    { "lcu_id": "lcu_001122334455", "ch_no": 2, "reason": "RELAY_FAULT" }
  ]
}
```

##### 3) REPORT_ENTITY_CHANGE (엔티티 데이터 변경 역동기화 보고)

오프라인 상태이거나 RTU 로컬 UI를 통해 현장 관리자가 직접 '스케줄'이나 '그룹' 구성 데이터를 생성, 수정, 삭제했을 경우 발행합니다. 이 데이터를 통해 서버는 CMG의 로컬 변경 사항을 서버 메인 DB에 역으로 업데이트(Upload)하여 충돌을 해결합니다.

- **Topic**: `exsaver/cmg/{cmg_id}/event/entity_change`

```json
{
  "event_type": "REPORT_ENTITY_CHANGE",
  "target_id": "cmg_001122334401",
  "entity_type": "SCHEDULE", // 대상 타입: 'GROUP' 또는 'SCHEDULE'
  "action": "UPSERT", // 수행 작업: 'UPSERT'(생성/수정) 또는 'DELETE'(삭제)
  "ts": 1773335000,
  "data": {
    // 💡 action이 'DELETE'일 경우 id 정보만 포함
    // 💡 action이 'UPSERT'일 경우 변경된 전체 최신 객체와 해시/수정시간 포함
    "schedule_id": "sch-001",
    "updated_ts": 1773334900,
    "hash": "a1b2c3d4e5f6...",
    "time": "19:00",
    "days": ["MON", "TUE", "WED", "THU", "FRI"],
    "actions": [
      {
        "target_type": "GROUP",
        "target_id": "grp-local-light-01",
        "action": "OFF"
      }
    ]
  }
}
```

##### 4) SYNC_DIGEST (데이터 무결성 확인용 해시 요약 보고)

CMG가 재부팅되어 서버와 재접속(Birth Message 발행 직후)하거나, 서버의 `REQUEST_SYNC_DIGEST` 명령을 수신했을 때 발행합니다. 그룹과 스케줄 데이터의 해시 요약본만 가볍게 서버로 전송하여 양측의 데이터 불일치를 감지하는 '방어적 동기화'의 핵심 이벤트입니다.

- **Topic**: `exsaver/cmg/{cmg_id}/event/sync_digest`

```json
{
  "event_type": "SYNC_DIGEST",
  "target_id": "cmg_001122334401",
  "ts": 1773316800,
  "data": {
    "groups": [
      {
        "id": "grp-local-light-01",
        "hash": "a1b2c3d4e5f6...",
        "updated_ts": 1773315000
      },
      {
        "id": "grp-local-hvac-01",
        "hash": "f8e7d6c5b4a3...",
        "updated_ts": 1773315000
      }
    ],
    "schedules": [
      { "id": "sch-001", "hash": "9d8c7b6a5e4f...", "updated_ts": 1773316000 }
    ]
  }
}
```

<br>

### 1.4. 데이터 무결성 검증용 해시(Hash) 생성 규칙

멀티 마스터 동기화 및 `SYNC_DIGEST` 이벤트에서 사용되는 `hash` 값은 서버와 CMG 간의 데이터 일치 여부를 판별하는 핵심 키입니다. JSON 직렬화 라이브러리 간의 공백/순서 차이로 인한 해시 불일치를 방지하기 위해, 반드시 아래의 **평문 결합 규칙(Plain Text Concatenation)**을 거친 후 해싱해야 합니다.

- **해시 알고리즘**: `SHA-256`
- **출력 포맷**: 64자리 소문자 Hex String

##### 1) 그룹(Group) 해시 생성 규칙

- **결합 포맷**: `{group_id}|{lcu_id}:{ch_no},{lcu_id}:{ch_no},...`
- **정렬 규칙**: `targets` 배열을 `lcu_id` 오름차순(A-Z) ➔ `ch_no` 오름차순 순서로 반드시 정렬한 후 콤마(`,`)로 이어 붙입니다.
- **생성 예시**:
  - 데이터: `grp-local-light-01`에 `lcu_...02`의 2번, 1번 채널이 속해 있을 경우
  - 평문: `grp-local-light-01|lcu_001122334402:1,lcu_001122334402:2`
  - 최종 Hash (SHA-256): `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` (예시)

##### 2) 스케줄(Schedule) 해시 생성 규칙

- **결합 포맷**: `{schedule_id}|{time}|{days}|{actions}`
- **정렬 및 결합 규칙**:
  - `days`: 월요일(MON)부터 일요일(SUN) 순서대로 정렬하여 콤마(`,`)로 결합. (예: `MON,TUE,WED`)
  - `actions`: `target_type` ➔ `target_id` (또는 `lcu_id`) 오름차순 ➔ `ch_no` 오름차순으로 정렬.
  - 액션 개별 포맷: `DEVICE` 타입은 `D:{lcu_id}:{ch_no}:{action}`, `GROUP` 타입은 `G:{target_id}:{action}` 형태로 변환 후 콤마(`,`)로 결합.
- **생성 예시**:
  - 데이터: `sch-001`, `18:00`, 월/수/금, [그룹 1개 OFF, 단일 LCU 1개 OFF]
  - 평문: `sch-001|18:00|MON,WED,FRI|D:lcu_001122334405:4:OFF,G:grp-local-light-01:OFF`
  - 최종 Hash (SHA-256): `(생성된 64자리 소문자 Hex 문자열)`
