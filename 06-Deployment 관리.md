# K8s Deployment 관리

# 01. Deployment란?

ReplicaSet과 Pod를 관리하는 리소스다. 원하는 파드를 원하는 개수만큼 실행되도록 유지하는 역할을 한다.

정확히는 미리 설정된 replicas 값에 맞춰서 ReplicaSet을 생성한뒤, 생성된 ReplicaSet이 템플릿에 저장된 설정대로 Pod를 생성하는 구조다.

```json
**# kubectl get deploy,rs,po -l app=test-deploy**
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps**/test-deploy**   1/1     1            1           108s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/**test-deploy-584c765db4**   1         1         1       108s

NAME                               READY   STATUS    RESTARTS   AGE
pod/**test-deploy-584c765db4-vwskk**   1/1     Running   0          108s
```

위 예시를 보면 test-deploy가 자신의 이름 뒤에 임의의 값을 넣은 뒤(test-deploy**-574c765db4**) replicaset을 생성한 것을 확인할 수 있다.

레플리카셋 역시 자신의 이름 뒤에 임의의 값을 추가한 뒤(test-deploy-584c765db4**-vwskk**) 설정된 갯수 만큼의 pod를 생성한 것을 알 수 있다.

# 02. 명령어를 통해 deployment 관리하기

## 01. deploy 생성

### 01. 기본 명령어

<aside>
💡 **# kubectl create deploy {디플로이명} --image={이미지명}**

</aside>

```json
**# kubectl create deploy nginx-deploy --image=nginx**
deployment.apps/nginx-deploy created

**# kubectl get deploy**
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1/1     1            1           35s

**# kubectl get pod**
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-864bcf4b95-dxhlj   1/1     Running   0          47s
```

Deployment가 생성되면 디플로이 명에  랜덤한 값이 붙은 pod가 생성된다.

이 때 pod를 삭제한다 해도 deploy가 살아있는 한 새로운 pod가 다시 생성된다.

```json
**# kubectl get pod**
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-864bcf4b95-dxhlj   1/1     Running   0          47s

**# kubectl delete pod nginx-deploy-864bcf4b95-dxhlj** 
pod "nginx-deploy-864bcf4b95-dxhlj" deleted

**# kubectl get pod**
NAME                            READY   STATUS              RESTARTS   AGE
nginx-deploy-864bcf4b95-2cp8q   0/1     ContainerCreating   0          4s
```

### 02. deploy 생성 시 멀티컨테이너 생성

--image 옵션을 두 번 사용해서 멀티컨테이너를 구현한다.

<aside>
💡 **# kubectl create deploy {디플로이명} --image={이미지명1} --image={이미지명2}**

</aside>

```json
**# kubectl create deploy multi-deploy --image=redis --image=nginx**
deployment.apps/multi-deploy created

**# kubectl get pod**
NAME                            READY   STATUS    RESTARTS   AGE
multi-deploy-5d985bb867-z7qvq   **2/2**     Running   0          43s

**# kubectl describe deploy multi-deploy** 
Name:                   multi-deploy
Namespace:              default
CreationTimestamp:      Fri, 23 Sep 2022 15:41:55 +0900
Labels:                 app=multi-deploy
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=multi-deploy
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=multi-deploy
  Containers:
   redis:
    Image:        redis
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
   nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   multi-deploy-5d985bb867 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  74s   deployment-controller  Scaled up replica set multi-deploy-5d985bb867 to 1
```

### 03. deploy 생성 시 Container Port 부여

deploy시 내부 컨테이너가 오픈할 port를 지정할 수 있다. **다만 멀티 컨테이너가 아닌 단일 컨테이너로 생성할때만 원하는 대로 생성될 것이다.**

<aside>
💡 **# kubectl create deploy {디플로이명} --image={이미지명} --port={포트번호}**

</aside>

```json
**# kubectl create deploy nginx-deploy --image=nginx --port=8080**
deployment.apps/nginx-deploy created

**# kubectl describe deployments.apps nginx-deploy** 
Name:                   nginx-deploy
Namespace:              default
CreationTimestamp:      Fri, 23 Sep 2022 15:36:41 +0900
Labels:                 app=nginx-deploy
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx-deploy
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx-deploy
  Containers:
   nginx:
    Image:        nginx
    **Port:         8080/TCP**
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deploy-74f5b55648 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  20s   deployment-controller  Scaled up replica set nginx-deploy-74f5b55648 to 1
```

