# 점심 메이트 API 명세

> 버전: v1
> 기준 기능 명세: [`LUNCH_MATE_FUNCTIONAL_SPEC.md`](../LUNCH_MATE_FUNCTIONAL_SPEC.md)
> Base URL: `/api/v1`

## 목적

이 디렉터리는 프론트엔드와 백엔드가 독립적으로 구현할 수 있도록 HTTP 계약을 고정한다. 프론트엔드는 문서의 성공·오류 예시로 Mock API를 만들고, 백엔드는 동일한 경로·필드·상태 코드로 API를 구현한다.

## 계약 원칙

1. JSON 필드명은 `camelCase`, URL은 복수 명사와 `kebab-case`를 사용한다.
2. 모든 성공 응답은 `{ "data": ... }`, 모든 오류 응답은 `{ "error": ... }` 형식을 사용한다.
3. 문서에 `nullable`로 표시되지 않은 응답 필드는 항상 존재해야 한다.
4. 응답에 필드를 추가하는 것은 허용하지만 기존 필드 삭제·이름·타입 변경은 양쪽 합의 없이 금지한다.
5. 목록이 비어 있으면 `null`이 아니라 `[]`를 반환한다.
6. 프론트엔드는 HTTP 상태 코드와 `error.code`로 분기하고, `error.message` 문자열 비교에 의존하지 않는다.
7. 백엔드는 문서에 없는 enum 값을 임의로 반환하지 않는다.
8. AI·OCR API 실패는 규칙 기반 추천과 수동 시간표 등록을 막지 않아야 한다.

## 문서 목록

| 문서 | 담당 기능 | 주요 API |
|---|---|---|
| [00-common.md](./00-common.md) | 공통 규약 | 응답, 오류, 인증 헤더, 날짜·시간, 페이지네이션 |
| [01-auth.md](./01-auth.md) | 인증 | 회원가입, 로그인, 토큰 갱신, 로그아웃, 내 계정 |
| [02-reference-data.md](./02-reference-data.md) | 기준 데이터 | 학교, 캠퍼스, 프로필 선택지 |
| [03-profile.md](./03-profile.md) | 프로필 | 내 프로필 조회·저장·수정 |
| [04-schedules.md](./04-schedules.md) | 시간표 | 수업 CRUD, 공강 계산, OCR 선택 기능 |
| [05-availability.md](./05-availability.md) | 선호 시간 | 선호 시간·최소 만남 시간 조회·저장 |
| [06-matches.md](./06-matches.md) | 매칭 | 추천 목록, 추천 상세, 관심·넘기기 선택 기능 |
| [07-proposals.md](./07-proposals.md) | 만남 제안 | 제안 생성·조회·수락·거절·취소 |
| [08-appointments.md](./08-appointments.md) | 약속 | 약속 목록·상세·취소 |
| [09-safety.md](./09-safety.md) | 안전 | 차단·신고 선택 기능 |

## 엔드포인트 전체 목록

### 인증 불필요

| Method | Path | 기능 |
|---|---|---|
| `POST` | `/auth/sign-up` | 회원가입 |
| `POST` | `/auth/login` | 로그인 |
| `POST` | `/auth/refresh` | Access Token 갱신 |
| `GET` | `/schools` | 학교 목록 |
| `GET` | `/schools/{schoolId}/campuses` | 캠퍼스 목록 |
| `GET` | `/profile-options` | 프로필 enum·선택지 |

### 인증 필요

