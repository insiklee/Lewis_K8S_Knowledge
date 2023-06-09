# K8s Daemonset 관리

# 01. DaemonSet이란?

DaemonSet은 클러스터에 포함된 모든 노드에서 동일한 Pod를 실행시키기 위한 리소스다.

# 02. DaemonSet 생성

## 01. DaemonSet Yaml

DasemonSet의 Yaml 형식은 Deploy와 거의 비슷하다. Deploy를 위한 Yaml에서 Kind만 바꿔주고 .spec.replicas만 제거해주면 될 정도다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-test
spec:
  selector:
    matchLabels:
      app: daemonset-test
  template:
    metadata:
      labels:
        app: daemonset-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
```

```json
**# kubectl apply -f daemonset-test.yaml** 
daemonset.apps/daemonset-test created

**# kubectl get ds**
NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-test   2         2         2       2            2           <none>          11s

# **kubectl get pod -o wide**
NAME                         READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
daemonset-test-57mmg         1/1     Running   0          14s     10.100.194.81    **k8s-worker1**   <none>           <none>
daemonset-test-bn28d         1/1     Running   0          14s     10.110.126.33    **k8s-worker2**   <none>           <none>
```

## 02. Taint된 노드(master)에 DaemonSet 적용

마스터 노드는 기본적으로 Taint가 적용되어서 NoScheduled 상태다. 다만, Calico와 같으 네트워크 플러그인이나, 프로메테우스같이 노드 모니터링 솔루션은 이런 마스터 노드에도 실행될 필요가 있다. 이때 tolerations를 사용하면 Taint 된 노드에서도 DaemonSet이 적용된다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-test2
spec:
****  selector:
    matchLabels:
      app: daemonset-test2
  template:
    metadata:
      labels:
        app: daemonset-test2
    spec:
      **tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule**
      containers:
      - name: nginx
        image: nginx:1.16.1
```

```json
**# kubectl apply -f daemonset-test2.yaml** 
daemonset.apps/daemonset-test2 created

**# kubectl get ds**
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-test    2         2         2       2            2           <none>          19m
**daemonset-test2   3         3         3       3            3           <none>          7m13s

# kubectl get pod -l app=daemonset-test2 -o wide**
NAME                    READY   STATUS    RESTARTS   AGE     IP              NODE          NOMINATED NODE   READINESS GATES
daemonset-test2-5tmvb   1/1     Running   0          8m33s   10.108.82.194   **k8s-master**    <none>           <none>
daemonset-test2-bc8zb   1/1     Running   0          8m33s   10.100.194.82   k8s-worker1   <none>           <none>
daemonset-test2-hjr97   1/1     Running   0          8m33s   10.110.126.34   k8s-worker2   <none>           <none>
```

## 03. 특정 노드만 선택하여 DaemonSet 생성

.spec.template.spec.nodeSelector를 적용하면 매칭되는 라벨이 있는 모든 노드에서 파드를 생성한다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-test3
spec:
****  selector:
    matchLabels:
      app: daemonset-test3
  template:
    metadata:
      labels:
        app: daemonset-test3
    spec:
      **nodeSelector:
        ds: test**
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: nginx
        image: nginx:1.16.1
```

```json

**# kubectl get nodes -l ds=test --show-labels**
NAME          STATUS   ROLES           AGE     VERSION   LABELS
k8s-master    Ready    control-plane   3d19h   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,**ds=test**,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-worker1   Ready    <none>          3d19h   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk=ssd**,ds=test**,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-worker1,kubernetes.io/os=linux

**# kubectl apply -f daemonset-test3.yaml** 
daemonset.apps/daemonset-test3 created

**# kubectl get ds**
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-test    2         2         2       2            2           <none>          26m
daemonset-test2   3         3         3       3            3           <none>          13m
**daemonset-test3   2         2         2       2            2           ds=test         23s

# kubectl get pod -l app=daemonset-test3 -o wide**
NAME                    READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
daemonset-test3-cvc6c   1/1     Running   0          41s   10.100.194.90   k8s-worker1   <none>           <none>
daemonset-test3-fmgxm   1/1     Running   0          41s   10.108.82.195   k8s-master    <none>           <none>
```
