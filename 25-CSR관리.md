# K8s RBAC - CertificateSigningRequests 관리

[Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)

# 01. CSR이란?

## 01. 개요

CSR은 인증서가 필요한 사용자(사람 혹은 회사 혹은 프로그램)가 인증서를 생성하기 위해 CA에 전달할 정보를 모아놓은 파일을 뜻한다.

CSR과 인증서, PKI 구조에 대해서 자세한 설명은 [x509 Certificate](https://www.notion.so/x509-Certificate-0adf6e4f50344a5b8abf1ee991fdae75?pvs=21) 에서 확인할 수 있다.

# 02. 인증서를 통한 API 접근 통제

## 01. 쿠버네티스에서의 PKI

클라이언트, 특히 개인 유저가 API 서버에 접근하기 위해서는 x509 인증서가 필요하다. TLS 통신을 하는 과정에서 서버가 클라이언트에 Certification Request를 하기 때문이다. 이때 클라이언트의 인증서는 당연히 쿠버네티스의 CA가 서명한 인증서여야만 한다.

## 02. API서버와 CA

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/77718b59-936b-41af-a885-aa0859ba174b/Untitled.png)

클러스터의 API Server는 자신에게 API 접근을 하는 클라이언트를 검증하기 위해 자신의 CA 인증서로 서명한 클라이언트 인증서를 발급한다. 

클라이언트는 API Server가 발급해준 인증서를 통해서 API 요청을 보낼 수 있다.

# 03. CSR 리소스와 API Client 유저 관리

## 01. CSR 리소스 등장 배경

API서버의 CA 인증서는 클러스터 어드민도 관리할 수 없다. openssl 명령어로 CA 서명이 들어간 인증서를 생성하려고 해도 아래와 같은 에러메시지가 발생하면서 생성되지 않는다.

```json
**# openssl x509 -in csrtest.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -out csrtest.crt -days 3650**
**unable to load certificate
139929184503104:error:0909006C:PEM routines:get_name:no start line:../crypto/pem/pem_lib.c:745:Expecting: TRUSTED CERTIFICATE**
```

일반적인 방법으로는 CA 서명이 포함된 인증서를 생성할 수 없다. 대신 쿠버네티스는 CSR을 API 리소스로 관리한다.

## 02. 리소스 형식

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

- spec.request : CSR의 요청문이다. base64로 인코딩해서 추가한다.
- signerName : 이 부분의 값은 kubernetes.io/kube-api-server로 고정이다.
- expirationSeconds : 조금 불합리해보이는데 초단위로 인증서 관리를 한다. 하루동안의 인증서를 보관하기 위해 86400이라는 긴 값을 입력해야 한다.
- usages : x509 확장자의 keyUsage 필드를 지정한다. client auth, digital signature, key encipherment 외에 다른 용도를 지정할 수 없다.

## 03. CSR 리소스를 통해 API 클라이언트 인증서 발급

### 01. 개인키 생성

<aside>
💡 **# openssl genrsa -aes256 -out {개인키 파일} 2048**

</aside>

```json
**# openssl genrsa -aes256 -out csrpre.key 2048**
Generating RSA private key, 2048 bit long modulus (2 primes)
...................+++++
.........................................................................................................+++++
e is 65537 (0x010001)
Enter pass phrase for csrpre.key:
Verifying - Enter pass phrase for csrtest.key:
```

개인키의 비밀번호는 필요 없으니 없애버린다.

<aside>
💡 **# openssl rsa -in {기존 개인키파일} -out {새 개인키 파일}**

</aside>

```json
**# openssl rsa -in csrpre.key -out csrtest.key**
Enter pass phrase for csrpre.key:
writing RSA key
```

### 02. CSR 생성

<aside>
💡 **# openssl req -new -key {개인키 파일} -out {csr 파일명}**

</aside>

