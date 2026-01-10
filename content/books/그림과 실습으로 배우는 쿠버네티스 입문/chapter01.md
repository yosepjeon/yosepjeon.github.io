+++
title = "Chapter 01. 도커 컨테이너 만들어보기"
date = "2025-11-27"
weight = 1
+++

# 1.1 쿠버네티스는 왜 도커가 필요한가?
쿠버네티스가 컨테이너 관리용 도구이기 때문에 도커가 필요하다.

# 1.2 도커 알아보기
## 1.2.1 도커란?
### docker: 컨테이너 가상화 기술 중 하나로 2013년 `Build, Share, Run`이라는 일련의 라이프사이클을 docker 명령어로 컨테이너 기술을 훨씬 더 사용하기 쉽게 제공

## 1.2.2 컨테이너란?
### 컨테이너: OS 위에 만들어지는 격리된 가상 환경
컨테이너는 프로세스와 달리 컨테이너 간에 실행 환경을 참조할 수 없다.  
ex1) 컨테이너 A 안에서 생성된 파일을 컨테이너 B가 마음대로 참조할 수 없다.
ex2) 컨테이너 A 안에 설치된 프로그램을 컨테이너 B가 실행할 수 없다.

## 1.2.3 왜 컨테이너인가?
### 1. 가상 머신보다 더 빠르게 애플리케이션을 실행할 수 있다.
컨테이너는 가상 머신과 달리 운영체제를 포함하지 않고 가상화하기 때문에 가상 머신보다 더 빠르게 애플리케이션을 실행할 수 있다.
```bash
┌───────────────────────────────────────────────────────────────┐        ┌───────────────────────────────────────────────┐
│                           가상 머신                             │        │                    컨테이너                     │
│                                                               │        │                                               │
│   ┌──────────────┐     ┌──────────────┐                       │        │   컨테이너               컨테이너                 │
│   │     앱       │     │     앱       │                        │        │  ┌──────────────┐     ┌──────────────┐        │
│   └──────────────┘     └──────────────┘                       │        │  │     앱       │     │     앱        │        │
│   ┌──────────────┐     ┌──────────────┐                       │        │  └──────────────┘     └──────────────┘        │
│   │   가상 OS     │     │   가상 OS     │                       │        │                                               │
│   └──────────────┘     └──────────────┘                       │        │  ┌───────────────────────────────────────┐    │
│                                                               │        │  │             컨테이너 런타임               │    │
│   ┌───────────────────────────────────────────────────────┐   │        │  └───────────────────────────────────────┘    │
│   │                    하이퍼바이저                          │   │        │  ┌───────────────────────────────────────┐    │
│   └───────────────────────────────────────────────────────┘   │        │  │                호스트 OS                │    │
│   ┌───────────────────────────────────────────────────────┐   │        │  └───────────────────────────────────────┘    │
│   │                      호스트 OS                          │   │        │  ┌───────────────────────────────────────┐    │
│   └───────────────────────────────────────────────────────┘   │        │  │                물리 머신                │    │
│   ┌───────────────────────────────────────────────────────┐   │        │  └───────────────────────────────────────┘    │
│   │                      물리 머신                          │   │        └───────────────────────────────────────────────┘
│   └───────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────┘

```

### 2. MSA(Micro Service Architecture) 환경에 적합하다.


## 1.2.4 그래서 도커란?
### 도커는 컨테이너 기술을 이용해 어디에서도 동일한 환경을 제공하는 플랫폼이다.
- 요즘은 보다 빠른 개발/배포/운영 사이클이 요구되고 있음
- 도커의 강점은 `docker` 명령어로 모든 사이클을 다룰 수 있다.

## 1.2.5 준비: 도커 환경 만들기
Window, macOS의 경우 도커 데스크톱을 추천, Linux의 경우 도커 엔진을 설치한다.

