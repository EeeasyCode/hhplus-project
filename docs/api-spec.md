# API 명세서

## 개요

콘서트 예약 서비스의 REST API 명세서입니다. 모든 API는 대기열 토큰 검증을 통과해야 사용 가능합니다. (토큰 발급 API 제외)

## 인증 및 권한

- 모든 API 요청 시 Header에 `Authorization: Bearer {token}` 포함 필요
- 토큰은 유저 대기열 토큰 발급 API를 통해 획득

---

## 1️⃣ 유저 대기열 토큰 관리 API

### 1.1 대기열 토큰 발급

사용자를 대기열에 등록하고 토큰을 발급합니다. <br>

<i>현재는 userId를 request Body Payload에 포함하여 발급 -> 추후 request authorization header에 토큰을 사용하도록 수정할 예정</i>

```http
POST /api/v1/queue/token
```

**Request Body**

```json
{
  "userId": "user-uuid-123"
}
```

**Response**

```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "queuePosition": 1247,
    "estimatedWaitTime": 900,
    "status": "WAITING"
  }
}
```

**Response Fields**

- `token`: 대기열 토큰 (JWT)
- `queuePosition`: 현재 대기 순서
- `estimatedWaitTime`: 예상 대기 시간(초)
- `status`: 대기 상태 (`WAITING`, `ACTIVE`, `EXPIRED`)

**JWT 토큰 구조**

대기열 토큰은 JWT 형식으로 다음과 같은 **최소한의 고정 정보**만 포함합니다.

**JWT Payload Fields**

- `iss/sub/aud/iat/exp`: 표준 JWT 필드
- `queueId`: 대기열 고유 ID
- `userId`: 사용자 고유 ID
- `issuedAt`: 토큰 발급 시간

**사용 flow**

1. 토큰 발급 시: API 응답으로 초기 대기열 정보 확인
2. 이후 상태 확인: 토큰으로 인증 + 상태 조회 API로 최신 정보 획득
3. 예약 시스템 이용: 토큰으로 인증 + 상태 조회 API로 `ACTIVE` 상태 확인 후 진행

**서버 측 토큰 처리**

- 모든 API에서 JWT 토큰을 파싱하여 `userId` 자동 추출
- 클라이언트는 `userId`를 별도로 전송할 필요 없음
- 토큰 검증과 동시에 사용자 식별 완료

### 1.2 대기열 상태 조회

현재 대기열 상태를 조회합니다. <br>

<i>현재는 polling 기반 -> 추후 sse 기반으로 수정할 예정</i>

```http
GET /api/v1/queue/status
```

**Headers**

```
Authorization: Bearer {token}
```

**Response**

```json
{
  "success": true,
  "data": {
    "queuePosition": 1240,
    "estimatedWaitTime": 780,
    "status": "WAITING",
    "activeUntil": null
  }
}
```

**Response Fields**

- `queuePosition`: 현재 대기 순서
- `estimatedWaitTime`: 예상 대기 시간(초)
- `status`: 대기 상태 (`WAITING`, `ACTIVE`, `EXPIRED`)
- `activeUntil`: 활성 상태 만료 시간 (ACTIVE 상태일 때만 값 존재)

---

## 2️⃣ 예약 가능 날짜 / 좌석 API

### 2.1 예약 가능 날짜 조회

예약 가능한 콘서트 날짜 목록을 조회합니다.

```http
GET /api/v1/concerts/dates
```

**Headers**

```
Authorization: Bearer {token}
```

**Response**

```json
{
  "success": true,
  "data": [
    {
      "concertId": "concert-123",
      "date": "2024-12-25",
      "title": "Christmas Special Concert",
      "venue": "Olympic Hall",
      "totalSeats": 50,
      "availableSeats": 35,
      "price": 100000
    },
    {
      "concertId": "concert-124",
      "date": "2024-12-31",
      "title": "New Year Concert",
      "venue": "Olympic Hall",
      "totalSeats": 50,
      "availableSeats": 42,
      "price": 100000
    }
  ]
}
```

**Response Fields**

- `concertId`: 콘서트 고유 ID
- `date`: 콘서트 날짜 (YYYY-MM-DD 형식)
- `title`: 콘서트 제목
- `venue`: 공연장 이름
- `totalSeats`: 전체 좌석 수
- `availableSeats`: 예약 가능한 좌석 수
- `price`: 좌석 가격 (원)

### 2.2 예약 가능 좌석 조회

특정 날짜의 예약 가능한 좌석 정보를 조회합니다.

```http
GET /api/v1/concerts/{concertId}/seats
```

**Headers**

```
Authorization: Bearer {token}
```

**Path Parameters**

- `concertId`: 콘서트 ID

**Response**

