# Kubernetes 모니터링 및 로깅 공부 정리

Kubernetes같은 동적이고 일시적인 시스템을 모니터링하려면 기존 방식과는 다른 접근이 필요함. 
메트릭과 로그를 상호 보완적으로 사용해서 문제를 진단하고 해결하는 게 핵심임.

## 3.2. 메트릭과 로그

**메트릭**

- ㅌㅈ일정 기간 동안 측정된 수치 데이터
- 시스템의 전반적인 상태와 성능 추세 파악용
- 예: CPU 사용률, 메모리 사용량, 응답 시간

**로그**

- 프로그램 실행 중 발생하는 이벤트 기록
- 특정 문제의 근본 원인 분석용
- 예: 오류 메시지, 경고, 주요 이벤트

**상호 보완성**
애플리케이션 성능이 저하될 때

1. 메트릭으로 지연 시간 증가 감지 (첫 번째 징후)
2. 로그로 실제 오류 원인 파악 (근본 원인 분석)

## 3.3. 모니터링 기법 및 패턴

### 기존 VM vs Kubernetes 환경

- **VM**: 24/7 가동, 정적, 상태 보존
- **Kubernetes**: 동적, 일시적, 파드 생성/소멸 반복
- 따라서 모니터링 접근 방식도 달라져야 함

### 모니터링 접근 방식

**폐쇄형 모니터링 (Black Box)**

- 애플리케이션 외부에서 모니터링
- CPU, 메모리, 스토리지 등 구성 요소 중심
- 한계: 애플리케이션 작동 방식에 대한 인사이트 부족

**오픈 박스 모니터링 (White Box)**

- 총 HTTP 요청 수, 500 오류 수, 요청 지연 시간 등
- 애플리케이션 상태의 맥락에서 세부 사항 중심
- 시스템 상태의 '원인'을 이해하는 데 도움이 됩니다.

### 모니터링 방법론

**USE 방법 (인프라 모니터링)**

Brenden Gregg가 널리 보급한 방법

- **U**tilization (활용률): 리소스 사용 정도
- **S**aturation (포화도): 리소스 포화 상태
- **E**rrors (오류): 오류율
- 모든 리소스에 대해 사용률, 포화도, 오류율 확인
- **시스템 리소스 제약과 오류율 신속 파악 가능 ⭐**

**RED 방법 (애플리케이션 모니터링)**
Tom Wilkie가 대중화, Google의 4가지 황금 신호에서 영감

- **R**ate (요청 수): 처리하는 요청의 양
- **E**rrors (오류 수): 실패하는 요청의 비율
- **D**uration (지연 시간): 요청 처리 시간
- 사용자의 서비스 경험에 초점

> **핵심**: USE는 **인프라**, RED는 **애플리케이션 최종 사용자 경험** 모니터링에 각각 특화되어 상호 보완적
> 

## 3.4. Kubernetes 메트릭 개요

Kubernetes 클러스터는 컨트롤 플레인과 노드 구성 요소로 구성되고, 모든 구성 요소 모니터링이 필요함.

### 주요 구성 요소

**cAdvisor (Container Advisor)**

- 노드에서 실행되는 컨테이너 리소스와 메트릭 수집
- kubelet에 내장됨 (모든 노드에서 실행됨)
- Linux cgroup 통해 메모리 및 CPU 메트릭 수집
- 모든 컨테이너 메트릭의 신뢰할 수 있는 출처

**메트릭 서버**

- Heapster 대체 (데이터 싱크 구현 방식 내 아키텍처적 단점 해결)
- CPU 및 메모리 같은 리소스 메트릭을 kubelet API에서 수집, 메모리에 저장
- Kubernetes 스케줄러, HPA, VPA에서 사용
- 사용자 정의 커스텀 메트릭 API 지원으로 커스텀 어댑터를 만들어 붙이면, 코어 리소스 메트릭 확장 가능
    - Prometheus 등 도구 확장 가능

**kube-state-metrics**

