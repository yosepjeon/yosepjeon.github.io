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
kubectl get pod <name> -o jsonpath='{.status.phase}'

# 전체 상태 확인
kubectl describe pod <name>
```


# 6.2 Pod의 다중화를 위한 ReplicaSet과 Deployment
지금까지 Pod에 대해 학습하고 트러블 슈팅을 해봤습니다.
하지만, Pod만으로는 컨테이너 다중화가 불가능하기 때문에 실제 운영 환경에서는 권장하지 않습니다.
그래서 Deployment라는 리소스를 사용합니다.

## ReplicaSet

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
  

## Deployment

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
$kubectl describe deployment nginx-deployment
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


