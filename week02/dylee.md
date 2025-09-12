# 3. 모니터링과 로깅
## 1. 메트릭 vs 로그

- 메트릭: 정해진 기간에 측정한 수치
- 로그: 에러, 경고, 중요 이벤트 등 프로그램 실행 중 일어난 사건을 추적

## 2. 모니터링

- 폐쇄형 모니터링
    - 애플리케이션 외부에서 모니터링
    - 인프라 수준의 컴포넌트를 모니터링 할 때 좋음 → cpu, memory, storage등
- 개방형 모니터링
    - 애플리케이션 상태 모니터링 → http 요청 수, 에러 횟수, 요청 레이턴시 등

## 3. 모니터링 패턴

- USE 방법론
    - 시스템의 리소스 제약, 에러 신속하게 파악 가능 → 모든 리소스에 대한 사용률 ,포화도, 에러 수 등
        - ex) 클러스터 노드의 네트워크 상태 모니터링 → 사용률, 포화도, 에러 수를 체크하여 네트워크 병목, 네트워크 스택 문제점 파악
    - 인프라 수준의 모니터링에 많이 사용
- RED 방법론
    - 애플리케이션의 엔드 유저 경험 위주로 모니터링 → 처리율, 에러 수, 처리 시간 등

## 4. k8s 메트릭 개요

- k8s cluster 구성
    - control plane
        - api 서버, etcd, scheduler, controller manager
    - node component
        - kubelet, container runtime, kube-proxy, kube-dns, pod

### 1. cAdvisor

- 노드에서 실행중인 컨테이너의 리소스와 메트릭을 수집하는 오픈 소스 구현체
- cluster의 모든 노드에서 실행됨
- cgroup 트리를 통해 메모리, cpu 메트릭을 수집
- linux kernel의 statfs로 디스크 메트릭 수집

### 2. 메트릭 서버

- 메트릭 서버
    - 리소스 메트릭 api의 표준 구현체
    - kubelet API가 cpu, 메모리 등의 리소스 메트릭 수집 → 메모리에 저장 → 스케줄러, HPA, VPA에서 사용(오토스케일러들)
- 커스텀 메트릭 API
    - 모니터링 시스템에서 임의의 메트릭을 수집할 수 있음
    - 프로메테우스 → 커스텀 메트릭 기반으로 오토스케일링(HPA)가능

### 3. kube-state-metrics

- 쿠버네티스에 저장된 오브젝트를 모니터링 → pod, deployment, node, jobs 확인

## 5. 어떤 메트릭을 모니터링?

- 노드 → cpu 사용률, 메모리 사용률, 네트워크 사용률, 디스크 사용률
- 클러스터 컴포넌트 → etcd 레이턴시
- 클러스터 애드온 → 클러스터 오토스케일러, 인그레스 컨트롤러
- 애플리케이션 → 컨테이너 cpu, 메모리, 네트워크 사용률, application framework 메트릭 등

## 6. 프로메테우스를 이용한 쿠버네티스 모니터링

- 프로메테우스 서버
    - 시스템에서 수집된 메트릭을 가져와 저장
- 프로메테우스 오퍼레이터
    - 프로메테우스 구성, 관리, 운영
- 노드 익스포터
    - 클러스터 내 쿠버네티스 노드에서 수집한 호스트 메트릭 익스포트
- kube-state-metrics
    - 쿠버네티스 관련 메트릭 수집

## 7. 로깅 개요

- 무엇을 로깅해야 할까?
    - 노드 로그
    - control plane log → api server, 컨트롤러 매니저, 스케줄러
        - 컨트롤 플레인 로그 수집 → 컴포넌트 내부의 근본적인 문제점을 알 수 있음
        - /var/log/kube-APIserver.log
    - 쿠버네티스 감사 로그
    - 애플리케이션 컨테이너 로그
        - 중앙 저장소로 포워딩 필요

## 9. 모니터링, 로깅, 알림

### 모니터링

- 모든 노드, 쿠버네티스 컴포넌트 상태로 사용률, 포화도, 에러율 모니터링 하기
- 애플리케이션 사용률, 에러 수, 실행 시간 모니터링 하기
- 예측 어려운 상태, 징후 → 폐쇄형 모니터링
- 시스템과 내부 → 개방형 모니터링

### 로깅

- 되도록이면 로그 전달기에 데몬셋 사용 + stdout으로 로그 보내기
# 4. 구성, 시크릿, RBAC 
## 1. 컨피그맵과 시크릿을 통한 구성

파드 수신 정보 저장, 데이터 etcd 저장

### 1.ConfigMap

- 컨테이너를 이용해 구성 정보와 애플리케이션을 떼어놓을 수 있음
- configmap api로 애플리케이션에 구성 정보 주입
    - pod 구성 정보, 컨트롤러, crd, 오퍼레이터 등.. → 민감하지 않은 문자열 데이터
- configMap data를 애플리케이션에서 사용하려면 pod에 볼륨 마운트 or 환경 변수로 주입

