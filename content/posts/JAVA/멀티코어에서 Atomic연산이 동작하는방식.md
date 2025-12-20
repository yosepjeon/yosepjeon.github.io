+++
title = "멀티코어에서 Atomic연산이 동작하는방식"
date = "2023-02-11"
weight = 2
+++

# 멀티코어에서 Atomic연산이 동작하는방식
학습을 진행하면서, 문득 이런 의문이 하나 생겼습니다.
"Atomic연산이 멀티코어 환경에서 어떻게 동작하지?"
- 프로그램 전체가 “한 번에 하나씩만 실행”되는 게 아니라 특정 메모리 위치(Atomic 변수 1개) 에 대한 read-modify-write 갱신만 원자적으로(선형화되게) 보장되는 것!


## Mac기준 16코어 환경에서 어떻게 1개씩 성공하는지?
### 1) Atomic 연산은 "CAS 시도"로 경쟁을 붙인다.
여러 코어가 동시에 `X`를 증각시키려 할 때의 tuesdo 코드입니다.
이 CAS연산이 단 1개의 코어에서만 일어나도록 CPU가 보장합니다.
```tuusdo
repeat:
    old = load(X)
    new = old + 1
until CAS(X, old, new) succeeds
```

### 2) CPU는 "캐시 일관성(coherence)"으로 그 메모리 주소를 한 코어만 수정 가능하게 만든다.
현대의 CPU는 보통 캐시 라인단위(예: 64바이트)로 "이 라인을 누가 수정할 권한이 있는지"를 조정합니다.  
즉, `X`가 속한 캐시 라인을 16코어 중 1개 코어만 "수정 가능(modified)" 상태로 만듭니다.

# 캐시일관성(cohgerence)란?
캐시 일관성(Cache Coherence) 은, 멀티코어 CPU에서 각 코어가 가진 “자기 L1/L2 캐시”들이 같은 메모리 값을 서로 모순 없이 보게 해주는 규칙/메커니즘
``` bash
Core0 ─ L1$ ┐
Core1 ─ L1$ ├─(공유) L3$ ─ DRAM(메인메모리)
Core2 ─ L1$ ┤
Core3 ─ L1$ ┘
```

각 코어는 속도를 위해 데이터를 자기 캐시에 복사해 두고 쓰는데, 문제가 있습니다.
* Core0가 x=1로 바꿔서 자기 캐시에만 반영해버리면
* Core1은 자기 캐시에 남아있는 옛 값 x=0을 계속 볼수도있다.  
이 <mark>코어마다 서로 다른 x를 보는 상태</mark>를 막는 게 <mark>coherence</mark>이다.
  
## coherence가 보장하려는 핵심
1. 한 주소(x 같은 변수 하나)에 대해 최신쓰기가 다른 코어에도 결국 보이게 한다.
2. 한 주소에 대한 쓰기 순서를 모든 코어가 같은 순서로 관측한다.

```bash
thread 0: Core0 L1: x=0    Core1 L1: x=0
t1: Core0이 x=1로 write -> coherence가 Core1의 x 캐시라인을 INVALID로 만듦
t2 Core1이 x를 read -> "내 캐시에 x 없음(Invalid)" -> 더 하위캐시(L3)나 메모리에서 최신값 x=1 읽음
```

# coherence vs consistency 차이
### 캐시 일관성(coherence)과 메모리 일관성(consistency)은 비슷해 보이지만 다른 개념입니다.
* Cache Coherence: 한 주소에 대한 쓰기 순서를 모든 코어가 같은 순서로 관측하는 것
* Memory Consistency: 여러 주소에 대한 읽기/쓰기 연산의 전체 순서를 모든 코어가 같은 순서로 관측하는 것
  
coherence만으로는 아래와 같은 상황에서 원하는 결과를 보장하지 못합니다.
```bash
Core0:
  x = 1
  y = 1

Core1:
  while (y == 0) { }   // y가 1 되는 거 기다림
  print(x)             // 여기서 x가 0으로 보일 수도? (순서/가시성 문제)
```
  
그래서 언어 레벨에서는 volatile, synchronized, atomic 등이 필요한 메모리 장벽을 제공합니다.  

### 성능 이슈로도 중요한 coherence
coherence는 "캐시라인(보통 64바이트)" 단위로 동작하는데, 서로 다른 변수라도 같은 캐시라인에 있으면 쓸데없는 invalidate 폭탄이 생길 수 있습니다.  
  
# coherence의 동작, 그리고 Atomic/Volatile이 왜 멀티코어에서 안전해지나?
### 1) 전제: coherence는 "캐시라인" 단위로 움직인다.
CPU 캐시는 보통 캐시라인(보통 64바이트) 단위로 데이터를 읽고 씁니다.
즉, 변수 `x` 하나만 바꿔도, 실제로는 `x`가 들어있는 64바이트 라인 전체의 상태가 바뀝니다.

### 2) MESI상태 
한 캐시라인이 각 코어의 L1/L2 캐시에 있을때, 그 "복사본"의 상태를 MESI 프로토콜로 관리합니다.
``` bash
M (Modified): 이 코어가 수정했음, 아직 메모리에 안 씀(더티). 다른 코어엔 없음.
E (Exclusive): 이 코어가 유일하게 가지고 있음(클린), 메모리와 같음.
S (Shared): 여러 코어가 "읽기용으로" 공유(클린), 메모리와 같음.
I (Invalid): 이 코어의 캐시라인은 무효화됨(읽기 불가, 데이터가 없다고 봐도 됨)
```
  
