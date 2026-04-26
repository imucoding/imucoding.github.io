---
layout: post
title: 파드와 디플로이먼트로 컨테이너 시작
subtitle: ''
excerpt_image: 
author: Hyunjune
categories: kubernetes
tags: [pod, deployment]
---
{% raw %}
### 쿠버네티스 이해
- 마스터 노드 (control plain)
  - 클러스터 노드, 파드 관리
- 워커 노드 (data plane)
  - 컨테이너 실행
- `kubeadm init` 으로 클러스터 초기화하여 새로운 마스터 노드 생성
- 마스터 노드는 기본적으로 컨테이너를 실행할 수 없지만 설정을 통해 가능하도록 할 수 있음
- kubectl은 마스터 노드의 api server에 요청하고 api server는 각 워커 노드에 설치된 `kubelet`과 통신하여 각 worker 노드에서 pod를 실행하도록 함
- 마스터 노드의 scheduler가 어떤 워커노드에 pod를 배정할 지 결정

<br>
<hr>

### 컨테이너 실행
- 간단한 pod는 yaml 파일 없이 직접 실행 가능
```
kubectl run hello-kiamol --image=kiamol/ch02-hello-kiamol
kubectl wait --for=condition=Ready pod hello-kiamol  // Ready 될 때까지 I/O 대기
kubectl get pods
kubectl describe pod hello-kiamol // 상세 정보 확인 가능
```

#### 포트 포워딩
```
kubectl port-forward pod/hello-kiamol 8080:80
```
- host PC port -> pod port 

<br>
<hr>

### 컨트롤러 객체
- 파드가 여러 노드에서 실행되다가 어떤 노드가 고장 발생 시 파드를 복구해주지 않음
- 또한 파드가 여러 노드에 고르게 흩어져서 실행된다는 보장이 없으므로 사람이 직접 개입해야하고 오케스트레이션 도구 사용의 의미가 없음
- 따라서 컨트롤러 객체를 사용해야 하고 파드를 관리하는 컨트롤러 객체는 디플로이먼트
  - `컨트롤러 객체` : 다른 리소스를 관리하는 쿠버네티스 리소스 

```
kubectl create deployment hello-kiamol --image=kiamol/ch02-hello-kiamol
```
- pod와 달리 run이 아닌 create로 생성

```
kubectl get pods

Name                      Ready      Status      Restarts      Age
hello-kiamol-21562342      1/1       Running         0         10s
```
- k8s가 디플로이먼트 이름 뒤 무작위 문자열을 붙여서 pod 생성

<br>
<hr>

### Deployment의 Pod 관리
- 모든 쿠버네티스 리소스 (deployment, pod ...) 은 key-value 쌍의 label을 가짐
- deployment는 자신의 selector와 label이 일치하는 pod를 관리함
- `kubectl describe deploy heelo-kiamol`에서 label과 selector를 확인해보면 둘다 `app:hello-kiamol`임을 알 수 있음
  - 본인의 label과 동일하게 selector를 지정하였고 이 selector를 pod에 부여함
- `kubectl get pods -l app=hello-kiamol`
  - app:hello-kiamol의 label을 가진 pod 조회
- 만약 pod의 label을 변경한다면 deployment의 관리에서 벗어남
  - `kubectl label pods -l app-hello-kiamol --overwrite app=x`
- pod를 다시 조회해보면 새로운 pod가 생성되었음이 확인되는데 deployment가 label selector와 일치하는 pod가 사라졌기 때문에 새로운 pod 생성
  - 만약 관리가 벗어난 pod의 app:x 레이블을 원래대로 복원 시 새로 생성한 pod가 삭제됨 (1개 pod 유지)
- `kubectl port-forward deploy/hello-kiamol 8080:80`
  - deployment를 port forwarding 시키면 deployment는 자신이 가진 pod 중 1개를 트래픽 전달 대상으로 삼음

<br>
<hr>

### 애플리케이션 manifest에 배포 정의

```yaml
# pod.yaml

apiVersion: v1 # k8s API version
kind: Pod # 리소스 유형
metadata: # 리소스에서 메타데이터는 name(필수)과 label(비필수)가 존재
  name: hello-kiamol
spec: # 리소스의 실제 정의 내용
  containers: # pod는 컨테이너 정의 필요
    - name: web # 컨테이너는 이름과 이미지로 정의
      image: kiamol/ch02-hello-kiamol
```
- 리소스에 대한 버전 확인
  - `kubectl explain pod`
  - group / version 형식으로 apiVersion에 작성

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kiamol # deployment 이름 지정
spec:
  selector: # 관리 대상 레이블 셀렉터 설정
    matchLabels:
        app:hello-kiamol
  replicas: 3
  template: # pod 생성 시 사용
    metadata: # pod 정의
      labels: # pod 이름 대신 label을 지정하는데 selector와 일치해야함
        app:hello-kiamol
      spec: # pod 컨테이너 정의
        containers:
          - name: web
            image: kiamol/ch02-hello-kiamol
```

<br>
<hr>

### Pod에서 실행 중인 애플리케이션에 접근
1) pod 접속 <br>
  `kubectl exec -it hello-kiamol sh` <br>
2) pod log 출력 <br>
  `kubectl logs --tail=2 hello-kiamol` <br>
3) deployment를 통해서 pod에서 명령 실행 <br>
  `kubectl exec deploy/hello-kiamol --sh -c 'wget -O - http://localhost > /dev/null` <br>
4) pod 속 컨테이너에서 local로 파일 복사 <br>
  `kubectl cp hello-kiamol:usr/share/nginx/html/index.html /tmp/index.html` <br>

<br>
<hr>

### 쿠버네티스 리소스 관리 이해

```
kubectl delete pods -all
```
- 모든 pod 삭제
- pod 조회 시 다시 pod가 생성됨, deploy가 존재하기 때문

```
kubectl get all
```
- 모든 리소스 확인

```
kubectl delete deploy -all
```
- deployment 삭제



{% endraw %}
