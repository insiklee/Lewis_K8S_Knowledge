# K8s Scheduler 관리

# 01. 스케쥴러란?

리소스를 어떤 노드에 할당할 것인지 정하는 오브젝트다.

# 02. 스케쥴링 방식

기본적으로 쿠버네티스 클러스터는 스케쥴러를 통해서 파드를 실행할 노드를 선택한다. 이때 사용하는 방식은 두 가지다. 바로 **필터링**과 **점수 계산법**이다. 

**필터링** 방식은 아래에서 소개할 nodeName이나 nodeSelector, nodeAffinity를 통해서 원하는 노드를 선택하는 방식이다.

**점수 계산법**은 필터링으로 걸러진 노드들 중에서 자원 사용량 등을 종합적으로 검토해서 더 적절한 노드에서 파드를 실행할 수 있도록 점수를 계산하는 방법이다.

# 03. 특정 노드에서 리소스 실행 방법

## 01. nodeName

nodeName 옵션으로 특정 파드의 실행 노드를 지정할 수 있다.

스케쥴러에 의해 관리되지 않고 지정된 노드의 kubelet이 Pod를 관리한다. 따라서 drain이나 cordon 상태를 무시하고 pod를 생성할 수 있다. 

**YAML 예시**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodename-pod
spec:
  nodeName: worker1.unnet.k8s.io
  containers:
  - image: nginx
    name: nginx
```

**YAML 실행 결과**

```json
**# kubectl apply -f nodeName-pod.yaml** 
pod/nodename-pod created

**# kubectl get pod -o wide**
NAME           READY   STATUS    RESTARTS   AGE   IP              NODE                   NOMINATED NODE   READINESS GATES
nodename-pod   1/1     Running   0          12s   172.16.79.152   worker1.unnet.k8s.io   <none>           <none>
```

**주의) nodeName을 사용할 경우 cordon 되어서 Unscheduled된 노드에서도 강제로 파드를 실행시킬 수 있게 된다.**

```json
**# kubectl get pod**
No resources found in default namespace.

**# kubectl cordon worker1.unnet.k8s.io** 
node/worker1.unnet.k8s.io cordoned

**# kubectl get nodes**
NAME                   STATUS                     ROLES           AGE   VERSION
master.unnet.k8s.io    Ready                      control-plane   20d   v1.25.0
**worker1.unnet.k8s.io   Ready,SchedulingDisabled**   <none>          20d   v1.25.0
worker2.unnet.k8s.io   Ready

**# kubectl apply -f nodeName-pod.yaml** 
pod/nodename-pod created

**# kubectl get pod**
NAME           READY   STATUS    RESTARTS   AGE
nodename-pod   1/1     **Running**   0          14s  <- unscheduled된 노드에서 실행됨
```

## 02. nodeSelector

nodeSelector는 노드에 부여된 label을 기준으로 노드를 선택할 수 있다.

노드의 hostname을 기준으로 선택할 수 있고, node에 부여된 role을 기준으로 선택할 수 있다.

### 01. 특정 노드 하나만 사용하는 경우

kubernetes.io/hostname 라벨을 사용해서 특정 노드를 선택한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodeselector-pod
spec:
  **nodeSelector:
    kubernetes.io/hostname: worker1.unnet.k8s.io**
  containers:
  - image: nginx
    name: container1
```

### 02. 특정 라벨을 기준으로 사용하는 경우

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  **nodeSelector:
    disktype: ssd**
```

**YAML 실행 결과**

```json
**# kubectl apply -f nodeSelector-pod.yaml** 
pod/nodeselector-pod created

**# kubectl get pod -o wide**
NAME               READY   STATUS    RESTARTS   AGE   IP              NODE                   NOMINATED NODE   READINESS GATES
nodeselector-pod   1/1     Running   0          9s    172.16.79.154   **worker1.unnet.k8s.io**   <none>           <none>
```

cordon된 노드를 nodeSelector로 지정하여 Pod를 실행할 경우 해당 Pod는 Pending이 된다.

```json
**# kubectl get nodes**
NAME                   STATUS                     ROLES           AGE   VERSION
master.unnet.k8s.io    Ready                      control-plane   20d   v1.25.0
**worker1.unnet.k8s.io   Ready,SchedulingDisabled**   <none>          20d   v1.25.0
worker2.unnet.k8s.io   Ready                      <none>          20d   v1.25.0

