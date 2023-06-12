# K8s RBAC - Role, ClusterRole 관리

# 01. RBAC란?

## 01. RBAC 개요

**역할 기반 접근 통제(Role-Based Access Control)라는 뜻**으로 각각의 유저가 지니고 있는 역할에 따라서 컴퓨터 자원이나 네트워크에 접근할 수 있는 권한을 제한하는 것을 뜻한다. 쿠버네티스는 **Role**과 **Cluster Role**을 지정하여 API Server에 대한 접근을 통제한다.

**Role**은 특정 네임스페이스 안에서 유저의 역할을 정의한다.

**ClusterRole**은 클러스터 전반에서의 역할을 정의한다.

Role이나 ClusterRole와 binding되지 않은 user나 serviceaccount는 api server에 대한 모든 요청이 거부된다.

## 02. RBAC의 속성

RBAC를 구현하기 위해 각 Role이나 ClusterRole은 어떤 **API GROUP**에 속한 **Resources**에 대해서 **HTTP Request verb**를 요쳥할지를 지정해야한다.

- apiGroups : apps, [networking.k8s.io](http://networking.k8s.io) 등 YAML 파일 생성 시 apiVersion에 들어가는 내용의 Prefix다.
- Resources : Pod, Service 등 apiGroups에 속한 각 리소스들이다. 해당 내용을 확인하고자 한다면 kubectl api-resources 명령어로 확인하면 된다.
- verbs : HTTP request시 사용하는 메소드에 따라서 verb 리스트가 정해졌다. 상세한 내용은 아래 표를 참고하자

| HTTP verb | request verb |
| --- | --- |
| POST | create |
| GET, HEAD | get (for individual resources), list (for collections, including full object content), watch (for watching an individual resource or collection of resources) |
| PUT | update |
| PATCH | patch |
| DELETE | delete (for individual resources), deletecollection (for collections) |

# 02. Role

## 01. Role 생성

### 01. 명령어로 생성

<aside>
💡 **# kubectl creaet role {클러스터롤 명} -n {네임스페이스 명} --resource={리소스1},{리소스2},{리소스3}… --verb={명령어1},{명령어2},{명령어3}…**

</aside>

```json
**# kubectl -n test-ns create role test-role --resource=deployment,pod,service --verb=get,watch,list** 
role.rbac.authorization.k8s.io/test-role created

**# kubectl -n test-ns get role**
NAME        CREATED AT
test-role   2022-10-02T08:38:29Z

**# kubectl -n test-ns describe role test-role** 
Name:         test-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  **pods              []                 []              [get watch list]
  services          []                 []              [get watch list]
  deployments.apps  []                 []              [get watch list]**
```

### 02. YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: test-role
  namespace: test-ns
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - services
  verbs:
  - get
  - watch
  - list
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
  - watch
  - list
```

## 02. RoleBinding 생성

유저 또는 그룹, 서비스어카운트에게 특정 네임스페이스에서의 역할을 부여하기 위해서 Role과 ****바인딩한다.

### 01. 명령어로 생성

<aside>
💡 **# kubectl -n {네임스페이스명} create rolebinding {롤바인딩 명} --role={롤 명} \
[--serviceaccount={서비스어카운트의 네임스페이스명}:{서비스어카운트 명}] \
[--user={유저명}] \
[ --group={그룹명}]**

</aside>

rolebinding을 할때 서비스어카운트나 유저, 그룹 중 최소한 하나를 지정해야 한다. 셋 모두 없어도 생성은 되지만 아무런 subject가 없는 껍데기가 만들어질 뿐이다.

```json
**# kubectl -n test-ns get role,sa**
NAME                                       CREATED AT
**role.rbac.authorization.k8s.io/test-role   2022-10-02T08:38:29Z**

NAME                     SECRETS   AGE
serviceaccount/default   0         4h52m
**serviceaccount/test-sa   0         2m24s

# kubectl -n test-ns create rolebinding test-role-binding --role=test-role --serviceaccount=test-ns:test-sa**
rolebinding.rbac.authorization.k8s.io/test-role-binding created

**# kubectl -n test-ns get rolebinding**
NAME                ROLE             AGE
test-role-binding   Role/test-role   2m23s
```

# 03. ClusterRole

## 01. ClusterRole 생성

### 01. 명령어로 생성

<aside>
💡 **# kubectl creaet clusterrole {클러스터롤 명} --resource={리소스1},{리소스2},{리소스3}… --verb={명령어1},{명령어2},{명령어3}…**

</aside>

```json
**# kubectl create clusterrole deployment-clusterrole --resource=deployment,statefulset,daemonset --verb=create**
clusterrole.rbac.authorization.k8s.io/deployment-clusterrole created

**# kubectl get clusterrole deployment-clusterrole** 
NAME                     CREATED AT
deployment-clusterrole   2022-09-25T06:30:59Z

**# kubectl describe clusterrole deployment-clusterrole** 
Name:         deployment-clusterrole
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources          Non-Resource URLs  Resource Names  Verbs
  ---------          -----------------  --------------  -----
  **daemonsets.apps    []                 []              [create]
  deployments.apps   []                 []              [create]
  statefulsets.apps  []                 []              [create]**
```

### 02. YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: deployment-clusterrole
rules:
- apiGroups:
  - apps
  resources:
  - deployments
  - statefulsets
  - daemonsets
  verbs:
  - create
```

## 02. ClusterRoleBinding 생성

### 01. 명령어로 생성

<aside>
💡 **# kubectl create rolebinding {롤바인딩 명} --clusterrole={클러스터롤 명} \
[--serviceaccount={서비스어카운트의 네임스페이스명}:{서비스어카운트 명}] \
[--user={유저명}] \
[ --group={그룹명}]**

</aside>

clusterrolebinding을 할때 서비스어카운트나 유저, 그룹 중 최소한 하나를 지정해야 한다. 셋 모두 없어도 생성은 되지만 아무런 subject가 없는 껍데기가 만들어질 뿐이다.

```json
**# kubectl create rolebinding deploy-rb -n app-team1 --clusterrole=deployment-clusterrole --serviceaccount=default:cicd-token**
rolebinding.rbac.authorization.k8s.io/deploy-rb created

**# kubectl get rolebindings -n app-team1**
NAME        ROLE                                 AGE
deploy-rb   ClusterRole/deployment-clusterrole   2m2s
```

# 04. RBAC를 위한 인증서

RBAC에 사용될 유저나 그룹 등은 별도의 인증서가 필요할 수 있다. 이들이 사용할 인증서는 별도로 만들어야 하며, 이를 위해 CSR(Certificate Signing Request) 리소스가 필요하다.

[K8s RBAC - CertificateSigningRequests 관리](https://www.notion.so/K8s-RBAC-CertificateSigningRequests-b2b35cb83aa44e35a29189463d3220c4?pvs=21)
