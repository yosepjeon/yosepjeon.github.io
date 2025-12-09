+++
title = "Chapter 06. 쿠버네티스 리소스 만들고 망가뜨리기"
date = "2025-12-01"
weight = 6
+++




# 6.1 pod의 라이프사이클 알기

```
                        Pod Lifecycle

    +-----------------------------------------------------------+
    |                                                           |
    |   Pending ──────> Running ──────> Succeeded               |
    |      │               │                                    |
    |      │               │                                    |
    |      │               ▼                                    |
    |      │            Failed                                  |
    |      │                                                    |
    |      └──────────> Unknown                                 |
    |                                                           |
    +-----------------------------------------------------------+

    Phase 설명:
    ┌─────────────┬────────────────────────────────────────────┐
    │ Pending     │ Pod이 승인되었지만 컨테이너가 아직 생성 안됨         │
    ├─────────────┼────────────────────────────────────────────┤
    │ Running     │ Pod이 노드에 바인딩, 모든 컨테이너 생성됨           │
    ├─────────────┼────────────────────────────────────────────┤
    │ Succeeded   │ 모든 컨테이너가 성공적으로 종료됨                  │
    ├─────────────┼────────────────────────────────────────────┤
    │ Failed      │ 모든 컨테이너 종료, 하나 이상 실패                 │
    ├─────────────┼────────────────────────────────────────────┤
    │ Unknown     │ Pod 상태를 알 수 없음 (노드 통신 오류)            │
    └─────────────┴────────────────────────────────────────────┘
```

### Container States

```
    ┌─────────────────────────────────────────────────────────┐
    │                    Container States                      │
    │                                                          │
    │    Waiting ──────> Running ──────> Terminated            │
    │       │               │                │                 │
    │       │               │                │                 │
    │       └───────────────┴────────────────┘                 │
    │                   (재시작 시)                              │
    └─────────────────────────────────────────────────────────┘
```

### Pod Phase vs STATUS 컬럼

**Pod Phase (라이프사이클)**는 5가지만 존재합니다:
- Pending, Running, Succeeded, Failed, Unknown

**STATUS 컬럼** (`kubectl get pods`)은 Phase + Container 상태 + 조건들을 조합한 **요약 정보**입니다. STATUS는 컨테이너의 `state`와 `reason` 필드에서 가져오며, 실제로 수십 가지가 존재합니다.

```
    ┌────────────────────┬─────────────┬──────────────────────────────────────┐
    │ STATUS 표시         │ 실제 Phase  │ 의미                                   │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ Pending            │ Pending     │ 스케줄링 대기 중                         │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ ContainerCreating  │ Pending     │ 컨테이너 생성 중                         │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ Init:0/1           │ Pending     │ Init 컨테이너 실행 중                    │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ PodInitializing    │ Pending     │ Init 컨테이너 완료, 메인 컨테이너 시작       │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ ImagePullBackOff   │ Pending     │ 이미지 풀 실패 (재시도 대기)               │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ ErrImagePull       │ Pending     │ 이미지 풀 실패 (즉시)                    │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ Running            │ Running     │ 정상 실행 중                            │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ CrashLoopBackOff   │ Running     │ 컨테이너 반복 크래시 (재시작 대기)           │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ Error              │ Running     │ 컨테이너 오류로 종료                      │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ OOMKilled          │ Running     │ 메모리 부족으로 강제 종료                  │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ Terminating        │ Running     │ 삭제 진행 중                            │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ Completed          │ Succeeded   │ 정상 종료 (Job 등)                      │
    ├────────────────────┼─────────────┼──────────────────────────────────────┤
    │ Unknown            │ Unknown     │ 노드와 통신 불가                         │
    └────────────────────┴─────────────┴──────────────────────────────────────┘
```

> **참고**: `ErrImagePull`과 `ImagePullBackOff`의 차이
> - `ErrImagePull`: 이미지 풀 첫 실패 시
> - `ImagePullBackOff`: 재시도 간격을 두고 대기 중 (exponential backoff)

```bash
# Phase 확인
$ kubectl get pod <name> -o jsonpath='{.status.phase}'

# 전체 상태 확인
$ kubectl describe pod <name>
```


# 6.2 Pod의 다중화를 위한 ReplicaSet과 Deployment
지금까지 Pod에 대해 학습하고 트러블 슈팅을 해봤습니다.
하지만, Pod만으로는 컨테이너 다중화가 불가능하기 때문에 실제 운영 환경에서는 권장하지 않습니다.
그래서 Deployment라는 리소스를 사용합니다.

## 6.2.1 ReplicaSet

ReplicaSet은 지정된 수의 Pod를 복제하는 리소스이며, 복제본이 항상 실행되도록 보장합니다.

```
                            ReplicaSet
    ┌─────────────────────────────────────────────────────────┐
    │                                                         │
    │   replicas: 3                                           │
    │   selector: app=nginx                                   │
    │                                                         │
    │   ┌─────────┐    ┌─────────┐    ┌─────────┐             │
    │   │  Pod 1  │    │  Pod 2  │    │  Pod 3  │             │
    │   │  nginx  │    │  nginx  │    │  nginx  │             │
    │   └─────────┘    └─────────┘    └─────────┘             │
    │                                                         │
    │   Pod이 죽으면 자동으로 새 Pod 생성                           │
    │                                                         │
    └─────────────────────────────────────────────────────────┘
```

ReplicaSet 매니페스트

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: httpserver
  labels:
    app: httpserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: httpserver
  template:
    metadata:
      labels:
        app: httpserver
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.3

        

