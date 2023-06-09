# K8s Service 관리

# 01. Service란?

[서비스](https://kubernetes.io/ko/docs/concepts/services-networking/service/)

모든 Pod는 CNI에 속한 고유 IP 주소를 지니게 된다. 다만 파드가 새로 생성될때마다 IP가 달라진다. 이 경우 파드간 통신시, 또는 외부에서 파드로 접근할때 연속적인 서비스에 어려움이 생긴다.

이런 문제를 해결하기 위해서 사용하는 것이 Service 리소스다.

### 02. IPTables에서 서비스 동작 원리

CNI는 커널의 netfilter 모듈을 통해서 작동한다. 리눅스 커널은 netfilter로 IPTables를 사용한다. 

[Kubernetes Service Proxy](https://ssup2.github.io/theory_analysis/Kubernetes_Service_Proxy/)

어떤 선배분이 IPtables에서 서비스 사용시 패킷의 NAT 방식을 설명해줬다. 

[Kubernetes's IPTables Chains Are Not API](https://kubernetes.io/blog/2022/09/07/iptables-chains-not-api/)

# 02. Service Type의 종류

## 01. ClusterIP

아무런 지정을 하지 않을 경우 Default로 생성된다. ClusterIP로 타입을 지정할 경우 kube-proxy를 사용하지 않을 경우 죽었다 깨어나도 해당 서비스로 접근할 수 없다.

### **ClusterIP의 yaml 예시**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-svc
spec:
  selector:
    app: cipapp
  ports:
  - port: 9000
    targetPort: 8080
  **type: ClusterIP**
```

### **명령어로 ClusterIP 타입 서비스 생성**

expose 명령어로 서비스를 생성하면 별도로 타입 지정을 안할 경우 자동으로 ClusterIP로 생성된다.

<aside>
💡 **# kubectl -n {네임스페이스명} expose deploy {디플로이명} --port {서비스포트번호} --target-port {컨테이너의포트번호}**

</aside>

```json
**# kubectl -n httpd-test expose deployment httpd-test --port 8080 --target-port 80**
service/httpd-test exposed

**# kubectl -n httpd-test get svc httpd-test -o yaml**
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2022-09-13T05:01:36Z"
  labels:
    app: httpd-test
  name: httpd-test
  namespace: httpd-test
  resourceVersion: "536838"
  uid: 2612d674-80c1-4455-a05e-bda9fcbbf64e
spec:
  clusterIP: 10.98.36.92
  clusterIPs:
  - 10.98.36.92
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  **- port: 8080
    protocol: TCP
    targetPort: 80**
  selector:
    app: httpd-test
  sessionAffinity: None
  **type: ClusterIP**
status:
  loadBalancer: {}
```

### kube-proxy(port-forward)로 **접속 테스트**

ClusterIP로 생성된 서비스로 어플리케이션에 접근하기 위해서는 kube-proxy를 사용해야한다. 방법은 아래와 같다.

<aside>
💡 **# kubectl -n {네임스페이스명} port-forward service {서비스명} {포트}:{서비스포트}**

</aside>

```json
**[session1] # kubectl -n httpd-test port-forward services/httpd-test 8001:8080**
Forwarding from 127.0.0.1:8001 -> 80
Forwarding from [::1]:8001 -> 80
```

port-foward 기능을 사용하면 명령어를 사용한 세션의 foreground를 가져가기 때문에 다른 명령어를 사용할 수 없다. 이에 접속 테스트를 위해 다른 세션을 열여서 확인한다.

```json
**[session2] # curl 127.0.0.1:8001**
<html><body><h1>It works!</h1></body></html>

**[session1] # kubectl -n httpd-test port-forward services/httpd-test 8001:8080**
Forwarding from 127.0.0.1:8001 -> 80
Forwarding from [::1]:8001 -> 80
**Handling connection for 8001**
```

다른 세션을 통해 접속이 가능한 것을 확인할 수 있으며,  port-forward를 한 세션에서는 연결 정보가 뜨는 것을 볼 수 있다.

다만 위와 같은 방법은 loopback 주소를 사용하기 때문에 로컬에서만 사용이 가능하다. 외부에서 연결을 하기 위해서는 --address 옵션을 추가해야한다.

<aside>
💡 **# kubectl -n {네임스페이스명} port-forward service {서비스명} --address {주소} {포트}:{서비스포트}**

</aside>

```json
**[master] # kubectl -n httpd-test port-forward services/httpd-test --address 192.168.122.10 8001:8080**
Forwarding from 192.168.122.10:8001 -> 80

**[worker] # curl 192.168.122.10:8001**
<html><body><h1>It works!</h1></body></html>
```

## 02. ClusterIP에서 **externalIP 사용**

ClusterIP에서 externalIP를 지정하면 kube-proxy를 사용하지 않고도  ClusterIP를 사용중인 서비스에 접근할 수 있다. 당연히 externalIP에는 파드가 올라갈 노드의 IP 주소를 입력해야 한다. 

### **externalIPs의 yaml 예시**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hi-devphp-svc
spec:
  selector:
    app: hi-devphp
  **externalIPs:
  - 192.168.56.101**
  ports:
  - port: 8811
    targetPort: 80
```

### **명령어로 externalIP 삽입 및 접속테스트**

<aside>
💡 **# kubectl -n {네임스페이스명} expose deploy {디플로이명} --external-ip {externalIP 주소} --port {서비스 포트번호} --target-port {컨테이너포트번호}**

</aside>

```json
**# kubectl -n httpd-test expose deploy httpd-test --external-ip 192.168.1.11 --port 8080 --target-port 80**
service/httpd-test exposed

**# kubectl -n httpd-test get svc**
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP    PORT(S)    AGE
httpd-test   ClusterIP   10.106.114.167   **192.168.1.11**   8080/TCP   11s

**# kubectl -n httpd-test get svc httpd-test -o yaml**
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2022-09-13T06:37:07Z"
  labels:
    app: httpd-test
  name: httpd-test
  namespace: httpd-test
  resourceVersion: "550583"
  uid: d7aa6a50-9645-4066-ae8e-64f8a7e8b607
spec:
  clusterIP: 10.106.114.167
  clusterIPs:
  - 10.106.114.167
  **externalIPs:
  - 192.168.1.11**
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    app: httpd-test
  sessionAffinity: None
  **type: ClusterIP**
status:
  loadBalancer: {}

**# curl 192.168.1.11:8080**
<html><body><h1>It works!</h1></body></html>
```

## 03. NodePort

L4기반의 로드밸런서로 이해하면 된다. **모든 노드에 공통된 포트를 열어놓고 해당 포트로 접근하게 하면 명시된 서비스로 접근할 수 있다.** nodeprt 번호는 30000~32767 중 랜덤한 값으로 정해지며, nodePort 옵션을 넣으면 원하는 포트넘버를 지정할 수 있다.

### Nodeport yaml 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mynode-svc
spec:
  selector:
    app: hi-mynode
  **type: NodePort**
  ports:
  - port: 8899
    targetPort: 8000
    **nodePort: 30000**
```

.spec.ports 하위에 nodePort라는 항목이 생성된다. NodePort는 모든 노드에 동시에 열리는 포트로 서비스포트 8899로 자동으로 포트포워딩 되는 포트다. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d4a54e6b-9830-42ff-a6cb-b206d594792e/Untitled.png)

### 명령어로 NodePort 타입 서비스 생성

<aside>
💡 **# kubectl -n {네임스페이스명} expose deploy {디플로이명} --type NodePort  --port {서비스 포트번호} --target-port {컨테이너포트번호}**

</aside>

```json
**# kubectl -n httpd-test expose deploy httpd-test --type NodePort --port 80 --target-port 80**
service/httpd-test exposed

**# kubectl -n httpd-test get svc**
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
httpd-test   NodePort   10.97.238.231   <none>        80:**31840**/TCP   7s

**# kubectl -n httpd-test get svc httpd-test -o yaml**
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2022-09-13T07:07:19Z"
  labels:
    app: httpd-test
  name: httpd-test
  namespace: httpd-test
  resourceVersion: "554950"
  uid: 914a6dd5-057c-4e19-ad6d-e2da02c1180f
spec:
  clusterIP: 10.97.238.231
  clusterIPs:
  - 10.97.238.231
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  **- nodePort: 31840
    port: 80
    protocol: TCP
    targetPort: 80**
  selector:
    app: httpd-test
  sessionAffinity: None
  **type: NodePort**
status:
  loadBalancer: {}

**# curl master:31840**
<html><body><h1>It works!</h1></body></html>
**# curl worker1:31840**
<html><body><h1>It works!</h1></body></html>
**# curl worker2:31840**
<html><body><h1>It works!</h1></body></html>
```

## 04. LoadBalancer

로드밸런서를 구현할때 사용한다. 퍼블릭 클라우드에서는 각 클라우드에서 제공하는 로드밸런서 서비스를 사용할 수 있고, BareMetal에서는 MetalLB를 사용하여 구현한다.

### MetalLB 구축

### LoadBalancer 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.105.196.75
  clusterIPs:
  - 10.105.196.75
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 31680
    port: 9090
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  **type: LoadBalancer**
```

### 명령어로 LoadBalancer 타입 서비스 생성

<aside>
💡 **# kubectl -n {네임스페이스명} expose deploy {디플로이명} --type LoadBalancer  --port {서비스 포트번호} --target-port {컨테이너포트번호}**

</aside>

```yaml
**# kubectl -n httpd-test expose deployment httpd-test --type LoadBalancer --port 8080 --target-port 80**
service/httpd-test exposed

**# kubectl -n httpd-test get svc**
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
httpd-test   LoadBalancer   10.109.125.5   192.168.122.11   8080:30892/TCP   6s

**# curl 192.168.122.11:30892**
<html><body><h1>It works!</h1></body></html>

```

## 05. ExternalName

컨테이너의 IP주소가 DNS 서버에 등록되어 있을 경우 FQDN으로 연결을 지원하는 방식이다.  CoreDNS나 Ingress에서 만든 클러스터 전용 도메인이 아니라 진짜 네임서버에 등록된 도메인 주소를 사용해야한다.
