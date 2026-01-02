+++
title = "db에Mysql FOR UPDATE SKIP LOCKED 학습하기 2"
date = "2024-11-21"
author = "Yosep"
tags = ["db", "mysql", "innodb", "locking"]
description = "MySQL에서 FOR UPDATE SKIP LOCKED가 무엇이고 언제 쓰는지, 워커(consumer) 큐 예제로 정리합니다."
weight = 2
+++

# MySQL <mark>FOR UPDATE SKIP LOCKED</mark> 학습하기 (2)

`FOR UPDATE SKIP LOCKED`는 락 충돌이 났을때 대기(wait)를 skip하고 워커 경쟁 상황에서 
블로킹을 없애주기 때문에 효율적입니다..

## (1) 내부적으로 어떻게 동작하는가?
### A. "Locking Read" 경로로 들어간다.
`SELECT ... FOR UPDATE` 구문은 단순 스냅샷 읽기가 아니라, Inno DB가 결과로 반활할 후보 
레코드에 '행 락'을 걸려고 시도하는 읽기로 동작한다.  
  
### B. 인덱스를 타고 스캔하면서 "레코드마다 락을 시도"
실제로 InnoDB의 row lock은 레코드 자체가 아니라 '인덱스 레코드'에 걸리는게 핵심이다. 
그래서 쿼리 플랜이 어떤 인덱스를 타는지가 락 범위/경합을 좌우한다.  
``` bash
B-Tree 인덱스 스캔
  ├─ 후보 레코드 R1 발견 → X(배타) 락 시도
  │    ├─ 성공 → 결과집합에 포함
  │    └─ 실패(다른 트랜잭션이 보유) → (여기서 NOWAIT/SKIP/WAIT가 갈림)
  ├─ 후보 레코드 R2 발견 → X 락 시도 → …
  └─ LIMIT 채우면 종료
```

### C. 충돌났을 때의 분기: WAIT vs NOWAIT vs SKIP
* DEFAULT(WAIT): 락이 이미 잡혀있으면 대기열에 들어가서 락이 풀릴 떄까지 기다린다.
* NOWAIT: 락이 이미 잡혀있으면 즉시 에러를 반환한다.
* SKIP LOCKED: 락이 잡혀있으면 그 레코드는 결과에서 빼고 다음 레코드로 진행한다.

## (2) 이전과 달리 뭐가 좋아졌길래 잡 큐가 쉬워졌나?
### ① 헤드-오브-라인 블로킹 제거 (처리량/지연 개선)
Head-of-Line Blocking: 큐(대기열)의 가장 앞부분에 있는 요청이나 패킷이 지연될 때, 뒤따르는 모든 항목이 함께 기다려야 하는 성능 저하 현상

#### 잡 큐에서 흔한 병목
- 워커 100개가 <marK>ORDER BY ... LIMIT 1 FOR UPDATE</mark>로 "맨 앞" Job을 보려고 몰림
- 그 "맨 앞 행"이 누군가에게 잠겨 있으면 나머지 워커 전체가 기다리면서(lock wait) 멈춰버림 → 처리량 급락

<mark>SKIP LOCKED</mark>는 이 상황에서 `맨 앞이 잠겨 있으면 그 다음 잠글 수 있는 걸 집어가라`로 바꿔서, 대기 시간을 거의 없앤다. MYSQL 블로그에서도 "hot rows(경합이 집중되는 행)"에서 SKIP LOCKED/NOWAIT로 lock으로 인한 병목을 피한다고 설명한다.  
참고: https://dev.mysql.com/blog-archive/mysql-8-0-1-using-skip-locked-and-nowait-to-handle-hot-rows  
  
### ② '정합성'은 엔진 락으로, '분산 경쟁'은 skip으로
#### 이전 mysql8 버전 이전에 흔한 패턴은
- 락 없이 SELECT로 고르고
- UPDATE ... WHERE ids in (...) AND status = 'READY'같은조건 없데이트로 점유 시도
- 실패하면 루프 재시도