```json
{
  "success": true,
  "data": {
    "concertId": "concert-123",
    "date": "2024-12-25",
    "seats": [
      {
        "seatNumber": 1,
        "status": "AVAILABLE"
      },
      {
        "seatNumber": 2,
        "status": "RESERVED"
      },
      {
        "seatNumber": 3,
        "status": "TEMPORARILY_RESERVED",
        "tempReservedUntil": "2024-12-20T10:05:00Z"
      }
    ]
  }
}
```

**Response Fields**

- `concertId`: 콘서트 고유 ID
- `date`: 콘서트 날짜 (YYYY-MM-DD 형식)
- `seats`: 좌석 목록 배열
  - `seatNumber`: 좌석 번호 (1-50)
  - `status`: 좌석 상태
  - `tempReservedUntil`: 임시 예약 만료 시간 (임시 예약 상태일 때만 존재)

**Seat Status**

- `AVAILABLE`: 예약 가능
- `RESERVED`: 예약 완료
- `TEMPORARILY_RESERVED`: 임시 예약 (5분간)

---

## 3️⃣ 좌석 예약 요청 API

### 3.1 좌석 예약 요청

특정 좌석을 임시 예약합니다. (토큰에서 userId 자동 추출)

```http
POST /api/v1/concerts/{concertId}/seats/{seatNumber}/reserve
```

**Headers**

```
Authorization: Bearer {token}
```

**Path Parameters**

- `concertId`: 콘서트 ID
- `seatNumber`: 좌석 번호 (1-50)

**Request Body**

```json
{}
```

**Response (성공)**

```json
{
  "success": true,
  "data": {
    "reservationId": "reservation-456",
    "concertId": "concert-123",
    "seatNumber": 15,
    "status": "TEMPORARILY_RESERVED",
    "reservedUntil": "2024-12-20T10:05:00Z",
    "price": 100000
  }
}
```

**Response Fields**

- `reservationId`: 예약 고유 ID
- `concertId`: 콘서트 고유 ID
- `seatNumber`: 예약된 좌석 번호
- `status`: 예약 상태 (`TEMPORARILY_RESERVED`)
- `reservedUntil`: 임시 예약 만료 시간 (ISO 8601 형식)
- `price`: 좌석 가격 (원)

**Response (실패 - 이미 예약된 좌석)**

```json
{
  "success": false,
  "error": {
    "code": "SEAT_ALREADY_RESERVED",
    "message": "해당 좌석은 이미 예약되었습니다."
  }
}
```

---

## 4️⃣ 잔액 충전 / 조회 API

### 4.1 잔액 조회

사용자의 현재 잔액을 조회합니다. (토큰에서 userId 자동 추출)

```http
GET /api/v1/me/balance
```

**Headers**

```
Authorization: Bearer {token}
```

**Response**

```json
{
  "success": true,
  "data": {
    "userId": "user-uuid-123",
    "balance": 150000,
    "lastUpdated": "2024-12-20T09:30:00Z"
  }
}
```

**Response Fields**

- `userId`: 사용자 고유 ID
- `balance`: 현재 잔액 (원)
- `lastUpdated`: 마지막 업데이트 시간 (ISO 8601 형식)

### 4.2 잔액 충전

사용자의 잔액을 충전합니다. (토큰에서 userId 자동 추출)

```http
POST /api/v1/me/balance/charge
```

**Headers**

```
Authorization: Bearer {token}
```

**Request Body**

```json
{
  "amount": 50000
}
```

**Response**

```json
{
  "success": true,
  "data": {
    "userId": "user-uuid-123",
    "previousBalance": 150000,
    "chargedAmount": 50000,
    "currentBalance": 200000,
    "transactionId": "charge-789"
  }
}
```

**Response Fields**

- `userId`: 사용자 고유 ID
- `previousBalance`: 충전 이전 잔액 (원)
- `chargedAmount`: 충전 금액 (원)
- `currentBalance`: 충전 후 현재 잔액 (원)
- `transactionId`: 충전 거래 고유 ID

**Validation**

- 충전 금액: 1,000원 ~ 1,000,000원
- 최대 잔액: 5,000,000원

---

## 5️⃣ 결제 API

### 5.1 결제 처리

임시 예약된 좌석에 대한 결제를 처리합니다. (토큰에서 userId 자동 추출)

```http
POST /api/v1/payments
```

**Headers**

```
Authorization: Bearer {token}
```

**Request Body**

```json
{
  "reservationId": "reservation-456"
}
```

**Response (성공)**

```json
{
  "success": true,
  "data": {
    "paymentId": "payment-101",
    "reservationId": "reservation-456",
    "amount": 100000,
    "previousBalance": 200000,
    "remainingBalance": 100000,
    "status": "COMPLETED",
    "paidAt": "2024-12-20T10:02:30Z"
  }
}
```

