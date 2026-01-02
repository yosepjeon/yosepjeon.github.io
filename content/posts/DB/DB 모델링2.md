+++
title = "DB 모델링 2"
date = "2024-03-27"
author = "Yosep"
tags = ["db", "modeling"]
description = "DB 모델링에 대해 학습 및 정리합니다."
weight = 3
+++

# PK(Primary Key)의 종류와 실무 활용

## 1. PK란?

**Primary Key(기본키)**는 테이블에서 각 행을 고유하게 식별하는 컬럼(또는 컬럼 조합)이다.

**PK의 특성:**
- **유일성**: 중복 값 불가
- **NOT NULL**: NULL 값 불가
- **불변성**: 한번 설정되면 변경하지 않는 것이 원칙
- **인덱스 생성**: 대부분의 DB에서 PK는 UNIQUE 인덱스를 자동 생성

---

## 2. PK의 종류

### 2.1 자연키 (Natural Key)
### 2.2 대리키 (Surrogate Key)
### 2.3 복합키 (Composite Key)

```
┌─────────────────────────────────────────────────────────────┐
│                        PK 종류                               │
├─────────────────┬─────────────────┬─────────────────────────┤
│    자연키         │     대리키       │        복합키             │
│  Natural Key    │  Surrogate Key  │    Composite Key        │
├─────────────────┼─────────────────┼─────────────────────────┤
│ 비즈니스 의미有     │ 비즈니스 의미無     │ 2개 이상 컬럼 조합          │
│ 주민번호, 이메일    │ AUTO_INCREMENT  │ (user_id, product_id)   │
│                 │ UUID            │                         │
└─────────────────┴─────────────────┴─────────────────────────┘
```

---

## 3. 자연키 (Natural Key)

### 3.1 개념
비즈니스적으로 의미가 있는 실제 데이터를 PK로 사용

**선정 기준(권장):**
- 표준 코드처럼 변경 가능성이 거의 없음
- 길이가 짧고 단순함 (인덱스 효율)
- 개인정보/민감정보가 아님

### 3.2 예시
```sql
-- 국가 코드 테이블 (ISO 표준 코드 사용)
CREATE TABLE country (
    country_code CHAR(2) PRIMARY KEY,  -- 'KR', 'US', 'JP'
    country_name VARCHAR(100) NOT NULL
);

-- 통화 테이블
CREATE TABLE currency (
    currency_code CHAR(3) PRIMARY KEY,  -- 'KRW', 'USD', 'JPY'
    currency_name VARCHAR(50) NOT NULL
);

-- 설정 테이블
CREATE TABLE system_config (
    config_key VARCHAR(100) PRIMARY KEY,  -- 'MAX_LOGIN_ATTEMPTS'
    config_value VARCHAR(500) NOT NULL
);
```

### 3.3 실무 사용 사례

**✅ 적합한 경우:**
```sql
-- 1. 국제 표준 코드
CREATE TABLE language (
    lang_code CHAR(2) PRIMARY KEY,  -- ISO 639-1: 'ko', 'en', 'ja'
    lang_name VARCHAR(50)
);

-- 2. 고정된 상태/타입 코드
CREATE TABLE order_status (
    status_code VARCHAR(20) PRIMARY KEY,  -- 'PENDING', 'SHIPPED', 'DELIVERED'
    status_name VARCHAR(50),
    display_order INT
);

-- 3. 설정/코드성 테이블
CREATE TABLE error_code (
    error_code VARCHAR(10) PRIMARY KEY,  -- 'E001', 'E002'
    error_message VARCHAR(500)
);
```

**❌ 부적합한 경우:**
```sql
-- 주민번호를 PK로? → 변경 가능성, 개인정보 보호 이슈
CREATE TABLE member (
    resident_number CHAR(13) PRIMARY KEY,  -- ❌ 위험!
    name VARCHAR(100)
);

-- 이메일을 PK로? → 사용자가 변경할 수 있음
CREATE TABLE member (
    email VARCHAR(255) PRIMARY KEY,  -- ❌ 변경 가능성
    name VARCHAR(100)
);
```

### 3.4 장단점

| 장점 | 단점 |
|------|------|
| 의미 파악 용이 | 값이 변경될 수 있음 |
| 추가 컬럼 불필요 | 복잡한 값은 조인 성능 저하 |
| 조회 시 직관적 | 개인정보 노출 위험 |

---

## 4. 대리키 (Surrogate Key)

