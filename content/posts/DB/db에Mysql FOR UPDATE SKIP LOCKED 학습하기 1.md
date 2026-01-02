+++
title = "db에Mysql FOR UPDATE SKIP LOCKED 학습하기 1"
date = "2024-11-12"
author = "Yosep"
tags = ["db", "mysql", "innodb", "locking"]
description = "MySQL에서 FOR UPDATE SKIP LOCKED가 무엇이고 언제 쓰는지, 워커(consumer) 큐 예제로 정리합니다."
weight = 1
+++

# MySQL <mark>FOR UPDATE SKIP LOCKED</mark> 학습하기 (1)

## 1) 한 줄 요약
`SELECT ... FOR UPDATE SKIP LOCKED`는 **(1) 조회한 행을 잠그되(FOR UPDATE)**, **(2) 이미 다른 트랜잭션이 잠근 행은 기다리지 않고(SKIP LOCKED)**, **(3) 잠기지 않은 행만 골라서 가져오는** MySQL(InnoDB)의 “락킹 리드(locking read)” 문법이다.

주로 **여러 워커가 같은 테이블에서 작업을 “경쟁적으로 가져가야” 하는 상황(큐/잡 테이블)** 에서, 워커들이 서로 블로킹 되지 않게 하기 위해 사용한다.  
그래서 작업 큐(Work Queue), 배치 병렬 처리, 여러 워커가 같은 테이블에서 경쟁적으로 일을 가져가는 패턴에 쓴다.

---

## 2) 먼저 `FOR UPDATE`가 뭘 하는가?
InnoDB에서 `SELECT ... FOR UPDATE`는 조회 결과에 포함된 행에 대해 **배타 잠금(Exclusive lock, X lock)** 을 잡는다.

즉, 같은 행을 다른 트랜잭션이:
- `UPDATE/DELETE` 하거나
- `SELECT ... FOR UPDATE`로 또 잠그려 하면

기본적으로는 **락이 풀릴 때까지 기다리게 된다.**

> 포인트: `FOR UPDATE`는 “읽기”처럼 보이지만, 실제로는 **잠금 획득이 목적**인 조회다. (그래서 “락킹 리드”)

---

## 3) 그럼 `SKIP LOCKED`는 뭘 바꾸는가?
`SKIP LOCKED`는 말 그대로 **잠긴 행을 만나면 대기(wait)하지 않고 건너뛰는** 옵션이다.

### 3-1) 대기하는 기본 동작 (SKIP LOCKED 없음)

```text
jobs 테이블 (status='READY' 인 작업을 1개씩 가져온다고 가정)

┌────┬────────┐
│ id │ status │
├────┼────────┤
│  1 │ READY  │
│  2 │ READY  │
│  3 │ READY  │
└────┴────────┘

TX-A: BEGIN;
TX-A: SELECT ... WHERE status='READY' ORDER BY id LIMIT 1 FOR UPDATE;
      -> id=1 을 선택 + (id=1) 행 락 획득

TX-B: BEGIN;
TX-B: SELECT ... WHERE status='READY' ORDER BY id LIMIT 1 FOR UPDATE;
      -> (id=1)을 잠그려고 시도하다가 TX-A가 COMMIT 할 때까지 대기...
```

### 3-2) 건너뛰는 동작 (SKIP LOCKED 사용)

```text
TX-A: BEGIN;
TX-A: SELECT ... FOR UPDATE;
      -> id=1 lock

TX-B: BEGIN;
TX-B: SELECT ... FOR UPDATE SKIP LOCKED;
      -> id=1 은 잠겨 있으니 skip
      -> 다음 후보 id=2 를 선택 + (id=2) lock
```

즉 **“대기” 대신 “가능한 것만 빨리 가져오기”** 로 전략이 바뀐다.

---

## 4) 예제 쿼리: “잡 테이블을 여러 워커가 소비”하는 패턴

### 4-1) 예시 테이블

``` sql
CREATE TABLE jobs (
  id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  status ENUM('READY','PROCESSING','DONE','FAILED') NOT NULL,
  payload JSON NOT NULL,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
    ON UPDATE CURRENT_TIMESTAMP,
  INDEX ix_jobs_status_id (status, id)
) ENGINE=InnoDB;
```

### 4-2) 워커가 “하나” 가져가서 처리 중으로 바꾸기 (claim)

``` sql
-- (권장) 트랜잭션을 짧게: "선택 + 상태변경"까지만 커밋하고,
-- 실제 비즈니스 처리(외부 API 호출 등)는 커밋 이후에 수행.

START TRANSACTION;

SELECT id
INTO @job_id
FROM jobs
WHERE status = 'READY'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE jobs
SET status = 'PROCESSING'
WHERE id = @job_id;

COMMIT;
```

