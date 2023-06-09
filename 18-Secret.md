# K8s Secret

# 01. Secret이란?

Configmap과 모든것이 동일하나 Secret의 경우  base64로 인코딩하여 내용을 즉각적으로 확인할 수 없도록 숨기면서 보안성을 조금이라도 높인 리소스다.

# 02. Secret 생성

따로 yaml을 통해 생성하는 것 보다는 커맨드 라인으로 생성하는 것이 더 간편하다. data의 내용을 일일히 base64로 인코딩 하는 것은 번거롭기 때문이다. 따라서 YAML을 통한 생성은 생략한다. 

## 01. 커맨드 라인으로 생성

### 01. 명령어의 줄글로 생성

명령어로 직접 key와 value를 입력해서 생성한다.

<aside>
💡 **# kubectl create secret generic {secret 명} --from-literal={key}={value}**

</aside>

```json
**# kubectl create secret generic super-secret --from-literal=password=bob**
secret/super-secret created

**# kubectl get secret** 
NAME           TYPE     DATA   AGE
super-secret   Opaque   1      7m54s

**# kubectl get secret super-secret -o yaml**
apiVersion: v1
**data:
  password: Ym9i**
kind: Secret
metadata:
  creationTimestamp: "2022-09-28T05:43:35Z"
  name: super-secret
  namespace: default
  resourceVersion: "244972"
  uid: 9385f993-a6e0-457b-b845-d5100415bc15
type: Opaque
```

**여러개의 키를 넣는 것도 가능하다.**

```json
# **kubectl create secret generic dessert --from-literal=key1=macaron --from-literal=key2=ice-cream --from-literal=key3=coffee --from-literal=key4=cake**
secret/dessert created

# **kubectl get secret dessert -o yaml**
apiVersion: v1
**data:
  key1: bWFjYXJvbg==
  key2: aWNlLWNyZWFt
  key3: Y29mZmVl
  key4: Y2FrZQ==**
kind: Secret
metadata:
  creationTimestamp: "2022-09-28T06:25:02Z"
  name: dessert
  namespace: default
  resourceVersion: "248870"
  uid: f9df45a6-6219-4089-a6dd-e6b0251613d9
type: Opaque
```

### 02. 파일을 통해 생성

파일에 key와 value를 미리 저장해 놓은 뒤 secret을 생성할때 불러온다. 이 때 파일명이 key가 되고 파일 내용은 value가 된다.

<aside>
💡 **# kubectl create secret generic {secret 명} --from-file={파일명}**

</aside>

```json
**# cat fruits.txt** 
key1=apple
key2=pear
key3=melon
key4=lemon
key5=carrot
key6="key5 is not fruits"

# **kubectl create secret generic fruits --from-file=./fruits.txt** 
secret/fruits created

**# kubectl get secret fruits -o yaml**
apiVersion: v1
**data:
  fruits.txt: a2V5MT1hcHBsZQprZXkyPXBlYXIKa2V5Mz1tZWxvbgprZXk0PWxlbW9uCmtleTU9Y2Fycm90CmtleTY9ImtleTUgaXMgbm90IGZydWl0cyIK**
kind: Secret
metadata:
  creationTimestamp: "2022-09-28T06:01:10Z"
  name: fruits
  namespace: default
  resourceVersion: "246645"
  uid: 5813e589-e433-4bb1-9530-d1bed378fe6c
type: Opaque
```

# 03. Secret 적용

## 01. Secret 내용을 파일로 저장 (volumeMount)

Secret을 Volume으로 취급하여 Pod에서 Mount 한다면 시크릿의 내용을 파일로 저장할 수 있다.  이 경우 key는 파일명이 되고, value가 파일 내용이 된다. 환경설정 파일 등을 적용할 때 사용한다.

위 예시에서 만든 secret/fruits 를 예시로 해보자.

**YAML 예시**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: fruits-secret-pod
  name: fruits-secret-pod
spec:
  containers:
  - image: nginx
    name: fruits-secret-pod
    volumeMounts:
    - name: fruits-secret-file
      mountPath: /fruits
  volumes:
  - name: fruits-secret-file
    secret:
      secretName: fruits
```

```json
**# kubectl apply -f fruits-secret-pod.yaml** 
pod/fruits-secret-pod created

**# kubectl get pod**
NAME                         READY   STATUS    RESTARTS   AGE
fruits-secret-pod            1/1     Running   0          3m9s

**# kubectl exec -it fruits-secret-pod -- ls /fruits**
fruits.txt

**# kubectl exec -it fruits-secret-pod -- cat /fruits/fruits.txt**
key1=apple
key2=pear
key3=melon
key4=lemon
key5=carrot
key6="key5 is not fruits"
```

## 02. Secret 전체 키 환경변수로 저장 (envFrom)

시크릿에 포함된 모든 key와 value를 컨테이너의 환경변수로 저장할 수 있다.

위 예시의 secret/dessert를 예시로 해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: dessert-secret-allevn-pod
  name: dessert-secert-allenv-pod
spec:
  containers:
  - image: nginx
    name: dessert-secret-allenv-pod
    **envFrom:
    - secretRef:
        name: dessert**
```

```json
**# kubectl apply -f dessert-secret-allenv-pod.yaml** 
pod/dessert-secret-allenv-pod created

# **kubectl exec -it dessert-secret-allenv-pod -- env**
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=dessert-cm-pod
NGINX_VERSION=1.23.1
NJS_VERSION=0.7.6
PKG_RELEASE=1~bullseye
**key1=macaron
key2=ice-cream
key3=coffee
key4=cake**
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

## 03. Secret의 특정 키 환경변수로 저장 (valueFrom)

시크릿의 일부 key만 가져와서 컨테이너의 환경변수로 저장할 수 있다. 이 경우 변수 명을 직접 지정할 수 있다. 

위 예시의 secret/dessert를 예시로 해보자.

**YAML 예시**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: dessert-secret-pod
  name: dessert-secret-pod
spec:
  containers:
  - image: nginx
    name: dessert-secret-pod
    **env:
    - name: FIRST_DAY_DESSERT <- 변수명을 secret의 key가 아닌 다른 값으로 정할 수 있다.
      valueFrom:
        secretKeyRef:
          name: dessert
          key: key1
    - name: SECOND_DAY_DESSERT
      valueFrom:
        secretKeyRef:
          name: dessert
          key: key1
    - name: THIRD_DAY_DESSERT
      valueFrom:
        secretKeyRef:
          name: dessert
          key: key1**
```

```json
**# kubectl apply -f dessert-secret-pod.yaml** 
pod/dessert-secret-pod created

**# kubectl get pod**
NAME                         READY   STATUS    RESTARTS   AGE
dessert-secret-pod           1/1     Running   0          47s

**# kubectl exec -it dessert-secret-pod -- env | grep DESSERT**
SECOND_DAY_DESSERT=ice-cream
THIRD_DAY_DESSERT=coffee
FIRST_DAY_DESSERT=macaron
```
