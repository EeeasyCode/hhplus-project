# 서비스 플로우 차트

## 개요

콘서트 예약 서비스의 주요 유즈케이스별 클라이언트-서버 상호작용 플로우를 정의합니다.

---

## 주요 유즈케이스

### 1. 대기열 진입 및 대기 플로우

### 2. 콘서트 예약 플로우

### 3. 잔액 충전 플로우

### 4. 결제 및 완료 플로우

---

## 1. 대기열 진입 및 대기 플로우

```mermaid
sequenceDiagram
    participant User as User
    participant Client as Client
    participant Server as Server
    participant Redis as Redis
    participant DB as PostgreSQL

    User->>Client: 콘서트 예약 서비스 접속
    Client->>Server: GET /api/v1/concerts/available<br/>Authorization: Bearer {userToken}
    Server->>DB: SELECT * FROM concerts WHERE booking_start_time <= NOW() AND booking_end_time >= NOW()
    Server-->>Client: 예약 가능한 콘서트 목록
    Client-->>User: 콘서트 목록 화면 표시

    User->>Client: "IU 콘서트" 선택
    Client->>Server: POST /api/v1/queue/token<br/>{userId: "user-123", concertId: "concert-iu-001"}

    Server->>Redis: ZADD queue:waiting:{concertId} {timestamp} {userId}
    Server->>Redis: HSET user:{userId}:queue position status queueId
    Server-->>Client: 200 OK<br/>{token, queuePosition: 1247, estimatedWaitTime: 900, status: "WAITING"}<br/>Note: estimatedWaitTime = position * 30초 (서버에서 계산)

    Client-->>User: "IU 콘서트" 대기열 화면 표시<br/>"현재 1247번째, 예상 대기시간 15분"

    loop 대기열 상태 확인 (30초마다)
        Client->>Server: GET /api/v1/queue/status<br/>Authorization: Bearer {token}
        Server->>Redis: HGET user:{userId}:queue position status

        alt 아직 대기 중
            Server-->>Client: {queuePosition: 1200, estimatedWaitTime: 600, status: "WAITING"}<br/>Note: estimatedWaitTime = position * 30초 계산
            Client-->>User: 대기 순서 업데이트
        else 활성화됨
            Server-->>Client: {queuePosition: 0, status: "ACTIVE", activeUntil: "..."}
            Client-->>User: "예약 가능! IU 콘서트 좌석 선택 화면으로 이동"
            Note over Client,User: 대기열 polling 종료
        end
    end
```

---

## 2. 콘서트 예약 플로우

```mermaid
sequenceDiagram
    participant User as User
    participant Client as Client
    participant Server as Server
    participant Redis as Redis
    participant DB as PostgreSQL

    Note over User,DB: "IU 콘서트" 대기열 통과 후 ACTIVE 상태

    User->>Client: 좌석 선택 화면 진입
    Client->>Server: GET /api/v1/concerts/concert-iu-001/seats<br/>Authorization: Bearer {token}
    Server->>DB: SELECT seat_number,<br/>CASE WHEN status='AVAILABLE' THEN 'AVAILABLE'<br/>WHEN status='TEMPORARILY_RESERVED' AND temp_reserved_until < NOW()<br/>THEN 'AVAILABLE' ELSE status END as actual_status<br/>FROM seats WHERE concert_id = ?
    Server-->>Client: 실시간 좌석 상태 응답
    Client-->>User: 좌석 선택 화면 표시

    User->>Client: 좌석 15번 선택
    Client->>Server: POST /api/v1/concerts/{concertId}/seats/15/reserve<br/>Authorization: Bearer {token}

    Server->>Redis: 분산 락 획득 시도 (seat:15)
    Server->>DB: BEGIN TRANSACTION
    Server->>DB: UPDATE seats SET status='TEMPORARILY_RESERVED', temp_reserved_by=userId<br/>WHERE id=15 AND (status='AVAILABLE' OR<br/>(status='TEMPORARILY_RESERVED' AND temp_reserved_until < NOW()))
    alt affected_rows = 0
        Server->>DB: ROLLBACK
        Server->>Redis: 락 해제
        Server-->>Client: 409 Conflict - "이미 예약된 좌석입니다"
    else 예약 성공
        Server->>DB: INSERT INTO reservations (...)
        Server->>DB: COMMIT
        Server->>Redis: 락 해제
        Server-->>Client: 예약 성공<br/>{reservationId, status: "TEMPORARILY_RESERVED", reservedUntil: "5분후"}
    end
    Client-->>User: "좌석 임시 예약 완료!<br/>5분 내 결제해주세요"
```

