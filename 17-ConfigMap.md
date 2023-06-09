# K8s ConfigMap

# 01. ConfigMap이란?

환경설정 정보를 저장하는 오브젝트로 Key: Value 구조를 지니고 있다. 주로 환경변수, 명령인수, 구성파일(*.conf)등을 Pod에 적용하기 위해 생성한다. Secret과는 달리 외부에서 내용을 확인할 수 있도록 일반적인 텍스트 저장한다.

# 02.  Configmap 생성

따로 YAML을 통해 생성하는 것 보다는 커맨드 라인으로 생성하는 것이 더 간단하다. 

## 01. 커맨드 라인으로 생성

### 01. 명령어의 줄글로 생성

<aside>
💡 **# kubectl create configmap log-level-cm --from-literal=LOG_LEVEL=DEBUG**

</aside>

```json
**# kubectl -n cm-test create configmap nginx-cm --from-literal=alice=attack**
configmap/nginx-cm created

**# kubectl -n cm-test get cm nginx-cm** 
NAME       DATA   AGE
**nginx-cm   1      8m16s

# kubectl -n cm-test get cm nginx-cm -o yaml**
apiVersion: v1
**data:
  alice: attack**
kind: ConfigMap
metadata:
  creationTimestamp: "2022-10-03T07:21:12Z"
  name: nginx-cm
  namespace: cm-test
  resourceVersion: "43718"
  uid: 15d50145-820a-41bc-b88e-7e3fd26497eb
```

여러 키를 넣는것도 가능하다

```json
**# kubectl -n cm-test create cm nginx-cm2 --from-literal=alice=attack --from-literal=bob=defence --from-literal=chalie=looking**
configmap/nginx-cm2 created

**# kubectl -n cm-test get cm nginx-cm2 -o yaml**
apiVersion: v1
**data:
  alice: attack
  bob: defence
  chalie: looking**
kind: ConfigMap
metadata:
  creationTimestamp: "2022-10-03T07:31:08Z"
  name: nginx-cm2
  namespace: cm-test
  resourceVersion: "44685"
  uid: 286b0fd9-7304-494a-b160-3a3908bde8a5
```

### 02. 파일을 통해 생성

configMap가 config 파일을 포함하는 경우 여러 줄글이 포함되어 있어야 하는 경우가 있다. 이 경우 config가 저장된 파일을 미리 준비해놓고 통째로 불러올 수 있다.

<aside>
💡 **# kubectl create cm {configmap 명} --from-file={파일경로}**

</aside>

```json
# cat nginx.conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;
events {
	worker_connections 768;
}
http {
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	ssl_prefer_server_ciphers on;
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
	gzip on;

}

**# kubectl -n cm-test create cm nginx-config --from-file=nginx.conf** 
configmap/nginx-config created

**# kubectl -n cm-test get cm nginx-config -o yaml**
apiVersion: v1
**data:
nginx.conf: "user www-data;\nworker_processes auto;\npid /run/nginx.pid;\nevents
    {\n\tworker_connections 768;\n}\nhttp {\n\tsendfile on;\n\ttcp_nopush on;\n\ttcp_nodelay
    on;\n\tkeepalive_timeout 65;\n\ttypes_hash_max_size 2048;\n\tssl_prefer_server_ciphers
    on;\n\taccess_log /var/log/nginx/access.log;\n\terror_log /var/log/nginx/error.log;\n\tgzip
    on;\n}\n\n"**
kind: ConfigMap
metadata:
  creationTimestamp: "2022-10-03T07:43:49Z"
  name: nginx-config
  namespace: cm-test
  resourceVersion: "45869"
  uid: dd049581-b1b4-41d2-90db-3662eb3b9a11
```

# 03. ConfigMap 적용

## 01. ConfigMap 내용을 파일로 저장(volumeMount)

