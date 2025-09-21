# CH5
## 새롭게 알게 된 것
### Continuous Deployment 기본
- 실제 코드의 예
```yaml
kind: Deployment
apiVersion: v1
metadata:
    name: frontend
spec:
    replicas: 3
    template:
        spec:
            containers:
            - name: frontend
            - image: brendanburns/frontend:v1
            - readinessProbe:
                httpGet:
                    path: /readiness
                    port: 80
            - lifecycle:
                preStop:
                    exec:
                        command: ["usr/sbin/nginx", "-s", "quit"]
    strategy:
        type: RollingUpdate
        rollingUpdate:
            maxSurge: 1
            maxUnavailable: 1
```

- readiness probe && lifecycle prestop hook
위의 방식에서 두가지는 아래와 같이 동작함
1. 파드가 Terminating 상태로 변경
2. 동시에 발생:
    2.1. PreStop Hook 실행
    2.2. Service 엔드포인트에서 제거 (Readiness와 무관하게)
3. PreStop Hook 완료 후 SIGTERM 전송
4. Grace period 대기
5. 여전히 실행 중이면 SIGKILL로 강제 종료

> lifecycle prestop hook은 gradeful shutdown할 때 사용하는 핸들러로 컨테이너가 삭제되기 직전에 작동함

### Rolling Update
하나를 삭제 -> 다음 것을 띄움 이러한 방식으로 운영

### Bluegreen Update
- 장점: 
    - 별도 두개의 스키마를 DB가 지원하지 않아도 됨
- 고려사항:
    - DB를 정확히 마이그레이션 해야하는데, 처리중인 트랜잭션등이 반영되지 않을 수 있음
    - 동일한 스펙의 두 환경을 준비해야하는 리소스가 필요함

### Canary Update
- 장점: 
    - 비율을 조정하여 배포할 수 있음
    - 이전버전과 새로운 버전에 대해서 효과를 비교하여 A/B test에 활용 가능
- 고려사항:
    - 현재의 버전이 이전버전보다 낫다는 것에 대한 확신의 메트릭이 필요
    - 이전 버전과 현재 버전의 DB가 스키마를 둘다 지원해야 함

### 운영에서의 테스트 
- 왜 운영에서 테스트해야하는가?
    - stage에서는 실제 운영과의 차이 발생
    - 다른 부하를 지닌환경
- 필요한 고려사항: distributed tracing, instrumentation, chaos engineering, traffic shadow
- 이에대한 도구들: canary deploy, A/B Test, Traffic Control, feature flag
- 모니터링 확보 -> 안정된 상태 정의 -> 가설 -> 실험 진행 (문제 범위를 최소화 해야함)


# CH6
## 새롭게 알게 된 것
### Helm으로 릴리즈 관리를 하는 방법
기존 values.yaml이 아래와 같을 때
```yaml
image: nginx:stable
```

새롭게 키-값을 추가하고자 하면
```yaml
image: nginx:stable
hello: world
```
`helm upgrade --namespace <ns> <release> <chart_path>` 의 방식으로 install이 아닌 upgrade를 사용해야함

그럼 helm chart의 `REVISION` 값이 올라가게 됨