## 구성 파일 관리

선언을 통해 구성 관리 -> 명령형 방식은 애플리케이션에서 발생한 문제를 이해하거나 재현하기 쉽지 않음.

쿠버네티스에서는 JSON, YAML 둘 다 사용

쿠버네티스 구성을 git으로 관리하며, 버전 관리에 드는 노력만큼 코드 리뷰에도 정성을 쏟아야 한다. (상태를 정확하게 관리하기 위해서)

저널 애플리케이션의 파일 레이아웃 예제 :

```
journal/
  frontend/
  redis/
  fileserver/
```

## 디플로이먼트를 이용한 복제 서비스 생성

### 이미지 관리 모범 사례

이미지 빌드 프로세스는 공급망 공격(신뢰할 수 있는 소스의 디펜던시에 해커가 악의적인 코드나 바이너리를 삽입하여 애플리케이션에 집어넣는 행휘)에 취약하다.

이미지 네이밍 모범 사례 : immutable tags, 이미지가 빌드된 커밋의 semantic version과 SHA hash를 조합하기 (v1.0.1-bfeda01f)

### 애플리케이션 레플리카 생성

ReplicaSet : containerized 특정 애플리케이션을 직접 복제하기 위해 만든 쿠버네티스 리소스. 하지만, 이걸 직접 쓰기보다 Deployment를 대신 사용하는 게 좋음
**Deployment** : ReplicaSet의 복제, versioning, staged rollout을 지원. 애플리케이션의 버전 변경을 매끄럽게 도움

레이블을 활용한 구별(app:front-end)

YAML 파일 곳곳에 있는 주석 👍

- requests(요청) : 애플리케이션을 실행하는 호스트 머신에 예약한 리소스
- limits(리밋) : 컨테이너가 사용할 수 있는 최대 리소스

예전에 읽었던 글 : https://itchain.wordpress.com/2018/05/16/kubernetes-resource-request-limit/

클러스터의 콘텐츠와 실제로 소스 코드에 정의한 콘텐츠가 서로 정확히 일치하는 그림이 가장 이상적 -> **GitOps**를 이용해 Ci/CD 시스템을 자동화하기

***참고 (MacOS)***
colima 설치  > minikube 설치 후 실습 진행

## HTTP 트래픽을 처리하는 외부 인그레스 설치

애플리케이션을 외부에 표출하여 트래픽이 들어오게 하려면 외부 IP 주소를 제공하는 서비스와 LB가 필요하다.

1. TCP/UDP 트래픽을 로드 밸런싱 하는 **Service**
2. HTTP 경로와 호스트 기반의 요청을 지능적으로 라우팅하도록 LB하는 **Ingress** : 인그레스 컨트롤러 컨테이너가 필요. CSP 혹은 OSS 중 구현체 선택.

***마이크로서비스에 HTTP로 액세스하는 방식을 두고 논란이 많았고, k8s용 Gateway API가 개발되었다. Gateway API(GatewayClass)는 k8s 익스텐션이므로 사용하려면 클러스터에 추가 컴포넌트를 설치해야 한다.***



