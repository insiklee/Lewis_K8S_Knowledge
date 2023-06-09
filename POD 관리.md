# K8s Pod 관리

# 01. 명령어로 Pod 생성

### 01. 기본 명령어

<aside>
💡 **# kubectl run {pod명} --image={이미지명}**

</aside>

```json
**# kubectl run nginx --image=nginx**
pod/nginx created
**# kubectl get pod**
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          42s
```

### 02. 명령어로 pod 생성시 변수 삽입

db같은 경우 환경변수를 넣어야 하는 경우가 있는데 이 경우 아래와 같이 --env 옵션을 추가하면 된다. 해당 옵션을 추가하면 파드내 컨테이너에서 변수로 적용된다.

<aside>
💡 **# kubectl run {pod명} --image={이미지명} --env={변수 key}={변수 value}**

</aside>

```json
**# kubectl run mysql --image=mysql --env=MYSQL_ROOT_PASSWORD="P@ssw0rd!"**
pod/mysql created

**# kubectl get pod**
NAME    READY   STATUS    RESTARTS   AGE
mysql   1/1     Running   0          104s

**# kubectl exec -it mysql -- env**
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TERM=xterm
HOSTNAME=mysql
**MYSQL_ROOT_PASSWORD=P@ssw0rd!**
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
GOSU_VERSION=1.14
MYSQL_MAJOR=8.0
MYSQL_VERSION=8.0.30-1.el8
MYSQL_SHELL_VERSION=8.0.30-1.el8
HOME=/root
```

### 03. 명령어로 pod 생성시 라벨 삽입

<aside>
💡 **# kubectl run {pod명} --image={이미지명} --labels={라벨명}={라벨값}**

</aside>

```json
**# kubectl run nginx --image=nginx --labels=app=nginx**
pod/nginx created

**# kubectl get pod nginx --show-labels**
NAME    READY   STATUS    RESTARTS   AGE    LABELS
nginx   1/1     Running   0          9m3s   **app=nginx**
```

### 04. 명령어로 pod 생성시 포트넘버 지정

컨테이너 포트를 직접 설정하기 위해서는 아래 명령어를 사용한다.

<aside>
💡 **# kubectl run {pod명} --image={이미지명} --port={포트번호}**

</aside>

```json
**# kubectl run nginx-port --image=nginx --port=8080**
pod/nginx-port created

**# kubectl get pod nginx-port -o jsonpath="{.spec.containers[].ports[].containerPort}{'\n'}"**
8080
```

### 05. 명령어로 pod의 yaml파일 생성

kubectl run 명령어는 간편하지만 멀티컨테이너나 volumes, nodeSelection 같은 추가 기능을 넣을 수 없는 문제가 있다. 이경우 오로지 yaml 파일을 직접 편집하는 방법밖에 없는데, 아래와 같은 방법을 쓰면 기본 yaml파일 형식이 조금 더 쉽게 만들어진다.

<aside>
💡 **# kubectl run {pod명} --image={이미지명} --dry-run=client -o yaml > {pod명}.yaml**

</aside>

```json
**# kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx-pod.yaml
# cat nginx-pod.yaml** 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

### 06. 파드 실행과 동시에 명령어 전달

파드를 실행함과 동시에 해당 파드 내부의 컨테이너에 접속할 필요가 있을 때 아래와 같이 사용한다. 

<aside>
💡 **# kubectl run {pod명} --image={이미지명} -it -- {명령어}**

</aside>

```json
**# kubectl run centos --image=centos -it -- bash**
If you don't see a command prompt, try pressing enter.
[root@centos /]#
[root@centos /]# hostname
centos
```

다만 위와 같은 명령어로 접속을 시도할 경우 Pod가 Completed나 Eror 상태로 여전히 남아있다. 

```json
[root@centos /]# exit
exit
**Session ended, resume using 'kubectl attach centos -c centos -i -t' command when the pod is running**

