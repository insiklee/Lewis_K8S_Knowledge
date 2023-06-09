# K8s Downward API 관리

# 01. Downward API란?

## 01. 개요

컨테이너는 가끔 자기 스스로의 정보를 사용하기도 해야한다.  파드가 속한 네임스페이스 정보가 필요하거나, 파드가 실행된 호스트 노드의 IP 주소가 필요할때도 있다. 이 때 필요한 컨테이너, 호스트의 정보를 환경변수나 파일로 받아서 사용할 수 있도록 하는 방법이 Downward API다.

## 02. Downward API의 사용

Downward API는 쿠버네티스 API의 특정 필드를 가져와서 사용한다. 모든 필드가 가능한 것은 아니고, Pod 레벨이나 컨테이너 레벨에서 사용할 수 있는 일부 필드가 있다. 목록은 아래와 같다.

### 01. pod 레벨에서의 filedRef

- [metadata.name](http://metadata.name) : 파드의 이름이다
- metadata.namespace : 파드의 네임스페이스다
- metadata.uid : 파드의 UID를 의미한다
- metadata.annotations[’{키}’] : annotation의 특정 키의 값을 의미한다. 환경변수로만 사용할 수 있고, volume 마운트는 불가능하다.
- metadata.labels[’{키}’] : label의 특정 키의 값을 의미한다. 환경변수로만 사용할 수 있고, volume 마운트는 불가능하다.
- spec.serviceAccountName : Pod가 지닌 서비스어카운트의 이름을 뜻한다. 환경변수로만 사용할 수 있고, volume 마운트는 불가능하다.
- spec.nodeName : pod가 실행된 노드 이름을 뜻한다. 환경변수로만 사용할 수 있고, volume 마운트는 불가능하다.
- status.hostIP : pod가 실행된 노드의 IP를 뜻한다. 환경변수로만 사용할 수 있고, volume 마운트는 불가능하다.
- status.podIP : pod의 IP를 뜻한다. 환경변수로만 사용할 수 있고, volume 마운트는 불가능하다.
- medatadata.labels : 파드의 모든 라벨이다. volume 마운트 했을때만 사용 가능하고, 환경변수로는 사용할 수 없다.
- medatata.annotations : 파드의 모든 annotation이다. volume 마운트 했을 때만 사용 가능하고, 환경변수로는 사용할 수 없다.

### 02. 컨테이너 레벨에서 resourceFiledRef

컨테이너 레벨에서 가져올 수 있는 field는 대부분 컨테이너의 리소스 제한과 관련된 데이터다. 

- resource: limits.cpu
- resource: requests.cpu
- resource: limits.memory
- resource: requests.memory
- resource: limits.hugepage-*
- resource: requests.hugepage-*
- resource: limits.ephemeral-storage
- resource: requests.ephemeral-storage

# 02. Downward API를 사용

## 01. 환경변수 지정

### 01. POD 레벨 Field 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: downapi-test
  name: downapi-test
spec:
  containers:
  - image: nginx
    name: downapi-test
    env:
    - name: DOWNAPI_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: DOWNAPI_POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: DOWNAPI_POD_SA
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: DOWNAPI_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: DOWNAPI_HOST_IP
      valueFrom:
        fieldRef:
          fieldPath: status.hostIP
    - name: DOWNAPI_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
```

```json
**# kubectl apply -f downpai-test.yaml** 
pod/downapi-test created

**# kubectl get pod**
NAME           READY   STATUS    RESTARTS   AGE
downapi-test   1/1     Running   0          39s

**# kubectl exec -it downapi-test -- env | grep DOWNAPI**
DOWNAPI_POD_IP=10.40.0.2
DOWNAPI_POD_NAME=downapi-test
DOWNAPI_POD_NAMESPACE=default
DOWNAPI_POD_SA=default
DOWNAPI_NODE_NAME=worker2
DOWNAPI_HOST_IP=192.168.2.13
```

### 02. Container 레벨 resourceField 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: downapi-test2
  name: downapi-test2
spec:
  containers:
  - image: nginx
    name: downapi-test2
    resources:
      requests:
        memory: 100Mi
        cpu: 100m
      limits:
        memory: 200Mi
        cpu: 200m
    env:
    - name: DOWNAPI_REQ_MEM
      valueFrom:
        resourceFieldRef:
            containerName: downapi-test2
            resource: requests.memory
    - name: DOWNAPI_REQ_CPU
      valueFrom:
        resourceFieldRef:
            containerName: downapi-test2
            resource: requests.cpu
    - name: DOWNAPI_LIMIT_MEM
      valueFrom:
        resourceFieldRef:
            containerName: downapi-test2
            resource: limits.memory
    - name: DOWNAPI_LIMIT_CPU
      valueFrom:
        resourceFieldRef:
            containerName: downapi-test2
            resource: limits.cpu
```

resourceFiled를 사용할때는 정보를 가져올 컨테이너의 이름을 알아야만 한다. 

```json
**# kubectl apply -f downpai-test2.yaml** 
pod/downapi-test2 created

**# kubectl exec -it downapi-test2 -- env | grep DOWNAPI**
DOWNAPI_LIMIT_MEM=209715200
DOWNAPI_LIMIT_CPU=1
DOWNAPI_REQ_MEM=104857600
DOWNAPI_REQ_CPU=1
```

## 02. 볼륨 마운트

### 01. POD 레벨 Filed 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: downapi-test3
  name: downapi-test3
spec:
  containers:
  - image: nginx
    name: downapi-test3
    volumeMounts:
    - name: podinfo
      mountPath: /podinfo
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "podname"
        fieldRef:
          fieldPath: metadata.name
      - path: "podnamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
```

```json
**# kubectl apply -f downpai-test3.yaml** 
pod/downapi-test3 created

**# kubectl exec -it downapi-test3 -- ls /podinfo**
labels	podname  podnamespace

**# kubectl exec -it downapi-test3 -- cat /podinfo/labels**
run="downapi-test3"

**# kubectl exec -it downapi-test3 -- cat /podinfo/podname**
downapi-test3

**# kubectl exec -it downapi-test3 -- cat /podinfo/podnamespace**
default
```

### 02. Container 레벨 resourceField 예시

```yaml
apiVersion: v1
kind: Pod 
metadata:
  labels:
    run: downapi-test4
  name: downapi-test4
spec:
  containers:
  - image: nginx
    name: downapi-test4
    resources:
      requests:
        memory: 100Mi
        cpu: 100m
      limits:
        memory: 200Mi
        cpu: 200m
    **volumeMounts:
    - name: podinfo
      mountPath: /podinfo**
  **volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: mem_req
        resourceFieldRef:
            containerName: downapi-test4
            resource: requests.memory
      - path: cpu_req
        resourceFieldRef:
            containerName: downapi-test4
            resource: requests.cpu
      - path: mem_limit
        resourceFieldRef:
            containerName: downapi-test4
            resource: limits.memory
      - path: cpu_limit
        resourceFieldRef:
            containerName: downapi-test4
            resource: limits.cpu**
```

```yaml
**# kubectl apply -f downpai-test4.yaml** 
pod/downapi-test4 created

**# kubectl exec -it downapi-test4 -- ls /podinfo**
cpu_limit  cpu_req  mem_limit  mem_req

**# kubectl exec -it downapi-test4 -- cat /podinfo/cpu_limit**
1

**# kubectl exec -it downapi-test4 -- cat /podinfo/cpu_req**
1

**# kubectl exec -it downapi-test4 -- cat /podinfo/mem_limit**
209715200

**# kubectl exec -it downapi-test4 -- cat /podinfo/mem_req**
104857600
```
