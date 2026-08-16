# Proposal API

## 목적

추천 상대에게 구체적인 날짜·시간·활동·장소로 만남을 제안하고, 수신자가 수락 또는 거절하며, 발신자가 대기 중 제안을 취소한다.

## 상태

```text
PENDING ──accept──> ACCEPTED
   ├──────reject──> REJECTED
   └──────cancel──> CANCELED
```

종료 상태인 `ACCEPTED`, `REJECTED`, `CANCELED`에서 제안 상태를 다시 변경할 수 없다. `ACCEPTED` 후 약속 취소는 Appointment API를 사용한다.

## 타입

```ts
type ProposalStatus = "PENDING" | "ACCEPTED" | "REJECTED" | "CANCELED";

type UserBrief = {
  userId: string;
  nickname: string;
  profileImageUrl: string | null;
  department: string;
};

type Proposal = {
  id: string;
  sender: UserBrief;
  receiver: UserBrief;
  date: string;
  startTime: string;
  endTime: string;
  timeZone: "Asia/Seoul";
  activity: Activity;
  place: string;
  message: string | null;
  status: ProposalStatus;
  createdAt: string;
  respondedAt: string | null;
  canceledAt: string | null;
};
```

## POST `/proposals`

추천 상대에게 만남 제안을 생성한다.

### 인증

Bearer Token 필요

### 권장 헤더

```http
Idempotency-Key: 9dfc0fb4-e8a5-49ba-9ed4-59059e6dc3c1
```

### Request Body

```json
{
  "receiverUserId": "01JZ8G1FB5Z3N8GDRP0EBWSG4H",
  "date": "2026-08-17",
  "startTime": "12:00",
  "endTime": "13:00",
  "timeZone": "Asia/Seoul",
  "activity": "LUNCH",
  "place": "학생회관 1층",
  "message": "월요일에 같이 점심 먹어요!"
}
```

### 검증

| 필드 | 필수 | 규칙 |
|---|---|---|
| `receiverUserId` | O | 자기 자신이 아닌 활성 추천 상대 |
| `date` | O | 오늘 이후, 다음 28일 이내의 `YYYY-MM-DD` |
| `startTime`, `endTime` | O | 30분 단위, 시작 < 종료 |
| `timeZone` | O | MVP는 `Asia/Seoul` |
| `activity` | O | 두 사용자의 공통 활동 중 하나 |
| `place` | O | 공백 제거 후 2~50자 |
| `message` | X | `null` 또는 최대 200자 |

- 날짜의 요일과 공통 가능 요일이 일치해야 한다.
- 제안 시간 전체가 두 사용자의 최신 공통 가능 시간 안에 포함되어야 한다.
- 제안 길이는 양쪽 최소 만남 시간 중 더 긴 값을 충족해야 한다.

### Success `201 Created`

```http
Location: /api/v1/proposals/proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40
```

```json
{
  "data": {
    "id": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
    "sender": {
      "userId": "01JZ8A1F01A9H9M4RNB8N4M88M",
      "nickname": "민지",
      "profileImageUrl": null,
      "department": "컴퓨터과학과"
    },
    "receiver": {
      "userId": "01JZ8G1FB5Z3N8GDRP0EBWSG4H",
      "nickname": "Alex",
      "profileImageUrl": null,
      "department": "경영학과"
    },
    "date": "2026-08-17",
    "startTime": "12:00",
    "endTime": "13:00",
    "timeZone": "Asia/Seoul",
    "activity": "LUNCH",
    "place": "학생회관 1층",
    "message": "월요일에 같이 점심 먹어요!",
    "status": "PENDING",
    "createdAt": "2026-08-16T06:40:00Z",
    "respondedAt": null,
    "canceledAt": null
  }
}
```

### Errors

| Status | code | 조건 |
|---:|---|---|
| `404` | `MATCH_NOT_FOUND` | 현재 제안 가능한 추천 상대가 아님 |
| `409` | `PROPOSAL_ALREADY_EXISTS` | 같은 사용자·날짜·시간의 대기 제안 존재 |
| `409` | `COMMON_SLOT_CHANGED` | 입력 사이 양쪽 가능 시간 변경 |
| `409` | `USER_BLOCKED` | 차단 관계 발생 |
| `422` | `DATE_OUT_OF_RANGE` | 날짜가 과거 또는 28일 초과 |
| `422` | `TIME_NOT_IN_COMMON_SLOT` | 공통 시간 밖 |
| `422` | `MINIMUM_DURATION_NOT_MET` | 최소 만남 시간 미달 |
| `422` | `ACTIVITY_NOT_SHARED` | 공통 활동이 아님 |
| `422` | `VALIDATION_ERROR` | 기타 입력 오류 |

## GET `/proposals`

내가 보냈거나 받은 제안 목록을 조회한다.

### Query Parameters

| 이름 | 타입 | 기본값 | 허용값 |
|---|---|---:|---|
| `direction` | enum | `ALL` | `SENT`, `RECEIVED`, `ALL` |
| `status` | enum list | 전체 | 쉼표 구분 `PENDING,ACCEPTED` 등 |
| `limit` | integer | `20` | `1~50` |
| `cursor` | string | 없음 | 서버 cursor |