**# kubectl apply -f nodeSelector-pod.yaml** 
pod/nodeselector-pod created

**# kubectl get pod**
NAME               READY   STATUS    RESTARTS   AGE
nodename-pod       1/1     Running   0          65m <- nodeName으로 지정하면 실행된다.
**nodeselector-pod   0/1     Pending   0          12s** <- nodeSelector로 선택하면 펜딩된다.
```

## 03. nodeAffinity

nodeAffinity는 nodeSelector 보다 조금 복잡한 구문을 요구하지만, 그만큼 더욱 세분화된 설정을 할 수 있도록 도와준다.

nodeAffinity는 required node affinity와 preferred node affinity가 있다.

### 01. Required Node Affinity

Required Node Affinity는 Matching된 Label에 속한 노드에서만 실행시키겠다는 뜻이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-affinity
  name: nginx-affinity
spec:
  **affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd**
  containers:
  - image: nginx
    name: nginx-affinity
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

Required Node Affinity는 **requiredDuringSchedulingIgnoredDuringExecution:** 구문을 사용한다. 왜이렇게 긴 구문을 사용하는지는 모르겠지만 외우기엔 너무 길기 때문에 가급적 복사를 하자.

무조건 매칭된 노드에서만 실행하기 때문에 그 하위 구문은 nodeSelectorTerms가 된다.

### 라벨이 포함된 노드가 있는 경우

nodeSelectorTems에서 매칭되는 라벨이 있는 노드에서 실행된다.

```json
# **kubectl get node -l disktype=ssd**
NAME          STATUS   ROLES    AGE     VERSION
k8s-worker1   Ready    <none>   3d21h   v1.24.1

**# kubectl apply -f nginx-affinity.yaml** 
pod/nginx-affinity created

**# kubectl get pod -o wide**
NAME                         READY   STATUS    RESTARTS   AGE    IP               NODE          NOMINATED NODE   READINESS GATES
nginx-affinity               1/1     Running   0          94s    10.100.194.120   k8s-worker1   <none>           <none>
```

### 라벨이 포함된 노드가 없는 경우

매칭되는 라벨이 없을 경우 Pending 상태가 되며 실행되지 않는다.

```json
**# kubectl get node -l disktype=ssd**
No resources found

**# kubectl apply -f nginx-affinity.yaml** 
pod/nginx-affinity created

**# kubectl get pod -o wide**
NAME                         READY   STATUS    RESTARTS   AGE    IP               NODE          NOMINATED NODE   READINESS GATES
nginx-affinity               0/1     **Pending**   0          2s     <none>           <none>        <none>           <none>
```

### 02. Preferred Node Affinity

Preffered Node Affinity는 Matching된 Label에 속한 노드에 우선 실행권을 주지만, matching 되는 노드가 없으면 그냥 다른 node에서 실행하겠다는 뜻이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-affinity
  name: nginx-affinity
spec:
  **affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd**
  containers:
  - image: nginx
    name: nginx-affinity
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

Perferred Node Affinity는 **preferredDuringSchedulingIgnoredDuringExecution:** 구문을 사용한다. 왜이렇게 긴 구문을 사용하는지는 모르겠지만 외우기엔 너무 길기 때문에 가급적 복사를 하자.

무조건 실행하는 것이 아닌, 우선순위를 더 부여하는 것이기 때문에 weight 구문과 prefference 구문을 사용한다.

**weight**는 1에서 100사이의 값을 넣을 수 있다. preference에 충족된 노드에게 weight 만큼의 가산점을 부여해서 점수를 높인다.

### 라벨이 포함된 노드가 있는 경우

```json
# **kubectl get node -l disktype=ssd**
NAME          STATUS   ROLES    AGE     VERSION
k8s-worker1   Ready    <none>   3d21h   v1.24.1

**# kubectl apply -f nginx-affinity.yaml** 
pod/nginx-affinity created

**# kubectl get pod -o wide**
NAME                         READY   STATUS    RESTARTS   AGE    IP               NODE          NOMINATED NODE   READINESS GATES
nginx-affinity               1/1     Running   0          94s    10.100.194.120   k8s-worker1   <none>           <none>
```

### 라벨이 포함된 노드가 없는 경우

아무 노드나 실행된다.

```json
**# kubectl get node -l disktype=ssd**
No resources found

**# kubectl apply -f nginx-affinity.yaml** 
pod/nginx-affinity created

