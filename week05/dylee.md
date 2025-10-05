# 9.네트워킹, 네트워크 보안, 서비스 메시

## 1. 쿠버네티스 네트워킹 원리

- 동일한 파드 내부의 컨테이너 간 통신
    - linux namespace docker networking덕분에 컨테이너를 모두 동일한 네트워크에 둘 수 있음 → 동일한 pod에 속한 컨테이너끼리는 localhost통신이 가능하고, 리스닝하는 port는 서로 달라야 함
- 파드 간 통신
    - 모든 pod는 NAT(네트워크 주소 변환)없이도 서로 통신이 가능해야 함  → pod끼리는 pod의 실제 ip 주소를 가지고 통신함
    - 따라서 같은 worker node에 있는 pod간의 통신과 동일한 cluster에 있지만 다른 worker node에 있는 pod간의 통신 모두 NAT없이 통신이 이루어져야 함
    - 실제 ip 주소를 가지고 통신하므로 host 기반 agent나 system demon은 항상 pod와 통신 가능
- 서비스와 파드 간 통신
    - 서비스: 모든 트래픽을 서비스에 매핑된 엔드포인트로 포워딩하는 각 노드마다 할당된 고정된 ip 주소 및 포트
    - iptables와 IPVS 사용하여 service가 요청을 pod에게 전달
        - worker node에서 실행되는 kube-proxy가 service와 pod의 변경 사항을 감지하여 iptables update
        - 트래픽이 clusterIP로 들어옴 → node의 iptables 규칙이 이 트래픽을 가로챔 → iptables는 이 IP를 pod의 실제 IP 중 하나로 변경하고 트래픽을 그곳으로 보냄 → 여러 pod에 걸쳐 traffic을 분산시키므로 L4로드밸런서 역할을 함
    - label selector를 통해 서비스를 파드에 묶으면 label이 동일한 서비스에서 파드로 트래픽이 전달됨
        - pod에 label 붙임 → service를 만들 때 selector.matchLabel을 이용하여 해당 label을 가진 pod들을 묶음 → control plane(EndpointSlice controller)는 이 selector을 감시하며 cluster내에서 해당 label을 가진 모든 pod를 찾아 이 pod들의 실제 ip주소와 port 목록(endpoint)의 목록을 만듬 → service는 이 목록을 보고 traffic을 어떤 pod로 보낼 지 결정
    - k8s에서 service registry가 필요 없는 이유
        - 내부 DNS 서버를 가지고 있어 Service 객체를 생성하면 해당 service에 대한 DNS레코드가 생성됨(`my-svc.my-namespace.svc.cluster-domain.example`)
        - k8s api server + etcd가 service registry역할 수행 → service와 pod 생성되면 label,ip등이 etcd에 저장 → dns 쿼리로 요청이 들어오면 dns 서버는 api 서버를 통해 해당 dns의 clusterIP 반환 → kube-proxy가 이 clusterIP로 보내는 요청을 가로채 iptables의 규칙에 따라 pod 중 하나로 해당 요청을 전달

## 2. 네트워크 플러그인

### 1. Kubenet

- 쿠버네티스 네트워크 플러그인
- 서로 다른 노드들간의 통신을 kubenet이 처리하는 방식
    - 같은 node에 있는 pod끼리 통신하는 방법 → cbr0 bridge로 통신
        - node의 eth0과 pod및 service의 interface 사이의 cbr0 bridge가 default gateway 역할 수행
        - 쿠버네티스는 각 노드의 bridge network 대역이 겹치지 않게 주소 대역 할당 → 그리고 게이트웨이 라우팅 테이블을 설정해서 어떤 패킷이 어떤 브릿지로 가야 하는지 설정
            - 이때 라우팅 규칙 + 가상 브릿지 조합 → 오버레이 네트워크
                - 오버레이 네트워크: 물리적인 인프라를 기반으로 네트워크 가상화 기술을 통해 end to end 통신을 구현하는 기술로 오버레이 네트워크가 언더레이 네트워크에 영향을 주지 않으며 분할된 네트워크에 자원을 할당하여 네트워크 자원 효율성을 높일 수 있고 네트워크 구성 시 암호화 알고리즘을 적용하여 높은 보안성을 지닐 수 있음 → calico
                - 언더레이 네트워크: 실제 네트워크 장비, 보안 장비, 서버등을 이용하여 구축한 물리적인 네트워크 인프라 → kubenet
            - ip network는 자신의 host에서 destination을 찾지 못하면 상위 gateway로 전달함
                
                ⇒  pod의 veth → cbr0 → eth0 순으로 올라가 node의 라우팅 테이블에 따라 다른 Node의 pod와 통신
                
    - Node간 통신
        - pod가 다른 Node의 pod와 통신할 때는 일단 패킷이 노드 밖으로 나가고 → 라우팅 규칙에 따라 올바른 노드로 전달됨(이때 pod ip는 바뀌지 않음)
    - 외부와의 통신
        - pod가 cluster 외부(인터넷)과 통신할 때는 ip 마스커레이딩을 통해 pod의 ip가 node의 공인 ip로 변경(NAT)됨