```

```bash
$ kubectl apply --filename chapter-06/replicaset.yaml --namespace default
replicaset.apps/httpserver created
```

ReplicaSet은 동일한 pod를 복제하기 떄문에 pod의 이름에 임의의 접미사가 자동으로 붙습니다.  
ReplicaSet의 이름이 httpserver인 경우 httpserver-xxx가 됩니다.

```bash
$ kubectl get pod --namespace default
httpserver-2xfgb                1/1     Running   0          70s
httpserver-cfqgh                1/1     Running   0          70s
httpserver-gcvqw                1/1     Running   0          70s
```
  
다음 명령으로 ReplicaSet 리소스를 조회할 수 있습니다.
```bash
$ kubectl get replicaset --namespace default
* DESIRED: 몇 개의 POD가 생성돼야하는지
NAME                      DESIRED   CURRENT   READY   AGE
httpserver                3         3         3       5m53s
```  
  

## 6.2.2 Deployment

Deployment는 ReplicaSet을 관리하며, 롤링 업데이트와 롤백 기능을 제공합니다.

```
                              Deployment
    ┌───────────────────────────────────────────────────────────────┐
    │                                                               │
    │   strategy: RollingUpdate                                     │
    │   replicas: 3                                                 │
    │                                                               │
    │   ┌─────────────────────────────────────────────────────┐     │
    │   │                   ReplicaSet                        │     │
    │   │                                                     │     │
    │   │    ┌─────────┐   ┌─────────┐   ┌─────────┐          │     │
    │   │    │  Pod 1  │   │  Pod 2  │   │  Pod 3  │          │     │
    │   │    │  v1.0   │   │  v1.0   │   │  v1.0   │          │     │
    │   │    └─────────┘   └─────────┘   └─────────┘          │     │
    │   │                                                     │     │
    │   └─────────────────────────────────────────────────────┘     │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘

                         Rolling Update 과정
    ┌───────────────────────────────────────────────────────────────┐
    │                                                               │
    │   v1.0 → v2.0 업데이트 시                                        │
    │                                                               │
    │   1. 새 ReplicaSet 생성 (v2.0)                                  │
    │   2. 새 Pod 하나씩 생성                                          │
    │   3. 기존 Pod 하나씩 종료                                         │
    │   4. 모든 Pod이 v2.0이 될 때까지 반복                              │
    │                                                               │
    │   ReplicaSet (old)          ReplicaSet (new)                  │
    │   ┌─────────────────┐       ┌─────────────────┐               │
    │   │ Pod Pod Pod     │   →   │ Pod Pod Pod     │               │
    │   │ v1.0 v1.0 v1.0  │       │ v2.0 v2.0 v2.0  │               │
    │   └─────────────────┘       └─────────────────┘               │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
```
본격적인 운영환경에서는 단순히 pod를 복제하여 다중화하는 것뿐만 아니라,  
pod를 '무중단 업데이터'해야합니다.  
ReplicaSet이 만드는 pod의 컨테이너 이미지를 v1에서 v2로 변경하려면,  
v2의 pod를 생성하는 ReplicaSet을 새로 만들어야합니다.  
v1에서 v2로 원활하게 전환하려면 각 ReplicaSet을 연결해 주는 상위 개념이 필요합니다.  
이렇게 여러 ReplicaSet을 관리하는 리소스가 Deployment입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24.0
        ports:
        - containerPort: 80
```

해당 매니페스트로 Deployment 리소스를 생성합니다.
```bash
$ kubectl apply --filename deployment.yaml --namespace default
deployment.apps/nginx-deployment created
```
  
Deployment가 생성한 ReplicaSet과 Pod를 확인합니다.
```bash
$ kubectl get deployment --namespace default
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           2m12s
```

pod 목록을 확인합니다.
```bash
$ kubectl get pod --namespace default
NAME                                   READY   STATUS    RESTARTS   AGE
nginx-deployment-5c689d4b7b-abcde      1/1     Running   0          2m12s
nginx-deployment-5c689d4b7b-fghij      1/1     Running   0          2m12s
nginx-deployment-5c689d4b7b-klmno      1/1     Running   0          2m12s
```

```bash
$ kubectl get replicaset --namespace default
NAME                              DESIRED   CURRENT   READY   AGE
nginx-deployment-5c689d4b7b       3         3         3       2m18s
```

### * Deployment vs ReplicaSet

```
    ┌──────────────┬──────────────────────────────────────────────┐
    │ 기능          │ ReplicaSet         │ Deployment              │
    ├──────────────┼────────────────────┼─────────────────────────┤
    │ Pod 복제      │ ✅                  │ ✅ (ReplicaSet 통해)     │
    ├──────────────┼────────────────────┼─────────────────────────┤
    │ 자동 복구      │ ✅                  │ ✅                      │
    ├──────────────┼────────────────────┼─────────────────────────┤
    │ 롤링 업데이트   │ ❌                  │ ✅                      │
    ├──────────────┼────────────────────┼─────────────────────────┤
    │ 롤백          │ ❌                  │ ✅                      │
    ├──────────────┼────────────────────┼─────────────────────────┤
    │ 버전 히스토리   │ ❌                  │ ✅                      │
    └──────────────┴────────────────────┴─────────────────────────┘
```

> **실무에서는 ReplicaSet을 직접 사용하지 않고, Deployment를 통해 관리합니다.**

## Rolling Update 실습

이제 nginx 이미지를 1.24.0에서 1.25.3으로 업데이트해보겠습니다.

먼저 현재 Pod 목록을 확인합니다.
```bash
$ kubectl get pod --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5c689d4b7b-abcde   1/1     Running   0          2m12s
nginx-deployment-5c689d4b7b-fghij   1/1     Running   0          2m12s
nginx-deployment-5c689d4b7b-klmno   1/1     Running   0          2m12s
```

매니페스트의 `image: nginx:1.24.0`을 `image: nginx:1.25.3`으로 변경한 후 적용합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3  # 1.24.0 → 1.25.3으로 변경
        ports:
        - containerPort: 80
