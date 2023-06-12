# kubectl set

# 01. 이미지 업데이트

<aside>
💡 **# kubectl -n {네임스페이스 명} set image {리소스 종류} {리소스 명} {컨테이너명}={이미지명}:{이미지 태그}**

</aside>

```json
**# kubectl -n test-set get deploy**
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
**test-set**   1/1     1            1           14s

**# kubectl -n test-set set image deploy test-set nginx=nginx:1.21.1**
deployment.apps/test-set image updated

**# kubectl -n test-set get deploy test-set -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'**
nginx:1.21.1
```

# 02. 환경변수 업데이트

<aside>
💡 **# kubectl -n {네임스페이스 명} set env {리소스 종류} {리소스 명} --from {[secret|configmap]}/{시크릿 또는 컨피그맵 명} [--prefix={변수에 포함될 전치사}]**

</aside>

시크릿과 컨피그맵을 만들자

```json
**# kubectl -n test-set create secret generic test-set --from-literal=testenv=12345**
secret/test-set created

**# kubectl -n test-set create configmap test-set --from-literal=envtest=12345**
configmap/test-set created
```

현재 존재하고 있는 deployment를 사용하자.

```json
**# kubectl -n test-set get deployment**
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
**test-set**   1/1     1            1           18m

**# kubectl -n test-set set env deployment test-set --from secret/test-set --from configmap/test-set**
Warning: key envtest transferred to ENVTEST
deployment.apps/test-set env updated

**# kubectl -n test-set set env deployment test-set --from secret/test-set --prefix TEST_**
Warning: key testenv transferred to TESTENV
deployment.apps/test-set env updated
```

set env를 사용하면 소문자로 지정되어 있던 변수명이 자동으로 대문자로 변경된다.

```json
**# kubectl -n test-set get deploy test-set -o jsonpath='{.spec.template.spec.containers[0].env}' | jq**
[
  {
    "name": "ENVTEST",
    "valueFrom": {
      "configMapKeyRef": {
        "key": "envtest",
        "name": "test-set"
      }
    }
  },
  {
    "name": "TEST_TESTENV",
    "valueFrom": {
      "secretKeyRef": {
        "key": "testenv",
        "name": "test-set"
      }
    }
  }
]
```

# 03. resource 업데이트

<aside>
💡 **# kubectl set resources {리소스 종류} {리소스 명} --requests cpu={cpu 값},memory={메모리 값} --limits cpu={cpu 값},memory={메모리 값}**

</aside>

```yaml
**# kubectl -n resource-set get deploy**
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           15s

**# kubectl -n resource-set set resources deploy nginx --requests cpu=100m,memory=100M --limits cpu=200m,memory=200M**
deployment.apps/nginx resource requirements updated

# **kubectl -n resource-set get deploy nginx -o jsonpath='{.spec.template.spec.containers[0].resources}' | jq**
{
  "limits": {
    "cpu": "200m",
    "memory": "200M"
  },
  "requests": {
    "cpu": "100m",
    "memory": "100M"
  }
}
```
