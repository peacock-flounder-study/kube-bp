# 1. 기본 서비스 설치


### 1. 구성 파일 관리

- 쿠버네티스는 클러스터의 원하는 상태를 yaml, json file 형태로 선언 → 모든 애플리케이션은 이에 따라 정의됨
- 선언형 방식은 애플리케이션에서 발생한 문제를 이해하거나 재현하기 쉬움
- yaml file에 선언한 상태 → source of truth → 이 상태를 정확하게 관리해야 함
    - 애플리케이션 상태 수정 시 변경사항 관리, 검사, 누가 변경했는지 감사, 실패했을 시 롤백이 가능해야 함
- 애플리케이션을 file system에 배치할 때 고유한 디렉터리 체계에 맞게 컴포넌트를 구성하는 것이 좋음
    - 보통 application service마다 하나의 디렉터리 사용
    - 이 디렉터리 내부에 서브컴포넌트별로 서브 디렉토리를 만들어서 씀

### 2. 디플로이먼트를 이용한 복제 서비스 생성

1. **이미지 관리 모범 사례**
    - 이미지 빌드 모범 사례
        - 이미지를 빌드할 때는 잘 알려진 공급자가 제공한 이미지를 베이스로 삼아햐 함
    - 이미지 네이밍 모범 사례
        - image registry에 있는 tag(컨테이너 이미지의 버전) → 불변으로 취급
        - image가 build된 commit의 시맨틱 버전 + SHA 해시를 조합해서 네이밍 하는것이 좋음
            - ex) image:v1 0.1-bfeda01f
        - 이미지 버전 latest는 지양하자
2. **애플리케이션 레플리카 생성**
    - replicaSet: 컨테이너화한 특정 버전의 애플리케이션을 직접 복제하기 위해 만든 쿠버네티스 리소스
    - 2개 이상의 replicas를 운영하는 것이 좋음
        - crash 대비, 애플리케이션 새 버전 rollout을 위함
    - replicaSet보다 deployment를 사용하는 것을 권장
        - pod와 replicaset에 대한 선언적 업데이트 제공
        - replicaSet의 복제, 버저닝, 단계적 롤아웃 지원
    - deployment, deployment가 생성하는 replicaset, pods → labels로 구분
        - 레이블을 통해 한 번의 요청으로 특정 레이어에 있는 모든 리소스를 가져올 수 있음
    - request
        - 애플리케이션을 실행하는 호스트 머신에 예약한 리소스
    - limits
        - 컨테이너가 사용할 수 있는 최대 리소스
        - 모니터링을 통해 조정
    - GitOps로 CI/CD를 자동화하여 특정 브랜치만 프로덕션에 배포 →소스 코드와 프로덕션 일치

### 3. HTTP 트래픽을 처리하는 외부 인그레스 설치

- 애플리케이션 컨테이너로 외부 트래픽이 들어오게 하기 위해 service, load balancer필요    
    - http경로와 호스트 기반의 요청을 라우팅 하도록 http로드 밸런싱 수행
    - 트래픽 진입점에 ingress를 두면 서비스 확장 시 유연성을 확보할 수 있음
    - ingress를 정의하려면 cluster에서 실행 중인 ingress container controller 필요 → Nginx, HAproxy많이 사용함
    - Nginx vs HAproxy
        
        
        | 차이점 | HAProxy | NGINX |
        | --- | --- | --- |
        | 핵심 기능 | 로드밸런서, 고성능, 리버스 프록시, 속도가 빠름 | 고성능 HTTP 서버, 리버스 프록시 및 간편한 구성 |
        | 구현 | HAProxy 구현 시 프로세스는 구성 매개변수에 영향을 미치는 메모리를 공유하지 않고, 세션 지속성이 없다. | NGINX 구현 시 프로세스는 메모리를 공유한다. |
        | 프로토콜 지원 | gRPC, HTTP, HTTP/2에 대한 프록시 프로토콜 지원 | HTTP, HTTPS 및 이메일 관련 프로토콜 지원 |

### 4. 컨피그맵으로 애플리케이션 구성

- k8s에서는 ConfigMap에 구성을 정의
- ConfigMap에는 key:value로 값이 담겨 있음 → 파일 또는 환경 변수 형태로 파드 내부의 컨테이너에 구성 정보를 전달할 수 있음
- ConfigMap의 name에 버전 번호를 넣어서 관리 → 변경을 해야 할 경우 새 버전 이름(v2)을 가진 ConfigMap을 새로 만든 뒤 deployment resource update → deployment rollout이 자동으로 트리거
- 롤백 시 v1 ConfigMap을 사용하면 됨

### 5. 시크릿 인증 관리

