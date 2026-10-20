# API v1.0 가이드

**Integration > EventBus > API 가이드 > API v1.0 가이드**

EventBus API로 사용자 애플리케이션에서 커스텀 이벤트를 발행하는 방법을 안내합니다. 발행한 이벤트는 버스에 등록된 규칙에 따라 대상으로 전달됩니다.

## API 공통 정보

### API 엔드포인트

| 리전 | 엔드포인트 |
|---|---|
| 한국(판교) 리전 | {API 엔드포인트} |

### 인증 및 권한

EventBus API는 User Access Key 토큰으로 인증합니다. 토큰을 발급하는 방법은 [User Access Key 토큰](/nhncloud/ko/public-api/user-access-key-token/)을 참고합니다.

모든 요청에 다음 헤더를 포함합니다.

| 이름 | 위치 | 필수 | 설명 |
|---|---|---|---|
| X-NHN-authorization | Header | 필수 | 발급받은 토큰입니다. `Bearer {토큰}` 형식으로 입력합니다. |
| X-NHN-appkey | Header | 필수 | EventBus를 사용하는 프로젝트의 앱키입니다. |
| X-EventBus-Bus | Header | 필수 | 이벤트를 발행할 버스의 버스 키입니다. 버스 키는 콘솔의 버스 상세 정보 **기본 정보** 탭에서 확인할 수 있습니다. |

### 요청 공통 정보

* 요청 본문은 JSON 형식으로 보내며, `Content-Type: application/json` 헤더를 포함합니다.
* 요청 본문 전체 크기는 최대 1MB입니다.
* 이벤트는 CloudEvents v1.0 형식으로 작성합니다. 자세한 내용은 [이벤트 레퍼런스](./event-list/)를 참고합니다.

### 응답 공통 정보

요청에 성공하면 HTTP 상태 코드 `202 Accepted`와 함께 결과를 반환합니다. 요청에 실패하면 HTTP 상태 코드와 함께 다음 형식의 오류 응답을 반환합니다.

**오류 응답 본문**

```json
{
  "error": "INVALID_REQUEST",
  "message": "요청 형식이 올바르지 않습니다.",
  "detail": "CloudEvents 필수 속성 누락: source",
  "timestamp": "2026-11-24T10:00:00Z"
}
```

| 이름 | 타입 | 설명 |
|---|---|---|
| error | String | 오류 코드입니다. |
| message | String | 오류 메시지입니다. |
| detail | String | 오류 상세 정보입니다. |
| timestamp | String | 오류가 발생한 시각입니다. |

### 호출량 제한

호출량 한도를 넘으면 `429 Too Many Requests`를 반환합니다. 응답의 `Retry-After` 헤더에 표시된 시간(초)만큼 기다린 뒤 다시 요청합니다.

| 기준 | 한도 |
|---|---|
| 프로젝트 | 초당 100회 |
| 버스 | 초당 50회 |

!!! tip "알아두기"
* 일괄 발행 API는 이벤트 수와 관계없이 요청 1회를 호출 1회로 셉니다.

## 이벤트 API

### 이벤트 발행

이벤트 1건을 발행합니다.

#### 요청

```
POST /v1/events
```

**요청 본문**

| 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|
| specversion | String | 필수 | CloudEvents 버전입니다. `1.0`으로 입력합니다. |
| id | String | 필수 | 이벤트 ID입니다. 발행하는 쪽에서 정합니다. |
| source | String | 필수 | 이벤트가 발생한 출처입니다. URI-reference 형식으로 입력합니다. |
| type | String | 필수 | 이벤트 유형입니다. |
| time | String | 선택 | 이벤트 발생 시각입니다. RFC 3339 형식으로 입력합니다. |
| subject | String | 선택 | 이벤트 주체입니다. 규칙의 필터 조건으로 사용할 수 있습니다. |
| datacontenttype | String | 선택 | `data`의 미디어 유형입니다. |
| dataschema | String | 선택 | `data`가 따르는 스키마의 URI입니다. |
| data | Object | 선택 | 이벤트 데이터입니다. |

**요청 예시**