```

```bash
$ kubectl apply --filename deployment.yaml --namespace default
deployment.apps/nginx-deployment configured
```

업데이트 후 Pod 목록을 확인합니다.
```bash
$ kubectl get pod --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-576c6b7b6-pqrst    1/1     Running   0          30s
nginx-deployment-576c6b7b6-uvwxy    1/1     Running   0          28s
nginx-deployment-576c6b7b6-zabcd    1/1     Running   0          26s
```

### Pod 이름 변경 비교

```
    업데이트 전 (nginx:1.24.0)              업데이트 후 (nginx:1.25.3)
    ┌─────────────────────────────┐       ┌─────────────────────────────┐
    │ nginx-deployment-5c689d4b7b │       │ nginx-deployment-576c6b7b6  │
    │            ↓                │       │            ↓                │
    │   -abcde                    │   →   │   -pqrst                    │
    │   -fghij                    │       │   -uvwxy                    │
    │   -klmno                    │       │   -zabcd                    │
    └─────────────────────────────┘       └─────────────────────────────┘

    ReplicaSet 해시: 5c689d4b7b            ReplicaSet 해시: 576c6b7b6
```

Pod 이름의 구조: `{Deployment명}-{ReplicaSet해시}-{Pod해시}`

- **ReplicaSet 해시가 변경됨**: `5c689d4b7b` → `576c6b7b6`
  - 이미지 버전이 변경되면 새로운 ReplicaSet이 생성되기 때문
- **Pod 해시도 새로 생성됨**: 새 ReplicaSet에서 새 Pod가 생성되므로 접미사도 변경

ReplicaSet 목록을 확인하면 기존 ReplicaSet은 유지되지만 replica 수가 0이 됩니다.
```bash
$ kubectl get replicaset --namespace default
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5c689d4b7b   0         0         0       5m30s   # 기존 (1.24.0)
nginx-deployment-576c6b7b6    3         3         3       45s     # 신규 (1.25.3)
```

> **포인트**: 기존 ReplicaSet이 삭제되지 않고 남아있어 롤백이 가능합니다.

## Deployment 업데이트 전략 (Strategy)

Deployment의 매니페스트를 출력해보면, StrategyType과 RollingUpdate 설정이 보입니다.

```bash
$ kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Fri, 05 Dec 2025 11:27:51 +0900
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 0 available | 3 unavailable
** StrategyType:           RollingUpdate
MinReadySeconds:        0
** RollingUpdateStrategy:  25% max unavailable, 25% max surge
```
  

Deployment는 두 가지 업데이트 전략을 제공합니다.

```
    ┌─────────────────────────────────────────────────────────────────┐
    │                    Deployment Strategy                          │
    ├─────────────────────────────┬───────────────────────────────────┤
    │       RollingUpdate         │           Recreate                │
    │        (기본값)               │                                   │
    ├─────────────────────────────┼───────────────────────────────────┤
    │  점진적으로 Pod 교체            │  모든 Pod를 한번에 종료 후 재생성        │
    │  무중단 배포 가능               │  다운타임 발생                        │
    │  리소스 더 많이 사용             │  리소스 효율적                       │
    └─────────────────────────────┴───────────────────────────────────┘
```

### 1. RollingUpdate (기본값)

기존 Pod를 하나씩 종료하면서 새 Pod를 생성하여 무중단 배포를 수행합니다.

```
    RollingUpdate 과정 (replicas: 3)

    Step 1: 새 Pod 1개 생성
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ■ ■ ■    NEW: □                               │
    │       3개           1개 (생성 중)                      │
    └─────────────────────────────────────────────────────┘

    Step 2: 새 Pod Ready, 기존 Pod 1개 종료
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ■ ■ ✕    NEW: ■                               │
    │       2개           1개                              │
    └─────────────────────────────────────────────────────┘

    Step 3: 반복...
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ■ ✕ ✕    NEW: ■ ■                             │
    │       1개           2개                              │
    └─────────────────────────────────────────────────────┘

    Step 4: 완료
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ✕ ✕ ✕    NEW: ■ ■ ■                           │
    │       0개           3개                              │
    └─────────────────────────────────────────────────────┘
```

#### RollingUpdate 세부 설정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 최대 추가 Pod 수
      maxUnavailable: 0  # 최대 사용 불가 Pod 수
  template:
    # ... 생략
```

**maxSurge**: 업데이트 중 replicas 수를 초과하여 생성할 수 있는 최대 Pod 수  
**maxUnavailable**: 업데이트 중 사용 불가능한 상태로 둘 수 있는 최대 Pod 수

```
    ┌─────────────────┬───────────────────────────────────────────────┐
    │ 설정             │ 설명                                           │
    ├─────────────────┼───────────────────────────────────────────────┤
    │ maxSurge        │ 정수 또는 퍼센트 (예: 1 또는 "25%")                 │
    │                 │ replicas + maxSurge 만큼 Pod 생성 가능            │
    ├─────────────────┼───────────────────────────────────────────────┤
    │ maxUnavailable  │ 정수 또는 퍼센트 (예: 1 또는 "25%")                 │
    │                 │ 0이면 항상 replicas 수 유지 (안전)                 │
    └─────────────────┴───────────────────────────────────────────────┘
```

#### 25% 기본값 동작 예시

기본값인 `maxSurge: 25%`, `maxUnavailable: 25%`일 때의 동작을 살펴봅니다.

```
    replicas: 4 기준, maxSurge: 25%, maxUnavailable: 25%

    25% of 4 = 1 (올림 처리)

    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  maxSurge: 25% (= 1개)                                           │
    │  ─────────────────────────────────────────────                  │
    │  replicas(4) + 1 = 최대 5개까지 Pod 동시 존재 가능                    │
    │                                                                 │
    │       기존 Pod      추가 가능                                      │
    │     ■ ■ ■ ■        □                                            │
    │     (4개)          (+1개)                                        │
    │     └────────────────┘                                          │
    │         최대 5개                                                  │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  maxUnavailable: 25% (= 1개)                                     │
    │  ─────────────────────────────────────────────                  │
    │  replicas(4) - 1 = 최소 3개는 항상 Running 상태 유지                  │
    │                                                                 │
    │       사용 가능      사용 불가 허용                                   │
    │     ■ ■ ■          ✕                                             │
    │     (최소 3개)       (1개까지)                                      │
    │     └──────────────┘                                            │
    │       항상 유지                                                   │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

```
    실제 Rolling Update 과정 (replicas: 4, 25%/25%)

    초기 상태
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ■ ■ ■ ■      NEW: (없음)                       │
    │       4개 Running                                   │
    └─────────────────────────────────────────────────────┘

    Step 1: 새 Pod 1개 생성 (maxSurge=1), 기존 1개 종료 시작 (maxUnavailable=1)
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ■ ■ ■ ✕      NEW: □                           │
    │       3개 Running       1개 생성 중                    │
    │       1개 종료 중                                     │
    │  총 Running: 3개 (≥ 4-1), 총 Pod: 5개 (≤ 4+1) ✓       │
    └─────────────────────────────────────────────────────┘

    Step 2: 새 Pod Ready, 다음 교체 진행
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ■ ■ ✕ ✕      NEW: ■ □                         │
    │       2개 Running       1개 Running                  │
    │                         1개 생성 중                   │
    │  총 Running: 3개 ✓                                   │
    └─────────────────────────────────────────────────────┘

    Step 3~4: 반복...

    완료
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ✕ ✕ ✕ ✕      NEW: ■ ■ ■ ■                     │
    │       0개                4개 Running                 │
    └─────────────────────────────────────────────────────┘
