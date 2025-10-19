# 14장

> 분산 학습 (Distributed Training)
> 
- 분산 훈련은 최적화하기 어려우며 아직 초기 단계에 있음
- **8개의 GPU가 필요한 경우, 4개 GPU가 탑재된 두 대의 머신보다 하나의 8-GPU 머신에서 훈련하는 것이 거의 항상 더 빠름**
- 분산 학습은 모델이 사용 가능한 가장 큰 머신에 맞지 않을 때만 사용해야함
- 분산 TensorFlow 아키텍처에서는 파라미터 서버에서 각 노드로의 변수 배포와 각 노드에서 파라미터 서버로 다시 그라데이션을 적용하는 단계에서 많은 네트워크 트래픽이 발생

> D. 특수 하드웨어 및 스케줄링
> 

**특수 하드웨어의 필요성 및 Kubernetes의 확장성**

- **특수 하드웨어의 역할:**
    - 머신 러닝 모델의 학습(Training) 및 서빙(Serving)은 대부분 **특수 하드웨어**에서 더 효율적
    - 이러한 특수 하드웨어의 가장 대표적인 예로는 상용 GPU (그래픽 처리 장치)
    - 일반적으로 더 많은 리소스를 투입할수록 학습이 더 빨리 완료됨
- **Kubernetes의 확장성:**
    - Kubernetes - 소스 코드를 변경하지 않고도 새로운 유형의 하드웨어를 스케줄러에 빠르고 쉽게 노출할 수 있는 확장 가능(Extensibility)한 플랫폼

**디바이스 플러그인(Device Plugins) 메커니즘**

- Kubernetes는 **디바이스 플러그인 프레임워크**를 통해 GPU와 같은 특수 하드웨어 리소스에 액세스하고 이를 스케줄러에 알림
- **공급업체 지원:**
    - 이 프레임워크는 공급업체가 특정 디바이스를 구현하기 위해 핵심 Kubernetes 코드를 수정할 필요가 없도록 기능을 용이하게 함
- **작동 방식:**
    - 이러한 디바이스 플러그인은 일반적으로 각 노드에서 데몬셋(DaemonSet)으로 실행.
    - 데몬셋은 해당 특정 리소스를 검사한 후, 이를 Kubernetes API에 스케줄링 가능한 리소스로 알리는 프로세스를 담당
    - 예
        - NVIDIA GPU에 액세스하기 위한 Kubernetes용 NVIDIA 디바이스 플러그인이 실행되면, 파드를 생성할 때 해당 리소스를 요청할 수 있으며, Kubernetes는 해당 리소스를 사용할 수 있는 노드에 파드가 스케줄
        
        ```yaml
        # 디바이스 플러그인을 통한 GPU 리소스 요청 예시
        spec:
          containers:
            # ...
            resources:
              limits:
               nvidia.com/gpu: 2 # GPU 2개 요청
        ```
        
- **적용 범위:**
    - 디바이스 플러그인은 GPU에만 국한되지 X.
    - **Field Programmable Gate Array (FPGA)** 또는 InfiniBand와 같이 특수 하드웨어가 필요한 모든 곳에서 사용할 수 있음

> 스케줄링의 특이 사항 및 제한 사항
> 

Kubernetes는 자신이 알지 못하는 리소스에 대해서는 결정을 내릴 수 없음. 

이로 인해 특수 하드웨어의 스케줄링 및 활용도에 몇 가지 특이 사항이 발생

- **제한된 정보 노출:**
    - 현재 디바이스 플러그인은 일반적으로 **GPU 코어 수만 노출**하며, 다음 정보는 생략하는 경우가 많음
        - 코어당 실행할 수 있는 **스레드 수**.
        - GPU 코어가 어느 버스(Bus)에 있는지.
- **활용도 병목 현상:**
    - 위와 같은 정보 부족으로 인해, 서로 메모리에 액세스하거나 통신해야 하는 작업이 동일한 Kubernetes 노드에 함께 배치되지 못할 수 있음.
    - 이 때문에 구매한 강력한 GPU를 100% 활용하지 못하고, 훈련 시 GPU가 최대 용량으로 실행되지 않는 경우가 발생할 수 있음
