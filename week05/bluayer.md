# Chapter 09. 네트워킹, 네트워크 보안, 서비스 예시

## 쿠버네티스 네트워킹의 원리

파드 네트워킹을 호스트하는 것 외에 아무 일도 안하는 모든 파드에 paused container를 사용해서 컨테이너를 모두 동일한 로컬 네트워크에 두는 리눅스 네임스페이스와 도커 네트워킹 덕분에 컨테이너 통신이 가능하다!

## 네트워크 플러그인

### Kubenet

바로 사용 가능한 가장 기본적이고 단순한 네트워크 플러그인

파드를 연결할 가상 이더넷 쌍인 리눅스 브릿지 cbr0를 제공.

### CNI 플로그인

스펙에 기본 요건이 명시. - IP 주소 관리 + 네트워크에 컨테이너 추가/삭제 할 정도의 기능만 최소한으로.

### CNI 모범 사례

- SDN 네트워크 공간과 따로 분리된 오버레이 네트워크를 제공하지 않는 CNI를 사용하는 경우, 다양한 용도로 사용 가능할만큼의 네트워크 주소 공간을 확보하자.

## 쿠버네티스 서비스

파드가 제공하는 서비스에 바로 액세스하는 것은 효율적이지 않다.

서비스 API를 통해 쉽게 변하지 않는 IP와 포트를 할당하고 매핑하자.

- Cluster IP, NodePort, ExternalName, LoadBalancer

### Ingress and Ingress Controller

Ingress API는 HTTP 수준의 라우터.

TLS 및 디폴트 백엔드의 세부 구성은 Ingress Controller가 처리. Ingress API와 분리되어 있으며, 취향에 맞게 Nginx, Traefik, HAProxy 등의 컨트롤러를 활용.

### Gateway API

Ingress API의 문제점
- 정의 자체가 표현성이 떨어짐
- 아키텍처 확장성이 거의 없음
- 벤더 종속적 애너테이션으로 인한 이식성 감소

Ingress API의 대체가 아니라 단점을 보완하는 역할.

**관련해서 읽어볼 것**
- https://www.cncf.io/blog/2025/05/02/understanding-kubernetes-gateway-api-a-modern-approach-to-traffic-management/
- https://gateway-api.sigs.k8s.io/geps/gep-1762/
- https://coffeewhale.com/packet-network1

### 서비스와 인그레스 컨트롤러 BP

- 클러스터 외부에서 액세스하는 서비스를 줄이자
- 가능한 한 사내 표준의 인그레스 컨트롤러를 사용하자
- 인그레스 컨트롤러를 파드 기반의 워크로드로 배포하는 경우, 고가용성, 성능 요건이 충분히 반영됐는지 확인하자

## 네트워크 보안 정책

Netpol을 통해 액세스 제어 가능.

East-west traffic firewall, ns level, podSelector required.

### BP
- 천천히 시작하고 복잡해지면 끔찍해질 수 있으니 민감한 워크로드로 흘러가는 흐름을 통제하자.
- default-deny라면, 실수로 파드가 표출되지 않게 하자.
- 규칙이 네임스페이스 단위로 구분되기 때문에 가능하면 애플리케이션 워크로드를 단일 네임스페이스 두자.

### 서비스 메시

사이드카 프록시를 활용. SMI 같은 걸 수립하려고 하고 있음.

- 메시 전체에 트래픽 조절 정책
- 서비스 디스커버리
- 추적 시스템을 통해 전체 분산 시스테 ㅁ추적
- 상호 인증을 통해 보안
- 서킷 브레이커, retry, deadline 등의 패턴

서비스 메시를 적용하는 게 항상 옳을까?

**읽어볼 것** : xDS, https://www.anyflow.net/sw-engineer/istio-internals-by-xds

# Chapter 10. 파드와 컨테이너 보안

## Pod Security Admission Controller

PodSecurityPolicy는 너무 복잡하고 어려워서 PodSecurityAdmission을 사용하게 됨. 물론, 네임스페이스 단위기 때문에 나중에는 게이트키퍼 프로젝트 같은 솔루션을 찾게 됨.

### 파드 보안 수준

- privileged : 아무 제한이 없음 
- baseline : 권한 상승 공격 등 각종 보안 취약점으로부터 보호
- restricted : 파드 보안에 대한 모범 사례

resetricted부터 적용하고 싶겠지만, 실제로는 충돌 때문에 제대로 작동하지 않을 수도 있음

보안 정책은 클러스터를 업그레이드한 이후 단계적으로 조금씩 업그레이드하는 것이 모범 사례다.

정책 활성화 수준
- enforce
- warn
- audit

**개인적인 의견** : 실제로는 Privileged는 거의 안 쓰는 방향으로 가고 있으며, root 권한의 경우에도 rootless로 가고 있는 중. 다만, GPU 관련 워크로드에서는 완벽한 rootless가 어려운 경우도 있는 것으로 보인다.

### 네임스페이스 레이블을 이용한 파드 시큐리티 활성화

보안 태세(security posture)은 여러 팀의 기능과 워크로드에 따라 결정되므로, 한 가지 절대적인 모범 사례 도출이 어렵다. 

## 워크로드 격리와 런타임 클래스

컨테이너 런타임은 안전하지 못한 워크로드 격리의 경계라고 바라보는 시각이 지배적.

Kata, gVisor, Firecracker는 nested virtualization 또는 시스템 호출 필터링을 통해 강력한 워크로드 격리를 함.

런타임클래스를 통해 컨테이너 런타임을 선택할 수 있게 됐음.

### BP

- 여러 워크로드를 런타임 클래스를 통해 따로따로 분리하면 운영 환경이 복잡해진다. 서로 다른 컨테이너 런타임 간에는 워크로드 이식이 불가능. 단일 런타임으로 구성된 클러스터를 각각 따로 두는 것이 좋음
- 워크로드 격리 != 멀티테넌시 보안. 쿠버네티스를 전체적으로 처음부터 끝까지 잘 생각해보기
- 런타임이 다 다르다는 것은, 트러블슈팅 과정에서 혼란과 복잡성이 생길 가능성 높음

## 파드와 컨테이너 보안 관련 고려 사항

- Admission controller
- 침입 및 이상 탐지 툴 : 대부분 리눅스 시스템 콜 리스팅/필터링하거나 eBPF 활용. e.g. Falco






