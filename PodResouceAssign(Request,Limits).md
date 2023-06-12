# K8s 파드 자원 할당 관리 (Request, Limits)

# 01. 파드 자원 할당

## 01. 개요

열심 Pod는 자신의 맡은바 소임을 아주 잘 하고 싶다. 그래서 아주 열심히 일을 해서 클러스터를 구성하는 하나의 일원으로서 인정받고 싶어한다. 하지만 열심 Pod를 실행한 노드가 Pod에게 줄 수 있는 CPU와 메모리 자원은 한정되어 있다. 그 때문에 열심 Pod가 노드의 모든 자원을 가져가버리면 다른 Pod가 가져갈 자원이 부족해진다.

이런 문제를 막기 위해서 열심 Pod가 가져갈 수 있는 자원을 제한해야만 한다. 이를 Limits이라 한다.

반면 열심 Pod에게 아무런 자원을 주지 않으면 금새 시무룩해지면서 제대로된 퍼포먼스를 못낼 수 있다. 이 때문에 최소한의 자원이 할당될 수 있도록 해주는데, 이를 Request라고 한다.

## 02. 파드 자원 할당의 활용

파드의 자원 할당은 아래 리소스에서 활용된다. 반대로 이야기하면 아래 오브젝트를 사용하기 위해서는 각 Pod에 자원 할당이 적절히 이뤄저야 한다는 뜻이다.

- HPA에서 현재 cpu 및 메모리 사용률의 기준이 되므로 반드시 Requests가 설정되어야 한다. Requests가 없을 경우 metric 계산이 불가능하여 HPA가 정상적으로 동작하지 않는다.
- ResourceQuota가 적용된 네임스페이스에서 Pod를 실행시키기 위해선 반드시 Limits이 설정되어야한다.

# 02. 파드 자원 할당

## 01. YAML로 자원 할당

간단하게 CPU와 메모리 자원을 할당하는 방법은 아래와 같다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: resources-pod
  name: resources-pod
spec:
  containers:
  - image: nginx
    name: resources-pod
    resources:
      requests:
        cpu: 5m
        memory: 10Mi
      limits:
        cpu: 10m
        memory: 20Mi
```

위 YAML은 cpu 5milicpu와 10Mi를 최소한으로 적용하고, cpu 10milicpu와 20Mi 이상은 사용하지 못하게 제한하는 방식이다.

```json
**# kubectl apply -f resource-pod.yaml** 
pod/resources-pod created

**# kubectl describe pod resources-pod | egrep -i -A 2 'Limits|Requests'**
    Limits:
      cpu:     10m
      memory:  20Mi
    Requests:
      cpu:        5m
      memory:     10Mi
