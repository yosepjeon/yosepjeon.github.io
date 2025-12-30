+++
title = "Chapter 10. 쿠버네티스 개발 워크플로 이해하기"
date = "2025-12-22"
weight = 10
+++

# 10.1 쿠버네티스에 배포하기
지금까지의 실습에서는 쿠버네티스에 매니페스트를 배포하기 위해 `kubectl apply --filename`과 같은 명령어를 사용했습니다. 
그런데 지속적인 배포를 생각하면 문제점이 있습니다.
* 누가 언제 실행했는지 알 수 없다.
* 명령어 실행으로 매니페스트 충돌이 발생할 수 있다.
* 매번 수동으로 배포하는 것은 번거롭고, 사람의 실수가 발생하기 쉽다.

이러한 문제점을 해결하기위해 push기반의 CIOps와 pull기반의 GitOps가 등장했습니다. 두가지 모두 깃허브와 같은 공유 리포지터리를 사용하여 
매니페스트의 충돌 및 차이점을 관리합니다.  

## 10.1.1 Push형 배포 방법: CIOps

```
┌─────────┐     ┌────────────────┐     ┌─────────────┐
│  개발자  │────▶│ Git Repository │────▶│ CI Pipeline │
└─────────┘     └────────────────┘     └──────┬──────┘
  1.코드 Push        2.트리거                    │
                                              │ 3.빌드 & 테스트
                                              ▼
┌─────────────────────┐              ┌──────────────────────┐
│ Kubernetes Cluster  │◀─────────────│  Container Registry  │
└─────────────────────┘              └──────────────────────┘
        ▲                                     ▲
        │ 5.kubectl apply                     │ 4.이미지 Push
        └─────────────────────────────────────┘
                    CI Pipeline에서 직접 배포
```

`kubectl apply --filename <파일 이름>`을 자동화하는 가장 직관적인 방법은 가령 master 브랜치에 feature 브랜치가 병합될 때 자동으로 kubectl apply가 실행되도록 하는 것입니다.

### CIOps의 장/단점
#### 장점
* 이해하기 쉽고 구축하기도 쉽다.

#### 단점
* CI/CD용 도구에 강력한 권한이 필요하다.
* 배포용 스크립트가 길고 복잡해지기 쉽다.

## 10.1.2 Pull형 배포 방법: GitOps

```
┌─────────┐     ┌────────────────┐     ┌─────────────┐
│  개발자   │────▶│ Git Repository │────▶│ CI Pipeline │
└─────────┘     └────────────────┘     └──────┬──────┘
  1.코드 Push         ▲                        │
                     │ 3.Poll                 │ 2.빌드 & 테스트
                     │  (주기적 감시)            ▼
┌────────────────────┴───────┐       ┌──────────────────────┐
│    Kubernetes Cluster      │       │  Container Registry  │
│  ┌──────────────────────┐  │◀──────└──────────────────────┘
│  │  GitOps Operator     │  │         4.이미지 Push
│  │  (ArgoCD, Flux 등)    │  │
│  └──────────────────────┘  │
│         │ 5.배포 적용        │
│         ▼                  │
│  ┌──────────────────────┐  │
│  │    애플리케이션 Pod     │  │
│  └──────────────────────┘  │
└────────────────────────────┘
       클러스터 내부에서 Pull하여 배포
```

### GitOps의 장/단점
#### 장점
* Pull형이기 때문에 읽기 권한만 있어도 구현이 가능하며, 인증 정보가 도난당해도 쓰기 권한이 없기 떄문에 CIOps보다 피해를 줄일 수 있다.  
* CI와 CD를 분리할 수 있다. 이에 따라 CI용 권한을 가진 사람(도구)과 CD용 권한을 가진 사람(도구)로 나누어 관리할수 있습니다.

#### 단점
* CIOps보다 구축이 어렵다.

### * GitOps를 구현하기 위해 공개된 대표적인 OSS
#### ArgoCD
ArgoCD는 쿠버네티스 네이티브 GitOps 지속적 배포 도구입니다. Argo CD는 Application이라는 이름의 Custom Resource를 사용하여, 
'어떤 레포지터리의', '어떤 매니페스트의', '어떤 버전(브랜치)의' 매니페스트를 '어떤 환경에' 적용할지를 지정합니다. ArgoCD 자체도 쿠버네티스에 구축됩니다.

