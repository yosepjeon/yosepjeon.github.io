+++
title = "Chapter 04. 쿠버네티스 클러스터 위에 애플리케이션 만들기"
date = "2025-11-27"
+++




# 4.1 쿠버네티스 클러스터 위에 애플리케이션 실행하기

## 4.1.1 리소스의 사양을 담은 매니페스트


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```
