# K8s Container HealthCheck (LivenessProbe, ReadinessProbe, StartupProbe)

[Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)

# 01. 컨테이너 의 상태 검증(Health Check)

## 01. 상태 검증의 필요성

직접 컨테이너 이미지를 커스텀한 nginx 파드가 실행됐다. kubectl get pod 명령어로 확인하면 pod의 상태는 running으로 표기되어 문제가 없어 보인다. 웹서버 런타임은 그럭저럭 실행된 것 같다. 그런데 정작 nginx 웹 서비스에 접속을 시도하면 접속이 안된다. 이 경우 해당 pod가 정상 작동한다고 볼 수 있을 까?

Pod의 컨테이너는 프로세스단위로 실행되기 때문에 컨테이너 이미지에서 최초 정의한 command가 정삭 작동하기만 하면 kubelet은 해당 pod가 running상태라고 인식한다. 하지만 우리가 원하는 것은 프로세스 관점의 작동여부가 아니다. **서비스 관점에서 목표로 하는 서비스를 제공할 수 있는지를 확인해야 한다.** 이 때문에 컨테이너별 상태 검증이 필요할 수 있다.

## 02. 헬스 체크 종류

### 01. liveness probe

- 프로세스가 정상 가동되고 있는지를 검증한다.
- 프로세스가 데드락 상태에 빠져서 어플리케이션을 실행되나, 아무런 작업이 진행되지 않을때 상태를 캐치한다.
- 위와 같은 상황이 발생할때 kubelet이 컨테이너를 재시작시킨다.

### 02. readiness probe

- 컨테이너가 트래픽을 받아들일 준비가 됐는지 검증한다.
- 주로 백엔드나 외부 서비스와의 통신여부를 체크한다.
- 컨테이너가 실행되는 동안 계속 동작한다.
- liveness probe와 달리 컨테이너를 재시작하지 않는다. 다만 서비스에서 endpoints 목록에서 제외된다.

### 03. startup probe

- 컨테이너가 제대로 시작했는지를 검증한다.
- 컨테이너가 정상 가동하기까지 시간이 걸릴 수 있는데, 이때 liveness probe나 readiness probe가 동작할 경우 서비스에 문제가 발생할 수 있다.
- 위 두 검증방식은 startup probe가 성공해야지 실행된다.
- startup probe가 실패할 경우 kubelet이  컨테이너를 재시작한다.

# 02. Liveness Probe 사용

[Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command)

## 01. command로 검증

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: liveness-exec
  name: liveness-exec
spec:
  containers:
  - image: busybox
    name: liveness-exec
    args:
    - /bin/sh
    - -c
    - "touch /tmp/healthy; while true; do sleep 1; done"
    **livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3**
```

cat 명령어로 /tmp/healthy 파일을 읽을 수 있다면 검증 성공이다. 다른 파라미터의 정보는 아래와 같다.

- initialDelaySeconds : 컨테이너가 처음 생성되고 liveness probe를 진행하기 전에 잠시 기다리는 시간이다. 위 예시는 컨테이너 생성 후 5초간 기다린다는 뜻이다.
- periodSeconds : 설정된 값만큼 간격을 두고 liveness 를 검증한다.
- failureThreshold: 한번 실패했다고 재시작하면 정없다. 여기에 설정된 값만큼 실패해야지 컨테이너를 재시작한다.

위 설정대로면 최초 5초간은 검증을 하지 않았다가, 이후 10초 간격으로 검증을 시도한다.

만약 실패상황이 발생했다면, 10초간 3번, 총 30초간 반복 검증 후, 상태가 그대로면 컨테이너를 재시작한다.

- **검증**
    
    위 yaml을 실행시켜보자.
    
    ```json
    **# kubectl apply -f liveness-exec.yaml** 
    pod/liveness-exec created
    
    **# kubectl get pod**
    NAME            READY   STATUS    RESTARTS   AGE
    liveness-exec   1/1     Running   0          14m
    ```
    
    그 후 /tmp/healthy 파일을 삭제한다.
    
    ```json
    **# kubectl exec -it liveness-exec -- rm -f /tmp/healthy**
    ```
    
    30초간 기다렸다가 pod 상태를 확인한다.
    
    ```json
    **# kubectl get pod**
    NAME            READY   STATUS    RESTARTS      AGE
    liveness-exec   1/1     Running   **1 (17s ago)**   16m
    ```
    
    restarts가 추가된 것을 확인 할 수 있다.
    
    /tmp/healthy 파일이 다시 생성됐는지 확인해보자.
    
    ```json
    **# kubectl exec -it liveness-exec -- ls /tmp**
    healthy
    ```
    

## 02. httpGet으로 검증

httpGet은 웹 서비스의 특정 경로에 HTTP GET 요청을 지속적으로 보내서 정상적인 접속이 가능한지 체크하는 기능이다. HTTP 응답이 200이상 400 미만으로 나오면 정상처리된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: liveness-httpget
  name: liveness-httpget
spec:
  containers:
  - image: httpd
    name: liveness-httpget
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
```

