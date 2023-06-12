# K8s PodSecurityPolicy 관리 (~1.24)

[파드 시큐리티 폴리시](https://kubernetes.io/ko/docs/concepts/security/pod-security-policy/)

영문 페이지에는 관련 내용이 더이상 나오지 않는다.

[PodSecurityPolicy Deprecation: Past, Present, and Future](https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/)

# 01. PodSecurityPolicy란?

## 01. 개요

쿠버네티스에 내장된 admission controller다. 클러스터 관리자가 파드 측면에서 보안 민감도를 조정할 수 있다. 

## 02. 등장 배경

PSP는 RBAC의 한계를 보완하기 위해 등장했다. RBAC는 클러스터의 각 리소스에 대한  list, get, create, edit, delete와 관련된 요청을 관리할 수 있으나, 호스트 노드로의 접근까지는 관리할 수 없다. 각 파드는 SecurityContext를 통해 호스트 노드의 네트워크나 파일시스템 접근, SELinux 컨텍스트에까지 접근할 수 있다. 또 privileged나 privilegedEscalation이 가능할 경우 호스트 노드의 root 권한까지 탈취해갈 수 있다. 이런식으로 **호스트 노드의 리소스에 접근 가능한 파드가 무분별하게 배포될 경우 클러스터의 각 노드들은 심각한 보안 위협에 처하게 된다.** 이를 막기 위해선 RBAC 외에 또다른 제어가 필요했다. 그 때문에 1.3 버전부터 PSP를 추가하여 Pod의 호스트 노드 접근에 제약을 걸 수 있도록 하였다.

## 03. 1.25에서 없어지는 이유?

PSP로 설정한 정책이 클러스터 전반에 너무 광범위하게 적용될 수 있다는 문제가 있다. 또 주어진 상황에서 어떤 psp가 적용됐는지 확인하기 힘들다. 따라서 [K8s PodSecurityAdmission 관리(1.21+)](https://www.notion.so/K8s-PodSecurityAdmission-1-21-74fa7fadf5c74164afc7729a7950c3ca?pvs=21) 라는 대안을 내놓았고, PSP는 역사의 뒤안길로 보내버렸다.

# 02. PodSecurityPolicy 사용

## 01. Admission Controller Plugin 활성화

api 서버에서 --enable-admission-plugins에 PodSecurityPolicy를 추가해준다.

```json
# vi /etc/kubernetes/manifests/kube-apiserver.yaml
------------/etc/kubernetes/manifests/kube-apiserver.yaml----------------
(전략)
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.72.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    **- --enable-admission-plugins=NodeRestriction,PodSecurityPolicy**
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
(하략)
```

이후 마스터노드에서 kubelet을 재시작해준다.

## 02. PodSecurityPolicy 리소스 생성

아래는 공식홈페이지에서 제공하는 리소스 예제다. CKS만 따면 되니 세부적인 내용은 생략하고 구현만 해보자.

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

privileged가 false로 설정됐기 때문에 모든 파드는 priveleged 모드를 지닐 수 없다.

그 외에는 selinux, supplementalGroups, runAsUser, fsGroup, volumes에 대해 모두 허용한다.

```json
**# kubectl apply -f psp.yaml** 
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/example created

**# kubectl get psp**
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
NAME      PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
example   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
```

## 03. Pod 생성 테스트

[02. PodSecurityPolicy 리소스 생성](https://www.notion.so/02-PodSecurityPolicy-49e64cb692d84f3e834f48ae74abb42c?pvs=21) 에서 만든 PSP를 가지고 Pod 생성 테스트를 해보자.

일단 아래와 같이 deployment를 생성해보자.

```json
**# kubectl create deploy nginx --image=nginx**
deployment.apps/nginx created
```

근데 deployment 상태를 확인해보면 pod가 ready가 되지 않는 것을 볼 수 있다.

```json
**# kubectl get deploy nginx**
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   **0/1**     0            0           2m31s
```

deployment가 생성한 레플리카셋의 정보를 보자

```json
**# kubectl get rs -l app=nginx**
NAME              DESIRED   CURRENT   READY   AGE
nginx-8f458dc5b   **1         0         0**       2m59s
```

레플리카셋에 문제가 있는것 같다. describe로 확인해보자.

```json
**# kubectl describe rs nginx-8f458dc5b** 
Name:           nginx-8f458dc5b
Namespace:      default
(중략)
Events:
  Type     Reason        Age                   From                   Message
  ----     ------        ----                  ----                   -------
  Warning  FailedCreate  40s (x16 over 3m24s)  replicaset-controller  E**rror creating: pods "nginx-8f458dc5b-" is forbidden: PodSecurityPolicy: unable to admit pod: []**
```

권한이 없어서 생성이 불가능하다고 한다.

디플로이먼트는 API 컨트롤러를 통해서 레플리카셋을 생성하고, 레플리카셋 역시 API 컨트롤러를 통해 파드를 생성한다. **이때 이들은 defaultServiceAccount를 통해서 하위 리소스를 생성한다.**

모든 네임스페이스에는 따로 serviceAccount가 명시되지 않은 Pod에게 할당하기 위한 default ServiceAccount가 있다.

```json
**# kubectl get sa -A | grep default**
app-team1                default                                   0         24d
cm-test                  default                                   0         23d
default                  default                                   0         24d
deploy                   default                                   0         8d
for-user1-ns             default                                   0         2d
gatekeeper-system        default                                   0         3d1h
ingress-nginx            default                                   0         23d
ingress-test             default                                   0         23d
kube-node-lease          default                                   0         24d
kube-public              default                                   0         24d
kube-system              default                                   0         24d
kubernetes-dashboard     default                                   0         25h
prod                     default                                   0         8d
prometheus               default                                   0         13d
provisioner-nfs-subdir   default                                   0         13d
test-ns                  default                                   0         24d
```

위 예시를 보면 모든 네임스페이스에 default 서비스어카운트가 있는것을 확인할 수 있다.

문제는 이 서비스어카운트에 적절한 role이 부여되지 않았다는 점이다.

## 04. PodSecurityPolicy 우회를 위한 RBAC

특정 ServiceAccount가 PodSecurityPolicy에 의해 권한을 차단당하는 것을 방지하기 위해 PSP를 사용하는 Role을 생성하여 SA에 부여해야한다.

<aside>
💡 **# kubectl -n {네임스페이스 명} create role {role 명} --verb=use --resource=PodSecurityPolicy --resource-name={PSP 리소스명}**

</aside>

```json
**# kubectl create role psp:allow --verb=use --resource=PodSecurityPolicy --resource-name=example**
role.rbac.authorization.k8s.io/psp:allow created

**# kubectl get role**
NAME        CREATED AT
psp:allow   2022-10-26T14:57:27Z
```

이제 default 서비스 어카운트를 role에 바인딩하자.

```json
**# kubectl create rolebinding psp-default-rolebinding --role=psp:allow --serviceaccount=default:default**
rolebinding.rbac.authorization.k8s.io/psp-default-rolebinding created

**# kubectl get rolebinding -o wide**
NAME                      ROLE             AGE   USERS   GROUPS   SERVICEACCOUNTS
psp-default-rolebinding   Role/psp:allow   17s                    default/default
```

다시 디플로이를 생성해보자.

```json
**# kubectl delete deploy nginx**
deployment.apps "nginx" deleted

**# kubectl get deploy,rs,po -l app=nginx**
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           13s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-8f458dc5b   1         1         1       13s

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-8f458dc5b-xn975   1/1     Running   0          13s
```

이제 정상적으로 생성되는 것을 확인할 수 있다.
