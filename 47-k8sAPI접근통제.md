# K8s API 접근통제

[Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)

# 01. API 접근통제 개요

!https://d33wubrfki0l68.cloudfront.net/673dbafd771491a080c02c6de3fdd41b09623c90/50100/images/docs/admin/access-control-overview.svg

kubectl을 사용하는 사람이나 Pod의 서비스어카운트는 API server에 접근할 때 여러가지 단계를 거치게 된다. 우선 API에 접근하기 위해서는 암호화된 TLS 접근만 허용되며, 접근 이후에는 위와 같 인증(authentication), 권한부여(authorization), 승인(Admission)의 과정을 거친다.

# 02. API 접근 단계

## 01. Tranport Security

### 01. TLS를 위한 API Server 설정

API 서버로의 접근은 오로지 암호화된 TLS 프로토콜을 통해서만 가능하다.  API 서버는 가동될때 이와 관련된 파라미터가 제공되어 있다.

```json
**# kubectl -n kube-system describe pod kube-apiserver-k8s-master**
(전략)
Command:
      kube-apiserver
      --advertise-address=192.168.122.10
      --allow-privileged=true
      --authorization-mode=Node,RBAC
      **--client-ca-file=/etc/kubernetes/pki/ca.crt**
      --enable-admission-plugins=NodeRestriction
      --enable-bootstrap-token-auth=true
      --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
      --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
      --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
      --etcd-servers=https://127.0.0.1:2379
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
      --requestheader-allowed-names=front-proxy-client
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
      --requestheader-extra-headers-prefix=X-Remote-Extra-
      --requestheader-group-headers=X-Remote-Group
      --requestheader-username-headers=X-Remote-User
      **--secure-port=6443**
      --service-account-issuer=https://kubernetes.default.svc.cluster.local
      --service-account-key-file=/etc/kubernetes/pki/sa.pub
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
      --service-cluster-ip-range=10.96.0.0/12
      **--tls-cert-file=/etc/kubernetes/pki/apiserver.crt
      --tls-private-key-file=/etc/kubernetes/pki/apiserver.key**
(하략)
```

위 내용은 API서버의 파드 상세 정보 중 컨테이너에 입력된 커맨드의 파라미터 정보다. 이 중 붉게 표시된 부분이 API가 TLS 통신을 위해 사용하는 정보들이다.

