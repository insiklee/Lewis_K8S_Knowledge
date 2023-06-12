# K8s ResourceQuota 관리

[Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

# 01. ResourceQuota란?

어떤 회사에서는 네임스페이스를 각 팀별 작업 할당 공간으로 부여하기도 한다. 이 경우 팀의 역할, 규모 등에 따라서 필요한 리소스의 양이 달라질 수 있다.

또 어떤 회사에서는 특정 네임스페이스를 대규모 트래픽이 발생하는 웹 서비스 전용으로 사용한다. 이 경우 엄청난 리소스가 할당될 필요가 있을 수 있다.

이런 경우를 대비하여 네임스페이스 단위로 자원을 관리하는 것이 ResourceQuota다.

## 01. ResourceQuota 이모저모

- resourceQuota에 requests.cpu나 requests.memory가 설정됐다면 해당 네임스페이스에서 생성되는 모든 Pod는 resource.requests 값이 설정되야 한다.
- resourceQuota에 limits.cpu나 limits.memory가 설정됐다면 해당 네임스페이스에서 생성되는 모든 Pod는 resource.limits 값이 설정되야 한다.
- 위의 과정이 번거롭다면 LimitRange로 디폴트 값을 설정하자

# 02. ResourceQuota 생성

## 01. 시스템 자원에 대한 quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-resource
  namespace: rq-test-ns
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 700Mi
    limits.cpu: "1.5"
    limits.memory: 1Gi
    hugepages-2Mi: 8Mi
```

위 requests.cpu와 requests.memory는 각각 cpu와 memory로 줄여서 사용할 수 있다.

```json
**# kubectl apply -f quota-resource.yaml** 
resourcequota/quota-resource created

**# kubectl -n rq-test-ns get resourcequota**
NAME             AGE   REQUEST                                                             LIMIT
quota-resource   34s   hugepages-2Mi: 0/8Mi, requests.cpu: 0/1, requests.memory: 0/700Mi   limits.cpu: 0/1500m, limits.memory: 0/1Gi

```

## 02. 스토리지에 대한 quota

### 01. PVC 요청용량 합계에 대한 quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-storage-requests-pvc
  namespace: rq-test-ns
spec:
  hard:
    **requests.storage: 80Gi**
```

### 02. PVC 개수에 대한 quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-storage-number-pvc
  namespace: rq-test-ns
spec:
  hard:
    persistentvolumeclaims: "10"
```

### 03. StorageClass별 PVC 요청용량 합계에 대한 quota

아래 코드를 만들기 전에 [02. nfs-subdir-external-provisioner](https://www.notion.so/02-nfs-subdir-external-provisioner-9c2e303c11c0455cbc686ff6fcc8a5b7?pvs=21) 를 참고하여 StorageClass **sc-nfs-subdir**를 생성한다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-storage-sc-requests-pvc
  namespace: rq-test-ns
spec:
  hard:
  # <StorageClass Name>.storageclass.storage.k8s.io/requests.storage
    sc-nfs-subdir.storageclass.storage.k8s.io/requests.storage: 80Gi
```

### 04. StorageClass별 PVC 개수에 대한 quota

아래 코드를 만들기 전에 [02. nfs-subdir-external-provisioner](https://www.notion.so/02-nfs-subdir-external-provisioner-9c2e303c11c0455cbc686ff6fcc8a5b7?pvs=21) 를 참고하여 StorageClass **sc-nfs-subdir**를 생성한다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-storage-sc-number-pvc
  namespace: rq-test-ns
spec:
  hard:
  # <StorageClass Name>.storageclass.storage.k8s.io/persistentvolumeclaims
    sc-nfs-subdir.storageclass.storage.k8s.io/persistentvolumeclaims: 5
```

### 05. 임시 볼륨에 대한 quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-storage-ephemeral
  namespace: rq-test-ns
spec:
  hard:
    requests.ephemeral-storage: 5Gi
    limits.ephemeral-storage: 10Gi
```

requests.ephemeral-storage는 ephemeral-storage로 줄여서 작성할 수 있다. 

## 03. 오브젝트 개수에 대한 quota

네임스페이스에서 실행되는 오브젝트에 대해서 제한을 걸 수 있다.

아래 예시를 보자

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-counts-simple
  namespace: rq-test-ns
spec:
  hard:
    pods: 20                    # Pod 개수
    services: 10                # service 개수
    services.nodeports: 3       # nodePort를 사용하는 service 개수
    services.loadbalancers: 3   # loadbalancer를 사용하는 service 개수
    configmaps: 10              # configmap 개수
    secrets: 10                 # secret 개수  
    persistentvolumeclaims: 5   # PVC 개수
    replicationcontrollers: 3   # rc 개수
    resourcequotas: 2           # resourcequota 개
```

위 오브젝트 외에 다른 오브젝트는 count/ 전치사를 붙이면 개수 제한이 가능하다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-counts-need-prefix
  namespace: rq-test-ns
spec:
  hard:
    count/cronjobs.batch: 3
    count/deployment.apps: 5
    count/statefulsets.apps: 2
```

d
