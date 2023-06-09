# K8s ServiceAccount 관리

# 01. ServiceAccount란?

## 01. 개요

kubectl로 쿠버네티스 클러스터에 접근하는 사람들은 대부분 admin 유저 계정을 통해서 접근이 가능하다. 이와 마찬가지로 **Pod에서 실행된 프로세스들도 api Server에 접근이 필요한 경우가 있는데, 이때 api로 부터 인증받는 계정이 Service Account**다.

## 02. ServiceAccount의 목적

ServiceAccount는 클러스터의 Role 리소스나 ClusterRole 리소스와 결합하여 사용한다. Role과 ClusterRole에 대한 설명은 [K8s RBAC - Role, ClusterRole 관리](https://www.notion.so/K8s-RBAC-Role-ClusterRole-398fcf1f53d748d99a1f3f622975cb8c?pvs=21) 에서 확인 가능하다.

ELK같이 모든 파드의 로그정보를 수집하는 어플리케이션의 경우 다른 파드의 상태를 가져올 수 있는 권한이 필요하다. Metric Server의 경우에도 각 파드의 자원 사용량을 계산하기 위한 접근 권한이 필요하다. 또 CoreDNS는 서비스와 파드의 DNS 주소를 생성하기 위해서 endpoint, service, pod, namespace 등의 정보를 읽을 수 있는 권한이 필요하다.

이런 경우 우리가 kubectl로 get 명령어를 사용하듯 api server에 접속하여 get이나 create 같은명령어를 사용할 수 있어야 한다.  

이렇게 특정 파드에서 실행되는 어플리케이션이 API를 통해 다른 파드나 서비스에 명령을 전달하기 위해서 적절한 Role을 부여해야 한다. 이 파드는 클러스터 내부의 모든 파드의 로그를 수집해야 하고, 다른 파드는 특정 네임스페이스에서 디플로이를 생성할 수 있어야 한다. 이런 식으로 각 파드의 역할에 따라서 Role이나 ClusterRole을 부여해야 할 수 있다.

이런 Role이나 ClusterRole과 바인딩을 하기 위해서 ServiceAccount를 생성하여 Pod와 연결시킨다. 이런 역할을 파드에 직접 매핑하지 않는 이유는 ServiceAccount가 API에 접근할 Token을 관리하기 때문이다.

# 02. ServiceAccount 생성

## 01. 명령어로 생성

<aside>
💡 **# kubectl cretae serviceaccount {서비스어카운트명}**

</aside>

```json
**# kubectl -n app-team1 create serviceaccount cicd-token** 
serviceaccount/cicd-token created

**# kubectl -n app-team1 get sa cicd-token -o yaml**
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-09-25T06:35:13Z"
  name: cicd-token
  namespace: app-team1
  resourceVersion: "164433"
  uid: 6b30e720-58e3-43e5-90ef-5002a8fe3c3e
```

## 02. YAML 생성

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-token
  namespace: app-team1
```

YAML로 만들기는 참 쉽다.

# 03. ServiceAcount 연결

## 01. default ServiceAccount

모든 파드는 ServiceAccount를 지닌다. 딱히 ServiceAccount를 지정하지 않은 Pod는 각 네임스페이스의 default ServiceAccount를 사용하기 때문이다.

```yaml
**## 현재 네임스페이스에 default 서비스어카운트가 존재한다.**
**# kubectl get sa**
NAME      SECRETS   AGE
default   0         4h40m

**## 대충 테스트를 위한 pod 하나를 생성한다.
# kubectl run nginx --image=nginx 
pod/nginx created

## 생성한 Pod의 내용을 확인해보면 deafult ServiceAccount를 사용중임을 확인할 수 있다.**
# kubectl get pod nginx -o yaml
**(전략)
  serviceAccount: default
  serviceAccountName: default
(중략)
  volumes:
  - name: kube-api-access-s562s
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
(하략)**
```

물론 default ServiceAccount는 아무런 role과 연결되지 않았기 때문에 api server에 대한 어떠한 접근 권한도 지니지 않고 있다.

## 02. 다른 ServiceAccount 연결

### 01. 명령어로 ServiceAccount 연결

기존에 실행중인 Pod는 수정이 불가능하기 때문에 명령어를 통해 ServiceAccount에 연결할 수 없다. 그 대신 Deployment나 DaemonSet같이 Pod를 관리하는 리소스에는 적용시킬 수 있다.

<aside>
💡 **# kubectl set serviceaccount [deploy|statefulset|daemonset] {수정할리소스명} {서비스어카운트명}**

</aside>

```json
**# kubectl -n app-team1 get deploy**
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
**test-deploy-sa   1/1     1            1           8s

# kubectl -n app-team1 set serviceaccount deploy test-deploy-sa cicd-token**
deployment.apps/test-deploy-sa serviceaccount updated

**# kubectl -n app-team1 get deployments.apps test-deploy-sa -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'**
cicd-token
```

### 02. YAML로 ServiceAccount 연결

Pod의 .spec에 serviceAccountName에 ServiceAccount 이름을 지정한 뒤 실행한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-sa
  name: nginx-sa
  namespace: app-team1
spec:
  **serviceAccountName: test-sa**
  containers:
  - image: nginx
    name: nginx-sa
```

```json
**# kubectl apply -f nginx-sa.yaml** 
pod/nginx-sa created

**# kubectl -n app-team1 get pod**
NAME       READY   STATUS    RESTARTS   AGE
nginx-sa   1/1     Running   0          4s
```

# 04. ServiceAccount의 토큰

## 01. ServiceAccount의 토큰 적용 개요

API 서버의 접근통제 방식중 하나는 Authentication이 있다. API 요청에 대한 사용자 혹은 서비스어카운트의 인증을 진행하는 것이다.

이때 사용자에 대해서는 자신이 직접 사인한 x509 인증서를 통해 인증하지만, 모든 서비스어카운트에게 인증서를 발급하는 것은 매우 어려운 일이다. 대신 서비스어카운트에게는 API 접속 토큰을 발행해서 사용할 수 있도록 한다.

쿠버네티스 1.21버전까지는 ServiceAccount를 생성하면 해당 서비스어카운트의 token값을 관리하는 Secret이 자동으로 생성됐었다.

### 1.21버전에서 ServiceAccount 생성 결과

```json
**# kubectl -n app-team1 create serviceaccount cicd-token** 
serviceaccount/cicd-token created

**# kubectl -n app-team1 get sa cicd-token -o yaml**
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-09-25T06:35:13Z"
  name: cicd-token
  namespace: app-team1
  resourceVersion: "164433"
  uid: 6b30e720-58e3-43e5-90ef-5002a8fe3c3e
**secrets:
- name: cicd-token-token-d5ksc   <- 관련 secret이 자동으로 생성된다.**
```

문제는 secret으로 token을 관리할 경우 token의 만료일이 따로 없어서 보안상 취약점이 생긴다는 점이다. 이 때문에 1.22버전 부터는 시크릿을 생성하지 않고  projected 볼륨을 대신 생성하여 직접 api server로부터 주기적으로 token을 받아올 수 있도록 변경됐다. 1시간 단위로 주기적으로 token을 갱신하기 때문에 보안성이 매우 강화됐다.

아래는 [03. ServiceAcount 연결](https://www.notion.so/03-ServiceAcount-c784e96d27b14659b5e9d29881c810f2?pvs=21) 에서 생성한 nginx-sa 파드의 token 정보다

```yaml
# **kubectl -n app-team1 get pod nginx-sa -o yaml**
(전략)
volumes:
  - name: kube-api-access-7szzk
    projected:
      defaultMode: 420
      sources:
      **- serviceAccountToken:**
          expirationSeconds: 3607
          path: token
      **- configMap:**
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      **- downwardAPI:**
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
(하략)
```

projectd 볼륨의 구성 정보는 아래와 같다.

- serviceAccountToken : TokenRequest API를 통해서 kube-apiserver로 부터 토큰 정보를 받아오는 기능을 한다. expirationSeconds는 처음 token 생성 후 유지하는 시간을 초단위로 지정한다. 위 설정대로라면 생성된 token은 1시간동안만 유지된다.
- configMap: 쿠버네티스 클러스터(kube-apiserver)에서 사용하는 root ca 인증서를 가져온다. 해당 ca 인증서를 통해 token을 생성해야 apiserver가 인가해줄 수 있기 때문이다.
- downwardAPI : 해당 파드의 네임스페이스 정보를 찾기위한 영역이다.

## 02. ServiceAccount의 토큰 확인

[02. 다른 ServiceAccount 연결](https://www.notion.so/02-ServiceAccount-78f108ba18934a02b9d85c7559e8f747?pvs=21)에서 사용한 예제를 기준으로 확인해 보자. app-team1 네임스페이스에 test-sa 서비스어카운트가 있고, nginx-sa 파드를 실행했다.

이후 nginx-sa 파드에 적용된 토큰값을 확인해보자. 토큰을 확인하는 방법은 아래와 같다.

<aside>
💡 **# kubectl -n {네임스페이스명} exec -it {Pod명} -- cat /var/run/secrets/kubernetes.io/serviceaccount/token**

</aside>

```json
**# kubectl -n app-team1 exec -it nginx-sa -- cat /var/run/secrets/kubernetes.io/serviceaccount/token**
eyJhbGciOiJSUzI1NiIsImtpZCI6InRMdUJOYW42Zm5yVzRpcXBkX0lMTFdkbnVpOU52WGl5NTh6MGd2eUQ5dmMifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjk2MjYwMzg3LCJpYXQiOjE2NjQ3MjQzODcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcHAtdGVhbTEiLCJwb2QiOnsibmFtZSI6Im5naW54LXNhIiwidWlkIjoiZWYyMGMyYTEtMTBmYS00YmMwLWJkNzMtMzRiMGRjNzJkM2QzIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJ0ZXN0LXNhIiwidWlkIjoiNjBlNzRhYmEtOWQ1MS00ZDI4LWExYTktMjQzNmFhYjE3YzEzIn0sIndhcm5hZnRlciI6MTY2NDcyNzk5NH0sIm5iZiI6MTY2NDcyNDM4Nywic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmFwcC10ZWFtMTp0ZXN0LXNhIn0.EFhu30J8ZYcFeFc5ceTdgiJbrEOy-GVJ3l23W4OhvgDxIw7sF-JkJzc6LhpBSRu2l2XjhX2lpwRTi3ulmX0BFvGxt6R0IxKbi3OF-xsQv1-N6LUZ561kHV6cSw7J0EjSk1crGHTWk8zcJmO41OZGMzxUwA-vACLcSuaSCi3BY1IqoqI9LE7GtiN_BnW35xeptM1iqAE0xdUTd5xEWZEDTho6IhcKTtzPVFJw4tXEkIIW7pY9PIk_3kO4Iffcoq9HxR0ofyOYVLCma7_Tu4LOSTnlB8wG48H4_ZTXmG-xmR2cmF-4YF1UxsEzl-z0uI1uCUKAye9gm_SrJVUkUOooDw
```

## 03. Pod에 token 정보 저장 안하기

서비스어카운트를 연결하면 자동으로 서비스어카운트의 토큰 정보를 마운트하여 저장하게 된다.

그런데 모든 pod가 API 서버와 연결될 필요가 없다. 이런 상황에서 파드에 token값을 저장하면 불필요한 보안 문제를 야기할 수 있다. 따라서  pod의 yaml에 automountServiceAccountToken 옵션을 false로 한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-sa2
  name: nginx-sa2
  namespace: app-team1
spec:
  serviceAccountName: test-sa
  **automountServiceAccountToken: false**
  containers:
  - image: nginx
    name: nginx-sa
```

```json
**# kubectl apply -f nginx-sa2.yaml** 
pod/nginx-sa2 created
```

**automountServiceAccountToken 옵션 유무에 따른 차이**

[02. YAML로 ServiceAccount 연결](https://www.notion.so/02-YAML-ServiceAccount-d161d1786f3647a79e7bf1877ad213ca?pvs=21) 에서 생성한 nginx-sa pod는 해당 옵션이 디폴트값인 true이기 때문에 토큰값이 volume의 하나로 지정되어 마운트됐다.

```json
**# kubectl -n app-team1 get pod nginx-sa -o yaml**
(전략)
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx-sa
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    **volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-4b82l
      readOnly: true**
(중략)
  **volumes:
  - name: kube-api-access-4b82l
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace**
(하략)
```

반면 방금 만든 nginx-sa2는 volume 영역이 아예 없다.

```json
**# kubectl -n app-team1 get pod nginx-sa2 -o yaml**
(전략)
spec:
  automountServiceAccountToken: false
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx-sa
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: k8s-worker2
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: test-sa
  serviceAccountName: test-sa
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
(하략)
```
