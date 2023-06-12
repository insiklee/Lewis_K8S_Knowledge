# K8s HPA 관리

[Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

# 01. HPA란?

Horizontal Pod Autoscaler의 약자로 ReplicaSet이나 Deployment, StatefulSet에서 관리하는 Pod의 자원 사용량에 따라서 Pod의 개수를 조절하는 오브젝트다. 다른 오브젝트와 마찬가지로 kubernetes API server에 있는 컨트롤러가 조정한다.

컨트롤되는 Pod들이 적은 자원만 사용한다면 Pod의 개수를 점진적으로 줄이고, 많은 자원을 사용한다면 Pod 개수를 점진적으로 증가시킨다.

당연한 말이지만 HPA를 사용하기 위해서는 메트릭 서버를 구축해야한다. [7. Metric-Server 설치](https://www.notion.so/7-Metric-Server-d9e71f765a8f410f89bbb83bcf7aeb9a?pvs=21) 에서 메트릭 서버를 설치하고 오자.

## 02. HPA 관련 이것저것

- HPA 컨트롤러가 파드의 메트릭 정보를 가져 오는 간격은 15초이다. 이는 kube-controller-manager 아래에 --horizontal-pod-autoscalser-sync-period에서 수정할 수 있다.

## 03. HPA 레플리카셋 계산 공식

HPA 컨트롤러가 새 레플리카셋을 계산하는 공식은 아래와 같다.

<aside>
💡 desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]

</aside>

현재 레플리카가 10이고 desiredMetric이 100m, 현재 Metric이 200m 이면 200/100=2.0이 된다. 이값이 현재 레플리카 개수인 10에 곱해져서 (10 * 2.0) desiredReplicas는 20이 된다.

여기서 currentMetric과 desiredMetric의 기준이 되는 값은 컨테이너가 지닌 resources.requests 값이 기준이다.

# 02. HPA 생성

[HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

## 00. 테스트 준비

HPA 테스트 진행을 위한 Deploy와 서비스를 하나 배포하자.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hpa-test-svc
  name: hpa-test-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: hpa-test-deploy
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hpa-test-deploy
  name: hpa-test-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hpa-test-deploy
  strategy: {}
  template:
    metadata:
      labels:
        app: hpa-test-deploy
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: 
          requests:
            cpu: 2m
            memory: 4Mi
          limits:
            cpu: 15m
            memory: 20Mi
```

```json
**# kubectl apply -f hpa-test.yaml** 
service/hpa-test-svc created
deployment.apps/hpa-test-deploy created

**# kubectl get deploy,svc,po**
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hpa-test-deploy   3/3     3            3           10m

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/hpa-test-svc   ClusterIP   10.103.56.78   <none>        80/TCP    57s
service/kubernetes     ClusterIP   10.96.0.1      <none>        443/TCP   14d

NAME                                   READY   STATUS    RESTARTS   AGE
pod/hpa-test-deploy-869f69b99f-65c9x   1/1     Running   0          10m
pod/hpa-test-deploy-869f69b99f-kkvx5   1/1     Running   0          10m
pod/hpa-test-deploy-869f69b99f-tvwfc   1/1     Running   0          10m
```

## 01. 커맨드로 HPA 리소스 생성

**커맨드로 만들어진 HPA는 autoscaling/v1 API 버전을 사용**한다. 기능상 한계가 많이 있는 버전이다.

<aside>
💡 **# kubectl autoscale deploy {디플로이명} --min={최소 파드 수} --max={최대 파드 수}**

</aside>

```json
**# kubectl autoscale deployment hpa-test-deploy --min=1 --max=5**
horizontalpodautoscaler.autoscaling/hpa-test-deploy autoscaled

**# kubectl get hpa**
NAME              REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-test-deploy   Deployment/hpa-test-deploy   0%/80%            1         5         3          8s
```

위 커맨드로 생성한 hpa는 default로 pod의 평균 cpu 사용량이 requests의 80%일 경우 scale up 하도록 설정되어 있다.

### 01. **HPA 테스트**

위와같이 hpa 생성 후 아무것도 안하면 자동으로 실행중인 pod 개수를 줄인다.

```json
**# kubectl get pod** 
NAME                               READY   STATUS    RESTARTS   AGE
hpa-test-deploy-869f69b99f-tvwfc   1/1     Running   0          16m
```

아래 스크립트로 pod에 부하를 걸면 hpa는 추가로 pod를 생성한다.

```json
IPADDR=$(kubectl get svc hpa-test-svc -o jsonpath='{.spec.clusterIP}')
while true
do
  curl $IPADDR
done
```

스크립트 실행 후 pod의 자원 사용량을 체크한다.

```json
**# kubectl top pod**
NAME                               CPU(cores)   MEMORY(bytes)   
hpa-test-deploy-869f69b99f-tvwfc   **15m**          4Mi      
       
**# kubectl get hpa hpa-test-deploy** 
NAME              REFERENCE                    TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
hpa-test-deploy   Deployment/hpa-test-deploy   **750%/80%**   1         5         1          13m
```

cpu 사용량이 15m에 hpa에서 평균 cpu 사용량이 requests에 비해 750% 사용중임을 알리고 있다.

시간이 조금 지나면 pod가 추가로 생성된다.

```json
**# kubectl get pod**
NAME                               READY   STATUS    RESTARTS   AGE
hpa-test-deploy-869f69b99f-6l6qp   1/1     Running   0          37s
hpa-test-deploy-869f69b99f-gtd86   1/1     Running   0          37s
hpa-test-deploy-869f69b99f-lqrph   1/1     Running   0          22s
hpa-test-deploy-869f69b99f-lwvfq   1/1     Running   0          37s
hpa-test-deploy-869f69b99f-tvwfc   1/1     Running   0          20m

**# kubectl get hpa**
NAME              REFERENCE                    TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
hpa-test-deploy   Deployment/hpa-test-deploy   340%/80%   1         5         5          15m
```

물론 여전히 requests보다 많은 자원을 사용중이다.  hpa의 max 값을 올리거나 deployment에서 requests 값을 추가하거나 하자.

### 02. 커맨드로 HPA 리소스 생성하며 cpu 사용률 target 지정

<aside>
💡 **# kubectl autoscale deploy {디플로이명} --min={최소 파드 수} --max={최대 파드 수} --cpu-percent={cpu 사용량}**

</aside>

```json
**# kubectl autoscale deployment hpa-test-deploy --min=1 --max=10 --cpu-percent=90**
horizontalpodautoscaler.autoscaling/hpa-test-deploy autoscaled

**# kubectl get hpa**
NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-test-deploy   Deployment/hpa-test-deploy   0%/90%    1         10        1          69s
```

## 02. YAML로 HPA 리소스 생성

YAML로 생성할 경우 다양한 기능을 사용하기 위해 API 버전을 autoscaling/v2로 설정하자.

### 01. CPU 사용률

**autoscaling/v1**

v1 버전에서는 targetCPUUtilizationPercentage를 사용한다.

```yaml
**apiVersion: autoscaling/v1**
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-test-deploy-cpu-utilization-v1
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-test-deploy
  **targetCPUUtilizationPercentage: 85**
status:
  currentReplicas: 0
  desiredReplicas: 0
~
```

**autoscaling/v2**

v2에서는 targetCPUUtilizationPercentage이 metrics로 시작하는 값으로 변경됐다. metrics를 이용해서 다양한 metric 정보를 기준으로 설정할 수 있게 됐다.

```json
**apiVersion: autoscaling/v2**
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-test-deploy-cpu-utilization-v2
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-test-deploy
  **metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 85**
```

metrics.resource.target.type을 Utilization으로 설정하면 원하는 리소스의 평균 사용률을 기준으로 조정할 수 있다. 

### 02. 메모리 사용률

```yaml
**apiVersion: autoscaling/v2**
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-test-deploy-memory-utilization
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-test-deploy
  **metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60**
```

metrics.resource.target.type을 Utilization으로 설정하면 원하는 리소스의 평균 사용률을 기준으로 조정할 수 있다. 

### 03. CPU 사용량

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-test-deploy-cpu-value
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-test-deploy
  **metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: AverageValue
        averageValue: 20m**
```

### 04. 메모리 사용량

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-test-deploy-memory-value
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-test-deploy
  **metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 30Mi**
```

# 03. HPA custom-metrics

custom-metrics를 사용하면 굉장히 복잡한 방식으로 HPA를 구현할 수 있다. 해당 방법은 추후 정리해야겠다.