- password를 애플리케이션 소스 코드나 내부 파일에 둔다면?
    - password 노출 문제
    - 액세스 제어 문제
    - 파라미터화 → 동일한 소스 코드를 다양한 환경에서 사용할 수 없음
- secret: k8s가 시크릿 데이터를 관리하는 리소스
    - secret key 관리는 클라우드 프로바이더의 서비스 같은 오픈소스 구현체를 활용하는 것이 좋음
- k8s Volume: 실행 중인 컨테이너에 마운트 가능한 파일이나 디렉터리를 유저가 지정한 것
    - secret volume: tmpfs 램 기반의 파일 시스템으로 생성한 후 컨테이너에 마운트 → 물리 장비가 손상되어도 공격자가 secret를 취득하기 어려움
- CSI 드라이버: 쿠버네티스 클러스터 외부에 위치한 KMS를 사용할 수 있음
    - CSI드라이버를 활용한 volume
        - azure keyvalut로 secret에 access하기
        - SecretProviderClass → 어떤 외부 secret를 가져올지 선언한 yaml 파일
        
        ```yaml
        # This is a SecretProviderClass example using service principal to access Keyvault
        apiVersion: secrets-store.csi.x-k8s.io/v1
        kind: SecretProviderClass
        metadata:
          name: akvprovider-demo
          namespace: demo
        spec:
          provider: azure
          parameters:
            usePodIdentity: "false"
            keyvaultName: <key-vault-name>
            cloudName:                           # Defaults to AzurePublicCloud
            objects:  |
              array:
                - |
                  objectName: DemoSecret
                  objectType: secret             # object types: secret, key or cert
                  objectVersion: ""              # [OPTIONAL] object versions, default to latest if empty
            tenantId: <tenant-Id>                # The tenant ID of the Azure Key Vault instance
        ```
        
        - deployment.yaml → secret를 지정한 deployment
        
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: busybox-secrets-store
          namespace: demo
          labels:
            app: busybox-secrets-store
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: busybox-secrets-store
          template:
            metadata:
              labels:
                app: busybox-secrets-store
            spec:
              containers:
              - name: busybox
                image: k8s.gcr.io/e2e-test-images/busybox:1.29
                command:
                - "/bin/sleep"
                - "10000"
                #정의한 volume을 volumeMounts field를 통해 특정 컨테이너에 마운트
                volumeMounts:
                - name: secrets-store-inline
                  mountPath: "/mnt/secrets-store"
                  readOnly: true
              #pods에 volumes를 추가(볼륨 엔트리) -> data source 정의
              volumes:
              - name: secrets-store-inline
                csi:
                  driver: secrets-store.csi.k8s.io
                  readOnly: true
                  volumeAttributes:
                    secretProviderClass: "akvprovider-demo"
                  nodePublishSecretRef:
                    name: secrets-store-creds
        ```

### 6. 간단한 스테이트풀 데이터베이스 배포 → 42

- stateful application → 상태를 고려해야 함
- k8s에서 stateful workload를 실행할 때 PersistentVolume을 사용하여 application의 상태를 관리해야 컨테이너가 이관되거나 재시작될 때 데이터가 소실되지 않음
- PersistentVolume
    - pod와 연관되어 특정 위치의 컨테이너에 마운트되는 볼륨
    - block storage 방식으로 마운트되는 원격 스토리지(NFS,SMB, iSCSI..)
        - block storage
            - 클라우드 기반 스토리지 환경에 데이터 파일을 저장하는데 사용되는 기술
            - 데이터를 블록으로 분할 → 해당 블록을 각각 고유 ID를 지닌 개별 조각으로 저장
            - block storage system의 데이터 요청 → 데이터 블록을 재구성한 후 데이터 제시
            - 데이터가 다수의 환경에 분산 → 데이터에 대한 다수의 경로가 만들어짐 → 사용자는 이를 통해 데이터를 더 빠르게 검색할 수 있음
    - 블록 스토리지 → 성능이 중요한 애플리케이션에서 더 좋음(데이터베이스)
    - 파일 기반 디스크 → 구조가 블록 스토리지보다 더 유연해, 성능이 중요하지 않다면 권장
- StatefulSet
    - pod의 순서와 고유성에 대해 보장
    - 각 pod에 영구적인 식별자(고정 id) 유지 → pv를 사용할 경우 pod가 제거되고 새로운 pod가 생성되더라도 동일한 pv에 마운트 할 수 있도록 함
    - 일관된 네이밍, 스케일 업, 스케일 다운의 순서 지정 등의 강점이 있음
    - replicaset을 배포할 때 편리 → pod가 증가하면 생성되는 pod마다 각자의 pvc를 가지게 됨
- PersistentVolumeClame
    - claim → 리소스 요청
    - k8s cluster가 알아서 적당한 persistent volume을 프로비저닝
    - specification이 서로 다른 클라우드, 온프레미스 간 이식 가능한 statsfulSet을 쓸 수 있음
    - pod마다 고유한 pv 할당 가능(pv type은 어느 한 파드에만 마운트 가능)
- 
    - pv: pod 컨테이너에 마운트되는 볼륨 → 이를 요청하기 위해 pvc 작성
    - pod가 제거되어도 데이터를 유지하기 위해 statefulset사용 → statefulset은 volume claim template를 통해 각 pod별로 pvc 자동 생성
        

### 7. 서비스를 응용한 TCP 로드밸런서 구축

- master-slave replication을 설정 → 어디로 접근해도 같은 데이터를 읽을 수 있음 → 쓰기 전용 클라이언트에서 쓰기 svc에 접근

### 8. ingress를 이용해 트래픽을 스태틱 파일 서버로 전달

- deployment를 이용해 static image를 서비스 하는 nginx container을 만들어 replica에 배포
- ingress → svc → static web server(deployment로 만들어진 컨테이너)로 요청이 전달됨

### 9. 헬름을 이용한 애플리케이션 파라미터화

- template system으로 다양한 환경에서 편리하게 배포, 실제 유제에게 영향을 미치지 않고 개발 iteration 진행 가능
- k8s는 helm으로 템플릿 시스템 제공
- helm chart → 여러 파일로 애플리케이션 패키징 → 애플리케이션에 필요한 값을 파라미터로 전달해서 쿠버네티스에 배포        

### 10. 서비스 배포 모범 사례

- 대부분의 서비스는 deployment resource로 배포하는 것이 좋음
- svc
    - 로드 밸런서 역할
    - svc는 기본적으로 클러스터 내부에 표출되며, 외부에도 표출 할 수 있음
- http 애플리케이션을 표출하려면 ingress controller를 사용해서 요청 라우팅, ssl등의 기능을 추가하기
- helm같은 패키징 툴로 parameter화를 해서 다양한 환경에서 애플리케이션 구성 재사용 하기

# 2. 개발자 워크플로

## 개발 클러스터 구축

- 단일 개발 클러스터
    - 효율적, 모니터링 로깅 쉽게 설치 가능
    - 개발자 간의 간섭, 유저 관리 프로세스 복잡 →  RBAC를 잘 활용하면 간섭을 줄일 수 있음
- 개발자마다 클러스터를 따로 부여
    - 단순함, 다른 개발자의 클러스터에 침범해 영향을 끼칠 일이 없음
    - 비효율적, 관리할 클러스터가 많아짐
- 플릿 매니지먼트 같은 툴을 사용해서 여러 클러스터를 쉽게 관리하자!

## 여러 개발자가 사용할 공용 클러스터 구축

- 유저를 k8s cluster에 온보딩 하는 방법
    1. 유저에게 새 인증서를 발급하고 로그인 시 사용할 kubeconfig file제공
    2. 외부 identity 시스템 활용 → ms entra id, aws iam..
- k8s 인증서 api 사용 방법
    1. CSR생성 → client-key.pem과 client-csr생성
    2. CertificateSigningRequest 리소스 생성 및 승인
        - k8s api로 CSR 제출 → 관리자가 CSR 승인 → 인증서 발급
    3. 인증서 추출
    4. client에게 k8s RBAC 적용 → namespace, k8s cluster에 대한 권한 부여
    5. client가 사용할 namespace에 RoleBinding object를 만들어 client의 role binding
        - namespace에 ResourceQuota를 설정하면 namespace의 resource사용량을 제한 할 수 있음
    6. kubeconfig 설정
        - 위에서 발급받은 인증서를 통해 kubelet을 설정 → 사용자가 실제로 api에 접근 가능하게 됨
- laas 시스템으로 로깅 → 여러 컨테이너에 흩어져 있는 로그들을 모으자

## 개발자 워크플로 활성화

- application의 dependency를 기술, 설치하는 스크립트로 작성해서 모든 환경에서도 작업할 수 있도록 하자
- 자동화 툴을 ci/cd 파이프라인에 통합해서 사용하자

## 개발 환경 설정 모범 사례

- 개발자 경험을 온보딩, 개발, 테스팅의 3단계로 생각하기
- 개발 클러스터는 공용 클러스터가 더 나음
- 클러스터에 유저를 추가할 때 유저 identity를 추가하고 각자 네임스페이스로만 액세스 할 수 있도록 설정하기
- namespace관리 시 오래된 미사용 리소스를 어떻게 회수할지 고민하기 + 리소스 작업 자동화 하기
- 로그, 모니터링 검토 → cluster 수준의 dependency를 helm chart같은 template을 구축하는것도 고려해보기