+++
title = "volatile, Atomic 그리고 Synchronized"
date = "2023-02-04"
weight = 1
+++

# <mark>Volatile, Atomic 그리고 Synchronized</mark>에 대한 정리

## 1) `volatile` 한 줄 요약
`volatile`은 **(1) 가시성(visibility)** 과 **(2) 순서(ordering)** 를 보장한다.

- **가시성**: 한 스레드가 쓴 값을 다른 스레드가 *즉시* (정확히는 “다음 volatile read 때”) 보게 만든다.
- **순서**: `volatile` 읽기/쓰기를 경계로 **재정렬(reordering)** 이 제한된다. (release/acquire 의미)

> 단, **원자성(atomicity)** 은 보장하지 않는다. (`count++` 같은 복합 연산은 깨짐)

---

## 2) 왜 문제가 생기나? (CPU 캐시/레지스터/컴파일러 최적화)
멀티코어에서는 각 코어가 캐시를 갖고 있고, 컴파일러/JIT가 성능을 위해 코드 순서를 바꾸거나 값을 레지스터에 “고정”해버릴 수 있다.

### volatile이 없을 때 “보였어야 할 값이 안 보이는” 전형적 상황



```
        (Thread A on Core 0)                 (Thread B on Core 1)
      ┌──────────────────────┐            ┌──────────────────────┐
      │   flag = true;       │            │ while (!flag) {      │
      │                      │            │   // do nothing      │
      └─────────┬────────────┘            └─────────┬────────────┘
                │                                   │
       (A의 캐시/스토어버퍼에만 반영)                      │ (B는 자기 캐시/레지스터 값만 봄)
                ▼                                   ▼
 ┌─────────────────────────┐           ┌─────────────────────────┐
 │ Core0 cache / store buf │           │ Core1 cache / register  │
 │     flag=true           │           │ flag=false (계속 이 값만)  │
 │  (아직 메모리로 전파 안 됨)   │           │                         │
 └────────────┬────────────┘           └────────────┬────────────┘
              │                                     │
              ▼                                     ▼
         ┌───────────────────────────────────────────────────┐
         │                  Main Memory                      │
         │                  flag=false                       │
         └───────────────────────────────────────────────────┘

```


핵심:
- A가 `flag=true`를 “썼다” 해도 **다른 코어로 전파가 늦을 수 있고**
- B는 `flag`를 매번 메모리에서 다시 안 읽고 **레지스터/캐시 값으로 무한루프**에 빠질 수도 있다.

---

## 3) `volatile`이 보장하는 것 (JMM 관점)

### 3-1) 가시성: “volatile write → volatile read”는 항상 보인다
- 스레드 A가 `volatile` 변수에 **write** 하면
- 스레드 B가 같은 변수를 **read** 할 때
- B는 **A의 최신 write(또는 그 이후의 write)** 를 보게 된다.

### 3-2) happens-before: volatile은 “메모리 경계”를 만든다
JMM의 핵심 규칙:

> 한 스레드의 `volatile write`는, 다른 스레드의 “그 변수에 대한” `volatile read`보다 **happens-before** 이다.

이게 중요한 이유:
- 단지 `flag` 값만 보이는 게 아니라
- `flag` 앞뒤의 일반 변수들도 “순서 있게” 보이게 만드는 데 쓰인다.

### Release/Acquire 의미

```java
// Thread A (writer)
data = 42;       // 일반 write
flag = true;     // volatile write

// Thread B (reader)
while (!flag) { } // volatile read
System.out.println(data); // data==42 보장
```

```
Thread A:   [data=42]  ----(Release: volatile write)---->  [flag=true]

Thread B:   [flag read] ----(Acquire: volatile read)---->  [data read]

```
- volatile write 이전의 일반 write들이 전파되도록 밀어내고

- 다른 스레드의 volatile read 이후 일반 read들이 그 값을 볼 수 있게 한다.

## 4) volatile이 보장하지 않는 것: 원자성(atomicity)

volatile은 단일 read/write 자체는 “잘 보이게” 하지만
count++처럼 여러 단계로 구성된 연산은 원자적이지 않다.

### * volatile int count; count++가 깨지는 이유
count++는 실제로는:
```java
temp = count;   // read
temp = temp + 1;
count = temp;   // write
```

두 스레드가 동시에 하면
```
초기 count = 0

T1: read count -> 0
T2: read count -> 0
T1: write count = 1
T2: write count = 1   (T1 증가분 덮어씀)

결과: 2번 증가했는데 최종 값은 1.
AtomicInteger.incrementAndGet() (CAS) 또는 synchronized로 해결 가능하다.
```

## 5) 언제 volatile을 쓰면 좋나? (대표 패턴 3개)
### 5-1) 종료 플래그(Stop Flag)
```java
class Worker {
    private volatile boolean running = true;

    public void stop() { running = false; }

    public void run() {
        while (running) {
            // 작업 수행
        }
    }
}

* running을 volatile로 안 두면, JIT가 while(running)을 최적화하며
“running은 안 바뀐다”고 보고 레지스터에 고정/hoist 해서 무한루프가 날 수 있다.
```

