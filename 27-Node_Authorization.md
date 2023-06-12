# K8s Node Authorization

[Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/)

# 01. Node Authorization이란?

## 01. 개요

kubelet에 의한 API 요청에 대해 인가하기 위한 특수한 목적을 가진 모드다.

## 02. Node Authorization의 허용 권한 목록

### 01. 읽기 권한 관리

- service, endpoints, nodes, pod 등의 각종 리소스
- secret, configmaps, PV, PVC 등 kubelet의 노드와 결합되는 파드와 관련된 리소스

### 02. 쓰기 권한 관리

- node와 node의 상태
- pod와 pod의 상태
- events

### 03. 권한 관련

- TLS 부트스트랩을 위한 CSR API의 접근

## 03. TLS 부트스트랩

[TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)

worker 노드가 제대로 일하려면 kubelet을 통해서 API Server와 지속적으로 통신해야한다. 이때도 역시 TLS 통신으로 진행해야 하며, 이를 위한 kubelet의 TLS 인증서가 필요하다.

kubelet이 사용하는 인증서는 노드가 최초로 클러스터에 합류할때 생성된다. 이때 kubelet의 인증서를 생성하는 과정이 TLS 부트스트랩이다.

아래는 새로운 노드가 클러스터에 새로 합류할때 나오는 로그 메시지다.

