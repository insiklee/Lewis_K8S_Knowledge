# K8s Job/CronJob 관리

# 01. Job이란?

Job은 Linux의 at 명령어와 같은 역할을 한다. 

## 01. Job 관리

### 01. 명령어로 Job 생성

<aside>
💡 **# kubectl create job {job명} --image={이미지명} -- {명령어}**

</aside>

```yaml
**$ kubectl create job nodeversion --image=node -- node -v**
job.batch/nodeversion created

**$ kubectl get job,po | grep node**
job.batch/nodeversion   0/1           11s        11s
pod/nodeversion-8zl6r               0/1     **ContainerCreating**   0          11s

**$ kubectl get job,po | grep node**
job.batch/nodeversion   1/1           20s        21s
pod/nodeversion-8zl6r               0/1     **Completed**   0          21s

**$ kubectl logs nodeversion-8zl6r** 
v18.9.0
```

### 02. YAML 파일로 Job 관리

Job의 parallelism이나 completions 같은 옵션을 부여하기 위해서는 Yaml 파일로 설정한 뒤 실행해야 한다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job2
spec:
  **parallelism: 5
  completions: 10**
  template:
    metadata:
    spec:
      containers:
      - command:
        - echo
        - Hello I'm running job
        image: busybox
        name: hello-job2
      restartPolicy: Never
```

- parallellism : 병렬도를 의미하며 5개의 job을 실행한다는 의미다
- completions : 총 10개의 job을 완료하라는 의미다.

위 yaml 예시와 같이 parallelism이 5이고 completions가 10으로 주어진다면, 한번에 5개의 job pod가 생성되며, 모두 다 해서 10개의 pod를 completions 하고 종료된다.

```json
**# kubectl apply -f hello-job2.yaml**
job.batch/hello-job2 created

# **kubectl get po | grep job2**
hello-job2-8khbr                0/1     ContainerCreating   0          6s
hello-job2-lhshg                0/1     ContainerCreating   0          6s
hello-job2-rks9q                0/1     Completed           0          6s
hello-job2-rmm4h                0/1     Completed           0          6s
hello-job2-s8fmk                0/1     ContainerCreating   0          6s

(10초 뒤)

**# kubectl get po | grep job2**
hello-job2-8khbr                0/1     Completed           0          16s
hello-job2-czkhq                0/1     Completed           0          10s
hello-job2-f5956                0/1     Completed           0          8s
hello-job2-gl5nd                0/1     ContainerCreating   0          5s
hello-job2-lhshg                0/1     Completed           0          16s
hello-job2-rks9q                0/1     Completed           0          16s
hello-job2-rmm4h                0/1     Completed           0          16s
hello-job2-rvqhg                0/1     Completed           0          8s
hello-job2-s8fmk                0/1     Completed           0          16s
hello-job2-zv5qv                0/1     Completed           0          10s

**# kubectl get job -w**
NAME          COMPLETIONS   DURATION   AGE
hello-job2    0/10          6s         6s
hello-job2    1/10          6s         6s
hello-job2    2/10          6s         6s
hello-job2    3/10          8s         8s
hello-job2    4/10          8s         8s
hello-job2    5/10          11s        11s
hello-job2    6/10          12s        12s
hello-job2    7/10          13s        13s
hello-job2    8/10          14s        14s
hello-job2    9/10          16s        16s
hello-job2    10/10         18s        18s
```

# 02. CronJob이란?

CronJob은 리눅스의 Cron과 동일한 역할을 한다. 분 시 일 월 요일을 지정해서 반복이 필요한 작업을 스케쥴링하는데 사용한다.

리눅스에서 스케쥴링을 하는데 사용하는 다섯 개의 별 표시(* * * * *)를 동일하게 사용한다. 

## 01. CronJob 관리

### 01. 명령어로 cronjob 생성

<aside>
💡 **# kubectl create cronjob {cronjob 명} --image={이미지명} --schedule=”* * * * *(스케쥴링)” -- {명령어}**

</aside>

```json
**# kubectl create cronjob test-cronjob --image=busybox --schedule="*/1 * * * *" -- date**
cronjob.batch/test-cronjob created

**# kubectl get cronjob**
NAME           SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
**test-cronjob   */1 * * * *   False     0        <none>          3s

# kubectl get pod
test-cronjob-27741920-5xsxm   0/1     Completed   0          2m38s
test-cronjob-27741921-6wgg9   0/1     Completed   0          98s
test-cronjob-27741922-8c99f   0/1     Completed   0          38s

# kubectl logs test-cronjob-27741922-8c99f** 
Fri Sep 30 05:22:03 UTC 2022
```

## 02. cronjob YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: test-cronjob
spec:
  jobTemplate:
    metadata:
      name: test-cronjob
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - command:
            - date
            image: busybox
            name: test-cronjob
          restartPolicy: OnFailure
  schedule: '*/1 * * * *'
```
