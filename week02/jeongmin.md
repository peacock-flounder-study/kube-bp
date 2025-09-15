# CH3
## 새롭게 알게 된 것
### 모니터링 방식의 차이
- Black Box monitoring: 직접 되나 안되나 테스트 해봄
- White Box monitoring: Application의 상태(다양한 정보)를 확인하고 그것에 대한 진단을 함

### monitoring pattern
- USE pattern (Infra monitoring에 적합)
    - 사용률, 포화도, 오류율
- RED pattern (Application monitoring에 적합)
    - 요청, 오류율, 소요 시간

### Monitoring의 대상 & 방법
- 대상
    - Control plane: API Server, etcd scheduler, controller manager
    - Worker node: kubelet, container runtime, kube-proxy, kube-dns, pod
- 방법
    - cAdvisor
        - kubelet 자체에 내장
        - linux cgroups tree를 통해 cpu, mem 모니터링 진행
    - metric server & API
        - kubelet API (=metric server 그잡채) -> Memory에 저장 -> VPA,HPA등에 활용
        - Prometheus 등을 통해 외부 메트릭으로 확장 가능
    - kube-state-metrics
        - 실제 K8s에서 동작하고 있는 오브젝트의 상태를 확인할 수 있게 함

### 도구
- prometheus : 보편적
- influxDB : IoT 에 특화 (Edge AI 하던 회사에서 자주 쓰던 것...)
- Datadog: 안써봤는데 함 써보고 싶음
- Azure Monitor: AKS 모니터링 해줌
- Container Insights: ECS, EKS, EC2위의 K8s ... (어느정도 수준인지 궁금함)

### 로깅 BP
- Metric과 함께 모니터링 해야함
- 로그는 콜드 스토리지를 잘 활용해야 함 (TTL)
- Sidecar pattern으로 필수적인 것만 전달 해야 함(Damonset)


# CH4
## 새롭게 알게 된 것
### Secret
- 시크릿: base64로 인코딩 되어 저장되므로 별도 암호화 필요
- 종류
    - generic: key-value pair
    - docker-registry: 레지스트리 보안
    - tls: pub/private key 로 시크릿을 생성 -> ingress에서 https 인증에 활용
- 특징
    - 사용하게 되는 pod의 tmpfs에 마운트되어 pod사라질때 같이 사라짐
    - etcd 의 데이터 스토리지에 평문으로 저장 -> etcd 보완도 잘 챙겨야 함
    - 서드파티 KMS 사용 권장

### RBAC & Service Account
> 도대체 Service Account가 뭔데
- RBAC
    - Role/ClusterRole: 권한 정의
    - RoleBinding/ClusterRoleBinding: SA에 권한 연결
- SA
    - 구분
        - User Account: 실제 사람이 kubectl을 통해 사용
        - Service Account: Pod 내부 프로세스가 API 서버와 통신할 때 사용
    - 3가지 컨트롤러
        - ServiceAccount Admission Controller: Pod에 기본 SA 할당
        - Token Controller: JWT 토큰 생성 및 관리
        - ServiceAccount Controller: SA와 Secret 생성/삭제 관리