```json
**# kubeadm join 192.168.122.10:6443 --token sfquq9.uipjl3fu512j8z4g --discovery-token-ca-cert-hash sha256:db460ad0310e47d6db152fef6c1f946314f74e5ef0e56cebaa48ccfe11be60ed**
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
**[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.**

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

kubelet이 시작하면서 TLS Bootstrap을 수행한다는 메시지가 나온다.

이후 CSR이 API Server에 전달되고, 그에 대한 응답을 받았다는 메시지 역시 확인할 수 있다. 

이후 /var/lib/kubelet/pki를 확인하면 TLS 부트스트랩으로 생성된 인증파일을 확인할 수 있다.

```json
**# ls -l /var/lib/kubelet/pki/**
total 12
**-rw------- 1 root root 1114 10월 24 17:04 kubelet-client-2022-10-24-17-04-57.pem
lrwxrwxrwx 1 root root   59 10월 24 17:04 kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2022-10-24-17-04-57.pem**
-rw-r--r-- 1 root root 2271 10월 24 17:04 kubelet.crt
-rw------- 1 root root 1679 10월 24 17:04 kubelet.key
```

위 kubelet-client-current.pem 파일에 kubelet의 인증서와 개인키가 담겨있다. 저 파일을 통해 kubelet이 API Server와 통신할 수 있다. 

# 02. Node Authorization 사용

## 01. authorization mode 활성화

apiserver 파드를 실행할때 컨테이너의 command에 --authorization-mode 옵션이 있다. 해당 옵션에 Node가 추가되어야 한다.

클러스터를 최초로 설치하고 별다른 설정을 안했다면 해당 mode는 자동으로 활성화되어 있다.

```json
**# kubectl -n kube-system get pod kube-apiserver-k8s-master -o jsonpath='{.spec.containers[0].command}' | jq**
[
  "kube-apiserver",
  "--advertise-address=192.168.122.10",
  "--allow-privileged=true",
 ** "--authorization-mode=Node,RBAC",**
  "--client-ca-file=/etc/kubernetes/pki/ca.crt",
  "--enable-admission-plugins=NodeRestriction",
  "--enable-bootstrap-token-auth=true",
  "--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt",
  "--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt",
  "--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key",
  "--etcd-servers=https://127.0.0.1:2379",
  "--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt",
  "--kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key",
  "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
  "--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt",
  "--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key",
  "--requestheader-allowed-names=front-proxy-client",
  "--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt",
  "--requestheader-extra-headers-prefix=X-Remote-Extra-",
  "--requestheader-group-headers=X-Remote-Group",
  "--requestheader-username-headers=X-Remote-User",
  "--secure-port=6443",
  "--service-account-issuer=https://kubernetes.default.svc.cluster.local",
  "--service-account-key-file=/etc/kubernetes/pki/sa.pub",
  "--service-account-signing-key-file=/etc/kubernetes/pki/sa.key",
  "--service-cluster-ip-range=10.96.0.0/12",
  "--tls-cert-file=/etc/kubernetes/pki/apiserver.crt",
  "--tls-private-key-file=/etc/kubernetes/pki/apiserver.key"
]
```

만약 kubelet에 쓰기 제한을 걸고 싶다면  --enabled-admission-plugins 옵션에 NodeRestriction이 추가되어야 한다.  그런데 이미 적용되어 있다.

```json
**# kubectl -n kube-system get pod kube-apiserver-k8s-master -o jsonpath='{.spec.containers[0].command}' | jq**
[
  "kube-apiserver",
  "--advertise-address=192.168.122.10",
  "--allow-privileged=true",
 **** "--authorization-mode=Node,RBAC",
  "--client-ca-file=/etc/kubernetes/pki/ca.crt",
  **"--enable-admission-plugins=NodeRestriction",**
  "--enable-bootstrap-token-auth=true",
  "--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt",
  "--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt",
  "--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key",
  "--etcd-servers=https://127.0.0.1:2379",
  "--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt",
  "--kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key",
  "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
  "--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt",
  "--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key",
  "--requestheader-allowed-names=front-proxy-client",
  "--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt",
  "--requestheader-extra-headers-prefix=X-Remote-Extra-",
  "--requestheader-group-headers=X-Remote-Group",
  "--requestheader-username-headers=X-Remote-User",
  "--secure-port=6443",
  "--service-account-issuer=https://kubernetes.default.svc.cluster.local",
  "--service-account-key-file=/etc/kubernetes/pki/sa.pub",
  "--service-account-signing-key-file=/etc/kubernetes/pki/sa.key",
  "--service-cluster-ip-range=10.96.0.0/12",
  "--tls-cert-file=/etc/kubernetes/pki/apiserver.crt",
  "--tls-private-key-file=/etc/kubernetes/pki/apiserver.key"
]
```

## 02. nodeRestriction

### 01. 개요

[Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)

nodeRestriction은 Authorization 이후 단계인 Admission Control 단계에서 실행된다. 특정 노드의 kubelet이 자기 자신의 node나 자신이 실행한 pod가 아닌 다른 노드의 설정이나 pod의 정보를 수정하는것을 제한한다. 

만약 클러스터의 특정 worker 노드를 탈취한 해킹범이 있을때, 해당 해킹범이 master 노드의 상태를 수정하거나, 다른 노드에서 실행중인 Pod를 삭제할 수 있을 경우, 심각한 보안 문제가 발생할 수 있다.

이 때문에 nodeRestriction을 설정하여 특정 노드가 다른 노드를 수정할 수 없도록 제한을 걸어야 한다.

### 02. 적용 확인 : 타 노드의 라벨 변경

k8s-worker3 노드는 최근에 클러스터에 합류한 신규 노드다. 아직 클러스터 환경에 익숙치 않은 3번 일꾼은 노드로서 지켜야할 규칙같은것이 아직 숙지되지 않았다. 그래서 함부로 master 노드의 label에 멋대로 추가하려고 한다.

우선 worker 노드의 kubelet 인증서로 kubernetes api에 접속해보자. /etc/kubernetes/ 아래를 확인하면 kubelet.conf 파일을 확인할 수 있다. 이 파일을 이용해서 kubectl 접속을 해보자. 특정 conf로 kubectl 사용하는 명령어는 아래와 같다.

<aside>
💡 **# kubectl --kubeconfig {컨피그 경로} {명령어}**

</aside>

```json
**# ls /etc/kubernetes/kubelet.conf** 
/etc/kubernetes/kubelet.conf