### 2. CNI 플러그인

- 컨테이너 간의 네트워킹을 제어하는 플러그인 표준 → 공통된 컨테이너 네트워크 인터페이스 제공
- 컨테이너 네트워크 인터페이스 구현체: calico, Flannel, Cilium..
- 멀티 호스트로 구성되어 있는 pod간의 통신을 위해 CNI를 사용함
- k8s kubelet 컴포넌트로 pod 생성, 삭제 명령 → CNI 표준 규칙에 맞춰 CNI 플러그인 호출 → 네트워킹은 CNI가 알아서 관리
- CNI 플러그인은 pod에 ip 주소 할당, 관리하는(컨테이너 삭제 → 생성했던 ip 회수) 역할을 함
- 다양한 CNI 플러그인들 사용 가능
- CNI 플러그인 마다 노드 네트워크 연결 방식 + ip 주소 할당 방식이 다르므로 용도에 따라 적절한 플러그인을 선택 하자.
- 퍼블릭 클라우드에서 클러스터를 실행할 경우, CNI 플러그인이 클라우드 프로바이더의 sdn에 네이티브 하지는 않은지 확인 하자.
- overlay network를 사용하지 않는다면 → 실제 물리적 리소스를 사용해 ip 주소를 할당하므로 충분한 네트워크 주소 공간을 미리 확보해 놓자

## 3. 쿠버네티스 서비스

- 네트워크 플러그인 때문에 pod는 동일한 clutser에 속한 pod하고만 직접 통신 가능
- 서비스 api로 k8s cluster 내부에서 잘 변하지 않는 ip, port 할당하고, 서비스 endpoint에 해당하는 pod에 매핑
- kube-proxy
    - iptables 또는 IPVS로 서비스와 pod 매핑
    - 클러스터 노드별로 한 개씩 실행되는 컨트롤러
    - 각 노드의 iptables 규칙 조정

### 1. ClusterIP 서비스 타입

- 미리 정의된 CIDR 범위의 IP 할당 → selector field로 pod에 ip, 포트, 프로토콜 매핑
- 서비스 선언 → DNS 부여 → 서비스 디스커버리 굿 + 클러스터 내 다른 서비스와 통신이 쉽다
- 접속 방식: 같은 ns → http://svc-name , 다른 ns → <svc.name>.<namespace.name>.svc.cluster.local
- kube-proxy가 관리 x → headless service

### 2. NodePort

- clsuter의 각 노드에 있는 고수준 포트(30,000~32,677)을 노드별 서비스 ip, port에 할당
- 자동 로드 밸런싱 구성을 제공하지 않는 온프레미스 클러스터나 맞춤형 솔루션에서 주로 활용

### 3. ExternalName 서비스 타입

- 클러스터의 dns name을 외부 dns 서비스에 전달
- 잘 사용 x

### 4. LoadBalancer 서비스 타입

- k8s cluster를 제공하는 인프라 프로바이더의 자체 로드밸런싱 매커니즘 배포하는 방법

### 5. 인그레스와 인그레스 컨트롤러

- HTTP 수준 라우터 → 호스트 기반 규칙 or 경로 기반 규칙에 따라 특정 백엔드 서비스로 트래픽을 줌
- 호스트 기반 라우팅으로 여러 호스트를 하나의 ingress에 둘 수 있음
- k8s secret에 인증서 정보를 담아서 → TLS termination 가능(443)
- 인그레스 컨트롤러
    - cluster에서 실행되고 수신 리소스에 따라 HTTP 로드 밸런서 구성
    - 쿠버네티스 인그레스 api 로직이 구현된 서드파티 컨트롤러
    - 구현체 → nginx, HAProxy 등등

### 6. 게이트웨이 api

