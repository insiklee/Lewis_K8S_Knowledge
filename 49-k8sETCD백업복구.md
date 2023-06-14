# K8s ETCD 백업 및 복구

# 01. ETCD 연결 확인

우선 etcd의 인증서 정보를 확인한다.

<aside>
💡 # cat /etc/kubernetes/manifests/etcd.yaml

</aside>

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://192.1
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.56.100:2379
    **- --cert-file=/etc/kubernetes/pki/etcd/server.crt**
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://192.168.56.100:2380
    - --initial-cluster=k8s-master=https://192.168.56.100:2380
    **- --key-file=/etc/kubernetes/pki/etcd/server.key**
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.5
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.56.100:2380
    - --name=k8s-master
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    **- --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
(하략)**
```

위에서 굵게 표시된 인증서 정보를 확인한 후 아래 명령어로 etcd 파드와 연결 상태를 확인한다.

<aside>
💡 # **kubectl -n kube-system exec -it etcd-k8s-master -- sh -c "ETCDCTL_API=3 \**
**ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
etcdctl endpoint health"**
127.0.0.1:2379 is healthy: successfully committed proposal: took = 6.067348ms

</aside>

# 02. ETCD 백업

<aside>
💡 # **kubectl -n kube-system exec -it etcd-k8s-master -- sh -c "ETCDCTL_API=3 etcdctl \
--endpoints=127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save /var/lib/etcd/snapshot.db"**
{"level":"info","ts":1650349973.3973117,"caller":"snapshot/v3_snapshot.go:68","msg":"created temporary db file","path":"/var/lib/etcd/snapshot.db.part"}
{"level":"info","ts":1650349973.402975,"logger":"client","caller":"v3/maintenance.go:211","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1650349973.4030206,"caller":"snapshot/v3_snapshot.go:76","msg":"fetching snapshot","endpoint":"127.0.0.1:2379"}
{"level":"info","ts":1650349973.4362574,"logger":"client","caller":"v3/maintenance.go:219","msg":"completed snapshot read; closing"}
{"level":"info","ts":1650349973.4426541,"caller":"snapshot/v3_snapshot.go:91","msg":"fetched snapshot","endpoint":"127.0.0.1:2379","size":"3.9 MB","took":"now"}
{"level":"info","ts":1650349973.442722,"caller":"snapshot/v3_snapshot.go:100","msg":"saved","path":"/var/lib/etcd/snapshot.db"}
Snapshot saved at /var/lib/etcd/snapshot.db

</aside>

# 03. ETCD 복구

<aside>
💡 **# kubectl -n kube-system exec -it etcd-k8s-master -- sh -c "ETCDCTL_API=3 etcdctl \
--endpoints=127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot restore /var/lib/etcd/snapshot.db"**
Deprecated: Use `etcdutl snapshot restore` instead.

2022-04-19T06:40:00Z	info	snapshot/v3_snapshot.go:251	restoring snapshot	{"path": "/var/lib/etcd/snapshot.db", "wal-dir": "default.etcd/member/wal", "data-dir": "default.etcd", "snap-dir": "default.etcd/member/snap", "stack": "[go.etcd.io/etcd/etcdutl/v3/snapshot.(*v3Manager](http://go.etcd.io/etcd/etcdutl/v3/snapshot.(*v3Manager)).Restore\n\t/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdutl/snapshot/v3_snapshot.go:257\[ngo.etcd.io/etcd/etcdutl/v3/etcdutl.SnapshotRestoreCommandFunc\n\t/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdutl/etcdutl/snapshot_command.go:147\ngo.etcd.io/etcd/etcdctl/v3/ctlv3/command.snapshotRestoreCommandFunc\n\t/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/ctlv3/command/snapshot_command.go:128\ngithub.com/spf13/cobra.(*Command](http://ngo.etcd.io/etcd/etcdutl/v3/etcdutl.SnapshotRestoreCommandFunc%5Cn%5Ct/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdutl/etcdutl/snapshot_command.go:147%5Cngo.etcd.io/etcd/etcdctl/v3/ctlv3/command.snapshotRestoreCommandFunc%5Cn%5Ct/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/ctlv3/command/snapshot_command.go:128%5Cngithub.com/spf13/cobra.(*Command)).execute\n\t/home/remote/sbatsche/.gvm/pkgsets/go1.16.3/global/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:856\[ngithub.com/spf13/cobra.(*Command](http://ngithub.com/spf13/cobra.(*Command)).ExecuteC\n\t/home/remote/sbatsche/.gvm/pkgsets/go1.16.3/global/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:960\[ngithub.com/spf13/cobra.(*Command](http://ngithub.com/spf13/cobra.(*Command)).Execute\n\t/home/remote/sbatsche/.gvm/pkgsets/go1.16.3/global/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:897\[ngo.etcd.io/etcd/etcdctl/v3/ctlv3.Start\n\t/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/ctlv3/ctl.go:107\ngo.etcd.io/etcd/etcdctl/v3/ctlv3.MustStart\n\t/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/ctlv3/ctl.go:111\nmain.main\n\t/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/main.go:59\nruntime.main\n\t/home/remote/sbatsche/.gvm/gos/go1.16.3/src/runtime/proc.go:225](http://ngo.etcd.io/etcd/etcdctl/v3/ctlv3.Start%5Cn%5Ct/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/ctlv3/ctl.go:107%5Cngo.etcd.io/etcd/etcdctl/v3/ctlv3.MustStart%5Cn%5Ct/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/ctlv3/ctl.go:111%5Cnmain.main%5Cn%5Ct/tmp/etcd-release-3.5.1/etcd/release/etcd/etcdctl/main.go:59%5Cnruntime.main%5Cn%5Ct/home/remote/sbatsche/.gvm/gos/go1.16.3/src/runtime/proc.go:225)"}
2022-04-19T06:40:00Z	info	membership/store.go:14Trimming membership information from the backend...
2022-04-19T06:40:00Z	info	membership/cluster.go:421	added member	{"cluster-id": "cdf818194e3a8c32", "local-member-id": "0", "added-peer-id": "8e9e05c52164694d", "added-peer-peer-urls": ["[http://localhost:2380](http://localhost:2380/)"]}
2022-04-19T06:40:00Z	info	snapshot/v3_snapshot.go:272	restored snapshot	{"path": "/var/lib/etcd/snapshot.db", "wal-dir": "default.etcd/member/wal", "data-dir": "default.etcd", "snap-dir": "default.etcd/member/snap"}

</aside>
