# API 명세서

## 개요

콘서트 예약 서비스의 REST API 명세서입니다. 모든 API는 JWT 토큰 기반 인증을 사용하며, 일관된 응답 형식을 제공합니다.

**Base URL**: `https://api.concert-booking.com/v1`

## 인증 및 권한

- 토큰 발급 API를 제외한 모든 API는 JWT 토큰 필요
- Header: `Authorization: Bearer {token}`
- JWT에서 `userId` 자동 추출로 별도 전송 불필요

---

## 🎫 대기열 관리

### 유저 대기열 토큰 발급

사용자를 대기열에 등록하고 대기열 정보가 포함된 토큰을 발급합니다.

```http
POST /queue/tokens
```

**Request Body**

```json
{
  "userId": "user-uuid-123",
  "concertId": "concert-123"
}
```

**Response**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "userId": "user-uuid-123",
  "queuePosition": 1247,
  "estimatedWaitTimeSeconds": 900,
  "status": "WAITING"
}
```

**토큰 구조 (JWT Payload)**

```json
{
  "sub": "user-uuid-123",
  "queueId": "queue-20241220-001",
  "position": 1247,
  "estimatedWaitTime": 900,
  "status": "WAITING",
  "iat": 1640000000,
  "exp": 1640086400
}
```

> **중요**: 이후 모든 API는 이 토큰을 통한 대기열 검증을 통과해야만 이용 가능합니다.

### 대기열 상태 조회

현재 대기열 위치와 상태를 조회합니다.

```http
GET /queue/status
Authorization: Bearer {token}
```

**Response**

```json
{
  "queue": {
    "position": 1240,
    "estimatedWaitTimeSeconds": 780,
    "status": "WAITING",
    "activeUntil": null
  }
}
```

**상태별 응답 예시**

**ACTIVE 상태**

```json
{
  "queue": {
    "position": 0,
    "estimatedWaitTimeSeconds": 0,
    "status": "ACTIVE",
    "activeUntil": "2024-12-20T11:00:00Z"
  }
}
```

**Status Values**

- `WAITING`: 대기 중
- `ACTIVE`: 예약 가능
- `EXPIRED`: 만료됨

---

## 🎵 콘서트 관리

### 예약 가능 날짜 조회 🔹 기본

예약 가능한 콘서트 날짜 목록을 조회합니다.

```http
GET /concerts/dates
Authorization: Bearer {token}
```

> **대기열 검증**: `ACTIVE` 상태 토큰만 접근 가능

**Response**

```json
{
  "availableDates": [
    {
      "concertId": "concert-123",
      "date": "2024-12-25",
      "title": "Christmas Special Concert",
      "venue": "Olympic Hall",
      "totalSeats": 50,
      "availableSeats": 35,
      "price": 100000
    }
  ]
}
```

### 특정 콘서트 조회

콘서트 상세 정보를 조회합니다.

```http
GET /concerts/{concertId}
Authorization: Bearer {token}
```

**Response**

```json
{
  "concert": {
    "id": "concert-123",
    "title": "Christmas Special Concert",
    "artist": "IU",
    "venue": "Olympic Hall",
    "date": "2024-12-25",
    "time": "19:00",
    "price": 100000,
    "totalSeats": 50,
    "availableSeats": 35,
    "bookingPeriod": {
      "startTime": "2024-12-20T10:00:00Z",
      "endTime": "2024-12-25T18:00:00Z"
    },
    "status": "BOOKING_OPEN"
  }
}
```

### 예약 가능 좌석 조회 🔹 기본

특정 날짜(콘서트)의 예약 가능한 좌석 정보를 조회합니다.

```http
GET /concerts/{concertId}/seats
Authorization: Bearer {token}
```

> **대기열 검증**: `ACTIVE` 상태 토큰만 접근 가능  
> **좌석 관리**: 1~50번 좌석으로 고정 관리

**Response**

```json
{
  "concertId": "concert-123",
  "date": "2024-12-25",
  "title": "Christmas Special Concert",
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
```

**좌석 상태**

- `AVAILABLE`: 예약 가능
- `TEMPORARILY_RESERVED`: 임시 예약됨 (5분간)
- `RESERVED`: 결제 완료로 확정 예약됨

---

## 🎟️ 좌석 예약 요청

### 좌석 예약 요청

날짜와 좌석 정보를 입력받아 좌석을 예약 처리합니다.

```http
POST /reservations
Authorization: Bearer {token}
```

> **핵심 제약사항**:
>
> - 좌석 예약과 동시에 **5분간 임시 배정**
> - 5분 내 결제 미완료 시 **자동 해제**
> - 임시 배정 중에는 **다른 사용자 예약 불가**

**Request Body**

```json
{
  "concertId": "concert-123",
  "seatNumber": 15
}
```

**Response (성공)**

```json
{
  "reservation": {
    "id": "reservation-456",
    "userId": "user-uuid-123",
    "concertId": "concert-123",
    "seatNumber": 15,
    "status": "TEMPORARILY_RESERVED",
    "price": 100000,
    "reservedAt": "2024-12-20T10:00:00Z",
    "reservedUntil": "2024-12-20T10:05:00Z"
  }
}
```

**동시성 제어**

- Redis 분산 락으로 좌석별 동시 접근 방지
- 락 획득 실패 시 즉시 `409 SEAT_ALREADY_RESERVED` 반환

### 예약 목록 조회

사용자의 예약 내역을 조회합니다.

```http
GET /reservations
Authorization: Bearer {token}
```

**Query Parameters**

- `status`: 예약 상태 필터 (optional)
- `page`: 페이지 번호 (default: 1)
- `limit`: 페이지 크기 (default: 10)

**Response**

```json
{
  "reservations": [
    {
      "id": "reservation-456",
      "concert": {
        "id": "concert-123",
        "title": "Christmas Special Concert",
        "date": "2024-12-25",
        "venue": "Olympic Hall"
      },
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
    "hasNext": false
  }
}
```

### 예약 취소

임시 예약을 취소합니다.

```http
DELETE /reservations/{reservationId}
Authorization: Bearer {token}
```

**Response**

```json
{
  "reservation": {
    "id": "reservation-456",
    "status": "CANCELLED",
    "cancelledAt": "2024-12-20T10:03:00Z"
  }
}
```

---

## 💰 잔액 관리

### 잔액 조회

현재 사용자의 잔액을 조회합니다.

```http
GET /users/me/balance
Authorization: Bearer {token}
```

**Response**

```json
{
  "balance": {
    "amount": 150000,
    "lastUpdated": "2024-12-20T09:30:00Z"
  }
}
```

### 잔액 충전

사용자 잔액을 충전합니다.

```http
POST /users/me/balance/charges
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
  "transaction": {
    "id": "charge-789",
    "type": "CHARGE",
    "amount": 50000,
    "previousBalance": 150000,
    "currentBalance": 200000,
    "processedAt": "2024-12-20T10:00:00Z"
  }
}
```

**Validation Rules**

- 최소 충전액: 1,000원
- 최대 충전액: 1,000,000원
- 최대 보유 잔액: 5,000,000원

### 거래 내역 조회

잔액 거래 내역을 조회합니다.

```http
GET /users/me/balance/transactions
Authorization: Bearer {token}
```

**Query Parameters**

- `type`: 거래 유형 필터 (`CHARGE`, `PAYMENT`, `REFUND`)
- `page`: 페이지 번호 (default: 1)
- `limit`: 페이지 크기 (default: 10)

**Response**

```json
{
  "transactions": [
    {
      "id": "txn-123",
      "type": "PAYMENT",
      "amount": 100000,
      "previousBalance": 200000,
      "newBalance": 100000,
      "description": "콘서트 티켓 결제",
      "createdAt": "2024-12-20T10:02:30Z"
    },
    {
      "id": "txn-122",
      "type": "CHARGE",
      "amount": 50000,
      "previousBalance": 150000,
      "newBalance": 200000,
      "description": "잔액 충전",
      "createdAt": "2024-12-20T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 15,
    "hasNext": true
  }
}
```

---

## 💳 결제 처리

### 결제 API

임시 예약된 좌석에 대해 결제를 진행합니다.

```http
POST /payments
Authorization: Bearer {token}
```

> **핵심 로직**:
>
> - 결제 완료 시 좌석 소유권 유저에게 배정
> - **대기열 토큰 자동 만료 처리**
> - 미리 충전한 잔액으로만 결제 가능

**Request Body**

```json
{
  "reservationId": "reservation-456"
}
```

**Response (성공)**

```json
{
  "payment": {
    "id": "payment-101",
    "userId": "user-uuid-123",
    "reservationId": "reservation-456",
    "amount": 100000,
    "status": "COMPLETED",
    "method": "BALANCE",
    "balance": {
      "previousAmount": 200000,
      "remainingAmount": 100000
    },
    "paidAt": "2024-12-20T10:02:30Z"
  },
  "reservation": {
    "id": "reservation-456",
    "status": "CONFIRMED",
    "confirmedAt": "2024-12-20T10:02:30Z"
  },
  "tokenStatus": "EXPIRED"
}
```

**결제 완료 후 처리**

1. 좌석 상태: `TEMPORARILY_RESERVED` → `RESERVED`
2. 예약 상태: `TEMPORARILY_RESERVED` → `CONFIRMED`
3. 대기열 토큰: `ACTIVE` → `EXPIRED`
4. 사용자 잔액 차감 및 거래 내역 생성

### 결제 내역 조회

사용자의 결제 내역을 조회합니다.

```http
GET /payments
Authorization: Bearer {token}
```

**Query Parameters**

- `status`: 결제 상태 필터 (optional)
- `page`: 페이지 번호 (default: 1)
- `limit`: 페이지 크기 (default: 10)

**Response**

```json
{
  "payments": [
    {
      "id": "payment-101",
      "reservation": {
        "id": "reservation-456",
        "concertTitle": "Christmas Special Concert",
        "seatNumber": 15,
        "date": "2024-12-25"
      },
      "amount": 100000,
      "status": "COMPLETED",
      "method": "BALANCE",
      "paidAt": "2024-12-20T10:02:30Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 3,
    "hasNext": false
  }
}
```

---

## 📊 공통 응답 형식

### 성공 응답

단일 리소스:

```json
{
  "resourceName": { ... }
}
```

컬렉션 리소스:

```json
{
  "resourceNames": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 25,
    "hasNext": true
  }
}
```

### 에러 응답

```json
{
  "error": {
    "code": "SEAT_ALREADY_RESERVED",
    "message": "해당 좌석은 이미 예약되었습니다.",
    "details": {
      "seatNumber": 15,
      "concertId": "concert-123"
    }
  }
}
```

---

## ⚠️ 에러 코드

### 인증/권한 오류 (4xx)

| HTTP Status | 코드               | 메시지                            |
| ----------- | ------------------ | --------------------------------- |
| 401         | `INVALID_TOKEN`    | 유효하지 않은 토큰입니다.         |
| 401         | `TOKEN_EXPIRED`    | 토큰이 만료되었습니다.            |
| 403         | `QUEUE_NOT_ACTIVE` | 아직 예약 가능한 순서가 아닙니다. |
| 403         | `ACCESS_DENIED`    | 접근 권한이 없습니다.             |

### 클라이언트 오류 (4xx)

| HTTP Status | 코드                    | 메시지                           |
| ----------- | ----------------------- | -------------------------------- |
| 400         | `INVALID_REQUEST`       | 잘못된 요청입니다.               |
| 400         | `INVALID_AMOUNT`        | 유효하지 않은 금액입니다.        |
| 400         | `INSUFFICIENT_BALANCE`  | 잔액이 부족합니다.               |
| 404         | `CONCERT_NOT_FOUND`     | 존재하지 않는 콘서트입니다.      |
| 404         | `RESERVATION_NOT_FOUND` | 존재하지 않는 예약입니다.        |
| 404         | `SEAT_NOT_FOUND`        | 존재하지 않는 좌석입니다.        |
| 409         | `SEAT_ALREADY_RESERVED` | 해당 좌석은 이미 예약되었습니다. |
| 409         | `DUPLICATE_RESERVATION` | 이미 진행 중인 예약이 있습니다.  |
| 410         | `RESERVATION_EXPIRED`   | 예약 시간이 만료되었습니다.      |
| 422         | `BOOKING_NOT_OPEN`      | 예매 가능한 시간이 아닙니다.     |
| 429         | `RATE_LIMIT_EXCEEDED`   | 요청 제한을 초과했습니다.        |

### 서버 오류 (5xx)

| HTTP Status | 코드                    | 메시지                                  |
| ----------- | ----------------------- | --------------------------------------- |
| 500         | `INTERNAL_ERROR`        | 서버 내부 오류가 발생했습니다.          |
| 502         | `PAYMENT_GATEWAY_ERROR` | 결제 처리 중 오류가 발생했습니다.       |
| 503         | `SERVICE_UNAVAILABLE`   | 서비스를 일시적으로 사용할 수 없습니다. |

---

## 🔒 Rate Limiting

API 호출 제한을 통해 서비스 안정성을 보장합니다.

### 제한 정책

| API 그룹         | 제한    | 기준    |
| ---------------- | ------- | ------- |
| 토큰 발급        | 5회/분  | IP 주소 |
| 대기열 상태 조회 | 12회/분 | 사용자  |
| 예약 관련        | 30회/분 | 사용자  |
| 기타 API         | 60회/분 | 사용자  |

### Rate Limit 헤더

```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 59
X-RateLimit-Reset: 1640000000
```

### Rate Limit 초과 시 응답

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "요청 제한을 초과했습니다.",
    "details": {
      "limit": 60,
      "resetTime": "2024-12-20T10:01:00Z"
    }
  }
}
```
