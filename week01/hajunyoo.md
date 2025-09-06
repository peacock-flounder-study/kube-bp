# Kubernetes 기본 서비스 설정 정리

## 전체 구조 이해

- **예제 애플리케이션**: 간단한 저널 서비스
    - NGINX 정적 파일 서버 + RESTful API + Redis DB 다중 계층 구조
- **선언적 구성 (YAML)**: 명령형 방식이 아니라 원하는 최종 상태를 선언
    - 시스템 상태를 명확하게 정의하고 복제하기 쉬움
- **Git 관리 필수**: YAML 파일들이 '진실의 원천(SOT)' 역할
    - 모든 변경사항 추적, 검토, 감사, 롤백 가능
- **디렉토리 구조**: 구성 파일이 많아지면 체계적 관리 필요
    
    ```
    journal/
    ├── frontend/
    ├── redis/
    └── fileserver/
    ```
    

## 1. Deployment

**사용 시점**: Stateless 프론트엔드 애플리케이션 배포

**주요 기능**:

- ReplicaSet 기반으로 파드 복제본 수 유지
- 롤링 업데이트: 버전 업데이트 시 다운타임 없이 점진적 교체
- 안정적인 서비스 운영 보장

**핵심 설정 항목**:

- `replicas`: 예기치 않은 장애 대비 + 가용성 향상 위해 2개 이상 권장
- `selector`: Deployment가 관리할 파드들을 레이블로 지정
- `resources`:
    - `requests`: CPU/메모리 최소 보장량
    - `limits`: CPU/메모리 최대 사용량
    - 처음엔 두 값을 동일하게 설정하여 리소스 사용을 예측 가능하게 만드는 게 안정성에 유리

**예제 코드**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
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
      - image: my-repo/journal-server:vl-abcde
        imagePullPolicy: IfNotPresent
        name: frontend
        resources:
          requests:
            cpu: "1.0"
            memory: "1G"
          limits:
            cpu: "1.0"
            memory: "1G"
```

## 2. Service & Ingress (외부 트래픽 노출)

**문제**: 배포된 애플리케이션은 기본적으로 클러스터 내부에서만 접근 가능

**Service의 역할**:

- 여러 파드 그룹에 대한 안정적인 단일 엔드포인트 제공 (고정 IP + 포트)
- 들어오는 TCP 트래픽을 레이블 셀렉터에 맞는 여러 파드에 분산
- **내부 로드 밸런서** 역할

**Ingress의 역할**:

- HTTP/HTTPS 프로토콜에 특화된 리소스
- URL 호스트나 경로에 따라 요청을 서로 다른 서비스로 전달
- **지능형 라우터** 역할
- 단일 IP 주소로 여러 서비스 운영 가능

**예제 코드**:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
spec:
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fileserver
            port:
              number: 80

```

**동작 방식**: `/api` 경로 요청 → `frontend` 서비스, 나머지 `/` 경로 요청 → `fileserver` 서비스

## 3. StatefulSet (상태 저장 데이터베이스)

**사용 시점**: Redis 같이 데이터 상태를 유지해야 하는 애플리케이션

**StatefulSet 특징**:

- 파드가 재시작되어도 고유한 네트워크 ID 유지 (`redis-0`, `redis-1`)
- 순서가 있는 배포 및 스케일링
- 안정적인 데이터 관리 보장

**스토리지 관리**:

- **문제점**: 파드는 언제든 다른 노드로 재스케줄될 수 있어 데이터 유실 위험
- **해결책**:
    - **PersistentVolume (PV)**: 데이터를 원격 스토리지에 저장
    - **PersistentVolumeClaim (PVC)**: 애플리케이션이 필요로 하는 스토리지 사양 요청

**예제 코드**:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: "redis"
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:5-alpine
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

**포인트**: `volumeClaimTemplates`로 각 Redis 파드에 10Gi 영구 스토리지를 동적 할당

## 4. ConfigMap & Secret (구성 및 보안 관리)

### ConfigMap

**목적**: 애플리케이션 설정을 코드와 분리하여 유연성 향상

**사용 방식**: 환경 변수나 파일 형태로 파드에 전달

**모범 사례**:

- 기존 ConfigMap 수정보다는 `config-v1`, `config-v2`처럼 버전을 붙여 새로 생성
- 안전한 롤아웃 보장

### Secret

**목적**: 데이터베이스 암호 같은 민감한 정보를 안전하게 관리

**주의사항**:

- 소스 코드나 컨테이너 이미지에 포함하면 보안상 매우 위험
- 인메모리 파일 시스템(tmpfs) 볼륨으로 마운트하여 디스크 저장 방지

**예제 코드**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
# ...
  template:
    # ...
    spec:
      containers:
      - image: my-repo/journal-server:vl-abcde
        name: frontend
        volumeMounts:
        - name: passwd-volume
          readOnly: true
          mountPath: "/etc/redis-passwd"
        # ...
      volumes:
      - name: passwd-volume
        secret:
          secretName: redis-passwd
```

**동작**: Redis 암호가 담긴 Secret을 `/etc/redis-passwd` 경로에 볼륨으로 마운트

## 5. Helm (템플릿화를 통한 배포 관리)

**필요성**: 개발, 스테이징, 프로덕션 등 여러 환경에 동일 애플리케이션 배포 시 각 환경의 미세한 차이를 효율적으로 관리

**주요 개념**:

- **헬름 차트(Chart)**: 템플릿 파일들과 환경별 설정 값을 하나의 패키지로 묶은 것
- **템플릿 문법**: YAML 파일 내 변경 필요 부분을 `{{ .Values.변수명 }}` 형태로 처리
- **values.yaml**: 템플릿 변수에 들어갈 실제 값을 환경별로 정의

**예제**:

**Deployment 템플릿** (`frontend-deployment.tmpl`):

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
```

**값 파일** (`values.yaml`):

```yaml
replicaCount: 2
```

**배포**: `helm install` 명령어로 템플릿과 값을 조합하여 원하는 환경에 배포

---

**전체 흐름 정리**:

1. Stateless 앱 → **Deployment** (복제본 관리, 롤링 업데이트)
2. 외부 접근 → **Service** (안정적 엔드포인트) + **Ingress** (HTTP 라우팅)
3. Stateful 앱 → **StatefulSet** + **PersistentVolume** (데이터 영속성)
4. 설정 분리 → **ConfigMap** (일반 설정) + **Secret** (민감 정보)
5. 다중 환경 → **Helm** (템플릿화로 효율적 관리)