## 1.2.6 컨테이너 실행하기
```bash
$ docker run --rm --detach --publish 8080:80 --name web nginx:1.25.3
Unable to find image 'nginx:1.25.3' locally <-- 로컬에 nginx:1.25.3이 없다는 뜻
1.25.3: Pulling from library/nginx <-- '로컬에 도커 이미지가 없기 때문에 library.nginx에서 pull'했다는 뜻
13e7597894b3: Pull complete 
f546e941f15b: Pull complete 
2e49d9ae23c1: Pull complete 
b70338a86458: Pull complete 
47229761a693: Pull complete 
bca0a94f9783: Pull complete 
ad1f53eee07d: Pull complete 
Digest: sha256:c7a6ad68be85142c7fe1089e48faa1e7c7166a194caa9180ddea66345876b9d2
Status: Downloaded newer image for nginx:1.25.3
9c43b96b75b250f569257c4443d4eed875e6f5976dd5150ebcd9571e6e649a7c
```

브라우저에서 localhost:8080 접속 시 Nginx 환영 페이지가 보인다.

## 1.2.7 컨테이너의 틀이 되는 도커 이미지
도커이미지: 애플리케이션을 실행하는 데 필요한 모든 것(모든 의존관례, 설정, 스크립트, 바이너리 등)과 메타데이터 등 컨테이너의 각종 설정의 집합체

도커 이미지는 여러 개의 이미지 레이어를 겹겹이 쌓아 올려 만든다. NGINX 컨테이너를 실행할 때 Pull complete라는 문구가 여러 줄에 걸쳐 나타났는데, 각 줄마다 해당 레이어를 로컬 환경에 다운로드한다고 보면된다.

```bash
    mkdir /tmp  ──────────────────────▶ ┌───────────────────────────┐
                                        │           레이어           │
                                        └───────────────────────────┘
    sudo apt install nginx ───────────▶ ┌───────────────────────────┐
                                        │           레이어           │
                                        └───────────────────────────┘
    Ubuntu : 22.04 ───────────────────▶ ┌───────────────────────────┐
                                        │       base 레이어           │
                                        └───────────────────────────┘


                         도커 이미지의 예



                                                     ┌───────────────────────────────┐
                                                     │ 가장 밑에 있는 레이어를             │
                                                     │ 베이스 레이어라고 불러              │
                                                     └───────────────────────────────┘
                                                              ^
                                                              |
                                                          /\_/\ 
                                                         ( 'ㅅ' )
                                                         /|   |\

```

컨테이너를 실행하는 데 필요한 모든 것이 도커 이미지에 정리되어 있기 떄문에, 동일한 도커 이미지를 사용하면 다른 컴퓨터에서 실행해도 항상 동일한 환경을 구축할 수 있다.

## 1.2.8 컨테이너 이미지의 설계서인 Dockerfile
### Dockerfile은 컨테이너 이미지의 설계서로, 이 파일에 대해 `docker build` 명령어를 실행하면 도커 이미지가 만들어진다.

예를 들어 내가 만든 애플리케이션을 컨테이너로 실행하려면 이를 위한 도커 이미지를 생성해야한다.  
'NGINX의 index.html을 교체하고 싶다'고 가정하고 Dockerfile을 작성해보면 아래와 같다.
```dockerfile
# FROM: 인자로 베이스가 되는 이미지를 지정
FROM nginx:1.25.3

# 도커 이미지의 파일 시스템으로 파일을 복사
COPY index.html /usr/share/nginx/index.html
```

* Dockerfile의 시작 부분은 반드시 FROM으로 시작해 베이스가 되는 이미지를 지정해야한다.  
* Dockerfile을 작성할 때는 명령어를 대문자로 쓰고 공백에 이어 인자를 지정한다.

## 1.2.9 도커 이미지 빌드하기
### 원격 환경에서 컨테이너를 실행하려면 'Build, Share, Run'의 세 단계가 필요하다.
#### 준비: index.html 작성하기
원하는 디렉토리(custom-=nginx라고 가정)에 index.html 파일을 작성한다.

