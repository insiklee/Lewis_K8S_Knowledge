# K8s EncryptionConfiguration 관리

[Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

# 01. EncryptionConfiguration이란?

## 01. 개요

etcd 내부에 리소스 정보를 저장할 때 외부 노출을 최소화해야할 리소스에 대해서 암호화해서 저장할 수 있도록 하는 리소스다.

## 02. 기존 시크릿의 문제점

기존 etcd에 저장된 시크릿은 평문으로 저장되기 때문에 외etcd의 인증서와 키를 탈취당했을때, 클러스터 내부의 시크릿 정보를 쉽게 알아낼 수 있다.

아래 예시를 보자.

### 01. etcd로 시크릿 내용 확인

일단 시크릿 한개 만들어 보자.

```json
**# kubectl create secret generic secret-noencrypt --from-literal=canuseeit="yes"**
secret/secret-noencrypt created
```

처음 시크릿을 생성하면 base64로 인코딩되어서 저장되기 때문에 get 명령어로 내용을 한눈에 확인하기 힘들다.

```json
**# kubectl get secrets secret-noencrypt -o yaml**
apiVersion: v1
data:
  **canuseeit: eWVz**
kind: Secret
metadata:
  creationTimestamp: "2022-10-25T09:09:31Z"
  name: secret-noencrypt
  namespace: default
  resourceVersion: "4164169"
  uid: 39618330-036c-44da-a6e3-a6e1c66e3b05
type: Opaque
```

하지만 etcd로 api를 호출하여 시크릿을 확인하면 어떻게 될까?

테스트를 위해 etcd-client 패키지를 설치하자.

```json
**# apt install etcd-client**
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following package was automatically installed and is no longer required:
  libsysmetrics1
Use 'apt autoremove' to remove it.
The following NEW packages will be installed:
  etcd-client
0 upgraded, 1 newly installed, 0 to remove and 9 not upgraded.
Need to get 4,572 kB of archives.
After this operation, 17.2 MB of additional disk space will be used.
Get:1 http://kr.archive.ubuntu.com/ubuntu focal-updates/universe amd64 etcd-client amd64 3.2.26+dfsg-6ubuntu0.1 [4,572 kB]
Fetched 4,572 kB in 3s (1,595 kB/s)      
Selecting previously unselected package etcd-client.
(Reading database ... 162812 files and directories currently installed.)
Preparing to unpack .../etcd-client_3.2.26+dfsg-6ubuntu0.1_amd64.deb ...
Unpacking etcd-client (3.2.26+dfsg-6ubuntu0.1) ...
Setting up etcd-client (3.2.26+dfsg-6ubuntu0.1) ...
Processing triggers for man-db (2.9.1-1) ...
```

이후 etcd 서버와 통신 여부를 확인해보자.

<aside>
💡 **# ETCDCTL_API=3 etcdctl \ 
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key \
--endpoints {etcd 서버 주소} \
endpoint health**

</aside>

```json
**# ETCDCTL_API=3 etcdctl \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key \
--endpoints 192.168.122.10:2379 \
endpoint health**
192.168.122.10:2379 is healthy: successfully committed proposal: took = 2.896606ms
```

통신이 잘 됐다면 secret 정보를 가져와보자.

<aside>
💡 **# ETCDCTL_API=3 etcdctl \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key \
--endpoints {etcd 서버 주소} \
get /registry/secrets/{네임스페이스명}/{시크릿명}**

</aside>

```json
**# ETCDCTL_API=3 etcdctl \
> --cacert /etc/kubernetes/pki/etcd/ca.crt \
> --cert /etc/kubernetes/pki/etcd/server.crt \
> --key /etc/kubernetes/pki/etcd/server.key \
> --endpoints 192.168.122.10:2379 \
> get /registry/secrets/default/secret-noencrypt**
/registry/secrets/default/secret-noencrypt
k8s

v1Secretف
¾ 
secret-noencryptdefault"*$39618330-036c-44da-a6e3-a6e1c66e3b052̏ޚzf
kubectl-createUpdatev̏ޚFieldsV1:2
0{"f:data":{".":{},"f:canuseeit":{}},"f:type":{}}B 
	**canuseeityes**Opaque"
```

etcd로 확인해보니 시크릿의 키와 값이 평문으로 제공되고 있는것을 알 수 있다.

# 02. EncryptionConfiguration 적용

## 01. config 파일 설명

Encryption Configuration은 쿠버네티스의 리소스 형태로 제공되는 것이 아니라, API 서버의 파라미터 형태로 전달되어야 한다. 리소스 그룹 자체가 apiserver.config.k8s.io/v1에 속해있기 때문이다.

이를 위해 apiserver에서 참고할 config 파일을 생성해야만 한다.

### 01. config 파일 기본 양식

config 파일의 양식은 아래와 같다.

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - identity: {}
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - secretbox:
          keys:
            - name: key1
              secret: YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY=
```

- .resources.resources : 암호화에 사용할 리소스를 정의한다. 우리는 secret을 암호화 할 것이기 때문에 secret을 정의한다.
- .resources.providers : 시크릿 암호화에 사용할 알고리즘과 키를 정의한다. name은 key1, key2로 키의 번호를 지정하는 것이 좋으며, s**ecret은 암호화에 사용할 키 값으로 패턴이 없는 난수를 base64로 인코딩해서 저장한다.**

### 02. encryption provider

- identity : 시크릿을 암호화하지 않을 경우 사용한다. 첫번째 provider로 지정하면 시크릿은 암호화되지 않는다.
- secretbox : 32바이트의 암호화 알고리즘으로 높은 수준의 보안이 필요할 때 사용한다.
- aesgcm : 가장 빠른 암호화 알고리즘이지만 키 교환 스킴이 실행중이지 않다면 권장하지 않는다.
- aescbc : CBC 블록 암호화 방식에 취약점이 많기 때문에 사용을 권장하지 않는다.
- kms : 써드파티 key rotation 툴을 사용한다면 사용한다. 매우 권장되는 알고리즘이다.

## 02. config 파일 생성

### 01. yaml 파일 생성

아래와 같이 config 파일을 생성하자.

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - secretbox:
      keys:
      - name: key1
        secret: <나중에 채워 넣자>
  - identity: {}
```

### 02. 암호화 키 생성

암호화에 사용할 키를 생성한 뒤 base64로 인코딩하자.

<aside>
💡 **# head -c 32 /dev/urandom | base64**

</aside>

```json
# head -c 32 /dev/urandom | base64
SDWOdnqiKNS/tMr2ThWQXiFUJJxDXRXJt/N+1I9Ocng=
```

생성된 base64 값을 yaml파일에 포함하자

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - secretbox:
      keys:
      - name: key1
        **secret: SDWOdnqiKNS/tMr2ThWQXiFUJJxDXRXJt/N+1I9Ocng=**
  - identity: {}
```

### 03. volume 마운트될 경로에 저장

이제 만들어진 config 파일을 api 서버 컨테이너가 volumeMount 할 수 있도록 적당한 위치에 저장한다.

```json
**# mkdir /etc/kubernetes/enc

# mv enc.yaml > /etc/kubernetes/enc

# ls /etc/kubernetes/enc/
enc.yaml**

```

## 03. api서버 설정

### 01. --encryption-provider-config 옵션 설정

api server의 command에 --encryption-provider-config 파라미터를 추가한다.

api server 파드는 마스터의 static-pod로 실행되기 때문에 /etc/kubernetes/manifest/kube-apiserver.yaml 파일을 수정하자.

**(주의) 마스터 서버가 여러개가 있을 경우 모든 마스터 서버에 동일한 설정을 해줘야 한다.**

```json
**# vi /etc/kubernetes/manifests/kube-apiserver.yaml**
----------------/etc/kubernetes/manifests/kube-apiserver.yaml------------------
(전략)
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.122.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
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
    **- --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
-------------------------------------------------------------------------------**
```

### 02. volume 설정

api server 컨테이너가 config 파일을 읽어야지 시크릿 암호화가 제대로 적용된다. 이를 위해 config가 위치한 디렉터리를 마운트할 수 있도록 설정해준다.

```json
**# vi /etc/kubernetes/manifests/kube-apiserver.yaml**
----------------/etc/kubernetes/manifests/kube-apiserver.yaml------------------
  volumeMounts:
(중략)
  **- mountPath: /etc/kubernetes/enc
      name: encrypt-config
      readOnly: true**
(중략)
  volumes:
(중략)
  **- hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
    name: encrypt-config
--------------------------------------------------------------------------------**
```

설정을 완료한 뒤 잠시 기다리거나 kubelet을 재시작한다.

## 04. 시크릿 암호화 테스트

### 01. 신규 시크릿 생성

```json
**# kubectl create secret generic secret-encrypt --from-literal=canuseeit="no"**
secret/secret-encrypt created
```

### 02. etcd로 확인

```json
# ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kuberneted/ca.crt --cert /es/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key --endpoints 192.168.122.10:2379 get /registry/secrets/default/secret-encrypt
/registry/secrets/default/secret-encrypt
**Ӳəs:enc:secretbox:v1:key1:ั¢͚´l2¬´KP%¦
󿂎v翋T򻰎µ'쥾CpyiLNm8ʒkײꗪo6zhᕳd񛺻0§򶮢#`¿½в⎯IR򤅭Ӄ5³欧/)m汧;ߺ5Q¸c.ܿjIĞj򋆄񾘯㿑3T!,ə©c.ո¯ªAՐ«naF±䐖ÿv§C\¨t񬀉g¼ʲ)=(cᵚ**
```

암호화되어서 출력되는 것을 알 수 있다.

## 05. 기존 시크릿 암호화

### 01. 여전히 암호화 안된 기존 시크릿들

encryptionConfiguration을 설정한 것 까지는 좋았으나, 문제는 기존의 시크릿은 암호화되지 않았다는 점이다.

```json
**# kubectl get secret**
NAME               TYPE     DATA   AGE
secret-encrypt     Opaque   1      79s  # 암호화 된 시크릿
secret-noencrypt   Opaque   1      69m  # 암호화 안 된 시크릿
```

암호화 전에 만든 secret인 secret-noencrypt를 etcd로 확인해보자.

```json
**# ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key --endpoints 192.168.122.10:2379 get /registry/secrets/default/secret-noencrypt
/registry/secrets/default/secret-noencrypt**
k8s

v1Secretف
¾ 
secret-noencryptdefault"*$39618330-036c-44da-a6e3-a6e1c66e3b052̏ޚzf
kubectl-createUpdatev̏ޚFieldsV1:2
0{"f:data":{".":{},"f:canuseeit":{}},"f:type":{}}B 
	**canuseeityes**Opaque"
```

여전히 값이 잘 보인다.

### 02. 기존 시크릿 암호화

이를 위해 기존 시크릿들을 전부 다 재생성해줘야만 한다. 아래 명령어로 가능하다.

<aside>
💡 **# kubectl get secret -A -o yaml | kubectl replace -f -**

</aside>

```json
**# kubectl get secret -A -o yaml | kubectl replace -f -**
secret/secret-encrypt replaced
secret/secret-noencrypt replaced
secret/ingress-nginx-admission replaced
secret/alertmanager-prom-stack-kube-prometheus-alertmanager replaced
secret/alertmanager-prom-stack-kube-prometheus-alertmanager-generated replaced
secret/alertmanager-prom-stack-kube-prometheus-alertmanager-tls-assets-0 replaced
secret/alertmanager-prom-stack-kube-prometheus-alertmanager-web-config replaced
secret/prom-stack-grafana replaced
secret/prom-stack-kube-prometheus-admission replaced
secret/prometheus-prom-stack-kube-prometheus-prometheus replaced
secret/prometheus-prom-stack-kube-prometheus-prometheus-tls-assets-0 replaced
secret/prometheus-prom-stack-kube-prometheus-prometheus-web-config replaced
secret/sh.helm.release.v1.prom-stack.v1 replaced
secret/sh.helm.release.v1.prom-stack.v2 replaced
```

이제 위의 시크릿들은 전부 암호화 됐다. secret-noencyrpt 시크릿으로 다시 테스트를 해보자.

```json
**# ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key --endpoints 192.168.122.10:2379 get /registry/secrets/default/secret-noencrypt
/registry/secrets/default/secret-noencrypt**
**k8s:enc:secretbox:v1:key1:¸󾶝O§ۧ¥1۾ÿ`ª³*L➿aдѕɾU[M򑫊	¿ªҐ԰b=Ă m|C#kTC·^´Eذᶙ	j͗m¿;ǽ^씤kH¯)𑅆IŲ(E񅏝A6cy𝣅4¹㫖¿ºɼ§I߰J¤Ьgo⠼£1@¹n»𺳭1ŹFAjӅ悊ү)¢ R\QB¾¹²չ1**
```

이제는 내용이 보이지 않는다.