| Method | Path | 기능 |
|---|---|---|
| `POST` | `/auth/logout` | 로그아웃 |
| `GET` | `/users/me` | 내 계정 요약 |
| `GET` | `/users/me/profile` | 내 프로필 조회 |
| `PUT` | `/users/me/profile` | 최초 프로필 저장 또는 전체 교체 |
| `PATCH` | `/users/me/profile` | 프로필 일부 수정 |
| `GET` | `/users/me/classes` | 내 수업 목록 |
| `POST` | `/users/me/classes` | 수업 등록 |
| `PATCH` | `/users/me/classes/{classId}` | 수업 수정 |
| `DELETE` | `/users/me/classes/{classId}` | 수업 삭제 |
| `GET` | `/users/me/free-slots` | 계산된 공강 조회 |
| `POST` | `/users/me/schedule-imports` | OCR 분석 요청 `[선택]` |
| `POST` | `/users/me/schedule-imports/{importId}/confirm` | OCR 결과 확정 `[선택]` |
| `GET` | `/users/me/availability` | 선호 시간 조회 |
| `PUT` | `/users/me/availability` | 선호 시간 전체 저장 |
| `GET` | `/matches` | 추천 목록 |
| `GET` | `/matches/{userId}` | 추천 상대 상세 |
| `POST` | `/match-actions` | 관심·넘기기 `[선택]` |
| `GET` | `/proposals` | 보낸·받은 제안 목록 |
| `POST` | `/proposals` | 제안 생성 |
| `GET` | `/proposals/{proposalId}` | 제안 상세 |
| `POST` | `/proposals/{proposalId}/accept` | 제안 수락 |
| `POST` | `/proposals/{proposalId}/reject` | 제안 거절 |
| `POST` | `/proposals/{proposalId}/cancel` | 대기 제안 취소 |
| `GET` | `/appointments` | 약속 목록 |
| `GET` | `/appointments/{appointmentId}` | 약속 상세 |
| `POST` | `/appointments/{appointmentId}/cancel` | 확정 약속 취소 |
| `GET` | `/blocks` | 차단 목록 `[선택]` |
| `POST` | `/blocks` | 사용자 차단 `[선택]` |
| `DELETE` | `/blocks/{blockId}` | 차단 해제 `[선택]` |
| `POST` | `/reports` | 사용자 신고 `[선택]` |

## 프론트엔드 병렬 작업 기준

- API 클라이언트는 `/api/v1`을 공통 prefix로 설정한다.
- 인증 요청은 쿠키 전송을 위해 `credentials: "include"`를 사용한다.
- 보호 API에는 `Authorization: Bearer {accessToken}`을 전달한다.
- `401`과 `AUTH_TOKEN_EXPIRED`를 받으면 `/auth/refresh`를 한 번 호출한 후 원 요청을 한 번만 재시도한다.
- 페이지 컴포넌트는 `loading`, `success`, `empty`, `error` 상태를 각각 구현한다.
- 목록 Mock은 최소 `0개`, `1개`, `다음 cursor 있음` 세 경우를 준비한다.
- 상태 변경 버튼은 요청 중 중복 클릭을 막고, `409` 응답 시 최신 상세를 다시 조회한다.

## 백엔드 병렬 작업 기준

- 입력은 컨트롤러 경계에서 검증하고 서비스 계층에는 검증된 값만 전달한다.
- 사용자 소유 리소스는 인증 사용자 ID로 조회한다. URL 또는 Body의 임의 사용자 ID를 소유자로 신뢰하지 않는다.
- 생성 API는 `201 Created`와 `Location` 헤더를 반환한다.
- 삭제 성공은 Body 없이 `204 No Content`를 반환한다.
- 제안 수락·거절·취소는 트랜잭션으로 상태를 원자적으로 변경한다.
- 비밀번호, 토큰, 전체 시간표, 강의실을 추천·제안 응답이나 로그에 포함하지 않는다.

## 계약 변경 절차

API 계약을 변경해야 하면 다음 순서를 따른다.

1. 해당 Markdown 문서의 요청·응답·오류를 먼저 수정한다.
2. 프론트엔드와 백엔드 담당자가 변경 내용을 확인한다.
3. 기존 필드를 깨는 변경이면 새 필드 추가 후 기존 필드를 유지하거나 `/api/v2`로 분리한다.
4. 양쪽 구현과 Mock 데이터를 함께 갱신한다.
