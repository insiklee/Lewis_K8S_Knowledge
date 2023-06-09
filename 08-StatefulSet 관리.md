# K8s StatefulSet 관리

[StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

# 01. StatefulSet이란?

## 01. StatefulSet 개요

상태기반의 어플리케이션을 관리하기 위한 리소스다.  Deployment와는 달리 생성된 Pod의 **고유성**을 유지하기 위해 사용한다.

## 02. StatefulSet의 특징

### 01. Pod의 고유성

StatefulSet으로 생성된 Pod는 고유한 식별자를 지닌다. 이런 식별자는 API server의 statefulset 컨트롤러에 의해서 특정한 hash값을 부여받는다.

```json
**# kubectl get sts**
NAME             READY   AGE
**nginx-stateful**   3/3     145m

**# kubectl get pod**
 NAME                           READY   STATUS    RESTARTS   AGE
nginx-stateful-0               1/1     Running   0          15h
nginx-stateful-1               1/1     Running   0          17h
nginx-stateful-2               1/1     Running   0          17h

**## statefulset 파드들의 라벨 확인**
**# kubectl get pod -l app=nginx-stateful -o jsonpath='{range .items[*]}{.metadata.labels}{"\n"}' | jq** 
{
  "app": "nginx-stateful",
  "controller-revision-hash": "nginx-stateful-7db594cfbd",
  "statefulset.kubernetes.io/pod-name": "nginx-stateful-0"
}
{
  "app": "nginx-stateful",
  "controller-revision-hash": "nginx-stateful-7db594cfbd",
  "statefulset.kubernetes.io/pod-name": "nginx-stateful-1"
}
{
  "app": "nginx-stateful",
  "controller-revision-hash": "nginx-stateful-7db594cfbd",
  "statefulset.kubernetes.io/pod-name": "nginx-stateful-2"
}
```

### **controller-revision-hash**

StatefulSet의 업데이트 이력을 남기기 위한 값이다. StatefulSet 이름 뒤에 hash된 난수가 붙는데, 이 난수가 revision 값이다. statefulset의 내용을 업데이트하면  바뀐다.

- **revision 변경 확인**
    
    ```json
    **# kubectl set image sts nginx-stateful nginx=nginx:1.21.1**
    statefulset.apps/nginx-stateful image updated
    
    **# kubectl get pod -l app=nginx-stateful -o jsonpath='{range .items[*]}{.metadata.labels}{.spec.hostName}{"\n"}' | jq** 
    {
      "app": "nginx-stateful",
      "controller-revision-hash": "nginx-stateful-7db594cfbd",
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-0"
    }
    {
      "app": "nginx-stateful",
      "controller-revision-hash": "nginx-stateful-7db594cfbd",
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-1"
    }
    {
      "app": "nginx-stateful",
      "controller-revision-hash": **"nginx-stateful-6c7d474cc",   # revision 값이 바뀜**
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-2"
    }
    
    **# kubectl get pod -l app=nginx-stateful -o jsonpath='{range .items[*]}{.metadata.labels}{.spec.hostName}{"\n"}' | jq** 
    {
      "app": "nginx-stateful",
      "controller-revision-hash": "nginx-stateful-7db594cfbd",
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-0"
    }
    {
      "app": "nginx-stateful",
      "controller-revision-hash": **"nginx-stateful-6c7d474cc",   # revision 값이 바뀜**
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-1"
    }
    {
      "app": "nginx-stateful",
      "controller-revision-hash": **"nginx-stateful-6c7d474cc",   # revision 값이 바뀜**
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-2"
    }
    
    **# kubectl get pod -l app=nginx-stateful -o jsonpath='{range .items[*]}{.metadata.labels}{.spec.hostName}{"\n"}' | jq** 
    {
      "app": "nginx-stateful",
      "controller-revision-hash":  **"nginx-stateful-6c7d474cc",   # revision 값이 바뀜**
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-0"
    }
    {
      "app": "nginx-stateful",
      "controller-revision-hash": **"nginx-stateful-6c7d474cc",   # revision 값이 바뀜**
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-1"
    }
    {
      "app": "nginx-stateful",
      "controller-revision-hash": **"nginx-stateful-6c7d474cc",   # revision 값이 바뀜**
      "statefulset.kubernetes.io/pod-name": "nginx-stateful-2"
    }
    ```
    

### **statefulset.kuberenetes.io/pod-name**

Pod의 고유 식별자이자 pod의 이름이자, pod의 호스트네임이다. 

### 02. 네트워크 상태저장

**StatefulSet은 Service와 직접 연결되지 않는다.**

```json
**# kubectl get sts**
NAME             READY   AGE
**nginx-stateful**   3/3     105m

# **kubectl expose sts nginx-stateful --port=80 --target-port=80 --type=NodePort --name=nginx-stateful-svc**
**error: cannot expose a StatefulSet.apps**
```

이와 관련된 자세한 내용은 [03. StatefulSet 네트워크](https://www.notion.so/03-StatefulSet-50848da5439649719bc39b9128a36742?pvs=21) 에서 설명한다.

### 03. 디스크 상태 저장

StatefulSet에서 생성되는 Pod는 전부 각자의 역할을 지닌 독립적인 개체다. 그렇기 때문에 Deployment와 달리 동시에 하나의 PV에 연결하지 않고 각자 별도의 PV와 연결하여 독자저인 데이터를 생성하고 유지할 필요가 있다. 

이때 **volumeClaimTemplates**을 사용하여 각가의 Pod가 별도의 PVC와 연결하도록 설정할 수 있다.

자세한 내용은 [04. StatefulSet 스토리지](https://www.notion.so/04-StatefulSet-78621bdd62714dc5a838f01ff0a14042?pvs=21) 에서 다룬다.

## 03. Deployment와의 차이

### 01. ReplicaSet 유무

Deployment는 ReplicaSet를 직접 생성한 뒤, Pod의 관리를 ReplicaSet에게 양도한다.

StatefulSet은 ReplicaSet을 생성하지 않고 Pod의 개수와 관리를 직접 한다. 

이 때문에 **deployment와는 달리 Pod 이름 뒤에 난수가 붙지 않고 깔끔하게 넘버링 된다.**

```json
**# kubectl get pod**
NAME                           READY   STATUS    RESTARTS   AGE
nginx-stateful-0               1/1     Running   0          10m
nginx-stateful-1               1/1     Running   0          146m
nginx-stateful-2               1/1     Running   0          145m  ## statefulset으로 생성된 pod
test-deploy-584c765db4-2wv4w   1/1     Running   0          63m
test-deploy-584c765db4-724td   1/1     Running   0          63m
test-deploy-584c765db4-vwskk   1/1     Running   0          102m  ## Deployment로 생성된 pod 
```

### 02. Pod의 상태

Deployment는 기존의 Pod가 삭제되고 새 Pod가 생성될때 연속성이 없는 완전히 새로운 Pod가 생성된다.

반면 StetefulSet에서 생성된 Pod는 자신의 정체성을 잊지않고 삭제 전의 상태를 유지한다.

### 03. 서비스 연결 방식의 차이

서비스와 연결되는 방식에도 차이가 있다.

Deployment는 service가 스스로 요청을 받은 뒤 각 Pod로 로드밸런싱을 해주는 반면

StatefulSet에서는 서비스의 역할은 축소되고 각 Pod가 직접 네트워크와 연결된다. 

# 02. StatefulSet 생성

## 01. YAML로 생성

StatefulSet은 명령어로 만들 수 없고, YAML을 통해서 생성해야 한다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx-stateful
  name: nginx-stateful
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful
  template:
    metadata:
      labels:
        app: nginx-stateful
    spec:
      containers:
      - image: nginx
        name: nginx
```

```json
**# kubectl apply -f nginx-stateful.yaml** 
statefulset.apps/nginx-stateful created

**# kubectl get sts**
NAME             READY   AGE
nginx-stateful   3/3     14s

**# kubectl get pod -l app=nginx-stateful**
NAME               READY   STATUS    RESTARTS   AGE
nginx-stateful-0   1/1     Running   0          26s
nginx-stateful-1   1/1     Running   0          22s
nginx-stateful-2   1/1     Running   0          19s
```

# 03. StatefulSet 네트워크

## 01. StatefulSet에서의 네트워크

StatefulSet의 목적은 Pod가 아무리 재시작하더라도 그 고유성을 유지하는데 있다. 여기에는 네트워크 정보 역시 변경되지 않아야 한다는 뜻이다. 즉, CoreDNS에서 관리하는 DNS 주소가 유지되어야 한다. 이는 기존 Deployment가 서비스 배포되는 방식과 조금 다르다

### 01. Deployment에서 생성된 Pod의 도메인주소

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9bcce899-49b4-4f78-96bf-1b4af96f8876/Untitled.png)

기존 Deployment에서 Service 연결은 서비스 리소스의 ClusterIP가 각 Pod가 지닌 IP로 로드밸런싱 하면서  접속한다.

아래 예시를 보자.

```json
**# kubectl get deploy,svc,po -l app=test-deploy**
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/test-deploy   3/3     3            3           22h   nginx        nginx    app=test-deploy

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/test-deploy   ClusterIP   **10.110.169.181**   <none>        80/TCP    52s   app=test-deploy

NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
pod/test-deploy-584c765db4-2wv4w   1/1     Running   0          21h   **10.110.126.40**   k8s-worker2   <none>           <none>
pod/test-deploy-584c765db4-724td   1/1     Running   0          21h   **10.100.194.79**   k8s-worker1   <none>           <none>
pod/test-deploy-584c765db4-vwskk   1/1     Running   0          22h   **10.110.126.39**   k8s-worker2   <none>           <none>
```

외부에서 Pod로 접속하기 위해 우리는 우선 Service의 ClusterIP로 접속을 시도해야한다. 그 후 서비스는 CoreDNS에 의해 만들어진 도메인 주소로 로드밸런싱을 하여 여러 Pod에 접속할 수 있도록 도와준다.

이때 각 Pod의 도메인 주소는 아래와 같다.

```json
**# kubectl run dnsquery --image=busybox --rm -it --restart=Never -- nslookup 10.100.194.79**
If you don't see a command prompt, try pressing enter.
Error attaching, falling back to logs: unable to upgrade connection: container dnsquery not found in pod dnsquery_default
Server:		10.96.0.10
Address:	10.96.0.10:53

79.194.100.10.in-addr.arpa	name = **10-100-194-79.test-deploy.default.svc.cluster.local**

pod "dnsquery" deleted

**# kubectl run dnsquery --image=busybox --rm -it --restart=Never -- nslookup 10.110.126.39**
Server:		10.96.0.10
Address:	10.96.0.10:53

39.126.110.10.in-addr.arpa	name = **10-110-126-39.test-deploy.default.svc.cluster.local**

pod "dnsquery" deleted

**# kubectl run dnsquery --image=busybox --rm -it --restart=Never -- nslookup 10.110.126.40**
Server:		10.96.0.10
Address:	10.96.0.10:53

40.126.110.10.in-addr.arpa	name = **10-110-126-40.test-deploy.default.svc.cluster.local**

pod "dnsquery" deleted
```

각 Pod의 DNS 주소를 역방향으로 resolving한 결과다. 도메인명에 각 파드의 IP 정보가 포함된 것을 확인할 수 있다.

### 02. StatefulSet의 서비스 네트워크

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/116b8b79-890e-408e-87db-80401ab1a18b/Untitled.png)

StatefulSet에서는 Service가 로드밸런싱 역할을 하지 않는다. 서비스는 ClusterIP를 지니지 않는다. 다만 CoreDNS가 DNS Record를 만들기 위한 마중물 정도의 역할만 한다. 이렇게 ClusterIP를 지니지 않고 Selector를 통해서 DNS record 생성용으로만 사용하는 서비스를 Headless Service라고 한다.

이런식의 연결 방식은 Deployment 방식와 아주 큰 차이를 갖게 된다. Deployment에서 생성된 모든 Pod는 로드밸런싱을 통해서 균등하게 패킷을 전달받기 때문에 모든 Pod가 동일한 역할을 수행한다. 반면 **StatefulSet은 각 Pod가 독립적인 네트워크 통신을 하기 때문에 Pod 마다 서로 다른 역할을 수행할 수 있게 된다.** 

## 02. StatefulSet을 위한 Headless Service 생성

### 01. 명령어로 생성

<aside>
💡 **# kubectl create service clusterip {서비스명 - 이 상황에선 StatefulSet 명} --tcp={from-port 번호}:{to-port 번호} --clusterip=None**

</aside>

```json
# kubectl create service clusterip nginx-stateful --tcp=80:80 --clusterip=None
service/nginx-stateful created

**# kubectl get svc nginx-stateful**
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx-stateful   ClusterIP   None         <none>        80/TCP    24s

**# kubectl get svc nginx-stateful -o jsonpath='{.spec.selector}{"\n"}'**
{"app":"nginx-stateful"}
```

Cluster-IP가 None으로 설정되어 있다. 그리고 selector가 자동으로 서비스명과 동일하게 생성된 것을 확인할 수 있다. 따라서 명령어로 StatefulSet 

### 02. YAML로 생성

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-stateful
  name: nginx-stateful
spec:
  **clusterIP: None**
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-stateful
  **type: ClusterIP**
```

다른 서비스 리소스를 만드는 법과 동일하지만 ClusetIP를 None으로 설정하는 것이 다르다.

## 03. StatefulSet과 Headless Service 연결

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx-stateful
  name: nginx-stateful
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful
  **serviceName: nginx-stateful**
  template:
    metadata:
      labels:
        app: nginx-stateful
    spec:
      containers:
      - image: nginx
        name: nginx
```

위에 내용과 같이 serviceName 항목을 추가하여 Headless Service 이름을 지정해준다. 이것만 해주면 된다. 참 쉽다.

## 04. StatefulSet으로 생성된 Pod의 Service 연결 확인

```json
**# kubectl get pod -l app=nginx-stateful -o wide**
NAME               READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
nginx-stateful-0   1/1     Running   0          27m   10.100.194.82   k8s-worker1   <none>           <none>
nginx-stateful-1   1/1     Running   0          27m   10.110.126.47   k8s-worker2   <none>           <none>
nginx-stateful-2   1/1     Running   0          27m   10.100.194.90   k8s-worker1   <none>           <none>

**# kubectl run dnsquery --image=busybox --rm -it --restart=Never -- nslookup 10.100.194.82**
Server:		10.96.0.10
Address:	10.96.0.10:53

82.194.100.10.in-addr.arpa	name = **nginx-stateful-0.nginx-stateful.default.svc.cluster.local**

pod "dnsquery" deleted

**# kubectl run dnsquery --image=busybox --rm -it --restart=Never -- nslookup 10.110.126.47**
Server:		10.96.0.10
Address:	10.96.0.10:53

47.126.110.10.in-addr.arpa	name = **nginx-stateful-1.nginx-stateful.default.svc.cluster.local**

pod "dnsquery" deleted

**# kubectl run dnsquery --image=busybox --rm -it --restart=Never -- nslookup 10.100.194.90**
Server:		10.96.0.10
Address:	10.96.0.10:53

90.194.100.10.in-addr.arpa	name = **nginx-stateful-2.nginx-stateful.default.svc.cluster.local**

pod "dnsquery" deleted
```

각 pod의 도메인에 ip가 없이 Pod명이 들어간 것을 확인할 수 있다. 이런 경우 Pod가 재생성되어 IP가 달라진다 하더라도 도메인이 그대로이기 때문에 네트워크 서비스에 연속성이 보장되며, 각 Pod마다 독립된 서비스를 제공할 수 있다.

# 04. StatefulSet 스토리지

## 01. Dynamic Persistent Volume

StatefulSet에서 생성되는 Pod는 전부 각자의 역할을 지닌 독립적인 개체다. 그렇기 때문에 Deployment와 달리 동시에 하나의 PV에 연결하지 않고 각자 별도의 PV와 연결하여 독자저인 데이터를 생성하고 유지할 필요가 있다.

이를 위해서 우선 StatefulSet은 Dynamic Persistent Volume이 우선 생성되어야 한다.

Dynamic Persistent Volume을 구성하기 위한 Storage Class 생성은 [03. Dynamic Persistent Volume 구성](https://www.notion.so/03-Dynamic-Persistent-Volume-159441cc787f459eb4f2707f4a798fe3?pvs=21)를 참고해보자.

## 02. volumeClaimTemplates

아래와 같이 YAML로 Headless Service와 StatefulSet을 정의하자.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-stateful2
  name: nginx-stateful2
spec:
  clusterIP: None
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-stateful2
  type: ClusterIP
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx-stateful2
  name: nginx-stateful2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful2
  serviceName: nginx-stateful2
  template:
    metadata:
      labels:
        app: nginx-stateful2
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nginx-stateful2-pvc
          mountPath: /usr/share/nginx/html
  **volumeClaimTemplates:
  - metadata:
      name: nginx-stateful2-pvc
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: sc-nfs
      resources:
        requests:
          storage: 2Gi**
```

volumeClaimTemplates는 deployment의 template와 만드는 방식이 동일하다. 그냥 인덴트 한번 더 추가하고 pvc를 yaml로 정의하면 된다. 단, storageClass를 반드시 정의해야 한다.

컨테이너에 volumeMounts 정책을 추가할때는 volumeClaimTemplates에서 정의한 이름과 mountPath만 정의한다. 그러면 나머지는 StatefulSet 컨트롤러가 알아서 나머지 요소들을 채워준다.

위 YAML을 실행한다.

```json
**# kubectl apply -f nginx-stateful2.yaml** 
service/nginx-stateful2 created
statefulset.apps/nginx-stateful2 created

**# kubectl get sts,svc,po,pvc -l app=nginx-stateful2**
NAME                               READY   AGE
statefulset.apps/nginx-stateful2   3/3     64s

NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/nginx-stateful2   ClusterIP   None         <none>        80/TCP    64s

NAME                    READY   STATUS    RESTARTS   AGE
pod/nginx-stateful2-0   1/1     Running   0          64s
pod/nginx-stateful2-1   1/1     Running   0          59s
pod/nginx-stateful2-2   1/1     Running   0          54s

NAME                                                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nginx-stateful2-pvc-nginx-stateful2-0   Bound    pvc-7ffabf9b-b35e-4e6b-bb52-cf5572e13d93   2Gi        RWO            sc-nfs         64s
persistentvolumeclaim/nginx-stateful2-pvc-nginx-stateful2-1   Bound    pvc-69f41388-2916-41de-b43d-c819ae62e243   2Gi        RWO            sc-nfs         59s
persistentvolumeclaim/nginx-stateful2-pvc-nginx-stateful2-2   Bound    pvc-a7e63014-d29f-4823-b125-9f8e2f507f04   2Gi        RWO            sc-nfs         54s
```

각 pod에 할당되는 pvc가 자동으로 생성된 것을 확인할 수 있다.
