# json path

[JSONPath Support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)

# 01. jsonpath 사용 법

## 00. json path 찾는 법

## 01. json 예시

```json
{
    "kind": "Deployment",
    "apiVersion": "apps/v1",
    "metadata": {
        "name": "test-jsonpath-deploy",
        "creationTimestamp": null,
        "labels": {
            "app": "test-jsonpath-deploy"
        }
    },
    "spec": {
        "replicas": 1,
        "selector": {
            "matchLabels": {
                "app": "test-jsonpath-deploy"
            }
        },
        "template": {
            "metadata": {
                "creationTimestamp": null,
                "labels": {
                    "app": "test-jsonpath-deploy"
                }
            },
            "spec": {
                "containers": [
                    {
                        "name": "nginx",
                        "image": "nginx:1.21.1",
                        "resources": {}
                    },
                    {
                        "name": "redis",
                        "image": "redis:7",
                        "resources": {}
                    },
                    {
                        "name": "memcached",
                        "image": "memcached:latest",
                        "resources": {}
                    }
                ]
            }
        },
        "strategy": {}
    },
    "status": {}
}
```

위는 nginx, redis, memcached 이미지를 사용하는 멀티컨테이너 디플로이먼트의  json 예시다. 이를 기준으로 json path 찾는 법을 알아보자.

## 02. 기본 경로 탐색

```json
{                                    <------------- .
    "kind": "Deployment",
    "apiVersion": "apps/v1",
    "metadata": {                    <-------------  metadata.
        "name": "test-jsonpath-deploy",
        "creationTimestamp": null, 
        "labels": {                 <-------------- labels.
            "app": "test-jsonpath-deploy"       <----------- app
        }                                           = .metadata.labels.app
     },
```

json은 중괄호{}로 하위 항목들을 묶는다. 위에서 metadata는 중괄호로 묶은 뒤 name, creationTimestamp, label를 하위 항목으로 두고 있으며, labels는 app을 하위 항목으로 두고 있다.