- Kubernetes에 저장된 오브젝트 모니터링하는 k8s addon
    - 리소스 사용량이 아닌 Kubernetes 오브젝트 "상태" 식별
    - 예: 배포된 파드 수, 보류 중인 파드, 노드 상태, deployment 업데이트 등
- cAdvisor와 메트릭 서버가 상세 메트릭 제공한다면, 얘는 클러스터 상태 정보 제공

### 계층화된 모니터링 접근

너무 많은 걸 모니터링하면 노이즈 발생 → 계층적 접근 필요

1. **물리적/가상 노드**: CPU, 메모리, 네트워크, 디스크 사용률
2. **클러스터 컴포넌트**: etcd 지연 시간
3. **클러스터 애드온**: 클러스터 자동 스케일러, 인그레스 컨트롤러
4. **end user 애플리케이션**: 컨테이너 메모리/CPU 사용률, 애플리케이션별 메트릭

계층적 접근 예 : 파드 문제(pending) → 노드 리소스 사용률 → 클러스터 수준의 컴포넌트 타겟팅

## 3.6. 모니터링 도구

**오픈소스 도구**

**Prometheus**

- CNCF 호스팅 오픈소스 프로젝트
- 키쌍으로 다차원 데이터 모델 구현 (Kubernetes 라벨링과 유사)
- 메트릭 엔드포인트 스크래핑하는 pull 모델
    - 프로메테우스 서버가 k8s cluster endpoint로부터 메트릭을 pull
    - alert manager에 push → if you want → slack pager duty
    - USE 방법론을 이용한 노드 메트릭 수집
- 많은 Kubernetes 생태계 프로젝트가 Prometheus 형식으로 메트릭 노출
- 사람이 읽을 수 있는 형식으로 메트릭 노출
    
    ```
    # HELP node_cpu_seconds_total Seconds the CPU is spent in each mode.
    # TYPE node_cpu_seconds_total counter
    node_cpu_seconds_total{cpu="0",mode="idle"} 5144.64
    ```
    
- **Prometheus server :** 시스템 수집 메트릭 가져와 저장
- **Prometheus Operator**: Kubernetes 네이티브하게 Prometheus 관리
- **Node Exporter**: Kubernetes 노드에서 호스트 메트릭 내보내기
- **Kube-state-metrics**: k8s 관련 메트릭 수집
- **Alert Manager** : 알림 설정 후, 외부 시스템 포워딩
- **Grafana**: Prometheus 데이터 시각화
- **InfluxDB**: 높은 쓰기/쿼리 부하 처리용 시계열 DB

**상용 도구**

- **Datadog**: 클라우드 규모 애플리케이션용 SaaS 모니터링
- **Sysdig**: 컨테이너 네이티브 앱 모니터링

**클라우드 provider 도구**

- **GCP Stackdriver**: GKE 클러스터용 맞춤 대시보드
- **Azure Monitor**: AKS 컨테이너 워크로드 성능 모니터링
- **AWS Container Insights**: ECS/EKS 메트릭과 로그 수집

## 3.8. 로깅 개요

메트릭과 함께 로깅은 Kubernetes 환경을 전체적으로 파악하는데 필수적.

**로깅의 문제점**

모든 것을 로깅하자… 그 것은 아래의 문제를 초래

1. **노이즈**: 너무 많으면 실제 문제 찾기 어려움
2. **비용**: 많은 리소스 소모, 비용 발생

### 로그 유형별 수집 대상

k8s cluster에서 로그 수집할만한 컴포넌트들 리스트업

**노드 로그**

- Docker 데몬 같은 필수 노드 서비스 이벤트 수집
- 컨테이너 실행을 위한 정상적인 Docker 데몬 필수
- 데몬 자체의 근본적인 문제 진단

**Kubernetes 컨트롤 플레인 로그**

- API 서버, 컨트롤러 매니저, 스케줄러
- 클러스터 내부 근본 문제에 대한 통찰력 제공
- /var/log/kube-APIserver.log, /var/log/kube-scheduler.log, /var/log/kube-controller-manager.log 에서 집계

**Kubernetes audit 로그**

