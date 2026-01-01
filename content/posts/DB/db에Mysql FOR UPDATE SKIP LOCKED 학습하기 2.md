+++
title = "db에Mysql FOR UPDATE SKIP LOCKED 학습하기 2"
date = "2025-12-29"
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

## (2) 