**# kubectl get pod -o wide**
NAME             READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
nginx-affinity   1/1     Running   0          7s    10.110.126.6   k8s-worker2   <none>           <none>
```

## 04. inter-pod Affinity / AntiAffinity

[Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#types-of-inter-pod-affinity-and-anti-affinity)

## 05. TopologySpreadConstraints

# 04. 특정 노드의 스케쥴링 제한

## 01. Taint / toleration

Taint는 더럽다는 뜻이다. 더러우니까 Pod를 실행시킬 수 없다. 

반면 toleration은 여기가 더러워도 버틸 수 있으니까(tolerate) Pod를 실행시키겠단 뜻이다.

### 01. Taint 설정

<aside>
💡 **# kubectl taint node {노드명} {키}={값}:NoSchedule**

</aside>

```json
**# kubectl get node**
NAME          STATUS   ROLES           AGE     VERSION
k8s-master    Ready    control-plane   3d22h   v1.24.1
k8s-worker1   Ready    <none>          3d22h   v1.24.1
k8s-worker2   Ready    <none>          3d22h   v1.24.1

**# kubectl taint node k8s-worker1 node-taint=taint-on:NoSchedule**
node/k8s-worker1 tainted

**# kubectl describe node k8s-worker1 | grep NoSchedule**
**Taints:             node-taint=taint-on:NoSchedule**

**## Taint 적용 테스트**
**# kubectl create deploy taint-test --image=nginx --replicas=5**
deployment.apps/taint-test created

# **kubectl get pod -o wide**
NAME                          READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
taint-test-7dd4467485-288cn   1/1     Running   0          30s   10.110.126.12   **k8s-worker2**   <none>           <none>
taint-test-7dd4467485-5mnlv   1/1     Running   0          30s   10.110.126.15   **k8s-worker2**   <none>           <none>
taint-test-7dd4467485-bmqwj   1/1     Running   0          30s   10.110.126.19   **k8s-worker2**   <none>           <none>
taint-test-7dd4467485-fvl8g   1/1     Running   0          30s   10.110.126.17   **k8s-worker2**   <none>           <none>
taint-test-7dd4467485-llpzl   1/1     Running   0          30s   10.110.126.18   **k8s-worker2**   <none>           <none>
```

### 02. toleration 설정

toleration은 yaml 파일을 통해 수정해야 한다.

- **toleration 구문**
    - **키 : 값 동일**
        
        operator를 Equal로 설정한 뒤, 정확한 value를 지정해줘야 한다.
        
        ```yaml
        tolerations:
          - key: node-taint
            **operator: Equal
            value: taint-on**
            effect: NoSchedule
        ```
        
    - **키의 존재 여부**
        
        operator를 Exists로 설정하면, 해당 키가 존재만 해도 toleration이 적용된다.
        
        ```yaml
        tolerations:
          - key: node-taint
            **operator: Exists**
            effect: NoSchedule
        ```
        

**Yaml 예시**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: taint-toleration-test
  name: taint-toleration-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: taint-toleration-test
  template:
    metadata:
      labels:
        app: taint-toleration-test
    spec:
      containers:
      - image: nginx
        name: nginx
      **tolerations:
      - key: node-taint
        operator: Equal
        value: taint-on
        effect: NoSchedule**
```

**적용 결과**

```json
**# kubectl apply -f taint-toleration-test.yaml** 
deployment.apps/taint-toleration-test created

**# kubectl get pod -o wide**
NOMINATED NODE   READINESS GATES
taint-toleration-test-688dd8b89-b6kcg   1/1     Running   0          21s   10.100.194.126   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-m9xlw   1/1     Running   0          21s   10.100.194.123   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-pkktn   1/1     Running   0          21s   10.110.126.24    **k8s-worker2**   <none>           <none>
taint-toleration-test-688dd8b89-xf5lx   1/1     Running   0          21s   10.100.194.122   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-z76f8   1/1     Running   0          21s   10.110.126.27    **k8s-worker2**   <none>           <none>
```

### 03. Taint 해제

<aside>
💡 **# kubectl taint node {node명} {키}-**

</aside>

```json
**# kubectl taint node k8s-worker1 node-taint-**
node/k8s-worker1 untainted

**# kubectl describe node k8s-worker1 | grep NoSchedule**
```

## 02. cordon / uncordon