---

## 3. 잔액 충전 플로우

```mermaid
sequenceDiagram
    participant User as User
    participant Client as Client
    participant Server as Server
    participant DB as PostgreSQL

    User->>Client: 잔액 충전 화면 이동
    Client->>Server: GET /api/v1/me/balance<br/>Authorization: Bearer {token}
    Server->>DB: SELECT balance FROM users WHERE id=userId
    Server-->>Client: {balance: 50000}
    Client-->>User: "현재 잔액: 50,000원"

    User->>Client: 100,000원 충전 요청
    Client->>Server: POST /api/v1/me/balance/charge<br/>{amount: 100000}<br/>Authorization: Bearer {token}

    Server->>DB: BEGIN TRANSACTION
    Server->>DB: SELECT balance FROM users WHERE id=userId FOR UPDATE
    Server->>DB: UPDATE users SET balance = balance + 100000
    Server->>DB: INSERT INTO balance_transactions (type='CHARGE', ...)
    Server->>DB: COMMIT

    Server-->>Client: 충전 성공<br/>{previousBalance: 50000, currentBalance: 150000}
    Client-->>User: "충전 완료!<br/>현재 잔액: 150,000원"
```

---

## 4. 결제 및 완료 플로우

```mermaid
sequenceDiagram
    participant User as User
    participant Client as Client
    participant Server as Server
    participant DB as PostgreSQL
    participant Redis as Redis

    Note over User,Redis: 임시 예약 상태에서 결제 진행

    User->>Client: 결제하기 버튼 클릭
    Client->>Server: POST /api/v1/payments<br/>{reservationId: "res-456"}<br/>Authorization: Bearer {token}

    Server->>DB: BEGIN TRANSACTION

    # 예약 상태 확인
    Server->>DB: SELECT * FROM reservations WHERE id='res-456'
    alt 예약 만료됨
        Server-->>Client: 410 Gone - "예약 시간이 만료되었습니다"
        Client-->>User: 예약 만료 알림
    else 예약 유효
        # 잔액 확인 및 차감
        Server->>DB: SELECT balance FROM users WHERE id=userId FOR UPDATE
        alt 잔액 부족
            Server-->>Client: 400 Bad Request - "잔액이 부족합니다"
            Client-->>User: 잔액 부족 알림 + 충전 유도
        else 잔액 충분
            # 결제 처리
            Server->>DB: UPDATE users SET balance = balance - amount
            Server->>DB: UPDATE reservations SET status='CONFIRMED', confirmed_at=NOW()
            Server->>DB: UPDATE seats SET status='RESERVED'
            Server->>DB: INSERT INTO payments (status='COMPLETED', ...)
            Server->>DB: INSERT INTO balance_transactions (type='PAYMENT', ...)

            # 캐시 업데이트
            Server->>Redis: DECR cache:concert:{concertId}:available_seats

            Server->>DB: COMMIT

            Server-->>Client: 결제 성공<br/>{paymentId, amount, remainingBalance}
            Client-->>User: "결제 완료!<br/>예약이 확정되었습니다"

            # 대기열에서 제거
            Server->>Redis: SREM queue:active {userId}
            Server->>Redis: DEL user:{userId}:queue
        end
    end
```

---

## 예외 상황 플로우

