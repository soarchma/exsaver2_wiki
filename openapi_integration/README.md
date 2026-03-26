# EXSAVER 2.0 외부 통합 관제 시스템 OpenAPI 연동 규격서

**문서 버전**: v1.0.0-draft.1<br>
**작성일**: 2026-03-10<br>
**대상**: EXSAVER 2.0 플랫폼과 연동하려는 3rd-party 시스템 개발자 (스마트 시티, 타사 BEMS 등)

[⬅️ EXSAVER 2.0 통합 기술 문서 홈으로 돌아가기](../README.md)

## 1. 연동 아키텍처 개요 (Northbound API)

EXSAVER 2.0은 대규모 트래픽 및 실시간 연동을 지원하기 위해 업계 표준인 RESTful API와 WebSocket(또는 SSE)을 혼합 제공합니다. 모든 실시간 조회 요청은 메인 DB가 아닌 **초고속 Redis In-memory Cache**를 통해 반환되므로, 1초 단위의 잦은 폴링(Polling)에도 안정적인 성능을 보장합니다.

## 2. 인증 및 보안 (Authentication)

외부 시스템 연동은 사용자 로그인 방식(JWT)이 아닌 **Server-to-Server 인증 방식**을 사용합니다.

- **방식**: `API Key` (또는 OAuth 2.0 Client Credentials Grant)
- **헤더 규격**: `Authorization: Bearer {ISSUED_API_KEY}`
- 모든 API 요청은 HTTPS 기반으로 암호화되어야 합니다.

## 3. REST API 규격 (상태 조회 및 과거 통계)

### 3.1. 사이트/건물 내 전체 기기 상태 일괄 조회 (Polling 최적화)

- **Endpoint**: `GET /api/v1/external/sites/{site_id}/status`
- **목적**: 특정 건물 내 모든 LCU/채널의 현재 전력량 및 릴레이 상태를 한 번에 조회합니다. (Redis Cache에서 즉시 반환)
- **Response 예시**:

```json
{
  "timestamp": "2026-03-10T14:00:00Z",
  "site_id": "site-test",
  "devices": [
    {
      "lcu_id": "lcu_001122334402",
      "ch_no": 1,
      "status": "ON",
      "power_w": 150.5,
      "cumulative_wh": 10500.5
    }
  ]
}
```

### 3.2. 과거 전력 통계 데이터 조회

- **Endpoint**: `GET /api/v1/external/sites/{site_id}/history`
- **Query Params**: `start_date`, `end_date`, `interval` (hourly, daily, monthly)
- **목적**: TimescaleDB의 CAGG(자동 집계 뷰)에서 정제된 통계 데이터를 외부 시스템에 제공합니다.

### 3.3. 외부 기기 제어 요청 (Control)

- **Endpoint**: `POST /api/v1/external/devices/control`
- **목적**: 외부 플랫폼에서 EXSAVER 하위의 릴레이를 제어(ON/OFF)합니다.
- **Body 예시**:

```json
{
  "lcu_id": "lcu_001122334402",
  "ch_no": 1,
  "action": "OFF"
}
```

## 4. 실시간 스트리밍 (WebSocket / SSE 연동)

1초 미만의 지연율이 필요한 통합 관제 센터 대시보드를 위해 상태 구독 모델을 지원합니다.

- **Endpoint**: `wss://api.exsaver.net/v1/stream/site/{site_id}`
- **동작 방식**: 외부 시스템이 소켓을 연결하면, EXSAVER의 Redis Pub/Sub 채널이 해당 건물에서 발생하는 상태 변화(전력량 갱신, 릴레이 상태 변경 등)를 Event 형식으로 실시간 Push 합니다.

## 5. 이벤트 웹훅 (Webhook)

장비 고장(Offline)이나 과부하 알람 등 비동기적이고 치명적인 이벤트를 외부 시스템에 즉각 통보합니다.

- **동작 방식**: 관리자 페이지에서 외부 시스템의 수신 URL(Target Endpoint)을 등록합니다.
- **Payload 예시**:

```json
{
  "event_type": "DEVICE_OFFLINE",
  "lcu_id": "lcu_001122334402",
  "message": "Device disconnected from CMG",
  "timestamp": "2026-03-10T14:15:00Z"
}
```