**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf get nodes**
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   28d   v1.24.1
k8s-worker1   Ready    <none>          28d   v1.24.1
k8s-worker2   Ready    <none>          28d   v1.24.1
k8s-worker3   Ready    <none>          98m   v1.24.1
```

node 정보를 확인할 수 있는 것을 알았다. 이제 master의 라벨에 낙서를 끄적여보자.

```json
**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf label node k8s-master scribble=funny**
Error from server (Forbidden): nodes "k8s-master" is forbidden: **node "k8s-worker3" is not allowed to modify node "k8s-master"**
```

앗 우리 3번 일꾼은 마스터에게 낙서를 할 수 없다. 금지됐기 때문이다. 그렇다면 제일 큰형인 1번 일꾼에겐 어떨까?

```json
**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf label node k8s-worker1 scribble=funny**
Error from server (Forbidden): nodes "k8s-worker1" is forbidden: **node "k8s-worker3" is not allowed to modify node "k8s-worker1"**
```

아 우리 큰형에게도 1번이 불가능하다.

### 03. 적용 확인 : 자신에게 [node-restriction.kubernetes.io](http://node-restriction.kubernetes.io) 라벨 부여

[02. 적용 확인 : 타 노드의 라벨 변경](https://www.notion.so/02-89e7c9476f654df8bb7e99fe54d5e47e?pvs=21) 에서 자신의 장난에 실패를 맛본 3번 일꾼은 의기소침해졌다. 웬지 모르겠지만 자신에게 [node-restriction.kubernetes.io](http://node-restriction.kubernetes.io)로 시작하는 라벨을 부여하고 싶어졌다. 특정 파드를 자신에게만 실행시키는 마법의 라벨이라고 한다. 자신에게 할당된 pod도 아직 없는데 한번 욕심내보려고 한다.

```json
**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf label node k8s-worker3 node-restriction.kubernetes.io/givemepod=plz**
Error from server (Forbidden): nodes "k8s-worker3" is forbidden: **is not allowed to modify labels: node-restriction.kubernetes.io/givemepod**
```

아 [node-restriction.kubernetes.io](http://node-restriction.kubernetes.io) 라벨은 pod 스케쥴과 관련된 중요한 라벨이라서 kubelet이 함부로 추가할거나 수정, 삭제할 수 없다고 한다.

### 04. 적용 확인 : 타 노드에서 실행된 pod 수정, 삭제

[03. 적용 확인 : 자신에게 [node-restriction.kubernetes.io](http://node-restriction.kubernetes.io) 라벨 부여](https://www.notion.so/03-node-restriction-kubernetes-io-6060a5b808f748acada8276ce976d77b?pvs=21) 에서 열받은 일꾼 3번은 심술을 부리기로 했다. 다른 노드에서 실행중인 pod를 삭제하려는것이다. 3번 일꾼은 타겟을 모색하다가 제일 큰형이 실행중인 test 파드를 발견한다.

```json
**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf get pod test -o wide** 
NAME   READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES
test   1/1     Running   0          3d1h   10.100.194.82   k8s-worker1   <none>           <none>
```

이를 몰래 삭제해보자

```json
**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf delete pod test**
Error from server (Forbidden): pods "test" is forbidden: **node "k8s-worker3" can only delete pods with spec.nodeName set to itself**
```

오 맙소사 너는 니가 실행한 pod만 삭제하란다. 혹시 나한테 삭제권한을 없앴나 생각이 들어 자신의 pod도 삭제해보려고 한다.

```json
**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf get pod nginx-node-restriction -o wide**
NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
nginx-node-restriction   1/1     Running   0          37s   10.100.100.194   **k8s-worker3**   <none>           <none>

**# kubectl --kubeconfig /etc/kubernetes/kubelet.conf delete pod nginx-node-restriction** 
pod "nginx-node-restriction" deleted
```

내 것은 너무 삭제가 잘된다….