이렇게 하면 동시에 여러 워커가 떠도:
- 이미 다른 워커가 집어간(락 잡은) `READY` 행은 **건너뛰고**
- 각자 **서로 다른 작업을 가져가** 처리할 수 있다.

### 4-3) 2개 워커가 동시에 실행될 때 그림

```text
Time →

Worker A (TX-A): BEGIN ─ SELECT ... FOR UPDATE SKIP LOCKED (id=1 lock)
                 └──── UPDATE id=1 -> PROCESSING ─ COMMIT

Worker B (TX-B): BEGIN ─ SELECT ... FOR UPDATE SKIP LOCKED (id=1 skip, id=2 lock)
                 └──── UPDATE id=2 -> PROCESSING ─ COMMIT

결과 (락은 COMMIT과 함께 해제되지만, status가 바뀌어 재선택되지 않음)

┌────┬────────────┐
│ id │ status     │
├────┼────────────┤
│  1 │ PROCESSING │
│  2 │ PROCESSING │
│  3 │ READY      │
└────┴────────────┘
```

---

## 5) 주의사항 (자주 하는 실수)

### 5-1) 트랜잭션 밖에서 쓰면 의미가 없다
`FOR UPDATE`는 **트랜잭션 동안만** 잠금을 유지한다.  
오토커밋 상태에서 단독으로 실행하면, 문장이 끝나자마자 커밋되어 **락도 바로 풀려버린다.**

### 5-2) “처리 시간”을 트랜잭션에 넣지 말기
락을 오래 잡고 있으면:
- 다른 워커들이 계속 스킵/스캔하느라 비효율이 커지고  
- 같은 테이블을 쓰는 다른 트랜잭션도 영향을 받는다.

그래서 보통은 **(1) 잡을 고르고 잠깐 락 잡아서 상태 바꾸고 COMMIT → (2) 실제 처리는 COMMIT 이후**로 분리한다.

### 5-3) SKIP LOCKED는 “순서 보장” 기능이 아니다
`ORDER BY id`를 써도, 락 때문에 어떤 행은 건너뛰므로 워커마다 실제 처리 순서는 달라질 수 있다.  
큐의 “정확한 순서”가 정말 중요하다면 다른 설계가 필요할 수 있다.

### 5-4) 갭락/넥스트키락(특히 MySQL InnoDB)과 결합되면 생각보다 넓게 잠길 수 있음
격리수준(예: REPEATABLE READ) + 범위 조건이 있으면 “행만”이 아니라 “범위”가 잠길 수 있다.  
작업 큐 쿼리는 가능하면 인덱스를 타는 단순 조건 + LIMIT로 좁게 설계하는 게 좋다.

### 5-5) “건너뛰기”라서 공정성(fairness)이 깨질 수 있음
잠긴 애를 계속 건너뛰다 보면 특정 작업이 오래 남아있을 수 있다(스타베이션 가능).
보통 created_at, priority, 재시도 횟수 등을 함께 설계해서 완화한다.

### 5-6) 인덱스/정렬이 매우 중요
WHERE status='READY' ORDER BY created_at LIMIT N이면  
(status, created_at) 같은 인덱스가 없으면 많은 스캔이 발생할 수 있다.  
특히 워커 수가 많아질수록 “찾다가 락 걸린 행을 건너뛰는 과정”이 비싸질 수 있다.

---

## 6) (옵션) `NOWAIT`와 차이
- `NOWAIT`: 잠긴 행을 만나면 **즉시 오류** (대기 없음)
- `SKIP LOCKED`: 잠긴 행을 만나면 **그 행을 건너뛰고 계속 탐색**

워크 큐/잡 소비 패턴에는 보통 `SKIP LOCKED`가 더 잘 맞는다.


## 예시 프로젝트를 진행해보자.
### 0) 목표 모델 (현실적인 보장 수준)
- At-least-once 처리(가장 현실적): 워커 크래시/네트워크 장애가 있으면 같은 잡이 “재실행”될 수 있음 → 업무 로직은 멱등(idempotent) 하게 설계.  
- “정확히 한 번(exactly-once)”은 DB 큐만으로는 어렵고, **멱등키 + 유니크 제약 + (필요시 Outbox/사가)**로 구현하는 게 정석.

### 1) 테이블 설계 (DDL)
상태 모델  
- READY : 실행 가능(스케줄 도래)  
- PROCESSING : 워커가 점유(lease 보유)
- DONE : 성공
- FAILED : 재시도 한도 초과/영구 실패
- CANCELED : 취소(선택)  

