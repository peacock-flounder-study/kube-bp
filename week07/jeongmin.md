# CH14
## 새롭게 알게 된 것
### 쿠버네티스 클러스터에 gpu가 있는 것을 확인하기
```
kubectl get nodes -o yaml | grep -i nvidia.com/gpu
```

### Job Resource를 생성하기
- 특히 resources에 nvidia.com/gpu를 통해 gpu 수를 지정
  - device plugin을 daemonset으로 설치하도록 함 (다른 디바이스에서도 활용 가능)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app: mnist-demo
  name: mnist-demo
spec:
  template:
    metadata:
      labels:
        app: mnist-demo
    spec:
      containers:
      - name: mnist_demo
        image: lachlanevenson/tf-mnist:gpu
        args: ["--max_step", "500"]
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            nvidia.com/gpu: 1
      restartPolicy: OnFailure
```

### taint and toleration
taint를 할당하면 toleration이 지정되지 않은 모든 워크로드는 해당 노드에 할당되지 않음 

### CH15
## 새롭게 알게된 것
### Sidecar
- logging, proxy, config reload 등의 사이드카 패턴을 활용하여 보조할 수 있음
- 동일한 네트워크, 네임스페이스, 스토리지를 공유함

### Admission Controller
- K8s api 서버로 들어온 요청이 etcd에 저장되기 전 가로채서 검증 및 변경하는 플러그인
- kubectl -> auth -> mutating admission (요청 변경) -> schema validation -> admission validation -> etcd저장