- 인그레스 컨트롤러의 문제점
    - 정의 자체가 표현성이 떨어져 canary, 헤더 기반 라우팅, 트래픽 미러링, 고급 라우팅 정책들을 사용하려면 어노테이션을 사용해야 함 → controller 구현체마다 다르고 표준이 아님
    - 아키텍처 확장이 어려움 → 클러스터 전체에 대한 라우팅 규칙이 ingress 객체에 정의됨
    - 멀티테넌시가 쉽지 않음 → 하나의 클러스터를 여러 테넌트(팀)들이 나눠 쓰기 쉽지 않음(권한, 호스트 선점 문제 등)
- 게이트웨이 api → 선언형 구문으로 http 어플리케이션 표출
- 개발자는 주어진 제약조건 내에서 자신의 서비스를 어떻게 표출할지만 고민하면 됨

## 4. 네트워크 보안 정책

- NetworkPolicy api로 파드들 간 또는 다른 엔드포인트와 통신하는 방법 제어 → 애플리케이션 중심적으로 접근해서 쿠버네티스에 호스팅된 애플리케이션의 트래픽 분할, 통제
- ingress/egress → 출발지/목적지 규칙을 정의한 리스트, 인바운드 통신 규칙 / 아웃바운드 통신 규칙 정의
- PolicyTypes → 해당 정책 오브젝트가 어느 네트워크 정책 규칙 타입과 연결되는지 명시
- 처음에는 pod로 유입되는 트래픽에 집중하고 → 제어하기
- pod를 인터넷에서 access 한다면 label을 사용해서 ingress를 허용하는 네트워크 정책 명시

## 5. 서비스 메시

- 전용 데이터 플레인과 컨트롤 플레인으로 서비스를 서로 연결하고 보호 → 트래픽 로드 밸런싱, 서비스 디스커버리, 분산 시스템 추적, 트래픽 보안, 서킷 브레이커 등의 기능들을 사용 가능
- sidecar proxy 덕분에 컨트롤 플레인 컴포넌트는 정책 및 보안을 서비스 메시 전체에서 동기화 가능
- 대표적인 서비스 메시 → istio

## 1. 파드 시큐리티 어드미션 컨트롤러

- 클러스터 관리자가 간단하게 파드 보안 설정 가능
- 복잡도는 낮지만 namespace 수준에서만 permission 적용 가능

### 1. 파드 시큐리티 어드미션 활성화

- k8s 버전 1.22 이상이면 활성화 되어 있음

### 2. 파드 보안 수준

- privileged : 아무 제한 x
- baseline: 권한 상승 공격 등 각종 보안 취약점으로부터 보호
- restricted: 현재 파드 보안에 대한 모범 사례
- 보안 정책은 latest를 바로 적용하기 보다는 단계적으로 조금씩 업그레이드 권장

### 3. 네임스페이스 레이블을 이용한 파드 시큐리티 활성화

- namespace label에 yaml을 추가해서 namespace별로 파드 시큐리티 활성화 → aduit, warn 수준부터 시작해서 파악 후 enforce

## 2. 워크로드 격리와 런타임클래스

- 중첩 가상화 기반 Kata, gVisor, 파이어크래커 → 강력한 워크로드 격리 지원
- containerd → 웹 어셈블리 기반의 컨테이너 지원 → 유저는 같은 클러스터 내에서 다양한 런타임을 격리된 환경에서 골라서 사용 가능

### 1. 런타임클래스 사용하기

- 직접 pod spec에 runtimeClassName을 지정해 사용 가능 → 내부적으로는 런타임 클래스가 CRI에 전달되는 RuntimeHandler 선택하는 방식으로 구현되어 있음

### 2. 런타임 구현체

- CRI containerd: 단순성, 견고성, 이식성에 중점을 둔 컨테이너 런타임용 api 퍼사드
- cri-o: 경량 oci 기반 k8s 런타임 구현체
- 파이어크래커: kvm 기반의 가상화 기술로 기존 비가상화 환경에서도 가상 머신의 보안 및 격리 기능을 이용하여 마이크로 vm 빠르게 구축 가능
- gVisor: 새로운 유저 공간 커널로 컨테이너를 실행하는 OCI 호환 샌드박스 런타임으로 오버헤드가 적고 안전하고 격리된 컨테이너 런타임 제공
- 카타 컨테이너: 컨테이너처럼 작동되는 경량급 가상 머신 실행 → vm과 유사한 보안, 격리 기능 제공

### 3. 모범 사례

- 되도록이면 단일 런타임으로 구성된 클러스터를 각각 두자
- 격리된 워크로드를 구축해도 공격이 완전히 차단되진 않음
- 런타임마다 사용하는 툴에 일관성이 없다는 것을 주의하자