```html
<h1>Hello World</h1>
```
#### Dockerfile 작성하기
custom-nginx 디렉토리에 Dockerfile을 작성한다.
```dockerfile
FROM nginx:1.25.3
COPY index.html /usr/share/nginx/index.html
```

이제 `docker build` 명령어로 도커 이미지를 빌드한다.  
<mark>docker build custom-nginx --tag nginx-custom:1.0.0 </mark>  
-- tag로 도커 이미지의 이름과 태그를 지정한다. 지정방법은 `이미지 이름:태그`
```bash
$ docker build custom-nginx --tag nginx-custom:1.0.0 
[+] Building 0.2s (7/7) FINISHED                                                                                            docker:rancher-desktop
 => [internal] load build definition from Dockerfile                                                                                          0.0s
 => => transferring dockerfile: 93B                                                                                                           0.0s
 => [internal] load metadata for docker.io/library/nginx:1.25.3                                                                               0.0s
 => [internal] load .dockerignore                                                                                                             0.0s
 => => transferring context: 2B                                                                                                               0.0s
 => [internal] load build context                                                                                                             0.0s
 => => transferring context: 59B                                                                                                              0.0s
 => [1/2] FROM docker.io/library/nginx:1.25.3@sha256:c7a6ad68be85142c7fe1089e48faa1e7c7166a194caa9180ddea66345876b9d2                         0.0s
 => => resolve docker.io/library/nginx:1.25.3@sha256:c7a6ad68be85142c7fe1089e48faa1e7c7166a194caa9180ddea66345876b9d2                         0.0s
 => [2/2] COPY index.html /usr/share/nginx/html                                                                                               0.0s
 => exporting to image                                                                                                                        0.0s
 => => exporting layers                                                                                                                       0.0s
 => => exporting manifest sha256:d1202e93b412c5500a20d479a0e647eaf517ded262c550f72bc02023d4d46cad                                             0.0s
 => => exporting config sha256:abdb90bc067478ba74d28635e881d5de9ae200a07f795037a2a76e8a3318a02b                                               0.0s
 => => exporting attestation manifest sha256:a6c5d2075126ac836b757d9d6fdcad12fb49a2e8eed671d0c3468d63890a8155                                 0.0s
 => => exporting manifest list sha256:678c6ee1014f3e0f58671c6a1470e2265c4bbdf39cc7b245c1183dee937c6511                                        0.0s
 => => naming to docker.io/library/nginx-custom:1.0.0                                                                                         0.0s
 => => unpacking to docker.io/library/nginx-custom:1.0.0                                                                                      0.0s
 
 각 명령어 마다 레이어가 생성되고 최종 결과물로 도커 이미지가 만들어지는 Build를 확인할 수 있다.
```

`docker images` 명령어로 생성된 도커 이미지를 확인할 수 있다.

``` bash
IMAGE                                                                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
ghcr.io/rancher-sandbox/rancher-desktop/rdx-proxy:latest                               c7487aa44aa6       11.4MB          5.7MB        
kindest/node@sha256:7416a61b42b1662ca6ca89f02028ac133a309a2a30ba309614e8ec94d976dc5a   7416a61b42b1       1.48GB          441MB    U   
nginx-custom:1.0.0                                                                     678c6ee1014f        274MB         67.2MB        
nginx:1.25.3                                                                           c7a6ad68be85        274MB         67.2MB    U   
```

## 1.2.10 직접 만든 도커 이미지로 컨테이너 실행하기
### 이번에는 'Build, Share, Run'의 세 단계 중 'Run'을 해보자. Share를 건너뛰는 이유는 로컬에서 빌드한 도커 이미지를 로컬에서 실행하는 경우에는 Share가 필요없기 때문이다.