```json
**# openssl req -new -key csrtest.key -out csrtest.csr**
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:KR
State or Province Name (full name) [Some-State]:Seoul
Locality Name (eg, city) []:Gang-nam
Organization Name (eg, company) [Internet Widgits Pty Ltd]:MYCOMPANY
Organizational Unit Name (eg, section) []:Cloud
Common Name (e.g. server FQDN or YOUR name) []:**csrtest**
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

CSR을 생성할 때 CN(Common Name)은 관리적인 차원에서 CSR 리소스 이름과 동일하게 하는 것이 좋다.

### 03. Request를 base64로 인코딩

<aside>
💡 **# cat {csr파일} | base64 | tr -d “\n”**

</aside>

```json
**# cat csrtest.csr | base64 | tr -d "\n"**
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ3F6Q0NBWk1DQVFBd1pqRUxNQWtHQTFVRUJoTUNTMUl4RGpBTUJnTlZCQWdNQlZObGIzVnNNUkV3RHdZRApWUVFIREFoSFlXNW5MVzVoYlRFU01CQUdBMVVFQ2d3SlRWbERUMDFRUVU1Wk1RNHdEQVlEVlFRTERBVkRiRzkxClpERVFNQTRHQTFVRUF3d0hZM055ZEdWemREQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0MKZ2dFQkFNV2VsbmdHbnpLREs2Z2o4MDNOLzkrL0JDU21GZ1ZVQ1padjFSQkxhZ2FuTmR6RjMxWkxDZU91V2RqbwpvVkdBd1lIalRXRkV2YU1CMytVUGZMbUVSQk9PQzhRR2NUbDdqbC9JWEJkTDlTcFhxMndLNmt1UWpWaWdSRzhRCmcxME01d0ZLT2lMMkUwQm5lcHZBV0JtMG5XVGZjTTg4d0tScFU1RmtMY1p1bis0ZHdvZXVmcnN3QnhBU3QxVU0KYXFiUkVyRnhFSFBrUmFDVm5JK1Z2QXdPRjhJY3E1alpveEZWTzVrU2VIUzJNczZ5RkRMeFJtQkNlREl3NWZwTQpScW9VNkJxVDdUeEpXTlpIWU0ybkxjdnNoNHgxNEo0eGtPazdoYmVjOTRDNkpCbGw3UTIrSHFEK091UGthQk0rCm03a3VETXBWd2RHV0pOYXoranBScnBrTDJRc0NBd0VBQWFBQU1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQisKTHRFcGZobVFremYzMHNKaGN2amc0WFgyUUozR3dHOUhORlRyOFVxN0FGRTRIUFgrSUpBNkl2NGV3RlVyQmI0YwpGR3JVV2l0ZVdnN1BRRkNPUXZGNGZlU2VieWdTTkVOaUJkaWg3Q29heWZNUElIaytGd0VmR1FGbkFWa0thbnJ6Cm5hMDRIZG91b01UMzZTV2h2QVlLZ3FnTkR3bENzMWNHYU9hL0hIcDkvb21jUE5vWk14VHE3NkFNOWsrQk56SEwKcTVhb1Ryc3VmYlpZeVhJY3FpVDV5Q2U1b0hyRUxJZjlLSUtxdTNNVWhmNUEwdlZKcFljclNONFNmZW5lSGZSagpLZWc1ZnJGUGhKejlMeDdEamxia0ZTVzBJdTBqTVlQLzRGSUNzaFYyU3dVR1NBb2hsN21kQ29YY0c1cExrNDVPCk40WE5jSHp4dVp5bjZOTzZpK1lRCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
```

위에 인코딩 결과를 카피한다.

### 04. CSR 리소스 생성

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: csrtest
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ3F6Q0NBWk1DQVFBd1pqRUxNQWtHQTFVRUJoTUNTMUl4RGpBTUJnTlZCQWdNQlZObGIzVnNNUkV3RHdZRApWUVFIREFoSFlXNW5MVzVoYlRFU01CQUdBMVVFQ2d3SlRWbERUMDFRUVU1Wk1RNHdEQVlEVlFRTERBVkRiRzkxClpERVFNQTRHQTFVRUF3d0hZM055ZEdWemREQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0MKZ2dFQkFNV2VsbmdHbnpLREs2Z2o4MDNOLzkrL0JDU21GZ1ZVQ1padjFSQkxhZ2FuTmR6RjMxWkxDZU91V2RqbwpvVkdBd1lIalRXRkV2YU1CMytVUGZMbUVSQk9PQzhRR2NUbDdqbC9JWEJkTDlTcFhxMndLNmt1UWpWaWdSRzhRCmcxME01d0ZLT2lMMkUwQm5lcHZBV0JtMG5XVGZjTTg4d0tScFU1RmtMY1p1bis0ZHdvZXVmcnN3QnhBU3QxVU0KYXFiUkVyRnhFSFBrUmFDVm5JK1Z2QXdPRjhJY3E1alpveEZWTzVrU2VIUzJNczZ5RkRMeFJtQkNlREl3NWZwTQpScW9VNkJxVDdUeEpXTlpIWU0ybkxjdnNoNHgxNEo0eGtPazdoYmVjOTRDNkpCbGw3UTIrSHFEK091UGthQk0rCm03a3VETXBWd2RHV0pOYXoranBScnBrTDJRc0NBd0VBQWFBQU1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQisKTHRFcGZobVFremYzMHNKaGN2amc0WFgyUUozR3dHOUhORlRyOFVxN0FGRTRIUFgrSUpBNkl2NGV3RlVyQmI0YwpGR3JVV2l0ZVdnN1BRRkNPUXZGNGZlU2VieWdTTkVOaUJkaWg3Q29heWZNUElIaytGd0VmR1FGbkFWa0thbnJ6Cm5hMDRIZG91b01UMzZTV2h2QVlLZ3FnTkR3bENzMWNHYU9hL0hIcDkvb21jUE5vWk14VHE3NkFNOWsrQk56SEwKcTVhb1Ryc3VmYlpZeVhJY3FpVDV5Q2U1b0hyRUxJZjlLSUtxdTNNVWhmNUEwdlZKcFljclNONFNmZW5lSGZSagpLZWc1ZnJGUGhKejlMeDdEamxia0ZTVzBJdTBqTVlQLzRGSUNzaFYyU3dVR1NBb2hsN21kQ29YY0c1cExrNDVPCk40WE5jSHp4dVp5bjZOTzZpK1lRCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 864000
  usages:
  - client auth
```