- --client-ca-file : API 서버에서 관리하는 CA 인증서다. [K8s RBAC - CertificateSigningRequests 관리](https://www.notion.so/K8s-RBAC-CertificateSigningRequests-b2b35cb83aa44e35a29189463d3220c4?pvs=21) 에서 유저에게 키를 생성해서 전달하는 역할을 하며, API 서버 자체를 인증하는 키 역시 생성한다.
- --tls-cert-file : API 서버의 인증서다. CA로부터 서명받았다.
- --tls-private-key : API 서버의 개인키다.
- --secure-port : HTTPS (TLS) 접속을 위해 열어놓은 포트 번호다.

### 02. TLS 협상 중 서버의 클라이언트 인증서 요청

API 서버는 클라이언트의 인증서를 TLS 통신 과정에서 가져온다. **클라이언트 인증서가 없을 경우 일단 TLS 협상 후, 암호화 통신은 가능하지만, 이후 Authentication 과정에서 Request가 거절된다.**

클라이언트 인증서와 관련된 내용은 [01. 인증서 인증](https://www.notion.so/01-396c2d0bfe0a46d380ffa81980aabc69?pvs=21) 을 참고하자.

- **클라이언트 인증서로 정상 TLS 협상**
    
    ```basic
    * Rebuilt URL to: https://192.168.122.10:6443/
    *   Trying 192.168.122.10...
    * TCP_NODELAY set
    * Connected to 192.168.122.10 (192.168.122.10) port 6443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * successfully set certificate verify locations:
    *   CAfile: ca
      CApath: none
    * TLSv1.3 (OUT), TLS handshake, Client hello (1):
    *** TLSv1.3 (IN), TLS handshake, Server hello (2):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Request CERT (13): ## 서버에서 클라이언트에 인증서를 요구한다.
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Certificate (11):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, CERT verify (15):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Finished (20):**
    * TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
    * TLSv1.3 (OUT), TLS handshake, [no content] (0):
    * TLSv1.3 (OUT), TLS handshake, Certificate (11):
    * TLSv1.3 (OUT), TLS handshake, [no content] (0):
    * TLSv1.3 (OUT), TLS handshake, CERT verify (15):
    * TLSv1.3 (OUT), TLS handshake, [no content] (0):
    * TLSv1.3 (OUT), TLS handshake, Finished (20):
    * SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
    * ALPN, server accepted to use h2
    * Server certificate:
    *  subject: CN=kube-apiserver
    *  start date: Oct 26 08:05:46 2022 GMT
    *  expire date: Oct 26 08:05:46 2023 GMT
    *  subjectAltName: host "192.168.122.10" matched cert's IP address!
    *  issuer: CN=kubernetes
    ***  SSL certificate verify ok. ## 클라이언트의 인증서 확인을 완료했다는 메시지다.**
    * Using HTTP2, server supports multi-use
    * Connection state changed (HTTP/2 confirmed)
    * Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * Using Stream ID: 1 (easy handle 0x55af54c114a0)
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    > GET / HTTP/2
    > Host: 192.168.122.10:6443
    > User-Agent: curl/7.61.1
    > Accept: */*
    > 
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
    * TLSv1.3 (IN), TLS app data, [no content] (0):
    * Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * TLSv1.3 (IN), TLS app data, [no content] (0):
    * TLSv1.3 (IN), TLS app data, [no content] (0):
    (이하, Response의 헤더와 바디 출력)
    ```
    
- **클라이언트 인증서 없이 비정상 TLS 협상**
    
    ```basic
    * Rebuilt URL to: https://192.168.122.10:6443/
    *   Trying 192.168.122.10...
    * TCP_NODELAY set
    * Connected to 192.168.122.10 (192.168.122.10) port 6443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * successfully set certificate verify locations:
    *   CAfile: /etc/pki/tls/certs/ca-bundle.crt
      CApath: none
    * TLSv1.3 (OUT), TLS handshake, Client hello (1):
    * TLSv1.3 (IN), TLS handshake, Server hello (2):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Request CERT (13):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Certificate (11):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, CERT verify (15):
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Finished (20):
    **## CERT verify가 누락된채 전달된다. 이 말은 크라이언트 측에서 인증서에 쓰일 공개키가 없다는 뜻이다.**
    *** TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
    * TLSv1.3 (OUT), TLS handshake, [no content] (0):
    * TLSv1.3 (OUT), TLS handshake, Certificate (11):
    * TLSv1.3 (OUT), TLS handshake, [no content] (0):
    * TLSv1.3 (OUT), TLS handshake, Finished (20):**
    * SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
    * ALPN, server accepted to use h2
    * Server certificate:
    *  subject: CN=kube-apiserver
    *  start date: Oct 26 08:05:46 2022 GMT
    *  expire date: Oct 26 08:05:46 2023 GMT
    *  issuer: CN=kubernetes
    ***  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway. ## 클라이언트의 인증서가 없지만 일단 협상을 진행한다.**
    * Using HTTP2, server supports multi-use
    * Connection state changed (HTTP/2 confirmed)
    * Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * Using Stream ID: 1 (easy handle 0x55c70d0594a0)
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    > GET / HTTP/2
    > Host: 192.168.122.10:6443
    > User-Agent: curl/7.61.1
    > Accept: */*
    > 
    * TLSv1.3 (IN), TLS handshake, [no content] (0):
    * TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
    * TLSv1.3 (IN), TLS app data, [no content] (0):
    * Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
    * TLSv1.3 (OUT), TLS app data, [no content] (0):
    * TLSv1.3 (IN), TLS app data, [no content] (0):
    * TLSv1.3 (IN), TLS app data, [no content] (0):
    < HTTP/2 403 
    < audit-id: c19ca88d-4cde-4617-8572-ba5befbb69c8
    < cache-control: no-cache, private
    < content-type: application/json
    < x-content-type-options: nosniff
    < x-kubernetes-pf-flowschema-uid: d35bb4da-7843-42d3-b847-30503c8081d3
    < x-kubernetes-pf-prioritylevel-uid: 447f5dd1-a338-4874-97ee-f57f10d41d05
    < content-length: 217
    < date: Tue, 01 Nov 2022 00:31:49 GMT
    < 
    * TLSv1.3 (IN), TLS app data, [no content] (0):
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {},
      "status": "Failure",
      **"message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
      "reason": "Forbidden", ## authentication에 의해 거절**
      "details": {},
      "code": 403
    * Connection #0 to host 192.168.122.10 left intact
    ```
    

## 02. Authentication (인증)

[Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

인증 과정은 TLS의 클라이언트 인증서, 평문으로 된 토큰, 부트스트랩 토큰, JSON Web 토큰 등을을 사용하고 진행된다. 여러개의 인증 모듈을 통해서 인증을 시도하며, 하나라도 인증에 성공할 경우 다음 단계로 진행된다.

만약 제대로된 인증을 받지 못할 경우 HTTP의 401 코드를 반환하며 접속이 차단된다.

아래는 비인증된 인증서로 접속을 시도할 때 나오는 메시지다.

```json
**# curl https://192.168.122.10:6443 --cacert /etc/kubernetes/pki/ca.crt --cert unauth.crt --key unauth.key** 
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  **"code": 401**
```

### 01. 인증서 인증

시스템 어카운트가 아닌 Human 유저들은 API 서버가 서명한 Client Certification을 통해 접속한다. Client Certification은 TLS 협상 과정에서 요청하며, 만약 인증서가 없을 경우 authetntication 과정에서 거절된다.

### **클라이언트 인증서 생성 후 API 서버로부터 인증**

일반 Client의 인증서 생성 방법은 [K8s RBAC - CertificateSigningRequests 관리](https://www.notion.so/K8s-RBAC-CertificateSigningRequests-b2b35cb83aa44e35a29189463d3220c4?pvs=21) 에 설명되어 있다.

다만 Cluster Admin의 경우 쿠버네티스를 구축할때 함께 생성되며, 이에 따른 인증서 역시 함께 만들어진다.

**Admin**의 인증서 정보는 아래 경로에 있다. 인증서를 파일로 보관하지 않고, Base64로 인코딩하여 conf 파일에 저장한다.

<aside>
💡 /etc/kubernetes/admin.conf

</aside>

이미 눈치챘을것이다.  위 파일은 쿠버네티스 초기 설치시 각 계정의 홈 디렉터리에 .kube/config 파일 명으로 복사하는 그 파일이 맞다. 

위 파일에서 인증서 관련 내용을 확인한다.

```json
**# cat /etc/kubernetes/admin.conf | egrep 'certi|key'**
    **certificate-authority-data**: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1Ea3lOakE0TlRrek5Gb1hEVE15TURreU16QTROVGt6TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTGkzCi9zY21mY3cwQWFVWGFVQlNZL1lDaDZIKy9GZC9zc01KMHNxdmpCZnJiSWE4UC9VMVMwVjZYcU50cTF5eHZIbVMKVkxqd1JWSFM2TmRyOVQ1Y3BZNVpJbkQxSXBFOVk0ekx4S0dpa0lHMlR5TDFzZ2l0RzRDRDN5VzJIeDBmMjgyQgpzajNTNWVEaWl0amF1Rkhmajl2VkFqTytFNW95TjJKNTQreENEdmlxSTRMMFkyWGllQnZkcnpuemJVV1RVM1RDCmZISTBoN0VFZmROQjQ4SkQyVFRHY29rQ3R5clg4ZUJ4d1A2bzdYdDNqaFhEd0FFaHo2Wk1ET290K0NSWmkyQW0KcU9JVnl1R0RzNUhFZ2JZQnFJRUhQSDAyclp0bHB3bFQ2dXFRcWk5dlVMNEJYeUR5QlNybDI4ZCt3bHBaNUROVwpUQkw3ZG1kVmx0cDRwa1VHLzZNQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZGNTI4VTJZbU1QSDNzYy9yU1BUNzQ2dDhDU0hNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBRHU5blRkQWlORWNCQWZFVThmYQpVYUEvMzlIY045SHlWWStCallSbVNSL0RTTUNLVEN6U0ZNYjhnLzBVSENGSTNmenR6Ylo1aGU2cWJPbFl2QlE3CmdLNkpOcTcrck8vUi9ZSk5MblRqVDJzNUwvRnJ3bzVWZkJOcnlJQjcyajl5eXhtZnJoczRsZUlDMEMxNzI3TloKQUhLMHpsWittUTNCUk5EYzFNYm1qSXF0THFqN0l6RmxKeDV2WDZkbDVFSGpqalVhQk93Z3dEeWRuU255Tmo0MQp1OFhhNlVnY2E5Y0tJcEVxTy9WWWFzdFRxc2JObm1DUWFuUFhjSzBkMjNya0NvM1JOMUFVaU1SNHdDUFNPNTVSClkzSXJKV2hDTlZMdjdlMVJqU3lRVUtFUGJ4ZlVDR3lWVnI0SEg3U1cxL2Y4VmV1NEdzUXBLL1huVzF4M2ZNWnUKeFJNPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    **client-certificate-data**: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJT3BkZ3RkWnVRTWN3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TWpBNU1qWXdPRFU1TXpSYUZ3MHlNekE1TWpZd09EVTVNemRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXRwczRMVjdnTFJ6Z3ZndDcKclJuQzhCSzZYWm93T0JFTkJSOWVVa01EU3Nrc1d3WURWcE56VDFIb0h0NnY1K1RISXZkUDNFVGp0ODMxQ1V2WgpWWCsvc3hZUThpNE1aa3pxSW84d3pUOUdyZE9pamNCbmlHWUZENS9QZC9seEVQeWVxbzB3ZUpvWmpXbmhyTE0wCmFxMzJ5TG9kRFBTQ0owQTBoNzEyQXc5SXdlYm42bWhOdkZ6STAzZ09QUExiZUxBbkdlT2lackorU2hSSUExSFEKamtUWVdqZ2d3ejVFbFYxOEpibTdYdWltNHZ6MHk0VXl0SXpGVkp0QWNSdXdONlhpL296aSt4U2lXSFFDVjRxaQowTzdpMC9VTXVpTGtpVGtPUjN6cjduYjhnTVhtQXFtZlZta3Bwc2R1d05aeFVtY2VpeWlSNjBhVEM0Z2JQRktWCit4UlNMd0lEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JSZWR2Rk5tSmpEeDk3SFA2MGowKytPcmZBawpoekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBVGFZdXdCYXNVVFZsYkNDb1phWW5WeTJsMTB1VUgwemh2MzVPCm0rSzVEZVBBdTRCbjRNeUFTc2hTcUl5NmZxVmg5QUdTNmIwT3h0S1drYWVLa0w0bG9MdlJzYnJaeU9jY3ovRGIKSDV0Tkc2TW9aMHdtYjFucmZMcmRyRXliVjgxNWl1ajlxYlJ1SmE3ckVhcHJtRlJLNW1acUVyTXRRR0JORGk5ZgpGa0F1V0ZvN3psc1ptU3pONWlRYmdJaFZzZUQvRUFoN0dNMFREVHcyT1k2Nm9Rb2JnQW1RbHhka0pEUGYvNGNOCk13Z2tDZWd0UEFEdUkzMzZqU29QaFhYWHgwQ3NVbEk0bGJnUVlOaTRrdXRFZG90UzdPWEhMdERUMzRVUjFqUVIKSit5dWhrOVdUQ29uTnhsY2dscEIyK1Q4TE11bUtHT2NQMmgwUHNrTkZPMkZxU2JSK2c9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    **client-key-data**: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBdHBzNExWN2dMUnpndmd0N3JSbkM4Qks2WFpvd09CRU5CUjllVWtNRFNza3NXd1lEClZwTnpUMUhvSHQ2djUrVEhJdmRQM0VUanQ4MzFDVXZaVlgrL3N4WVE4aTRNWmt6cUlvOHd6VDlHcmRPaWpjQm4KaUdZRkQ1L1BkL2x4RVB5ZXFvMHdlSm9aalduaHJMTTBhcTMyeUxvZERQU0NKMEEwaDcxMkF3OUl3ZWJuNm1oTgp2RnpJMDNnT1BQTGJlTEFuR2VPaVpySitTaFJJQTFIUWprVFlXamdnd3o1RWxWMThKYm03WHVpbTR2ejB5NFV5CnRJekZWSnRBY1J1d042WGkvb3ppK3hTaVdIUUNWNHFpME83aTAvVU11aUxraVRrT1IzenI3bmI4Z01YbUFxbWYKVm1rcHBzZHV3Tlp4VW1jZWl5aVI2MGFUQzRnYlBGS1YreFJTTHdJREFRQUJBb0lCQUZIZmRtaWhTVkh3eUxOcwo0cDdTRmgwZHlJRi9TRzlhOWNOK05RUWRGN1RJVGlMaHAwMkIvd2xwWi9HdlZwOWFiQTY1WkEwV3RpTUxMUHBtCkQ2UE9DMTE0WDFDMlpNalpZNERyUXE1RDJLVEhadkszZWJRbVNjNmZrSjN5TVVlMGZFOXJ6bmZFWUFDUG9LZVcKRWNKakRXc2lSelF2ek10Y2RqRUdPWXRWcHdHSWl5aUhySGRYL0NhbDdsNG5ZRzdXMHUvMExwVmRyZWNXNUV0dgp3OXJJR0ExSkJzRVBTeWJ3T1lnWkdjNTdubjY2V2RjQ3l3SHdiamprc0hqK0NxeFhVZWRDcEFLZ2M1QmRFN1piCmNJbTdLZGNrSEtIYUhZVHptTzFzcXgyU05GaEx6ZUlNVHFoUGQ3MzVCQnBORVM2bDdabWJwcHllRzZOYURGL1cKcUpERDc4RUNnWUVBd21tWVV1cTdrVENjWW9kaGZIbktLZmJVeWw4ci9NemVjSllPczlEcXhmSU0xaDZTVTY5ZQprenJHcTI2MFR1WGU3K3RpVk9vU1FIMXBTL2h4Uml1T2gzK3cyZ3NLdk95citlamwydThMWUJPZGU2WGo2TmdSClRSeGNOQ25Qbll0OXg0OU12TDFiN1dmQzB3N3EwQmlUNTQvc0kvUUV5UmIxbGYxeFZ2MW9PUzBDZ1lFQThIUXMKSlh1RG9wQ0NhZkxhRkFRenI3YWRxdG9MbWFINEQwM2MzeXlmOUVBbk5zTGVLQXQydDF3ZG9ZcGwvcTkzYjdYYgo1WHgwOFJIYTlkRFlxV0c3Z3RGRnZPdlJZeFE4UU51RTNtdGV4bWhtdWx1Z29xQlpNRlBhTGRFVlZsMFBlSEwrCnlYZG5iSC9Va1gxL1c5Z2QwdmdXdUp3ZThCRTlBUEZaZUtMUkdrc0NnWUVBazJpMWtzbGhCeW13cWhTMG1rbE8KUEp0bnBUcWNnOFpqTTBMVVN3dXh0LzFjTms1ZjdRd2Z6Y3JYTU0xejhnN2lCMUNXOG9PNDZ5VXNYZW8zR1ZtVgpiTEFwVEdycTdXMFd5UnNLamdLS3dZS2QrazlDakI2b242dE5UbEFWbUFOWWo2UGNMNC8wMEFISSszZG9HL2xHCnpHR1lUM3FLMWw2T1AvZzNwQm5vbU5FQ2dZQUxTRGdtRGhTUUZSMjVZT2F3bDczaEdiMXVIY3I3aTJqN050a04KTTZmUnF3enIrZHE0b0VrU3MyVEVocHpnaFZVaVRiTWlvbU5PU0Zzd3UzcmUvN0h2b21nV1JDNVA2c3drOHVmYQpFOG1mbjVocVdCQkNjU21lSmVFUDAwYWdCYi9MRkFJMmE3N1RqVy9vMzYyUkhxUFBtVXBmb1J1bWdmaU55Y1U3CjdzL0czd0tCZ1FDdFlRVTRLMFZIM3BDbFlaWVB6a1ZOT2k1NVc0RGNERm9WRkV4Q1V2a1hpZlZrcW42dG0vTzIKMEdHVWF6bDJIV2UwZ2I2TC9qSmlhc3BNanpxRWZsQzlkOEduT0VSK2VJY1pQV2E1VEl2VTZPYUZyWkRYRWdIQwo4UE9FYWt6MU1NQU0wMWZ2eGpPU21CWTdHL3pRSXVvVWNMNHc2eHV5VHcyL1JqT0JOWHhzTlE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

여기서 **certificate-authority-data**는 /etc/kubernetes/pki/ca.crt 파일을 의미한다.

나머지  **client-certificate-data**와 **client-key-data**는 API 서버에 접속할 때 클라이언트 인증을 위한 인증서와 개인키다. 이 파일들을 따로 저장하면 kubelet config 없이도 api에 접속할 수 있다.

우선 admin 클라이언트의 인증서를 저장해보자.

**admin 클라이언트 인증서 저장**

```json
**# sudo cat /etc/kubernetes/admin.conf | grep client-certificate-data | awk '{print $2}' | base64 -d**
-----BEGIN CERTIFICATE-----
MIIDITCCAgmgAwIBAgIIOpdgtdZuQMcwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0yMjA5MjYwODU5MzRaFw0yMzA5MjYwODU5MzdaMDQx
FzAVBgNVBAoTDnN5c3RlbTptYXN0ZXJzMRkwFwYDVQQDExBrdWJlcm5ldGVzLWFk
bWluMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtps4LV7gLRzgvgt7
rRnC8BK6XZowOBENBR9eUkMDSsksWwYDVpNzT1HoHt6v5+THIvdP3ETjt831CUvZ
VX+/sxYQ8i4MZkzqIo8wzT9GrdOijcBniGYFD5/Pd/lxEPyeqo0weJoZjWnhrLM0
aq32yLodDPSCJ0A0h712Aw9Iwebn6mhNvFzI03gOPPLbeLAnGeOiZrJ+ShRIA1HQ
jkTYWjggwz5ElV18Jbm7Xuim4vz0y4UytIzFVJtAcRuwN6Xi/ozi+xSiWHQCV4qi
0O7i0/UMuiLkiTkOR3zr7nb8gMXmAqmfVmkppsduwNZxUmceiyiR60aTC4gbPFKV
+xRSLwIDAQABo1YwVDAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYBBQUH
AwIwDAYDVR0TAQH/BAIwADAfBgNVHSMEGDAWgBRedvFNmJjDx97HP60j0++OrfAk
hzANBgkqhkiG9w0BAQsFAAOCAQEATaYuwBasUTVlbCCoZaYnVy2l10uUH0zhv35O
m+K5DePAu4Bn4MyASshSqIy6fqVh9AGS6b0OxtKWkaeKkL4loLvRsbrZyOccz/Db
H5tNG6MoZ0wmb1nrfLrdrEybV815iuj9qbRuJa7rEaprmFRK5mZqErMtQGBNDi9f
FkAuWFo7zlsZmSzN5iQbgIhVseD/EAh7GM0TDTw2OY66oQobgAmQlxdkJDPf/4cN
MwgkCegtPADuI336jSoPhXXXx0CsUlI4lbgQYNi4kutEdotS7OXHLtDT34UR1jQR
J+yuhk9WTConNxlcglpB2+T8LMumKGOcP2h0PskNFO2FqSbR+g==
-----END CERTIFICATE-----

**# sudo cat /etc/kubernetes/admin.conf | grep client-certificate-data | awk '{print $2}' | base64 -d > admin.crt

# ls admin.crt** 
admin.crt
```

개인키도 저장한다.

**admin 클라이언트 개인 키 저장**

```json
**# sudo cat /etc/kubernetes/admin.conf | grep client-key-data | awk '{print $2}' | base64 -d**
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAtps4LV7gLRzgvgt7rRnC8BK6XZowOBENBR9eUkMDSsksWwYD
VpNzT1HoHt6v5+THIvdP3ETjt831CUvZVX+/sxYQ8i4MZkzqIo8wzT9GrdOijcBn
iGYFD5/Pd/lxEPyeqo0weJoZjWnhrLM0aq32yLodDPSCJ0A0h712Aw9Iwebn6mhN
vFzI03gOPPLbeLAnGeOiZrJ+ShRIA1HQjkTYWjggwz5ElV18Jbm7Xuim4vz0y4Uy
tIzFVJtAcRuwN6Xi/ozi+xSiWHQCV4qi0O7i0/UMuiLkiTkOR3zr7nb8gMXmAqmf
VmkppsduwNZxUmceiyiR60aTC4gbPFKV+xRSLwIDAQABAoIBAFHfdmihSVHwyLNs
4p7SFh0dyIF/SG9a9cN+NQQdF7TITiLhp02B/wlpZ/GvVp9abA65ZA0WtiMLLPpm
D6POC114X1C2ZMjZY4DrQq5D2KTHZvK3ebQmSc6fkJ3yMUe0fE9rznfEYACPoKeW
EcJjDWsiRzQvzMtcdjEGOYtVpwGIiyiHrHdX/Cal7l4nYG7W0u/0LpVdrecW5Etv
w9rIGA1JBsEPSybwOYgZGc57nn66WdcCywHwbjjksHj+CqxXUedCpAKgc5BdE7Zb
cIm7KdckHKHaHYTzmO1sqx2SNFhLzeIMTqhPd735BBpNES6l7ZmbppyeG6NaDF/W
qJDD78ECgYEAwmmYUuq7kTCcYodhfHnKKfbUyl8r/MzecJYOs9DqxfIM1h6SU69e
kzrGq260TuXe7+tiVOoSQH1pS/hxRiuOh3+w2gsKvOyr+ejl2u8LYBOde6Xj6NgR
TRxcNCnPnYt9x49MvL1b7WfC0w7q0BiT54/sI/QEyRb1lf1xVv1oOS0CgYEA8HQs
JXuDopCCafLaFAQzr7adqtoLmaH4D03c3yyf9EAnNsLeKAt2t1wdoYpl/q93b7Xb
5Xx08RHa9dDYqWG7gtFFvOvRYxQ8QNuE3mtexmhmulugoqBZMFPaLdEVVl0PeHL+
yXdnbH/UkX1/W9gd0vgWuJwe8BE9APFZeKLRGksCgYEAk2i1kslhBymwqhS0mklO
PJtnpTqcg8ZjM0LUSwuxt/1cNk5f7QwfzcrXMM1z8g7iB1CW8oO46yUsXeo3GVmV
bLApTGrq7W0WyRsKjgKKwYKd+k9CjB6on6tNTlAVmANYj6PcL4/00AHI+3doG/lG
zGGYT3qK1l6OP/g3pBnomNECgYALSDgmDhSQFR25YOawl73hGb1uHcr7i2j7NtkN
M6fRqwzr+dq4oEkSs2TEhpzghVUiTbMiomNOSFswu3re/7HvomgWRC5P6swk8ufa
E8mfn5hqWBBCcSmeJeEP00agBb/LFAI2a77TjW/o362RHqPPmUpfoRumgfiNycU7
7s/G3wKBgQCtYQU4K0VH3pClYZYPzkVNOi55W4DcDFoVFExCUvkXifVkqn6tm/O2
0GGUazl2HWe0gb6L/jJiaspMjzqEflC9d8GnOER+eIcZPWa5TIvU6OaFrZDXEgHC
8POEakz1MMAM01fvxjOSmBY7G/zQIuoUcL4w6xuyTw2/RjOBNXxsNQ==
-----END RSA PRIVATE KEY-----

**# sudo cat /etc/kubernetes/admin.conf | grep client-key-data | awk '{print $2}' | base64 -d > admin.key

# ls admin.key**
admin.key
```

기존의 CA 파일과 함께 API 서버에 접속해보자. API 서버 주소는 아래와 같다.

```json
**# kubectl -n kube-system describe pod kube-apiserver-k8s-master | grep advertise-address**
Annotations:          kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.122.10:6443
      --advertise-address=**192.168.122.10**
```

curl로 접속을 시도해본다.

```json
**# curl https://192.168.122.10:6443/api --cacert /etc/kubernetes/pki/ca.crt --cert admin.crt --key admin.key** 
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.122.10:6443"
    }
  ]
```

### 02. 토큰 인증

### 서비스어카운트의 token 인증

서비스어카운트 같은 경우엔 인증서를 발급받을 수 없는 대신 Bearer 방식의 토큰을 사용한다. 자세한 내용은 [04. ServiceAccount의 토큰](https://www.notion.so/04-ServiceAccount-78b0bc9f5ba6465abde0e7ed95f7d781?pvs=21) 를 확인하자.

서비스어카운트는 HTTP Request를 할때 HTTP 헤더 값에 “Authorization : Bearer(토큰값) 을 추가해서 인증을 받는다. Pod에 접속해서 아래 명령어를 사용해 보자.

<aside>
💡 **# curl https://kubernetes.default -H “Authorization: Bearer $(/var/run/secrets/kubernetes.io/serviceaccount/token)" -k**

</aside>

```basic
**# kubectl get pod**
NAME       READY   STATUS    RESTARTS   AGE
nginx-aa   1/1     Running   0          3d15h
nginx-rc   1/1     Running   0          3d15h

**# kubectl exec -it nginx-rc -- bash
root@nginx-rc:/# curl https://kubernetes.default/api -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" -k**
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.122.10:6443"
    }
  ]
}
```

### 클러스터 조인을 위한 token

아래는 클러스터에 합류하기 위한 node의 승인을 위해서 제공하는 token 값의 정보다.

```json
**# kubeadm token create --print-join-command**
kubeadm join 192.168.122.10:6443 **--token sfquq9.uipjl3fu512j8z4g** --discovery-token-ca-cert-hash sha256:db460ad0310e47d6db152fef6c1f946314f74e5ef0e56cebaa48ccfe11be60ed
```

노드가 처음 클러스터 그룹에 포함할때 API Server의 CA로부터 클라이언트 인증서를 발급받는 TLS BootStrap 과정을 수행한다. 하지만 이때 인증서 발급 전이라 API Server로부터 적절한 인증을 받을 수 없으니 kubeadm에서 발급된 token으로 임시 인증을 진행하는 것이다.

## 03. Authorization (인가)

Authorization는 API 서버로 들어오는 HTTP Request에 대해 허용할지 거절하지를 결정한다. 적절한 권한이 없는 사용자가 권한 밖의 요청을 하게 된다면 API 서버는 해당 단계에서 막을 수 있다.

- Node : 각 노드에 설치된 kubelet의 Authorization을 담당한다. [K8s Node Authorization](https://www.notion.so/K8s-Node-Authorization-0fea8bc3d5cf4dc79803a97ab1953ddf?pvs=21) 을 참고하자.
- RBAC : 쿠버네티스 인증의 기본이다. [K8s RBAC - Role, ClusterRole 관리](https://www.notion.so/K8s-RBAC-Role-ClusterRole-398fcf1f53d748d99a1f3f622975cb8c?pvs=21) 를 참고하자
- ABAC :  Attribute-bases access control의 약자다. [K8s ABAC 관리](https://www.notion.so/K8s-ABAC-568e42ec1ce34d1f8113336ba60d83a5?pvs=21) 를 참고하자.
- Webhook: 어떤 이벤트가 발생하면특정 url로 이벤트 정보와 함께 요청을 보내는 방식을 뜻한다. LDAP이나 AD와 연동하기 위해서 사용한다.

## 04. Admission Control(승인관리)

자세한 내용은 [K8s Admission Controller](https://www.notion.so/K8s-Admission-Controller-142ce1393bcf4593b71fd9b0db48497e?pvs=21) 에서 확인하자