#### 중요한 규칙
* 어떤 시점에도 M(Modified)은 <mark>딱 한 코어만</mark> 가질 수 있습니다.(쓰기 독점)
* S는 여러 코어가 동시에 가질 수 있습니다.(읽기 공유)

### 3) 읽기(read) 때의 흐름
#### (1) 내 캐시에 없으면 -> 가져온다
``` bash
Core1 L1: I  (x 라인 없음)
   |
   |  Read request (Bus/Ring)
   v
메모리 또는 다른 코어 캐시에서 라인을 받음
```

#### (2) 누군가가 M으로 들고 있으면 "그 코어가 최신"
``` bash
Core0 L1: M (x 최신, 더티)
Core1 L1: I

Core1 read x
  -> Core0가 라인을 "전달/쓰기반영(Flush)" 해주거나
  -> Core0가 응답해 최신 값을 넘겨줌
Core1은 보통 S로 받음
```

#### (3) 쓰기(write) 때의 흐름: invalidate 기반
대부분의 멀티코어는 "Update(다른 캐시를 바로 갱신)" 방식이 아니라 "Invalidate(다른 캐시 복사본을 무효화)" 방식으로 쓰기를 처리합니다.  

``` bash
<1> 쓰기를 하려면 "독점 소유권"을 먼저 얻어야합니다. (Read for Ownership, RFO)  
예를 들어 Core0가 x에 write하려고 할 때

Core0: write x
  |
  |  RFO(Read For Ownership) / GetX 요청
  v
다른 코어들의 그 캐시라인 상태를 I로 만들기(invalidate)
Core0는 그 라인을 M(또는 E→M)로 만들고 write 수행

<2> 예시 타임라인
t0)
Core0 L1: S (x=0)
Core1 L1: S (x=0)

t1) Core0가 x=1로 write 하고 싶다
Core0: "나 이 라인 독점할게!" -> Core1의 라인 INVALIDATE

t2)
Core0 L1: M (x=1)   (더티, 최신)
Core1 L1: I

t3) Core1이 x를 다시 read
Core1: "내 캐시에 없네(I)" -> Core0(또는 메모리)에서 최신 라인 가져옴
Core1 L1: S (x=1)
Core0 L1: (상황에 따라 M 유지 or S로 다운그레이드)

```
  
#### (4) 그럼 Atomic은 왜 "진짜로 하나씩만" 될까?
Atomic의 핵심 연산(CAS)은 CPU가 Read-Modify-Write를 쪼개지지 않는 한 덩어리로 보장합니다.
```bash
CAS(x, expected, update):
  1) x 읽고
  2) expected랑 비교하고
  3) 같으면 update로 쓰고
  4) 다르면 실패
```
  
이걸 한 트랜잭션으로 묶으려면:
* 이 순간 다른 코어가 같은 라인을 동시에 건드리면 안됨
* CPU는 coherence를 이용해 라인 소유권(M)을 획득하고, 그동안 다른 코어는 I가 돼서 접근이 지연/재시도됨  

``` bash
둘 다 동시에 x++ 하려는 상황

Core0: RFO(GetX) -> 라인 독점(M) -> RMW 완료 -> release
Core1: (그 동안 라인 I) -> 나중에 다시 RFO -> RMW 완료
```

Lock이랑 같은 느낌이지만, 전통적인 뮤텍스 락이 아니라 캐시라인 소유권을 coherence로 뺏고 뺏기는 방식이라서 
동시에 같은 라인에 RMW는 물리적으로 불가능해집니다.  

** 성능 포인트: cache line bouncing **
다만 경합이 심하면 라인이 왔다갔다하면서(ownership 이동) 성능이 급락합니다.
``` bash
Core0 M <--> Core1 M <--> Core2 M ...
(계속 invalidate + ownership 이동)
```

그래서 LongAdder 같은 "분산 카운터"가 나왔습니다.

#### (5) volatile은 coherence만이 아니라 “순서(메모리 모델)”까지 묶어준다
coherence는 "한 주소에 대한 쓰기 순서"만 보장하지만, 서로 다른 변수 x,y 사이의 관측 순서까지는 자동으로 해결해주지 않습니다.
그래서 volatile은 보통 이렇게 동작합니다.
* volatile write = release
그 volatile write 이전의 일반 write들이 “먼저” 보이도록

* volatile read = acquire
그 volatile read 이후의 일반 read들이 “나중”에 오도록

```bash
Core0:
  data = 123          // 일반 write
  ready = true        // volatile write (release)

Core1:
  while (!ready) {}   // volatile read (acquire)
  print(data)         // 여기서 123을 보장받고 싶음
```
* coherence만 믿으면 “ready는 보이는데 data는 옛값” 같은 재주문/가시성 문제를 완전히 배제하기 어려움
* volatile은 그걸 언어/컴파일러/CPU 배리어로 막아줌

Atomic = “원자성(단일 연산 덩어리)” + (대개) 강한 배리어 효과
volatile = “가시성 + 순서(특히 publish/consume)”

#### (6) 서로 다른 변수라도 같은 캐시라인(64바이트) 안에 있으면?
두 변수는 논리적으로는 독립적인 개념이지만, coherence는 라인 단위이기 때문에 반복해서 invalidate가 발생합니다.  