```json
**# kubectl apply -f csrtest.yaml** 
certificatesigningrequest.certificates.k8s.io/csrtest created

**# kubectl get csr csrtest -o yaml**
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"certificates.k8s.io/v1","kind":"CertificateSigningRequest","metadata":{"annotations":{},"name":"csrtest"},"spec":{"expirationSeconds":864000,"request":"LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ3F6Q0NBWk1DQVFBd1pqRUxNQWtHQTFVRUJoTUNTMUl4RGpBTUJnTlZCQWdNQlZObGIzVnNNUkV3RHdZRApWUVFIREFoSFlXNW5MVzVoYlRFU01CQUdBMVVFQ2d3SlRWbERUMDFRUVU1Wk1RNHdEQVlEVlFRTERBVkRiRzkxClpERVFNQTRHQTFVRUF3d0hZM055ZEdWemREQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0MKZ2dFQkFNV2VsbmdHbnpLREs2Z2o4MDNOLzkrL0JDU21GZ1ZVQ1padjFSQkxhZ2FuTmR6RjMxWkxDZU91V2RqbwpvVkdBd1lIalRXRkV2YU1CMytVUGZMbUVSQk9PQzhRR2NUbDdqbC9JWEJkTDlTcFhxMndLNmt1UWpWaWdSRzhRCmcxME01d0ZLT2lMMkUwQm5lcHZBV0JtMG5XVGZjTTg4d0tScFU1RmtMY1p1bis0ZHdvZXVmcnN3QnhBU3QxVU0KYXFiUkVyRnhFSFBrUmFDVm5JK1Z2QXdPRjhJY3E1alpveEZWTzVrU2VIUzJNczZ5RkRMeFJtQkNlREl3NWZwTQpScW9VNkJxVDdUeEpXTlpIWU0ybkxjdnNoNHgxNEo0eGtPazdoYmVjOTRDNkpCbGw3UTIrSHFEK091UGthQk0rCm03a3VETXBWd2RHV0pOYXoranBScnBrTDJRc0NBd0VBQWFBQU1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQisKTHRFcGZobVFremYzMHNKaGN2amc0WFgyUUozR3dHOUhORlRyOFVxN0FGRTRIUFgrSUpBNkl2NGV3RlVyQmI0YwpGR3JVV2l0ZVdnN1BRRkNPUXZGNGZlU2VieWdTTkVOaUJkaWg3Q29heWZNUElIaytGd0VmR1FGbkFWa0thbnJ6Cm5hMDRIZG91b01UMzZTV2h2QVlLZ3FnTkR3bENzMWNHYU9hL0hIcDkvb21jUE5vWk14VHE3NkFNOWsrQk56SEwKcTVhb1Ryc3VmYlpZeVhJY3FpVDV5Q2U1b0hyRUxJZjlLSUtxdTNNVWhmNUEwdlZKcFljclNONFNmZW5lSGZSagpLZWc1ZnJGUGhKejlMeDdEamxia0ZTVzBJdTBqTVlQLzRGSUNzaFYyU3dVR1NBb2hsN21kQ29YY0c1cExrNDVPCk40WE5jSHp4dVp5bjZOTzZpK1lRCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=","signerName":"kubernetes.io/kube-apiserver-client","usages":["client auth"]}}
  creationTimestamp: "2022-10-21T05:46:12Z"
  name: csrtest
  resourceVersion: "3423960"
  uid: 2238b17e-504e-4261-a7d8-274f9dc620f0
spec:
  expirationSeconds: 864000
  groups:
  - system:masters
  - system:authenticated
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ3F6Q0NBWk1DQVFBd1pqRUxNQWtHQTFVRUJoTUNTMUl4RGpBTUJnTlZCQWdNQlZObGIzVnNNUkV3RHdZRApWUVFIREFoSFlXNW5MVzVoYlRFU01CQUdBMVVFQ2d3SlRWbERUMDFRUVU1Wk1RNHdEQVlEVlFRTERBVkRiRzkxClpERVFNQTRHQTFVRUF3d0hZM055ZEdWemREQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0MKZ2dFQkFNV2VsbmdHbnpLREs2Z2o4MDNOLzkrL0JDU21GZ1ZVQ1padjFSQkxhZ2FuTmR6RjMxWkxDZU91V2RqbwpvVkdBd1lIalRXRkV2YU1CMytVUGZMbUVSQk9PQzhRR2NUbDdqbC9JWEJkTDlTcFhxMndLNmt1UWpWaWdSRzhRCmcxME01d0ZLT2lMMkUwQm5lcHZBV0JtMG5XVGZjTTg4d0tScFU1RmtMY1p1bis0ZHdvZXVmcnN3QnhBU3QxVU0KYXFiUkVyRnhFSFBrUmFDVm5JK1Z2QXdPRjhJY3E1alpveEZWTzVrU2VIUzJNczZ5RkRMeFJtQkNlREl3NWZwTQpScW9VNkJxVDdUeEpXTlpIWU0ybkxjdnNoNHgxNEo0eGtPazdoYmVjOTRDNkpCbGw3UTIrSHFEK091UGthQk0rCm03a3VETXBWd2RHV0pOYXoranBScnBrTDJRc0NBd0VBQWFBQU1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQisKTHRFcGZobVFremYzMHNKaGN2amc0WFgyUUozR3dHOUhORlRyOFVxN0FGRTRIUFgrSUpBNkl2NGV3RlVyQmI0YwpGR3JVV2l0ZVdnN1BRRkNPUXZGNGZlU2VieWdTTkVOaUJkaWg3Q29heWZNUElIaytGd0VmR1FGbkFWa0thbnJ6Cm5hMDRIZG91b01UMzZTV2h2QVlLZ3FnTkR3bENzMWNHYU9hL0hIcDkvb21jUE5vWk14VHE3NkFNOWsrQk56SEwKcTVhb1Ryc3VmYlpZeVhJY3FpVDV5Q2U1b0hyRUxJZjlLSUtxdTNNVWhmNUEwdlZKcFljclNONFNmZW5lSGZSagpLZWc1ZnJGUGhKejlMeDdEamxia0ZTVzBJdTBqTVlQLzRGSUNzaFYyU3dVR1NBb2hsN21kQ29YY0c1cExrNDVPCk40WE5jSHp4dVp5bjZOTzZpK1lRCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
  username: kubernetes-admin
status: {}
```

