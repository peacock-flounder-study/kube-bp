# CH9
## 새롭게 알게 된 것
### 컴포넌트 간 통신
- 동일한 파드 내 컨테이너 : 네트워크 공간 자체를 공유함 (localhost로 서로 접근 가능)
- pod간 : 아래의 것들을 달성함
    - 모든 Pod는 NAT 없이 다른 모든 Pod와 통신 가능
    - 모든 Node는 NAT 없이 모든 Pod와 통신 가능
    - Pod가 자신을 바라보는 IP는 다른 Pod가 바라보는 IP와 동일

### CNI 개요

- **목적**: 컨테이너 런타임과 네트워크 플러그인 간 표준 인터페이스
- **책임**:
    - 컨테이너 네트워크 연결
    - 컨테이너 삭제 시 리소스 정리

### CNI 동작 과정

1. 브리지 생성 (없으면)
2. veth 페어 생성
3. veth 한쪽을 컨테이너 네임스페이스로 이동
4. IP 주소 할당
5. 라우팅 테이블 설정
6. iptables 규칙 추가 (필요시)

## 4. Calico CNI

### 아키텍처 컴포넌트

### BIRD

- BGP 데몬
- 노드 간 라우팅 정보 교환
- 기본: Full mesh
- 대규모: Route Reflector 모드

### Felix

- calico-node 컨테이너 내 동작
- etcd에서 정보 읽기
- 라우팅 테이블 생성
- iptables/ipvs 규칙 관리

### Confd

- 설정 관리 도구
- BIRD 설정 변환

### Proxy ARP 메커니즘

- **동작 원리**:
    1. Pod가 게이트웨이로 ARP 요청
    2. Proxy ARP 활성화된 veth가 자신의 MAC 응답
    3. 패킷이 커널로 전달
    4. 라우팅 테이블에 따라 목적지로 전달

### service mesh
- 데이터 플레인
    - 각 서비스 옆에 배포되는 사이드카 프록시 (Envoy, Linkerd proxy)
    - 실제 네트워크 트래픽 처리
    - 서비스 간 모든 인바운드/아웃바운드 통신 중계
- 컨트롤 플레인
    - 프록시 설정 및 정책 관리
    - 서비스 디스커버리 제공
    - 메트릭 수집 및 집계
    - 인증서 관리 및 배포

# CH10
## 새롭게 알게 된 것
### PodSecurityPolicy와 런타임 클래스를 이용한 보안 설정
- kubectl get psp 명령으로 정책 확인
- api-server flag --enable-admission-plugins 설정 필요

- 다양한 예
    - pause-deployment 생성 및 보안 정책 적용
    - privileged 정책 생성 (특권 모드 허용)
    - restricted 정책 생성 (제한적 보안 설정)
    - ClusterRole과 ClusterRoleBinding을 통한 RBAC 권한 설정
    - ServiceAccount 생성 및 RoleBinding 연결

### 워크로드 격리
- CRI containerd: 범용 컨테이너 런타임
- cri-o: OCI 기반 쿠버네티스 전용 런타임
- Firecracker: 가상화 기술 적용 microVM
- gVisor: OCI 호환 샌드박스 런타임
- Kata Containers: VM 격리 기능 제공

> runtimeClassName: firecracker 설정으로 특정 런타임 지정 가능

### admission controller
- SecurityContext 워크로드 설정
- DenyExecOnPrivileged, DenyEscalatingExec 같은 컨트롤러로 보안 강화