#### 도커 컨테이너 정지하기
`docker run`에서 `--detach` 옵션을 사용하면 컨테이너를 백그라운드에서 실행할 수 있다. 백그라운드에서 실행중인 컨테이너 하나를 종료하기 위해서는 `docker ps`로 실행중인 컨테이너를 찾아야한다. 그다음 해당 컨테이너의 ID를 이용하여 stop 명령어를 실행한다.
``` bash
$ docker ps
CONTAINER ID   IMAGE                        COMMAND                   CREATED        STATUS        PORTS                                     NAMES
9c43b96b75b2   nginx:1.25.3                 "/docker-entrypoint.…"    30 hours ago   Up 30 hours   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   web
b6a8400f9978   d477ecd186ea                 "/metrics-server --c…"    5 days ago     Up 5 days                                               k8s_metrics-server_metrics-server-7bfffcd44-xhngw_kube-system_c0167989-7c71-4689-b495-ca244b0fe65d_1
d316de1739dd   c17464474775                 "/coredns -conf /etc…"    5 days ago     Up 5 days                                               k8s_coredns_coredns-6d668d687-hclf9_kube-system_4a4fa7a9-c531-4085-a717-5c480c0e7538_1
87bf5aea04cf   fadaf2f157b4                 "local-path-provisio…"    5 days ago     Up 5 days                                               k8s_local-path-provisioner_local-path-provisioner-869c44bfbd-br4cn_kube-system_d600c4fd-48fe-4fea-8872-e5ff1ee4b4e5_1

$ docker stop 9c43b96b75b2
9c43b96b75b2
```

#### 컨테이너 실행하기
<mark>docker run --rm --detach --publish 8080:80 --name web nginx-custom:1.0.0</mark>  
`docker run <이미지 이름>` 명령으로 도커 컨테이너를 실행할 수 있다. 
* --rm: 컨테이너가 종료되면 자동으로 삭제
* --detach: 백그라운드에서 실행
* --publish 8080:80: 호스트의 8080 포트를 컨테이너의 80 포트에 매핑
* --name web: 컨테이너 이름을 web으로 지정

브라우저 http://localhost:8080 접속 시 "Hello World"가 표시된다.

`docker start/stop` 명령어는 컨테이너 ID뿐만 아니라 컨테이너의 이름을 지정하여 정지시킬 수도 있다.
``` bash
$ docker stop web
web
```

## 1.2.11 도커 이미지 공개하기
### `Build, Share, Run`의 세 단계 중 'Share'를 해보자. Shard는 본인이 만든 이미지를 다른 컴퓨터에 공유하는 방법이다.
컨테이너 레지스트리: 깃허브에 바이너리를 업로드하면 다운로드한 로컬 컴퓨터에서 프로그램을 실행할 수 있는 것처럼 컨테이너 이미지를 업로드하는 공간  
`docker run` 명령어에서 컨테이너 레지스트리 이름을 지정하지 않으면 기본적으로 도커 허브에 업로드된 도커 이미지를 참조한다. 도커 허브는 리포지터리 단위로 이미지를 관리하며, 리포지터리 하나에 여러개의 태그를 저장할 수 있다.

#### docker push 명령어
`docker push` 명령어는 도커 이미지를 레지스트리에 업로드할 때 사용한다.  
예를들어 도커 허브를 사용하는 경우에는 다음과 같은 순서를 따른다.  
1. 계정 생성
2. 리포지터리 생성
3. docker login 명령으로 터미널에서 도커 허브에 로그인
4. docker push <리포지터리 이름>으로 업로드

## 1.2.12 Dockerfile 작성 팁
### 주의: 비밀 및 기밀 정보는 도커 이미지에 포함시키지 말 것
레이어를 겹쳐 쌓는다고 해서 컨테이너가 실행된 후 하위 레이어의 정보가 보이지 않게 되는 것이 아니다. 비밀 및 기밀 정보가 포함된 상태로 도커 이미지를 빌드하면 최상위 레이어에서는 비밀 정보가 보이지 않더라도 여전히 참조 가능하다.

