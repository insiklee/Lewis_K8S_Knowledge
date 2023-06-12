# K8s ABAC 관리

[Using ABAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/abac/)

# 01. ABAC란?

## 01. 개요

Attribute based access control(속성 기반 접근 통제)의 약자다. RBAC가 사용자의 역할에 따라 접근 권한을 통제했다면, ABAC는 사용자에게 할당된 속성에 따라 접근통제한다.

## 02. RBAC와 ABAC의 차이

**RBAC는 역할에 따라 단순하게 접근권한을 부여한다.** 

어떤 회사에 1팀의 A 대리와 2팀의 B과장이 있다. 1팀은 인프라 업무를 하고 있고 2팀은 개발 업무를 한다. 그렇다면 A 대리는 회사에서 운영하는 인프라 전반에 대해서 접근 권한을 지니고, B 과장은 자신이 개발을 담당하는 특정 서버에만 접근권한이 부여된다. 또 B 과장은 사내에서 과장 이상만 접근할 수 있는 예비관리자 교육 페이지에 접근할 수 있으나, A 대리는 직급이 낮아서 접근하지 못한다.

이렇게 개인의 소속, 직위 등에 따라서 역할을 달리하는게 RBAC다.

ABAC는 개인에게 여러가지 속성을 할당하여 접근권한을 부여한다.

이러한 **속성에는 Role도 포함되며, 해당 개인이 접속을 시도하는 물리적 위치나 시간 까지도 포함될 수 있다.** 1팀의 A 대리와 C 대리는 같은 팀의 같은 직위를 갖고 있지만, A 대리가 서울 지사에서 근무하고 C 대리가 부산 지사로 잠시 파견갔다면, 두 사람의 접근 권한에는 차이가 있을 필요가 발생한다.

# 02. ABAC 사용

ABAC를 사용하기 위해서는 API Server의 --authorization-mode에 ABAC가 추가되고,--authorization-policy-file에 ABAC 관련 json 설정파일이 지정되야 한다.

## 01. ABAC Config 파일 생성

ABAC 모드를 활성화 하기 전에 config 파일을 생성해야 한다.

### 01. ABAC Config 파일 형식

**ABAC의 각 resource는 json 형식의 한 줄로 표기되어야 한다**. 어렵게 생각할 것 없이, 우리가 YAML로 정의하던 리소스들을 json으로 바꾼 후, 라인 하나로 표기한거라 생각하면 된다. 따라서 apiVersion, kind, spec 등 우리가 익숙하게 사용하는 요소들이 그대로 사용된다.

다만 여기에 쓰이는 json에서는 리스트나 맵 형식은 절대 허용하지 않는다. 즉, 대괄호가([])가 쓰일 일이 없다.

아래는 CNCF에서 예시로 제공하는 ABAC Config 파일의 내용이다.

```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "nonResourcePath": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:unauthenticated", "nonResourcePath": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"admin",     "namespace": "*",              "resource": "*",         "apiGroup": "*"                   }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"scheduler", "namespace": "*",              "resource": "pods",                       "readonly": true }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"scheduler", "namespace": "*",              "resource": "bindings"                                     }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubelet",   "namespace": "*",              "resource": "pods",                       "readonly": true }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubelet",   "namespace": "*",              "resource": "services",                   "readonly": true }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubelet",   "namespace": "*",              "resource": "endpoints",                  "readonly": true }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubelet",   "namespace": "*",              "resource": "events"                                       }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"alice",     "namespace": "projectCaribou", "resource": "*",         "apiGroup": "*"                   }}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"bob",       "namespace": "projectCaribou", "resource": "*",         "apiGroup": "*", "readonly": true }}
```

### 02. Policy Object의 속성

