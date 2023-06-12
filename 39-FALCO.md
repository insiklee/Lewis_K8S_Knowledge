# FALCO

[The Falco Project](https://falco.org/docs/)

본 문서는 CKS 준비하다가 정리하게 됐다.

실무에서 FALCO보다 더 좋은, 돈만 주면 기술지원까지 해주는 그런 솔루션 쓰지 않을까…? 란 생각에 그냥 대충 정리한다.

# 01. FALCO란?

## 01. 개요

Sysdig에서 개발하고 CNCF가 기여한 프로젝트로 런타임 보안 오픈소스 툴이다.

커널에서 발생하는 시스템 콜을 파싱하여 규칙을 위반한 사항이 발생할 경우 즉시 경고를 할 수 있다.

## 02. FALCO에서 체크하는 내용들

- privileged 컨테이너의 권한상승
- setns 툴을 이용한 커널 네임스페이스 변경
- 잘 알려진 경로(/etc, /usr/bin)의 읽기쓰기
- 심볼릭 링크 생성
- 파일의 소유권 및 권한 변경
- 예상치 못한 네트워크 연결이나 소켓 변조
- execve를 사용하여 생성된 프로세스
- bash, sh 같은 쉘 명령어 사용
- ssh, scp 같은 ssh 명령어 사용
- 리눅스 coreutils (ls, mkdir, cd 등)의 변조
- 로그인 실행파일의 변조
- shadowutil 이나 passwd 같은 계정 관련 명령어의 변조

## 03. FALCO의 룰 목록

너무 방대하기 때문에 일일히 정리하는 것 보단 공식 홈페이지에서 확인하는 것이 좋을 것 같다.

[Rules](https://falco.org/docs/rules/)

# 02. FALCO 설치

## 01. 데비안 계열

```json
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | tee -a /etc/apt/sources.list.d/falcosecurity.list
apt-get update -y
apt-get -y install linux-headers-$(uname -r)
apt-get install -y falco
```

## 02. 페도라 계열

```json
rpm --import https://falco.org/repo/falcosecurity-3672BA8F.asc
curl -s -o /etc/yum.repos.d/falcosecurity.repo https://falco.org/repo/falcosecurity-rpm.repo
yum -y install kernel-devel-$(uname -r)
yum -y install falco
```

## 03. 쿠버네티스에 설치

# 03. FALCO 실습

## 01. FALCO 설정 파일 확인

FALCO를 처음 설치하면 /etc/falco 아래에 FALCO의 rule, audit, config 파일이 위치한다.

```json
**# ls -l /etc/falco**
total 216
-rw-r--r-- 1 root root  12314 10월 14 20:17 aws_cloudtrail_rules.yaml
-rw-r--r-- 1 root root   1136 10월 19 22:31 falco_rules.local.yaml
-rw-r--r-- 1 root root 140728 10월 19 22:31 falco_rules.yaml
-rw-r--r-- 1 root root  15594 10월 19 22:31 falco.yaml
-rw-r--r-- 1 root root  30940 10월 14 20:18 k8s_audit_rules.yaml
drwxr-xr-x 2 root root   4096 10월 27 09:44 rules.available
drwxr-xr-x 2 root root   4096 10월 19 22:58 rules.d
```

- falco.yaml : 설정파일이다.
- falco_rules.yaml : falco가 미리 세팅해준 아주 잘 짜여지고 강력한 룰셋이다.
- falco_rules.local.yaml : 사용자가 커스텀해서 쓸 수 있도록 제공된 파일이다.
- k8s_audit_rules.yaml : k8s의 audit과 연계해서 쓰는 룰인데, k8saudit 플러그인을 설치해야 한다. 귀찮으니 생략한다.

## 02. rule 사용하기

### 01. 설정파일에서 rule 파일 읽기

falco.yaml에서 rules_file에 추가한다.

```json
**# vi /etc/falco/falco.yaml**
------------------/etc/falco/falco.yaml------------------------
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/rules.d
(하략)
---------------------------------------------------------------
```

### 02. rule 설정하기

룰 설정하는 방법은 좀 복잡하다. CKS 

## 03. falco 실행

실행파일로 실행하는 방법도 있고, systemd로 실행하는 방법도 있다.

실행파일로 실행할 경우 로그를 실시간으로 확인할 수 있다.

```json
**# falco**
Thu Oct 27 18:24:01 2022: Falco version: 0.33.0 (x86_64)
Thu Oct 27 18:24:01 2022: Falco initialized with configuration file: /etc/falco/falco.yaml
Thu Oct 27 18:24:01 2022: Loading rules from file /etc/falco/falco_rules.yaml
Thu Oct 27 18:24:01 2022: Loading rules from file /etc/falco/falco_rules.local.yaml
Thu Oct 27 18:24:01 2022: The chosen syscall buffer dimension is: 8388608 bytes (8 MBs)
Thu Oct 27 18:24:01 2022: Starting health webserver with threadiness 4, listening on port 8765
Thu Oct 27 18:24:01 2022: Enabled event sources: syscall
Thu Oct 27 18:24:01 2022: Opening capture with Kernel module
**## 파드 생성
18:24:53.670696605: Warning Log files were tampered (user=root user_loginuid=-1 command=containerd pid=738 file=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/858/fs/var/log/dpkg.log container_id=host image=<NA>)**

```
