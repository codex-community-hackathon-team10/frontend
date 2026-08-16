# Appointment API

## 목적

수락된 제안으로 생성된 확정 약속을 조회하고 취소한다. 실시간 채팅과 연락처 공개는 MVP 범위에 포함하지 않는다.

## 상태

```text
CONFIRMED ──cancel──> CANCELED
```

지난 약속은 별도 상태로 변경하지 않는다. 서버는 현재 시각과 `date`, `endTime`을 비교해 목록의 `timeCategory`를 계산한다.

## 타입

```ts
type AppointmentStatus = "CONFIRMED" | "CANCELED";
type AppointmentTimeCategory = "UPCOMING" | "PAST" | "CANCELED";

type Appointment = {
  id: string;
  proposalId: string;
  counterpart: UserBrief;
  date: string;
  startTime: string;
  endTime: string;
  timeZone: "Asia/Seoul";
  activity: Activity;
  place: string;
  message: string | null;
  status: AppointmentStatus;
  timeCategory: AppointmentTimeCategory;
  createdAt: string;
  canceledAt: string | null;
  canceledByUserId: string | null;
};
```

## GET `/appointments`

내 약속 목록을 조회한다.

### 인증

Bearer Token 필요

### Query Parameters

| 이름 | 타입 | 기본값 | 허용값 |
|---|---|---:|---|
| `category` | enum | `UPCOMING` | `UPCOMING`, `PAST`, `CANCELED`, `ALL` |
| `limit` | integer | `20` | `1~50` |
| `cursor` | string | 없음 | 서버 cursor |

### 정렬

- `UPCOMING`: 날짜·시작 시간 오름차순
- `PAST`: 날짜·시작 시간 내림차순
- `CANCELED`: 취소 시각 내림차순
- `ALL`: 최근 생성 시각 내림차순

### Success `200 OK`

```json
{
  "data": [
    {
      "id": "appointment_01JZ8J16T2A02AG7P5QCH2N6DH",
      "proposalId": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
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
      "message": "월요일에 같이 점심 먹어요!",
      "status": "CONFIRMED",
      "timeCategory": "UPCOMING",
      "createdAt": "2026-08-16T06:50:00Z",
      "canceledAt": null,
      "canceledByUserId": null
    }
  ],
  "meta": {
    "hasNext": false,
    "nextCursor": null
  }
}
```

빈 목록은 `data: []`를 반환한다.

### Errors

| Status | code | 조건 |
|---:|---|---|
| `400` | `INVALID_QUERY` | category·limit·cursor 오류 |

## GET `/appointments/{appointmentId}`

내가 참여한 약속 상세를 조회한다.

### 인증

Bearer Token 필요

### Success `200 OK`

```json
{
  "data": {
    "id": "appointment_01JZ8J16T2A02AG7P5QCH2N6DH",
    "proposalId": "proposal_01JZ8HR66FQ3F6VM8FJ5V5NB40",
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
    "message": "월요일에 같이 점심 먹어요!",
    "status": "CONFIRMED",
    "timeCategory": "UPCOMING",
    "createdAt": "2026-08-16T06:50:00Z",
    "canceledAt": null,
    "canceledByUserId": null
  }
}
```

### Errors

| Status | code | 조건 |
|---:|---|---|
| `404` | `APPOINTMENT_NOT_FOUND` | 존재하지 않거나 내가 참여하지 않음 |

## POST `/appointments/{appointmentId}/cancel`

참여자가 확정 약속을 취소한다.

### 인증

Bearer Token 필요. 발신자와 수신자 모두 가능하다.

### Request Body

```json
{
  "reason": "일정이 변경되었어요."
}
```

| 필드 | 필수 | 검증 |
|---|---|---|
| `reason` | X | `null` 또는 최대 200자 |

취소 사유는 상대에게 표시하지 않고 운영 기록에만 저장한다.

### Success `200 OK`

```json
{
  "data": {
    "id": "appointment_01JZ8J16T2A02AG7P5QCH2N6DH",
    "status": "CANCELED",
    "timeCategory": "CANCELED",
    "canceledAt": "2026-08-16T07:00:00Z",
    "canceledByUserId": "01JZ8A1F01A9H9M4RNB8N4M88M"
  }
}
```

### Errors

| Status | code | 조건 |
|---:|---|---|
| `404` | `APPOINTMENT_NOT_FOUND` | 존재하지 않거나 참여하지 않음 |
| `409` | `APPOINTMENT_ALREADY_CANCELED` | 이미 취소됨 |
| `409` | `APPOINTMENT_ALREADY_ENDED` | 종료 시각이 지남 |
| `422` | `VALIDATION_ERROR` | 취소 사유 길이 오류 |

## 비활성 상대 표시

약속 상대가 탈퇴 또는 비활성 상태가 되면 약속 기록은 유지하고 다음처럼 반환한다.

```json
{
  "counterpart": {
    "userId": "01JZ8G1FB5Z3N8GDRP0EBWSG4H",
    "nickname": "비활성 사용자",
    "profileImageUrl": null,
    "department": ""
  }
}
```

프론트는 빈 학과를 숨긴다.

## 프론트 Mock 상태

1. 예정 약속 1개
2. 지난 약속 목록
3. 취소 약속과 취소자 표시
4. 빈 예정 약속
5. 약속 취소 성공
6. 이미 취소·종료된 약속 `409`
7. 비활성 상대 약속