```

> **참고**: 퍼센트 계산 시 `maxSurge`는 올림, `maxUnavailable`은 내림 처리됩니다.
> - replicas: 4, maxSurge: 25% → 4 × 0.25 = 1 (올림)
> - replicas: 4, maxUnavailable: 25% → 4 × 0.25 = 1 (내림)

#### 설정 예시 비교

```
    replicas: 4 기준

    예시 1: maxSurge=1, maxUnavailable=0 (안전한 배포)
    ┌─────────────────────────────────────────────────────┐
    │ 동작: 새 Pod 먼저 생성 → Ready 후 기존 Pod 종료            │
    │ 최소 Pod: 4개 (항상 유지)                               │
    │ 최대 Pod: 5개 (4 + 1)                                 │
    │ 특징: 무중단, 리소스 더 사용                              │
    └─────────────────────────────────────────────────────┘

    예시 2: maxSurge=0, maxUnavailable=1 (리소스 절약)
    ┌─────────────────────────────────────────────────────┐
    │ 동작: 기존 Pod 먼저 종료 → 새 Pod 생성                    │
    │ 최소 Pod: 3개 (4 - 1)                                │
    │ 최대 Pod: 4개                                        │
    │ 특징: 순간 용량 감소, 리소스 효율적                         │
    └─────────────────────────────────────────────────────┘

    예시 3: maxSurge=2, maxUnavailable=1 (빠른 배포)
    ┌─────────────────────────────────────────────────────┐
    │ 동작: 동시에 여러 Pod 교체                               │
    │ 최소 Pod: 3개 (4 - 1)                                 │
    │ 최대 Pod: 6개 (4 + 2)                                 │
    │ 특징: 빠른 배포, 리소스 많이 사용                           │
    └─────────────────────────────────────────────────────┘
```

### 2. Recreate

모든 기존 Pod를 먼저 종료한 후 새 Pod를 생성합니다. 다운타임이 발생하지만 리소스를 효율적으로 사용합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: Recreate  # RollingUpdate 설정 불필요
  template:
    # ... 생략
```

```
    Recreate 과정 (replicas: 3)

    Step 1: 모든 기존 Pod 종료
    ┌─────────────────────────────────────────────────────┐
    │  OLD: ✕ ✕ ✕    NEW: (없음)                           │
    │       ↑ 다운타임 발생!                                 │
    └─────────────────────────────────────────────────────┘

    Step 2: 새 Pod 모두 생성
    ┌─────────────────────────────────────────────────────┐
    │  OLD: (없음)    NEW: ■ ■ ■                           │
    │                     서비스 복구                        │
    └─────────────────────────────────────────────────────┘
```

#### 언제 Recreate를 사용하나요?

```
    ┌─────────────────────────────────────────────────────────────────┐
    │ Recreate 전략이 적합한 경우                                          │
    ├─────────────────────────────────────────────────────────────────┤
    │ • 구버전과 신버전이 동시에 실행되면 안 되는 경우                           │
    │   (예: DB 스키마 변경, 파일 잠금 등)                                  │
    │                                                                 │
    │ • 개발/테스트 환경에서 빠른 배포가 필요한 경우                             │
    │                                                                 │
    │ • 리소스가 제한적이어서 추가 Pod를 생성할 여유가 없는 경우                   │
    └─────────────────────────────────────────────────────────────────┘
```

### 전략 비교 요약

```
    ┌────────────────┬─────────────────────┬─────────────────────┐
    │                │ RollingUpdate       │ Recreate            │
    ├────────────────┼─────────────────────┼─────────────────────┤
    │ 다운타임        │ 없음                 │ 있음                  │
    ├────────────────┼─────────────────────┼─────────────────────┤
    │ 배포 속도       │ 느림 (점진적)         │ 빠름 (한번에)            │
    ├────────────────┼─────────────────────┼─────────────────────┤
    │ 리소스 사용     │ 더 많음              │ 효율적                  │
    ├────────────────┼─────────────────────┼─────────────────────┤
    │ 버전 공존       │ 가능 (잠시 동안)      │ 불가능                  │
    ├────────────────┼─────────────────────┼─────────────────────┤
    │ 롤백 용이성     │ 용이함               │ 용이함                  │
    ├────────────────┼─────────────────────┼─────────────────────┤
    │ 권장 환경       │ 프로덕션             │ 개발/테스트              │
    └────────────────┴─────────────────────┴─────────────────────┘
```

## 실습
Recreate 매니패스트를 작성하고 적용합니다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 10
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24.0
        ports:
        - containerPort: 80
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]