#### Spinnaker
Netflix에서 개발한 도구로, ArgoCD가 쿠버네티스 전용인 반면, Spinnaker는 쿠버네티스 포함 주요 클라우드 서비스의 여러 기능들을 지원합니다. 
그래서 ArgoCD는 쿠버네티스 클러스터에 대한 배포만을 다루지만, Spinnaker는 도커 이미지 빌드와 같은 CI 파이프라인 구축도 가능합니다.

#### FluxCD
GitOps를 제안한 Weaveworks에서 개발한 도구로, ArgoCD와 매우 유사하지만, 멀티테넌시를 지원합니다.

# 10.2 쿠버네티스 매니페스트 관리 
매니페스트의 개수가 늘어날수록 관리하기가 어려워집니다. 예를들어 Stage, Production 환경에서 Secret 값만 다른 경우, 거의 비슷한 매니페스트를 작성하게 됩니다. 
이렇게 되면 수십, 수백개의 서버를 관리한다면 공통부분을 수정할 때 실수가 발생할 수 있습니다. 이를 위해 매니페스트를 쉽게 관리하기 위한 도구들이 있습니다.

## 10.2.1 Helm
Helm: 쿠버네티스용 패키지 매니저로, 매니페스트를 템플릿화하여 관리할 수 있습니다. Chart라는 템플릿을 기반으로 helm install을 실행하여 쿠버네티스 클러스터에 매니페스트를 배포합니다.  
Kustomize에 비해 `템플릿`에 가까운 형식이라 간단하고 이해하기 쉽습니다. 반면, 템플릿에 작성된 것 이상의 작업을 하고 싶을때는 다른 방법을 고려해야합니다.  

### Helm 설치하기
공식 문서를 참고하여 설치합니다: https://helm.sh/docs/intro/install/  
```bash 
$ brew install helm
```

### Helm Chart Repository 추가하기
Helm Chart를 설치하기 전에 Helm Chart Repository를 추가해야 합니다. 다음 명령으로 Prometheus용 repository를 추가합니다.  
```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### 설치할 네임스페이스 생성하기
네임스페이스를 미리 만듭니다.
```bash
$ kubectl create namespace monitoring
namespace/monitoring created
```

### Helm install 실행하기
``` bash
$ helm install kube-prometheus-stack --namespace monitoring prometheus-community/kube-prometheus-stack
NAME: kube-prometheus-stack
LAST DEPLOYED: Wed Dec 31 03:01:10 2025
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
NOTES:
...생략...

// 잠시 기다리면 몇 개의 pod가 실행되는 것을 확인할 수 있습니다.
$ kubectl get pod --namespace monitoring 
NAME                                                       READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0          2/2     Running   0          54s
kube-prometheus-stack-grafana-66494876ff-pd5fn             3/3     Running   0          64s
kube-prometheus-stack-kube-state-metrics-669dcf4b9-jtgx2   1/1     Running   0          64s
kube-prometheus-stack-operator-7685dd7d99-jqmz9            1/1     Running   0          64s
kube-prometheus-stack-prometheus-node-exporter-dhnpv       1/1     Running   0          64s
prometheus-kube-prometheus-stack-prometheus-0              2/2     Running   0          54s

// 생성된 Service에 포트 포워딩을 수행하여 대시보드 로그인 화면에 접속하겠습니다.
$ kubectl port-forward service/kube-prometheus-stack-grafana --namespace monitoring 8080:80
Forwarding from 127.0.0.1:8080 -> 3000
Forwarding from [::1]:8080 -> 3000

// localhost:8080으로 접속하면 Grafana 로그인 화면이 표시됩니다.
// username: admin, password: prom-operator를 입력하면 로그인할 수 있습니다.
// 이것만으로는 Helm의 편리함을 충분히 이해하기 어려울 것 같아서, 조금더 보겠습니다.
// 다음 명령어를 실행하면 어떤 값을 커스텀할 수 있는지 확인할 수 있습니다.
$ helm show values prometheus-community/kube-prometheus-stack
# Default values for kube-prometheus-stack.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

## Provide a name in place of kube-prometheus-stack for `app:` labels
##
nameOverride: ""

## Override the deployment namespace
##
namespaceOverride: ""
...

// 여기서 출력되는 설정값은 기본값에 해당합니다. 이 기본값을 변경하려면 values.yaml에 설정사항을 기쟇고, helm install의 인자로 지정하면 됩니다. 
```
예를들어 보안을 고려하여 admin 비밀번호를 변경하고 싶은 경우 다음과 같이 values.yaml을 작성하면 됩니다.
```yaml
grafana:
  adminPassword: secure-password
