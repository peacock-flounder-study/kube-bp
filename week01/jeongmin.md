## 새롭게 알게 된 것
### 이미지 관리 모범 사례
- 컨테이너 이미지는 공급망 공격에 취약함 
    - 도커 이미지 잘 알아보고 official만 사용할 것 
- 네이밍 컨벤션
    - 이미지를 빌드한 코드의 커밋 해쉬 값을 사용하여 명시적으로 지정할 것

### Replicaset & Deployment
- Replicaset은 복제 등의 역할을 함
- Deployment는 Replicaset에 더해 버전 관리 및 롤아웃 지원

### Statefulset
- 상태에 대한 복제를 할 땐 유용하다고 함
    - Why????
    - claude에게 물어보니...
        - 각 포드에 대한 안정적이고 고유한 네트워크 ID
        - 정렬되고 예측 가능한 포드 이름(예: app-0, app-1, app-2)
        - 포드를 따르는 영구 저장소
        - 순서대로 배치 및 확장(0, 1, 2...)
        - 순서화된 종료(역순: 2, 1, 0)
        - 재시작 후에도 유지되는 안정적인 호스트 이름
    - 진짠가? 이유가 더 없나? 굳이 이게 전부라면 statefulset에 집착할 이유는 없을 것 같은데...
- Redis, DB 등에 쓰임

### Headless Service
- Headless Service를 사용하면 각 Pod에 대해 안정적인 DNS 레코드가 생성됨
    - 예를 들어 MySQL StatefulSet의 경우, mysql-0.mysql-svc.default.svc.cluster.local 같은 형식으로 각 Pod에 직접 접근할 수 있음
- `clusterIP: None` 으로 설정

```yaml
# 읽기 전용 일반 Service
apiVersion: v1
kind: Service
metadata:
  name: mongodb-read
spec:
  selector:
    app: mongodb
    role: secondary  # Replica들만 선택
  ports:
    - port: 27017
---
# 쓰기용 일반 Service  
apiVersion: v1
kind: Service
metadata:
  name: mongodb-write
spec:
  selector:
    app: mongodb
    role: primary  # Primary만 선택
  ports:
    - port: 27017
```


### Helm
- 여러 개발 및 운영 환경 구성이 필수적
    - 다양한 환경 등에 맞게 배포할 수 있도록 template화를 도와주는 도구 = helm
- 활용 방식
    1. 기존 yaml -> templates 내부에 yaml 파일을 위치하고 `{{ .replicaCount }}` 형태로 원하는 변수들을 작성
    2. 값 자체는 values.yaml로 빼오기
    3. `helm install path/to chart --values path/to/values.yaml`

### 시크릿
- 시크릿은 tmpfs 램 기반의 파일시스템으로 볼륨을 생성해서 컨테이너에 마운트됨
    - 장비가 물리적인 피해를 입어도 공격자가 시크릿을 취득하기 어려움
- `kubectl create secret generic <이름> --from-literal=passwd=${RANDOM}`


## 실습
- https://github.com/brendandburns/kbp-sample 를 베이스로 진행

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:2.0
        name: frontend
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  labels:
    app: frontend
  name: frontend
  namespace: default
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: frontend
  type: ClusterIP

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: localhost
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 8080
```