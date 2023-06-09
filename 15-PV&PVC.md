# K8s PV/PVC 관리

# 01. PV 및 PVC란?

[컨테이너 스토리지 원리](https://www.notion.so/adbce942bba944e2b1a80ef5907c77d1?pvs=21)

일단 컨테이너 스토리지 원리를 알아보고 오자. 컨테이너 이미지는 쓰기가 불가능한 여러개의 레이어가 층층이 쌓여있는 구조이며, 유일하게 쓰기가 가능한 컨테이너 레이어가 최 상단에 위치한 구조다.  문제는 이 컨테이너 레이어가 스토리지 오버레이에서 실행된다는 점이다. 컨테이너가 종료됨과 동시에 오버레이 역시 삭제되며, 오버레이 경로에 포함된 컨테이너 레이어 역시 삭제된다.

이때문에 DB와 같이 데이터가 유지될 필요가 있는 컨테이너의 경우 호스트의 특정 경로를 컨테이너에 마운트한다. [컨테이너 스토리지 연결](https://www.notion.so/3842804c49f44adba545300ec3aa842e?pvs=21) 여기를 읽으면 컨테이너 볼륨 마운트 방법이 소개되어 있다.

쿠버네티스에서도 파드에서 실행되는 컨테이너의 데이터 가용성, 무결성을 유지하기 위해 **Persistant Volume(PV)**이라는 리소스를 제공한다. 호스트나 NFS, GlusterFS등의 공유 스토리지를 지정하여 컨테이너가 언제든 마운트 할 수 있도록 준비하도록 대기한다.

**Persistant Volume Claim(PVC)은** 말 그대로 PV 사용을 요청하는 리소스다. 하나의 PV와 묶여서(Bound) 사용된다. 또 Pod가 직접 연결되는 매개체이기도 하다.

**PV와 PVC를 굳이 분리한 이유는?**

PV와 PVC를 분리한 이유는 PVC와 연결된 Pod가 어떤 스토리지 구조에서도 volumeMount를 할 수 있도록 하기 위해서다. PV는 스토리지 종속적인 리소스로 스토리지 종류에 따라서 설정하는 내용이 달라진다. (NFS, GlusterFS, Hostlocal 등) PVC는 필요한 용량과 AccessMode에 따라 스스로 PV를 찾아서 바운드 해준다. 굳이 Pod나 PVC 단위에서 스토리지 구조에 대해서 깊게 알 필요가 없는 것이다.

이러한 특성 때문에 Cloud Native의 구현, 즉 벤더 종속성을 최소화하여 베어메탈이나 퍼블릭 클라우드(AWS,Azure,GCP) 간 어플리케이션 마이그레이션을 용이케 하는데 도움이 된다.

# 02. PV accessMode

PV는 세가지 AccessMode가 있다. yaml에서는 accessModes:로 지정한다.

- ReadWriteOnce : 하나의 노드가 볼륨을 RW하도록 마운트
- ReadOnlyMany : 여러개의 노드가 Read 전용으로 사용하도록 마운트
- ReadWriteMany: 여러개의 노드가 RW하도록 마운트

# 03. PV Reclaim Policy

PVC와 바운드를 해제할때 기존 PV에 저장된 데이터를 어떻게 처리할지 결정한다.

- Retain : PV 및 데이터 보존
- Recycle : PV안의 데이터를 전부 삭제한 후 PV 재사용
- Delete : PV 삭제

# 04.yaml 예시

[Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)

## 01. PV 예시

### 01. local

특정 노드의 디스크나 파티션, 디렉터리를 선택한다. 데이터의 영속성과 무결성을 위해서 특정 노드에서만 실행될 수 있도록 nodeAffinity 옵션이 함께 지정되야한다. local 스토리지에 사용된 노드에 문제가 생긴다면, 해당 PV를 마운트한 Pod 역시 제대로 동작하지 않기 때문에, local의 사용은 매우 제한적일 수 밖에 없다. 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  **local:
    path: /data_dir
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - {key: kubernetes.io/hostname, operator: In, values: [k8s-node1]}**
```

### 02. hostPath

hostPath는 호스트의 파일시스템에 속한 파일이나 디렉터리를 파드에서 사용할 수 있도록 해준다. 호스트 서버에 직접 마운트하기 때문에 보안 문제가 발생할 가능성이 매우 높기 때문에 반드시 필요한 경우가 아니면 그다지 권장하지 않는다.

**local과 달리 nodeAffinity를 사용하지 않아도 된다. hostPath는 주로 Daemonset과 함께 사용되어 호스트 노드의 성능을 모니터링하거나 특정 기능을 사용하는 것이 목적이기 때문이다.** 프로메테우스 같은 노드 상태 모니터링이나 메트릭 서버의 노드 자원 모니터링 등을 하기 위해서는 /sys 디렉터리를 파드가 접근할 필요가 있다. 또 Calico에서 CNI를 구성할때 노드의 네트워크 관련 자원이 필요하다. 이 때 hostPath를 사용하게 되며, 데이터 무결성이 중요하지 않기 때문에 nodeAffinity가 필요 없다. 어차피 hostPath에 연결할 경로는 ReadOnly로 지정해야만 하며, root 사용자 외에는 쓰기가 불가능해야 한다. 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  **hostPath:
    path: /data_dir
    type: DirectoryOrCreate**
```

hostPath의 Type은 아래와 같은 종류가 있다.

- “”(공백) : hostPath 볼륨을 마운트할때 아무런 체크를 하지 않는다는 뜻이다.
- DirectoryOrCreate : 디렉터리를 볼륨으로 사용하는데, 해당 경로가 존재하지 않는다면 디렉터리를 생성한다.
- Directory : 디렉터리가 반드시 있으야 마운트 할 수 있다.
- FileOrCreate : 파일을 사용할건데, 해당 파일이 존재하지 않는다면 생성한다.
- File : 파일이 반드시 있어야 사용할 수 있다.
- Socket : Unix Socket 파일이 반드시있어야 한다.
- CharDevice : Character 디바이스가 반드시 있어야 한다.
- BlockDevice : 블록 디바이스가 반드시 있어야 한다.

### 03. NFS

PV를 NFS에 연결하는 방법이다. 설정하기 매우 간편하고, 데이터의 영속성, 무결성도 유지할 수 있다. 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  mountOptions:
    - hard
  **nfs:
    path: /data_dir
    server: 192.168.56.101**
```

## 02. PVC Yaml 예제

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

d