```

대부분의 경우 기본값이 작성된 values.yaml을 깃헙 레포지터리에서 확인할 수 있습니다. 그러면 `helm show values`를 실행하는 것보다 쉽게 값을 확인할 수 있습니다. 
* `helm install` 명령어를 직접 실행하는 것은 GitOps의 철학과 맞지 않습니다.
* ArgoCD같은 GitOps에이전트의 사양에 맞게 Helm 설치를 수행하거나, CI를 사용하여 생성된 매니페스트를 GitOps로 관리하는 방법이 있습니다. 이 경우, 로컬에서 템플릿을 렌더링하는 `helm template`를 사용합니다.

## 10.2.2 Jsonnet
Helm과 Kustomize보다 훨씬 유연한 도구입니다. 그러나, Jssonet자체는 쿠버네티스에 특화된 도구가 아닙니다.

## 10.2.3 자체 템플릿
어떤 도구도 잘 맞지 않는 경우, 템플릿을 직접 작성하는 방법도 있습니다.

## 10.2.4 Kustomize
* 환경별로 매니페스트가 약간씩만 다른 경우에는 수정 작업을 최소화하기 위해 차이점만 관리하는 것이 효율적입니다.  
* Kustomize는 매니페스트의 공통 부분을 base라는 디렉터리에서 관리하고, 환경별 차이점을 overlays라는 디렉터리에서 관리합니다.  
* ArgoCD등의 GitOps도 지원하므로, CIOps, GitOps 모두 사용가능합니다.
* kubectl도 지원하지만, 버전에 따라 kustomize의 버전이 달리지는 부분을 주의해야합니다.

## 10.2.5 Kustomize로 매니페스트를 이해하기 쉽게 만들기
### 사전 지식 
base 디렉터리와 overlays 디렉터리가 있다고 설명했는데, 빌드할 때는 kustomization.yaml이란 파일을 참조합니다.
``` bash
hello-server
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── production
    │   ├── deployment.yaml
    │   └── kustomization.yaml
    └── staging
        ├── deployment.yaml
        └── kustomization.yaml
```
위 예의 base에는 staging과 production의 공통 매니페스트가 배치되며, overlays의 각 디렉터리에는 각 환경 고유의 설정이 배치됩니다.

### 준비
```bash
brew install kustomize
```

### 요구사항
```yaml
---
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
        image: blux2/hello-server:1.8
        resources:
          requests:
            memory: "256Mi"
            cpu: "10m"
          limits:
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: hello-server-pdb
spec:
  maxUnavailable: 10%
  selector:
    matchLabels:
      app: hello-server
```

다음 요구사항이 충족되도록 각 디렉터리에 매니페스트를 작성합니다.
* production 환경과 staging 환경에 배포한다.
* production의 replicas는 10으로 설정한다.
* production의 requests.memory와 requests.limits는 1Gi로 설정한다.
* staging에서는 PodDisruptionBudget을 사용하지 않는다.

### 매니페스트 분할하기
지금까지는 편의를 위해 하나의 매니페스트 파일에 모든 내용을 적었습니다. 하지만 `kustomize build <디렉터리 이름>`을 실행하면 여러 매니페스트 파일들을 한번에 출력할 수 있습니다.
* overlays와 base를 나눌때는 리소스 별로 파일이 분리되어 있는 것이 더 관리하기 쉽습니다.

```bash
분리 전
┌───────────────────────────────┐
│  chapter-10                   │
│   └── hello-server.yaml       │
└───────────────────────────────┘


분리 후
┌───────────────────────────────┐
│  chapter-10                   │
│   ├── deployment.yaml         │
│   └── pdb.yaml                │
└───────────────────────────────┘
```

### 파일을 base 디렉터리에 배치하기
base에는 공통 매니페스트를 배치합니다.
* 이번에는 PodDisruptionBudget은 production에서만 사용하고, Deployment는 staging/production 공통으로 사용합니다.
* 루트 디렉터리는 서비스의 이름인 hello-server로 했는데, 어떤 이름이어도 괜찮습니다.
* 다음과 같은 구조가 되도록 base, overlays, production, stagin 디렉터리를 생성하고 각각 파일을 배치합니다.
```bash
┌──────────────────────────────────────┐
│ hello-server                         │
│ ├── base                             │
│ │   └── deployment.yaml              │
│ └── overlays                         │
│     ├── production                   │
│     │   └── pdb.yaml                 │
│     └── staging                      │
└──────────────────────────────────────┘
```

### 매니페스트의 차이점을 overlays에 배치하기
production만의 차이점을 overlays/production/deployment.ymal에 작성합니다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
spec:
  replicas: 10
  template:
    spec:
      containers:
      - name: hello-server
        resources:
          requests:
            memory: "1Gi"
          limits:
            memory: "1Gi"
```

