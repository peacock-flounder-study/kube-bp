## 새롭게 알게 된 것
### 개발 환경
- RBAC으로 사용자별 권한 할당을 함
- myuser라는 사용자에게 my-namespace 네임스페이스 내에서 edit 권한을 부여
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding 
metadata:
    name: example
    namespace: my-namespace
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: edit
subjects:
- apiGroup: rbac.authorization.k8s.io
    kind: User
    name: myuser
```
- 사용량 제한은 ResourceQuota를 사용 (네임스페이스별 할당)
- 삭제 자동화를 한다는데...
    - TTL을 하면 의도하지 않은 리소스가 삭제될 수 있는 것 아닌가?
    - 어떻게 이것을 방지할 수 있지..