- spec에서는 user와 group등 속성을 부여할 **주체**를 지정해야 한다.
- 리소스명이 정의되어 있을 경우 아래와 같은 세 옵션이 추가되어야 한다.
    - apiGroup : apps나 [networking.k](http://networking.k9s.io)8s.io 같은 것들이다.
    - namespace: 우리가 아는 그 네임스페이스다.
    - resource: Pod나 Service 같은 것들을 지정한다.
- 리소스명이 정의되지 않을 경우 아래의 옵션을 사용한다.
    - nonResourcePath : /version 이나 /apis 같이 api server로부터 가져올 수 있는 api 정보들이다.
- 모드 항목에는 와일드카드 *를 사용할 수 있다.
- readonly : 해당 항목이 true로 설정된다면 읽기 속성만 가질 수 있다.

### 03. ABAC config 파일 생성 실습

ABAC config 파일 실습을 위해서 임시로 user1 계정을 생성하여 API 서버로 접근할 수 있도록 했다.

```json
**# kubectl config get-users** 
NAME
kubernetes-admin
**user1

# kubectl config get-contexts** 
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
          **user1                         kubernetes   user1**
```

그리고 user1이 맘놓고 뛰어놀 수 있는 for-user1-ns 네임스페이스를 생성했다.

```json
**# kubectl get ns for-user1-ns** 
NAME           STATUS   AGE
for-user1-ns   Active   12m
```

이제 본격적으로 config 파일을 생성해보자.

user1은 for-user1-ns 네임스페이스에서 모든 권한을 지니고

default 네임스페이스에서는 읽기 권한만 부여한다.

컨피그 파일은 /etc/kubernetes/policy 디렉터리를 추가한 뒤 생성한다.

```json
**# mkdir /etc/kubernetes/policy**
**# vi /etc/kubernetes/policy/user1-policy-file.json**
-----------------------/etc/kubernetes/policy/user1-policy-file.json--------------------------
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "user1", "namespace": "for-user1-ns", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "user1", "namespace": "default", "resource": "*", "apiGroup": "*", "readonly": true}}
---------------------------------------------------------------------------------------
```

## 02. ABAC 모드 활성화

### 01. 모드 활성화

API 서버의 파라미터를 변경하기 위해서 kube-apiserver.yaml 파일을 수정해준다. [03. ABAC config 파일 생성 실습](https://www.notion.so/03-ABAC-config-e3f470dc2e8d4147a494c24d9fd310c7?pvs=21) 에서 생성한 파일 경로를 --authorization-policy-file에 포함시키고, --authorization-mode에 ABAC를 포함시킨다.

```json
# vi /etc/kubernetes/manifests/kube-apiserver.yaml
------------------/etc/kubernetes/manifests/kube-apiserver.yaml---------------------
(전략)
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.72.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC,**ABAC   # ABAC 추가
    - --authorization-policy-file=/etc/kubernetes/policy/user1-policy-file.json**
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
------------------------------------------------------------------------------------
```

### 02. Config 파일 볼륨 마운트

[03. ABAC config 파일 생성 실습](https://www.notion.so/03-ABAC-config-e3f470dc2e8d4147a494c24d9fd310c7?pvs=21) 에서 생성한 config 파일의 경로를  hostPath 방식으로 마운트한다. 이거 안하면 policy 파일을 못찾아서 컨테이너 실행이 안된다.

```json
# vi /etc/kubernetes/manifests/kube-apiserver.yaml
------------------/etc/kubernetes/manifests/kube-apiserver.yaml---------------------
(전략)
    **volumeMounts:**
(중략)
    **- mountPath: /etc/kubernetes/policy
      name: abac-policy
      readOnly: true**
(중략)
  **volumes:**
(중략)
  **- hostPath:
      path: /etc/kubernetes/policy
      type: DirectoryOrCreate
    name: abac-policy**
------------------------------------------------------------------------------------
```

### 03. ABAC 검증

우선 user1 컨텍스트로 바꾸자

```json
**# kubectl config use-context user1** 
Switched to context "user1".

**# kubectl config current-context** 
user1
```

for-user1-ns에서 nginx 파드를 하나 생성하고 삭제해본다.

```json
**# kubectl -n for-user1-ns run nginx --image=nginx**
pod/nginx created

**# kubectl -n for-user1-ns get pod**
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          5s

**# kubectl -n for-user1-ns delete pod nginx**
pod "nginx" deleted
```

default 네임스페이스에서도 nginx 파드를 생성해본다.

```json
**# kubectl run nginx --image=nginx**
Error from server (Forbidden): pods is forbidden: **User "user1" cannot create resource "pods" in API group "" in the namespace "default": No policy matched.**
```

만들지 못한다. 대신 조회를 해보자.

```json
**# kubectl get pod**
NAME                               READY   STATUS      RESTARTS       AGE
hpa-test-deploy-847d5fd5bb-npd5b   1/1     Running     6 (159m ago)   14d
kube-bench-hxr4d                   0/1     Completed   0              7d
pod                                1/1     Running     2 (159m ago)   2d6h
```

조회는 잘 된다. 

삭제를 시도해도 할 수 없다.

```json
**# kubectl delete pod pod**
Error from server (Forbidden): pods "pod" is forbidden: **User "user1" cannot delete resource "pods" in API group "" in the namespace "default": No policy matched.**
```