### 05. CSR 승인

**CSR을 승인하는 주체는 클러스터 관리자다.**

인증서 사인 요청이 들어왔다면 이를 승인해서 인증서를 발급해줘야 한다. 승인하기 전까지 csr의 상태는 아래와 같이 pending 상태가 된다.

```json
# **kubectl get csr**
NAME      AGE    SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
csrtest   119s   kubernetes.io/kube-apiserver-client   kubernetes-admin   10d                 **Pending**
```

**CSR 승인**

CSR을 승인하여 인증서를 발급하는 명령어는 아래와 같다.

<aside>
💡 **# kubectl certificate approve {csr 명}**

</aside>

```json
**# kubectl certificate approve csrtest**
certificatesigningrequest.certificates.k8s.io/csrtest approved

**# kubectl get csr**
NAME      AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
csrtest   27m   kubernetes.io/kube-apiserver-client   kubernetes-admin   10d                 **Approved,Issued**
```

**CSR 거절**

만약 승인 거절을 해야할 경우 아래와 같이 한다.

<aside>
💡 **# kubectl certificate deny {csr 명}**

</aside>

```json
**# kubectl certificate deny csrtest**
certificatesigningrequest.certificates.k8s.io/csrtest denied

**# kubectl get csr**
NAME      AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
csrtest   71s   kubernetes.io/kube-apiserver-client   kubernetes-admin   10d                 **Denied**
```

