# AppArmor

[AppArmor](https://apparmor.net/)

[AppArmor를 사용하여 리소스에 대한 컨테이너의 접근 제한](https://kubernetes.io/ko/docs/tutorials/security/apparmor/)

# 01. AppArmor란?

AppArmor는 Fedora 계열 리눅스에서 사용하는 SELinux와 같은 역할을 한다. MAC(Mandatory Access Control)에 속하는 보안 솔루션으로 개별 어플리케이션의 파일, capability, 네트워크 엑세스 등을 통제할 수 있다.

Fedora 계열 리눅스(RHEL, CentOS, Rocky)에서는 SELinux가 있기 때문에 AppArmor를 딱히 제공하지 않는다.

# 02. AppArmor 설치

## 01. 데비안 계열

<aside>
💡 **# apt install -y apparmor-utils**

</aside>

# 03. AppArmor 프로필 관리

## 01. 프로필 목록 확인

<aside>
💡 **# aa-status**

</aside>

```json
**# aa-status**
apparmor module is loaded.
36 profiles are loaded.
36 profiles are in enforce mode.
   /snap/snapd/17029/usr/lib/snapd/snap-confine
   /snap/snapd/17029/usr/lib/snapd/snap-confine//mount-namespace-capture-helper
   /snap/snapd/17336/usr/lib/snapd/snap-confine
   /snap/snapd/17336/usr/lib/snapd/snap-confine//mount-namespace-capture-helper
   /usr/bin/evince
   /usr/bin/evince-previewer
   /usr/bin/evince-previewer//sanitized_helper
   /usr/bin/evince-thumbnailer
   /usr/bin/evince//sanitized_helper
   /usr/bin/man
   /usr/lib/NetworkManager/nm-dhcp-client.action
   /usr/lib/NetworkManager/nm-dhcp-helper
   /usr/lib/connman/scripts/dhclient-script
   /usr/lib/cups/backend/cups-pdf
   /usr/lib/snapd/snap-confine
   /usr/lib/snapd/snap-confine//mount-namespace-capture-helper
   /usr/sbin/chronyd
   /usr/sbin/cups-browsed
   /usr/sbin/cupsd
   /usr/sbin/cupsd//third_party
   /usr/sbin/tcpdump
   /{,usr/}sbin/dhclient
   cri-containerd.apparmor.d
   ippusbxd
   lsb_release
   man_filter
   man_groff
   nvidia_modprobe
   nvidia_modprobe//kmod
   snap-update-ns.snap-store
   snap-update-ns.yq
   snap.snap-store.hook.configure
   snap.snap-store.snap-store
   snap.snap-store.ubuntu-software
   snap.snap-store.ubuntu-software-local-file
   snap.yq.yq
0 profiles are in complain mode.
10 processes have profiles defined.
10 processes are in enforce mode.
   /usr/sbin/chronyd (764) 
   /usr/sbin/chronyd (775) 
   /usr/sbin/cups-browsed (143816) 
   /usr/sbin/cupsd (143815) 
   /usr/local/bin/etcd (1632) cri-containerd.apparmor.d
   /coredns (2973) cri-containerd.apparmor.d
   /coredns (3079) cri-containerd.apparmor.d
   /usr/local/bin/kube-controller-manager (40866) cri-containerd.apparmor.d
   /usr/local/bin/kube-scheduler (40875) cri-containerd.apparmor.d
   /usr/local/bin/kube-apiserver (41305) cri-containerd.apparmor.d
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.
```

## 02. 프로필 추가

### 01. 명령어로 프로필 추가

<aside>
💡 **# aa-genprof {명령어}**

</aside>

curl 명령어에 대한 프로필을 추가해보자. 일단 프로필 추가 전에 curl 명령어가 잘 수행되는지 확인해본다.

```json
**# curl google.com -v**
*   Trying 142.250.207.110:80...
* TCP_NODELAY set
* Connected to google.com (142.250.207.110) port 80 (#0)
> GET / HTTP/1.1
> Host: google.com
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 301 Moved Permanently
< Location: http://www.google.com/
< Content-Type: text/html; charset=UTF-8
< Date: Fri, 28 Oct 2022 02:15:22 GMT
< Expires: Sun, 27 Nov 2022 02:15:22 GMT
< Cache-Control: public, max-age=2592000
< Server: gws
< Content-Length: 219
< X-XSS-Protection: 0
< X-Frame-Options: SAMEORIGIN
< 
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
* Connection #0 to host google.com left intact
```

아주 잘 된다. 이제 프로필을 생성해보자.

```json
# aa-genprof curl
Writing updated profile for /usr/bin/curl.
Setting /usr/bin/curl to complain mode.

Before you begin, you may wish to check if a
profile already exists for the application you
wish to confine. See the following wiki page for
more information:
https://gitlab.com/apparmor/apparmor/wikis/Profiles

Profiling: /usr/bin/curl

Please start the application to be profiled in
another window and exercise its functionality now.

Once completed, select the "Scan" option below in 
order to scan the system logs for AppArmor events. 

For each AppArmor event, you will be given the 
opportunity to choose whether the access should be 
allowed or denied.

[(S)can system log for AppArmor events] / (F)inish    **## F 버튼**
Setting /usr/bin/curl to enforce mode.

Reloaded AppArmor profiles in enforce mode.

Please consider contributing your new profile!
See the following wiki page for more information:
https://gitlab.com/apparmor/apparmor/wikis/Profiles

Finished generating profile for /usr/bin/curl.
```

프로필 목록에서 새 프로필 정보를 확인한다.

```json
**# aa-status | grep curl**
   /usr/bin/curl
```

curl 명령어를 수행해본다.

```json
**# curl google.com -v**
* Could not resolve host: google.com
* Closing connection 0
curl: (6) Could not resolve host: google.com
```

막혔다.

### 02. 커스텀 프로필 추가

아래는 CKS udemy 수업 듣다가 주워온 프로필이다. 서버에 저장하자. 프로필명이 docker-nginx라고 적혀있지만, CRI 런타임에 구애받지 않고 적용할 수 있도록 설정됐기 때문에 contaeinrd나 cri-o에서도 사용할 수 있다.

- **프로필 파일**
    
    ```c
    #include <tunables/global>
    
    profile docker-nginx flags=(attach_disconnected,mediate_deleted) {
      #include <abstractions/base>
    
      network inet tcp,
      network inet udp,
      network inet icmp,
    
      deny network raw,
    
      deny network packet,
    
      file,
      umount,
    
      deny /bin/** wl,
      deny /boot/** wl,
      deny /dev/** wl,
      deny /etc/** wl,
      deny /home/** wl,
      deny /lib/** wl,
      deny /lib64/** wl,
      deny /media/** wl,
      deny /mnt/** wl,
      deny /opt/** wl,
      deny /proc/** wl,
      deny /root/** wl,
      deny /sbin/** wl,
      deny /srv/** wl,
      deny /tmp/** wl,
      deny /sys/** wl,
      deny /usr/** wl,
    
      audit /** w,
    
      /var/run/nginx.pid w,
    
      /usr/sbin/nginx ix,
    
      deny /bin/dash mrwklx,
      deny /bin/sh mrwklx,
      deny /usr/bin/top mrwklx,
    
      capability chown,
      capability dac_override,
      capability setuid,
      capability setgid,
      capability net_bind_service,
    
      deny @{PROC}/* w,   # deny write for all files directly in /proc (not in a subdir)
      # deny write to files not in /proc/<number>/** or /proc/sys/**
      deny @{PROC}/{[^1-9],[^1-9][^0-9],[^1-9s][^0-9y][^0-9s],[^1-9][^0-9][^0-9][^0-9]*}/** w,
      deny @{PROC}/sys/[^k]** w,  # deny /proc/sys except /proc/sys/k* (effectively /proc/sys/kernel)
      deny @{PROC}/sys/kernel/{?,??,[^s][^h][^m]**} w,  # deny everything except shm* in /proc/sys/kernel/
      deny @{PROC}/sysrq-trigger rwklx,
      deny @{PROC}/mem rwklx,
      deny @{PROC}/kmem rwklx,
      deny @{PROC}/kcore rwklx,
    
      deny mount,
    
      deny /sys/[^f]*/** wklx,
      deny /sys/f[^s]*/** wklx,
      deny /sys/fs/[^c]*/** wklx,
      deny /sys/fs/c[^g]*/** wklx,
      deny /sys/fs/cg[^r]*/** wklx,
      deny /sys/firmware/** rwklx,
      deny /sys/kernel/security/** rwklx,
    }
    ```
    
    대충 nginx 컨테이너 파일이 시스템의 주요 경로에 쓰기나 조회를 할 수 없도록 막아놓는 정책이라 보면 된다.
    
- **프로필 파일 app 프로필 관리 디렉터리에 저장**
    
    ```c
    **# mv docker-nginx /etc/apparmor.d/**
    
    **# ls /etc/apparmor.d/**
    abstractions    lsb_release      usr.bin.evince                   usr.sbin.cups-browsed
    disable         nvidia_modprobe  usr.bin.firefox                  usr.sbin.cupsd
    **docker-nginx**    sbin.dhclient    usr.bin.man                      usr.sbin.ippusbxd
    force-complain  tunables         usr.lib.snapd.snap-confine.real  usr.sbin.rsyslogd
    local           usr.bin.curl     usr.sbin.chronyd                 usr.sbin.tcpdump
    ```
    
- **프로필 등록**
    
    <aside>
    💡 **# apparmor_parser {프로필 파일 경로}**
    
    </aside>
    
    ```json
    **# apparmor_parser /etc/apparmor.d/docker-nginx**
    
    **# aa-status | grep docker-nginx**
       docker-nginx
    ```
    

# 04. Kubernetes에 AppArmor 적용

## 01. AppArmor 사용 조건

- 커널에 AppArmor 모듈이 활성화되어야 한다.
- 호스트 노드에 AppArmor 프로필이 적용되어 있어야 한다.
- 쿠버네티스는 1.4 버전 이상이어야 한다.
- 컨테이너 런타임이 AppArmor를 지원해야 한다.

## 02. 파드에 AppArmor 프로필 적용

별다른 파드나 리소스를 생성할 필요 없이 annotation을 추가하면 된다.

<aside>
💡 container.apparmor.security.beta.kubernetes.io/{컨테이너명}: localhost/{프로필 명}

</aside>

아래 pod는 apparmor를 적용한 pod의 예제다. 프로필은 [02. 커스텀 프로필 추가](https://www.notion.so/02-5a017896665948548ae623e88b12c905?pvs=21) 에서 만든 프로필을 사용한다. 해당 프로필을 사용하면 컨테이너 내부의 중요 경로에 파일 생성이나 조회 수정이 불가능해진다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    **container.apparmor.security.beta.kubernetes.io/nginx: localhost/docker-nginx**
  labels:
    run: nginx-aa
  name: nginx-aa
spec:
  **nodeName: k8s-worker2   # 강제로 profile 적용된 노드에 스케쥴링 하기 위해 추가**
  containers:
  - image: nginx
    name: nginx
```

생성된 파드에 접근해보자.

```json
**# kubectl exec -it nginx-aa -- bash**
root@nginx-aa:/# touch /etc/aaa
**touch: cannot touch '/etc/aaa': Permission denied**
root@nginx-aa:/# echo "aaa" >> /etc/resolv.conf 
**bash: /etc/resolv.conf: Permission denied**
root@nginx-aa:/#
```

쓰기나 수정이 불가능 한것을 확인할 수 있다.
