# CH11
## 새롭게 알게 된 것
### 정책과 거버넌스
- 개발자의 신속성과 자율성을 저해하면 안됨
- 개방형 정책 에이전트(OPA)를 통해 정책 엔진이 잘 지키고 있는지를 판단함
    - Gatekeeper: OPA를 통해 구현되고 있는 오픈소스 프로젝트

### GATEKEEPER
- OPA를 활용해 CRD기반 정책을 구성
- 정책 템플릿 == 제약 템플릿 : 여러 클러스터에서 공유 및 재사용 가능
- Examples
    - 서비스가 인터넷에 공개적으로 노출되면 안됨
    - 신뢰할 수 있는 레지스트리만 사용
    - 모든 컨테이너에는 리소스 제한이 있어야 함
    - ...
- components
    - ConstraintTemplate: 재사용 가능한 정책 템플릿 정의
    - Constraint: ConstraintTemplate의 인스턴스화, 실제 적용 규칙
    - Config: Gatekeeper 동작 설정 (네임스페이스 제외, 동기화 등)
    - Mutation: 리소스 자동 수정 기능 (v3.3+)
- Gatekeeper v3가 현재 주류 (이전 v1/v2 대비 대폭 개선)


- 컨테이너 레지스트리를 제한하는 예시
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not strings.any_prefix_match(container.image, input.parameters.repos)
          msg := sprintf("container <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }
```

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: must-have-allowed-repos
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
    namespaces: ["production", "staging"]
  parameters:
    repos:
    - "gcr.io/my-company/"
    - "docker.io/mycompany/"
    - "ecr.amazonaws.com/mycompany/"
```
- Prometheus 메트릭 제공 (위반 횟수, 평가 시간 등)
    - 구조화된 로그로 감사 추적 가능

# CH12
## 새롭게 알게 된 것
### 멀티 클러스터의 고려사항
- 폭발 반경: 한 클러스터의 장애가 전체 시스템에 영향을 주지 않도록 격리
- 규정: 클러스터별 독립적인 로깅 및 모니터링 체계 (국가/지역 특성)
- 보안: 클러스터 간 트래픽 제어 및 방화벽 설정
- 멀티테넌시
- 리전별 워크로드
- 특수한 워크로드

### 다중 클러스터의 설계 고려사항
- 데이터 복제
- 서비스 디스커버리
- 네트워크 라우팅
- 운영관리
- 지속적 배포

-> 기술 스택의 예시
- 서비스 디스커버리: Consul, Linkerd의 Service Mirroring
- 네트워크: Spanner, CosmosDB 등 글로벌 데이터베이스
- 레지스트리: Harbor 등으로 멀티테넌시 구현

### GitOps 원칙
- 선언적 인프라 관리
- Git을 single source of truth로 활용
- Pull 기반 배포 (Push 대신)

### Flux 워크플로우
- Git 저장소 클론
- 매니페스트 변경 감지
- 클러스터 상태와 비교
- 자동 동기화 및 배포


### 다중 클러스터 관리도구
- Rancher: 중앙집중식 UI 제공, 온프레미스/클라우드/호스팅 클러스터 모니터링
- 퀸(Queen): Kubernetes 클러스터 프로비저닝, Mirantis에서 개발
- Gardener: Kubernetes Primitives를 사용한 최종 사용자 중심 클러스터 형태 제공



# CH13
## 새롭게 알게 된 것
### 쿠버네티스로 서비스 가져오기


- 셀렉터가 없는 서비스로 안정적인 IP 주소 사용
  - 외부 리소스에 대한 안정적인 IP 주소 제공
  - 수동으로 엔드포인트 리소스 생성 필요
- 안정적인 DNS 이름을 위한 CNAME 기반 서비스
- 액티브 컨트롤러 방식
  - 외부 서비스에 대한 안정적인 DNS 주소와 IP 주소 제공

### 쿠버네티스 서비스 내보내기
- 내부 로드 밸런서를 사용해 서비스 내보내기
- 클라우드 환경에서 내부 로드 밸런서 설정
  Azure 예시:`service.beta.kubernetes.io/azure-load-balancer-internal: "true"`
  AWS 예시: `service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0`
  - 내부 IP 주소로 내부 로드 밸런서 생성
  - 클라우드별 특정 어노테이션 사용

- NodePort 서비스 구성
  - 각 노드의 특정 포트(30000-32767) 범위에서 서비스 노출
  - 외부 로드 밸런서와 통합 가능
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-node-port-service
spec:
  type: NodePort
  ports:
    - nodePort: 30919
```

### 외부 서비스와 쿠버네티스 통합
- kubelet: 각 노드에서 실행되어 DNS 기반 서비스 검색 제공
- kube-proxy: 노드별 프록시로 iptables 규칙 설정
- CoreDNS/kube-dns: 클러스터 내부 DNS 서비스 제공