### 06. 클라이언트 인증서 저장

인증서는 클라이언트가 직접 저장하고 보관할 필요가 있다. 다만 API 서버가 인증해준 인증서는 곧바로 파일로 만들어지지 않기 때문에 이를 따로 추출하는 과정이 필요하다.

CSR이 승인되면 .status에 아래와 같이 certificate 항목이 생긴다.

```json
**# kubectl get csr csrtest -o yaml**
(전략)
status:
  **certificate: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURURENDQWpTZ0F3SUJBZ0lSQU5iOG1DYmx6TndIMkxkR0Y4ckRJQjB3RFFZSktvWklodmNOQVFFTEJRQXcKRlRFVE1CRUdBMVVFQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TWpFd01qRXdOakU1TWpsYUZ3MHlNakV3TXpFdwpOakU1TWpsYU1HWXhDekFKQmdOVkJBWVRBa3RTTVE0d0RBWURWUVFJRXdWVFpXOTFiREVSTUE4R0ExVUVCeE1JClIyRnVaeTF1WVcweEVqQVFCZ05WQkFvVENVMVpRMDlOVUVGT1dURU9NQXdHQTFVRUN4TUZRMnh2ZFdReEVEQU8KQmdOVkJBTVRCMk56Y25SbGMzUXdnZ0VpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElCRHdBd2dnRUtBb0lCQVFERgpucFo0QnA4eWd5dW9JL05OemYvZnZ3UWtwaFlGVkFtV2I5VVFTMm9HcHpYY3hkOVdTd25qcmxuWTZLRlJnTUdCCjQwMWhSTDJqQWQvbEQzeTVoRVFUamd2RUJuRTVlNDVmeUZ3WFMvVXFWNnRzQ3VwTGtJMVlvRVJ2RUlOZERPY0IKU2pvaTloTkFaM3Fid0ZnWnRKMWszM0RQUE1Da2FWT1JaQzNHYnAvdUhjS0hybjY3TUFjUUVyZFZER3FtMFJLeApjUkJ6NUVXZ2xaeVBsYndNRGhmQ0hLdVkyYU1SVlR1WkVuaDB0akxPc2hReThVWmdRbmd5TU9YNlRFYXFGT2dhCmsrMDhTVmpXUjJETnB5M0w3SWVNZGVDZU1aRHBPNFczblBlQXVpUVpaZTBOdmg2Zy9qcmo1R2dUUHB1NUxneksKVmNIUmxpVFdzL282VWE2WkM5a0xBZ01CQUFHalJqQkVNQk1HQTFVZEpRUU1NQW9HQ0NzR0FRVUZCd01DTUF3RwpBMVVkRXdFQi93UUNNQUF3SHdZRFZSMGpCQmd3Rm9BVVhuYnhUWmlZdzhmZXh6K3RJOVB2anEzd0pJY3dEUVlKCktvWklodmNOQVFFTEJRQURnZ0VCQUJFRi9QdE1BWjB1R3VCTkJtSWFTR1NEdysxcGNTdW51TXR2RE1URDV6czAKV2pGMFUrRzZPdExIaWZRU1ZWRDZncktueTlpT01haDRzNWx1SG1TNmFXdkZVcHpvekUwSG1nVDZsempXMENucwp0U0xCNVVQUlZrVC9MZDdBOUZVZnFnWmpZNHlETVZhVzZ5czd0dm1iTHBGTEVSLy90ai92SGZWN3RtYmN5R25lCnlHOFd4UDhjZ0hldGxLQTRmWjEwVldkckZYSGpYMjdDUWg2aytVMnpQZi90Q2VNVmx4Y0d1ZkxuMzVrQjAxVU0KcWxCQm9IWnliS0VKanNoang5RDk1cDViOGcxTWVSUzlrM2dlMWVLRVNsYlRLalpxU0MxK2NyRXA0cFVNOVVGKwpCMTRyNHd6djVNSCtrQzgrbnAraFR3V29sS24xOENKZ0JxVDhRZXRjVkVnPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==**
  conditions:
  - lastTransitionTime: "2022-10-21T06:24:29Z"
    lastUpdateTime: "2022-10-21T06:24:29Z"
    message: This CSR was approved by kubectl certificate approve.
    reason: KubectlApprove
    status: "True"
    type: Approved
```