마운트 기법으로 진행하면 key값이 파일 명, value값이 파일내용으로 컨테이너에 포함된다.
주로 config 정보를 파일단위로 넣을때 필요한 방식이다.

위[02. 파일을 통해 생성](https://www.notion.so/02-3bf616182e4242d39365618642a7e0a8?pvs=21) 에서 만든 nginx.conf를 예시로 해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-config
  name: nginx-config
  namespace: cm-test
spec:
  containers:
  - image: nginx
    name: nginx-config
    **volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx**
  **volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-confi**
```

```json
**# kubectl apply -f nginx-config-pod.yaml** 
pod/nginx-config created

# **kubectl -n cm-test get pod**
NAME           READY   STATUS             RESTARTS   AGE
nginx-config   1/1     Running            0          6s

**# kubectl -n cm-test exec -it nginx-config -- cat /etc/nginx/nginx.conf**
user www-data;
worker_processes auto;
pid /run/nginx.pid;
events {
	worker_connections 768;
}
http {
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	ssl_prefer_server_ciphers on;
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
	gzip on;
}
```

### 02. envFrom 방식

참조기법으로 사용하면 env형식으로 컨테이너의 환경설정으로 적용된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-pod1
spec:
  containers:
  - image: nginx:1.21-alpine
    name: cm-container1
    **envFrom:
    - configMapRef:
        name: log-level-cm**
```

## 02. ConfigMap 전체 키 환경변수로 저장(envFrom)

[01. 명령어의 줄글로 생성](https://www.notion.so/01-fac1a7d51c174eaf8b8365c0adec5bbc?pvs=21) 에서 생성한 nginx-cm2 컨피그맵으로 실습해본다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-envfrom
  name: nginx-envfrom
  namespace: cm-test
spec:
  containers:
  - image: nginx
    name: nginx-envfrom
    **envFrom:
    - configMapRef:
        name: nginx-cm2**
```

```json
**# kubectl apply -f nginx-envfrom.yaml** 
pod/nginx-envfrom created

**# kubectl -n cm-test get pod**
NAME            READY   STATUS         RESTARTS   AGE
**nginx-envfrom   1/1     Running        0          5s

# kubectl -n cm-test exec -it nginx-envfrom -- env**
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-envfrom
NGINX_VERSION=1.23.1
NJS_VERSION=0.7.6
PKG_RELEASE=1~bullseye
**alice=attack
bob=defence
chalie=looking**
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
TERM=xterm
HOME=/root
```

## 03. ConfigMap 특정 키 환경변수로 저장 (valueFrom)

컨피그맵의 일부 key만 가져와서 컨테이너의 환경변수로 지정할 수 있다. 이 경우 변수 명을 직접 지정할 수 있다. [01. 명령어의 줄글로 생성](https://www.notion.so/01-fac1a7d51c174eaf8b8365c0adec5bbc?pvs=21) 에서 생성한 nginx-cm2 컨피그맵으로 실습해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-valuefrom
  name: nginx-valuefrom
  namespace: cm-test
spec:
  containers:
  - image: nginx
    name: nginx-valuefrom
    **env:
    - name: TEST_ENV_1
      valueFrom:
        configMapKeyRef:
          name: nginx-cm2
          key: alice
    - name: TEST_ENV_2
      valueFrom:
        configMapKeyRef:
          name: nginx-cm2
          key: bob**
```

```json
**# kubectl apply -f nginx-valuefrom.yaml** 
pod/nginx-valuefrom created

**# kubectl -n cm-test get pod**
NAME              READY   STATUS             RESTARTS   AGE
**nginx-valuefrom   1/1     Running            0          5s

# kubectl -n cm-test exec -it nginx-valuefrom -- env**
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-valuefrom
NGINX_VERSION=1.23.1
NJS_VERSION=0.7.6
****PKG_RELEASE=1~bullseye
**TEST_ENV_1=attack
TEST_ENV_2=defence**
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
TERM=xterm
HOME=/root
```