``` sql
CREATE TABLE job_queue (
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  queue         VARCHAR(64) NOT NULL,          -- 큐/토픽/업무 종류
  status        TINYINT NOT NULL,              -- 0=READY,1=PROCESSING,2=DONE,3=FAILED,4=CANCELED
  priority      INT NOT NULL DEFAULT 0,        -- 클수록 먼저
  run_at        DATETIME(6) NOT NULL,          -- 실행 가능 시각(지연/백오프용)

  attempts      INT NOT NULL DEFAULT 0,
  max_attempts  INT NOT NULL DEFAULT 25,

  payload       JSON NOT NULL,
  result        JSON NULL,
  last_error    TEXT NULL,

  locked_by     VARCHAR(128) NULL,             -- 워커 식별자(예: podName)
  lock_token    BINARY(16) NULL,               -- UUID_TO_BIN(UUID())
  locked_at     DATETIME(6) NULL,
  lock_until    DATETIME(6) NULL,              -- lease 만료 시각(visibility timeout)

  dedupe_key    VARBINARY(64) NULL,            -- 선택: 멱등/중복 방지 키(업무별)
  created_at    DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at    DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  finished_at   DATETIME(6) NULL,

  PRIMARY KEY (id),

  -- (선택) dedupe_key를 쓰면 큐 단위 중복 방지 가능 (NULL은 여러 개 허용)
  UNIQUE KEY uq_queue_dedupe (queue, dedupe_key),

  -- 픽업(READY + run_at) 최적화
  KEY ix_pick (queue, status, run_at, priority, id),

  -- 리스 만료 스캐닝(죽은 워커 복구)
  KEY ix_lease (status, lock_until),

  -- 워커별 모니터링/연장
  KEY ix_worker (locked_by, status)
) ENGINE=InnoDB;
```

[핵심 포인트]
* run_at: 지연/재시도 백오프를 정확히 표현.
* lock_until: 워커가 죽어도 일정 시간 후 자동 회수(requeue).
* lock_token: “내가 잡은 잡만 ack/연장 가능”하게 만드는 안전장치.

### 2) 인덱스/락 관련 중요한 주의점 (InnoDB 특성)
InnoDB는 잠금 읽기/UPDATE/DELETE 시 스캔한 인덱스 레코드 범위에 락을 건다(조건에서 제외되는 행도 스캔되면 락 영향).  
그래서 픽업 쿼리가 반드시 ix_pick를 타도록 만드는 게 중요. → 실전에서는 FORCE INDEX (ix_pick)를 자주 쓴다.


### 3) 잡 등록(Enqueue)
``` sql
INSERT INTO job_queue(queue, status, priority, run_at, payload, dedupe_key)
VALUES (?, 0, ?, NOW(6), CAST(? AS JSON), ?);
```
* dedupte_key가 있으면 중복 등록 방지 가능.

### 4) 잡 가져오기 - FOR UPDATE SKIP LOCKED
#### 권장 패턴: <mark>짧은 트랜잭션으로 점유만 하고 바로 커밋</mark>
-> 긴 작업을 트랜잭션에 묶어서 락을 오래 잡으면 전체 처리량 급락

(1) 점유 트랜잭션
``` sql
START TRANSACTION;

-- 1) 실행 가능 + READY 인 것 중 잠글 수 있는 것만 가져오기
SELECT id
FROM job_queue FORCE INDEX (ix_pick)
WHERE queue = ?
  AND status = 0
  AND run_at <= NOW(6)
ORDER BY priority DESC, run_at ASC, id ASC
LIMIT ?
FOR UPDATE SKIP LOCKED;

-- 2) 방금 뽑은 id들을 PROCESSING으로 전환 + lease 부여
-- (애플리케이션에서 id 리스트를 바인딩)
UPDATE job_queue
SET status     = 1,
    locked_by  = ?,
    lock_token = UUID_TO_BIN(UUID()),
    locked_at  = NOW(6),
    lock_until = NOW(6) + INTERVAL ? SECOND
WHERE id IN ( ... );

COMMIT;
```
* <mark>SKIP LOCKED</mark>는 이미 다른 워커가 잠근 행은 건너뛰고 잠금 가능한 행만 가져온다.
* 복제환경이면, SKIP LOCKED는 statement-based replication(문장 기반 복제)에 unsafe하기 때문에 운영 설정 점검이 필요하다.

(2) 워커 처리(트랜잭션 외부에서)
* 비즈니스 처리 수행(HTTP 호출, 파일 처리, 다른 DB 작업 등)

