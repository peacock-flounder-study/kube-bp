# 03. 모니터링과 로깅

### 모니터링 개요
- 폐쇄형 모니터링: 애플리케이션 외부에서 관찰 → 인프라 수준 파악엔 유용, 앱 동작 인사이트는 부족  
- 개방형 모니터링: 애플리케이션 내부 상태/엔드 유저 경험 집중  

#### 모니터링 패턴
- k8s는 수명이 짧고 다이나믹한 파트 특성 잘 나타내는 모니터링 체계 필요.

1. USE 방법론  
   - U(사용률), S(포화도), E(에러수)  
   - 인프라 지표 중심  
2. RED 방법론  
   - R(처리율), E(에러수), D(처리시간)  
   - 사실상 구글 4대 시그널과 동일 → 엔드 유저 경험 중심  

→ 두 방법론은 상호보완 관계  

### 쿠버네티스 메트릭 개요
이제 쿠버 클러스터에서 어떤 컴포넌트 모니터링할지 보기.  

클러스터와 애플리케이션 정상 가동하려면 모든 컴포넌트 빠짐없이 모니터링해야함.  

- **cAdvisor**: 노드에서 실행중인 컨테이너 리소스 메트릭 수집 (진실 공급원)  
- **Metrics Server**  
  1. kubelet API에서 리소스 메트릭 수집 → 메모리에 저장  
  2. Custom Metrics API 지원 → 프로메테우스 같은 외부 시스템에서 임의 메트릭 수집 가능  
- **kube-state-metrics**: 쿠버네티스 오브젝트 상태 모니터링  

### 어떤 메트릭을 모니터링하나?
계층적으로 접근 필요  

- 물리/가상 노드  
- 클러스터 컴포넌트  
- 클러스터 에드온  
- 엔드 유저 애플리케이션  

### 모니터링 툴
- Prometheus, InfluxDB, Datadog, Sysdig .. (생략)

#### 프로메테우스
- Kubernetes 메타데이터·레이블과 궁합 좋음  
- Exporter → 다양한 서비스 메트릭 변환  
- Operator → 프로메테우스/Alertmanager 네이티브 관리  
- Alertmanager → 알람 전송, 외부 시스템 연동  
- Grafana → 시각화  

### 로깅 개요
- 로그는 운영 필수 → 하지만 과도하면 노이즈 + 비용 폭증  
- 그래서 **보존 정책 필수** (보통 30~45일이면 충분)  

#### 무엇을 로깅해야 하는가 
- 노드 로그: 노드 이벤트  
- 컨트롤 플레인 로그: apiserver, controller-manager, scheduler → `/var/log/...`  
- 감사 로그: 누가/언제/무슨 요청 했는지 (보안용, 노이즈 많음)  
- 애플리케이션 로그: STDOUT 권장 / 파일 기록 시 사이드카 필요  

#### 로그 수집 방식
- 데몬셋 기반 수집기 → 모든 노드 로그 자동 수집  
- 사이드카 패턴 → 앱이 파일로 로그 쓰는 경우 사용  

### 로깅 툴
- Loki → lightweight, 프로메테우스와 찰떡  
- Elastic Stack(ELK) → 강력하지만 무거움  
- Datadog / Sumo Logic / Sysdig → SaaS 기반, 운영 부담 ↓  
- CloudWatch, Stackdriver, Azure Monitor 등 클라우드 제공 툴  

운영 전략: 로그 중앙화는 SaaS hosted-service 쪽이 효율적 (규모 커질수록 관리 비용 ↓)  
(해봐야 제대로 이해될만한 게 많은 듯)


# 04. 구성, 시크릿, RBAC
> 아주 간단하게 정리했습니다.. 

### 컨피그맵과 시크릿을 통한 구성
#### ConfigMap
- 민감하지 않은 설정값 관리  
- Pod에 볼륨/환경 변수 형태로 주입  
- 애플리케이션과 설정을 분리 → 이식성 확보  

#### Secret
- 민감한 데이터 저장  
- 기본 etcd에 평문 저장됨 → 보안 강화 필요  
- tmpfs 기반으로 노드에만 마운트, Pod 종료 시 사라짐  
- 타입:  
  - generic (key-value)  
  - docker-registry (프라이빗 레지스트리 인증)  
  - tls (공개키/개인키 쌍)  

### 컨피그맵, 시크릿 API 모범 사례
- Pod보다 먼저 존재해야 함  
- 수정해도 자동 반영 안 됨 → Pod 재시작 필요  
- Admission Controller로 정책 강제 가능  
- Helm hook으로 배포 순서 제어  
- 일반 앱은 k8s API 접근 필요 없음 → `automountServiceAccountToken: false` 설정  
- 외부 스토리지(Vault, AWS Secrets Manager 등) 활용 가능  

### RBAC 개요
- 인증: 유저(외부 시스템), 서비스 계정(k8s 자체)  
- 인가: RBAC 사용  

구성 요소  
- 주체: User / Group / ServiceAccount  
- 규칙: 어떤 리소스에 어떤 동작 허용 (CRUD, watch, list 등)  
- Role / ClusterRole → 권한 범위  
- RoleBinding / ClusterRoleBinding → 권한 할당  

### RBAC 모범 사례
- 최소 권한 원칙 적용  
- CI/CD 전용 계정 분리  
- 일반 애플리케이션은 k8s API 접근 차단  
- 짧은 기간 고권한 필요 시 JIT Access 사용  
