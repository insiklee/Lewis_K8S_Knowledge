# K8s Audit 관리

[Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)

# 01. Audit이란?

## 01. 개요

audit은 우리말로 감사다. 선물 받을때 말하는 감사가 아니라 단체나 조직원의 행동에 문제가 없는지 조사하고 감찰한다고 할때 그 감사다.

쿠버네티스에서는 여러 사용자가 API 서버에 접근하여 다양한 리소스를 생성하거나 수정, 삭제할 수 있다. 문제는 이런 일련의 작업들에 대한 기록이 남아있지 않는다면 장애사항이나 보안사고 발생 시 원인 규명이 매우 힘들어 질 수 있다.

이 때문에 쿠버네티스 내부에서의 다양한 작업에 대해서 감사 기록을 남길 필요가 있다.

리눅스의 syslog, rsyslog와 비슷한 역할을 한다고 생각하면 된다.

## 02. Audit Stage

클라이언트(사용자, 서비스어카운트)와 API 서버 사이에 HTTP Request와 HTTP Response가 오갈 때, 어느 단계에서 감사를 진행할지를 결정한다.

- RequestRecived :  API 서버가 요청을 받았을때 활성화된다.
- ResponseStarted : API 서버가 watch와 같이 아주 기—인 요청을 받았을 때, 일단 HTTP Response의 헤더값만 보내고 바디를 아직 안보낼때 활성화된다.
- ResponseComplete : API 서버가 HTTP Response를 완전히 보냈을 때 활성화된다.
- Panic : 패닉사항이 발생했을때 활성화된다.

## 03. Audit 정책

Audit의 정책은 어떤 종류의 이벤트가 기록되고, 어떤 데이터가 포함됐는지를 결정한다. Policy 리소스는 audit.k8s.io 그룹에서 정의되어 있으며 감사 레벨에 따라서 정책을 설정할 수 있다.

### 01. 감사 레벨

- None : none으로 설정된 이벤트는 로그를 남기지 않는다.
- Metadata : 메타데이터 (유저, 타임스탬프, 리소스, verb와 같은 헤더 정보)를 로깅하고, request와 response의 body 는 남기지 않는다.
- Request : 메타데이터와 request body 내용을 로깅하지만, responce body는 로깅하지 않는다.
- RequestResponse : 메타데이터와 reqeust, response body 내용을 로깅한다.

# 02. Audit 적용

## 01. Audit Policy 생성

Audit policy는 아래와 같은 형식을 지니고 있다.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    namespaces: ["kube-system"]
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions"
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

