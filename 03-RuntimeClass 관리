# K8s RuntimeClass 관리 (gVisor)

[Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/)

# 01. RuntimeClass란?

## 01. 런타임?

런타임이란  컨테이너를 실행시키기 위해 Host에 설치된 CRI(Container Runtime Interface)를 의미한다. 

쿠버네티스 클러스터를 처음 설치할 때 containerd나 crio가 먼저 설치되어야 했다. 1.24버전 이전에는 도커가 실행되고 있어야만 클러스터 설치가 가능했다. 여기서 containerd랑 crio, docker가 컨테이너 실행을 위한 런타임이다.

## 02. RuntimeClass

쿠버네티스에서 사용하는 runtimeClass는 기본적으로 클러스터 설치에 사용된 CRI를 따라간다. containerd를 사용했다면 모든 pod는 기본 runtimeHandler에 의해 containerd를 runtime으로 사용한다.

그런데 일부 pod의 경우 보안 목적으로 조금 다른 런타임을 사용해야하는 경우가 있다.  이 때 해당 pod를 기본 CRI가 아닌 별도의 런타임으로 실행할 수 있도록 설정해야한다. 

## 03. 제약사항

RuntimeClass를 하기 위해서 각 노드에 추가 CRI를 설치할 때, 모든 노드에 동일하게 CRI가 설치되어야 한다.

# 02. RuntimeClass 사용

## 01. YAML 파일 생성

RuntimeClass의 YAML 형식은 아래와 같다.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: myclass 
handler: myconfiguration
```

RuntimeClass 리소스는 spec 대신에 handler 항목이 있어서 어떤 런타임을 사용할지 결정한다. handler에는 CRI 이름을 명시해준다.

## 02. Pod에 적용

pod의 spec에 runtimeClassName을 지정해준다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  **runtimeClassName: myclass**
  containers:
  - image: nginx
    name: nginx
```

# 03. runsc 런타임 사용(gVisor)

[What is gVisor?](https://gvisor.dev/docs/)

## 01. runsc란?

**RuntimeClass는 사실 runsc를 사용하기 위해서 만들어진 리소스와 다름없다.**

### 01. 기존 runc의 문제

컨테이너는 기존 VM보다 비교적 가볍다는 장점이 있으나, 호스트의 커널을 그대로 사용하고 있다는 단점이 있다. 실제로 임시로 pod를 생성하여 커널 정보를 확인해보면 host OS의 커널 버전을  그대로 사용하는 것을 알 수 있다.

```json
**# kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh**
If you don't see a command prompt, try pressing enter.
/ # uname -r
**5.15.0-52-generic**

## **다른세션에서**
# **kubectl get pod test-pod -o wide**
NAME       READY   STATUS    RESTARTS   AGE    IP               NODE          NOMINATED NODE   READINESS GATES
test-pod   1/1     Running   0          114s   10.100.100.199   **k8s-worker3**   <none>           <none>

## **k8s-worker3 노드에서
# uname -r
5.15.0-52-generic**
```

이 경우 해커에 의해서 pod가 탈취될 경우 호스트의 커널까지 위험해질 수 있다는 뜻이다.

### 02. runsc의 해결

runsc는 VM과 seccomp, AppArmor의 장점을 혼합하여 Host의 커널과 컨테이너 간에 이격을 준다. 일종의 VM과 같은 환경을 제공하여 sandbox 형식으로 컨테이너를 실행하게 된다. 커널에 의해 프로세스 형태로 실행되는 기존 runc 런타임과는 달리 컨테이너와 호스트 OS 사이에 강력한 분리 독립이 이뤄진다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2d53d904-0ca4-4933-8290-d42011e01513/Untitled.png)

gVisor의 runsc가 Host의 커널과 컨테이너 사이에 위치한다. runsc는 컨테이너에게는 마치 VM의 커널처럼 작동하는 한편, Host의 커널에게는 제한된 시스템 콜만 요쳥할 수 있다.

## 02. runsc 설치

**클러스터에 포함된 모든 노드에서 진행해야 한다.**

### 01. runsc 다운로드 및 설치

```json
(
  set -e
  ARCH=$(uname -m)
  URL=https://storage.googleapis.com/gvisor/releases/release/latest/${ARCH}
  wget ${URL}/runsc ${URL}/runsc.sha512 \
    ${URL}/containerd-shim-runsc-v1 ${URL}/containerd-shim-runsc-v1.sha512
  sha512sum -c runsc.sha512 \
    -c containerd-shim-runsc-v1.sha512
  rm -f *.sha512
  chmod a+rx runsc containerd-shim-runsc-v1
  sudo mv runsc containerd-shim-runsc-v1 /usr/local/bin
)
```

### 02. containerd에 runsc 플러그인 추가

```json
# mkdir /etc/contaienrd
# cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2
[plugins."io.containerd.runtime.v1.linux"]
  shim_debug = true
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
EOF
```

## 03. runtimeClass 적용

### 01. runtimeClass 생성

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
**handler: runsc**
```

```json
**# kubectl apply -f runtimeClass-runsc.yaml** 
runtimeclass.node.k8s.io/gvisor created

**# kubectl get runtimeclass**
NAME     HANDLER   AGE
**gvisor   runsc     100s**
```

### 02. Pod에 적용

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: runsc-test
  name: runsc-test
spec:
  **runtimeClassName: gvisor**
  containers:
  - image: busybox
    name: runsc-test
    args:
    - /bin/sh
    - -c
    - "sleep 3600"
```

```json
**# kubectl apply -f runsc-test.yaml** 
pod/runsc-test created

**# kubectl get pod runsc-test -o jsonpath='{.spec.runtimeClassName}'**
gvisor

**# kubectl get pod -o wide**
NAME         READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
runsc-test   1/1     Running   0          50s   10.100.100.200   **k8s-worker3**   <none>           <none>
```

### 03. runsc 적용 확인

실행중인 pod의 커널 정보를 확인해보자

```json
**# kubectl exec -it runsc-test -- uname -r**
4.4.0

**# ssh student@k8s-worker3 -- uname -r**
student@k8s-worker3's password: 
5.15.0-52-generic
```

pod의 커널 정보와 실행된 host의 커널 정보가 다른 것을 확인할 수 있다.