### 5-2) “값 공개(publication)” — 한 번 만들어 공유하는 참조
```java
class Holder {
    private volatile Config config;

    void init() {
        config = new Config(...); // 생성 + 내부 필드 셋업
    }

    Config get() {
        return config; // 다른 스레드에서 최신 참조를 안전하게 봄
    }
}

* config != null을 봤을 때 초기화가 덜 된 객체(반쯤 생성) 를 잡을 위험을 줄인다.
```

### 5-3) Double-Checked Locking (DCL) 싱글톤
```java
class Singleton {
    private static volatile Singleton instance;

    static Singleton getInstance() {
        Singleton r = instance;
        if (r == null) {
            synchronized (Singleton.class) {
                r = instance;
                if (r == null) {
                    r = new Singleton();
                    instance = r;
                }
            }
        }
        return r;
    }
}
* instance가 volatile이 아니면, 재정렬로 인해 다른 스레드가 생성 덜 된 인스턴스를 볼 수 있다.
```

## 6) volatile vs synchronized vs Atomic 비교
### 6-1) volatile
   무엇을 보장?

- ✅ 가시성(Visibility): 한 스레드의 write가 다른 스레드의 read에 “보이게” 됨

- ✅ 순서(Ordering): volatile write/read를 경계로 재정렬이 제한됨 (release/acquire)

- ❌ 원자성(Atomicity): count++ 같은 복합 연산은 깨짐

장점

- 아주 가벼움(락 없음)

- “신호 전달”에 최적: stop flag, ready flag, 상태 플래그

단점/주의

- 단일 변수 read/write는 되지만 복합 연산은 안전하지 않음

- 여러 변수의 불변식(“둘은 같이 바뀌어야 함”)은 보장하기 어려움

대표 사용처

- 종료 플래그 / 상태 플래그

- 초기화 완료 신호(ready)

- “한 번 쓰고 여러 번 읽는” 구성 값 공개(publication) (참조 volatile)

### 6-2) synchronized
   무엇을 보장?

- ✅ 원자성(상호배제): 임계구역 전체를 “한 덩어리”로 보호 (복합 연산 포함)

- ✅ 가시성 + 순서: unlock 이전의 변경이 lock 이후 다른 스레드에 보임 (happens-before)

- ✅ 조건 대기 협업: wait/notify/notifyAll 가능

장점

- 여러 줄/여러 변수/자료구조까지 정확하게 보호 가능 (불변식 유지에 최강)

- 구현이 직관적이고 실수 여지가 상대적으로 적음

- 조건 대기(생산자/소비자 등)에 유리

단점/주의

- 블로킹: 경합이 심하면 대기/문맥전환 비용이 커져 병목 가능

- 데드락 위험(락 순서 꼬임)

- tryLock/timeout 같은 세밀 제어는 기본 synchronized만으로 어려움(필요하면 Lock 고려)

대표 사용처

- 여러 필드가 함께 갱신되어야 하는 로직

- 공유 Map/List/캐시 업데이트

- “정확성 1순위”인 임계구역

- 조건 대기 필요 시

### 6-3) Atomic* (AtomicInteger/Long/Reference 등)
   무엇을 보장?

- ✅ 단일 변수에 대한 원자적 업데이트(CAS 기반): incrementAndGet, compareAndSet 등

- ✅ 가시성/순서(대체로 volatile급 메모리 의미): 읽기/쓰기가 다른 스레드에 안전하게 관측됨

- ✅ 비블로킹 성격: 락을 잡고 기다리지 않음(대부분)

장점

- 카운터/통계/상태 전이처럼 “단일 값 업데이트”에 매우 강함

- 경합이 적당한 환경에서 성능이 좋고, 락 홀더로 전체가 멈추는 리스크가 줄어듦

- AtomicReference로 불변 객체를 통째로 교체하는 패턴도 가능

단점/주의

- 기본은 “단일 값” 중심이라 여러 변수 불변식은 어렵다
→ 해결: synchronized로 묶거나, 불변 객체 + AtomicReference(CAS 교체) 패턴

- 경합이 매우 심하면 CAS 재시도로 CPU를 태울 수 있음
→ 고경합 카운터는 LongAdder가 더 나을 때 많음

- 상태기계/루프 설계가 복잡해질 수 있고(ABA 등) 실수 여지 있음

대표 사용처

- 카운터/메트릭/통계 집계

- CAS 기반 상태 전이(FSM)

- 락 없이 빠른 업데이트가 필요할 때

### [선택 기준]

- 신호만 주고받으면 된다 → <mark>volatile</mark>
(stop/ready 플래그, 단순 상태 표시)

- 여러 값/자료구조를 “같이” 맞춰야 한다 → <mark>synchronized</mark>
(불변식 유지, 복합 업데이트, 공유 컬렉션)

- 단일 값(카운터/상태)을 원자적으로 빠르게 갱신 → <mark>Atomic</mark>
(CAS 기반, 중간 경합에서 성능 좋음, 고경합은 LongAdder 고려)