#### 이 방식도 가능하지만
- 재시도/경합이 애플리케이션 레벨로 올라감
- 경합이 커지면 불필요한 쿼리 폭탄(select/update 루프)이 생김
- 애플리케이션에서 잘못 구현하면 두 워커가 같은 잡을 집었다가 한 쪽이 늦게 실패 같은 애매한 처리 흐름이 생김
  
<mark>FOR UPDATE SKIP LOCKED</mark>는 "선택 + 점유"를 DB에서 관리해주기 때문에 워커 경쟁을 DB가 자연스럽게 종재하고 앱은 단순해진다.

### ③ 데드락/타임아웃의 ‘폭발’을 줄이는 방향
<mark>LOCK WAIT</mark>은 락 대기열을 키우고, 대기 시간이 늘면 타임아웃/데드락이 눈덩이처럼 커질수 있다. InnoDB는 데드락을 감지해 롤백 시키거나, 
타임아웃으로 정리한다는 전제가 있고, <mark>SKIP LOCKED</mark>는 기다리지 않고 다른 요소를 찾기때문에 병목을 완화하는 쪽으로 작동한다.

## 🚨"정합성이 더 강해진 건 아니다!!"
<mark>SKIP LOCKED</mark>는 'inconsistent view'를 반환한다. 즉, 원래는 먼저 처리되어야 할 Job이 잠겨 있으면 그걸 건너뛰고 뒤의 잡을 처리할 수 있다. 
그래서 일반적인 트랜잭션 정합성(정확한 순서보장, 완전히 일관된 조회)에는 부적합이고, 개발자 문서에서도 `큐 같은 테이블에서 경합을 피하는 용도`로 한정해서 권장한다.  
참고: https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html

## ⭐️ SKIP LOCKED도 락과 관련해서 잘 고려해야한다!!
`SIP LOCKED`는 "남이 잡고 있는 락을 만났을 때 기다리지 않고 건너뛰겠다"이지만 두가지는 그대로다.
### (A) 스캔하는 범위는 중요하다
락은 탐색할 행에만 거는게 아니라, 스캔한 인덱스 레코드들에 락이 걸릴수 있다. 즉, 인덱스를 못 타거나 범위가 넓으면
- 워커가 Job하나를 찾기위해 엄청나게 많은 인덱스 레코드를 스캔해야한다.
- 그 과정에서 불필요한 레코드/갭락을 잡아버린다.
- 결과적으로 해당 워커가 다른 워커의 DB 행위를 막게된다.  
  
<mark>SKIP LOCKED는 "남의 락을 기다리지 않는 것"이지, 락을 넓게 잡는 문제를 해결해주지는 못한다.</mark>

### (B) SKIP LOCKED는 오히려 "더 많이 스캔"하게 만들수 있다.
예를 들어 `LIMIT 100 FOR UPDATE SKIP LOCKED`라고 가정했을때, 앞쪽 100개가 이미 다른 서버에게 잡혀있다면
- 101~200... 더 뒤를 계속 스캔해야함
- 그 과정에서 스캔 범위가 커지고 락 영향도 커짐

그래서 Job Queue쿼리는 
1) 잘 타는 인덱스
2) 범위를 최대한 좁게
3) LIMIT을 작게
이러한 조건을 지키는것이 중요하다.

```bash
(일반 SELECT)                 (SELECT ... FOR UPDATE SKIP LOCKED)
- 스냅샷(read view) 기준       - 인덱스 스캔하며 락 시도
- 필요하면 undo로 과거버전       - 잠긴 레코드 만나면:
재구성해서 읽음                         * WAIT: 기다림
                                    * NOWAIT: 에러
                                    * SKIP: 결과에서 제외하고 계속 스캔
                             => BUT: 스캔 범위가 넓으면
                              내가 많은 index record / gap을 잠글 수 있음
```