- 시스템 내에서 누가 어떤 작업 수행했는지 인사이트
- 보안 모니터링용이지만 감사 로그 특성 상, 매우 노이즈 많음
- 환경에 맞게 조정 필요

**애플리케이션 컨테이너 로그**

- 애플리케이션에서 발생하는 실제 로그
- STDOUT으로 전송하거나 사이드카 패턴 사용

### 로그 수집 패턴

**권장: STDOUT + DaemonSet**

- 모든 애플리케이션 로그를 STDOUT으로 전송
- 모니터링 데몬셋이 Docker 데몬에서 직접 로그 수집
- 애플리케이션을 일관되게 로깅하는 방식, 리소스 효율적

**사이드카 패턴**

- 애플리케이션 컨테이너 옆에 로그 포워딩 컨테이너 실행
- 앱이 파일 시스템에 로그 기록하는 경우에만 사용
- 더 많은 리소스 사용

**보존 및 보관 정책 (참고)**

- **권장 보존 기간**: 30-45일
- **장기 보관**: 규정 준수 등 필요시 저비용 리소스 사용
- 저장 로그 수 증가 문제 해결 위한 정책 필요

## 3.9 로깅 툴

툴마다 로깅을 어떻게 구현했는지 잘 살펴봐야함

(Daemonset, sidecar pattern 등)

> 오픈소스
> 

**Loki**

- Grafana 제공 로깅 시스템
    - Loki, Promtail, Grafana로 구성됨
- Prometheus와 유사한 레이블 기반 인덱싱/쿼리 제공

**Elastic Stack**

- Elasticsearch, Logstash, Kibana
- 널리 사용되는 중앙집중식 로깅 솔루션

> 상용/클라우드
> 
- **Datadog, Splunk, Sysdig**: 상용 로깅 솔루션
- **클라우드 서비스**: GCP Stackdriver, Azure Monitor, CloudWatch

로그 중앙화 시, 호스티드 솔루션이 운영 비용을 많이 덜어줌. 

자체 호스팅은 초기엔 좋지만 환경 커지면 유지보수 복잡해짐.

## 3.11 알림

알림은 신중히 관리해야 하는 "양날의 검". 

보내야할 것과 모니터링만 해야할 것 간 균형

**알림 피로 방지**

- 너무 많은 알림 → 중요한 이벤트가 노이즈에 묻힘
- 파드 실패마다 알림은 부적절 (Kubernetes가 자동 재시작함)

**SLO 기반 알림**

- 서비스 수준 목표(SLO)에 영향 미치는 이벤트만
- 최종 사용자(엔드 유저) 경험에 초점
- 예: FE의 경우, SLO가 20ms 응답시간인데 평균보다 높은 지연 발생 시

**즉각적 조치 필요성**

- 대기 중인 엔지니어가 즉시 개입 필요한 문제만
- 애플리케이션 UX에 영향 미치는 문제인 경우에는 사람이 즉시 대응 필요
- 하지만 "저절로 해결된" 문제는 알림 불필요

### 알림 설계 모범 사례

**문제의 원인 해결 및 조치 자동화**

- 즉각적 조치 불필요한 일시적 문제는 자동화
- 예: 디스크 가득 참 → 로그 자동 삭제 (rog rotation or flush)
- Kubernetes 라이브니스 프로브로 응답 없는 프로세스 자동 재시작

**Alert 임계값 설정**

- 너무 자주 발생하는 오탐 방지 위해 5분 이상으로 설정
- 표준 임계값: 5분, 10분, 30분, 1시간 패턴 사용

**알림 정보**

- 알림 내 유의미한 정보 포함 하면 좋음
    - like 문제 해결 가이드('플레이북') 링크
    - 데이터센터, 지역, 앱 소유자, 영향받는 시스템 정보

**알림 채널**

- 문제 책임질 사용자에게 직접 라우팅
- 대규모 그룹 전송은 노이즈로 간주되어 필터링됨

**점진적 개선**

- 첫날부터 완벽할 수 없음
- 알림 피로 방지하며 점진적으로 개선
- 직원 피로와 시스템 문제 방지

