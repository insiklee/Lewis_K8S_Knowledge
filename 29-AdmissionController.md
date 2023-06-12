# K8s Admission Controller

[Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

# 01. API Admission Controller란?

## 01. 개요

Admission Controller는 Authentication로 사용자 인증을 받고, Authorization로 권한에 대한 인가를 받은 뒤, 앞 단계에서 수행하지 못하는 추가적인 승인 통제를 진행한다. 조금 더 세부적으로는 **리소스에 대한 생성, 삭제, 수정 요청을 제한하고, Proxy를 이용한 접속 역시 통제**한다. 쉽게 말해 Authorization의 보조역할을 한다고 하면 이해하기 쉽다. 다만 authentication과 authorization과 달리, **여러 모듈 중에서 하나만 실패하더라도 요청이 거부**된다.

admission Controller는 mutating admission controller와 validating admission controller가 있다. mutating 관련 컨트롤러가 먼저 동작한 뒤, validating 컨트롤러가 돌아가는 식이다. 물론 둘 모두의 기능을 하는 컨트롤러도 있다.

## 02. 우리가 이미 알고 있는 Admission Controller

쿠버네티스 보안 및 서버 자원 관리를 목적으로 하는 많은 리소스 들이 있다. ServiceAccount와 ResourceQuota, LimitRange 등이 그 예다. 이런 서버 자원을 관리하는 리소스들은 이미 Admission Controller에 의해서 관리되고 있는 리소스들이다.

## 03. Default Plugins

아래 리스트는 다른 설정을 하지 않아도 클러스터 설치와 함께 적용되는 Admission Controller Plugins 목록이다.

```json
CertificateApproval
CertificateSigning
CertificateSubjectRestriction
DefaultIngressClass
DefaultStorageClass
DefaultTolerationSeconds
LimitRanger
MutatingAdmissionWebhook
NamespaceLifecycle
PersistentVolumeClaimResize
PodSecurity
Priority
ResourceQuota
RuntimeClass
ServiceAccount
StorageObjectInUseProtection
TaintNodesByCondition
ValidatingAdmissionWebhook
```

# 02. Admission Controller 활성 및 제거

## 01. Admission Controller 활성

default로 설정된 plugin 말고 추가 plugin을 사용하고자 한다면 api-server 명령어 중 --enable-admission-plugins에 추가 플러그인을 명시한다.

kube-apiserver 파드는 static-pod로 master 노드의 로컬에 manifest 형태로 저장되어 있다. 경로는 /etc/kubernetes/manifests/kube-apiserver.yaml 이다. 해당 파일을 실행하여 위 옵션을 수정해준다.

```json
**# vi /etc/kubernetes/manifests/kube-apiserver.yaml**
------------------/etc/kubernetes/manifests/kube-apiserver.yaml------------------
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.122.10:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.122.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    **- --enable-admission-plugins=NodeRestriction**
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
(하략)
-----------------------------------------------------------------------------------
```

위 옵션에 필요한 plugin 정보를 입력한 뒤, 약 10초간 기다리거나 kubelet을 재시작하면 추가 컨트롤러가 반영된다.

## 02. Admission Controller 비활성화

api server의 커맨드에 --disable-admission-plugins 옵션을 추가하면 원하는 controller를 비활성화 할 수 있다. default 컨트롤러중 일부를 제거할때 사용한다.

```json
**# vi /etc/kubernetes/manifests/kube-apiserver.yaml**
------------------/etc/kubernetes/manifests/kube-apiserver.yaml------------------
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.122.10:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.122.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    **- --disable-admission-plugins=DefaultIngressClass,DefaultStorageClass**
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
(하략)
-----------------------------------------------------------------------------------
```

## 03. Admission Controller Configuration

API 서버에서 활성화된 Admission Controller Plugin의 세부 설정을 넣을 수 있다. 본 예시에서는 쿠버네티스 1.25 버전에서 Pod Security Admission의 configuration 파일을 적용시켜볼 예정이다.

### 01. Configuration 파일 준비

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: privileged
      enforce-version: v1.25
      audit: privileged
      audit-version: v1.25
      warn: privileged
      warn-version: v1.25
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - default
```

### 02. API 서버에 등록

API 서버에서 Configuration 파일을 읽을 수 있도록 적절한 위치로 옮긴다.

```json
**# mkdir /etc/kubernetes/admission-configs

# cp pod-security-admission-config.yaml /etc/kubernetes/admission-configs/**
```

이제 매니페스트의 kube-apiserver.yaml의 내용을 수정한다.

```json
# vi /etc/kubernetes/manifests/kube-apiserver.yaml
---------------/etc/kubernetes/manifests/kube-apiserver.yaml-------------------
(전략)
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.2.11
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    **- --admission-control-config-file=/etc/kubernetes/admission-configs/pod-security-admission-config.yaml**
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
(계속)
```

이제 admission-confings 디렉터리를 파드에 볼륨 마운트 하자.

```json
(계속)
(전략)
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/pki
      name: etc-pki
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    **- mountPath: /etc/kubernetes/admission-configs
      name: admission-configs
      readOnly: true**
(중략)
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/pki
      type: DirectoryOrCreate
    name: etc-pki
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  **- hostPath:
      path: /etc/kubernetes/admission-configs
      type: DirectoryOrCreate
    name: admission-configs
---------------------------------------------------------------------------**
```

이제 파드가 재실행될 때 까지 기다리거나 kubelet을 재시작한다.

### 03. 적용 확인

```json
**# kubectl -n kube-system get pod kube-apiserver-master -o jsonpath='{.spec.containers[0].command}' | jq**
[
  "kube-apiserver",
  "--advertise-address=192.168.2.11",
  "--allow-privileged=true",
  "--authorization-mode=Node,RBAC",
  "--client-ca-file=/etc/kubernetes/pki/ca.crt",
  "--enable-admission-plugins=NodeRestriction",
  **"--admission-control-config-file=/etc/kubernetes/admission-configs/pod-security-admission-config.yaml",**
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

# 03. Admission Controller의 활용

## 01. PodSecurityPolicy (~1.24)

[K8s PodSecurityPolicy 관리 (~1.24)](https://www.notion.so/K8s-PodSecurityPolicy-1-24-6d07fdf605964d1a9e7998088015a9e5?pvs=21)

## 02. Pod Security Admission

[K8s PodSecurityAdmission 관리(1.21+)](https://www.notion.so/K8s-PodSecurityAdmission-1-21-74fa7fadf5c74164afc7729a7950c3ca?pvs=21)