## 02. deploy 삭제

<aside>
💡 **# kubectl delete deploy {디플로이명}**

</aside>

```json
**# kubectl get deploy**
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
**nginx-deploy**   1/1     1            1           10s

**# kubectl delete deploy nginx-deploy** 
deployment.apps "nginx-deploy" deleted

**# kubectl get deploy**
No resources found in default namespace.
```

# 03. yaml파일을 이용해 deployment 관리

## 01. 명령어로 yaml파일 생성

kubectl create deploy 명령어는 멀티 컨테이너 관리나 여러 추가 설정이 곤란해진다. 이럴땐 마음 편하게 yaml 파일로 만들어서 수정하는 것이 좋다. 

<aside>
💡 **# kubectl create deploy {디플로이명} --image={이미지명} --dry-run=client -o yaml > {yaml 파일 명}**

</aside>

```yaml
**# kubectl create deploy nginx-deploy-yaml --image=nginx --dry-run=client -o yaml > nginx-deploy-yaml.yaml

# cat nginx-deploy-yaml.yaml** 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-deploy-yaml
  name: nginx-deploy-yaml
spec:
  **replicas: 1**
  **selector:
    matchLabels:
      app: nginx-deploy-yaml**
  **strategy: {}**
  template:
    metadata:
      creationTimestamp: null
      **labels:
        app: nginx-deploy-yaml**
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

**spec.template**은 나중에 pod를 생성할때 각 pod의 metadatae와 spec에 사용될 설정값들이다.

**spec.selector**는 해당 디플로이먼트가 관리할 파드를 선택할 수 있는 항목이다.

여기서 중요한 것은 **spec.template.metadata.labels와 spec.selector.matchLabels의 label 명을 반드시 일치시켜야 한다는 것**이다. 만약 두 항목이 일치하지 않는다면 Selector에서 관리해야할 pod들을 찾지 못하고 무한대로 pod를 생성하는 문제가 발생할 수 있다. 다행히 위 두 항목이 다를 경우 kubectl apply 명령시 error 메시지를 표시한다.

위 YAML 파일에서 deployment의 이름은 nginx-deploy-yaml이고 해당 디플로이에서 생성할 파드의 개수는 1개다. 각 파드는 nginx-deploy-yaml 레이블을 갖게 되고, [docker.io/nginx](http://docker.io/nginx) 이미지로부터 컨테이너를 생성하며 pod의 이름은 nginx-deploy-yaml로 시작한다.

## 02. yaml로 deploy 생성

<aside>
💡 **# kubectl create -f {deploy yaml 파일}**

</aside>

또는

<aside>
💡 **kubectl apply -f {deploy yaml 파일}**

</aside>

```json
**# kubectl apply -f nginx-deploy-yaml.yaml** 
deployment.apps/nginx-deploy-yaml created

**# kubectl get deploy,pod**
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy-yaml   1/1     1            1           14s

