# Seccomp

[Restrict a Container's Syscalls with seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)

# 01. Seccomp란?

Secure Computing의 약자로 프로세스가 호출하는 시스템 콜을 제어한다.

프로세스를 샌드박스로 실행한 뒤, 유저 영역에서 커널로 전달하는 콜을 제한하는 방식이다. 

# 02. 쿠버네티스에 seccomp 적용

## 01. 프로필 파일 저장

### 01. 프로필 내용

CKS 강의듣다가 주워왔다.

- ************************default.json************************
    
    ```json
    {
      "defaultAction": "SCMP_ACT_ERRNO",
      "archMap": [
        {
          "architecture": "SCMP_ARCH_X86_64",
          "subArchitectures": [
            "SCMP_ARCH_X86",
            "SCMP_ARCH_X32"
          ]
        },
        {
          "architecture": "SCMP_ARCH_AARCH64",
          "subArchitectures": [
            "SCMP_ARCH_ARM"
          ]
        },
        {
          "architecture": "SCMP_ARCH_MIPS64",
          "subArchitectures": [
            "SCMP_ARCH_MIPS",
            "SCMP_ARCH_MIPS64N32"
          ]
        },
        {
          "architecture": "SCMP_ARCH_MIPS64N32",
          "subArchitectures": [
            "SCMP_ARCH_MIPS",
            "SCMP_ARCH_MIPS64"
          ]
        },
        {
          "architecture": "SCMP_ARCH_MIPSEL64",
          "subArchitectures": [
            "SCMP_ARCH_MIPSEL",
            "SCMP_ARCH_MIPSEL64N32"
          ]
        },
        {
          "architecture": "SCMP_ARCH_MIPSEL64N32",
          "subArchitectures": [
            "SCMP_ARCH_MIPSEL",
            "SCMP_ARCH_MIPSEL64"
          ]
        },
        {
          "architecture": "SCMP_ARCH_S390X",
          "subArchitectures": [
            "SCMP_ARCH_S390"
          ]
        }
      ],
      "syscalls": [
        {
          "names": [
            "accept",
            "accept4",
            "access",
            "adjtimex",
            "alarm",
            "bind",
            "brk",
            "capget",
            "capset",
            "chdir",
            "chmod",
            "chown",
            "chown32",
            "clock_adjtime",
            "clock_adjtime64",
            "clock_getres",
            "clock_getres_time64",
            "clock_gettime",
            "clock_gettime64",
            "clock_nanosleep",
            "clock_nanosleep_time64",
            "close",
            "connect",
            "copy_file_range",
            "creat",
            "dup",
            "dup2",
            "dup3",
            "epoll_create",
            "epoll_create1",
            "epoll_ctl",
            "epoll_ctl_old",
            "epoll_pwait",
            "epoll_wait",
            "epoll_wait_old",
            "eventfd",
            "eventfd2",
            "execve",
            "execveat",
            "exit",
            "exit_group",
            "faccessat",
            "faccessat2",
            "fadvise64",
            "fadvise64_64",
            "fallocate",
            "fanotify_mark",
            "fchdir",
            "fchmod",
            "fchmodat",
            "fchown",
            "fchown32",
            "fchownat",
            "fcntl",
            "fcntl64",
            "fdatasync",
            "fgetxattr",
            "flistxattr",
            "flock",
            "fork",
            "fremovexattr",
            "fsetxattr",
            "fstat",
            "fstat64",
            "fstatat64",
            "fstatfs",
            "fstatfs64",
            "fsync",
            "ftruncate",
            "ftruncate64",
            "futex",
            "futex_time64",
            "futimesat",
            "getcpu",
            "getcwd",
            "getdents",
            "getdents64",
            "getegid",
            "getegid32",
            "geteuid",
            "geteuid32",
            "getgid",
            "getgid32",
            "getgroups",
            "getgroups32",
            "getitimer",
            "getpeername",
            "getpgid",
            "getpgrp",
            "getpid",
            "getppid",
            "getpriority",
            "getrandom",
            "getresgid",
            "getresgid32",
            "getresuid",
            "getresuid32",
            "getrlimit",
            "get_robust_list",
            "getrusage",
            "getsid",
            "getsockname",
            "getsockopt",
            "get_thread_area",
            "gettid",
            "gettimeofday",
            "getuid",
            "getuid32",
            "getxattr",
            "inotify_add_watch",
            "inotify_init",
            "inotify_init1",
            "inotify_rm_watch",
            "io_cancel",
            "ioctl",
            "io_destroy",
            "io_getevents",
            "io_pgetevents",
            "io_pgetevents_time64",
            "ioprio_get",
            "ioprio_set",
            "io_setup",
            "io_submit",
            "io_uring_enter",
            "io_uring_register",
            "io_uring_setup",
            "ipc",
            "kill",
            "lchown",
            "lchown32",
            "lgetxattr",
            "link",
            "linkat",
            "listen",
            "listxattr",
            "llistxattr",
            "_llseek",
            "lremovexattr",
            "lseek",
            "lsetxattr",
            "lstat",
            "lstat64",
            "madvise",
            "membarrier",
            "memfd_create",
            "mincore",
            "mkdir",
            "mkdirat",
            "mknod",
            "mknodat",
            "mlock",
            "mlock2",
            "mlockall",
            "mmap",
            "mmap2",
            "mprotect",
            "mq_getsetattr",
            "mq_notify",
            "mq_open",
            "mq_timedreceive",
            "mq_timedreceive_time64",
            "mq_timedsend",
            "mq_timedsend_time64",
            "mq_unlink",
            "mremap",
            "msgctl",
            "msgget",
            "msgrcv",
            "msgsnd",
            "msync",
            "munlock",
            "munlockall",
            "munmap",
            "nanosleep",
            "newfstatat",
            "_newselect",
            "open",
            "openat",
            "openat2",
            "pause",
            "pipe",
            "pipe2",
            "poll",
            "ppoll",
            "ppoll_time64",
            "prctl",
            "pread64",
            "preadv",
            "preadv2",
            "prlimit64",
            "pselect6",
            "pselect6_time64",
            "pwrite64",
            "pwritev",
            "pwritev2",
            "read",
            "readahead",
            "readlink",
            "readlinkat",
            "readv",
            "recv",
            "recvfrom",
            "recvmmsg",
            "recvmmsg_time64",
            "recvmsg",
            "remap_file_pages",
            "removexattr",
            "rename",
            "renameat",
            "renameat2",
            "restart_syscall",
            "rmdir",
            "rseq",
            "rt_sigaction",
            "rt_sigpending",
            "rt_sigprocmask",
            "rt_sigqueueinfo",
            "rt_sigreturn",
            "rt_sigsuspend",
            "rt_sigtimedwait",
            "rt_sigtimedwait_time64",
            "rt_tgsigqueueinfo",
            "sched_getaffinity",
            "sched_getattr",
            "sched_getparam",
            "sched_get_priority_max",
            "sched_get_priority_min",
            "sched_getscheduler",
            "sched_rr_get_interval",
            "sched_rr_get_interval_time64",
            "sched_setaffinity",
            "sched_setattr",
            "sched_setparam",
            "sched_setscheduler",
            "sched_yield",
            "seccomp",
            "select",
            "semctl",
            "semget",
            "semop",
            "semtimedop",
            "semtimedop_time64",
            "send",
            "sendfile",
            "sendfile64",
            "sendmmsg",
            "sendmsg",
            "sendto",
            "setfsgid",
            "setfsgid32",
            "setfsuid",
            "setfsuid32",
            "setgid",
            "setgid32",
            "setgroups",
            "setgroups32",
            "setitimer",
            "setpgid",
            "setpriority",
            "setregid",
            "setregid32",
            "setresgid",
            "setresgid32",
            "setresuid",
            "setresuid32",
            "setreuid",
            "setreuid32",
            "setrlimit",
            "set_robust_list",
            "setsid",
            "setsockopt",
            "set_thread_area",
            "set_tid_address",
            "setuid",
            "setuid32",
            "setxattr",
            "shmat",
            "shmctl",
            "shmdt",
            "shmget",
            "shutdown",
            "sigaltstack",
            "signalfd",
            "signalfd4",
            "sigprocmask",
            "sigreturn",
            "socket",
            "socketcall",
            "socketpair",
            "splice",
            "stat",
            "stat64",
            "statfs",
            "statfs64",
            "statx",
            "symlink",
            "symlinkat",
            "sync",
            "sync_file_range",
            "syncfs",
            "sysinfo",
            "tee",
            "tgkill",
            "time",
            "timer_create",
            "timer_delete",
            "timer_getoverrun",
            "timer_gettime",
            "timer_gettime64",
            "timer_settime",
            "timer_settime64",
            "timerfd_create",
            "timerfd_gettime",
            "timerfd_gettime64",
            "timerfd_settime",
            "timerfd_settime64",
            "times",
            "tkill",
            "truncate",
            "truncate64",
            "ugetrlimit",
            "umask",
            "uname",
            "unlink",
            "unlinkat",
            "utime",
            "utimensat",
            "utimensat_time64",
            "utimes",
            "vfork",
            "vmsplice",
            "wait4",
            "waitid",
            "waitpid",
            "write",
            "writev"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {},
          "excludes": {}
        },
        {
          "names": [
            "ptrace"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": null,
          "comment": "",
          "includes": {
            "minKernel": "4.8"
          },
          "excludes": {}
        },
        {
          "names": [
            "personality"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [
            {
              "index": 0,
              "value": 0,
              "op": "SCMP_CMP_EQ"
            }
          ],
          "comment": "",
          "includes": {},
          "excludes": {}
        },
        {
          "names": [
            "personality"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [
            {
              "index": 0,
              "value": 8,
              "op": "SCMP_CMP_EQ"
            }
          ],
          "comment": "",
          "includes": {},
          "excludes": {}
        },
        {
          "names": [
            "personality"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [
            {
              "index": 0,
              "value": 131072,
              "op": "SCMP_CMP_EQ"
            }
          ],
          "comment": "",
          "includes": {},
          "excludes": {}
        },
        {
          "names": [
            "personality"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [
            {
              "index": 0,
              "value": 131080,
              "op": "SCMP_CMP_EQ"
            }
          ],
          "comment": "",
          "includes": {},
          "excludes": {}
        },
        {
          "names": [
            "personality"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [
            {
              "index": 0,
              "value": 4294967295,
              "op": "SCMP_CMP_EQ"
            }
          ],
          "comment": "",
          "includes": {},
          "excludes": {}
        },
        {
          "names": [
            "sync_file_range2"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "arches": [
              "ppc64le"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "arm_fadvise64_64",
            "arm_sync_file_range",
            "sync_file_range2",
            "breakpoint",
            "cacheflush",
            "set_tls"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "arches": [
              "arm",
              "arm64"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "arch_prctl"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "arches": [
              "amd64",
              "x32"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "modify_ldt"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "arches": [
              "amd64",
              "x32",
              "x86"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "s390_pci_mmio_read",
            "s390_pci_mmio_write",
            "s390_runtime_instr"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "arches": [
              "s390",
              "s390x"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "open_by_handle_at"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_DAC_READ_SEARCH"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "bpf",
            "clone",
            "fanotify_init",
            "lookup_dcookie",
            "mount",
            "name_to_handle_at",
            "perf_event_open",
            "quotactl",
            "setdomainname",
            "sethostname",
            "setns",
            "syslog",
            "umount",
            "umount2",
            "unshare"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_ADMIN"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "clone"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [
            {
              "index": 0,
              "value": 2114060288,
              "op": "SCMP_CMP_MASKED_EQ"
            }
          ],
          "comment": "",
          "includes": {},
          "excludes": {
            "caps": [
              "CAP_SYS_ADMIN"
            ],
            "arches": [
              "s390",
              "s390x"
            ]
          }
        },
        {
          "names": [
            "clone"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [
            {
              "index": 1,
              "value": 2114060288,
              "op": "SCMP_CMP_MASKED_EQ"
            }
          ],
          "comment": "s390 parameter ordering for clone is different",
          "includes": {
            "arches": [
              "s390",
              "s390x"
            ]
          },
          "excludes": {
            "caps": [
              "CAP_SYS_ADMIN"
            ]
          }
        },
        {
          "names": [
            "reboot"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_BOOT"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "chroot"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_CHROOT"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "delete_module",
            "init_module",
            "finit_module"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_MODULE"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "acct"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_PACCT"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "kcmp",
            "process_vm_readv",
            "process_vm_writev",
            "ptrace"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_PTRACE"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "iopl",
            "ioperm"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_RAWIO"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "settimeofday",
            "stime",
            "clock_settime"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_TIME"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "vhangup"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_TTY_CONFIG"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "get_mempolicy",
            "mbind",
            "set_mempolicy"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYS_NICE"
            ]
          },
          "excludes": {}
        },
        {
          "names": [
            "syslog"
          ],
          "action": "SCMP_ACT_ALLOW",
          "args": [],
          "comment": "",
          "includes": {
            "caps": [
              "CAP_SYSLOG"
            ]
          },
          "excludes": {}
        }
      ]
    }
    ```
    

### 02. 프로필 파일 저장

프로필 파일을 저장하자. 

프로필은 /var/lib/kubelet/seccomp에 저장한다. 이러면 따로 프로필 파일을 컨테이너에 마운트 안해도 kubelet이 알아서 찾아간다.

```json
**# mkdir /var/lib/kubelet/seccomp

# mv default.json /var/lib/kubelet/seccomp**
```

### 03. 파드에 SecurityContext로 seccomp 적용

파드를 하나 만들자

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-seccomp
  name: nginx-seccomp
spec:
  nodeName: k8s-worker2
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: default.json
  containers:
  - image: nginx
    name: nginx
```

```json
**# kubectl apply -f nginx-seccomp.yaml** 
pod/nginx-seccomp created
```

파드를 생성하면 seccomp로 시작하는 annotation이 생성된 것을 볼 수 있다.

```json
# kubectl get pod nginx-seccomp -o jsonpath='{.metadata.annotations}' | jq
{
  "cni.projectcalico.org/containerID": "5286282126fb8a41e81d6b9401964a44ada72d79f0aa14ed56e49f3e1492364f",
  "cni.projectcalico.org/podIP": "10.110.126.16/32",
  "cni.projectcalico.org/podIPs": "10.110.126.16/32",
  "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"run\":\"nginx-seccomp\"},\"name\":\"nginx-seccomp\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"image\":\"nginx\",\"name\":\"nginx\"}],\"nodeName\":\"k8s-worker2\",\"securityContext\":{\"seccompProfile\":{\"localhostProfile\":\"default.json\",\"type\":\"Localhost\"}}}}\n",
  **"seccomp.security.alpha.kubernetes.io/pod": "localhost/default.json"**
}
```

파드가 실행된 worker2에서 crictl inspect로 실행된 컨테이너 정보를 확인해보자

```json
**# crictl inspect $(crictl ps | grep nginx-seccomp | awk '{print $1}') | jq .info.runtimeSpec.linux.seccomp**
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "accept",
        "accept4",
        "access",
        "adjtimex",
        "alarm",
        "bind",
        "brk",
        "capget",
        "capset",
        "chdir",
        "chmod",
        "chown",
        "chown32",
        "clock_adjtime",
        "clock_adjtime64",
        "clock_getres",
        "clock_getres_time64",
        "clock_gettime",
        "clock_gettime64",
        "clock_nanosleep",
        "clock_nanosleep_time64",
        "close",
        "connect",
        "copy_file_range",
        "creat",
        "dup",
        "dup2",
        "dup3",
        "epoll_create",
        "epoll_create1",
        "epoll_ctl",
        "epoll_ctl_old",
        "epoll_pwait",
        "epoll_wait",
        "epoll_wait_old",
        "eventfd",
        "eventfd2",
        "execve",
        "execveat",
        "exit",
        "exit_group",
        "faccessat",
        "faccessat2",
        "fadvise64",
        "fadvise64_64",
        "fallocate",
        "fanotify_mark",
        "fchdir",
        "fchmod",
        "fchmodat",
        "fchown",
        "fchown32",
        "fchownat",
        "fcntl",
        "fcntl64",
        "fdatasync",
        "fgetxattr",
        "flistxattr",
        "flock",
        "fork",
        "fremovexattr",
        "fsetxattr",
        "fstat",
        "fstat64",
        "fstatat64",
        "fstatfs",
        "fstatfs64",
        "fsync",
        "ftruncate",
        "ftruncate64",
        "futex",
        "futex_time64",
        "futimesat",
        "getcpu",
        "getcwd",
        "getdents",
        "getdents64",
        "getegid",
        "getegid32",
        "geteuid",
        "geteuid32",
        "getgid",
        "getgid32",
        "getgroups",
        "getgroups32",
        "getitimer",
        "getpeername",
        "getpgid",
        "getpgrp",
        "getpid",
        "getppid",
        "getpriority",
        "getrandom",
        "getresgid",
        "getresgid32",
        "getresuid",
        "getresuid32",
        "getrlimit",
        "get_robust_list",
        "getrusage",
        "getsid",
        "getsockname",
        "getsockopt",
        "get_thread_area",
        "gettid",
        "gettimeofday",
        "getuid",
        "getuid32",
        "getxattr",
        "inotify_add_watch",
        "inotify_init",
        "inotify_init1",
        "inotify_rm_watch",
        "io_cancel",
        "ioctl",
        "io_destroy",
        "io_getevents",
        "io_pgetevents",
        "io_pgetevents_time64",
        "ioprio_get",
        "ioprio_set",
        "io_setup",
        "io_submit",
        "io_uring_enter",
        "io_uring_register",
        "io_uring_setup",
        "ipc",
        "kill",
        "lchown",
        "lchown32",
        "lgetxattr",
        "link",
        "linkat",
        "listen",
        "listxattr",
        "llistxattr",
        "_llseek",
        "lremovexattr",
        "lseek",
        "lsetxattr",
        "lstat",
        "lstat64",
        "madvise",
        "membarrier",
        "memfd_create",
        "mincore",
        "mkdir",
        "mkdirat",
        "mknod",
        "mknodat",
        "mlock",
        "mlock2",
        "mlockall",
        "mmap",
        "mmap2",
        "mprotect",
        "mq_getsetattr",
        "mq_notify",
        "mq_open",
        "mq_timedreceive",
        "mq_timedreceive_time64",
        "mq_timedsend",
        "mq_timedsend_time64",
        "mq_unlink",
        "mremap",
        "msgctl",
        "msgget",
        "msgrcv",
        "msgsnd",
        "msync",
        "munlock",
        "munlockall",
        "munmap",
        "nanosleep",
        "newfstatat",
        "_newselect",
        "open",
        "openat",
        "openat2",
        "pause",
        "pipe",
        "pipe2",
        "poll",
        "ppoll",
        "ppoll_time64",
        "prctl",
        "pread64",
        "preadv",
        "preadv2",
        "prlimit64",
        "pselect6",
        "pselect6_time64",
        "pwrite64",
        "pwritev",
        "pwritev2",
        "read",
        "readahead",
        "readlink",
        "readlinkat",
        "readv",
        "recv",
        "recvfrom",
        "recvmmsg",
        "recvmmsg_time64",
        "recvmsg",
        "remap_file_pages",
        "removexattr",
        "rename",
        "renameat",
        "renameat2",
        "restart_syscall",
        "rmdir",
        "rseq",
        "rt_sigaction",
        "rt_sigpending",
        "rt_sigprocmask",
        "rt_sigqueueinfo",
        "rt_sigreturn",
        "rt_sigsuspend",
        "rt_sigtimedwait",
        "rt_sigtimedwait_time64",
        "rt_tgsigqueueinfo",
        "sched_getaffinity",
        "sched_getattr",
        "sched_getparam",
        "sched_get_priority_max",
        "sched_get_priority_min",
        "sched_getscheduler",
        "sched_rr_get_interval",
        "sched_rr_get_interval_time64",
        "sched_setaffinity",
        "sched_setattr",
        "sched_setparam",
        "sched_setscheduler",
        "sched_yield",
        "seccomp",
        "select",
        "semctl",
        "semget",
        "semop",
        "semtimedop",
        "semtimedop_time64",
        "send",
        "sendfile",
        "sendfile64",
        "sendmmsg",
        "sendmsg",
        "sendto",
        "setfsgid",
        "setfsgid32",
        "setfsuid",
        "setfsuid32",
        "setgid",
        "setgid32",
        "setgroups",
        "setgroups32",
        "setitimer",
        "setpgid",
        "setpriority",
        "setregid",
        "setregid32",
        "setresgid",
        "setresgid32",
        "setresuid",
        "setresuid32",
        "setreuid",
        "setreuid32",
        "setrlimit",
        "set_robust_list",
        "setsid",
        "setsockopt",
        "set_thread_area",
        "set_tid_address",
        "setuid",
        "setuid32",
        "setxattr",
        "shmat",
        "shmctl",
        "shmdt",
        "shmget",
        "shutdown",
        "sigaltstack",
        "signalfd",
        "signalfd4",
        "sigprocmask",
        "sigreturn",
        "socket",
        "socketcall",
        "socketpair",
        "splice",
        "stat",
        "stat64",
        "statfs",
        "statfs64",
        "statx",
        "symlink",
        "symlinkat",
        "sync",
        "sync_file_range",
        "syncfs",
        "sysinfo",
        "tee",
        "tgkill",
        "time",
        "timer_create",
        "timer_delete",
        "timer_getoverrun",
        "timer_gettime",
        "timer_gettime64",
        "timer_settime",
        "timer_settime64",
        "timerfd_create",
        "timerfd_gettime",
        "timerfd_gettime64",
        "timerfd_settime",
        "timerfd_settime64",
        "times",
        "tkill",
        "truncate",
        "truncate64",
        "ugetrlimit",
        "umask",
        "uname",
        "unlink",
        "unlinkat",
        "utime",
        "utimensat",
        "utimensat_time64",
        "utimes",
        "vfork",
        "vmsplice",
        "wait4",
        "waitid",
        "waitpid",
        "write",
        "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "ptrace"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "personality"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 0,
          "op": "SCMP_CMP_EQ"
        }
      ]
    },
    {
      "names": [
        "personality"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 8,
          "op": "SCMP_CMP_EQ"
        }
      ]
    },
    {
      "names": [
        "personality"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 131072,
          "op": "SCMP_CMP_EQ"
        }
      ]
    },
    {
      "names": [
        "personality"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 131080,
          "op": "SCMP_CMP_EQ"
        }
      ]
    },
    {
      "names": [
        "personality"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 4294967295,
          "op": "SCMP_CMP_EQ"
        }
      ]
    },
    {
      "names": [
        "sync_file_range2"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "arm_fadvise64_64",
        "arm_sync_file_range",
        "sync_file_range2",
        "breakpoint",
        "cacheflush",
        "set_tls"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "arch_prctl"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "modify_ldt"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "s390_pci_mmio_read",
        "s390_pci_mmio_write",
        "s390_runtime_instr"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "open_by_handle_at"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "bpf",
        "clone",
        "fanotify_init",
        "lookup_dcookie",
        "mount",
        "name_to_handle_at",
        "perf_event_open",
        "quotactl",
        "setdomainname",
        "sethostname",
        "setns",
        "syslog",
        "umount",
        "umount2",
        "unshare"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "clone"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 2114060288,
          "op": "SCMP_CMP_MASKED_EQ"
        }
      ]
    },
    {
      "names": [
        "clone"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 1,
          "value": 2114060288,
          "op": "SCMP_CMP_MASKED_EQ"
        }
      ]
    },
    {
      "names": [
        "reboot"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "chroot"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "delete_module",
        "init_module",
        "finit_module"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "acct"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "kcmp",
        "process_vm_readv",
        "process_vm_writev",
        "ptrace"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "iopl",
        "ioperm"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "settimeofday",
        "stime",
        "clock_settime"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "vhangup"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "get_mempolicy",
        "mbind",
        "set_mempolicy"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": [
        "syslog"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

우리가 프로필을 저장했던 내용이 그대로 출력되는 것을 볼 수 있다.
