# K8s LimitRange 관리

# 01. LimitRange란?

## 01. 개요

LimitRange는 네임스페이스에서 각 파드가 resource를 limit 설정할 수 있는 범위를 지정한다. 또 default value를 설정하여 별도의 requests와 limit이 지정되지 않은 pod에게 default 값으로 자원 할당 정책을 세운다.

# 02. LimitRange 설정

[design-proposals-archive/admission_control_limit_range.md at main · kubernetes/design-proposals-archive](https://github.com/kubernetes/design-proposals-archive/blob/main/resource-management/admission_control_limit_range.md)

## 01. container의 LimitRange 설정

단일 컨테이너에 대한 limit을 설정한다. 

container 타입은 cpu와 memory에 대해서만 설정이 가능하다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limitrange-test
spec:
  limits:
  - type: Container
    default:
      cpu: 20m
      memory: 10Mi
    defaultRequest:
      cpu: 10m
      memory: 5Mi
    max:
      cpu: 30m
      memory: 20Mi
    min:
      cpu: 5m
      memory: 5Mi
```

- type: LimitRange 설정을 할 타입을 지정한다. Container, Pod, PersistentVolumeClaim 세 종류가 있다. type을 지정하지 않을 경우 Container가 자동으로 선택된다. Pod와 PersistentVolumeClaim은 아래서 다룬다.
- default : resource.limits가 정의되지 않을 경우 컨테이너에 부여할 기본 limits 값이다.
- defaultRequest: resource.requests가 정의되지 않을 경우 컨테이너에 부여할 기본 requests 값이다.
- min : defaultRequest 보다 작거나 같게 설정한다. request나 limit이 적용될 최소값이다.
- max : default 보다 크거나 같게 설정한다. request나 limit이 적용될 최대값이다.

테스트를 위해 위 YAML파일을 적용시켜보자.

```json
**# kubectl apply -f container-limitrange.yaml -n resource-limit** 
limitrange/container-limitrange-test created

**# kubectl -n resource-limit get limitrange**
NAME                        CREATED AT
container-limitrange-test   2022-10-11T07:21:43Z

**# kubectl -n resource-limit describe limitrange container-limitrange-test** 
Name:       container-limitrange-test
Namespace:  resource-limit
Type        Resource  Min  Max   Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---   ---------------  -------------  -----------------------
**Container   cpu       5m   30m   10m              20m            -
Container   memory    5Mi  20Mi  5Mi              10Mi           -**
```

아무런 자원 할당 없이 Pod를 생성해보자.

```json
**# kubectl -n resource-limit run non-spec-resource --image=nginx**
pod/non-spec-resource created

**# kubectl -n resource-limit describe pod non-spec-resource | egrep -A 2 'Limits|Request**s'
    Limits:
      **cpu:     20m
      memory:  10Mi**
    Requests:
      **cpu:        10m
      memory:     5Mi**
```

디폴트로 설정한 requests와 limits가 설정된 것을 확인할 수 있다.

### 01. min 불충족 테스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-less-than-min
  namespace: resource-limit
spec:
  containers:
  - image: nginx
    name: test-limitrange
    resources:
      requests:
        cpu: 1m
        memory: 4Mi
      limits:
        cpu: 5m
        memory: 5Mi
```

limit은 min 값에 충족하지만 requests는 min보다 작게 설정됐다.

```json
**# kubectl apply -f test-less-than-min.yaml** 
Error from server (Forbidden): error when creating "test-less-than-min.yaml": pods "test-less-than-min" is forbidden: [**minimum cpu usage per Container is 5m, but request is 1m, minimum memory usage per Container is 5Mi, but request is 4Mi**]
```

최소값을 충족시키지 못했다는 메시지가 뜨면서 적용이 안된다.

### 02. max 불충족 테스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-more-than-max
  namespace: resource-limit
spec:
  containers:
  - image: nginx
    name: test-limitrange
    resources:
      requests:
        cpu: 5m
        memory: 5Mi
      limits:
        cpu: 31m
        memory: 21Mi
```

requests는 min 값에 정확히 맞췄지만 limits이 max보다 높게 설정됐다.

```json
**# kubectl apply -f test-more-than-max.yaml** 
Error from server (Forbidden): error when creating "test-more-than-max.yaml": pods "test-more-than-max" is forbidden: [maximum cpu usage per Container is 30m, but limit is 31m, maximum memory usage per Container is 20Mi, but limit is 21Mi]
```

맥스보다 더 높게 설정됐다며 에러메시지가 뜬다.

### 03. requests가 default보다 높을 경우 테스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-requests-more-than-default
  namespace: resource-limit
spec:
  containers:
  - image: nginx
    name: test-limitrange
    resources:
      requests:
        cpu: 21m
        memory: 11Mi
```

requests에 할당된 자원이 default로 설정된 limit보다 크게 설정한 뒤, limits은 따로 지정하지 않는다.

```json
**# kubectl apply -f test-requests-more-than-default.yaml** 
The Pod "test-requests-more-than-default" is invalid: 
*** spec.containers[0].resources.requests: Invalid value: "21m": must be less than or equal to cpu limit
* spec.containers[0].resources.requests: Invalid value: "11Mi": must be less than or equal to memory limit**
```

request 값이 limits 보다 크다는 에러메시지가 뜬다.

## 02. Pod의 LimitRange 설정

Pod에 속한 모든 Containers의 총합에 대한 LimitRange를 설정한다. 컨테이너 단위가 아니라 Pod 단위로 설정하기 때문에 default 값을 따로 설정할 수 없다.

pod 타입은 container 타입과 마찬가지로 cpu와 memory만 설정이 가능하다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limitrange-test
spec:
  limits:
  - type: Pod
    max:
      cpu: 100m
      memory: 500Mi
    min:
      cpu: 5m
      memory: 5Mi
```

- max  : 모든 컨테이너의 자원사용량 총합이 max에 지정된 값보다 적어야 한다.
- min :  모든 컨테이너의 자원사용량 총합이 min에 지정된 값보다 높아야 한다.

```json
# kubectl apply -f pod-limitrange-test.yaml -n resource-limit 
limitrange/pod-limitrange-test created

**# kubectl -n resource-limit get limitranges** 
NAME                  CREATED AT
pod-limitrange-test   2022-10-11T07:52:47Z

**# kubectl -n resource-limit describe limitranges pod-limitrange-test** 
Name:       pod-limitrange-test
Namespace:  resource-limit
Type        Resource  Min  Max    Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---    ---------------  -------------  -----------------------
Pod         cpu       5m   100m   -                -              -
Pod         memory    5Mi  500Mi  -                -              -
```

### 01. min 불충족 테스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-less-than-min
  namespace: resource-limit
spec:
  containers:
  - image: nginx
    name: c1
    resources:
      requests:
        cpu: 3m
        memory: 3Mi
      limits:
        cpu: 5m
        memory: 5Mi
  - image: busybox
    name: c2
    resources:
      requests:
        cpu: 1m
        memory: 1Mi
      limits:
        cpu: 5m
        memory: 5Mi
```

c1 컨테이너와 c2 컨테이너의 requests 값이 min 보다 낮게 설정됐다. (cpu 4m, memory 4Mi)

```json
**# kubectl apply -f test-pod-less-than-min.yaml** 
Error from server (Forbidden): error when creating "test-pod-less-than-min.yaml": pods "test-pod-less-than-min" is forbidden: **[minimum cpu usage per Pod is 5m, but request is 4m, minimum memory usage per Pod is 5Mi, but request is 4194304**]
```

### 02. max 불충족 테스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-more-than-max
  namespace: resource-limit
spec:
  containers:
  - image: nginx
    name: c1
    resources:
      requests:
        cpu: 5m
        memory: 5Mi
      limits:
        cpu: 60m
        memory: 400Mi
  - image: busybox
    name: c2
    resources:
      requests:
        cpu: 5m
        memory: 5Mi
      limits:
        cpu: 41m
        memory: 101Mi
```

c1 컨테이너와 c2 컨테이너의 limits 합이 max보다 높게 설정됐다. (cpu: 101m, memory: 501Mi)

```json
**# kubectl apply -f test-pod-more-than-max.yaml** 
Error from server (Forbidden): error when creating "test-pod-more-than-max.yaml": pods "test-pod-more-than-max" is forbidden: [**maximum cpu usage per Pod is 100m, but limit is 101m, maximum memory usage per Pod is 500Mi, but limit is 525336576**]
```

너무 높아서 에러메시지가 뜬다.

## 03. PVC의 LimitRange 설정

PVC의 LimitRange는 Pod나 Container가 아닌 PVC의 requests에 대해 제한을 건다.

PersistentVolumeClaim 타입은 storage에 대해서 제한을 걸 수 있다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-limitrange-test
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 20Gi
    min:
      storage: 2Gi
```

- max : PVC가 요구할 수 있는 용량의 최고 수치다.
- min : PVC가 요구할 수 있는 용량의 최저 수치다.

```json
**# kubectl apply -f pvc-limitrange-test.yaml -n resource-limit** 
limitrange/pvc-limitrange-test created

**# kubectl -n resource-limit get limitranges** 
NAME                  CREATED AT
**pvc-limitrange-test**   2022-10-11T08:18:17Z

**# kubectl -n resource-limit describe limitranges pvc-limitrange-test** 
Name:                  pvc-limitrange-test
Namespace:             resource-limit
Type                   Resource  Min  Max   Default Request  Default Limit  Max Limit/Request Ratio
----                   --------  ---  ---   ---------------  -------------  -----------------------
PersistentVolumeClaim  storage   2Gi  20Gi  -                -              -
```

### 01. min 불충족 테스트

테스트를 위해 [02. nfs-subdir-external-provisioner](https://www.notion.so/02-nfs-subdir-external-provisioner-9c2e303c11c0455cbc686ff6fcc8a5b7?pvs=21) 에서 만든 pvc-nfs-subdir 스토리지 클래스를 사용한다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc-less-than-min
  namespace: resource-limit
spec:
  storageClassName: pvc-nfs-subdir
  resources:
    requests:
      **storage: 1Gi**
  accessModes:
  - ReadWriteOnce
```

request의 값이 min보다 적다.

```json
**# kubectl apply -f test-pvc-less-than-min.yaml** 
Error from server (Forbidden): error when creating "test-pvc-less-than-min.yaml": persistentvolumeclaims "test-pvc-less-than-min" is forbidden: **minimum storage usage per PersistentVolumeClaim is 2Gi, but request is 1Gi**
```

최소값보다 적게 설정되서 안만들어진다.

### 02. max 불충족 테스트

테스트를 위해 [02. nfs-subdir-external-provisioner](https://www.notion.so/02-nfs-subdir-external-provisioner-9c2e303c11c0455cbc686ff6fcc8a5b7?pvs=21) 에서 만든 pvc-nfs-subdir 스토리지 클래스를 사용한다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc-more-than-max
  namespace: resource-limit
spec:
  storageClassName: pvc-nfs-subdir
  resources:
    requests:
      **storage: 21Gi**
  accessModes:
  - ReadWriteOnce
```

request의 값이 max보다 크다.

```json
**# kubectl apply -f test-pvc-more-than-max.yaml** 
Error from server (Forbidden): error when creating "test-pvc-more-than-max.yaml": persistentvolumeclaims "test-pvc-more-than-max" is forbidden: **maximum storage usage per PersistentVolumeClaim is 20Gi, but request is 21Gi**
```

최대값보다 크게 설정되서 안만들어진다.