- **분수 요청 불가:**
    - 현재로서는 GPU의 일부분(예: 0.1개)만 요청할 수 없음.
    - 특정 GPU가 여러 스레드의 동시 실행을 지원하더라도, 이 용량을 활용할 수 없다는 의미입니다.

> 네트워킹 및 전문 프로토콜
> 
- **네트워킹 영향:**
    - 훈련 단계(특히 분산 훈련)는 네트워크에 큰 영향을 미침.
    - 이 교환에 걸리는 시간은 모델 훈련 시간에 직접적인 영향을 미치므로 빠른 네트워크가 중요.
    - 높은 대역폭이 필요한 경우 InfiniBand를 고려할 수 있음.
        - InfiniBand
            - InfiniBand는 고성능 컴퓨팅(HPC)과 데이터센터에서 사용되는 고속 네트워크 인터커넥트 기술
            - 주요 특징
                - **초고속 대역폭과 낮은 지연시간**
                    - 현재 최대 400Gbps까지 지원 (HDR 기준)
                - 마이크로초 단위의 매우 낮은 지연시간
                - CPU 오버헤드를 줄이는 RDMA(Remote Direct Memory Access) 지원
            - **RDMA 기술**
                - InfiniBand의 핵심 기능 중 하나는 RDMA
- **RDMA (원격 직접 메모리 액세스):**
    - 노드나 애플리케이션 코드 수정 없이 네트워크 트래픽을 가속화할 수 있으며, 컴퓨터가 프로세서나 OS 개입 없이 메인 메모리의 데이터를 교환할 수 있게 함
    - 한 서버의 메모리에서 다른 서버의 메모리로 CPU 개입 없이 직접 데이터를 전송할 수 있어, CPU 사용률을 크게 줄이고 처리 속도를 향상시킴
- **전문 프로토콜:**
    - 분산 학습 확장 문제를 해결하기 위해 공급업체별 전문 프로토콜이 존재.
        - 노드 CPU와 OS 개입 없이 여러 노드의 GPU 간에 정보를 직접 교환할 수 있게 함
        - **MPI (Message Passing Interface):** 분산 프로세스 간 데이터 전송을 위한 표준화된 휴대용 API.
        - **NCCL (NVIDIA Collective Communications Library):** 토폴로지 인식 멀티 GPU 통신 프리미티브 라이브러리.



**스마트 스케줄링 및 자동 스케일링**

- 대부분의 ML 워크플로우는 일괄 처리(Batch) 특성을 가지므로 **클러스터 오토스케일러**를 활용하는 것이 좋음
- GPU 지원 하드웨어는 비용이 많이 들기 때문에, **사용하지 않을 때 비용을 지불하지 않도록** 하는 것이 중요
- 테인트(Taints) 및 허용 오차(Tolerations)를 사용.
    - 확장된 리소스(예: NVIDIA GPU)를 키로 사용하여 노드를 테인트
        - GPU 지원 하드웨어는 비용이 많이 들기 때문에, 클러스터 관리자는 일반적으로 테인트(Taints)를 사용하여 특정 노드에 머신 러닝 워크로드만 스케줄되도록 제한
    - 어드미션 컨트롤러 활용
        - **Admission Controller**는 API 요청이 etcd에 저장되기 전에 요청을 가로채서 검증하거나 수정하는 플러그인
                
                ```yaml
                # kube-apiserver 설정에 추가
                --enable-admission-plugins=ExtendedResourceToleration,...
                ```
                
                ```yaml
                사용자/kubectl → API Server → Authentication → Authorization → Admission Controllers → etcd
                                                                              ↑ 여기서 요청 검증/수정
                ```
                
            - Admission Controller의 두 가지 타입
                
                **Validating Admission Controller**
                
                - 요청을 검증만 함 (승인/거부 결정)
                - 예: "이 네임스페이스에는 GPU 리소스 요청 금지"
                
                **Mutating Admission Controller**
                
                - 요청을 수정할 수 있음
                - 예: "GPU 요청하는 파드에 자동으로 toleration 추가"
            - Taint : `Key: nvidia.com/gpu, Effect: NoSchedule`
                
                ```yaml
                kubectl taint nodes gpu-node-1 nvidia.com/gpu=:NoSchedule
                ```
                
            - `ExtendedResourceToleration`
                - **ExtendedResourceToleration** 어드미션 컨트롤러를 활용하면, 확장 리소스(Extended Resources)를 요청하는 파드에 대해 사용자가 수동으로 toleration을 추가할 필요가 없음
                - 파드가 특정 확장 리소스(예: `nvidia.com/gpu: 1`)를 요청하는 경우, 이 컨트롤러는 노드에 설정된 해당 테인트(예: `nvidia.com/gpu`)에 대한 **적절한 toleration을 파드에 자동으로 추가**하여, 해당 파드가 GPU 노드에 스케줄될 수 있도록 함
                    
                    ```yaml
                    apiVersion: v1
                    kind: Pod
                    metadata:
                      name: gpu-pod
                    spec:
                      containers:
                      - name: cuda-container
                        resources:
                          limits:
                            nvidia.com/gpu: 1  # GPU 요청만 하면 됨
                      # toleration이 자동으로 추가됨!
                    ```
                    
    - **GPU 포화 상태 유지:**
        - 가장 비용이 많이 드는 리소스인 GPU가 병목 현상을 일으키지 않도록 활용도를 높여야 함.
        - GPU, CPU, 네트워크 및 스토리지 사용률을 추적하는 모니터링을 설정해야 함