```

```bash
$ kubectl apply --filename deployment-recreate.yaml --namespace default
deployment.apps/nginx-deployment created
```

pod가 정상적으로 생성되었는지 확인합니다.

```bash
$ kubectl get pod --namespace default
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-bbf758c66-5d55w   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-6lc65   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-dfm6h   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-f5hvt   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-fg85x   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-hnbm5   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-m9qqn   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-mfkwh   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-tklng   1/1     Running   0          4m51s
nginx-deployment-bbf758c66-vvvs8   1/1     Running   0          4m51s
```

<mark>kubectl get pod --watch</mark>를 실행해서 관찰합니다.  
<mark>--watch</mark> 옵션을 붙이면 <mark>kubectl get pod</mark>의 결과를 계속해서 모니터링할 수 있습니다.

이제 image를 nginx:1.25.3으로 변경합니다.

```bash
kubectl get pod --watch --namespace default
NAME                               READY   STATUS               RESTARTS   AGE
nginx-deployment-bbf758c66-5d55w   1/1     Running              0          72m
nginx-deployment-bbf758c66-6lc65   1/1     Running              0          72m
nginx-deployment-bbf758c66-dfm6h   1/1     Running              0          72m
...
nginx-deployment-bbf758c66-f5hvt   1/1     Terminating          0          72m
nginx-deployment-bbf758c66-fg85x   1/1     Terminating          0          72m
nginx-deployment-bbf758c66-vvvs8   1/1     Terminating          0          72m
...
nginx-deployment-6f4857f9c6-5k2mw   0/1     Pending             0          0s
nginx-deployment-6f4857f9c6-c88j4   0/1     Pending             0          0s
nginx-deployment-6f4857f9c6-285f9   0/1     Pending             0          0s
nginx-deployment-6f4857f9c6-p9jcz   0/1     Pending             0          0s
...
nginx-deployment-6f4857f9c6-c88j4   0/1     ContainerCreating   0          0s
nginx-deployment-6f4857f9c6-285f9   0/1     ContainerCreating   0          0s
nginx-deployment-6f4857f9c6-pgmdg   0/1     ContainerCreating   0          0s
...
nginx-deployment-bbf758c66-5d55w    0/1     Completed           0          72m
nginx-deployment-bbf758c66-5d55w    0/1     Completed           0          72m
nginx-deployment-bbf758c66-mfkwh    0/1     Completed           0          72m
...
nginx-deployment-6f4857f9c6-pgmdg   1/1     Running             0          8s
nginx-deployment-6f4857f9c6-9fwxw   1/1     Running             0          10s
nginx-deployment-6f4857f9c6-c88j4   1/1     Running             0          11s
...
```

pod가 Terminating -> ContainerCreating -> Running으로 전환되는것을 확인할 수 있습니다.
  
이제 strategy를 RollingUpdate로 바꾸겠습니다.

```bash
$ kubectl apply --filename deployment-rollingupdate.yaml --namespace default
deployment.apps/nginx-deployment created
```

업데이트가 진행되면서 pod의 수가 두 배로 늘어나는 것을 볼 수 있습니다.  
max surge가 100%면 기존 pod의 수와 동일한 수의 새로운 pod를 생성합니다.  
max surge 100%는 pod를 가장 빠르고 안전하게 업데이트하는 방법입니다.  
하지만 필요한 리소스가 일시적으로 두 배로 늘어나기 때문에 가용 리소스가 충분한지 미리 확인해야 합니다.

관련 파드를 삭제함으로 테스트를 완료합니다.
```bash
$ kubectl delete --filename deployment-rollingupdate.yaml --namespace default
```

## 6.2.3 Deployment를 만들고 망가뜨리기
```bash
$ kubectl apply --filename deployment-hello-server.yaml --namespace default
deployment.apps/hello-server created

$ kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-84d5bd575b-5zmfj   1/1     Running   0          48s
hello-server-84d5bd575b-8lx5s   1/1     Running   0          48s
hello-server-84d5bd575b-vxhqc   1/1     Running   0          48s
```

먼저 pod를 삭제하면 어떻게 되는지 확인해 보겠습니다.

```bash
$ kubectl delete pod hello-server-84d5bd575b-5zmfj --namespace default 
pod "hello-server-84d5bd575b-5zmfj" deleted from default namespace
$ chapter-06 % kubectl get pod --namespace default                                 
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-84d5bd575b-plzn6   1/1     Running   0          2s
hello-server-84d5bd575b-8lx5s   1/1     Running   0          6h35m
hello-server-84d5bd575b-vxhqc   1/1     Running   0          6h35m
```

하나의 pod만 AGE가 짧은 것이 보입니다. 삭제된 pod를 대신해 새로운 pod가 만들어졌습니다.  
Deployment를 사용하면 pod가 지워지더라도 지정한 개수에 맞게 쿠버네티스가 자동으로 다시 생성합니다.  
  
이제 Deplyment를 RollingUpdate를 해보겠습니다. 먼저 현재 서버가 문제없이 동작하는지 확인합니다.
```bash
$ kubectl port-forward deployment/hello-server 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

$ curl localhost:8080
Hello, world!%             
```
  
그다음 매니페스트를 적용하여 RollingUpdate를 진행합니다.
```bash
$ kubectl apply --filename deployment-hello-server-rollingupdate.yaml --namespace default
deployment.apps/hello-server configured

