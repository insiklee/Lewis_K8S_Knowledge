# K8s PodSecurityAdmission 관리(1.21+)

[Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

[Enforce Pod Security Standards by Configuring the Built-in Admission Controller](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/#configure-the-admission-controller)

# 01. Pod Security Admission이란?

## 01. 개요

Pod Security Admission은 API Admission Controller Plugin의 하나다. 파드 보안 표준에 서 정의된 세 가지 수준(Privileged, Baseline, Rstricted)에 따라서 Pod의 SecurityContext 및 기타 관련 필드에 대한 요구사항을 지정한다.

1.25버전부터 없어진 PodSecurityPolicy의 대체제이며, PSP보다 더욱 쉽게 적용할 수 있다는 장점이 있다.

## 02. Pod Security Standard

[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

우리말로 파드 보안 표준이다. **Privileged, Baseline, Restricted 세 가지 프로필**을 기준으로 파드의 보안 수준을 결정한다.

### 01. Privileged

아무런 제한 없이 매우 폭넓고 강력한 권한을 지닐 수 있는 프로필이다. 리눅스의 root와 비슷하다 보면 된다. 시스템 및 인프라스트럭쳐 레벨에서 워크로드를 관리하기 위한 권한이다.

### 02. Baseline

잘 알려진 권한 상승을 방지하는 한편, **일반적으로 컨테이너화에 사용되는 작업중 일부를 허용할지 결정**한다. 컨테이너화에 사용되는 작업이란 컨테이너로 실행되는 **프로세스가 호스트 OS의 네임스페이스나 네트워크 포트, 커널 파라미터등을 가져와 사용하는 것**을 의미한다. 

SecurityContext 등에서 사용하지 못하도록 제한하는 항목이 있으며, 제한된 항목에는 별도로 허용된 값 이외에는 사용할 수 없다. 

### Baseline에서 제한하는 항목

### 1) HostProcess

윈도우 노드에서 실행되는 파드에 대한 권한을 관리한다.

**제한하는 항목**

- `spec.securityContext.windowsOptions.hostProcess`
- `spec.containers[*].securityContext.windowsOptions.hostProcess`
- `spec.initContainers[*].securityContext.windowsOptions.hostProcess`
- `spec.ephemeralContainers[*].securityContext.windowsOptions.hostProcess`

**허용하는 값**

- 미정의(Undefined/nil)
- false

### 2) Host Namespace

호스트 노드의 네임스페이스 자원을 공유하는 것을 제한한다.

**********************************제한하는 항목**********************************

- `spec.hostNetwork`
- `spec.hostPID`
- `spec.hostIPC`

**허용하는 값**

- 미정의(Undefined/nil)
- false

### 3) Privileged Containers

파드가 privileged되는 것을 방지한다.

**********************************제한하는 항목**********************************

- `spec.containers[*].securityContext.privileged`
- `spec.initContainers[*].securityContext.privileged`
- `spec.ephemeralContainers[*].securityContext.privileged`

**허용하는 값**

- 미정의(Undefined/nil)
- false

### 4) Capabilities

허용되지 않은 것 이외의 capabilities를 요구하는 것을 제한한다.

**********************************제한하는 항목**********************************

- `spec.containers[*].securityContext.capabilities.add`
- `spec.initContainers[*].securityContext.capabilities.add`
- `spec.ephemeralContainers[*].securityContext.capabilities.add`

****************************허용하는 값****************************

- 미정의(Undefined/nil)
- AUDIT_WRITE
- CHOWN
- DAC_OVERRIDE
- FOWNER
- FSETID
- KILL
- MKNOD
- NET_BIND_SERVICE
- SETFCAP
- SETGID
- SETPCAP
- SETUID
- SYS_CHROOT

### 5) HostPath Volumes

HostPath를 이용해서 볼륨 마운트 하는 것을 제한한다.

**제한하는 항목**

• `spec.volumes[*].hostPath`

**허용하는 값**

- 미정의(Undefined/nil)

### 6) Host Ports

호스트 포트 사용을 제한하거나 잘 알려진 포트를 사용하는 것을 최소화한다.

**제한하는 항목**

- `spec.containers[*].ports[*].hostPort`
- `spec.initContainers[*].ports[*].hostPort`
- `spec.ephemeralContainers[*].ports[*].hostPort`

**허용하는 값**

- 미정의(Undefined/nil)
- 알려진 포트 리스트
- 0

### 7) AppArmor

Apparmor를 사용하는 호스트의 경우, runtime/default가 기본적으로 적용된다. Baseline 정책에서는 이 AppArmor의 default 프로필 사용 못하게 막거나 다른 프로필로 덮어쓰는 일을 방지해준다. 

**제한하는 항목**

• `metadata.annotations["container.apparmor.security.beta.kubernetes.io/*"]`

**허용하는 값**

- 미정의(Undefined/nil)
- runtime/default
- localhost/*

### 8) SELinux

SElinux 타입을 변경하거나 커스텀 SELinux 유저나 롤을 생성하는것을 제한한다.

**제한하는 항목 1**

- `spec.securityContext.seLinuxOptions.type`
- `spec.containers[*].securityContext.seLinuxOptions.type`
- `spec.initContainers[*].securityContext.seLinuxOptions.type`
- `spec.ephemeralContainers[*].securityContext.seLinuxOptions.type`

**1의 허용하는 값**

- 미정의(Undefined/nil)
- container_t
- container_init_t
- container_kvm_t

**제한하는 항목 2**

- `spec.securityContext.seLinuxOptions.user`
- `spec.containers[*].securityContext.seLinuxOptions.user`
- `spec.initContainers[*].securityContext.seLinuxOptions.user`
- `spec.ephemeralContainers[*].securityContext.seLinuxOptions.user`
- `spec.securityContext.seLinuxOptions.role`
- `spec.containers[*].securityContext.seLinuxOptions.role`
- `spec.initContainers[*].securityContext.seLinuxOptions.role`
- `spec.ephemeralContainers[*].securityContext.seLinuxOptions.role`

**2의 허용하는 값**

- 미정의(Undefined/nil)

### 9) /proc Mount Type

기본적으로 마운트되는 /proc 정보를 마스크처리하여 보안성을 강화할 수 있다.

**제한하는 항목**

- `spec.containers[*].securityContext.procMount`
- `spec.initContainers[*].securityContext.procMount`
- `spec.ephemeralContainers[*].securityContext.procMount`

**허용하는 값**

- 미정의(Undefined/nil)
- Default

### 10) Seccomp

Seccomp가 명시적으로 무제한으로 설정되는 것을 막는다

**********************************제한하는 항목**********************************

- `spec.securityContext.seccompProfile.type`
- `spec.containers[*].securityContext.seccompProfile.type`
- `spec.initContainers[*].securityContext.seccompProfile.type`
- `spec.ephemeralContainers[*].securityContext.seccompProfile.type`

**허용하는 값**

- 미정의(Undefined/nil)
- RuntimeDefault
- Localhost

### 11) Sysctl

sysctl을 수저알 수 있다면 호스트나 다른 컨테이너에 까지 치명적인 보안 문제를 야기할 수 있다. 따라서 허용된 안전한 sysctl 파라미터 외에는 사용을 제한한다.

**제한하는 항목**

• `spec.securityContext.sysctls[*].name`

**허용하는 값**

- 미정의(Undefined/nil)
- kernel.shm_rmid_forced
- net.ipv4.ip_local_port_range
- net.ipv4.ip_unprivileged_port_start
- net.ipv4.tcp_syncookies
- net.ipv4.ping_group_range

### 03. Restricted

Pod에게 부여할 수 있는 최고의 보안정책을 부여한다.

### Restricted에서 제한하는 항목

### 1) Baseline에서 제한하는 항목들

[Baseline에서 제한하는 항목](https://www.notion.so/Baseline-4fcc61c01492404b95e3200b7fd01897?pvs=21) 

### 2) Volume Types

허용된 볼륨 타입 외에는 모두 제한한다.

**제한하는 항목**

• `spec.volumes[*]`

**허용하는 값**

아래 항목 외에는 전부 허용되지 않는다.

- `spec.volumes[*].configMap`
- `spec.volumes[*].csi`
- `spec.volumes[*].downwardAPI`
- `spec.volumes[*].emptyDir`
- `spec.volumes[*].ephemeral`
- `spec.volumes[*].persistentVolumeClaim`
- `spec.volumes[*].projected`
- `spec.volumes[*].secret`

### 3) Privilege Escalation

SetUID나 SetGID 같은 권한 상승이 허용되지 않는다.  리눅스에서만 반영된다.

**제한하는 항목**

- `spec.containers[*].securityContext.allowPrivilegeEscalation`
- `spec.initContainers[*].securityContext.allowPrivilegeEscalation`
- `spec.ephemeralContainers[*].securityContext.allowPrivilegeEscalation`

**허용되는 값**

- false

### 4) Running as Non-root

root 유저로 컨테이너를 실행하지 못하도록 막는다.

**제한하는 항목**

- `spec.securityContext.runAsNonRoot`
- `spec.containers[*].securityContext.runAsNonRoot`
- `spec.initContainers[*].securityContext.runAsNonRoot`
- `spec.ephemeralContainers[*].securityContext.runAsNonRoot`

**허용하는 값**

- true

### 5) Running as Non-root user

runAsUser 값이 0으로 설정되면 안된다.

**제한하는 항목**

- `spec.securityContext.runAsUser`
- `spec.containers[*].securityContext.runAsUser`
- `spec.initContainers[*].securityContext.runAsUser`
- `spec.ephemeralContainers[*].securityContext.runAsUser`

**허용하는 값**

- 0이 아닌 모든 값
- 미정의(Undefined/null)

### 6) Seccomp

Unconfined 프로필 외에 absence 프로필 역시 금지한다. 리눅스만 적용된다.

**제한하는 항목**

- `spec.securityContext.seccompProfile.type`
- `spec.containers[*].securityContext.seccompProfile.type`
- `spec.initContainers[*].securityContext.seccompProfile.type`
- `spec.ephemeralContainers[*].securityContext.seccompProfile.type`

**허용하는 값**

- RuntimeDefault
- Localhost

### 7) Capabilities

모든 capabilities를 Drop시켜야 한다. 유일하게 허용하는 Capability는 NET_BIND_SERVICE 뿐이다.

**제한하는 항목 1**

- `spec.containers[*].securityContext.capabilities.drop`
- `spec.initContainers[*].securityContext.capabilities.drop`
- `spec.ephemeralContainers[*].securityContext.capabilities.drop`

**1의 허용하는 값**

- 기존의 모든 Capabilities의 리스트

**제한하는 항목 2**

- `spec.containers[*].securityContext.capabilities.add`
- `spec.initContainers[*].securityContext.capabilities.add`
- `spec.ephemeralContainers[*].securityContext.capabilities.add`

**2의 허용하는 값**

- 미정의(Undefined/nil)
- NET_BIND_SERVICE

# 02. Pod Security Admission 적용

## 01. API 서버에 Admission Controller 등록

1.25버전부터 Pod Security Admission Controller는 디폴트로 설정되어 있다. 만약 그 이전 버전을 사용하고 있다면 admission 컨트롤러를 다운로드 받은 뒤 설치해줘야 한다. 방법은 아래 링크를 참조하자.

[Pod Security Admission](https://v1-24.docs.kubernetes.io/docs/concepts/security/pod-security-admission/#webhook)

## 02. 특정 네임스페이스에 적용

Pod네임스페이스에 라벨을 추가함으로써 원하는 Pod Security Standard의 프로필을 사용할 수 있다. 라벨에는 security mode와 security level을 명시한다.

### 01. Security mode

- enforce : 정책을 위반한 파드의 실행을 거절한다.
- audit : 정책을 위반한 파드가 발생했을 경우 audit 로그로 남기고 일단은 실행시킨다.
- warn : 정책 위반이 있을 경우 유저가 볼 수 있도록 warning을 표시한 다음에 파드를 실행시킨다.

### 02. Security Level

Pod Security Standard의 세 프로필을 의미한다. 자세한건 [02. Pod Security Standard](https://www.notion.so/02-Pod-Security-Standard-21559e5a955f47c3bbadc1625ca2d5b2?pvs=21) 를 참고하자.

### 03. mode version

쿠버네티스 버전이 올라갈 때 마다 각 Security Level에서 제한하는 내용이 추가되거나 변경될 수 있다.

그런데 Pod Security Admission이 적용된 클러스터에서 마이너 버전 업그레이드를 진행할때, 특정 Level에서 관리하는 항목이 변경될 경우 어떤 영향을 끼칠지 알 수 없다.

따라서 특정 마이너 버전의 level 정책을 그대로 유지하기 위해서 mode에 version을 명시한다.

<aside>
💡 **pod-security.kubernetes.io/{Security Mode}-version: [latest|v1.25]**

</aside>

### 04. 예시

아래와 같은 형식으로 라벨을 부여해야한다.

<aside>
💡 **pod-security.kubernetes.io/{Security Mode}: {Security Level}**

</aside>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: psa-test-ns
  labels:
    **pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: v1.25**
```

```json
**# kubectl apply -f psa-test-ns.yaml** 
namespace/psa-test-ns created

# kubectl get ns psa-test-ns --show-labels
NAME          STATUS   AGE   LABELS
psa-test-ns   Active   18s   kubernetes.io/metadata.name=psa-test-ns,pod-security.kubernetes.io/warn-version=v1.25,pod-security.kubernetes.io/warn=baseline
```

- **파드 실행 테스트**
    
    baseline 정책을 위반하는 hostPath 마운트를 시도하는 파드를 실행해보자.
    
    ```json
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: psa-nginx
      name: psa-nginx
      namespace: psa-test-ns
    spec:
      volumes:
      - name: hostpath-volume
        hostPath:
          path: /tmp
      containers:
      - image: nginx
        name: psa-nginx
        volumeMounts:
        - name: hostpath-volume
          mountPath: /tmp
    ```
    
    ```json
    **# kubectl apply -f psa-nginx.yaml** 
    **Warning: would violate PodSecurity "baseline:latest": hostPath volumes (volume "hostpath-volume")**
    pod/psa-nginx created
    ```
    
    Pod Security Admission을 위반했다고 경고메시지가 나온다.
    

## 03. Admission Configuration

API Server에 적용된 Pod Security Admission의 Config를 수정할 수 있다.

### 01. config 적용

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
      audit: restricted
      audit-version: v1.25
      warn: restricted
      warn-version: v1.25
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: 
      - default
```

- **defaulsts**

모든 네임스페이스에 적용될 기본 보안 정책을 의미한다. 

위 내용은 enforce 모드에는 privileged 레벨을 지정하고, audit과 warn 모드에서는 restricted 레벨을 부여했다.

- **exemptions**

Pod Security Admission이 Pod Security Policy와 가장 다른 부분이 이 예외 항목이다.

한번 설정하면 무차별적으로 보안 정책이 적용되던 PSP와 달리 Pod Security Admission은 특정 유저나 RuntimeClass, Namespace에서는 보안 정책이 적용되지 않도록 에외항목을 둘 수 있다.

위 내용은 default 네임스페이스에 대해서 경고 메시지를 출력하지 않는 config 구성이다.

### 02. API 서버에 적용

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

default에서 아래와 같이 hostPath를 사용하는 Pod를 실행해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: psa-nginx
  name: psa-nginx
spec:
  volumes:
  - name: hostpath-volume
    hostPath:
      path: /tmp
  containers:
  - image: nginx
    name: psa-nginx
    volumeMounts:
    - name: hostpath-volume
      mountPath: /tmp
```

```json
**# kubectl apply -f psa-nginx.yaml** 
pod/psa-nginx created
```

default가 exemption에 등록되어 있기 때문에 경고메시지가 발생하지 않는다.

이제 test 네임스페이스를 생성해서 실행해보자.

```json
# kubectl create ns test
namespace/test created

**# kubectl apply -f psa-nginx.yaml -n test**
**Warning: would violate PodSecurity "restricted:v1.25": allowPrivilegeEscalation != false (container "psa-nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "psa-nginx" must set securityContext.capabilities.drop=["ALL"]), restricted volume types (volume "hostpath-volume" uses restricted volume type "hostPath"), runAsNonRoot != true (pod or container "psa-nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "psa-nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")**
pod/psa-nginx created
```

경고메시지가 발생하는 것을 알 수 있다.