**# kubectl get pod**
NAME         READY   STATUS    RESTARTS   AGE
centos       0/1     Error     0          2m28s
```

이 경우 --rm 옵션을 추가하여 자동으로 파드를 삭제하도록 해야한다.

<aside>
💡 **# kubectl run {pod명} --image={이미지명} --rm -it -- {명령어}**

</aside>

```json
**# kubectl run centos --image=centos --rm -it -- bash**
If you don't see a command prompt, try pressing enter.
[root@centos /]# exit
exit
Session ended, resume using 'kubectl attach centos -c centos -i -t' command when the pod is running
**pod "centos" deleted**

**# kubectl get pod** 
No resources found in default namespace.
```

# 02. Pod내부 컨테이너 관리

## 01. 멀티컨테이너

파드 안에는 여러개의 컨테이너가 포함될 수 있다. 

### 01. 멀티 컨테이너 구현

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multicontainer-prac
spec:
  containers:
  - image: nginx
    name: container1
  - image: mysql
    name: container2
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "123qwe.."
  - image: redis
    name: container3
```

### 02. 멀티컨테이너 파드에서 컨테이너 네트워크

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5c668f48-214c-4135-b5a0-3c181626a27f/Untitled.png)

포트 내부의 컨테이너는 자체 IP를 지니지 못하고 Pod의 IP를 공유한다.  대신 각 컨테이너는 서로 다른 Port를 열어서 대기한다.  만약 하나의 파드안에 두 개 이상의 컨테이너가 같은 port 넘버를 지니려고 할 경우, 하나의 컨테이너는 제대로 동작하지 않게 된다. **이는 Pod가 커널단위에서 Namespace로 분리된 가상 프로세스이기 때문이다.**

아래는 **멀티 컨테이너 구현** 항목에서 정의한 yaml 파일을 적용했을때 multicontainer-prac의 네임스페이스 정보다.

```json
**# lsns | egrep 'nginx|mysql|redis'**
**4026532405 uts         5 1405688 root             nginx: master process nginx -g daemon off;
4026532406 ipc         5 1405688 root             nginx: master process nginx -g daemon off;
4026532408 net         5 1405688 root             nginx: master process nginx -g daemon off;**
4026532555 mnt         3 1405688 root             nginx: master process nginx -g daemon off;
4026532561 pid         3 1405688 root             nginx: master process nginx -g daemon off;
4026532562 mnt         1 1405771 systemd-coredump mysqld
4026532564 pid         1 1405771 systemd-coredump mysqld
4026532565 mnt         1 1405926 systemd-coredump redis-server *:6379
4026532566 pid         1 1405926 systemd-coredump redis-server *:6379
```