이렇게 하위항목을 표현하는 중괄호 중 왼쪽 부분( ’{’ )을 .으로 바꿔보자. 그러면 **metadata 아래에 있는 labels와 app까지의 경로를 metadata.labels.app으로 표현**할 수 있다.

그런데 metadata 역시 kind와 apiVersion과 같이 상위 항목에 포함되어 있다. 최 상위에 모든 항목을 묶는 중괄호{}가 있으니 여기의 왼쪽 부분을 .으로 바꾸자.

그러면 **최상단으로부터 app까지의 경로는 아래와 같이 표현된다.**

<aside>
💡 **.metadata.label.app**

</aside>

위 경로를 기준으로 kubectl get 명령어로 찾고자 한다면 아래와 같은 명령어를 사용한다. 그런데 jsonpath를 사용할 경우 자동 줄바꿈이 안되기 때문에 마지막에 이스케이프문 {”\n”}를 추가해준다.

<aside>
💡 **# kubectl get deploy test-jsonpath-deploy -o jsonpath=’{.metadata.labels.app}{”\n”}’**

</aside>

```json
**# kubectl get deploy test-jsonpath-deploy -o jsonpath='{.metadata.labels.app}{"\n"}'**
test-jsonpath-deploy
```

## 03. json map 경로 탐색

근데 json파일을 보다보면 아래와 같이 대괄호[]도 찾을 수 있다.

```json
{                                                    <------------- .
(중략)
    "spec": {                                        <------------- spec.
(중략)
        "template": {                                <------------- template.
(중략)
            "spec": {                                <------------- spec.
                "containers": [                      <------------- contaienrs[]
                    {   ## containers[0].
                        "name": "nginx",
                        "image": "nginx:1.21.1",
                        "resources": {}
                    },
                    {   ## containers[1].
                        "name": "redis",
                        "image": "redis:7",
                        "resources": {}
                    },
                    {   ## containers[2].
                        "name": "memcached",
                        "image": "memcached:latest",
                        "resources": {}
                    }
                ]
            }
(하략)
```

대괄호는 하위에 동일한 내용이 반복될때 사용한다. 멀티컨테이너 구조상 image와 name과 같은 하위항목이 반복될 수 밖에 없다. 이럴 때 대괄호로 일괄적으로 묶은 뒤 중괄호로 하위 항목을 세분화해서 나눈다. **이것을 JSON Map이라고 한다.**

이때 경로를 표현하려면 우선 대괄호를 일괄적으로 다 넣은 **containers[]**로 표현한다.  그렇다면 conateinrs까지의 경로는 아래와 같다.

<aside>
💡 **.spec.template.spec.containers[]**

</aside>

위 JSON 구조를 보면 containers 아래에 세 개의 컨테이너 정보가 있다. 이 세 컨테이너는 위에서 부터 차례대로 0, 1, 2 index를 부여받는다. 따라서 **nginx컨테이너의 image 정보를 가져오고 싶다면** 경로를 아래와 같이 설정한다.

<aside>
💡 **.spec.template.spec.containers[0].image**

</aside>

<aside>
💡 # **kubectl get deploy test-jsonpath-deploy -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'**

</aside>

```json
**# kubectl get deploy test-jsonpath-deploy -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'**
nginx:1.21.1
```

모든 컨테이너의 이미지를 가져오고 싶다면 아래와 같은 경로를 사용하면 된다.

<aside>
💡 **.spec.template.spec.containers[*].image**

</aside>

<aside>
💡 # **kubectl get deploy test-jsonpath-deploy -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'**

</aside>

```json
**# kubectl get deploy test-jsonpath-deploy -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'**
nginx:1.21.1 redis:7 memcached:latest
```

## 04. Json Map 범위 검색

범위 검색은 JSON MAP에서 반복되는 구문들을 일괄 표기할때 사용한다. 다시 아래 예시를 보자.

```json
{                                                    <------------- .
(중략)
    "spec": {                                        <------------- spec.
(중략)
        "template": {                                <------------- template.
(중략)
            "spec": {                                <------------- spec.
                "containers": [                      <------------- contaienrs[]
                    {   ## containers[0].
                        "name": "nginx",
                        "image": "nginx:1.21.1",
                        "resources": {}
                    },
                    {   ## containers[1].
                        "name": "redis",
                        "image": "redis:7",
                        "resources": {}
                    },
                    {   ## containers[2].
                        "name": "memcached",
                        "image": "memcached:latest",
                        "resources": {}
                    }
                ]
            }
(하략)
```

위 json을 기본으로 모든 container의 name과 image를 출력하고자 한다. 이를 위해 아래와 같은 명령어를 사용했다.

```json
**# kubectl get deploy test-jsonpath-deploy -o jsonpath='{.spec.template.spec.containers[].name}{"\t"}{.spec.template.spec.containers[].image}{"\n"}'
nginx redis memcached	nginx:1.21.1 redis:7 memcached:latest**
```

위 결과를 보면  모든 name이 먼저 출력되고 image가 그 뒤에 출력된다. 컨테이너 네임과 이미지가 제대로 매칭되지 않아서 보기 매우 힘들어진다.

이때 사용하는 방법이 {range}{end} 구문이다. 개발언어에서 for 구문과 같은 기능을 한다. 구문의 사용방법은 아래와 같다.

<aside>
💡 **{range (반복 검색을 위한 json map의 경로)}{(json map의 하위 경로)}{end}**

</aside>

위의 json path를 다시 봐보자. 우리는 containers 아래에 있는 컨테이너의 정보를 알아내려고 한다. 그러다면 range 구문은 아래와 같이 적어야 한다.

<aside>
💡 **{range .spec.template.spec.containers[*]}**

</aside>

그 후 우리가 필요한 정보는 name과 image다. 이 부분을 아래와 같이 적되, 명확한 구분을 위해 중간에 이스케이프문으로  \t을 추가하고, 줄바꿈을 위해 \n도 추가하자

<aside>
💡 **{.name}{”\t”}{.image}{”\n”}**

</aside>

찾고자 하는 모든 하위 경로를 적었으니 range 반복을 끝낸다는 표시자를 넣어준다.

<aside>
💡 **{end}**

</aside>

위 모든 요소들을 합치면 아래와 같다.

<aside>
💡 **{range .spec.template.spec.containers[*]}{.name}{”\t”}{.image}{”\n”}{end}**

</aside>

이를 기준으로 jsonpath를 검색해보자.

<aside>
💡 # **kubectl get deploy test-jsonpath-deploy -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'**

</aside>

```json
# **kubectl get deploy test-jsonpath-deploy -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'**
nginx	nginx:1.21.1
redis	redis:7
memcached	memcached:latest
```

훨씬 보기 편하게 출력된다.

## 05. Json Map 조건 검색

Json Map에서 특정한 조건을 만족하는 하위 항목만 검색하는 방법도 있다. 일단 다시 아래 JSON 예시를 보자.

```json
{                                                    <------------- .
(중략)
    "spec": {                                        <------------- spec.
(중략)
        "template": {                                <------------- template.
(중략)
            "spec": {                                <------------- spec.
                "containers": [                      <------------- contaienrs[]
                    {   ## containers[0].
                        "name": "nginx",
                        "image": "nginx:1.21.1",
                        "resources": {}
                    },
                    {   ## containers[1].
                        "name": "redis",
                        "image": "redis:7",
                        "resources": {}
                    },
                    {   ## containers[2].
                        "name": "memcached",
                        "image": "memcached:latest",
                        "resources": {}
                    }
                ]
            }
(하략)
```

위 내용에서 containers아래에서 name이 nginx인 항목의 image 정보를 가져오고자 한다. 이때 사용하는 구문이 ?()다.

<aside>
💡 **상위항목경로[?(조건문) @].하위항목**

</aside>

위 내용만 보면 이해가 잘 안가니까 예시를 들어보자.

우리는 containers 하위에서 조건을 찾을 예정이다. 따라서 우선 containers까지의 경로를 그려보자.

<aside>
💡 **.spec.template.spec.containers[]**

</aside>

이전에 우리는 저 대괄호[] 안에서 index 번호를 기입해서 하위 항목을 선택했다. 그렇다면 조건문 역시 저 대괄호 안에 들어가서 하위 항목의 조건을 지정할 수 있겠다.

<aside>
💡 **.spec.template.spec.containers[?(조건문 들어갈 자리)]**

</aside>

이제 조건문을 넣자. 조건은 .spec.template.spec.containers[*].name이 nginx인 항목이다. 비교문으로 적으로 아래와 같다.

<aside>
💡 **.spec.template.spec.containers[*].name==”nginx”**

</aside>

그런데 위 조건문을 그대로 쓰면 아래와 같이 매우 복잡하고 지저분해진다.

<aside>
💡 **.spec.template.spec.containers[?(.spec.template.spec.containers[*].name==”nginx”)]**

</aside>

이 때문에 조금 더 간편하고 경제적인 방법을 사용한다. 아래 예시와 같이 일단 **그동안 지나온 json 경로를 “@”로 표기하는 방법**이다. 

<aside>
💡 **.spec.template.spec.containers[?(조건문 들어갈 자리)]
——————————————,
                                                      \ _  @로 변환**

</aside>

즉 조건문을 이렇게 표기할 수 있다.

<aside>
💡 **@.name==”nginx”**

</aside>

그러면 아래와 같이 훨씬 간편하게 경로를 설정할 수 있다.

<aside>
💡 **.spec.template.spec.containers[?(@.name==”nginx”)]**

</aside>

name이 nginx인 항목의 image를 확인하고자 했으니 아래와 같이 하위항목을 image로 지정한다.

<aside>
💡 **.spec.template.spec.containers[?(@.name==”nginx”)].image**

</aside>

전체 명령어는 아래와 같다.

<aside>
💡 # kubectl get deploy test-jsonpath-deploy -o jsonpath=’{**.spec.template.spec.containers[?(@.name==”nginx”)].image}{”\n”}’**

</aside>

```json
**# kubectl get deploy test-jsonpath-deploy -o jsonpath='{.spec.template.spec.containers[?(@.name=="nginx")].image}{"\n"}'**
nginx:1.21.1
```

## 06. 전체 Resource 단위에서 검색

kubectl get 명령어로 Resource를 검색할 때, 우리 눈에 보이는 목록 역시 이미 json 코드로 출력이 가능하다.

현재 띄워져 있는 Pod의 목록이다.

```json
# **kubectl get pod**
NAME         READY   STATUS    RESTARTS   AGE
apache-pod   1/1     Running   0          11m
mysql-pod    1/1     Running   0          13m
nginx-pod    1/1     Running   0          13m
redis-pod    1/1     Running   0          13m
```

이 상태에서 -o json으로 출력하면 각 pod의 설정 상단에 아래와 같은 항목이 있는 것을 확인할 수 있다.

```json
$ kubectl get pod -o json | head
{
    **"apiVersion": "v1",
    "items": [**
        {
            "apiVersion": "v1",
            "kind": "Pod",
            "metadata": {
                "annotations": {
                    "cni.projectcalico.org/containerID": "3f35fffbd1896179fb737b6e617381486e5c891152c22e524e9d6d166511df9b",
                    "cni.projectcalico.org/podIP": "10.100.194.106/32",

(하략)
```

각 pod 목록의 상단에 items가 있는 것을 확인할 수 있다. 마치 아래 구조처럼 말이다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8442505b-2491-4c33-9d7a-b60bbf106dc3/Untitled.png)

