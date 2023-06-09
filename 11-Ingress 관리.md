# K8s Ingress 관리

# 01. Ingress란?

L7 로드밸런서라고 생각하면 된다.

# 02. Ingress Controller 설치

## 01. Nginx Controller

[Installation Guide - NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/)

<aside>
💡 **kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.1/deploy/static/provider/baremetal/deploy.yaml**

</aside>

## 02. Istio Controller

Istio를 설치하는 방법은 아래 링크를 통해 확인 가능하다. 

[Istio](https://www.notion.so/Istio-230dfcaa4c5a4ca2b64581cdb4568a06?pvs=21)

# 03. Ingress 리소스 생성

## 01. 명령어로 Ingress 생성

<aside>
💡 **# kubectl create ingress {ingress 명} --rule={path경로}:{서비스명}:{서비스포트} --class={인그레스컨트롤러명}**

</aside>

```json
**# kubectl -n ing-internal create ing ping --rule=/hi=hi:5678 --class=nginx
kubectl -n ing-internal get ing**
NAME   CLASS   HOSTS   ADDRESS   PORTS   AGE
ping   nginx   *                 80      7s
```

명령어로 ingress를 만드는 것은 간편해 보이지만 사실 rewrite-target 관련 annotation 생성이 제한되고, pathType을 설정할 수 없어서 한계가 매우 많다. 그 때문에 명령어 보다는 직접 YAML을 생성해서 만드는 것이 좋다.

## 02. Yaml로 Ingress 생성

### nginx 컨트롤러

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  **annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /**
spec:
  **ingressClassName: nginx**
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

nginx 인그레스를 사용하는 경우 nginx.ingress.kubernetes.io/rewrite-target을 annotaion에 지정해줘야 한다.

rules를 통해서 /testpath로 접근하는 요청에 대해서 test 서비스의 80번 포트로 연결이 된다. 문제는 test 서비스가 라우팅하는 pod 역시 하단에 /testpath 경로가 있을 수 있다는 점이다. 이 경우 /testpath 하단의 주소를 제대로 호출하지 못하는 문제가 발생할 수 있기 때문에, rewrite-target annotation을 추가하여 /testpath로 시도한 접속 경로를 /로 다시 변경해야 한다.

ingressClassName은 Default로 설정된 ingressClass가 없다면 반드시 지정해줘야 한다. ingressClass 목록 확인은 아래 명령어로 할 수 있다.

<aside>
💡 **# kubectl get ingressclass**

</aside>

```json
**# kubectl get ingressclass**
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       3d7h
```

### **istio 컨트롤러**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: istio-ingress-to-nginx
  annotations:
    kubernetes.io/ingress.class: istio
spec:
  rules:
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx-istio
            port:
              number: 80
```

istio 컨트롤러는 annotations에 아래와 같은 주석을 추가해야한다.

kubernetes.io/ingress.class: istio

istio 컨트롤러는 자동으로 IngressClass가 생성되지 않기 때문에 예전에 사용하던 방식대로 annotation으로 ingress class를 설정해줘야 한다.

만약 ingressClass를 설정하고자 한다면 아래와 같이 추가로 IngressClass리소스를 생성해야 한다.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: IngressClass
metadata:
  name: istio
spec:
  controller: istio.io/ingress-controller
```

이렇게 하면 ingressClassName을 사용할 수 있따.

**rewrite 방법은 차차 알아봐야 할 것 같다.**

# 04. Ingress에 TLS 사용

## 01. tls 시크릿 생성

[x509 Certificate](https://www.notion.so/x509-Certificate-0adf6e4f50344a5b8abf1ee991fdae75?pvs=21) 를 참고하여 dashboard 용 인증서와 개인키를 생성했다.

```json
**# ls dashboard***
dashboard.crt  dashboard.csr  dashboard.key
```

이를 기반으로 tls 시크릿을 생성한다.

<aside>
💡 **# kubectl -n {네임스페이스 명} create secret tls {시크릿 명} --cert={인증서 파일} --key={개인키 파일}**

</aside>

```json
**# kubectl create -n kubernetes-dashboard secret tls kubernetes-dashboard-ing-cert --cert dashboard.crt --key dashboard.key** 
secret/kubernetes-dashboard-ing-cert created

**# kubectl -n kubernetes-dashboard get secrets kubernetes-dashboard-ing-cert -o yaml**
apiVersion: v1
**data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURlekNDQW1NQ0ZHcUR5anFFeW94UVhwSmthMmdjWVdKaUUvNktNQTBHQ1NxR1NJYjNEUUVCQ3dVQU1Ib3gKQ3pBSkJnTlZCQVlUQWt0U01STXdFUVlEVlFRSURBcEhlV1Z2Ym1kcExXUnZNUlF3RWdZRFZRUUhEQXRUWlc5dQpaMjVoYlMxemFURVBNQTBHQTFVRUNnd0diWGxvYjIxbE1ROHdEUVlEVlFRTERBWnRlWEp2YjIweEhqQWNCZ05WCkJBTU1GV1JoYzJoaWIyRnlaQzUwWVdWdExtczRjeTVwYnpBZUZ3MHlNakV3TWpVeE1qVTFNVFZhRncwek1qRXcKTWpJeE1qVTFNVFZhTUhveEN6QUpCZ05WQkFZVEFrdFNNUk13RVFZRFZRUUlEQXBIZVdWdmJtZHBMV1J2TVJRdwpFZ1lEVlFRSERBdFRaVzl1WjI1aGJTMXphVEVQTUEwR0ExVUVDZ3dHYlhsb2IyMWxNUTh3RFFZRFZRUUxEQVp0CmVYSnZiMjB4SGpBY0JnTlZCQU1NRldSaGMyaGliMkZ5WkM1MFlXVnRMbXM0Y3k1cGJ6Q0NBU0l3RFFZSktvWkkKaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLdzllNmNkZjF2c1M0MFRGVk5RUTBkZjhheTNXcTVpSmM4YwpVTUNML2w5Z2FOMU4wUUtFUGZ3MmZmOGdDelhjeGpNU0swM2Q3NmVqK282VFZZUDR0L0d4aGFHdUhHRnJ6NkRHClRUd0lzNkk1dVdYaWhIWkJ0bTJkUFdkUWRLRHQ0d1AzL2RsKzVJQnRpU1NpU2UwVEdqQzQxRXU0dlNPU1RwaEYKVWM4TXNzanh3MmE3OTJldk5jVER3ZXdOaEt0NUcwbjJMQ3ZJYmFmQnI4RjdRMnhwRU1SbENvZzhmbmpMUHZNZgpjazNHQkhBVTZYS0tzZHZtK2d3d20yWXdLd1o3V2FkWFV4TlFodVpoTXFsUHdlZUdhWXhLRmFzdndDRTM2L1BaCmpOdXVKRHFXa3pib2N2UFZlcW0vZkFlNWdyYXVhTDNIeExWQlZLNC9Yamc1VkFJVmVJa0NBd0VBQVRBTkJna3EKaGtpRzl3MEJBUXNGQUFPQ0FRRUFndy92RnVBRkJQMkdyT25LSFRvV3BCa3dNVXM4WGYwVm9OVU82bXgxL1FCaQphOWZYVEJ6Z2tvRWNORGhmc01HVjdldHNLZVZCVGZucXhxWXN0eHQ1RFMzS1h6WFIxckdGNkpJbm5tSkN5SEJUCjMyYkFPZnlDMFpkY1VnZmkxYlJNbTBPY04yc2wzVzNkaEtNdnZXd1Z4Rlg0QjZBSGhWK1hSV1c1QU4yQ1BDTlYKQk9xejBGOVhma2ZnSHpvdjlSdzlxamxOVC90NEtjd0xhR2ZneXB4RTQ3SHZDVTJweFNYQkx1cFMrbXduVTlWTgpEZ3ZmcW9IS3BiL0svZDRTbXJRbEc2dHFyOWxHbnRmeFY3UUJybkxqRUVqeUw4SThaWGJlTE0rV1oxSXBKcjJUCkVKdC8yVW9Hdlk5RDRCbFYrdFFFT0JmS0pFN3dUcXkxWnVuSm9yVk1lQT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBckQxN3B4MS9XK3hMalJNVlUxQkRSMS94ckxkYXJtSWx6eHhRd0l2K1gyQm8zVTNSCkFvUTkvRFo5L3lBTE5kekdNeElyVGQzdnA2UDZqcE5WZy9pMzhiR0ZvYTRjWVd2UG9NWk5QQWl6b2ptNVplS0UKZGtHMmJaMDlaMUIwb08zakEvZjkyWDdrZ0cySkpLSko3Uk1hTUxqVVM3aTlJNUpPbUVWUnp3eXl5UEhEWnJ2MwpaNjgxeE1QQjdBMkVxM2tiU2ZZc0s4aHRwOEd2d1h0RGJHa1F4R1VLaUR4K2VNcys4eDl5VGNZRWNCVHBjb3F4CjIrYjZERENiWmpBckJudFpwMWRURTFDRzVtRXlxVS9CNTRacGpFb1ZxeS9BSVRmcjg5bU0yNjRrT3BhVE51aHkKODlWNnFiOThCN21DdHE1b3ZjZkV0VUZVcmo5ZU9EbFVBaFY0aVFJREFRQUJBb0lCQUhpRlpScmd0eGQ1VnJ4VwpXQnUrRS9YRG12WkNMbi9MU2EyTW9LeTZ5TG13V25CUVhTb25vci95MldORjV0SS9zNmhVMUZ4ZUthM2lQaGE1CjNhTEV6T0dnV0dOejA0UVB6bTh2a3llbzV4bGl6dW9PQUtaSEFRSGVmdkxtQjFYOFgxZU5sZUUwdTJ0cU9nYWEKVUtSRk01UllJS1VEbGNWb1FQcW50c0RzbjhXZTBEMlI0dVhoQy9TRUk1TUJ5K2ZRS0xUbWRCWU9sTDhhWTUvbApFZ3ZwcWJkaDNUWlFDNkMyQ0hTZVQ2S1hpQXdEUm5SaHB6bFZkY2crbFlmSWNHZkYvTjlqY2NVVCttZ1NUZnVsClYxdUdCZHJwa2JsOWdLRUdIUEhaRDZNeHRHUkI5OWE5djg3RDUyR1lsY0laWmhaRUFnVE9zeGw2QkxjU3JtT0EKcGtJQVBRRUNnWUVBMDViWEhqcTZ1djFkRTJNTmFWSjR5VEtOcjVoWWszRW5MNlMrNTBmM1R6NW9YZHEvSFEyMApleFlZeGcwS1NoNUdSeUQ0Tm14YVYrSUVSRDFyZHRUUncyRTJsL25wK0VYSHFLeC8vQmZQb2FlTFhpSll2VWJ0Cjk4OXRoODJpd3kxdEFHUnYwcVB0Wlp0UVhXb3ErelJ6dXM0NURUS3ZzR1VXNXlPOWtTdXVDeGtDZ1lFQTBHUlUKb1BOU2RKdHU1ZzFEYWpmOGZQc25pOHAwTGcxak5URzE4NXh2WnZoZnJubG8zdmFoTnVuKzBwRFZ2K2VRMVc3NApBYjUxaEJEWll2S0VydXhJRkpCSWlkWTJMcmgzWXFHdElqSXorZkRJc2Q0MkJoWnJQSVpJUXUzbmhRNzFPMS9yCjd5cTlmNTN5cUNuSURNTi9xbS9GYk9UZW4yYVBBYkQwSjA2TDl2RUNnWUE1NU8vL2FYcG1aNlRzQlJKS1d6S0oKZXJlaDhFRnNObTNPYjNsOHR3aElPbjg4RHZwejdLZ1JkYjVaa24vYVArWmkxL2FTalpzNnFMRWFLdVFZbzZxeApsd3ZsRVpDZlNoaVRZbit5YnFGMVRlNm9WeVdJeEx1Z0xyVjlqeHFWNVB3S08zRU5aYVV6UkFmOVIydHpTS3JSCjFsTnQ5UXgxYTNPVTB3YXZqaEFWSVFLQmdRREZ3eEVWRlJUaEdFaXNCVlkrelJiTnZNTVF4SFp3NWIrS1VieXMKalg2akozNFY0NTRFU2VWQWFkdXNGRXJsTFdxalFnWVdFWnNRVTdVWlU3RmJGMXhvTjJ5L2NneEZWa1hsMGl5dAowUnJHVFIwSXZ5cGhxSkRvQlQ4NlZPOXJ0SUJCY294Q2tqcjNpdnNuWDA4NzNhT2dLU1lnYXlwaDkwQXJpTFNMClFOMU80UUtCZ0F5TVJraGdkVFZFSU9Xc1dGc1YzRmxCdEh1RHh1clV1a3Vua0IxRWdTeDJDSUZKL21CMUtUY2sKbUpSNy9DaTZNSmpvZlkrVmdUellFd3N0cmt4amZkZVdEQ1JvVE4wSmk2ZzZ5YlB5Q0RoeGxCa0JHS2ZZVHFRZwpGWjFIdG56Q2NqOVZTd0U0cVA0VW1IQkc2bVFGYmxVSCtNWXdqNjNyK3o4VytzaG1lcWtGCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==**
kind: Secret
metadata:
  creationTimestamp: "2022-10-25T13:18:23Z"
  name: kubernetes-dashboard-ing-cert
  namespace: kubernetes-dashboard
  resourceVersion: "274213"
  uid: 33a76548-f438-4727-9e02-1226b902b0f7
**type: kubernetes.io/tls**
```

## 02. ingress에 tls 설정

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ing
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  **tls:
  - hosts:
    - dashboard.taem.k8s.io
    secretName: kubernetes-dashboard-ing-cert**
  rules:
  - host: dashboard.taem.k8s.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 9090
```

기존과 동일한 양식에 .spec.tls 항목만 추가한다.

```json
**# kubectl apply -f dashboard-ing.yaml** 
ingress.networking.k8s.io/kubernetes-dashboard-ing created

# kubectl -n kubernetes-dashboard get ing
NAME                       CLASS   HOSTS                   ADDRESS   PORTS     AGE
kubernetes-dashboard-ing   nginx   dashboard.taem.k8s.io             **80, 443**   5s
```

## 03. 접속 확인

ingress의 external-ip를 확인한다.

```json
# kubectl -n kubernetes-dashboard get ing
NAME                       CLASS   HOSTS                   ADDRESS         PORTS     AGE
kubernetes-dashboard-ing   nginx   **dashboard.taem.k8s.io   192.168.72.12**   80, 443   13m
```

해당 ip와 host를  윈도우의 hosts 파일에 등록한다 경로는 C:\Windows\System32\dirvers\etc\hosts 다.

```json
192.168.72.12 dashboard.taem.k8s.io
```

본 테스트 환경은 nginx ingress를 활용한다. ingress controller의 nodePort를 확인한다.

```json
**# kubectl -n ingress-nginx get svc**
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.108.186.202   <none>        80:32057/TCP,**443:31369/TCP**   22d
ingress-nginx-controller-admission   ClusterIP   10.108.31.255    <none>        443/TCP                      22d
```

https://dashboard.taem.k8s.io:31369로 접속해보자

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4b42fa83-d7e7-4c56-a731-bc6f06de1c3e/Untitled.png)

잘 접속된다.