위 예시는 httpd 웹 서버에서 80번 포트로 / 경로에 접속이 가능한지 여부를 체크한다.

- **검증**
    
    ```json
    **# kubectl apply -f liveness-httpget.yaml** 
    pod/liveness-httpget created
    
    **# kubectl get pod -o wide**
    NAME               READY   STATUS    RESTARTS      AGE   IP               NODE          NOMINATED NODE   READINESS GATES
    liveness-httpget   1/1     Running   0             13s   10.100.194.83    k8s-worker1   <none>           <none>
    
    **# curl 10.100.194.83**
    <html><body><h1>It works!</h1></body></html>
    ```
    
    웹 서버의 루트 디렉터리를 삭제해보자.
    
    ```json
    **# kubectl exec -it liveness-httpget -- rm -rf /usr/local/apache2/htdocs
    
    # curl 10.100.194.83**
    <!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
    <html><head>
    <title>**403** Forbidden</title>
    </head><body>
    <h1>Forbidden</h1>
    <p>You don't have permission to access this resource.</p>
    </body></html>
    ```
    
    403 Forbidden 응답을 받았다. 이후 잠시 시간이 지난 뒤 Pod 상태를 확인해보자
    
    ```json
    **# kubectl get pod**
    NAME               READY   STATUS    RESTARTS      AGE
    liveness-httpget   1/1     Running   **1 (13s ago)**   9m4s
    ```
    
    컨테이너가 1회 리스타트된 것을 알 수 있다.
    

## 03. TCP로 검증

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: liveness-tcp
  name: liveness-tcp
spec:
  containers:
  - image: nginx
    name: liveness-tcp
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 15
```

## 04. gRPC로 검증

gRPC는 구글에서 만든 Remote Procedure Call 프레임 워크다. 서비스간 RPC 통신이 원할하게 이뤄지고 있는지를 검증하는데 사용할 수 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd-with-grpc
spec:
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.1-0
    command: [ "/usr/local/bin/etcd", "--data-dir",  "/var/lib/etcd", "--listen-client-urls", "http://0.0.0.0:2379", "--advertise-client-urls", "http://127.0.0.1:2379", "--log-level", "debug"]
    ports:
    - containerPort: 2379
    livenessProbe:
      grpc:
        port: 2379
      initialDelaySeconds: 10
```

# 03. Readiness Probe 사용

## 01. command로 검증

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: readiness-exec
  name: readiness-exec
spec:
  containers:
  - image: busybox
    name: readiness-exec
    args:
    - /bin/sh
    - -c
    - "touch /tmp/healthy; while true; do sleep 1; done"
    ports:
    - containerPort: 9999
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 10
```

- **검증**
    
    위 YAML을 실행하고 service와 연결한다.
    
    ```yaml
    **# kubectl apply -f readiness-exec.yaml** 
    pod/readiness-exec created
    
    **# kubectl get pod -o wide**
    NAME               READY   STATUS    RESTARTS       AGE    IP               NODE          NOMINATED NODE   READINESS GATES
    readiness-exec     1/1     Running   0              11s    10.100.194.65    k8s-worker1   <none>
    
    **# kubectl expose pod readiness-exec --port=9999 --target-port=9999**
    service/readiness-exec exposed
    
    **# kubectl describe svc readiness-exec | grep Endpoints**
    Endpoints:         10.100.194.65:9999
    ```
    
    이제 컨테이너의 healthy 파일을 삭제해보자
    
    ```yaml
    **# kubectl exec -it readiness-exec -- rm -f /tmp/healthy
    
    # kubectl describe svc readiness-exec | grep Endpoints
    Endpoints:
    
    # kubectl get pod
    NAME               READY   STATUS    RESTARTS       AGE
    readiness-exec     0/1     Running   0              8m10s**
    ```
    
    서비스의 EndPoints에서 사라진 것을 확인할 수 있다. Pod 역시 실행중이나 Ready 상태가 아닌 것을 알 수 있다.
    

그 외 httpGet, TCP, gRPC 모두 Liveness Prove와 동일한 형식이다. 고로 귀찮으니 생략한다.

# 04. Startup Probe 사용

## 01. livenessProbe와 함께 사용

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: startup-httpget
  name: startup-httpget
spec:
  volumes:
  - name: probe-dir
    emptyDir: {}
  containers:
  - image: httpd
    name: startup-httpget
    mountVolumes:
    - name: probe-dir
      mountPath: /usr/local/apache2/htdocs
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
    **startupProbe:
      httpGet: 
        path: /health
        port: 80
      periodSeconds: 60**
  - image: busybox
    name: container-to-make-check-dir
    **args:
    - /bin/sh
    - -c
    - "sleep 60; mkdir /usr/local/apache2/htdocs/health"**
    mountVolumes:
    - name: probe-dir
      mountPath: /usr/local/apache2/htdocs
```