$ kubectl get pod --namespace default
NAME                            READY   STATUS         RESTARTS   AGE
hello-server-84cc6c9ccf-tvvr4   0/1     ErrImagePull   0          34s
hello-server-84d5bd575b-8lx5s   1/1     Running        0          6h46m
hello-server-84d5bd575b-plzn6   1/1     Running        0          10m
hello-server-84d5bd575b-vxhqc   1/1     Running        0          6h46m
```

pod가 하나 늘어났지만 오류가 발생한 것을 알 수 있습니다. 애플리케이션이 동작하지 않는 것은 아닙니다.
fortforward 상태로 curl 명령어를 호출하면 정상동작합니다.

```bash
$ kubectl get deployment --namespace default
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-server   3/3     1            3           6h49m
```

UP-TO-DATE가 1입니다. 오래된 버전의 pod는 그대로 두고, 새로운 버전의 pod를 1개 생성하던 중 오류가 발생했음을 알 수 있습니다.
기본 설정 maxUnavailable: 25%, maxSurge: 25%일 때 pod의 개수 3의 25%는 0.75입니다.
소수점일떄 maxUnavailable은 내림, maxSurge는 올림을 적용합니다.
따라서 여기서는 maxUnavailable: 0, maxSurge: 1입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Rolling Update 전략 (Pod 3개)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   기본 설정: maxUnavailable: 25%, maxSurge: 25%                       │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  3 × 25% = 0.75                                             │   │
│   │                                                             │   │
│   │  • maxUnavailable: 0.75 → 0 (내림)  → 최소 3개 유지 필요         │   │
│   │  • maxSurge:       0.75 → 1 (올림)  → 최대 4개까지 허용          │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                         업데이트 과정                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   [초기 상태] READY: 3, UP-TO-DATE: 3                                 │
│   ┌───────┐  ┌───────┐  ┌───────┐                                   │
│   │ Pod 1 │  │ Pod 2 │  │ Pod 3 │                                   │
│   │  v1   │  │  v1   │  │  v1   │                                   │
│   │  ✓    │  │  ✓    │  │  ✓    │                                   │
│   └───────┘  └───────┘  └───────┘                                   │
│                                                                     │
│                          ↓ 업데이트 시작                               │
│                                                                     │
│   [업데이트 중] READY: 3, UP-TO-DATE: 1                                │
│   ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐                        │
│   │ Pod 1 │  │ Pod 2 │  │ Pod 3 │  │ Pod 4 │  ← maxSurge: 1         │
│   │  v1   │  │  v1   │  │  v1   │  │  v2   │    (새 버전 1개 추가)     │
│   │  ✓    │  │  ✓    │  │  ✓    │  │  ⚠️   │    (생성 중/오류)         │
│   └───────┘  └───────┘  └───────┘  └───────┘                        │
│                                                                     │
│   ※ maxUnavailable: 0 이므로 기존 Pod를 삭제할 수 없음                     │
│   ※ 새 Pod가 Ready 상태가 되어야 기존 Pod 삭제 가능                         │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                         오류 발생 시                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   새 Pod(v2)가 Ready 상태가 되지 않으면:                                  │
│   • 기존 v1 Pod 3개는 계속 서비스 유지 (maxUnavailable: 0)                │
│   • 새 v2 Pod는 오류 상태로 대기                                         │
│   • 롤아웃이 중단된 상태로 유지                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

애플리케이션이 정상적으로 동작하는 것은 이전 버전의 pod가 그대로 남아 있기 때문입니다.
```bash
$ kubectl get replicaset --namespace default
NAME                      DESIRED   CURRENT   READY   AGE
hello-server-84cc6c9ccf   1         1         0       6h58m
hello-server-84d5bd575b   3         3         3       13h
```

그럼 이제 RollingUpdate를 고쳐 보겠습니다. STATUS가 ErrImagePull인데 상세 내용을 확인하겠습니다.
```bash
$ kubectl describe pod hello-server-84cc6c9ccf --namespace default
Name:             hello-server-84cc6c9ccf-tvvr4
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-control-plane/172.24.0.2
Start Time:       Tue, 09 Dec 2025 00:04:09 +0900
Labels:           app=hello-server
                  pod-template-hash=84cc6c9ccf

...


Events:
  Type     Reason   Age                 From     Message
  ----     ------   ----                ----     -------
  Normal   BackOff  57s (x419 over 7h)  kubelet  Back-off pulling image "blux2/hello-server:1.3"
  Warning  Failed   57s (x419 over 7h)  kubelet  Error: ImagePullBackOff
```
hello-server:1.3을 찾을 수 없다고 출력되었습니다. 따라서 1.2태그를 사용하도록 수정합니다.

```bash
$ kubectl edit deployment hello-server --namespace default
deployment.apps/hello-server edited

$ kubectl get pod,replicaset --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
pod/hello-server-77665655cd-2l5pt   1/1     Running   0          19s
pod/hello-server-77665655cd-bjvql   1/1     Running   0          11s
pod/hello-server-77665655cd-wpzqv   1/1     Running   0          12s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-server-77665655cd   3         3         3       19s
replicaset.apps/hello-server-84cc6c9ccf   0         0         0       7h5m
replicaset.apps/hello-server-84d5bd575b   0         0         0       13h

$ kubectl port-forward hello-server 8080:8080 --namespace default
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

별도의 터미널에서 수행
$ curl localhost:8080
Hello, world! Let's learn Kubernetes!

실습 정리
$ kubectl delete --filename deployment-hello-server-rollingupdate.yaml
deployment.apps "hello-server" deleted from default namespace
```

# 6.3 pod로의 접속을 도와주는 Service
Deplyment는 IP 주소를 가지지 않기 때문에 접근하려면 각 pod에 할당된 IP로 접근해야 합니다.  
그러면 RollingUpdate 기능을 사용한다고 해도 접속 중인 pod가 지워지면 연결이 끊어지게 됩니다.  
Deployment로 생성한 여러 pod에게로 적절히 라우팅하기 위해 service라는 리소스를 사용합니다.  
서비스의 매니패스트입니다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
  labels:
    app: hello-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-server
  template:
    metadata:
      labels:
        app: hello-server
    spec:
      containers:
      - name: hello-server
        image: blux2/hello-server:1.0
        ports:
        - containerPort: 8080
```

service 만으로는 동작을 확인할 수 없기 때문에 service에 연결된 Deployment도 만들어 동작을 확인합니다.  
```bash
$ kubectl apply --filename deployment-hello-server.yaml --namespace default
deployment.apps/hello-server created

// pod가 생성되었는지 확인
$ kubectl get pod --namespace default
NAME                               READY   STATUS             RESTARTS   AGE
hello-server-84d5bd575b-9r2md      1/1     Running            0          40s
hello-server-84d5bd575b-mbzjb      1/1     Running            0          40s
hello-server-84d5bd575b-znf8f      1/1     Running            0          40s

// service 리소스 생성
$ kubectl apply --filename service.yaml --namespace default
service/hello-server-service created

// 서비스 생성 확인
$ kubectl get service hello-server-service --namespace default
NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
hello-server-service   ClusterIP   10.96.16.1   <none>        8080/TCP   33s

// port-forward로 동작 확인
$ kubectl port-forward svc/hello-server-service 8080:8080 --namespace default 
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080


// curl 동작확인
$ curl localhost:8080
Hello, world!
```