# 15장

Kubernetes를 확장하여 더 높은 수준의 기능을 제공하는 기술 솔루션들

> A. 사이드카 (Sidecars)
> 

사이드카 컨테이너는 메인 애플리케이션 컨테이너와 함께 실행되어 메인 애플리케이션에서 분리된 추가 기능을 제공. 이들은 종종 별도의 팀에서 유지 관리

- **용도:**
    - 서비스 메시의 맥락에서 대중화되었으며, 예를 들어 mTLS 인증을 투명하게 제공할 수 있습니다.
    - Dapr (분산 애플리케이션 런타임) 프로젝트가 좋은 예

> B. 어드미션 컨트롤러 (Admission Controllers)
> 

어드미션 컨트롤러는 클러스터의 백킹 스토어에 저장되기 전에 **Kubernetes API 요청을 읽는 인터셉터**.

- **기능:**
    - API 오브젝트의 유효성을 검사하거나 수정할 수 있음
- **활용 (수정):**
    - 사이드카를 사용하여 클러스터에서 생성된 모든 파드에 사이드카를 **자동으로 추가**할 수 있음
    - 이는 개발자가 사이드카에 대해 알 필요 없이 이점을 누릴 수 있도록 함
- **활용 (유효성 검사/제한):**
    - Kubernetes 사용 모범 사례를 따르는 파드 및 기타 리소스를 제출하도록 개발자를 유도하는 'linter'(코드 분석 도구) 역할을 할 수 있음.
    - 예를 들어, 리소스를 예약하지 않은 요청을 가로채서 거부할 수 있음.
- **중요 고려 사항:**
    - 고급 사용자가 린트 규칙을 거부할 수 있도록 특수 주석과 같은 **이스케이프 해치**를 남겨두어야 함.

> C. 사용자 정의 리소스 정의 (CRD, Custom Resource Definitions)
> 

CRD는 기존 Kubernetes 클러스터에 **새 리소스(new resources)를 동적으로 추가**하는 방법

개발자에게 더 높은 수준의 기본 요소를 제공하기 위해 Kubernetes 위에 **더 높은 수준의 추상화**를 추가하는 데 중요한 역할을 수행. 

- **작동 방식:**
    - 예를 들어, 새로운 `ReplicatedService` 리소스를 추가하는 데 CRD를 사용할 수 있음
    - 이러한 새로운 리소스 유형은 클러스터 자체에 배포되는 제어 루프(control loop)로 관리됨
    - 이 제어 루프는 사용자가 선언한 새로운 리소스의 상태를 모니터링
    - 클러스터의 실제 상태를 원하는 상태와 일치시키기 위해 내부적으로 필요한 표준 Kubernetes 리소스(예: Pods, Services, Deployments)를 생성하거나 수정하는 역할을 수행
- 새로운 상위 리소스가 기존 Kubernetes 객체와 함께 공존하므로, 사용자는 기본 제공 도구를 사용하여 모든 Kubernetes 리소스(기본 제공 리소스 및 확장 리소스 모두)와 상호 작용할 수 있음 (친숙한 환경 유지)