# K8s POD 우선순위 관리

# 01. PriorityClass

[Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)

## 01. PriorityClass 개요

파드에 우선순위를 부여할 수 있는 리소스다. 새로 생성되는 파드가 리소스 부족이나 기타 이유로 스케쥴되지 못할 경우, 파드에 부여된 우선순위에 따라서 스케쥴을 조정한다. 새로 생성되는 pod보다 우선순위가 낮은 pod는 실행이 중지되어 새 POD가 실행될 공간을 만들어 준다.

## 02. PriorityClass 생성

### 01. YAML 예시

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

- value : 우선순위를 가르기 위한 값이다. 값이 클 수록 우선순위가 더 커진다. 32비트 정수형이며 10억보다 작은 수가 들어간다.
- globalDefault : priorityClassName을 지정하지 않아도 모든 파드에 우선순위를 부여할때 사용한다. 모든 priorityClass 중 단 하나만 true로 설정할 수 있다.
- description : priorityClass의 설명란이다. 어느 종류의 파드나 서비스에 적용할지를 설명하는 일종의 가이드문을 넣어준다.

### 02. globalDefault 적용 원리

- globalDefault가 적용된 priorityClass는 오로지 하나만 생성할 수 있다.
- globalDefault가 없을 경우, priorityClassName을 사용하지 않은 Pod의 우선순위 값은 0이다.
- globalDefault가 적용된 priorityClass가 생성됐을때, 기존에 존재하던 Pod에게는 우선순위가 부여되지 않는다.

### 03. PriorityClass로 Pod에 우선순위 설정

일단 아래 PriorityClass를 실행한다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 10000000
globalDefault: false
description: "Test Priority Class"
```

```json
**# kubectl apply -f high-priority.yaml** 
priorityclass.scheduling.k8s.io/high-priority created

**# kubectl get priorityclass**
NAME                      VALUE        GLOBAL-DEFAULT   AGE
**high-priority             10000000     false            9m14s**
system-cluster-critical   2000000000   false            5d22h
system-node-critical      2000001000   false            5d22h
```

**참고)** system-cluster-critical과 system-node-critical은 클러스터가 생성될때 자동으로 생성되는 PriorityClass다. kube-apiserver와 같이 클러스터 운영에 필수적인 파드가 사용한다.

PriorityClass를 실행하고 난 뒤 새로 생성하는 Pod에 아래와같이 priorityClassName을 지정한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: priority-test
  name: priority-test
spec:
  priorityClassName: high-priority
  containers:
  - image: nginx
    name: priority-test
```

```json
**# kubectl apply -f priority-test.yaml** 
pod/priority-test created

**# kubectl get pod priority-test -o jsonpath='{.spec.priority}{"\t"}{.spec.priorityClassName}{"\n"}'**
10000000	high-priority
```

## 03. Preemption과 Non-Preemption

### 01. preemption 개요

Preemption은 선점이라는 뜻이다. 우선순위가 낮은 파드의 자리를 우선순위가 높은 파드가 빼앗아 가는 것을 의미한다. 

파드가 처음 생성되면 스케쥴이 될때 까지 대기열에 포함된다. 클러스터의 스케쥴러는 대기열에 있는 파드를 선택하여 노드에 할당한다. 그런데 만약 스케쥴러가 파드를 할당할 수 있는 노드를 찾을 수 없다면 우선순위에 따라서 기존의 파드를 evict(축출)한 뒤, 그 자리에 우선순위가 높은 파드를 스케쥴링한다.

### 02. default preemptionPolicy

Pod의 premption Policy는 Priority Admission Controller에 의해서 **기본값으로 PreemptLowerPriority**이모든 파드에 부여된다. 문자 그대로 해당 파드가 스케쥴되지 못할 때, 그보다 우선순위가 낮은 파드를 preemption하겠다는 뜻이다.

만약 파드에서 PreemptionPolicy를 다른 값(Never)으로 수정하려고 하면 아래와 같이 에러가 발생한다.

```yaml
Error from server (Forbidden): error when creating "priority-test.yaml": pods "priority-test2" is forbidden: the string value of PreemptionPolicy (Never) must not be provided in pod spec; priority admission controller computed PreemptLowerPriority from the given PriorityClass name
```

### 03. Non-Preemption
Preemption Policy에는 Never가 있다. **우선순위가 높은 파드가 스케쥴되지 못해 pending 상태에 있다 하더라도 기존 파드를 preemption하지 않고 자연스럽게 할당 공간이 나올 때 까지 대기하도록 하는 Preemption Policy다.** 

파드에 직접 부여할 순 없고 PriorityClass에 부여해야한다.

예시는 아래와 같다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
**preemptionPolicy: Never**
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```