- omitstages : 어떤 단계에서 감사를 진행할지 결정한다. 종류는 [02. Audit Stage](https://www.notion.so/02-Audit-Stage-ebcb16d5edfe416ba7853a5a991e7556?pvs=21) 에서 확인하자
- rules : 정책에 들어갈 룰을 결정한다.
- rules.level : 어떤 감사 레벨을 사용할지 결정한다. [01. 감사 레벨](https://www.notion.so/01-973564809d6744b895633f992fb4fb82?pvs=21) 에 목록이 있다.
- rules.resources.group : API 리소스 그룹을 의미한다. v1일 경우 그냥 “”만 표기하고, 그 외에는 apps나 networking.k8s.io 같은 리소스 그룹을 넣는다.
- rules.resources.resources : 정책에 반영할 리소스를 선택한다.
- rules.namespace : 어느 네임스페이스에서 감사를 할지 결정한다.
- rules.nonResourceURLs : 리소스형태가 아닌 경로에 대해 요청할때 감사를 한다.

위 내용을 적절한 위치에 저장하자.

```json
**# mkdir /etc/kubernetes/audit

# mv audit-policy.yaml /etc/kubernetes/audit/**
```

## 02. API 서버에 반영

### 01. api server 파라미터 수정

Audit Policy는 API 서버에서 동작한다. 따라서 resource 형태로 실행하는 것이 아닌, API 서버에 직접 적용시켜야 한다.

이를 위해 api-server 파드에 관련 파라미터를 추가해준다.

```yaml
**# vi /etc/kubernetes/manifests/kube-apiserver.yaml**
------------vi /etc/kubernetes/manifests/kube-apiserver.yaml-------------**-**
(전략)
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.122.10
    - --allow-privileged=true
    **- --audit-policy-file=/etc/kubernetes/audit/audit-policy.yaml
    - --audit-log-path=/etc/kubernetes/audit/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=1000**
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
(하략)
----------------------------------------------------------------------------
```

- --audit-policy-file : audit을 위한 policy가 저장된 yaml 파일 위치를 뜻한다.
- --audit-log-path : audit 로그를 저장하기 위한 경로를 뜻한다.
- --audit-log-maxage : 로그를 보관하기 위한 기간을 일 단위로 적는다.
- --audit-log-maxbackup : 로테이트가 돈 후 몇 개의 로그를 보관할지 결정한다.
- --audit-log-maxsize : 하나의 로그 파일의 최대 크기를 메가파이트 단위로 결정한다.

### 02. hostPath volume mount

api 서버가 Host에 로그 파일을 저장하고, policy 파일을 읽어들일 수 있도록 hostPath 방식으로 volume mount를 진행한다.

```yaml
**# vi /etc/kubernetes/manifests/kube-apiserver.yaml**
------------vi /etc/kubernetes/manifests/kube-apiserver.yaml-------------**-**
(전략)
    **volumeMounts:
    - mountPath: /etc/kubernetes/audit
      name: audit**
(중략)
volumes:
  **- hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
    name: audit**
(하략)
```

apiserver 파드에서 모든 volumeMount는 readOnly 방식으로 마운트된다. 하지만 audit은 로그 파일을 저장하기 위한 쓰기 권한이 필요하므로 readOnly 설정을 제외한다.

설정이 완료되면 kubelet을 재시작한다.

## 03. 로그 생성 확인

### 01. 로그 파일 생성 확인

마스터 노드에서 /etc/kubernetes/audit 디렉터리에 로그 파일이 생성됐는지 확인한다.

```json
**# ls /etc/kubernetes/audit/**
audit.log  audit-policy.yam
```

tail -f 명령어로 확인하면 지속적으로 로그가 쌓이는 것을 확인할 수 있다.

```json
**# tail -f /etc/kubernetes/audit/audit.log
{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata","auditID":"fd80c915-980f-4cb3-8d6b-d5ba44a9118d","stage":"ResponseComplete","requestURI":"/api/v1/namespaces/ingress-nginx/configmaps/ingress-controller-leader","verb":"get","user":{"username":"system:serviceaccount:ingress-nginx:ingress-nginx","uid":"65477c13-f569-4a19-8f90-6e99c36f0263","groups":["system:serviceaccounts","system:serviceaccounts:ingress-nginx","system:authenticated"],"extra":{"authentication.kubernetes.io/pod-name":["ingress-nginx-controller-78786894bf-lxhr5"],"authentication.kubernetes.io/pod-uid":["f3d5b493-0885-45a5-b8e5-1f6248aa88a8"]}},"sourceIPs":["192.168.122.12"],"userAgent":"nginx-ingress-controller/v1.1.2 (linux/amd64) ingress-nginx/bab0fbab0c1a7c3641bd379f27857113d574d904","objectRef":{"resource":"configmaps","namespace":"ingress-nginx","name":"ingress-controller-leader","apiVersion":"v1"},"responseStatus":{"metadata":{},"code":200},"requestReceivedTimestamp":"2022-10-27T05:14:51.942085Z","stageTimestamp":"2022-10-27T05:14:51.949349Z","annotations":{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":"RBAC: allowed by RoleBinding \"ingress-nginx/ingress-nginx\" of Role \"ingress-nginx\" to ServiceAccount \"ingress-nginx/ingress-nginx\""}}
{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata","auditID":"3a5a1b63-7fc9-4301-91a2-9730603a3d52","stage":"ResponseComplete","requestURI":"/api/v1/namespaces/ingress-nginx/configmaps/ingress-controller-leader","verb":"update","user":{"username":"system:serviceaccount:ingress-nginx:ingress-nginx","uid":"65477c13-f569-4a19-8f90-6e99c36f0263","groups":["system:serviceaccounts","system:serviceaccounts:ingress-nginx","system:authenticated"],"extra":{"authentication.kubernetes.io/pod-name":["ingress-nginx-controller-78786894bf-lxhr5"],"authentication.kubernetes.io/pod-uid":["f3d5b493-0885-45a5-b8e5-1f6248aa88a8"]}},"sourceIPs":["192.168.122.12"],"userAgent":"nginx-ingress-controller/v1.1.2 (linux/amd64) ingress-nginx/bab0fbab0c1a7c3641bd379f27857113d574d904","objectRef":{"resource":"configmaps","namespace":"ingress-nginx","name":"ingress-controller-leader","uid":"a996d2ca-1eac-4a66-b6c6-fa1b95be3805","apiVersion":"v1","resourceVersion":"202652"},"responseStatus":{"metadata":{},"code":200},"requestReceivedTimestamp":"2022-10-27T05:14:51.951079Z","stageTimestamp":"2022-10-27T05:14:51.954284Z","annotations":{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":"RBAC: allowed by RoleBinding \"ingress-nginx/ingress-nginx\" of Role \"ingress-nginx\" to ServiceAccount \"ingress-nginx/ingress-nginx\""}}
(하략)**
```

### 02. 리소스 생성에 대한 감사 확인

시크릿을 한개 만들어 보자.

```json
**# kubectl create secret generic audit-test-secret --from-literal=audit=lookatme**
secret/audit-test-secret created
```

내가 시크릿 만든 것이 저장됐는지 확인해보자.

```json
# cat /etc/kubernetes/audit/audit.log | grep audit-test-secret | jq
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "c555a5cb-002e-4356-bc87-5c8aa4ac5d5a",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/secrets?fieldManager=kubectl-create&fieldValidation=Strict",
  "verb": "create",
  "user": {
    "username": "kubernetes-admin",
    "groups": [
      "system:masters",
      "system:authenticated"
    ]
  },
  "sourceIPs": [
    "192.168.122.10"
  ],
  "userAgent": "kubectl/v1.24.1 (linux/amd64) kubernetes/3ddd0f4",
  "objectRef": {
    "resource": "secrets",
    "namespace": "default",
    **"name": "audit-test-secret",**
    "apiVersion": "v1"
  },
  "responseStatus": {
    "metadata": {},
    "code": 201
  },
  "requestReceivedTimestamp": "2022-10-27T05:25:51.281925Z",
  "stageTimestamp": "2022-10-27T05:25:51.289909Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": ""
  }
}
```