cordon은 저지선이라는 뜻이다. 특정 노드에 임시로 Pod가 실행되지 않도록 일종의 차단벽을 설정한다. Node의 PM이나 Node 상의 OS 작업, kubelet 업그레이드 등의 작업을 할 때 사용한다.

### 01. cordon 설정

<aside>
💡 **# kubectl cordon {노드 명}**

</aside>

```json
**# kubectl cordon k8s-worker1**
node/k8s-worker1 cordoned

# **kubectl get nodes**
NAME          STATUS                     ROLES           AGE     VERSION
k8s-master    Ready                      control-plane   3d22h   v1.24.1
k8s-worker1   Ready**,SchedulingDisabled**   <none>          3d22h   v1.24.1
k8s-worker2   Ready                      <none>          3d22h   v1.24.1
```

cordon을 설정하면 해당 노드는 자동으로 Taint 설정이 추가된다. 이때 키는 node.kubernetes.io/unschedulabe이 되고 따로 값은 지정되지 않는다. 

```json
**# kubectl describe nodes k8s-worker1 | grep Taint**
Taints:             node.kubernetes.io/unschedulable:NoSchedule
**# kubectl get nodes k8s-worker1 -o jsonpath='{.spec.taints}' | jq .**
[
  {
    "effect": "NoSchedule",
    "key": "node.kubernetes.io/unschedulable",
    "timeAdded": "2022-09-30T08:15:36Z"
  }
]
```

주의해야할 것이 있는데 cordon을 설정한다 하더라도 기존에 실행중인 pod들이 다른 node로 이동하지 않는다. 

```json
**## 작업 전 Pod 상태**
**# kubectl get pod -o wide**
NAME                                    READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
taint-toleration-test-688dd8b89-b6kcg   1/1     Running   0          8m34s   10.100.194.126   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-m9xlw   1/1     Running   0          8m34s   10.100.194.123   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-pkktn   1/1     Running   0          8m34s   10.110.126.24    k8s-worker2   <none>           <none>
taint-toleration-test-688dd8b89-xf5lx   1/1     Running   0          8m34s   10.100.194.122   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-z76f8   1/1     Running   0          8m34s   10.110.126.27    k8s-worker2   <none>           <none>

**## cordon 실행
# kubectl cordon k8s-worker1**
node/k8s-worker1 cordoned

****# **kubectl get nodes**
NAME          STATUS                     ROLES           AGE     VERSION
k8s-master    Ready                      control-plane   3d22h   v1.24.1
k8s-worker1   Ready**,SchedulingDisabled**   <none>          3d22h   v1.24.1
k8s-worker2   Ready    

**## 작업 후 Pod 상태
# kubectl get pod -o wide**
NAME                                    READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
taint-toleration-test-688dd8b89-b6kcg   1/1     Running   0          8m34s   10.100.194.126   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-m9xlw   1/1     Running   0          8m34s   10.100.194.123   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-pkktn   1/1     Running   0          8m34s   10.110.126.24    k8s-worker2   <none>           <none>
taint-toleration-test-688dd8b89-xf5lx   1/1     Running   0          8m34s   10.100.194.122   **k8s-worker1**   <none>           <none>
taint-toleration-test-688dd8b89-z76f8   1/1     Running   0          8m34s   10.110.126.27    k8s-worker2   <none>           <none>
```

### 02. uncordon 설정

<aside>
💡 **# kubectl uncordon {노드명}**

</aside>

```json
# **kubectl get nodes**
NAME          STATUS                     ROLES           AGE     VERSION
k8s-master    Ready                      control-plane   3d22h   v1.24.1
k8s-worker1   Ready**,SchedulingDisabled**   <none>          3d22h   v1.24.1
k8s-worker2   Ready    

**# kubectl uncordon k8s-worker1**
node/k8s-worker1 uncordoned

# **kubectl get nodes**
NAME          STATUS   ROLES           AGE     VERSION
k8s-master    Ready    control-plane   3d22h   v1.24.1
k8s-worker1   Ready    <none>          3d22h   v1.24.1
k8s-worker2   Ready    <none>          3d22h   v1.24.1
```

## 03. Drain

drain은 배수하다란 뜻이다. 하수구에 물 빠지듯이 특정 노드에서 실행중인 모든 pod를 다른 node로 옮기겠다는 뜻이다.

**단,** **deployment나 daemonset 등으로 관리되지 않는 독립적인 Pod들은 그대로 삭제되니 주의해야 한다.**

### 01. Drain 사용

