# 이벤트 레퍼런스

**Integration > EventBus > 이벤트 레퍼런스**

EventBus가 전달하는 이벤트의 구조와 지원하는 이벤트, 규칙에서 이벤트를 고르는 이벤트 패턴의 작성 방법을 안내합니다.

## 이벤트 구조

EventBus는 모든 이벤트를 [CloudEvents v1.0](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) 형식으로 다룹니다. 대상에는 이벤트 하나를 JSON 본문으로 전송합니다. 외부 HTTP 대상에는 POST 메서드와 `Content-Type: application/json` 헤더로 전송합니다.

| 속성 | 설명 | NHN Cloud 이벤트 | 커스텀 이벤트 |
|---|---|---|---|
| specversion | CloudEvents 버전 | `1.0` | `1.0` |
| id | 이벤트 ID | CloudTrail 이벤트 로그 ID | 발행할 때 입력한 값 |
| source | 이벤트가 발생한 출처 | 이벤트가 발생한 조직과 프로젝트를 나타내는 경로 | 발행할 때 입력한 값 |
| type | 이벤트 유형 | CloudTrail 이벤트 ID | 발행할 때 입력한 값 |
| time | 이벤트 발생 시각 | CloudTrail 이벤트 발생 시각 | 발행할 때 입력한 값. 입력하지 않으면 EventBus가 이벤트를 받은 시각 |
| datacontenttype | `data`의 미디어 유형 | `application/json` | `application/json` |
| subject | 이벤트 주체 | 없음 | 발행할 때 입력한 값(선택) |
| dataschema | `data`가 따르는 스키마의 URI | 없음 | 발행할 때 입력한 값(선택) |
| data | 이벤트 데이터 | CloudTrail 이벤트 원본 | 발행할 때 입력한 값 |

**NHN Cloud 이벤트 예시**

```json
{
  "specversion": "1.0",
  "id": "0f8b7c6e-1d2a-4e3b-9c5d-6a7b8c9d0e1f",
  "source": "/cloudtrail/organizations/{orgId}/projects/{projectId}",
  "type": "event_id.object_storage.object.upload",
  "time": "2026-11-24T10:00:00.000Z",
  "datacontenttype": "application/json",
  "data": {
    "eventSource": { "projectId": "{projectId}", "region": "KR1" }
  }
}
```

## 지원 이벤트

NHN Cloud 이벤트는 CloudTrail이 수집하는 이벤트 중 프로젝트에서 발생한 이벤트를 지원합니다. 지원하는 서비스와 이벤트는 콘솔의 규칙 생성 화면에서 서비스와 이벤트를 선택할 때 확인할 수 있습니다. 각 이벤트의 상세 내용은 CloudTrail의 [수집되는 이벤트 목록](/Governance%20%26%20Audit/CloudTrail/ko/event-list/)을 참고합니다.

JSON 편집기로 이벤트 패턴을 작성할 때는 CloudTrail 이벤트 ID(`event_id.{서비스}.{이벤트}`)를 기준으로 다음과 같이 서비스와 이벤트를 지정합니다.

| 속성 | 값 | 예시 |
|---|---|---|
| servicesource | `nhncloud.` 뒤에 이벤트 ID의 서비스 부분을 붙인 값 | `nhncloud.iaas` |
| eventtype | 이벤트 ID에서 `event_id.{서비스}.`를 뺀 나머지 | `event_id.iaas.instance.create` → `instance.create` |

사용자 애플리케이션이 발행한 이벤트는 커스텀 이벤트입니다. 커스텀 이벤트의 `servicesource`는 `nhncloud.custom`입니다.

!!! tip "알아두기"
    * 조직 단위로 발생하는 이벤트는 지원하지 않습니다.
    * 이벤트를 자동으로 수신하지 않는 서비스의 이벤트는 커스텀 이벤트로 직접 발행하여 사용할 수 있습니다.

## 이벤트 패턴 작성하기

이벤트 패턴은 규칙이 전달할 이벤트를 고르는 조건입니다. 콘솔의 폼 빌더로 작성하거나 JSON 편집기로 직접 작성할 수 있습니다. 이 절에서는 JSON 편집기에서 사용하는 패턴 문법을 설명합니다.

### 기본 규칙

이벤트 패턴은 JSON 객체입니다. 키에는 이벤트의 속성 이름을, 값에는 일치할 값의 배열을 입력합니다. `data`처럼 하위 구조가 있는 속성은 객체로 중첩하여 입력합니다.

* 한 속성의 배열에 값을 여러 개 입력하면 그중 하나와 일치할 때 조건을 만족합니다(OR).
* 속성을 여러 개 입력하면 모든 속성의 조건을 만족할 때 이벤트를 전달합니다(AND).
* 서비스와 이벤트는 `servicesource`와 `eventtype` 속성으로 지정합니다. 서비스의 모든 이벤트를 전달하려면 `eventtype`을 입력하지 않습니다.
* 조건으로 사용할 수 있는 속성은 NHN Cloud 이벤트가 `source`, `time`, `id`, `data`이고, 커스텀 이벤트는 여기에 `subject`를 더 사용할 수 있습니다.

**Instance 생성 이벤트만 전달**

```json
{
  "servicesource": ["nhncloud.iaas"],
  "eventtype": ["instance.create"]
}
```

**Object Storage의 모든 이벤트 전달**

```json
{
  "servicesource": ["nhncloud.object_storage"]
}
```

### 비교 연산자

값 자리에 다음 연산자를 사용할 수 있습니다.

| 연산자 | 문법 | 조건 |
|---|---|---|
| 정확히 일치 | `["order-app"]` | 값이 정확히 같을 때 |
| 접두사 | `[{"prefix": "order-"}]` | 지정한 문자열로 시작할 때 |
| 접미사 | `[{"suffix": "-app"}]` | 지정한 문자열로 끝날 때 |
| 제외 | `[{"anything-but": "test-app"}]` | 지정한 값이 아닐 때 |
| 존재 여부 | `[{"exists": true}]` | 속성이 있을 때(`false`이면 없을 때) |
| 숫자 범위 | `[{"numeric": [">=", 0, "<", 10]}]` | 숫자가 범위 안에 있을 때 |

**커스텀 이벤트 중 주문 서비스에서 발생한 1만 원 이상 주문만 전달**

```json
{
  "servicesource": ["nhncloud.custom"],
  "source": [{"prefix": "order-"}],
  "type": ["order.created"],
  "data": {
    "amount": [{"numeric": [">=", 10000]}]
  }
}
```

!!! tip "알아두기"
    * `servicesource`, `eventtype`, `eventid`, `busid`는 EventBus가 사용하는 확장 속성으로, 대상에 전달하는 이벤트에는 포함되지 않습니다.
    * 폼 빌더는 위 6가지 연산자를 제공합니다. 여러 서비스를 함께 지정하는 등 복잡한 조건은 JSON 편집기로 작성합니다.

!!! danger "주의"
    * 규칙은 버스가 속한 프로젝트의 이벤트만 받습니다. 프로젝트와 리전 조건은 EventBus가 자동으로 추가하며, 패턴에 `busid`나 `data.eventSource`의 `projectId`, `region`을 입력해도 EventBus의 값으로 바뀝니다.
    * 패턴의 속성 이름은 검사하지 않습니다. 이벤트에 없는 속성 이름을 입력하면 이벤트가 전달되지 않으므로 속성 이름을 정확히 입력합니다.