위 그림을 보고 이미 눈치챘겠지만, 이 역시 모든 item들이 json map 형식으로 분류되어 있다는 것을 확인할 수 있다. 그렇다면 우리는 아래와 같이 활용할 수 도 있을 것이다.

### 현재 실행중인 모든 pod의 name 출력

<aside>
💡 **# kubectl get pod -A -o jsonpath='{.items[*].metadata.name}{"\n"}'**

</aside>

```json
**# kubectl get pod -A -o jsonpath='{.items[*].metadata.name}{"\n"}'**
apache-pod mysql-pod nginx-pod redis-pod nginx-label hi-pod ingress-nginx-admission-create-np55k ingress-nginx-admission-patch-2j775 ingress-nginx-controller-b7b55cccc-zb98p calico-kube-controllers-6799f5f4b4-t64zf calico-node-fftdj calico-node-sqbl2 calico-node-wp2v2 coredns-6d4b75cb6d-lmgkw coredns-6d4b75cb6d-vkzfm etcd-k8s-master kube-apiserver-k8s-master kube-controller-manager-k8s-master kube-proxy-fzqph kube-proxy-rccwh kube-proxy-xgmrx kube-scheduler-k8s-master metrics-server-7b857dcf59-7gq8j
```

### 현재 default namespace에서 실행중인 모든 pod의 name과 contaienr image 출력

<aside>
💡 **# kubectl get pod -n default -o jsonpath='{range .items[]}{.metadata.name}{"\t"}{.spec.containers[].image}{"\n"}{end}'**

</aside>

```json
# **kubectl get pod -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'**
apache-pod	httpd
mysql-pod	mysql
nginx-pod	nginx
redis-pod	redis
```