위 예시를 보자.  service-container 컨테이너에서 /heath 경로로 주기적으로 http 통신이 가능한지 liveness로 체크하고 있다.

다른 컨테이너인 contaienr-to-make-check-dir에서는 최초 실행 후 60초간 기다렸다가 /health 경로를 생성하는 것을 확인할 수 있다.

위 구성대로라면 service-container의 livenessProbe.initailDelaySeconds에 정해진 값 10에 의해 10초 뒤 곧바로 liveness 체크를 시작해야하는데, /health 경로가 60초뒤에 생성되기 때문에 livenessProbe는 fail 상태로 인식하게 된다.

이를 대비하기 위해 startupProbe 옵션을 추가하여, /health 경로가 생성될때까지 60초간 기다리도록 설정했다.

위 YAML을 실행하자

```json
**# kubectl apply -f startup-httpget.yaml** 
pod/startup-httpget created

# kubectl get pod -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
startup-httpget   1/2     Running   0          11s   10.100.194.106   k8s-worker1   <none>           <none>
```

처음 YAML을 실행하고 나면 한동안 READY가 1/2로 표기된다. container-to-make-check-dir 컨테이너가 먼저 실행되고, service-container가 startupProbe를 완료할때까지 기다리기 때문이다.

```json
**# kubectl get pod startup-httpget -o jsonpath='{.status.containerStatuses[?(@.name=="container-to-make-check-dir")].ready}{"\n"}'**
true

**# kubectl get pod startup-httpget -o jsonpath='{.status.containerStatuses[?(@.name=="service-container")].ready}{"\n"}'**
false

**# kubectl describe pod startup-httpget**
(전략)
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  119s  default-scheduler  Successfully assigned default/startup-httpget to k8s-worker1
  Normal   Pulling    118s  kubelet            Pulling image "httpd"
  Normal   Pulled     116s  kubelet            Successfully pulled image "httpd" in 2.120176832s
  Normal   Created    116s  kubelet            Created container service-container
  Normal   Started    116s  kubelet            Started container service-container
  Normal   Pulling    116s  kubelet            Pulling image "busybox"
  Normal   Pulled     113s  kubelet            Successfully pulled image "busybox" in 2.196060148s
  Normal   Created    113s  kubelet            Created container container-to-make-check-dir
  Normal   Started    113s  kubelet            Started container container-to-make-check-dir
  **Warning  Unhealthy  59s   kubelet            Startup probe failed: HTTP probe failed with statuscode: 404**
```

시간이 지나면 startupProbe가 검증을 완료하고 pod가 정상적으로 READY 상태가 된다.

```json
**# kubectl get pod**
NAME              READY   STATUS    RESTARTS   AGE
startup-httpget   **2/2**     Running   0          2m34s

**# kubectl get pod startup-httpget -o jsonpath='{.status.containerStatuses[?(@.name=="service-container")].ready}{"\n"}'**
true

**# curl 10.100.194.78/health**
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>**301 Moved Permanently**</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://10.100.194.78/health/">here</a>.</p>
</body></html>
```

readinessProbe와 함께 사용하는 것도 livenessProbe와 동일한 방식이다. 고로 귀찮으니 생략한다.