위 내용을 추출해서 base64 디코드를 한 뒤 인증서 파일로 저장한다.

<aside>
💡 **# kubectl get csr {csr 명} -o jsonpath='{.status.certificate}' | base64 -d > {인증서 명}**

</aside>

```json
**# kubectl get csr csrtest -o jsonpath='{.status.certificate}' | base64 -d > csrtest.crt

# ls csrtest.crt**
csrtest.crt

**# openssl x509 -in csrtest.crt -noout -text**
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            d6:fc:98:26:e5:cc:dc:07:d8:b7:46:17:ca:c3:20:1d
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Oct 21 06:19:29 2022 GMT
            Not After : Oct 31 06:19:29 2022 GMT
        Subject: C = KR, ST = Seoul, L = Gang-nam, O = MYCOMPANY, OU = Cloud, CN = csrtest
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:c5:9e:96:78:06:9f:32:83:2b:a8:23:f3:4d:cd:
                    ff:df:bf:04:24:a6:16:05:54:09:96:6f:d5:10:4b:
                    6a:06:a7:35:dc:c5:df:56:4b:09:e3:ae:59:d8:e8:
                    a1:51:80:c1:81:e3:4d:61:44:bd:a3:01:df:e5:0f:
                    7c:b9:84:44:13:8e:0b:c4:06:71:39:7b:8e:5f:c8:
                    5c:17:4b:f5:2a:57:ab:6c:0a:ea:4b:90:8d:58:a0:
                    44:6f:10:83:5d:0c:e7:01:4a:3a:22:f6:13:40:67:
                    7a:9b:c0:58:19:b4:9d:64:df:70:cf:3c:c0:a4:69:
                    53:91:64:2d:c6:6e:9f:ee:1d:c2:87:ae:7e:bb:30:
                    07:10:12:b7:55:0c:6a:a6:d1:12:b1:71:10:73:e4:
                    45:a0:95:9c:8f:95:bc:0c:0e:17:c2:1c:ab:98:d9:
                    a3:11:55:3b:99:12:78:74:b6:32:ce:b2:14:32:f1:
                    46:60:42:78:32:30:e5:fa:4c:46:aa:14:e8:1a:93:
                    ed:3c:49:58:d6:47:60:cd:a7:2d:cb:ec:87:8c:75:
                    e0:9e:31:90:e9:3b:85:b7:9c:f7:80:ba:24:19:65:
                    ed:0d:be:1e:a0:fe:3a:e3:e4:68:13:3e:9b:b9:2e:
                    0c:ca:55:c1:d1:96:24:d6:b3:fa:3a:51:ae:99:0b:
                    d9:0b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Extended Key Usage: 
                TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier: 
                keyid:5E:76:F1:4D:98:98:C3:C7:DE:C7:3F:AD:23:D3:EF:8E:AD:F0:24:87

    Signature Algorithm: sha256WithRSAEncryption
         11:05:fc:fb:4c:01:9d:2e:1a:e0:4d:06:62:1a:48:64:83:c3:
         ed:69:71:2b:a7:b8:cb:6f:0c:c4:c3:e7:3b:34:5a:31:74:53:
         e1:ba:3a:d2:c7:89:f4:12:55:50:fa:82:b2:a7:cb:d8:8e:31:
         a8:78:b3:99:6e:1e:64:ba:69:6b:c5:52:9c:e8:cc:4d:07:9a:
         04:fa:97:38:d6:d0:29:ec:b5:22:c1:e5:43:d1:56:44:ff:2d:
         de:c0:f4:55:1f:aa:06:63:63:8c:83:31:56:96:eb:2b:3b:b6:
         f9:9b:2e:91:4b:11:1f:ff:b6:3f:ef:1d:f5:7b:b6:66:dc:c8:
         69:de:c8:6f:16:c4:ff:1c:80:77:ad:94:a0:38:7d:9d:74:55:
         67:6b:15:71:e3:5f:6e:c2:42:1e:a4:f9:4d:b3:3d:ff:ed:09:
         e3:15:97:17:06:b9:f2:e7:df:99:01:d3:55:0c:aa:50:41:a0:
         76:72:6c:a1:09:8e:c8:63:c7:d0:fd:e6:9e:5b:f2:0d:4c:79:
         14:bd:93:78:1e:d5:e2:84:4a:56:d3:2a:36:6a:48:2d:7e:72:
         b1:29:e2:95:0c:f5:41:7e:07:5e:2b:e3:0c:ef:e4:c1:fe:90:
         2f:3e:9e:9f:a1:4f:05:a8:94:a9:f5:f0:22:60:06:a4:fc:41:
         eb:5c:54:48
```