```
curl -X POST "{API 엔드포인트}/v1/events" \
  -H "Content-Type: application/json" \
  -H "X-NHN-authorization: Bearer {토큰}" \
  -H "X-NHN-appkey: {앱키}" \
  -H "X-EventBus-Bus: {버스 키}" \
  -d '{
    "specversion": "1.0",
    "id": "order-a1-created",
    "source": "order-service",
    "type": "order.created",
    "data": { "orderId": "A-1", "amount": 15000 }
  }'
```

#### 응답

| 이름 | 타입 | 설명 |
|---|---|---|
| eventId | String | EventBus가 발급한 이벤트 ID입니다. |
| status | String | 처리 상태입니다. 성공하면 `ACCEPTED`입니다. |
| timestamp | String | 이벤트 시각입니다. |

**응답 예시**

```json
{
  "eventId": "evt-20261124-abcd1234",
  "status": "ACCEPTED",
  "timestamp": "2026-11-24T10:00:00Z"
}
```

### 이벤트 일괄 발행

이벤트를 최대 10건까지 한 번에 발행합니다. 이벤트마다 따로 검증하며, 일부 이벤트만 실패해도 나머지 이벤트는 발행됩니다.

#### 요청

```
POST /v1/events/batch
```

**요청 본문**

이벤트 발행 API의 요청 본문과 같은 형식의 이벤트를 배열로 입력합니다. 배열에는 이벤트를 1~10건 담을 수 있습니다.

**요청 예시**

```
curl -X POST "{API 엔드포인트}/v1/events/batch" \
  -H "Content-Type: application/json" \
  -H "X-NHN-authorization: Bearer {토큰}" \
  -H "X-NHN-appkey: {앱키}" \
  -H "X-EventBus-Bus: {버스 키}" \
  -d '[
    { "specversion": "1.0", "id": "order-a1", "source": "order-service",
      "type": "order.created", "data": { "orderId": "A-1", "amount": 15000 } },
    { "specversion": "1.0", "id": "order-a2", "source": "order-service",
      "type": "order.created", "data": { "orderId": "A-2", "amount": 22000 } }
  ]'
```

#### 응답

| 이름 | 타입 | 설명 |
|---|---|---|
| totalCount | Integer | 요청한 이벤트 수입니다. |
| successCount | Integer | 발행에 성공한 이벤트 수입니다. |
| failedCount | Integer | 발행에 실패한 이벤트 수입니다. |
| results | Array | 이벤트별 결과입니다. 요청한 순서대로 반환합니다. |
| results[].index | Integer | 요청 배열에서 이벤트의 위치입니다. 0부터 시작합니다. |
| results[].status | String | 처리 상태입니다. `ACCEPTED` 또는 `FAILED`입니다. |
| results[].eventId | String | 성공한 경우 EventBus가 발급한 이벤트 ID입니다. |
| results[].error | String | 실패한 경우 오류 코드입니다. |
| results[].message | String | 실패한 경우 오류 메시지입니다. |

**응답 예시**

```json
{
  "totalCount": 2,
  "successCount": 1,
  "failedCount": 1,
  "results": [
    { "index": 0, "status": "ACCEPTED", "eventId": "evt-20261124-abcd1234" },
    { "index": 1, "status": "FAILED", "error": "INVALID_ENTRY", "message": "CloudEvents 필수 속성 누락: source" }
  ]
}
```

!!! tip "알아두기"
* `servicesource`, `eventtype`, `eventid`, `busid`는 EventBus가 사용하는 확장 속성입니다. 같은 이름의 속성을 보내면 EventBus의 값으로 바뀝니다.

!!! danger "주의"
* 일괄 발행 API는 일부 이벤트가 실패해도 `202 Accepted`를 반환합니다. 응답 본문의 `failedCount`와 `results`를 확인하여 실패한 이벤트를 다시 발행합니다.
* 이벤트를 한 건도 발행하지 못하고 그 원인이 EventBus 내부에 있으면 `500`, `502`, `504`를 반환합니다. 이때는 요청 전체를 다시 보냅니다.