## 6.3.1 Service의 Type 알기

Service는 4가지 Type을 제공하며, 각 Type에 따라 접근 범위와 방식이 달라집니다.

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │                        Service Types 개요                           │
    ├─────────────┬───────────────────────────────────────────────────────┤
    │ ClusterIP   │ 클러스터 내부에서만 접근 가능 (기본값)                      │
    ├─────────────┼───────────────────────────────────────────────────────┤
    │ NodePort    │ 클러스터 외부에서 노드IP:포트로 접근 가능                   │
    ├─────────────┼───────────────────────────────────────────────────────┤
    │ LoadBalancer│ 클라우드 로드밸런서를 통해 외부 접근                        │
    ├─────────────┼───────────────────────────────────────────────────────┤
    │ ExternalName│ 외부 DNS 이름을 클러스터 내부 서비스로 매핑                  │
    └─────────────┴───────────────────────────────────────────────────────┘
```

### 1. ClusterIP (기본값)

클러스터 내부에서만 접근할 수 있는 가상 IP를 할당합니다. 외부에서는 직접 접근할 수 없습니다.

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │                         Kubernetes Cluster                          │
    │                                                                     │
    │    ┌───────────────────────────────────────────────────────────┐    │
    │    │                     ClusterIP Service                     │    │
    │    │                   (10.96.16.1:8080)                       │    │
    │    └───────────────────────────────────────────────────────────┘    │
    │                              │                                      │
    │              ┌───────────────┼───────────────┐                      │
    │              ▼               ▼               ▼                      │
    │        ┌─────────┐     ┌─────────┐     ┌─────────┐                  │
    │        │  Pod 1  │     │  Pod 2  │     │  Pod 3  │                  │
    │        │ :8080   │     │ :8080   │     │ :8080   │                  │
    │        └─────────┘     └─────────┘     └─────────┘                  │
    │                                                                     │
    │    ※ 클러스터 내부의 다른 Pod에서만 접근 가능                             │
    │    ※ 외부에서 접근하려면 kubectl port-forward 필요                      │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘

              ✕ 외부 접근 불가
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-server-service
spec:
  type: ClusterIP  # 기본값이므로 생략 가능
  selector:
    app: hello-server
  ports:
    - protocol: TCP
      port: 8080        # Service 포트
      targetPort: 8080  # Pod 포트
```

**사용 사례**
- 클러스터 내부 마이크로서비스 간 통신
- 외부에 노출할 필요 없는 백엔드 서비스 (DB, 캐시 등)

### 2. NodePort

ClusterIP 기능에 더해, 모든 노드의 특정 포트를 열어 외부에서 접근할 수 있게 합니다.

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │                                                                     │
    │     외부 클라이언트                                                   │
    │           │                                                         │
    │           │ <NodeIP>:30080                                          │
    │           ▼                                                         │
    │    ┌─────────────────────────────────────────────────────────────┐  │
    │    │                    Kubernetes Cluster                       │  │
    │    │                                                             │  │
    │    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │  │
    │    │  │   Node 1    │  │   Node 2    │  │   Node 3    │          │  │
    │    │  │  :30080     │  │  :30080     │  │  :30080     │          │  │
    │    │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │  │
    │    │         │                │                │                 │  │
    │    │         └────────────────┼────────────────┘                 │  │
    │    │                          ▼                                  │  │
    │    │         ┌────────────────────────────────────┐              │  │
    │    │         │         NodePort Service           │              │  │
    │    │         │  ClusterIP: 10.96.16.1:8080        │              │  │
    │    │         │  NodePort: 30080                   │              │  │
    │    │         └────────────────────────────────────┘              │  │
    │    │                          │                                  │  │
    │    │          ┌───────────────┼───────────────┐                  │  │
    │    │          ▼               ▼               ▼                  │  │
    │    │    ┌─────────┐     ┌─────────┐     ┌─────────┐              │  │
    │    │    │  Pod 1  │     │  Pod 2  │     │  Pod 3  │              │  │
    │    │    │ :8080   │     │ :8080   │     │ :8080   │              │  │
    │    │    └─────────┘     └─────────┘     └─────────┘              │  │
    │    │                                                             │  │
    │    └─────────────────────────────────────────────────────────────┘  │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-server-nodeport
spec:
  type: NodePort
  selector:
    app: hello-server
  ports:
    - protocol: TCP
      port: 8080        # Service 포트 (클러스터 내부)
      targetPort: 8080  # Pod 포트
      nodePort: 30080   # 노드 포트 (30000-32767 범위)
```

```
    ┌─────────────────┬───────────────────────────────────────────────┐
    │ 포트             │ 설명                                          │
    ├─────────────────┼───────────────────────────────────────────────┤
    │ port            │ Service의 ClusterIP에서 사용하는 포트            │
    ├─────────────────┼───────────────────────────────────────────────┤
    │ targetPort      │ 실제 Pod 컨테이너가 리스닝하는 포트               │
    ├─────────────────┼───────────────────────────────────────────────┤
    │ nodePort        │ 모든 노드에서 외부 접근용으로 열리는 포트           │
    │                 │ 범위: 30000-32767 (지정 안하면 자동 할당)         │
    └─────────────────┴───────────────────────────────────────────────┘
