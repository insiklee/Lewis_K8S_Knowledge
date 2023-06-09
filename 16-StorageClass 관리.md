# K8s StorageClass 관리

[Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

# 01. StorageClass란?

 Dynamic Persistent Volume을 구성하는데 반드시 필요한 리소스다.

# 02. StorageClass 예시

## 01. Yaml 예시

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

### 01. provisioner

provisioner는 PV가 프로비전된 볼륨 플러그인을 설정한다. Dynamic Persistent Volume을 구성하기 위해 필요한다. Provisioner 종류는 [여기](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)를 참조하자

provisioner는 internal과 external이 있다. internal의 경우 쿠버네티스 설치와 동시에 provisioner가 설치되기 때문에 Prefix로 kubernetes.io만 추가하면 바로 사용할 수 있다.

external의 경우 별도로 provisioner를 설치해줘야만 사용할 수 있다.

### 02. allowvolumeExpansion

PVC는 한 번 바운드되면 미리 request한 용량을 수정할 수 없다. 다만 StorageClass에서 해당 설정을 추가해준다면 PVC의 용량을 수정할 수 있다.

# 03. Dynamic Persistent Volume 구성

당장 구성 가능한 것부터 시작하여 차근차근 정리하자.

## 01. NFS

NFS는 internal provisioner가 없다. 따라서 별도로 provisioner를 설치해야한다.

### 01. nfs-ganesha-server-and-external-provisioner

nfs-ganesha-server-and-external-provisioner는 NFS 서비스를 제공하는 Pod를 생성한 뒤, 각 노드의 /srv 디렉터리를 NFS로 export 하여 자동으로 NFS를 제공하는 provisioner다.

노드의 용량을 차지하기 때문에 그다지 추천하지 않는 provisioner다. 그보다는 NFS 서버를 따로 두고 사용하는 방식을 쓰는 것이 더 좋다.

우선 git으로 매니페스트를 다운받는다.

<aside>
💡 **# git clone https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner.git**

</aside>

```json
**# git clone https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner.git**
Cloning into 'nfs-ganesha-server-and-external-provisioner'...
remote: Enumerating objects: 6997, done.
remote: Counting objects: 100% (273/273), done.
remote: Compressing objects: 100% (140/140), done.
remote: Total 6997 (delta 120), reused 247 (delta 111), pack-reused 6724
Receiving objects: 100% (6997/6997), 7.18 MiB | 17.70 MiB/s, done.
Resolving deltas: 100% (3265/3265), done.

# **cd nfs-ganesha-server-and-external-provisioner/**
```

provisioner를 관리할 네임스페이스를 생성한다.

```json
**# kubectl create ns provisioner-nfs-ganesha**
namespace/provisioner-nfs-ganesha created
```

```json
# vi deploy/kubernetes/deployment.yaml
---------------deploy/kubernetes/deployment.yaml-------------------
(전략)
              args:
            #   - "-provisioner=example.com/nfs" <- 삭제
                - "-provisioner=k8s.unnet.io/nfs-ganesha" <- 추가
(하략)
-------------------------------------------------------------------

**# kubectl apply -f deploy/kubernetes/deployment.yaml -n provisioner-nfs-ganesha** 
serviceaccount/nfs-provisioner created
service/nfs-provisioner created
deployment.apps/nfs-provisioner created
```

프로비저너가 PV를 만들기 위해서는 clusterrole과 role을 생성해야 한다. 이와 연결할 ServiceAccount의 네임스페이스 정보를 설정해준다. 34번째 줄과 57번째 줄이다.

```json
**# vi deploy/kubernetes/rbac.yaml**
-----------------------deploy/kubernetes/rbac.yaml--------------------------
(전략)
30 subjects:
 31   - kind: ServiceAccount
 32     name: nfs-provisioner
 33      # replace with namespace where provisioner is deployed
 34     **namespace: provisioner-nfs-ganesha**
(중략)
53 subjects:
 54   - kind: ServiceAccount
 55     name: nfs-provisioner
 56     # replace with namespace where provisioner is deployed
 57      **namespace: provisioner-nfs-ganesha**
(하략)
------------------------------------------------------------------------------

**# kubectl apply -f deploy/kubernetes/rbac.yaml -n provisioner-nfs-ganesha** 
clusterrole.rbac.authorization.k8s.io/nfs-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
```

이제 스토리키즐래스 리소스를 정의하자. nfs-ganesha는 hostPath로 마운트하여 노드에다 직접 NFS 서버를 생성하기 때문에 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-nfs
provisioner: k8s.unnet.io/nfs-ganesha
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
- vers: 4.1
```

```json
**# kubectl apply -f sc-nfs.yaml** 
storageclass.storage.k8s.io/sc-nfs created

**# kubectl get sc**
NAME     PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
**sc-nfs**   k8s.unnet.io/nfs-ganesha   Retain          Immediate           true                   4m16s
```

- **마운트 테스트**
    
    PVC를 생성하자.
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs-claim
    spec:
      storageClassName: sc-nfs
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
    ```
    
    ```json
    **# kubectl apply -f nfs-claim.yaml** 
    persistentvolumeclaim/nfs-claim unchanged
    
    **# kubectl get pvc**
    NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    nfs-claim   Bound    pvc-aa416224-a874-4679-a475-91a5ef149d45   3Gi        RWO            sc-nfs         4s
    
    **# kubectl get pv**
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
    pvc-aa416224-a874-4679-a475-91a5ef149d45   3Gi        RWO            Retain           Bound    default/nfs-claim   sc-nfs                  112s
    ```
    
    worker 노드에 PV를 위한 NFS 볼륨이 생성됐는지 확인해본다.
    
    ```json
    $ ls /srv
    ganesha.log               **pvc-aa416224-a874-4679-a475-91a5ef149d45**  v4recov
    nfs-provisioner.identity  v4old                                     vfs.conf
    ```
    

### 02. nfs-subdir-external-provisioner

이미 존재하고 있는 NFS 서버에 subdir을 생성하여 PV를 생성하는 방식이다.

해당 provisioner는 kustomize 방식으로 설치할 예정이다. 이를 위해 kustomization.yaml 파일을 생성한다.

```yaml
# mkdir nfs-subdir
# cd nfs-subdir
# vi kustomization.yaml
--------------------nfs-subdir/kustomization.yaml-----------------------
namespace: provisioner-nfs-subdir
bases:
  - github.com/kubernetes-sigs/nfs-subdir-external-provisioner//deploy
resources:
  - namespace.yaml
patchesStrategicMerge:
  - patch_nfs_details.yaml
------------------------------------------------------------------------
```

네임스페이스를 정의한다.

```yaml
# vi namespace.yaml
-----------------nfs-subdir/namespace.yaml---------------------
apiVersion: v1
kind: Namespace
metadata:
  name: provisioner-nfs-subdir
---------------------------------------------------------------
```

기존 매니페스트에서 덮어쓰기를 할 deployment patch 파일을 생성한다.

```yaml
# vi patch_nfs_details.yaml
-----------------nfs-subdir/patch_nfs_details.yaml-------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nfs-client-provisioner
  name: nfs-client-provisioner
spec:
  template:
    spec:
      containers:
        - name: nfs-client-provisioner
          env:
            - name: PROVISIONER_NAME
              **value: k8s.unnet.io/nfs-subdir  # provisioner 이름 편하게 수정**
            - name: NFS_SERVER
              **value: 192.168.122.1   # NFS 서버 주소 수정**
            - name: NFS_PATH
              **value: /NFS # NFS 경로 수정**
      volumes:
        - name: nfs-client-root
          nfs:
            **server: 192.168.122.1   # NFS 서버 주소 수정
            path: /NFS               # NFS 경로 수정**
```

kustomize로 디플로이한다.

```json
# kubectl apply -k .
namespace/provisioner-nfs-subdir created
storageclass.storage.k8s.io/nfs-client unchanged
serviceaccount/nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner unchanged
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner configured
deployment.apps/nfs-client-provisioner created
student@k8s-master:~/nfs-subdir$ kubectl get deploy -n provisioner-nfs-subdir
```

storageClass를 생성하자.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-nfs-subdir
provisioner: k8s.unnet.io/nfs-subdir
reclaimPolicy: Retain
allowVolumeExpansion: true
```

```json
# kubectl apply -f sc-nfs-subdir.yaml
storageclass.storage.k8s.io/sc-nfs-subdir created

**# kubectl get sc**
NAME            PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
sc-nfs-subdir   k8s.unnet.io/nfs-subdir                       Retain          Immediate           true                   6s
```

- **마운트 테스트**
    
    pvc를 생성하자
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-nfs-subdir
    spec:
      storageClassName: sc-nfs-subdir
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
    ```
    
    ```json
    **# kubectl apply -f pvc-nfs-subdir.yaml** 
    persistentvolumeclaim/pvc-nfs-subdir created
    
    **# kubectl get pvc**
    NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
    pvc-nfs-subdir                          Bound    pvc-82083396-9c65-47a9-8bda-995c7473c91b   3Gi        RWO            sc-nfs-subdir   17s
    
    **# kubectl get pv**
    pvc-82083396-9c65-47a9-8bda-995c7473c91b   3Gi        RWO            Retain           Bound    default/pvc-nfs-subdir                          sc-nfs-subdir            7m1s
    ```
    

## 02. GlusterFS

GlusterFS는 internal provisioner가 있기 때문에 별도로 provisioner를 설치할 필요는 없다. 다만, heketi를 설치해서 RESTful API로 접근해야 한다. 

### 01. glusterfs 구축 및 heketi 설치

[glusterFS](https://www.notion.so/glusterFS-1a2d290154724c4bb2b45d307bc76843?pvs=21) 에서 구축 및 heketi 설치 방법을 확인할 수 있다.

### 02. secret 생성

heketi에 접속하기 위해서는 rest 계정의 ID와 비밀번호가 필요하다. 여기서 비밀번호를 평문으로 저장할 경우 보안상 문제가 될 수 있기 때문에 secret으로 비밀번호를 관리한다.

<aside>
💡 **# kubectl create secret generic {시크릿 명} --type="kubernetes.io/glusterfs" --from-literal=key={hehti admin 비밀번호}**

</aside>

```json
**# kubectl create secret generic heketi-secret --type="kubernetes.io/glusterfs" --from-literal=key="P@ssw0rd"**
secret/heketi-secret created

**# kubectl get secret heketi-secret** 
NAME            TYPE                      DATA   AGE
heketi-secret   kubernetes.io/glusterfs   1      40s
```

### 03. StorageClass Yaml 생성

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: http://192.168.2.21:8080
  restuser: admin
  secretNamespace: default
  secretName: heketi-secret
  clusterid: 13572cb0348472df511c8d7aac04b6a
  volumetype: replica:2
reclaimPolicy: Retain
allowVolumeExpansion: true
```

파라미터에 대한 정보는 아래와 같다.

- resturl : heketi 서버의 주소다
- restuser : heketi에 접속할 유저 명이다.
- secretName:  heketi 접속에 사용할 비밀번호를 저장한 시크릿 이름이다.
- secretNamespace : 위 시크릿이 속한 네임스페이스 정보다
- clusterid : 현 스토리지클래스에서 사용할 clusterid 정보다. heketi 서버 하나에서 관리하는 클러스터가 여러개 일 수 있기 때문에 지정해준다.
- volumetype : glusterfs의 어떤 볼륨 타입을 사용할지 결정한다. 자세한 내용은 [03. GlusterFS 볼륨 생성](https://www.notion.so/03-GlusterFS-c7dcf3208b694b43b7da4f9fe1d4db71?pvs=21) 의 볼름 생성 항목을 확인한다.

```json
**# kubectl apply -f glusterfs.yaml** 
storageclass.storage.k8s.io/glusterfs created
```

### 04. 테스트

PVC를 하나 만들자.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glusterfs-pvc
  namespace: default
spec:
  storageClassName: glusterfs
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

PVC를 실행한다.

```json
**# kubectl apply -f glusterfs-pvc.yaml** 
persistentvolumeclaim/glusterfs-pvc created

**# kubectl get pvc**
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
glusterfs-pvc   Bound    **pvc-7be531e8-7a21-417f-8725-e69118db3ccd**   1Gi        RWO            glusterfs      17s

**# kubectl get pv**
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
pvc-7be531e8-7a21-417f-8725-e69118db3ccd   1Gi        RWO            Retain           Bound    default/glusterfs-pvc   glusterfs               73s
```

PV가 자동으로 생성된 것을 확인할 수 있다.# K8s StorageClass 관리

[Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

# 01. StorageClass란?

 Dynamic Persistent Volume을 구성하는데 반드시 필요한 리소스다.

# 02. StorageClass 예시

## 01. Yaml 예시

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

### 01. provisioner

provisioner는 PV가 프로비전된 볼륨 플러그인을 설정한다. Dynamic Persistent Volume을 구성하기 위해 필요한다. Provisioner 종류는 [여기](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)를 참조하자

provisioner는 internal과 external이 있다. internal의 경우 쿠버네티스 설치와 동시에 provisioner가 설치되기 때문에 Prefix로 kubernetes.io만 추가하면 바로 사용할 수 있다.

external의 경우 별도로 provisioner를 설치해줘야만 사용할 수 있다.

### 02. allowvolumeExpansion

PVC는 한 번 바운드되면 미리 request한 용량을 수정할 수 없다. 다만 StorageClass에서 해당 설정을 추가해준다면 PVC의 용량을 수정할 수 있다.

# 03. Dynamic Persistent Volume 구성

당장 구성 가능한 것부터 시작하여 차근차근 정리하자.

## 01. NFS

NFS는 internal provisioner가 없다. 따라서 별도로 provisioner를 설치해야한다.

### 01. nfs-ganesha-server-and-external-provisioner

nfs-ganesha-server-and-external-provisioner는 NFS 서비스를 제공하는 Pod를 생성한 뒤, 각 노드의 /srv 디렉터리를 NFS로 export 하여 자동으로 NFS를 제공하는 provisioner다.

노드의 용량을 차지하기 때문에 그다지 추천하지 않는 provisioner다. 그보다는 NFS 서버를 따로 두고 사용하는 방식을 쓰는 것이 더 좋다.

우선 git으로 매니페스트를 다운받는다.

<aside>
💡 **# git clone https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner.git**

</aside>

```json
**# git clone https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner.git**
Cloning into 'nfs-ganesha-server-and-external-provisioner'...
remote: Enumerating objects: 6997, done.
remote: Counting objects: 100% (273/273), done.
remote: Compressing objects: 100% (140/140), done.
remote: Total 6997 (delta 120), reused 247 (delta 111), pack-reused 6724
Receiving objects: 100% (6997/6997), 7.18 MiB | 17.70 MiB/s, done.
Resolving deltas: 100% (3265/3265), done.

# **cd nfs-ganesha-server-and-external-provisioner/**
```

provisioner를 관리할 네임스페이스를 생성한다.

```json
**# kubectl create ns provisioner-nfs-ganesha**
namespace/provisioner-nfs-ganesha created
```

```json
# vi deploy/kubernetes/deployment.yaml
---------------deploy/kubernetes/deployment.yaml-------------------
(전략)
              args:
            #   - "-provisioner=example.com/nfs" <- 삭제
                - "-provisioner=k8s.unnet.io/nfs-ganesha" <- 추가
(하략)
-------------------------------------------------------------------

**# kubectl apply -f deploy/kubernetes/deployment.yaml -n provisioner-nfs-ganesha** 
serviceaccount/nfs-provisioner created
service/nfs-provisioner created
deployment.apps/nfs-provisioner created
```

프로비저너가 PV를 만들기 위해서는 clusterrole과 role을 생성해야 한다. 이와 연결할 ServiceAccount의 네임스페이스 정보를 설정해준다. 34번째 줄과 57번째 줄이다.

```json
**# vi deploy/kubernetes/rbac.yaml**
-----------------------deploy/kubernetes/rbac.yaml--------------------------
(전략)
30 subjects:
 31   - kind: ServiceAccount
 32     name: nfs-provisioner
 33      # replace with namespace where provisioner is deployed
 34     **namespace: provisioner-nfs-ganesha**
(중략)
53 subjects:
 54   - kind: ServiceAccount
 55     name: nfs-provisioner
 56     # replace with namespace where provisioner is deployed
 57      **namespace: provisioner-nfs-ganesha**
(하략)
------------------------------------------------------------------------------

**# kubectl apply -f deploy/kubernetes/rbac.yaml -n provisioner-nfs-ganesha** 
clusterrole.rbac.authorization.k8s.io/nfs-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
```

이제 스토리키즐래스 리소스를 정의하자. nfs-ganesha는 hostPath로 마운트하여 노드에다 직접 NFS 서버를 생성하기 때문에 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-nfs
provisioner: k8s.unnet.io/nfs-ganesha
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
- vers: 4.1
```

```json
**# kubectl apply -f sc-nfs.yaml** 
storageclass.storage.k8s.io/sc-nfs created

**# kubectl get sc**
NAME     PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
**sc-nfs**   k8s.unnet.io/nfs-ganesha   Retain          Immediate           true                   4m16s
```

- **마운트 테스트**
    
    PVC를 생성하자.
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs-claim
    spec:
      storageClassName: sc-nfs
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
    ```
    
    ```json
    **# kubectl apply -f nfs-claim.yaml** 
    persistentvolumeclaim/nfs-claim unchanged
    
    **# kubectl get pvc**
    NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    nfs-claim   Bound    pvc-aa416224-a874-4679-a475-91a5ef149d45   3Gi        RWO            sc-nfs         4s
    
    **# kubectl get pv**
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
    pvc-aa416224-a874-4679-a475-91a5ef149d45   3Gi        RWO            Retain           Bound    default/nfs-claim   sc-nfs                  112s
    ```
    
    worker 노드에 PV를 위한 NFS 볼륨이 생성됐는지 확인해본다.
    
    ```json
    $ ls /srv
    ganesha.log               **pvc-aa416224-a874-4679-a475-91a5ef149d45**  v4recov
    nfs-provisioner.identity  v4old                                     vfs.conf
    ```
    

### 02. nfs-subdir-external-provisioner

이미 존재하고 있는 NFS 서버에 subdir을 생성하여 PV를 생성하는 방식이다.

해당 provisioner는 kustomize 방식으로 설치할 예정이다. 이를 위해 kustomization.yaml 파일을 생성한다.

```yaml
# mkdir nfs-subdir
# cd nfs-subdir
# vi kustomization.yaml
--------------------nfs-subdir/kustomization.yaml-----------------------
namespace: provisioner-nfs-subdir
bases:
  - github.com/kubernetes-sigs/nfs-subdir-external-provisioner//deploy
resources:
  - namespace.yaml
patchesStrategicMerge:
  - patch_nfs_details.yaml
------------------------------------------------------------------------
```

네임스페이스를 정의한다.

```yaml
# vi namespace.yaml
-----------------nfs-subdir/namespace.yaml---------------------
apiVersion: v1
kind: Namespace
metadata:
  name: provisioner-nfs-subdir
---------------------------------------------------------------
```

기존 매니페스트에서 덮어쓰기를 할 deployment patch 파일을 생성한다.

```yaml
# vi patch_nfs_details.yaml
-----------------nfs-subdir/patch_nfs_details.yaml-------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nfs-client-provisioner
  name: nfs-client-provisioner
spec:
  template:
    spec:
      containers:
        - name: nfs-client-provisioner
          env:
            - name: PROVISIONER_NAME
              **value: k8s.unnet.io/nfs-subdir  # provisioner 이름 편하게 수정**
            - name: NFS_SERVER
              **value: 192.168.122.1   # NFS 서버 주소 수정**
            - name: NFS_PATH
              **value: /NFS # NFS 경로 수정**
      volumes:
        - name: nfs-client-root
          nfs:
            **server: 192.168.122.1   # NFS 서버 주소 수정
            path: /NFS               # NFS 경로 수정**
```

kustomize로 디플로이한다.

```json
# kubectl apply -k .
namespace/provisioner-nfs-subdir created
storageclass.storage.k8s.io/nfs-client unchanged
serviceaccount/nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner unchanged
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner configured
deployment.apps/nfs-client-provisioner created
student@k8s-master:~/nfs-subdir$ kubectl get deploy -n provisioner-nfs-subdir
```

storageClass를 생성하자.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-nfs-subdir
provisioner: k8s.unnet.io/nfs-subdir
reclaimPolicy: Retain
allowVolumeExpansion: true
```

```json
# kubectl apply -f sc-nfs-subdir.yaml
storageclass.storage.k8s.io/sc-nfs-subdir created

**# kubectl get sc**
NAME            PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
sc-nfs-subdir   k8s.unnet.io/nfs-subdir                       Retain          Immediate           true                   6s
```

- **마운트 테스트**
    
    pvc를 생성하자
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-nfs-subdir
    spec:
      storageClassName: sc-nfs-subdir
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
    ```
    
    ```json
    **# kubectl apply -f pvc-nfs-subdir.yaml** 
    persistentvolumeclaim/pvc-nfs-subdir created
    
    **# kubectl get pvc**
    NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
    pvc-nfs-subdir                          Bound    pvc-82083396-9c65-47a9-8bda-995c7473c91b   3Gi        RWO            sc-nfs-subdir   17s
    
    **# kubectl get pv**
    pvc-82083396-9c65-47a9-8bda-995c7473c91b   3Gi        RWO            Retain           Bound    default/pvc-nfs-subdir                          sc-nfs-subdir            7m1s
    ```
    

## 02. GlusterFS

GlusterFS는 internal provisioner가 있기 때문에 별도로 provisioner를 설치할 필요는 없다. 다만, heketi를 설치해서 RESTful API로 접근해야 한다. 

### 01. glusterfs 구축 및 heketi 설치

[glusterFS](https://www.notion.so/glusterFS-1a2d290154724c4bb2b45d307bc76843?pvs=21) 에서 구축 및 heketi 설치 방법을 확인할 수 있다.

### 02. secret 생성

heketi에 접속하기 위해서는 rest 계정의 ID와 비밀번호가 필요하다. 여기서 비밀번호를 평문으로 저장할 경우 보안상 문제가 될 수 있기 때문에 secret으로 비밀번호를 관리한다.

<aside>
💡 **# kubectl create secret generic {시크릿 명} --type="kubernetes.io/glusterfs" --from-literal=key={hehti admin 비밀번호}**

</aside>

```json
**# kubectl create secret generic heketi-secret --type="kubernetes.io/glusterfs" --from-literal=key="P@ssw0rd"**
secret/heketi-secret created

**# kubectl get secret heketi-secret** 
NAME            TYPE                      DATA   AGE
heketi-secret   kubernetes.io/glusterfs   1      40s
```

### 03. StorageClass Yaml 생성

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: http://192.168.2.21:8080
  restuser: admin
  secretNamespace: default
  secretName: heketi-secret
  clusterid: 13572cb0348472df511c8d7aac04b6a
  volumetype: replica:2
reclaimPolicy: Retain
allowVolumeExpansion: true
```

파라미터에 대한 정보는 아래와 같다.

- resturl : heketi 서버의 주소다
- restuser : heketi에 접속할 유저 명이다.
- secretName:  heketi 접속에 사용할 비밀번호를 저장한 시크릿 이름이다.
- secretNamespace : 위 시크릿이 속한 네임스페이스 정보다
- clusterid : 현 스토리지클래스에서 사용할 clusterid 정보다. heketi 서버 하나에서 관리하는 클러스터가 여러개 일 수 있기 때문에 지정해준다.
- volumetype : glusterfs의 어떤 볼륨 타입을 사용할지 결정한다. 자세한 내용은 [03. GlusterFS 볼륨 생성](https://www.notion.so/03-GlusterFS-c7dcf3208b694b43b7da4f9fe1d4db71?pvs=21) 의 볼름 생성 항목을 확인한다.

```json
**# kubectl apply -f glusterfs.yaml** 
storageclass.storage.k8s.io/glusterfs created
```

### 04. 테스트

PVC를 하나 만들자.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glusterfs-pvc
  namespace: default
spec:
  storageClassName: glusterfs
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

PVC를 실행한다.

```json
**# kubectl apply -f glusterfs-pvc.yaml** 
persistentvolumeclaim/glusterfs-pvc created

**# kubectl get pvc**
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
glusterfs-pvc   Bound    **pvc-7be531e8-7a21-417f-8725-e69118db3ccd**   1Gi        RWO            glusterfs      17s

**# kubectl get pv**
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
pvc-7be531e8-7a21-417f-8725-e69118db3ccd   1Gi        RWO            Retain           Bound    default/glusterfs-pvc   glusterfs               73s
```

PV가 자동으로 생성된 것을 확인할 수 있다.