### 4.1 개념
비즈니스와 무관한 인위적인 식별자를 PK로 사용

### 4.2 종류

#### 4.2.1 AUTO_INCREMENT (정수형 순차 증가)
```sql
-- MySQL
CREATE TABLE member (
    member_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- PostgreSQL
CREATE TABLE member (
    member_id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL
);

-- Oracle
CREATE TABLE member (
    member_id NUMBER PRIMARY KEY,
    email VARCHAR2(255) NOT NULL UNIQUE,
    name VARCHAR2(100) NOT NULL
);
-- + 시퀀스 사용
CREATE SEQUENCE member_seq START WITH 1 INCREMENT BY 1;
```

> 참고: PostgreSQL 10+는 `GENERATED AS IDENTITY`를 권장하며, Oracle 12c+는 IDENTITY 컬럼도 지원한다.

#### 4.2.2 UUID (Universally Unique Identifier)
```sql
-- UUID v4 (랜덤)
CREATE TABLE payment (
    payment_id CHAR(36) PRIMARY KEY,  -- 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11'
    order_id BIGINT NOT NULL,
    amount DECIMAL(15,2) NOT NULL
);

-- MySQL 8.0+에서 UUID 생성
INSERT INTO payment (payment_id, order_id, amount)
VALUES (UUID(), 12345, 50000.00);

-- PostgreSQL
CREATE TABLE payment (
    payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id BIGINT NOT NULL,
    amount DECIMAL(15,2) NOT NULL
);
```

> 참고: PostgreSQL의 `gen_random_uuid()`는 `pgcrypto` 확장이 필요하다.
> MySQL은 `UUID()`가 문자열을 반환하므로 대용량 테이블에서는 `BINARY(16)` 저장을 고려한다.

#### 4.2.3 ULID / Snowflake ID (시간순 정렬 가능)
```sql
-- ULID: 시간순 정렬 가능한 UUID 대안
-- 형식: 01ARZ3NDEKTSV4RRFFQ69G5FAV (26자)

CREATE TABLE event_log (
    event_id CHAR(26) PRIMARY KEY,  -- ULID
    event_type VARCHAR(50),
    created_at TIMESTAMP
);

-- Snowflake ID (Twitter 방식): 64bit 정수
-- 구조: [시간(41bit)][머신ID(10bit)][시퀀스(12bit)]
CREATE TABLE tweet (
    tweet_id BIGINT PRIMARY KEY,  -- Snowflake ID
    content VARCHAR(280),
    user_id BIGINT
);
```

#### 4.2.4 DB별 구현 참고 (IDENTITY/SEQUENCE/UUID)

| DB | 순차 ID | 시퀀스 | UUID 생성 | UUID 타입/저장 |
|----|---------|--------|-----------|----------------|
| MySQL | AUTO_INCREMENT | SEQUENCE(8.0+) | UUID() | CHAR(36) 또는 BINARY(16) |
| PostgreSQL | GENERATED AS IDENTITY / BIGSERIAL | 내부 SEQUENCE | gen_random_uuid() | UUID 타입 |
| Oracle | IDENTITY(12c+) | CREATE SEQUENCE | SYS_GUID() | RAW(16) 또는 CHAR(32) |
| SQL Server | IDENTITY | SEQUENCE | NEWID(), NEWSEQUENTIALID() | UNIQUEIDENTIFIER |

실무 팁:
- MySQL은 `UUID_TO_BIN()`/`BIN_TO_UUID()`로 저장 효율을 개선할 수 있다.
- PostgreSQL은 `pgcrypto` 확장을 활성화해야 UUID 함수를 사용할 수 있다.
- Oracle은 시퀀스 + 트리거 방식도 여전히 널리 사용된다.
- SQL Server는 `NEWSEQUENTIALID()`로 인덱스 단편화를 줄일 수 있다.

#### 4.2.5 DB별 SQL 예시 (IDENTITY/UUID 저장)

**MySQL (AUTO_INCREMENT + BINARY(16) UUID)**
```sql
CREATE TABLE member (
    member_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_uuid BINARY(16) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    UNIQUE KEY uk_member_uuid (member_uuid)
);

INSERT INTO member (member_uuid, email, name)
VALUES (UUID_TO_BIN(UUID()), 'a@example.com', 'Alice');

SELECT BIN_TO_UUID(member_uuid) AS member_uuid, email
FROM member;
```