## 3.12. 모범 사례

> 모니터링
> 
- USE/RED 방법으로 노드, Kubernetes 구성요소, 애플리케이션 모니터링
    - 노드, 쿠버네티스 컴포넌트: 사용률, 포화도, 에러율
    - 앱 : 사용률, 에러수, 실행 시간
- 예측하기 어려운 상태, 질후 → 폐쇄형 모니터링
- 오픈 박스 모니터링으로 시스템 내부 검사
- 시계열 메트릭과 Prometheus 같은 고차원성 시스템 활용하여 앱을 잘 분석
- 사실 데이터는 평균 메트릭
- 합계 메트릭으로는 특정 메트릭 분포 시각화

> 로깅
> 
- 메트릭과 함께 사용하여 환경 전체적 파악
- 30-45일 로그 보존, 필요시 저렴한 리소스로 장기 보관
- STDOUT + DaemonSet 방식 선호
- 사이드카 패턴 리소스 사용 제한
    - 로그 전달기에 데몬셋 + STDOUT을 사용

> 알림
> 
- 알림 피로 방지
- SLO 및 고객 영향 증상에 대해서만 알림
- 일시적 문제는 자동화 집중
- 관련 정보 포함, 책임자에게 라우팅

## 결론

Kubernetes 환경 모니터링은 처음부터 다시 생각해서 접근해야 함. 사후 구현하면 시스템 이해 방해됨.

**Our Goal**: 시스템 통찰력 확보, 복원력 향상, 궁극적으로 애플리케이션의 더 나은 최종 사용자 경험 제공

분산 애플리케이션과 Kubernetes 같은 분산 시스템 모니터링은 많은 작업 필요하므로 도입할 때부터 준비가 필요하다는 사실.


---
# **Kubernetes 구성, 암호 및 RBAC**

Kubernetes에서 **구성과 코드의 엄격한 분리**는 클라우드 네이티브 애플리케이션 개발의 핵심 원칙

컨테이너의 컴포저블(조합 가능한) 특성을 활용하여 런타임에 구성 데이터를 주입함으로써 애플리케이션 기능과 실행 환경을 분리 가능

민감한 데이터가 Kubernetes API 객체로 이동할 때는 RBAC(역할 기반 접근 제어)를 통한 API 접근 보호가 필수적

### ConfigMap - 비민감 구성 데이터 관리

- **용도**: 민감하지 않은 구성 데이터 저장
- **지원 형식**:
    - 키/값 쌍
    - JSON, XML 등 복잡한 벌크 데이터
    - `|` 기호를 사용한 전체 블록 원시 데이터

사용 대상

- 파드 (Pods)
- 컨트롤러 (Controllers)
- CRD (Custom Resource Definitions)
- 운영자 (Operators)
- 기타 복잡한 시스템 서비스

주입 방법

1. **볼륨 마운트**: 파일 시스템으로 마운트
2. **환경 변수**:
    - `envFrom`: 모든 키/값 쌍을 환경 변수로 로드
    - `configMapKeyRef`: 개별 키의 값을 특정 환경 변수에 할당
3. **명령줄 인수**: `$(ENV_KEY)` 보간 구문 사용

예시 구성

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-http-config
  namespace: myapp-prod
data:
  config: |
    http {
      server {
        location / {
          root /data/html;
        }
        location /images/ {
          root /data;
        }
      }
    }

