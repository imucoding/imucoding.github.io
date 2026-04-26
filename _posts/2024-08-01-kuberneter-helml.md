---
layout: post
title: 헬름을 이용한 애플리케이션 패키징 및 관리
subtitle:
excerpt_image: https://imucoding.github.io//assets/images/chart_arc.png
author: Hyunjune
categories: kubernetes
tags: helm
---
{% raw %}
### 헬름이 제공하는 기능
- 헬름 사용 시 여러 개의 YAML 정의 및 스크립트를 하나의 아티팩트로 묶어 공개 또는 비공개 리포지토리에 공유 가능
- 헬름 프로젝트는 CNCF에서 관리함
- Helm3 부터 서버 컴포넌트 필요 없이 사용가능 (kubectl과 동일 접속 정보 사용), 다만 패키지 리포지토리를 따로 설정 해야함
  
ex)
```
helm repo add test_repo https://test_repo.net 
helm repo update  # 로컬 리포지토리 캐시 업데이트
helm search repo web --versions # 애플리케이션 검색
```

- 헬름에서 애플리케이션의 패키지를 `chart`라고 함
- 차트는 로컬에서 만들어져서 로컬 컴퓨터에 저장될 수 있고 리포지토리에 배포될 수 있음
- 설치된 차트를 `release`라고 함. 릴리즈는 이름을 붙일 수 있고 릴리즈 이름을 달리해가며 같은 차트를 여러번 설치 가능

<br>

**패키지(차트) 구조** 

![image](https://imucoding.github.io/assets/images/chart_arc.png)
- 차트는 압축 파일 형태로 패키징되며 차트 이름과 버전이 부여됨
- 압축 파일에는 디렉토리가 들어있는데 이 디렉토리 이름이 차트 이름이 됨
- `Chart.yaml`
  - 차트 이름, 버전, 설명 등 메타 데이터가 존재
- `templates`
  - 쿠버네티스 매니페스트가 들어있는 디렉토리
- `values.yaml`
  - 쿠버네티스 매니페스트에 쓰인 파라미터 기본 값이 들어있음

<br>

- 차트 파라미터 값 확인
```
 helm show values kiamol/vweb --version 1.0.0 # 
```
- 파라미터 기본 값 수정하여 설치
```
 helm install --set servicePort=8010 --set replicatCount=1 ch10-vweb kiamol/vweb --version 1.0.0 
```
- 레플리카 수 변경하여 업그레이드
```
helm upgrade --set replicatCount=3 ch10-vweb kiamol/vweb --version 1.0.0 
```
- 릴리즈 확인
```
helm ls
```

<br>
<hr>

### 헬름으로 애플리케이션 패키징

- 애플리케이션의 매니페스트 (yaml) 파일을 모으고, 파라미터 값을 선정하고, 정의상 실제 설정 값을 템플릿 변수로 수정
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }} # 릴리즈 이름이 들어갈 템플릿 변수
  labels:
    kiamol: {{ .Values.kiamolChapter }} # kiamolChapter 값이 들어갈 템플릿 변수
```
- 치환되는 값은 여러 출처에서 옴
- Release 객체 
  - install 혹은 upgrade시 관련 정보를 담아 생성됨
- Values 객체
  - 차트에 포함된 기본값에 사용자가 지정한 값을 overwrite한 정보를 담아 생성

<br>

![image](https://imucoding.github.io/assets/images/web-ping.png)

```
helm lint web-ping
```
- `lint` : 유효성 검증, 차트 설치에 실패할 수 있는 원인을 출력해줌
- 대상 차트는 압축 파일이 아니어도 가능, 차트 디렉토리에서 작업 가능

```
helm install wp1 web-ping
```
<br>

**web-ping-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    kiamol: {{ .Values.kiamolChapter }}
spec:
  selector:
    matchLabels:
      app: web-ping
      instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: web-ping
        instance: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: kiamol/ch10-web-ping
          env:
            - name: TARGET
              value: {{ .Values.targetUrl }}
            - name: METHOD
              value: {{ .Values.httpMethod }}
            - name: INTERVAL
              value: {{ .Values.pingIntervalMilliseconds | quote }}
```
- `quote` : 삽입된 값의 앞뒤로 따옴표가 없을 때 따옴표를 붙여주는 역할 수행

<br>

```
helm install --set targetUrl=kiamol.net wp2 web-ping/
```
- 요청 대상 url을 달리하여 wp2라는 이름으로 릴리즈 추가 배치

<br>

- 차트에 포함된 템플릿을 대상으로는 kubectl apply 명령 사용 불가
  - 템플릿 변수 때문에 유효하지 못한 yaml 파일로 인식함
- 리포지토리는 웹 서버에 저장된 차트와 버전의 정보가 담긴 index.yaml 파일임
  - `ChartMuseum` 이라는 오픈소스를 통해 리포지토리 호스팅 가능

<br>

#### 차트 리포지토리에 배포
1) 차트 패키징 <br>
   ```helm package web-ping```