**Response Fields**

- `paymentId`: 결제 고유 ID
- `reservationId`: 예약 고유 ID
- `amount`: 결제 금액 (원)
- `previousBalance`: 결제 이전 잔액 (원)
- `remainingBalance`: 결제 후 잔액 (원)
- `status`: 결제 상태 (`COMPLETED`)
- `paidAt`: 결제 완료 시간 (ISO 8601 형식)

**Response (실패 - 잔액 부족)**

```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "잔액이 부족합니다.",
    "details": {
      "requiredAmount": 100000,
      "currentBalance": 50000
    }
  }
}
```

---

## 6️⃣ 추가 관리 API

### 6.1 예약 내역 조회

사용자의 예약 내역을 조회합니다. (토큰에서 userId 자동 추출)

```http
GET /api/v1/me/reservations
```

**Headers**

```
Authorization: Bearer {token}
```

**Query Parameters**

- `status`: 예약 상태 필터 (optional)
- `page`: 페이지 번호 (default: 1)
- `limit`: 페이지 크기 (default: 10)

**Response**

```json
{
  "success": true,
  "data": {
    "reservations": [
      {
        "reservationId": "reservation-456",
        "concertId": "concert-123",
        "concertTitle": "Christmas Special Concert",
        "date": "2024-12-25",
        "seatNumber": 15,
        "status": "CONFIRMED",
        "price": 100000,
        "reservedAt": "2024-12-20T10:00:00Z",
        "confirmedAt": "2024-12-20T10:02:30Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 5,
      "totalPages": 1
    }
  }
}
```

**Response Fields**

- `reservations`: 예약 목록 배열
  - `reservationId`: 예약 고유 ID
  - `concertId`: 콘서트 고유 ID
  - `concertTitle`: 콘서트 제목
  - `date`: 콘서트 날짜 (YYYY-MM-DD 형식)
  - `seatNumber`: 좌석 번호
  - `status`: 예약 상태 (`CONFIRMED`, `TEMPORARILY_RESERVED`, `CANCELLED`, `EXPIRED`)
  - `price`: 좌석 가격 (원)
  - `reservedAt`: 예약 생성 시간 (ISO 8601 형식)
  - `confirmedAt`: 예약 확정 시간 (확정된 경우에만 존재, ISO 8601 형식)
- `pagination`: 페이지네이션 정보
  - `page`: 현재 페이지 번호
  - `limit`: 페이지당 항목 수
  - `total`: 전체 예약 건수
  - `totalPages`: 전체 페이지 수

### 6.2 임시 예약 취소

임시 예약된 좌석을 수동으로 취소합니다.

```http
DELETE /api/v1/reservations/{reservationId}
```

**Headers**

```
Authorization: Bearer {token}
```

**Response**

```json
{
  "success": true,
  "data": {
    "reservationId": "reservation-456",
    "status": "CANCELLED",
    "cancelledAt": "2024-12-20T10:03:00Z"
  }
}
```

**Response Fields**

- `reservationId`: 취소된 예약의 고유 ID
- `status`: 변경된 예약 상태 (`CANCELLED`)
- `cancelledAt`: 취소 처리 시간 (ISO 8601 형식)

---

## 공통 응답 형식

### 성공 응답

```json
{
  "success": true,
  "data": { ... }
}
```

### 오류 응답

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "오류 메시지",
    "details": { ... }
  }
}
```

## 주요 에러 코드

| 코드                    | HTTP Status | 메시지                            |
| ----------------------- | ----------- | --------------------------------- |
| `INVALID_TOKEN`         | 401         | 유효하지 않은 토큰입니다.         |
| `TOKEN_EXPIRED`         | 401         | 토큰이 만료되었습니다.            |
| `QUEUE_NOT_ACTIVE`      | 403         | 아직 예약 가능한 순서가 아닙니다. |
| `SEAT_ALREADY_RESERVED` | 409         | 해당 좌석은 이미 예약되었습니다.  |
| `INSUFFICIENT_BALANCE`  | 400         | 잔액이 부족합니다.                |
| `RESERVATION_EXPIRED`   | 410         | 예약 시간이 만료되었습니다.       |
| `INVALID_SEAT_NUMBER`   | 400         | 유효하지 않은 좌석 번호입니다.    |
| `CONCERT_NOT_FOUND`     | 404         | 존재하지 않는 콘서트입니다.       |
| `RESERVATION_NOT_FOUND` | 404         | 존재하지 않는 예약입니다.         |

## Rate Limiting

- 대기열 토큰 발급: 사용자당 1회/분
- 대기열 상태 조회: 사용자당 30회/분
- 기타 API: 사용자당 60회/분