### 4.1 임시 예약 만료 처리

```mermaid
sequenceDiagram
    participant User as User
    participant API as API Server
    participant DB as PostgreSQL
    participant Redis as Redis

    User->>API: 좌석 조회 요청
    API->>DB: SELECT seats WHERE concert_id = ?<br/>실시간 만료 체크 포함

    Note over DB: SELECT seat_number,<br/>CASE<br/>  WHEN status = 'AVAILABLE' THEN 'AVAILABLE'<br/>  WHEN status = 'TEMPORARILY_RESERVED'<br/>    AND temp_reserved_until < NOW()<br/>  THEN 'AVAILABLE'<br/>  ELSE status<br/>END as actual_status

    DB-->>API: 실시간 좌석 상태
    API-->>User: 정확한 좌석 정보

    User->>API: 좌석 예약 요청
    API->>Redis: 분산 락 획득 시도
    API->>DB: BEGIN TRANSACTION

    Note over DB: 실시간 만료 체크와 함께<br/>원자적 예약 처리

    API->>DB: UPDATE seats SET status='TEMPORARILY_RESERVED'<br/>WHERE status='AVAILABLE'<br/>OR (status='TEMPORARILY_RESERVED'<br/>AND temp_reserved_until < NOW())

    alt 예약 성공 (affected_rows > 0)
        API->>DB: INSERT INTO reservations
        API->>DB: COMMIT
        API->>Redis: 락 해제
        API-->>User: 예약 성공
    else 예약 실패 (이미 예약됨)
        API->>DB: ROLLBACK
        API->>Redis: 락 해제
        API-->>User: 409 - 이미 예약된 좌석
    end
```

### 4.2 대기열 상태 변경 처리

```mermaid
sequenceDiagram
    participant System as System
    participant Redis as Redis

    loop 10초마다 실행
        System->>Redis: SCARD queue:active (활성 사용자 수 확인)

        alt 활성 사용자 < 100명
            System->>Redis: ZPOPMIN queue:waiting 10 (대기열에서 10명 꺼내기)

            loop 새로 활성화된 사용자들
                System->>Redis: SADD queue:active {userId}
                System->>Redis: EXPIRE queue:active:{userId} 3600
                System->>Redis: HSET user:{userId}:queue status "ACTIVE"
                System->>Redis: HSET user:{userId}:queue activeUntil "1시간후"
            end
        end

        # 대기 순서 업데이트
        System->>Redis: ZCARD queue:waiting (대기 중인 사용자 수)
        System->>Redis: HSET queue:stats totalWaiting {count}

        loop 대기 중인 각 사용자
            System->>Redis: ZRANK queue:waiting {userId} (순서 조회)
            System->>Redis: HSET user:{userId}:queue position {rank+1}
            Note over System: estimatedWaitTime은 API 응답시 실시간 계산
        end
    end
```

---

## 클라이언트 상태 관리

### 주요 상태 전환

```mermaid
stateDiagram-v2
    [*] --> 서비스접속
    서비스접속 --> 콘서트목록조회: 이용 가능한 콘서트 확인
    콘서트목록조회 --> 콘서트선택: 특정 콘서트 선택
    콘서트선택 --> 대기열등록: 해당 콘서트 대기열 진입
    대기열등록 --> 대기중: WAITING 상태
    대기중 --> 대기중: 순서 업데이트 (polling)
    대기중 --> 예약가능: ACTIVE 상태로 변경
    예약가능 --> 좌석선택: 해당 콘서트 좌석 화면
    좌석선택 --> 임시예약: 좌석 예약 성공
    임시예약 --> 잔액충전: 잔액 부족시
    임시예약 --> 결제진행: 잔액 충분시
    잔액충전 --> 결제진행: 충전 완료
    결제진행 --> 예약완료: 결제 성공
    예약완료 --> [*]

    임시예약 --> 예약만료: 5분 경과
    예약만료 --> 좌석선택: 다시 시도
```

---