```bash
┌──────────────────────────────────────┐
│ hello-server                         │
│ ├── base                             │
│ │   └── deployment.yaml              │
│ └── overlays                         │
│     ├── production                   │
│     │   ├── deployment.yaml          │
│     │   └── pdb.yaml                 │
│     └── staging                      │
└──────────────────────────────────────┘
```

### kustomization.yaml 작성하기
해당 파일이 없으면 아무것도 빌드되지 않습니다. 
```bash
$ kustomize build
Error: unable to find one of 'kustomization.yaml', 'kustomization.yml' or 'Kustomization' in directory '/Users/yosep/desktop/학습/그림과 실습으로 배우는 쿠버네티스 입문/chapter-10'
```

kustomization.yaml에서 자주 사용되는 설정 키워드로 resources와 patches가 있습니다.
* resources: 사용할 리소스를 담은 디렉터리나 파일을 지정합니다. base 디렉터리나 해당 디렉터리 고유의 파일을 지정합니다.
* patches: overlays에서 base의 설정을 덮어쓸 때 사용합니다. 덮어쓸 파일명을 지정합니다.
  
reesources로 디렉터리를 지정할 때는 대상 디렉터리에도 kustomization.yaml이 있어야합니다. 따라서 base 디렉터리에도 작성해야합니다. 
다음 매니페스트를 hello-server/base/kustomization.yaml 경로에 저장합니다.
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
```
  
다음 매니페스트를 hello-server/overlays/production/kustomization.yaml 경로에 저장합니다.
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - pdb.yaml
patches:
  - path: deployment.yaml
```
다음 매니페스트를 hello-server/overlays/staging/kustomization.yaml 경로에 저장합니다.
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
```

### kustomize build로 파일을 빌드하고 클러스터에 적용하기
(1) hello-server가 현재 디렉터리가 되도락 이동합니다.  
(2) 먼저 로컬에서 `kustomize build`를 실행하여 빌드 결과를 확인합니다.  
<mark>kustomize build ./overlays/staging</mark>
```bash
$ kustomize build ./overlays/staging
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-server
  name: hello-server
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
      - image: blux2/hello-server:1.8
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        name: hello-server
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          limits:
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 256Mi
```
<mark>kustomize build ./overlays/production</mark>  
```bash
$ kustomize build ./overlays/production
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-server
  name: hello-server
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-server
  template:
    metadata:
      labels:
        app: hello-server
    spec:
      containers:
      - image: blux2/hello-server:1.8
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        name: hello-server
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          limits:
            memory: 1Gi
          requests:
            cpu: 10m
            memory: 1Gi
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: hello-server-pdb
spec:
  maxUnavailable: 10%
  selector:
    matchLabels:
      app: hello-server

```

staging/production에서 각각 다른 매니페스트가 출력된었습니다. 여기서는 staging용 매니페스트를 쿠버네티스 환경에 적용해보겠습니다.

```bash
$ kustomize build ./overlays/staging | kubectl --namespace default apply -f -                            
deployment.apps/hello-server created

$ kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-7cb666f9b7-jdljl   1/1     Running   0          25s
hello-server-7cb666f9b7-nrj6z   1/1     Running   0          25s
hello-server-7cb666f9b7-xfrkg   1/1     Running   0          25s
```

파드가 성공적으로 생성되었습니다. 이제 실습을 정리하겠습니다. 이번 실습은 이전과 달리 `kustomize build`로 얻은 매니페스트를 적용했으므로, 
삭제할 때도 동일하게 `kustomize build`를 사용해야합니다.  
<mark>kustomize build ./overlays/staging | kubectl --namespace default delete -f -</mark>

```bash
$ kustomize build ./overlays/staging | kubectl --namespace default delete -f -
deployment.apps "hello-server" deleted from default namespace
```