<br>
2) 패키징 된 압축 파일 차트 뮤지엄에 업로드 <br>
   ```curl --data-binary "@web-ping-0.1.0.tgz" {차트 뮤지엄 url}```
<br>
3) 리포지토리 인덱스에 새로운 차트 정보 추가 -> 차트 뮤지엄이 대신 해줌
<br>
4) 차트가 추가 되었는지 확인 가능 <br>
```curl {차트 뮤지엄 주소}/index.yaml``` 
  

<br>

#### 설정 값 파일을 사용하여 차트를 설치
```helm install -f web-ping-values.yaml wp3 local/web-ping```
- 환경 별로 설정 값 따로 저장 가능
- 버전 지정하지 않으면 가장 최신 차트가 설치됨

<br>
<hr>

### 차트 간 의존 관계 모델링

**Chart.yaml**
```yaml
apiVersion: v2 # 헬름 정의 규격 버전
name: pi # 차트 이름
description: A Pi calculator 
type: application
version: 0.1.0 # 차트 버전
dependencies: # 이 차트가 의존하는 다른 차트
  - name: vweb # 의존 차트 이름
    version: 2.0.0
    repository: https://kiamol.net # 차트 출처가 리포지토리
    condition: vweb.enabled # 필요할 때만 설치 
  - name: proxy # 의존 차트 이름
    version: 0.1.0
    repository: file://../proxy # 로컬 디렉토리에만 있는 차트
    condition: proxy.enabled # 필요할 때만 설치
```
- 상위 차트 `pi`가 하위 차트 `proxy, vweb`을 의존하는 관계
- proxy 차트를 의존 차트로 사용하기 위해서 상의 차트 정의에서 의존 차트 목록에 추가하여 하위 차트로 삼아야 함
- 이후 하위 차트 설정 값을 상위 차트의 정의 에서 지정

**values.yaml**
```yaml
# number of app Pods to run
replicaCount: 2
# type of the app Service:
serviceType: LoadBalancer
# settings for vweb
vweb:
   # whether to deploy vweb
   enabled: false    
# settings for the reverse proxy
proxy: # proxy 설정 값
  # whether to deploy the proxy
  enabled: false
  # name of the app Service to proxy
  upstreamToProxy: "{{ .Release.Name }}-web" 
  # port of the proxy Service
  servicePort: 8030
  # number of proxy Pods to run
  replicaCount: 2
```
- `upstreamToProxy: "{{ .Release.Name }}-web"`
  - pi 애플리케이션 서비스 명과 일치시키기 위해 템플릿 변수 사용
- 하위 차트 설정 값 참조 시 설정 값 이름 앞에 차트 이름 붙임
  - ex) proxy.upstreamToProxy
- 현재 proxy에 대한 의존성 condition은 `proxy.enabled`를 참조하고 있고, enabled 값이 false이므로 하위 차트 정의가 무시됨

<br>

- 차트를 설치하거나 패키징 하려면 의존하는 하위 차트를 사용할 수 있어야 함
  - `helm dependency build pi`
- 의존 차트 빌드 시 원격 리포지토리의 차트를 내려받거나 로컬 차트가 압축 되어 상위 차트 디렉토리 아래에 복사됨
  - pi/charts에 `proxy-0.1.0.tgz`, `vweb-2.0.0.tgz` 생성됨
- ``helm install --set serviceType=ClusterIP --set proxy.enabled=true pi1 ./pi``
  - 상위 차트 설치 시 하위 차트의 압축을 풀어서 함께 설치

<br>
<hr>

### 헬름 릴리즈의 업그레이드와 롤백
- 현재 vweb 차트 1.0.0 버전이 설치된 상태
- 새 버전의 차트의 설정 값 확인
  - `helm show values kiamol/vweb --version 2.0.0`
- 기존 릴리즈 설정 값을 재사용하여 버전 2.0.0 업그레이드
  - `helm upgrade --reuse-values --atomic vweb kiamol/vweb --version 2.0.0`
    - `--atomic` : 모든 리소스의 업그레이드를 기다리다가 일부 실패 시 전부 롤백
    - 릴리즈 명은 기존과 동일해야 함. 다를 경우 upgrade가 아닌 install 사용
- 버전 2.0.0에는 serviceType이라는 새로운 설정 값이 존재하여 기존 설정 값을 재사용하여 업그레이드 실패
  - 업그레이드 실패 시 자동으로 이전 버전으로 롤백

<br>

- 릴리즈 히스토리 확인 가능
```
helm history vweb
```
- 현재 릴리즈의 설정 값 적용
```
helm get values vweb -o yaml > vweb-valuse.yaml
helm upgrade -f vweb-values.yaml --atomic vweb kiamol/vweb --version 2.0.0
```
- 롤백은 리비전을 지정해서 확인
```
helm rollback vweb 2
```

<br>
<hr>

### 헬름 사용에 대한 고찰
- 매니페스트 파일을 한 번 템플릿 변수로 구성하면 이전으로 돌리기 어려움
- 따라서 팀에서 helm 사용 시 모두가 함께 사용해야 함
{% endraw %}