## 04. 인증서를 통해 유저 생성

인증서를 생성했다면 해당 인증서를 사용할 클라이언트, 즉 유저를 생성해야한다. RoleBinding이나 ClusterRoleBinding에서 

### 01. kube config에 유저 등록

~/.kube/config에 클러스터를 사용할 유저 정보를 저장한다. 유저는 인증서와 개인키 정보가 함께 제공되야한다.

<aside>
💡 # kubectl config set-credentials {유저명} --client-key={클라이언트 개인키} --client-certificate={클라이언트 인증서} --embed-certs=true

</aside>

- --embed-certs : 해당 설정이 true로 되어있지 않으면 ~/.kube/config 파일에 신규 유저의 인증서와 개인키 정보가 추가되지 않는다.

```json
**# kubectl config set-credentials csrtest --client-key=csrtest.key --client-certificate=csrtest.crt --embed-certs=true**
User "csrtest" set.

**# kubectl config view**
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.122.10:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
**- name: csrtest
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED**
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

**# kubectl config get-users** 
NAME
**csrtest**
kubernetes-admin
```

### 02. 유저 사용을 위한 context 등록

context는 일종의 프리셋으로 특정 쿠버네티스 클러스터나 유저 계정의 접속정보를 미리 기억해놨다가 빠르게 접속할 수 있도록 설정한 구문이다. *자세한 내용은 차후에 정리하겠다.*