```

**사용 사례**
- 개발/테스트 환경에서 간단한 외부 접근
- 로드밸런서가 없는 온프레미스 환경

**주의사항**
- 모든 노드에서 해당 포트가 열리므로 보안에 주의
- 프로덕션에서는 LoadBalancer나 Ingress 권장

### 3. LoadBalancer

클라우드 프로바이더(AWS, GCP, Azure 등)의 로드밸런서를 자동으로 프로비저닝하여 외부 트래픽을 받습니다.

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │                                                                     │
    │     외부 클라이언트                                                   │
    │           │                                                         │
    │           │ http://my-app.example.com                               │
    │           ▼                                                         │
    │    ┌─────────────────────────────────────────────────────────────┐  │
    │    │              Cloud Load Balancer                            │  │
    │    │           (External IP: 203.0.113.10)                       │  │
    │    └─────────────────────────────────────────────────────────────┘  │
    │                          │                                          │
    │                          ▼                                          │
    │    ┌─────────────────────────────────────────────────────────────┐  │
    │    │                    Kubernetes Cluster                       │  │
    │    │                                                             │  │
    │    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │  │
    │    │  │   Node 1    │  │   Node 2    │  │   Node 3    │          │  │
    │    │  │  :30080     │  │  :30080     │  │  :30080     │          │  │
    │    │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │  │
    │    │         │                │                │                 │  │
    │    │         └────────────────┼────────────────┘                 │  │
    │    │                          ▼                                  │  │
    │    │         ┌────────────────────────────────────┐              │  │
    │    │         │       LoadBalancer Service         │              │  │
    │    │         │  ClusterIP: 10.96.16.1:8080        │              │  │
    │    │         │  External: 203.0.113.10:80         │              │  │
    │    │         └────────────────────────────────────┘              │  │
    │    │                          │                                  │  │
    │    │          ┌───────────────┼───────────────┐                  │  │
    │    │          ▼               ▼               ▼                  │  │
    │    │    ┌─────────┐     ┌─────────┐     ┌─────────┐              │  │
    │    │    │  Pod 1  │     │  Pod 2  │     │  Pod 3  │              │  │
    │    │    └─────────┘     └─────────┘     └─────────┘              │  │
    │    │                                                             │  │
    │    └─────────────────────────────────────────────────────────────┘  │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-server-lb
spec:
  type: LoadBalancer
  selector:
    app: hello-server
  ports:
    - protocol: TCP
      port: 80          # 외부 로드밸런서 포트
      targetPort: 8080  # Pod 포트
```

```bash
$ kubectl get service hello-server-lb --namespace default
NAME              TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
hello-server-lb   LoadBalancer   10.96.16.1    203.0.113.10    80:30080/TCP   1m
```

**사용 사례**
- 프로덕션 환경에서 외부 트래픽 처리
- 클라우드 환경에서 안정적인 외부 접근 필요 시

**주의사항**
- 클라우드 프로바이더 비용 발생 (서비스당 로드밸런서 1개)
- 온프레미스에서는 MetalLB 같은 추가 구성 필요
- 여러 서비스가 있으면 Ingress 사용 권장

### 4. ExternalName

클러스터 내부에서 외부 서비스를 DNS 이름으로 참조할 때 사용합니다. 실제 라우팅이 아닌 DNS CNAME 레코드를 반환합니다.

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │                       Kubernetes Cluster                            │
    │                                                                     │
    │    ┌───────────┐                                                    │
    │    │   Pod     │                                                    │
    │    │           │──── "external-db" 요청                              │
    │    └───────────┘          │                                         │
    │                           ▼                                         │
    │         ┌────────────────────────────────────┐                      │
    │         │       ExternalName Service         │                      │
    │         │  name: external-db                 │                      │
    │         │  externalName: db.example.com      │                      │
    │         └────────────────────────────────────┘                      │
    │                           │                                         │
    │                           │ DNS CNAME 반환                           │
    │                           ▼                                         │
    └───────────────────────────┼─────────────────────────────────────────┘
                                │
                                ▼
                   ┌────────────────────────┐
                   │   외부 서비스            │
                   │   db.example.com       │
                   └────────────────────────┘
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com  # 외부 DNS 이름
```

**사용 사례**
- 외부 데이터베이스(RDS, Cloud SQL 등) 연결
- 외부 API 서비스 참조
- 클러스터 마이그레이션 시 기존 서비스 참조

**특징**
- ClusterIP가 할당되지 않음
- selector도 사용하지 않음
- 단순히 DNS CNAME 레코드만 반환

### Service Type 비교 요약

```
    ┌────────────────┬────────────┬────────────┬──────────────┬──────────────┐
    │                │ ClusterIP  │ NodePort   │ LoadBalancer │ ExternalName │
    ├────────────────┼────────────┼────────────┼──────────────┼──────────────┤
    │ 외부 접근       │ ✕          │ ✓          │ ✓            │ -            │
    ├────────────────┼────────────┼────────────┼──────────────┼──────────────┤
    │ ClusterIP 할당 │ ✓          │ ✓          │ ✓            │ ✕            │
    ├────────────────┼────────────┼────────────┼──────────────┼──────────────┤
    │ 노드 포트 오픈  │ ✕          │ ✓          │ ✓            │ ✕            │
    ├────────────────┼────────────┼────────────┼──────────────┼──────────────┤
    │ 외부 LB 생성   │ ✕          │ ✕          │ ✓            │ ✕            │
    ├────────────────┼────────────┼────────────┼──────────────┼──────────────┤
    │ 클라우드 비용   │ 없음        │ 없음        │ 있음          │ 없음          │
    ├────────────────┼────────────┼────────────┼──────────────┼──────────────┤
    │ 주요 용도       │ 내부 통신   │ 개발/테스트 │ 프로덕션      │ 외부 서비스   │
    └────────────────┴────────────┴────────────┴──────────────┴──────────────┘
```

```
    Service Type 계층 구조

    ┌─────────────────────────────────────────────────────────────────┐
    │                        LoadBalancer                             │
    │  ┌───────────────────────────────────────────────────────────┐  │
    │  │                       NodePort                            │  │
    │  │  ┌─────────────────────────────────────────────────────┐  │  │
    │  │  │                    ClusterIP                        │  │  │
    │  │  │                                                     │  │  │
    │  │  │              클러스터 내부 접근                        │  │  │
    │  │  │                                                     │  │  │
    │  │  └─────────────────────────────────────────────────────┘  │  │
    │  │                  + 노드 포트 오픈                          │  │
    │  └───────────────────────────────────────────────────────────┘  │
    │                    + 외부 로드밸런서                             │
    └─────────────────────────────────────────────────────────────────┘

    ※ LoadBalancer는 NodePort를 포함하고, NodePort는 ClusterIP를 포함합니다.
```