**PostgreSQL (IDENTITY + UUID)**
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE member (
    member_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    member_uuid UUID NOT NULL DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL
);

INSERT INTO member (email, name) VALUES ('a@example.com', 'Alice');
```

**Oracle (IDENTITY + SYS_GUID)**
```sql
CREATE TABLE member (
    member_id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    member_uuid RAW(16) DEFAULT SYS_GUID() NOT NULL,
    email VARCHAR2(255) NOT NULL UNIQUE,
    name VARCHAR2(100) NOT NULL
);

SELECT RAWTOHEX(member_uuid) AS member_uuid, email FROM member;
```

**SQL Server (IDENTITY + UNIQUEIDENTIFIER)**
```sql
CREATE TABLE member (
    member_id BIGINT IDENTITY(1,1) PRIMARY KEY,
    member_uuid UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID(),
    email NVARCHAR(255) NOT NULL UNIQUE,
    name NVARCHAR(100) NOT NULL
);

INSERT INTO member (email, name) VALUES (N'a@example.com', N'Alice');
SELECT CONVERT(VARCHAR(36), member_uuid) AS member_uuid, email FROM member;
```

> 참고: UUID를 CHAR(36)로 저장하면 사람이 읽기 쉽지만 인덱스 크기가 커진다.

### 4.3 실무 사용 사례

**AUTO_INCREMENT 사용:**
```sql
-- 일반적인 비즈니스 엔티티
CREATE TABLE member (
    member_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE product (
    product_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    price DECIMAL(12,2) NOT NULL,
    stock_quantity INT DEFAULT 0,
    category_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id BIGINT NOT NULL,
    order_number VARCHAR(20) NOT NULL UNIQUE,  -- 비즈니스 주문번호는 별도
    total_amount DECIMAL(15,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    ordered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (member_id) REFERENCES member(member_id)
);
```

**UUID 사용:**
```sql
-- 1. 외부 노출되는 ID (URL에 사용)
CREATE TABLE short_url (
    url_id CHAR(36) PRIMARY KEY,  -- UUID
    original_url VARCHAR(2000) NOT NULL,
    short_code VARCHAR(10) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- /api/urls/a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11 (추측 불가)
-- vs /api/urls/12345 (다음 ID 추측 가능)

-- 2. 분산 시스템에서 충돌 방지
CREATE TABLE distributed_log (
    log_id CHAR(36) PRIMARY KEY,  -- 각 서버에서 독립적으로 생성
    server_id VARCHAR(50),
    log_message TEXT,
    created_at TIMESTAMP
);

-- 3. 결제/보안 민감 데이터
CREATE TABLE payment_transaction (
    transaction_id CHAR(36) PRIMARY KEY,  -- UUID
    payment_method VARCHAR(20),
    amount DECIMAL(15,2),
    status VARCHAR(20)
);
```

### 4.4 AUTO_INCREMENT vs UUID 비교

| 구분 | AUTO_INCREMENT | UUID |
|------|----------------|------|
| **크기** | 8 bytes (BIGINT) | 36 bytes (문자열) / 16 bytes (바이너리) |
| **인덱스 성능** | ✅ 우수 (순차 삽입) | ⚠️ 보통 (랜덤 삽입) |
| **분산 환경** | ⚠️ 별도 처리 필요 | ✅ 충돌 없음 |
| **보안** | ❌ 예측 가능 | ✅ 예측 불가 |
| **정렬** | ✅ 생성순 정렬 | ❌ 정렬 불가 (v4) |
| **가독성** | ✅ 12345 | ❌ a0ee-bc99-... |

> 참고: 시간순 정렬이 필요하면 UUID v7/ULID 같은 시간 기반 키를 고려한다.

### 4.5 실무 선택 가이드

```
AUTO_INCREMENT 선택:
├── 단일 DB 서버
├── 내부 시스템용 ID
├── 조회 성능이 중요한 경우
└── 대용량 테이블 (인덱스 크기 중요)

UUID 선택:
├── 분산 시스템 / 마이크로서비스
├── 외부 API로 노출되는 ID
├── 보안이 중요한 경우 (ID 추측 방지)
└── 여러 DB 간 데이터 병합 필요

ULID/Snowflake 선택:
├── UUID의 장점 + 시간순 정렬 필요
├── 대용량 분산 시스템
└── 로그/이벤트 데이터
```

---

## 5. 복합키 (Composite Key)

### 5.1 개념
2개 이상의 컬럼을 조합하여 PK로 사용

### 5.2 실무 사용 사례

#### 5.2.1 다대다(M:N) 관계의 중간 테이블
```sql
-- 회원-상품 찜하기 (M:N)
CREATE TABLE wishlist (
    member_id BIGINT,
    product_id BIGINT,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (member_id, product_id),
    FOREIGN KEY (member_id) REFERENCES member(member_id),
    FOREIGN KEY (product_id) REFERENCES product(product_id)
);

-- 학생-강좌 수강신청 (M:N)
CREATE TABLE enrollment (
    student_id BIGINT,
    course_id BIGINT,
    semester VARCHAR(10),  -- '2024-1'
    grade VARCHAR(2),
    PRIMARY KEY (student_id, course_id, semester),
    FOREIGN KEY (student_id) REFERENCES student(student_id),
    FOREIGN KEY (course_id) REFERENCES course(course_id)
);

-- 사용자-역할 매핑 (M:N)
CREATE TABLE user_role (
    user_id BIGINT,
    role_id BIGINT,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_by BIGINT,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (role_id) REFERENCES roles(role_id)
);
```

#### 5.2.2 기간별/버전별 데이터
```sql
-- 상품 가격 이력 (특정 시점의 가격)
CREATE TABLE product_price_history (
    product_id BIGINT,
    effective_date DATE,
    price DECIMAL(12,2) NOT NULL,
    created_by BIGINT,
    PRIMARY KEY (product_id, effective_date),
    FOREIGN KEY (product_id) REFERENCES product(product_id)
);

-- 환율 정보
CREATE TABLE exchange_rate (
    from_currency CHAR(3),
    to_currency CHAR(3),
    rate_date DATE,
    rate DECIMAL(15,6) NOT NULL,
    PRIMARY KEY (from_currency, to_currency, rate_date)
);

-- 약관 버전 관리
CREATE TABLE terms_version (
    terms_type VARCHAR(20),     -- 'SERVICE', 'PRIVACY', 'MARKETING'
    version INT,
    content TEXT NOT NULL,
    effective_from DATE NOT NULL,
    PRIMARY KEY (terms_type, version)
);
```

#### 5.2.3 다차원 데이터
```sql
-- 일별 상품별 재고
CREATE TABLE daily_inventory (
    warehouse_id BIGINT,
    product_id BIGINT,
    stock_date DATE,
    quantity INT NOT NULL,
    PRIMARY KEY (warehouse_id, product_id, stock_date)
);

-- 월별 카테고리별 매출 집계
CREATE TABLE monthly_sales_summary (
    year_month CHAR(7),       -- '2024-01'
    category_id BIGINT,
    region_code VARCHAR(10),
    total_sales DECIMAL(15,2),
    total_orders INT,
    PRIMARY KEY (year_month, category_id, region_code)
);
```

### 5.3 복합키 vs 대리키 + UK

**복합키 방식:**
```sql
CREATE TABLE order_item (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

**대리키 + UK 방식:**
```sql
CREATE TABLE order_item (
    order_item_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    UNIQUE KEY uk_order_product (order_id, product_id)
);
```

### 5.4 비교

| 구분 | 복합키 | 대리키 + UK |
|------|--------|------------|
| **FK 참조** | 복잡 (여러 컬럼) | 단순 (1개 컬럼) |
| **인덱스 크기** | 작음 | 큼 (PK + UK) |
| **ORM 호환성** | ⚠️ 일부 제한 | ✅ 우수 |
| **확장성** | ⚠️ 컬럼 추가 어려움 | ✅ 유연함 |
| **의미 명확성** | ✅ 비즈니스 의미 | ⚠️ 인위적 |

### 5.5 실무 권장 사항

```
복합키 권장:
├── 중간 테이블 (M:N 관계)
├── 조회가 주 목적인 집계 테이블
├── 컬럼 추가 가능성 낮음
└── 해당 테이블을 FK로 참조하지 않음

대리키 + UK 권장:
├── 다른 테이블에서 FK로 참조됨
├── JPA/ORM 사용 환경
├── 향후 컬럼 추가 가능성 있음
└── 단순한 API 설계 원할 때
```

> 참고: 복합키는 컬럼 순서가 인덱스 효율에 영향을 준다.
> 조회 조건이 `(A, B)`인 경우, PK를 `(A, B)`로 두는 것이 유리하다.

---

## 6. 실무 PK 설계 패턴

### 6.1 주문 시스템 예시

```sql
-- 주문 테이블: 대리키 + 비즈니스 키 분리
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,      -- 내부 PK
    order_number VARCHAR(20) NOT NULL UNIQUE,        -- 비즈니스 주문번호 (O2024010100001)
    member_id BIGINT NOT NULL,
    total_amount DECIMAL(15,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    ordered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_member_id (member_id),
    INDEX idx_ordered_at (ordered_at)
);

-- 주문 상세: 복합키 사용
CREATE TABLE order_item (
    order_id BIGINT,
    line_number INT,                                  -- 순번
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (order_id, line_number),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- 결제: UUID 사용 (외부 노출)
CREATE TABLE payment (
    payment_id CHAR(36) PRIMARY KEY,                 -- UUID (PG사 연동용)
    order_id BIGINT NOT NULL,
    payment_method VARCHAR(20) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    pg_transaction_id VARCHAR(100),
    paid_at TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

### 6.2 사용자 시스템 예시

```sql
-- 회원: 대리키 (내부) + 비즈니스 식별자들 (UK)
CREATE TABLE member (
    member_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_uuid CHAR(36) NOT NULL UNIQUE,            -- 외부 API용 UUID
    email VARCHAR(255) NOT NULL UNIQUE,              -- 로그인용
    phone VARCHAR(20) UNIQUE,                        -- 본인인증용
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 로그인 이력: 복합키 (조회 최적화)
CREATE TABLE login_history (
    member_id BIGINT,
    login_at TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent VARCHAR(500),
    success BOOLEAN,
    PRIMARY KEY (member_id, login_at),
    FOREIGN KEY (member_id) REFERENCES member(member_id)
);

-- 소셜 로그인 연동: 복합키
CREATE TABLE social_account (
    member_id BIGINT,
    provider VARCHAR(20),                            -- 'GOOGLE', 'KAKAO', 'NAVER'
    provider_user_id VARCHAR(255) NOT NULL,
    connected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (member_id, provider),
    UNIQUE KEY uk_provider_user (provider, provider_user_id),
    FOREIGN KEY (member_id) REFERENCES member(member_id)
);
```

### 6.3 코드/설정 테이블 예시

```sql
-- 공통 코드: 자연키
CREATE TABLE common_code (
    code_group VARCHAR(30),
    code VARCHAR(30),
    code_name VARCHAR(100) NOT NULL,
    display_order INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (code_group, code)
);

-- 데이터 예시
INSERT INTO common_code VALUES
('ORDER_STATUS', 'PENDING', '주문대기', 1, TRUE),
('ORDER_STATUS', 'PAID', '결제완료', 2, TRUE),
('ORDER_STATUS', 'SHIPPED', '배송중', 3, TRUE),
('ORDER_STATUS', 'DELIVERED', '배송완료', 4, TRUE),
('PAYMENT_METHOD', 'CARD', '신용카드', 1, TRUE),
('PAYMENT_METHOD', 'BANK', '계좌이체', 2, TRUE);
```

---

## 7. PK 선택 체크리스트

```
□ 비즈니스 요구사항
  ├── 외부 노출 여부? (노출 → UUID 고려)
  ├── 분산 환경? (분산 → UUID/ULID 고려)
  └── 시간순 정렬 필요? (필요 → AUTO_INCREMENT/ULID)

□ 성능 요구사항
  ├── 대용량 테이블? (대용량 → AUTO_INCREMENT)
  ├── 쓰기 빈도? (높음 → AUTO_INCREMENT)
  └── 인덱스 크기 민감? (민감 → AUTO_INCREMENT)

□ 운영 요구사항
  ├── 다른 테이블에서 FK 참조? (있음 → 대리키)
  ├── ORM 사용? (사용 → 단일 컬럼 대리키)
  └── 데이터 병합 가능성? (있음 → UUID)
```

---

## 8. PK 설계 시 추가 고려사항

### 8.1 데이터 타입 일관성
- PK와 FK의 타입/길이/문자셋/콜레이션은 반드시 동일하게 맞춘다.
- INT vs BIGINT, CHAR(36) vs BINARY(16) 불일치는 조인 성능/인덱스 사용에 영향을 준다.

### 8.2 키 길이와 인덱스 크기
- PK는 짧을수록 인덱스가 작아지고 조인 성능이 좋아진다.
- UUID를 사용할 때는 DB 타입(UUID/BINARY)을 우선 고려한다.

### 8.3 변경/노출 리스크
- PK 변경은 FK 연쇄 수정이 필요해 운영 리스크가 크다.
- 개인정보(주민번호/이메일 등)는 PK로 사용하지 않는 것이 안전하다.

### 8.4 PK/UK/보조 인덱스 구조(간단 도식)

일반적인 개념:
```
PK Index (UNIQUE)  -> Row Data
UK Index (UNIQUE)  -> Row Data
Secondary Index    -> Row Data
```

InnoDB(예시, 클러스터 인덱스):
```
PK Index (clustered) -> Row Data
UK/Secondary Index   -> PK Key -> Row Data
```

> PK가 길면 보조 인덱스가 함께 커져 저장/캐시 효율이 떨어질 수 있다.

### 8.5 EXPLAIN/Execution plan 예시 (버전별)

**MySQL 8.0 (EXPLAIN)**
```sql
CREATE TABLE member (
    member_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    KEY idx_member_email (email)
);

EXPLAIN SELECT name FROM member WHERE email = 'a@example.com';
EXPLAIN SELECT email FROM member WHERE email = 'a@example.com';
```

예시 출력 (name 조회):
```
id  select_type  table   partitions  type  possible_keys      key               key_len  ref   rows  filtered  Extra
1   SIMPLE       member  NULL        ref   idx_member_email   idx_member_email  1022     const 1     100.00    Using where
```

예시 출력 (커버링 인덱스):
```
id  select_type  table   partitions  type  possible_keys      key               key_len  ref   rows  filtered  Extra
1   SIMPLE       member  NULL        ref   idx_member_email   idx_member_email  1022     const 1     100.00    Using index
```

**PostgreSQL 15 (EXPLAIN)**
```sql
CREATE INDEX idx_member_email ON member(email);

EXPLAIN SELECT name FROM member WHERE email = 'a@example.com';
EXPLAIN SELECT email FROM member WHERE email = 'a@example.com';
```

예시 출력:
```
Index Scan using idx_member_email on public.member  (cost=0.29..8.31 rows=1 width=64)
  Index Cond: (email = 'a@example.com'::text)

Index Only Scan using idx_member_email on public.member  (cost=0.29..4.30 rows=1 width=32)
  Index Cond: (email = 'a@example.com'::text)
  Heap Fetches: 0
```

**Oracle 19c (DBMS_XPLAN)**
```sql
EXPLAIN PLAN FOR
SELECT name FROM member WHERE email = 'a@example.com';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

예시 출력:
```
Plan hash value: 1234567890

---------------------------------------------------------------------------------
| Id  | Operation                     | Name             | Rows | Bytes | Cost |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |                  |    1 |    32 |    2 |
|   1 |  TABLE ACCESS BY INDEX ROWID  | MEMBER           |    1 |    32 |    2 |
|   2 |   INDEX RANGE SCAN            | IDX_MEMBER_EMAIL |    1 |       |    1 |
---------------------------------------------------------------------------------
```

**SQL Server 2019 (SHOWPLAN_TEXT)**
```sql
SET SHOWPLAN_TEXT ON;
GO
SELECT name FROM dbo.member WHERE email = N'a@example.com';
GO
SET SHOWPLAN_TEXT OFF;
```

예시 출력:
```
  |--Nested Loops(Inner Join)
       |--Index Seek(OBJECT:([dbo].[member].[idx_member_email]), SEEK:([dbo].[member].[email]=N'a@example.com') ORDERED FORWARD)
       |--Key Lookup(OBJECT:([dbo].[member].[PK_member]), SEEK:([dbo].[member].[member_id]=[member].[member_id]) LOOKUP ORDERED FORWARD)
```

> 참고: 예시 출력의 비용/rows/key_len 값은 통계/버전/데이터 분포에 따라 달라진다.

---

## 9. 요약

| PK 유형 | 사용 시점 | 대표 예시 |
|---------|----------|----------|
| **자연키** | 변하지 않는 표준 코드 | 국가코드, 통화코드, 상태코드 |
| **AUTO_INCREMENT** | 일반적인 비즈니스 엔티티 | 회원, 상품, 주문 |
| **UUID** | 외부 노출, 분산 환경 | 결제ID, API 리소스 ID |
| **ULID/Snowflake** | 대규모 분산 + 정렬 필요 | 이벤트 로그, 메시지 ID |
| **복합키** | M:N 관계, 이력/집계 테이블 | 찜목록, 가격이력, 일별재고 |
