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
LAST DEPLOYED: Mon Dec 23 10:00:00 2024
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
...생략...

// 잠시 기다리면 몇 개의 pod가 실행되는 것을 확인할 수 있습니다.

```