### 5) 완료 처리(Ack)
#### 성공 Ack(내가 점유한 잡만 완료)
``` sql
UPDATE job_queue
SET status = 2,
    result = CAST(? AS JSON),
    finished_at = NOW(6)
    locked_by = NULL,
    lock_token = NULL,
    locked_at = NULL,
    lock_until = NULL
WHERE id = ?
  AND status = 1
  AND locked_by = ?
  AND lock_token = UUID_TO_BIN(?);
```
* lock_token 조건으로 "옛 워커가 늦게 와서 DONE 처리"같은 사고를 방지.

### (6) 실패/재시도(Backoff + run_at)
#### 실패 Ack + 재시도 가능하면 READY로 되돌리기
``` sql
UPDATE job_queue
SET attempts = attempts + 1,
    last_error = ?,
    -- 예: 지수 백오프(초) = LEAST(3600, POW(2, attempts) * 5)
    run_at = NOW(6) + INTERVAL ? SECOND,
    status = CASE
              WHEN attempts + 1 < max_attempts THEN 0  -- READY
              ELSE 3                                  -- FAILED
             END,
    locked_by = NULL,
    lock_token = NULL,
    locked_at = NULL,
    lock_until = NULL,
    finished_at = CASE
                    WHEN attempts + 1 < max_attempts THEN NULL
                    ELSE NOW(6)
                  END
WHERE id = ?
  AND status = 1
  AND locked_by = ?
  AND lock_token = UUID_TO_BIN(?);
```
* `base=5s`, `factor=2`, `cap=3600s` 같은 파라미터와 약간의 jitter로 지수 백오프 설계 가능.
* 이 백오프를 통해 "동시 재시도 폭발"이 줄어듦

### (7) 워커 크래시/장기 실행 대비: Lease 연장 + 만료 회수
#### (1) Lease 연장(heartbeat)
긴 작업이면 주기적으로:
``` sql
UPDATE job_queue
SET lock_until = NOW(6) + INTERVAL ? SECOND
WHERE id = ?
  AND status = 1
  AND locked_by = ?
  AND lock_token = UUID_TO_BIN(?)
  AND lock_until > NOW(6);
```

#### (2) 만료 회수
워커가 죽어서 lock_until이 지났으면 READY로 되돌림
``` sql

UPDATE job_queue
SET status = 0,
    locked_by = NULL,
    lock_token = NULL,
    locked_at = NULL,
    lock_until = NULL
WHERE status = 1
  AND lock_until < NOW(6)
LIMIT 1000;
```

### (8) 운영 팁 (성능/안정성)
#### A. "픽업 쿼리"는 반드시 인덱스를 타도록 강제
* InnoDB는 스캔한 인덱스 범위에 락이 걸릴 수 있어서, 픽업이 느리거나 락 범위가 커지면 큐 전체가 망가질수 있다.  
참조: https://dev.mysql.com/doc/refman/9.1/en/innodb-locks-set.html?utm_source=chatgpt.com

#### B. 격리수준
* 작업 큐 픽업 트랜잭션만 READ COMMITTED로 낮추면 불필요한 락 경합이 줄어드는 경우가 많다.  
- 단, 서비스 전체 일관성 요구와 충돌하지 않도록 큐 전용 커넥션/트랜잭션 분리 추천.

#### C. 워커 루프 전략
* LIMIT은 보통 10~100 정도로 적당히 잡는다.
* `가져올 게 없으면` busy spin 하지 말고 짧게 sllep(예: 50~200ms + jitter) 후 재시도.

#### D. 데드락/락 타임아웃은 "정상적인 운영 이벤트"
InnoDB는 데드락을 감지하면 한 트랜잭션을 롤백시킬 수 있고, 감지 기능을 끄면 `innodb_lock_wait_timeout`으로 정리된다. 즉, 재시도 로직이 필요하다.

### (9) 이 방식이 보장해주는 것과 보장하지 못하는 부분을 방어해야하는 것
#### 보장해주는 것
* 여러 워커가 동시에 돌아도, 동일 잡을 동시에 점유하지 않음
* 잠긴 잡은 `SKIP LOCKED`로 건너뛰어서 대기 없이 처리량 유지
* 워커가 죽어도 `lock_until`로 자동 복구 및 재시도 가능

#### 비즈니스 처리(외부 API 호출, 결제, 송금 등)는 멱등하게:
* dedupe_key같은 업무 키를 유니크로 잡는다.
* 처리 결과 저장도 "이미 처리됨"상태이면 무시하도록 설계한다.