위 항목에 대한 정보는 [프로세스 격리 (namespace)](https://www.notion.so/namespace-fe4823ca8e87413f9b5857837286ea00?pvs=21) 에서 확인 가능하다.

nginx의 네임스페이스에만 uts, ipc, net이 할당된 것을 확인할 수 있고, 같은 pod에서 동작하는 컨테이너인 mysqld와 redis-server는 mnt와 pid만 할당된 것을 확인할 수 있다. uts는 호스트네임이고 ipc는 프로세스간의 시스템콜, net은 격리된 네트워크 영역을 뜻한다. VM을 만들기 위해 꼭 필요한 요소들만 들어간 셈이다. 즉, 이 세 영역은 COMMAND가 nginx로 표기되어있으나 실제론 파드 생성을 위해 만들어진 Namespace임을 유추할 수 있다. 이렇게 Pod를 위해 격리된 공간에 각 컨테이너가 mnt로 달라붙고 pid로 프로세스 넘버를 받은 구조다. 

따라서 멀티 컨테이너 내부의 모든 컨테이너는 독립된 Namespace를 통해서 동일한 네트워크 영역에 속해 있으며, 이 때문에 컨테이너는 자신의 IP주소를 부여받지 못한다. 대신 독립된 프로세스로서(pid) 동작하게 된다. 

### 03. 사이드카 컨테이너

사이드카 컨테이너는 런타임 컨테이너와 동일한 스토리지를 공유한다.

런타임 컨테이너가 스토리지에 저장된 파일을 통해 서비스를 하는 동안, 사이드카 컨테이너는 스토리지 영역의 파일을 업데이트하거나 추가하는 역할을 한다. 

주로 로깅 작업을 할때 사이드카 컨테이너가 쓰인다.

- **사이드카 컨테이너 사용 예시**
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: sidecar-pod
    spec:
      containers:
      - image: busybox
        name: sidecar1
        args: [ /bin/sh, -c, "while :; do  echo 12345 >> /tmp/sidecartxt.txt; done" ]
        volumeMounts:
        - name: sidecar-volume
          mountPath: /tmp
      - image: busybox
        name: sidecar2
        args: [ /bin/sh, -c, "tail -f /tmp/sidecartxt.txt" ]
        volumeMounts:
        - name: sidecar-volume
          mountPath: /tmp
      volumes:
      - name: sidecar-volume
        emptyDir: {}
    ```
    
    위는 사이드카를 사용한 yaml 파일의 예시다. Shared 볼륨을 생성한 뒤 두 컨테이너에서 동시에 마운트하는 설정이 들어갔다.
    
    우선 위 YAML을 적용시킨다.
    
    ```json
    **# kubectl apply -f sidecar-pod.yaml** 
    pod/sidecar-pod created
    ```
    
    이후 sidecar1 컨테이너에서 /tmp/sidecartxt.txt가 생성됐는지 확인해보자.
    
    ```json
    **# kubectl exec -it sidecar-pod -c sidecar1 -- ls /tmp/sidecartxt.txt**
    /tmp/sidecartxt.txt
    ```
    
    이후 sidecar2 컨테이너에서 logs로 tail -f 명령어가 적용됐는지 확인한다.
    
    ```json
    **# kubectl logs sidecar-pod -c sidecar2**
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    12345
    ```
    

## 02. init 컨테이너

### 01. init 컨테이너란?

init 컨테이너는 이전 컨테이너가 실행되기 전 먼저 실행됐다가 종료되는 컨테이너를 의미한다.

컨테이너는 프로세스다. 특정 명령을 수행한 뒤 정상 종료될 경우 컨테이너는 자동으로 exited가 된다. 그런데 어떤 컨테이너의 경우에는 실행되기 전에 특정 명령을 실행하여 어플리케이션 구동을 위한 환경을 설정하거나 특정 파일을 생성해야할 필요가 있다. 

이 때문에 컨테이너 실행 시 args나  command로 사전 환경설정을 위한 명령어를 입력해줘야 한다. 다만, 컨테이너는 프로세스 단위로 실행되기 때문에 위의 명령어를 실행하면 자동으로 exited 된다. 

결국 이런 방식의 어플리케이션을 실행하기 위해서는 단일 컨테이너가 아닌 사전 설정을 위한 시작 컨테이너, 즉 init container를 생성해줘야만 한다. 

### 02. **init contaienr 예시**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

# 03. 파드 삭제

### 01. 파드 삭제

<aside>
💡 **# kubectl delete pod {pod명}**

</aside>

```json
**# kubectl get pod**
NAME                  READY   STATUS    RESTARTS   AGE
multicontainer-prac   3/3     Running   0          2d1h
nginx-labels          1/1     Running   0          23h

**# kubectl delete pod nginx-labels** 
pod "nginx-labels" deleted
```

### 02. 파드 즉시 삭제

파드를 gracefule하게 삭제할 경우 최대 30초의 시간이 걸리는 경우가 있다. 그런거 없이 즉시 삭제하고 싶다면 아래와 같이 --grace-period 옵션을 주면 된다.

<aside>
💡 **# kubectl delete pod {pod명} --grace-period=0 --force**

</aside>

```json
**# kubectl delete pod multicontainer-prac --grace-period=0 --force**
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "multicontainer-prac" force deleted
```

- **03. Running 제외 파드 삭제**
```yaml
kubectl get pods --all-namespaces | grep -E OutOfcpu\|Evicted\|OOMKilled\|Error\|ContainerStatusUnknown | awk '{print "kubectl delete po " $2 " -n " $1 }' | bash
```