[01. kube config에 유저 등록](https://www.notion.so/01-kube-config-bf9f1bde700145d3bcb41be4b1c85d58?pvs=21)에서 생성한 user로 apiserver에 접속하기 위한 context를 생성하자. 

<aside>
💡 **# kubectl config set-context {컨텍스트 명} --cluster={클러스터 명} --user={유저 명}**

</aside>

```json
**# kubectl config set-context csrtest --cluster=kubernetes --user=csrtest**
Context "csrtest" created.

**# kubectl config view**
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.122.10:6443
  name: kubernetes
contexts:
**- context:
    cluster: kubernetes
    user: csrtest
  name: csrtest**
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: csrtest
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

**# kubectl config get-contexts** 
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          csrtest                       kubernetes   csrtest            
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

### 03. 유저의 Role 또는 ClusterRole 생성

생성된 유저에게는 아무런 Role이 없기 때문에 어떠한 작업도 진행할 수 없다. 따라서 유저에게 매핑될 Role이나 ClusterRole을 생성한다.

Role, ClusterRole에 대한 자세한 내용은 [K8s RBAC - Role, ClusterRole 관리](https://www.notion.so/K8s-RBAC-Role-ClusterRole-398fcf1f53d748d99a1f3f622975cb8c?pvs=21) 를 참고하자

현 테스트에서는 pod, deployment, statefulset, daemonset에 대한 모든 권한을 지닌 ClusterRole을 생성한다.

```json
**# kubectl create clusterrole app-master --resource=Pod,Deployment,StatefulSet,DaemonSet --verb=get,watch,list,create,delete,update**
clusterrole.rbac.authorization.k8s.io/app-master created

**# kubectl get clusterrole app-master**
NAME         CREATED AT
app-master   2022-10-21T07:28:28Z
```

### 04. RoleBinding 또는 ClusterRoleBinding 생성

RoleBindinng 및 ClusterRoleBinding과 관련된 자세한 내용은 [K8s RBAC - Role, ClusterRole 관리](https://www.notion.so/K8s-RBAC-Role-ClusterRole-398fcf1f53d748d99a1f3f622975cb8c?pvs=21) 를 참고한다. 

현 테스트에서는 [03. 유저의 Role 또는 ClusterRole 생성](https://www.notion.so/03-Role-ClusterRole-db523b9d71fd49ce85e44df7e23b147e?pvs=21) 에서 만든 ClusterRole과 유저 csrtest를 바인딩한다.

```json
**# kubectl create clusterrolebinding app-master-user:csrtest --clusterrole=app-m-master-user:csrtest --claster --user=csrtest**
clusterrolebinding.rbac.authorization.k8s.io/app-master-user:csrtest created

**# kubectl get clusterrolebinding app-master-user:csrtest -o wide**
NAME                      ROLE                     AGE   USERS     GROUPS   SERVICEACCOUNTS
**app-master-user:csrtest   ClusterRole/app-master   30s   csrtest**
```

## 05. 생성한 유저 사용

아래 명령어로 생성된 유저로 클러스터에 접속할 수 있다.

<aside>
💡 **# kubectl config use-context {컨텍스트명}**

</aside>

```json
**# kubectl config use-context csrtest** 
Switched to context "csrtest".

**# kubectl config current-context** 
csrtest
```

## 06. 유저 삭제

유저를 삭제하기 위해선 우선 config의 해당 유저가 사용하는 context를 제거한 뒤, confg의 유저 삭제, Rolebinding 삭제로 진행한다.

### 01. context 삭제

<aside>
💡 **# kubectl config delete-context {컨텍스트 명}**

</aside>

```json
**# kubectl config delete-context csrtest**
deleted context csrtest from /home/student/.kube/config
```

### 02. 유저 삭제

<aside>
💡 **# kubectl config delete-user {유저명}**

</aside>

```json
**# kubectl config delete-user csrtest**
deleted user csrtest from /home/student/.kube/config
```

### 03. 삭제된 유저와 binding된 Role 및 ClusterRole 검색

kube config에서 유저가 삭제되었다 해도, Role이나 ClusterRole에는 해당 유저가 여전히 바인딩되어 있다. 만약 다른 디바이스에서 해당 유저정보로 접근하게 된다면 스니핑 등 보안 사고가 발생할수 있으니 binding된 내용을 전부 걷어내야 할 것이다.

좀 복잡하지만 아래 명령어를 사용하면 삭제된 유저와 바인딩된 role과 clusterrole을 찾을 수 있다. 

- **RoleBindng**
    
    <aside>
    💡 # **kubectl get rolebindings -A -o jsonpath='{range .items[*]}{.metadata.namespace}:{.metadata.name}{"\t"}{.subjects[?(@.kind=="User")].name}{"\n"}{end}' | grep {삭제된 유저명}**
    
    </aside>
    
    ```json
    **# kubectl get rolebindings -A -o jsonpath='{range .items[*]}{.metadata.namespace}:{.metadata.name}{"\t"}{.subjects[?(@.kind=="User")].name}{"\n"}{end}' | grep csrtest**
    default:app-master	**csrtest**
    ```
    
- **ClusterRoleBinding**
    
    <aside>
    💡 **# kubectl get clusterrolebindings -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.subjects[?(@.kind=="User")].name}{"\n"}{end}' | grep {삭제된 유저명}**
    
    </aside>
    
    ```json
    **# kubectl get clusterrolebindings -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.subjects[?(@.kind=="User")].name}{"\n"}{end}' | grep csrtest**
    app-master-user:csrtest	**csrtest**
    ```
    

검색된 바인딩을 삭제하거나 subjects에서 제거하거나 편한 방식으로 제거하면 된다. 어차피 다 알고 있을텐데 귀찮으니 방법은 설명 안한다.