<aside>
💡 **# kubectl drain {노드명}**

</aside>

위의 명령어는 기본형이다. 만약 drain을 시도하는 노드에서 개별 pod나 daemonset, emptydir을 사용하는 Pod가 있다면 에러가 발생하면서 drain이 되지 않는다.

```json
**# kubectl drain k8s-worker1**
node/k8s-worker1 cordoned
error: unable to drain node "k8s-worker1" due to error:[cannot delete Pods with local storage (use --delete-emptydir-data to override): default/emptdir-sidecar-pod, cannot delete Pods declare no controller (use --force to override): echo/nginx, echo/nginx-80, my-app/curly, my-app/test-ingress, my-app/wget, cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/calico-node-sqbl2, kube-system/kube-proxy-fzqph], continuing command...
There are pending nodes to be drained:
 k8s-worker1
**cannot delete Pods with local storage (use --delete-emptydir-data to override): default/emptdir-sidecar-pod   <- emptDir 사용때문에 삭제 불가**
**cannot delete Pods declare no controller (use --force to override): echo/nginx, echo/nginx-80, my-app/curly, my-app/test-ingress, my-app/wget <- 단일 Pod 때문에 삭제 불가**
**cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/calico-node-sqbl2, kube-system/kube-proxy-fzqph <- DaemonSet 때문에 삭제 불가**
```

따라서 아래와 같이 일부 옵션을 추가해서 진행해야한다.

<aside>
💡 **# kubectl drain {노드명} --ignore-daemonsets --delete-emptydir-data --force**

</aside>

- --ignore-daemonsets : 데몬셋으로 실행된 Pod를 무시한다.
- --delete-emptydir-data : emptyDir로 선언된 볼륨의 데이터를 삭제한다.
- --force : 단일 Pod를 삭제한다.

```json
**# kubectl drain k8s-worker1 --ignore-daemonsets --delete-emptydir-data --force** 
**node/k8s-worker1 already cordoned <- 해당 노드를 cordon 처리**
**WARNING: deleting Pods that declare no controller: default/emptdir-sidecar-pod, echo/nginx, echo/nginx-80, my-app/curly, my-app/test-ingress, my-app/wget; ignoring DaemonSet-managed Pods: kube-system/calico-node-sqbl2, kube-system/kube-proxy-fzqph <- 단일 Pod는 삭제되지만 DaemonSet Pod는 무시된다는 메시지**
evicting pod my-app/wget
evicting pod default/emptdir-sidecar-pod
evicting pod default/taint-toleration-test-688dd8b89-m9xlw
evicting pod default/taint-toleration-test-688dd8b89-xf5lx
evicting pod echo/nginx-80
evicting pod echo/nginx
evicting pod my-app/curly
evicting pod my-app/test-ingress
evicting pod default/taint-toleration-test-688dd8b89-b6kcg
pod/curly evicted
pod/taint-toleration-test-688dd8b89-xf5lx evicted
pod/emptdir-sidecar-pod evicted
pod/taint-toleration-test-688dd8b89-m9xlw evicted
pod/nginx-80 evicted
pod/taint-toleration-test-688dd8b89-b6kcg evicted
pod/nginx evicted
pod/test-ingress evicted
pod/wget evicted
node/k8s-worker1 drained

**# kubectl describe node k8s-worker1
(전략)
## DaemonSet이 관리하는 Pod만 실행중**
Non-terminated Pods:          (2 in total)
  Namespace                   Name                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                 ------------  ----------  ---------------  -------------  ---
  **kube-system                 calico-node-sqbl2**    250m (6%)     0 (0%)      0 (0%)           0 (0%)         3d23h
  **kube-system                 kube-proxy-fzqph**     0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d23h
```
## 03. Drain

drain은 배수하다란 뜻이다. 하수구에 물 빠지듯이 특정 노드에서 실행중인 모든 pod를 다른 node로 옮기겠다는 뜻이다.

**단,** **deployment나 daemonset 등으로 관리되지 않는 독립적인 Pod들은 그대로 삭제되니 주의해야 한다.**

### 01. Drain 사용

<aside>
💡 **# kubectl drain {노드명}**

</aside>

위의 명령어는 기본형이다. 만약 drain을 시도하는 노드에서 개별 pod나 daemonset, emptydir을 사용하는 Pod가 있다면 에러가 발생하면서 drain이 되지 않는다.