```

### Secret - 민감 데이터 관리

- **인코딩**: Base64 (암호화 아님)
- **크기 제한**: 1MB (실제 데이터 약 750KB)
- **보안 특징**:
    - 필요한 노드의 tmpfs에만 마운트
    - 파드 삭제 시 자동 제거
    - 노드 디스크에 남지 않음

etcd 저장 보안

- **중요한 보안 고려사항**: 기본적으로 Secret은 etcd에 평문으로 저장
- **보안 강화 방법**:
    - etcd 노드 간 mTLS 설정
    - etcd 데이터의 미사용 암호화 활성화
    - KMS(Key Management System) 제공자 사용
    - 외부 시크릿 관리 시스템 활용

- Secret 유형
    
    1. Generic Secret
    
    ```bash
    kubectl create secret generic mysecret \
      --from-literal=key1=$3cr3t1 \
      --from-literal=key2=@3cr3t2
    ```
    
    2. Docker Registry Secret
    
    ```bash
    kubectl create secret docker-registry registryKey \
      --docker-server=myreg.azurecr.io \
      --docker-username=myreg \
      --docker-password=$up3r$3cr3tP@ssw0rd \
      --docker-email=ignore@dummy.com
    
    ```
    
    3. TLS Secret
    
    ```bash
    kubectl create secret tls www-tls \
      --key=./path_to_key/wwwtls.key \
      --cert=./path_to_crt/wwwtls.crt
    ```
    

### ConfigMap/Secret 공통 모범 사례

**동적 변경 지원**

- **권장 방법**:
    - ConfigMap/Secret을 **볼륨으로 마운트**
    - 애플리케이션에 **파일 워처** 구성
    - 변경된 파일 데이터 감지 및 재구성 자동화
- **주의사항**:
    - `volumeMounts.subPath` 속성 사용 금지 (동적 업데이트 불가)
- 배포 전 검증
    - ConfigMap/Secret은 파드 배포 **이전에** 해당 네임스페이스에 존재해야 함
    - 선택적 플래그를 사용하여 존재하지 않을 경우 파드 시작 방지 가능
    - 어드미션 컨트롤러를 통한 특정 구성 데이터 보장
- CI/CD 통합
    - **구성 변경 반영 전략**:
        1. ConfigMap/Secret의 `name` 속성 업데이트
        2. 배포의 참조도 함께 업데이트
        3. Kubernetes 업데이트 전략을 통한 배포 트리거
- **Helm 사용 시**
    - annotation을 이용해서 컨피그맵,시크릿의 sha256 checksum을 확인
        
        ```yaml
        annotations:
          checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        ```
        
    - 위 데이터가 변경 시, helm upgrade를 통해 deployment 알아서 업데이트함
- 환경 변수 매핑 규칙
    - configMapKeyRef/secretKeyRef 사용 시:
        - 해당 키가 존재하지 않으면 **파드 시작 실패**
- envFrom 사용 시:
    - 유효하지 않은 환경 변수 키는 건너뛰지만 **파드는 시작됨**
        - 이벤트에 `InvalidVariableNames` 오류 기록

### Secret 전용 모범 사례

- API 자격 증명 자동 마운팅 제어
    
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: app1-svcacct
    automountServiceAccountToken: false
    
    ```
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: app1-pod
    spec:
      serviceAccountName: app1-svcacct
      automountServiceAccountToken: false
    
    ```
    
- 외부 시크릿 관리 시스템
    - **권장 솔루션**:
        - HashiCorp Vault
        - AWS Secrets Manager
        - Google Cloud KMS
        - Azure Key Vault
        - **ExternalSecrets Operator** (Linux 재단 프로젝트)

- imagePullSecrets 자동 할당
    
    ```bash
    # Docker registry secret 생성
    kubectl create secret docker-registry registryKey \
      --docker-server=myreg.azurecr.io \
      --docker-username=myreg \
      --docker-password=$up3r$3cr3tP@ssw0rd \
      --docker-email=ignore@dummy.com
    
    # 기본 서비스 계정에 패치
    kubectl patch serviceaccount default -p \
      '{"imagePullSecrets": [{"name": "registryKey"}]}'
    
    ```
    
- 보안 책임 분리
    - **보안 관리팀**: Secret 생성 및 암호화
    - **개발팀**: 예상되는 Secret 이름만 참조
    - **CI/CD 도구**: 하드웨어 보안 모듈(HSM) 활용

## RBAC (Role-Based Access Control)

- 주체 (Subject)
    1. **사용자**: 외부 인증 모듈에서 관리
        - 기본 인증
        - x.509 클라이언트 인증서
        - 베어러 토큰 (OpenID Connect 등)
    2. **서비스 계정**: Kubernetes 내부 관리
        - 네임스페이스에 바인딩
        - 프로세스용 (사람이 아닌)
    3. **그룹**: 사용자 그룹

- 규칙 (Rules)
    - **동사**: `create`, `get`, `update`, `delete`, `watch`, `list`, `exec`
    - **리소스**: API 객체 (pods, deployments 등)
    - **API 그룹**: 코어 API (`apiGroup: ""`), apps API 등
- 역할 (Roles)
    - **Role**: 네임스페이스 범위
    - **ClusterRole**: 클러스터 전체 범위
- 역할 바인딩 (Role Bindings)
    - **RoleBinding**: 네임스페이스 범위
    - **ClusterRoleBinding**: 클러스터 전체 범위
- Role 예시
    
    ```yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      namespace: default
      name: pod-viewer
    rules:
    - apiGroups: [""] # 코어 API 그룹
      resources: ["pods"]
      verbs: ["get", "watch", "list"]
    ```
    
- RoleBinding 예시
    
    ```yaml
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: noc-helpdesk-view
      namespace: default
    subjects:
    - kind: User
      name: helpdeskuser@example.com
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: pod-viewer
      apiGroup: rbac.authorization.k8s.io
    ```
    

## RBAC 모범 사례

애플리케이션 RBAC 원칙

**기본 규칙**:

> "Kubernetes에서 실행되도록 개발된 애플리케이션에는 RBAC 역할 및 이와 관련된 역할 바인딩이 거의 필요하지 X. 
애플리케이션 코드가 Kubernetes API와 직접 상호 작용하는 경우에만 RBAC 구성이 필요"
> 
> 
> 계층적 보안 구조
> 
> 1. **인증 (Authentication)**: 신원 확인
> 2. **권한 부여 (Authorization)**: RBAC를 통한 접근 제어
> 3. **입학 제어 (Admission Control)**: 정책 기반 요청 검증
- 최소 권한 원칙
    - 새 서비스 계정 생성
    - 목표 달성에 **필요한 최소한의 권한**만 부여
    - 파드 사양에 서비스 계정 지정
- ID 관리 및 인증
    - **권장 구성**:
        - **OpenID Connect** 서비스 사용
        - **2단계 인증** 지원
        - 사용자 그룹을 최소 권한 역할에 매핑
- 특별 권한 관리
    - **적시 액세스 (JIT) 시스템**:
        - SRE (Site Reliability Engineers)
        - 운영자
        - 단기간 에스컬레이션 권한이 필요한 사용자
        - 더 엄격한 감사가 적용되는 별도 ID 사용

- CI/CD 도구 관리
    - **전용 서비스 계정** 사용
    - 클러스터 내 감사 추적 가능
    - 배포/삭제 주체 식별 가능

- Helm 관련 보안
    - Helm v2 (Tiller) 사용 시
        - **Tiller**: Kubernetes 클러스터 내부에서 실행되는 서버 컴포넌트
            - **Tiller가** Helm 차트의 실제 배포 수행, Kubernetes API와 직접 통신
        - `kube-system`의 기본 Tiller 사용 금지 (cluster-admin 권한으로 실행)
        - **각 네임스페이스별 Tiller 전용 서비스 계정** 생성 (보안상 권장)
        - 네임스페이스별 Tiller 배포
    - 권장사항
        - **Helm v3으로 마이그레이션** - 클라이언트 기반 아키텍처로 Tiller 불필요
        - **Helm v3의 장점**:
            - 완전히 **클라이언트 기반 아키텍처**
            - 보안 복잡성 제거
            - RBAC 권한 단순화
            - 네임스페이스별 Tiller 관리 불필요
            - 더 나은 멀티테넌시 지원

- Secret API 접근 제한
    - **엄격한 제한 정책**:
        - `watch` 및 `list` 권한을 필요한 애플리케이션으로만 제한
        - 특정 Secret 접근 시 `get` 사용을 직접 할당된 Secret으로만 제한
        - 애플리케이션이 해당 네임스페이스의 모든 Secret을 볼 수 없도록 방지