### 2. Secret

- pod에 시크릿 주입 → 복호화 → 일반 텍스트
- 시크릿 타입
    - generic: key-value 쌍
    - docker-registry: private docker registry 인증에 필요한 자격증명 시크릿을 kubelet이 pod template에 전달 → private image pull 가능
    - tls
        - public/private key 쌍으로 생성한 전송 레이어 보안 시크릿 → cert 경로에 올바른 PEM 포맷의 인증서 파일이 있으면 키 쌍은 시크릿으로 인코딩된 후 ssl/tls 사용하는 파드에 전달
- secert는 자신을 필요로 하는 파드가 있는 노드에서만 tmpfs에 마운트 → 파드 사라지면 시크릿도 함께 삭제됨
- 시크릿은 etcd 저장소에 일반 텍스트로 저장됨 → etcd 보안에 신경 써야 함
    - KMS, kubeseal 등을 사용하자

## 2. 컨피그맵, 시크릿 api 모범 사례

- 새 버전의 pod를 재배포 하지 않고 애플리케이션을 변경한다면
    
    → configMap과 Secret를 별도의 볼륨으로 마운트
    
- 파드가 배포되기 전에 configMap/Secret는 자신을 사용할 pod의 namespace에 존재해야 함
- admission controller를 사용해서 deployment가 배포되기 전에 정책 강제
    - 특정 구성 데이터가 존재하도록 강제하거나 특정 구성 값이 없다면 배포 차단 가능
- helm →  life cycle hook을 걸어 디플로이먼트 배포 전에 configmap/secret template 배포되도록 하기
- 환경변수 사용할 때 컨피그맵 데이터 주입 방법
    - envFrom → configMap의 모든 key/value 쌍을 환경 변수로 마운트해서 pod에 주입
    - configMapRef, secertRef사용하여 key별로 값 할당
- **configMap/Secret를 환경변수로 주입하고 데이터를 수정한다면 → 파드를 재시작해야 반영됨 → Deployment 업데이트 할 것**

## 3. 시크릿 모범 사례

- workload에서 k8s api에 access할 필요가 없다면 서비스 어카운트에 대한 api 크리덴셜이 자동으로 마운트 되지 않게 차단하기 → 컨트롤 플레인 호출을 줄일 수 있음
    - ex) 모니터링, ci/cd 도구 등등은 k8s api에 access해서 k8s cluster 정보를 읽어 와야 하지만 일반적인 애플리케이션들은 k8s api에 access할 필요가 없음 → 이것들은 k8s api에 접근하지 못하도록 차단해 주자
    
    ```java
    apiVersion: v1
    kind: ServiceAccount
    metadata:
    	name: app1-svacct
    autoServiceAccountToken: false
    ```
    
- imagePullSecrets를 서비스 어카운트에 할당 → pod.spec에 선언하지 않아도 파드가 시크릿 자동 마운트
- 외부 스토리지 시스템에 시크릿 데이터 보관 가능
- 데브옵스 프로세스
    - 보안팀 → 시크릿 생성, 암호화
    - 개발자 → 시크릿 네임 참조 → ci/cd로 HSM으로 암호화된 보안 저장소에서 시크릿 가져오기

## 4. RBAC

- 인증(여권)
    - k8s cluster 유저 인증 → 외부 권한에 의존, 서비스 계정은 k8s가 직접 관리
- 인가(비자 또는 여행 정책)
    - 인가 방법으로 RABC가 많이 사용됨
- admission controller(세관 출입국 관리)
    - 정의된 규칙, 정책에 따라 api 요청 허용/거부

### 1. RBAC → 주체가 어떤 작업을 할 수 있는지 권한을 관리하는 시스템

- 주체
    - 액세스를 확인할 대상 → 서비스, 어카운트, 그룹
    - 유저와 그룹은 bear token 등 외부의 인가 모듈을 사용해서 처리
- 규칙
    - api에 있는 어떤 오브젝트(리소스)나 오브젝트 그룹에서 실제로 수행 가능한 액션 리스트
    - 오브젝트 → 다양한 api 컴포넌트
- 롤
    - 규칙의 범위
    - role → 한 네임스페이스
    - clusterRole → 모든 ns에 적용
- 롤바인딩
    - 주체(유저,그룹)을 특정 롤에 매핑한 것
    - roleBinding: 특정 ns에 적용
    - clusterRoleBinding: 모든 cluster에 적용

### 2. RBAC 모범 사례

- 애플리케이션 코드가 k8s api와 직접 인터랙션 하는 경우에만 RBAC 필요
    - kubectl로 k8s api호출해서 하는 작업들
- 애플리케이션이 필요로 하는 k8s api 작업에 맞는 role 부여하기
- 짧은 기간 동안 권한을 승격시켜야 할 유저가 있다면 → 적시 액세스 시스템 사용하기
- ci/cd 계정은 별도로 마련할 것
- 시크릿 api의 watch와 list를 필요로 하는 애플리케이션 삼가기