기본 정렬은 `PENDING` 우선, 그 안에서 생성 시각 내림차순, 이후 나머지 생성 시각 내림차순이다.

### Success `200 OK`

```json
{
  "data": [
    {
      "id": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
      "direction": "SENT",
      "counterpart": {
        "userId": "01JZ8G1FB5Z3N8GDRP0EBWSG4H",
        "nickname": "Alex",
        "profileImageUrl": null,
        "department": "경영학과"
      },
      "date": "2026-08-17",
      "startTime": "12:00",
      "endTime": "13:00",
      "timeZone": "Asia/Seoul",
      "activity": "LUNCH",
      "place": "학생회관 1층",
      "status": "PENDING",
      "createdAt": "2026-08-16T06:40:00Z"
    }
  ],
  "meta": {
    "hasNext": false,
    "nextCursor": null
  }
}
```

## GET `/proposals/{proposalId}`

내가 발신자 또는 수신자인 제안 상세를 조회한다.

### Success `200 OK`

POST 생성 응답과 동일한 Proposal을 반환한다. 수락된 경우 다음 필드를 추가한다.

```json
{
  "data": {
    "id": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
    "status": "ACCEPTED",
    "appointmentId": "appointment_01JZ8J16T2A02AG7P5QCH2N6DH"
  }
}
```

실제 응답에는 위 필드 외 Proposal 전체 필드가 포함된다.

### Errors

- `404 PROPOSAL_NOT_FOUND`: 존재하지 않거나 내가 참여하지 않은 제안

## POST `/proposals/{proposalId}/accept`

수신자가 대기 제안을 수락하고 약속을 생성한다.

### 인증·권한

Bearer Token 필요. 제안 수신자만 가능하다.

### Request Body

없음

### Success `200 OK`

```json
{
  "data": {
    "proposal": {
      "id": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
      "status": "ACCEPTED",
      "respondedAt": "2026-08-16T06:50:00Z"
    },
    "appointment": {
      "id": "appointment_01JZ8J16T2A02AG7P5QCH2N6DH",
      "proposalId": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
      "date": "2026-08-17",
      "startTime": "12:00",
      "endTime": "13:00",
      "timeZone": "Asia/Seoul",
      "activity": "LUNCH",
      "place": "학생회관 1층",
      "status": "CONFIRMED",
      "createdAt": "2026-08-16T06:50:00Z"
    }
  }
}
```

수락 성공은 제안 상태 변경과 약속 생성을 하나의 트랜잭션으로 처리한다.

### Errors

| Status | code | 조건 |
|---:|---|---|
| `403` | `PROPOSAL_RECEIVER_ONLY` | 발신자 또는 제3자가 수락 시도 |
| `404` | `PROPOSAL_NOT_FOUND` | 존재하지 않거나 참여하지 않음 |
| `409` | `PROPOSAL_NOT_PENDING` | 이미 처리된 제안 |
| `409` | `COMMON_SLOT_CHANGED` | 수락 직전 가능 시간 변경 |
| `409` | `APPOINTMENT_TIME_CONFLICT` | 기존 확정 약속과 시간 충돌 |

## POST `/proposals/{proposalId}/reject`

수신자가 대기 제안을 거절한다.

### Request Body

없음

### Success `200 OK`

```json
{
  "data": {
    "id": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
    "status": "REJECTED",
    "respondedAt": "2026-08-16T06:50:00Z"
  }
}
```

### Errors

- `403 PROPOSAL_RECEIVER_ONLY`
- `404 PROPOSAL_NOT_FOUND`
- `409 PROPOSAL_NOT_PENDING`

## POST `/proposals/{proposalId}/cancel`

발신자가 대기 중인 제안을 취소한다.

### Request Body

없음

### Success `200 OK`

```json
{
  "data": {
    "id": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
    "status": "CANCELED",
    "canceledAt": "2026-08-16T06:52:00Z"
  }
}
```

### Errors

- `403 PROPOSAL_SENDER_ONLY`
- `404 PROPOSAL_NOT_FOUND`
- `409 PROPOSAL_NOT_PENDING`

## 상태 충돌 공통 응답

```json
{
  "error": {
    "code": "PROPOSAL_NOT_PENDING",
    "message": "이미 처리된 제안입니다.",
    "details": [
      {
        "field": "status",
        "reason": "현재 상태는 ACCEPTED입니다."
      }
    ],
    "requestId": "req_01JZ8J67HJ4EZ6GSD0QTYMZP0X"
  }
}
```

프론트는 이 오류를 받으면 상세와 목록을 다시 조회한다.

## 프론트 Mock 상태

1. 보낸 `PENDING` 제안
2. 받은 `PENDING` 제안
3. `ACCEPTED`, `REJECTED`, `CANCELED` 상태
4. 제안 생성 중 공통 시간 변경 `409`
5. 동시 수락으로 상태 충돌 `409`
6. 빈 제안 목록
