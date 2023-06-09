# K8s SecurityContext 적용

# 01. SecurityContext란?

## 01. 개요

파드나 컨테이너에 특수한 권한을 정의하거나 접근 통제를 할 수 있는 구문이다. 

## 02. Security Context의 기능

[Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#securitycontext-v1-core)

- UID와 GID를 지정할 수 있다
- SELinux의 context를 추가할 수 있다
- priveilege(sudo)를 부여할 수 있다.
- root 계정이 지니고 있는 무소불위의 권한중 일부 권한만 골라서 부여할 수 있다.
- AppArmor 설정이 가능하다.
- seccomp로 프로세스 시스템 콜을 제한할 수 있따.
- 컨테이너의 RootFilesystem을 읽기전용으로 마운트 할 수 있다.

# 02. SecurityContext 적용

- **PodSecurityContext**
    
    [Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#podsecuritycontext-v1-core)
    
- **ContainerSecurityContext**
    
    [Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#securitycontext-v1-core)
    

## 01. UID/GID 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: run-as-user
  name: run-as-user
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 2000
  containers:
  - image: nginx
    name: run-as-user
```

```yaml
**# kubectl apply -f run-as-user.yaml** 
pod/run-as-user created

**# kubectl get pod**
NAME          READY   STATUS    RESTARTS   AGE
run-as-user   1/1     Running   0          4s

**# kubectl exec -it run-as-user -- ps**
PID   USER     TIME  COMMAND
    1 **1000**      0:00 /bin/sh -c while true; do sleep 36000; done
    7 **1000**      0:00 sleep 36000
   22 **1000**      0:00 ps

**# kubectl exec -it run-as-user -- id**
**uid=1000 gid=2000**
```

## 02. Privileged 설정

privileged를 설정하면 pod에게 **pod를 실행한 Host node의 root 계정과 똑같은 권한을 부여**한다. 디폴트로 false로 되어 있다.

**privileged 설정을 안했을 경우**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-test
  name: sc-test
spec:
  containers:
  - image: busybox
    name: sc-test
    args:
    - /bin/sh
    - -c
    - "sleep 36000"
```

```json
**# kubectl get pod**
NAME      READY   STATUS    RESTARTS   AGE
sc-test   1/1     Running   0          19s

**# kubectl exec -it sc-test -- sysctl kernel.hostname=changehostname**
sysctl: error setting key 'kernel.hostname': Read-only file system
```

privileged가 설정 안됐을 경우 root의 권한인 sysctl 수정이 불가능하다.

**privileged가 true로 설정됐을 경우**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-test
  name: sc-test
spec:
  containers:
  - image: busybox
    name: sc-test
    args:
    - /bin/sh
    - -c
    - "sleep 36000"
    securityContext:
      privileged: true
```

```json
**# kubectl get pod**
NAME      READY   STATUS    RESTARTS   AGE
sc-test   1/1     Running   0          23s

**# kubectl exec -it sc-test -- sysctl kernel.hostname=changehostname**
kernel.hostname = changehostname
```

잘 바뀐다.

## 03. allowPrivilegeEscalation 설정

allowPrivilegeEscalation은 SetUID나 SetGID같은 특수권한 사용여부를 결정한다. false로 설정하면 프로세스의 NoNewPrivs 플래그를 활성화하며, 이는 SetUID 또는 SetGID 사용을 차단하여 더 큰 권한을 못지니게 한다.

**allowprivilegeEscalation 적용**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-test
  name: sc-test
spec:
  containers:
  - image: busybox
    name: sc-test
    args:
    - /bin/sh
    - -c
    - "sleep 36000"
    securityContext:
      **allowPrivilegeEscalation: true**
```

```yaml
# kubectl exec -it sc-test -- cat /proc/1/status | grep NoNewPrivs
NoNewPrivs:	0
```

**allowPrivilegeEscalation 미적용**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-test
  name: sc-test
spec:
  containers:
  - image: busybox
    name: sc-test
    args:
    - /bin/sh
    - -c
    - "sleep 36000"
    securityContext:
      **allowPrivilegeEscalation: false**
```

```yaml
# kubectl exec -it sc-test -- cat /proc/1/status | grep NoNewPrivs
NoNewPrivs:	**1**
```

## 04. capability 설정

해당 내용을 설정하기 위해서는 아래 링크를 우선 참조해야한다.

[capabilities(7) - Linux manual page](https://man7.org/linux/man-pages/man7/capabilities.7.html)

아래 링크도 함께 봐보자

[Linux Capabilities in OpenShift](https://cloud.redhat.com/blog/linux-capabilities-in-openshift)

우선 아무런 securityContext를 적용하지 않은 pod의 capbilities 정보를 확인해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-test
  name: sc-test
spec:
  containers:
  - image: busybox
    name: sc-test
    args:
    - /bin/sh
    - -c
    - "sleep 36000"
```

```yaml
**# kubectl exec -it sc-test -- cat /proc/1/status | grep Cap**
CapInh:	0000000000000000
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
```

보기 좀 복잡하다.  이 경우 capsh 명령어를 사용하면 쉽게 해독이 가능하다.

<aside>
💡 **capsh --decode={Capabilities code}**

</aside>

```json
# capsh --decode=00000000a80425fb | tr ',' '\n'
WARNING: libcap needs an update (cap=40 should have a name).
0x00000000a80425fb=cap_chown
cap_dac_override
cap_fowner
cap_fsetid
cap_kill
cap_setgid
cap_setuid
cap_setpcap
cap_net_bind_service
cap_net_raw
cap_sys_chroot
cap_mknod
cap_audit_write
cap_setfcap
```

이제 컨테이너에는 없는 CAP_NET_ADMIN과 CAP_SYS_TIME capability를 부여해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-test
  name: sc-test
spec:
  containers:
  - image: busybox
    name: sc-test
    args:
    - /bin/sh
    - -c
    - "sleep 36000"
    securityContext:
      capabilities:
        add:  
        - NET_ADMIN
        - SYS_TIME
```

```json
**# kubectl exec -it sc-test -- cat /proc/1/status | grep Cap**
CapInh:	0000000000000000
CapPrm:	00000000aa0435fb
CapEff:	00000000aa0435fb
CapBnd:	00000000aa0435fb
CapAmb:	0000000000000000
```

내용이 조금 바뀌었다. capsh로 확인해보자.

```json
**# capsh --decode=00000000aa0435fb | tr ',' '\n'**
WARNING: libcap needs an update (cap=40 should have a name).
0x00000000aa0435fb=cap_chown
cap_dac_override
cap_fowner
cap_fsetid
cap_kill
cap_setgid
cap_setuid
cap_setpcap
cap_net_bind_service
**cap_net_admin**
cap_net_raw
cap_sys_chroot
**cap_sys_time**
cap_mknod
cap_audit_write
cap_setfcap
```

capability가 추가된 것을 확인할 수 있다.

## 05. readOnlyRootFilesystem 설정

컨테이너의 변조를 방지하기 위해서 모든 파일시스템을 ReadOnly로 설정한다. 

다만 로그를 생성하거나 컨테이너 내부에 pid 파일을 저장해야하는 일부 어플리케이션(웹서버, DB)에게 해당 옵션을 부여하면 컨테이너 실행에 실패할 수 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-test
  name: sc-test
spec:
  containers:
  - image: busybox
    name: sc-test
    args:
    - /bin/sh
    - -c
    - "sleep 36000"
    securityContext:
      readOnlyRootFilesystem: true
```

```json
**# kubectl exec -it sc-test -- sh**
/ # touch /home/AAA
touch: /home/AAA: Read-only file system
```