```json
**# kubectl drain k8s-worker1**
node/k8s-worker1 cordoned
error: unable to drain node "k8s-worker1" due to error:[cannot delete Pods with local storage (use --delete-emptydir-data to override): default/emptdir-sidecar-pod, cannot delete Pods declare no controller (use --force to override): echo/nginx, echo/nginx-80, my-app/curly, my-app/test-ingress, my-app/wget, cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/calico-node-sqbl2, kube-system/kube-proxy-fzqph], continuing command...
There are pending nodes to be drained:
 k8s-worker1
**cannot delete Pods with local storage (use --delete-emptydir-data to override): default/emptdir-sidecar-pod   <- emptDir 사용때문에 삭제 불가**
**cannot delete Pods declare no controller (use --force to override): echo/nginx, echo/nginx-80, my-app/curly, my-app/test-ingress, my-app/wget <- 단일 Pod 때문에 삭제 불가**
**cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/calico-node-sqbl2, kube-system/kube-proxy-fzqph <- DaemonSet 때문에 삭제 불가**
```

따라서 아래와 같이 일부 옵션을 추가해서 진행해야한다.

<aside>
💡 **# kubectl drain {노드명} --ignore-daemonsets --delete-emptydir-data --force**

</aside>

- --ignore-daemonsets : 데몬셋으로 실행된 Pod를 무시한다.
- --delete-emptydir-data : emptyDir로 선언된 볼륨의 데이터를 삭제한다.
- --force : 단일 Pod를 삭제한다.

```json
**# kubectl drain k8s-worker1 --ignore-daemonsets --delete-emptydir-data --force** 
**node/k8s-worker1 already cordoned <- 해당 노드를 cordon 처리**
**WARNING: deleting Pods that declare no controller: default/emptdir-sidecar-pod, echo/nginx, echo/nginx-80, my-app/curly, my-app/test-ingress, my-app/wget; ignoring DaemonSet-managed Pods: kube-system/calico-node-sqbl2, kube-system/kube-proxy-fzqph <- 단일 Pod는 삭제되지만 DaemonSet Pod는 무시된다는 메시지**
evicting pod my-app/wget
evicting pod default/emptdir-sidecar-pod
evicting pod default/taint-toleration-test-688dd8b89-m9xlw
evicting pod default/taint-toleration-test-688dd8b89-xf5lx
evicting pod echo/nginx-80
evicting pod echo/nginx
evicting pod my-app/curly
evicting pod my-app/test-ingress
evicting pod default/taint-toleration-test-688dd8b89-b6kcg
pod/curly evicted
pod/taint-toleration-test-688dd8b89-xf5lx evicted
pod/emptdir-sidecar-pod evicted
pod/taint-toleration-test-688dd8b89-m9xlw evicted
pod/nginx-80 evicted
pod/taint-toleration-test-688dd8b89-b6kcg evicted
pod/nginx evicted
pod/test-ingress evicted
pod/wget evicted
node/k8s-worker1 drained

**# kubectl describe node k8s-worker1
(전략)
## DaemonSet이 관리하는 Pod만 실행중**
Non-terminated Pods:          (2 in total)
  Namespace                   Name                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                 ------------  ----------  ---------------  -------------  ---
  **kube-system                 calico-node-sqbl2**    250m (6%)     0 (0%)      0 (0%)           0 (0%)         3d23h
  **kube-system                 kube-proxy-fzqph**     0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d23h
```

### 02. Drain 종료

Drain은 우선 Node를 Cordon 설정한 뒤 모든 Pod들을 옮기는 명령어다. Drain을 종료한다는 뜻은 결국 uncordon 설정한다는 뜻과 같다.

<aside>
💡 **# kubectl uncordon {노드명}**

</aside>

```json
# **kubectl get nodes**
NAME          STATUS                     ROLES           AGE     VERSION
k8s-master    Ready                      control-plane   3d22h   v1.24.1
k8s-worker1   Ready**,SchedulingDisabled**   <none>          3d22h   v1.24.1
k8s-worker2   Ready    

**# kubectl uncordon k8s-worker1**
node/k8s-worker1 uncordoned

# **kubectl get nodes**
NAME          STATUS   ROLES           AGE     VERSION
k8s-master    Ready    control-plane   3d22h   v1.24.1
k8s-worker1   Ready    <none>          3d22h   v1.24.1
k8s-worker2   Ready    <none>          3d22h   v1.24.1
```