NAME                                     READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-yaml-8644fc66f9-7f948   1/1     Running   0          14s
```

### **참고) create와 apply의 차이**

kubectl create 명령어로 deployment를 생성하면 해당 deployment는 반드시 kubectl edit 명령어를 통해서 수정을 해야한다. 이후 다시 create 명령어나 kubectl apply 명령어를 통해서 수정된 yaml 파일을 적용시킬 순 없다. 만약 kubectl apply로 수정을 시도한다면 아래와 같은 경고 메시지가 발생한다.

```json
**# kubectl apply -f deploy-nginx.yaml**
**Warning: resource deployments/nginx-w-deploy is missing the [kubectl.kubernetes.io/last-applied-configuration](http://kubectl.kubernetes.io/last-applied-configuration) annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.**
deployment.apps/nginx-w-deploy configured
```

위 경고메시지는 해당 deployment에 기존의 설정값이 저장되어 있지 않다는 경고메시지다. kubectl create 명령어로 리소스를 생성한다면 기존 yaml파일에 저장된 값의 내용을 따로 저장하지 않는다. 반면 kubectl apply 명령어를 통해 리소스를 생성할 경우, metadata.annotation에 [kubectl.kubernetes.io/last-applied-configuration](http://kubectl.kubernetes.io/last-applied-configuration) 항목이 생성되며, 기존 yaml파일에 지정된 설정 값들이 저장되어 있다. 여기에 정의된 값과 새로 apply 하는 yaml 파일의 값을 비교하여 바뀐 내용을 토대로 deployment의 설정 값을 수정할 수 있다.

위 경고 메시지는 기존 yaml 파일의 설정값을 저장한 주석(annotation)이 없다는 경고 메시지다.

만약 대량의 yaml파일을 통해서 관리를 해야할 시스템이라면 초기에 리소스를 생성할때 create 명령어가 아닌 apply 명령어를 통해서 생성하는 것이 더 좋을 것이다.

## 03. yaml로 deploy 수정

.metadata.name이 동일한 deploy에 대해서 수정이 진행된다.

<aside>
💡 **# kubectl replace -f {yaml 파일 명}**

</aside>

또는

<aside>
💡 **# kubectl apply -f {yaml 파일 명}**

</aside>

## 04. yaml로 deploy 삭제

.metadata.name이 동일한 deploy에 대해서 삭제가 진행된다.

<aside>
💡 **# kubectl delete -f {yaml 파일 명}**

</aside>

```json
**# kubectl delete -f nginx-deploy-yaml.yaml** 
deployment.apps "nginx-deploy-yaml" deleted
```

# 04. replicaset 관리

Deployment를 생성하면 가장 먼저 ReplicaSet을 생성한다. 이 때문에 굳이 ReplicaSet을 따로 생성해줄 필요는 없으며, Deployment가 관리하도록 내버려 두는 것이 가장 마음 편하다.

## 01. deployment 생성 시 replica 설정

<aside>
💡 **# kubectl create deploy {디플로이명} --image={이미지명} --replicas={생성할 파드 개수}**

</aside>

```json
**# kubectl create deploy nginx-deploy2 --image=nginx --replicas=2**
deployment.apps/nginx-deploy2 created

**# kubectl get rs**
NAME                       DESIRED   CURRENT   READY   AGE
nginx-deploy2-6548cf9df7   **2         2         2**       74s

**# kubectl get pod**
NAME                             READY   STATUS    RESTARTS   AGE
nginx-deploy2-6548cf9df7-64hkp   1/1     Running   0          100s
nginx-deploy2-6548cf9df7-j2kvt   1/1     Running   0          100s
```

## 02. 기존 생성된 deployment의 replicaset 변경

### 01. 명령어로 변경

<aside>
💡 **# kubectl scale deploy {디플로이명} --replicas={변경할 파드 개수}**

</aside>

```json
**# kubectl scale deploy nginx-deploy2 --replicas=3**
deployment.apps/nginx-deploy2 scaled

**# kubectl get rs**
NAME                       DESIRED   CURRENT   READY   AGE
nginx-deploy2-6548cf9df7   3         3         3       3m23s

**# kubectl get pod**
NAME                             READY   STATUS    RESTARTS   AGE
nginx-deploy2-6548cf9df7-572pm   1/1     Running   0          72s
nginx-deploy2-6548cf9df7-64hkp   1/1     Running   0          3m45s
nginx-deploy2-6548cf9df7-j2kvt   1/1     Running   0          3m45s
```

### 02. YAML에서 변경

deployment에서 replicas 값을 수정해주면 된다.

```json
**# kubectl get deploy test-deploy -o jsonpath='{.spec.replicas}{"\n"}'**
1

**# kubectl edit deploy test-deploy**
-----------deploy/test-deploy---------------
(전략)
spec:
  progressDeadlineSeconds: 600
  **replicas: 3 # 수정**
(하략)
---------------------------------------------

**# kubectl get rs,po -l app=test-deploy**
NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/test-deploy-584c765db4   3         3         3       39m

NAME                               READY   STATUS    RESTARTS   AGE
pod/test-deploy-584c765db4-2wv4w   1/1     Running   0          12s
pod/test-deploy-584c765db4-724td   1/1     Running   0          12s
pod/test-deploy-584c765db4-vwskk   1/1     Running   0          39m
```

# 05. Rolling update / RollBack

## 01. Rolling update

Deployment에서 별도로 strategy를 지정하지 않으면  RollingUpdate 타입을 사용한다. Rolling update는 기존에 동작중인 Pod를 Running 상태로 유지한 뒤, 새로운 Pod를 실행하고, 새로운 Pod가 Running 상태가 되면 기존의 Pod를 Terminating 한다.

<aside>
💡 **# kubectl set image deploy {디플로이 명}  {컨테이너명}={이미지}:{이미지버전}**

</aside>

```json
**# kubectl get deploy webserver -o jsonpath='{.spec.template.spec.containers[].imag**e}'
nginx:1.16.1

$ kubectl get deploy webserver -o jsonpath='{.spec.strategy}' | jq
{
  "rollingUpdate": {
    "maxSurge": "25%",
    "maxUnavailable": "25%"
  },
  **"type": "RollingUpdate"**
}

**# kubectl get pod -l app=webserver -w**
NAME                         READY   STATUS    RESTARTS   AGE
webserver-566b9f9975-x6wfv   1/1     Running   0          6s

(다른 세션 실행)
**# kubectl set image deploy webserver nginx=nginx:1.21.1**
deployment.apps/webserver image updated

**# kubectl get deploy webserver -o jsonpath='{.spec.template.spec.containers[].image}'**
nginx:1.21.1

(이전 세션)
**# kubectl get pod -l app=webserver -w**
NAME                         READY   STATUS    RESTARTS   AGE
webserver-566b9f9975-x6wfv   1/1     Running   0          6s     **<- 기존 Pod**
webserver-b5d59dc86-gp69h    0/1     Pending   0          0s     **< - 새 Pod 생성 준비**
webserver-b5d59dc86-gp69h    0/1     ContainerCreating   0      0s  **<- 새 Pod 생성**
webserver-b5d59dc86-gp69h    1/1     Running             0      3s  **<- 새 Pod 실행**
webserver-566b9f9975-x6wfv   1/1     Terminating         0      18s  **<- 기존 Pod 삭제**
```

## 02. RollBackUp

Rolling Update를 실행한 Deploy의 설정을 다시 이전으로 복구한다. 업데이트와 마찬가지로 새로운 Pod가 running 상태가 되어야 이전 Pod가 Terminated 된다.

### 01. Update 바로 이전 상태로 복귀

<aside>
💡 **# kubectl rollout undo deploy {디플로이 명}**

</aside>

```json
**# kubectl get deploy webserver -o jsonpath='{.spec.template.spec.containers[].image}'**
nginx:1.21.1

**# kubectl get pod -l app=webserver -w**
NAME                        READY   STATUS    RESTARTS   AGE
webserver-b5d59dc86-gp69h   1/1     Running   0          61m

(다른 세션 실행)
**# kubectl rollout undo deploy webserver**
deployment.apps/webserver rolled back

(이전 세션)
**# kubectl get pod -l app=webserver -w**
NAME                        READY   STATUS    RESTARTS   AGE
webserver-b5d59dc86-gp69h   1/1     Running   0          61m
webserver-77f8b4648d-vqm6d   0/1     Pending   0          0s
webserver-77f8b4648d-vqm6d   0/1     ContainerCreating   0          1s
webserver-77f8b4648d-vqm6d   1/1     Running             0          5s
webserver-b5d59dc86-gp69h    1/1     Terminating         0          63m
```

### 02. Update 이력 확인

<aside>
💡 **# kubectl rollout history deploy {디플로이 명}**

</aside>

```json
$ kubectl rollout history deploy webserver
deployment.apps/webserver 
REVISION  CHANGE-CAUSE
7         <none>
8         <none>
9         <none>
11        <none>
12        <none>
```

history 기능을 사용하면 해등 디플로이먼트의 지난 업데이트 버전의 목록을 확인할 수 있다. 특정 버전(REVISION)의 내용을 확인하고 싶다면 아래와 같이 --revision 옵션을 사용한다.

<aside>
💡 **# kubectl rollout history deploy {디플로이 명} --revision={리비전 번호}**

</aside>

```json
# **kubectl rollout history deploy webserver --revision=7**
deployment.apps/webserver with revision #7
Pod Template:
  Labels:	app=webserver
	pod-template-hash=566b9f9975
  Containers:
   nginx:
    Image:	nginx
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

### 03. 특정 Revision으로 복귀

<aside>
💡 **# kubectl rollout undo deploy {디플로이 명} --to-revision={리비전 번호}**

</aside>

```json
**# kubectl get deploy webserver -o jsonpath='{.spec.template.spec.containers[].image}'**
nginx:1.21.1

# **kubectl rollout history deploy webserver --revision=7**
deployment.apps/webserver with revision #7
Pod Template:
  Labels:	app=webserver
	pod-template-hash=566b9f9975
  Containers:
   nginx:
    **Image:	nginx**
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>

**# kubectl rollout undo deploy webserver --to-revision=7**
deployment.apps/webserver rolled back

**# kubectl get deploy webserver -o jsonpath='{.spec.template.spec.containers[].image}'**
nginx
```

# 06. Canary update

## 01. Canary update란?

카나리는 예전에 탄광에서 유독가스가 나오는지 확인하기 위해 데려다 놓은 새를 의미한다. ~~액젓 만드는 그 생선 아니다.~~ 카나리는 일산화탄소나 메탄가스에 매우 예민했기 때문에, 탄광에서 작업을 하던 인부들은 갑자기 이 새가 죽어버리면 유독 가스가 나오는 것을 곧바로 인지한 후 대피했다고 한다.

어플리케이션을 배포할때도 이런 카나리가 쓰인다. 구 버전을 실행중인 서버들 사이에 은근 슬쩍 새 버전을 끼워넣어 본다. 그러니까 전체 10대의 서버 중에서 1대만 새 버전으로 바꾸는 것이다. 그리고 동일하게 로드밸런싱한다.

기존의 구 버전을 쓰는 9대 서버는 여전히 정상 작동할 것이다. 그런데 새 버전에서 문제가 있다면 새로 업그레이드된 1대 서버에 접속할때 이상이 발견될 것이다. 그러면 조용히 1대의 신규 버전의 서버를 걷어낸 뒤, 구 버전 서버를 다시 복구한다.

만약 문제가 발견되지 않는다면 1대에서 2대로, 2대에서 3대로 점진적으로 신규 버전의 비율을 늘린다. 이 과정에서도 또 문제가 발생하면 구 버전으로 다시 복구한다.

이렇게 점진적으로 신규 버전의 비중을 늘리다가 아무런 문제를 발견하지 못하면 구 버전을 모두 걷어낸 뒤 새 버전으로 완전히 바꾼다.

이렇게 같은 서비스 내에서 점진적으로 새 버전의 비중을 늘리는 방법이 Canary Update다.

## 02. Canary update 테스트

### 01. 기존 Deployment에 라벨 추가

현재 webserver deployment는 컨테이너 이미지 nginx:1.16.1을 사용하고 있다.

```json
**# kubectl get deploy webserver -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'**
nginx:1.16.1
```

현재 서비스중인 deployment에 현재 버전을 나타내는 라벨을 추가해준다.

```json
**# kubectl label deployments.apps webserver version=v1**
deployment.apps/webserver labeled

**# kubectl get deployments.apps webserver --show-labels**
NAME        READY   UP-TO-DATE   AVAILABLE   AGE    LABELS
webserver   5/5     1            1           121m   **app=webserver**,**version=v1**
```

### 02. 서비스의 셀렉터 확인

서비스의 셀렉터는 기존에 등록된 app=webserver 라벨만 바라보도록 한다. 이 서비스를 통해서 새로 업그레이드할 Deployment 역시 로드밸런싱할 예정이다.

```json
**# kubectl get svc webserver -o jsonpath='{.spec.selector}' | jq**
{
  "app": "webserver"
}
```

### 03. 카나리 적용할 Deployment 추가

컨테이너 이미지 nginx:1.17.1을 사용하는 새로운 Deployment를 생성한다.  기존 서비스와 연결되야 하기 때문에 기존 라벨인 app=webserver는 그대로 유지하고 version만 v2로 지정해준다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    **app: webserver
    version: v2**
  name: webserver-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      **app: webserver
      version: v2**
  strategy: {}
  template:
    metadata:
      labels:
        **app: webserver
        version: v2**
    spec:
      containers:
      **- image: nginx:1.17.1**
        name: nginx
        resources: {}
status: {}
```

위 YAML을 실행한다.

```yaml
**# kubectl apply -f webserver-v2.yaml** 
deployment.apps/webserver-v2 created

**# kubectl get deployments.apps** 
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
webserver      5/5     5            5           133m
**webserver-v2   1/1     1            1           4s**
```

### 04. webserver 서비스의 Endpoints 확인

서비스의 endpoints를 확인해보면 webserver에서 생성한 pod와 webserver-v2에서 생성한 pod 두 개가 동시에 연결된 것을 확인할 수 있다.

```yaml
**# kubectl get endpoints webserver -o yaml**
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    endpoints.kubernetes.io/last-change-trigger-time: "2022-10-14T07:01:04Z"
  creationTimestamp: "2022-10-14T06:54:41Z"
  labels:
    app: webserver
  name: webserver
  namespace: default
  resourceVersion: "2199995"
  uid: 6352f70e-46b4-4e8c-a75f-db54918dbd5f
subsets:
- addresses:
  - ip: 10.100.194.115
    nodeName: k8s-worker1
    targetRef:
      kind: Pod
      name: webserver-78896968d6-l6npt
      namespace: default
      uid: 4efdbf7d-c6b2-4703-9f38-5c905c9091d4
  - ip: 10.100.194.126
    nodeName: k8s-worker1
    targetRef:
      kind: Pod
      name: webserver-78896968d6-z9bw7
      namespace: default
      uid: 1c4e6ee9-8259-43af-a516-4c432cd1f9d4
  **- ip: 10.100.194.127
    nodeName: k8s-worker1
    targetRef:
      kind: Pod
      name: webserver-v2-67c5f499d5-gpmpq
      namespace: default
      uid: ad1231c0-23f3-4341-9366-4553296cd56f**
  - ip: 10.100.194.79
    nodeName: k8s-worker1
    targetRef:
      kind: Pod
      name: webserver-78896968d6-r4g2h
      namespace: default
      uid: 6d49f15e-da2e-42a6-9618-a6ea6300df44
  - ip: 10.110.126.15
    nodeName: k8s-worker2
    targetRef:
      kind: Pod
      name: webserver-78896968d6-4r5ts
      namespace: default
      uid: b524ba9c-a74e-442b-9e9e-ba03f5031bcb
  - ip: 10.110.126.18
    nodeName: k8s-worker2
    targetRef:
      kind: Pod
      name: webserver-78896968d6-sdfsg
      namespace: default
      uid: 6975781f-fbeb-4514-8a8a-3031d62efed7
  ports:
  - port: 80
    protocol: TCP
```

### 05. 기존 Deployment의 ReplicaSet 조절

카나리 적용할 새로운 버전의 Deployment가 생성됐으니 기존 Deployment의 replication을 조절한다.

```yaml
**# kubectl scale deployment webserver --replicas=4**
deployment.apps/webserver scaled

**# kubectl get pod**
NAME                            READY   STATUS    RESTARTS   AGE
webserver-78896968d6-4r5ts      1/1     Running   0          60m
webserver-78896968d6-lqmss      1/1     Running   0          43s
webserver-78896968d6-sdfsg      1/1     Running   0          60m
webserver-78896968d6-z9bw7      1/1     Running   0          3h5m
webserver-v2-67c5f499d5-gpmpq   1/1     Running   0          53m
```

서비스의 엔드포인트도 확인하자

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  creationTimestamp: "2022-10-14T06:54:41Z"
  labels:
    app: webserver
  name: webserver
  namespace: default
  resourceVersion: "2206617"
  uid: 6352f70e-46b4-4e8c-a75f-db54918dbd5f
subsets:
- addresses:
  - ip: 10.100.194.108
    nodeName: k8s-worker1
    targetRef:
      kind: Pod
      name: webserver-78896968d6-lqmss
      namespace: default
      uid: a6e93e9f-b828-4847-81f5-f96d6cd2187f
  - ip: 10.100.194.126
    nodeName: k8s-worker1
    targetRef:
      kind: Pod
      name: webserver-78896968d6-z9bw7
      namespace: default
      uid: 1c4e6ee9-8259-43af-a516-4c432cd1f9d4
  - ip: 10.100.194.127
    nodeName: k8s-worker1
    targetRef:
      kind: Pod
      name: webserver-v2-67c5f499d5-gpmpq
      namespace: default
      uid: ad1231c0-23f3-4341-9366-4553296cd56f
  - ip: 10.110.126.15
    nodeName: k8s-worker2
    targetRef:
      kind: Pod
      name: webserver-78896968d6-4r5ts
      namespace: default
      uid: b524ba9c-a74e-442b-9e9e-ba03f5031bcb
  - ip: 10.110.126.18
    nodeName: k8s-worker2
    targetRef:
      kind: Pod
      name: webserver-78896968d6-sdfsg
      namespace: default
      uid: 6975781f-fbeb-4514-8a8a-3031d62efed7
  ports:
  - port: 80
    protocol: TCP
```

위 과정을 다 끝낸 후, 서비스에 아무런 문제가 없다면, 구 버전과 신 버전의 ReplicaSet을 점진적으로 조정하면 된다.