```

- **limits 적용 확인**
    
    비교를 위해서 아래와 같이 resources.limits가 적용된 파드와 적용이 안된 파드를 비교해보자. 
    
    ```json
    $ kubectl get pod -l "run in(resources-pod,non-resources-pod)" -o wide
    NAME                READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
    non-resources-pod   1/1     Running   0          42m   10.110.126.33   k8s-worker2   <none>           <none>
    resources-pod       1/1     Running   0          49m   10.110.126.3    k8s-worker2   <none>           <none>
    
    **# kubectl get pod resources-pod -o jsonpath='{.spec.containers[0].resources}' | jq**
    {
      "limits": {
        "cpu": "10m",
        "memory": "20Mi"
      },
      "requests": {
        "cpu": "5m",
        "memory": "10Mi"
      }
    }
    
    **# kubectl get pod non-resources-pod -o jsonpath='{.spec.containers[0].resources}' | jq**
    {}
    ```
    
    세션 두 개를 실행해서 아래 스크립트를 실행하여 지속적으로 각 Pod에 HTTP 연결을 시도한다. 
    
    ```json
    IPADDR={Pod의 IP주소}
    while true
    do
      curl $IPADDR
    done
    ```
    
    이후 Pod의 자원 사용량을 확인해본다.
    
    ```json
    $ kubectl top pod -l "run in (non-resources-pod,resources-pod)"
    NAME                CPU(cores)   MEMORY(bytes)   
    non-resources-pod   20m          3Mi             
    resources-pod       10m          4Mi
    ```
    
    limits가 적용된 pod는 설정된대로 10milicpu 이상 자원을 사용하지 않지만, limits가 적용되지 않은 pod는 그 이상 자원을 사용하는 것을 확인할 수 있다.
    

## 02. hugepage 할당

hugepage를 할당하기 위해서는 파드가 실행되는 노드에 hugepage가 이미 설정되어 있어야 한다. hugepage를 설정하기 위해 [Hugepage 설정](https://www.notion.so/Hugepage-6ff1b10a671844f0bdf4581dc007a481?pvs=21) 페이지를 참고하자.

hugepage를 할당할때는 pod에 할당될 hugepage의 총 크기를 입력해야 한다. 만약 2Mi를 사용하는 hugepage를 10개 할당한다면, hugepage-2Mi: 20Mi로 입력해야한다. 반대로 1Gi를 사용하는 hugepage를 20개 할당한다면 hugepage-2Gi: 20Gi로 입력한다.

hugepage를 설정하려면 **request와 limits에 cpu와 memory 관련 설정을 함께 추가해줘야** 한다.

**또한 resource와 limits에 들어가는 hugepage의 크기가 동일해야한다.**

- **2Mi hugepage 사용시**
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: hugepage-2m-pod
      name: hugepage-2m-pod
    spec:
      containers:
      - image: nginx
        name: hugepage-pod
        resources:
          requests:
            cpu: 10m
            memory: 20Mi
            **hugepages-2Mi: 10Mi**
          limits:
            cpu: 15m
            memory: 30Mi
            **hugepages-2Mi: 10Mi**
    ```
    
- **1Gi hugepage 사용시**
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: hugepage-1g-pod
      name: hugepage-1g-pod
    spec:
      containers:
      - image: nginx
        name: hugepage-pod
        resources:
          requests:
            cpu: 10m
            memory: 20Mi
            **hugepages-1Gi: 10Gi**
          limits:
            cpu: 15m
            memory: 30Mi
            **hugepages-1Gi: 10Gi**
    ```
    

```json
**# kubectl apply -f hugepage-2m-pod.yaml** 
pod/hugepage-2m-pod created

**# kubectl get pod hugepage-2m-pod -o wide**
NAME              READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
hugepage-2m-pod   1/1     Running   0          5s    **10.100.194.86**   k8s-worker1   <none>           <none>

**# kubectl describe node k8s-worker1 | grep -A 8 "Allocated resources"**
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                260m (6%)    15m (0%)
  memory             20Mi (0%)    30Mi (0%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  **hugepages-2Mi      10Mi (100%)  10Mi (100%)**
```

## 03. 임시 볼륨 크기 할당

Pod에 할당되는 임시 볼륨(emptyDir)은 말은 임시 볼륨이지만 실제로는 Pod가 실행되는 노드의 로컬 스토리지를 임시로 사용하게 된다. 그런데 만약 노드의 로컬 스토리지 공간이 부족하다면 emptyDir 생성에 제한이 생길 수도 있다. 따라서 emptyDir로 생성할 임시 볼륨의 크기 역시 제한할 필요도 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: limit-emptydir-pod
  name: limit-emptydir-pod
spec:
  containers:
  - image: nginx
    name: limit-emptydir-pod
    resources: 
      requests:
        ephemeral-storage: 1Gi
      limits:
        ephemeral-storage: 2Gi
    volumeMounts:
    - name: emptydir-limit
      mountPath: /emptydir
  volumes:
  - name: emptydir-limit
    emptyDir: {}
```

```json
**# kubectl apply -f limit-emptydir-pod.yaml** 
pod/limit-emptydir-pod created

**# kubectl get pod -o wide**
NAME                 READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
limit-emptydir-pod   1/1     Running   0          32s   10.100.194.95   k8s-worker1   <none>           <none>

**# kubectl describe node k8s-worker1 | grep -A 8 "Allocated resources"**
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests   Limits
  --------           --------   ------
  cpu                250m (6%)  0 (0%)
  memory             0 (0%)     0 (0%)
  **ephemeral-storage  1Gi (5%)   2Gi (11%)**
  hugepages-1Gi      0 (0%)     0 (0%)
  hugepages-2Mi      0 (0%)     0 (0%)
```