``` bash
                    /\_/\ 
                   (  . .)        얼핏 보면
                    \  _/         안 보이지만
                      \              ↘
                                       ↘


                         ┌────────────────────────────┐
                        /|                           /|
                       / |                          / |
                      /  |                         /  |
                     └────────────────────────────┘   |
                     |   ┌────────────────────────┐   |
                     |  /|                       /|   |
                     | / |      (  🔑  )        / |   |   }  마음만 먹는다면
                     |/  |                     /  |   |   }  하위 레이어에
                     └────────────────────────┘   |   |   }  접근할 수 있다
                     |   ┌───────────────────────┐|   |
                     |  /|                       /|   |
                     | / |                      / |   |
                     |/  |                     /  |   |
                     └────────────────────────┘   |  /
                     |                            | /
                     └────────────────────────────┘


      비밀 키가
      포함된 레이어  ───────────────►   (가운데 레이어)

```
* 비밀 정보가 하위 레이어에 포함된 상태로 도커 이미지를 레지스트리에 업로드하면 `docker pull`을 수행하는 모든 사용자가 해당 비밀 및 기밀 정보에 접근할 수 있다.
* 이데 대한 대책으로 외부로부터 비밀 정보를 지정하도록 구성하거나 이어서 설명할 멀티 스테이지 빌드를 사용하여 공개해서는 안 될 정보가 최종 결과물에 포함되지 않도록 구성한다.

### 권장 사항: 멀티 스테이지 빌드 수행
컨테이너는 가능한 가벼울수록 실행과 업데이트가 빨라서 좋다. 또한 취약성을 줄이기 위해 컨테이너에 포함되는 라이브러리 및 애플리케이션은 최소화하는 것이 좋다.  
이러한 목표를 달성하기 위해 스테이지라고 하는 어느정도 묶여진 단위로 빌드한다. 빌드의 결과물만 최종 스테이지의 이미지에 포함시킴으로써 프로그램 실행에 필요한 최소한의 파일만으로 컨테이너를 실행할 수 있다.  

```dockerfile
FROM golang:1.21 AS builder # AS XXX라고 쓰면 스테이지에 이름을 부여할 수 있다.
WORKDIR /app
COPY <<-EOF main.go
package main
import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
EOF
ENV CGO_ENABLED=0
RUN go build -o hello \
    && go mod tidy \ 
    && go build -o hello main.go
FROM scratch # Docker가 공식으로 제공하는 경량 이미지다.
COPY --from=builder /app/hello /hello # --from=<스테이지 이름>으로 지정한 스테이지의 결과물이 이 스테이지에 복사된다.
CMD ["/hello"]

```
멀티 스테이지 빌드를 사용하면 이 도커 이미지로 실행되는 컨테이너에는 main이라는 바이너리만 포함된다.

# 1.3 [만들기] 나만의 http server 컨테이너 실행하기
### hello-server 구현하기
예시로 GO 언어를 사용한다.
``` go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, world!")
	})

	log.Println("Starting server on port 8080")
	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		log.Fatal(err)
	}
}
```

### Dockerfile 만들기
main.go를 저장한 디렉터리에 Dockerfile을 작성한다.
``` dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
ENV CGO_ENABLED=0
RUN go build -o hello .

FROM scratch
COPY --from=builder /app/hello /hello
CMD ["/hello"]
```
또한 다음과 같은 go.mod 파일이 없으면 도커 이미지를 빌드할 수 없으므로 이 파일도 업로드해두자.
``` go.mod
module github.com/bbf-kubernetes

go 1.21
```

### 도커 이미지 빌드하기
``` bash
# `docker build` 명령을 사용해 이미지를 빌드한다.
$ docker build ./hello-server --tag hello-server:1.0
# 도커 이미지 목록을 출력하여 이미지가 무사히 만들어졌는지 확인한다.
$ docker images hello-server
REPOSITORY      TAG       IMAGE ID       CREATED          SIZE
hello-server    1.0       d1e5f3c4b6a2   10 seconds ago   6.45MB
```

### 도커 컨테이너 실행/정지하기
```bash
$ docker run --rm --detach --publish 8080:8080 --name hello-server hello-server:1.0

$ curl http://localhost:8080
Hello, world!

$ docker stop hello-server
```