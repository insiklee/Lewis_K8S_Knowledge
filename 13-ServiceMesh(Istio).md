# Istio

# 1. Istio 설치

1. 필수 패키지 다운로드

<aside>
💡 [root@matser ~]# cd /
[root@master /]# curl -L https://istio.io/downloadIstio | sh -

% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
Dload  Upload   Total   Spent    Left  Speed
100   102  100   102    0     0    102      0  0:00:01 --:--:--  0:00:01   102
100  4549  100  4549    0     0   3781      0  0:00:01  0:00:01 --:--:--  3781

Downloading istio-1.11.3 from https://github.com/istio/istio/releases/download/1.11.3/istio-1.11.3-linux-amd64.tar.gz ...

Istio 1.11.3 Download Complete!

Istio has been successfully downloaded into the istio-1.11.3 folder on your system.

Next Steps:
See https://istio.io/latest/docs/setup/install/ to add Istio to your Kubernetes cluster.

To configure the istioctl client tool for your workstation,
add the /root/istio-1.11.3/bin directory to your environment path variable with:
export PATH="$PATH:/root/istio-1.11.3/bin"

Begin the Istio pre-installation check by running:
istioctl x precheck

Need more information? Visit https://istio.io/latest/docs/setup/install/

</aside>

1. 환경변수 추가

<aside>
💡 [root@master /]# echo "export PATH=\"$PATH:/istio-1.11.3/bin\"" >> /etc/profile

[root@master /]# source /etc/profile

</aside>

1. Istio 설치

<aside>
💡 [root@master /]# cd istio-1.11.3/
[root@master istio-1.11.3]# istioctl install --set profile=demo
This will install the Istio 1.11.3 default profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete
Thank you for installing Istio 1.11.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/kWULBRjUv7hHci7T6

</aside>

1. 파드 설치 확인

<aside>
💡 [root@master istio-1.11.3]# kubectl get pods -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istio-ingressgateway-7776dd578-4n8wg   1/1     Running   0          79s
istiod-56c76db8bc-wcb8z                1/1     Running   0          97s

</aside>

# 2. Istio Dynamic request 설정

1. bookinfo 샘플 배포

<aside>
💡 [root@master istio-1.11.3]# kubectl apply -f samples/^C
[root@master istio-1.11.3]# cd /istio-1.11.3/
[root@master istio-1.11.3]# kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
[virtualservice.networking.istio.io/productpage](http://virtualservice.networking.istio.io/productpage) created
[virtualservice.networking.istio.io/reviews](http://virtualservice.networking.istio.io/reviews) created
[virtualservice.networking.istio.io/ratings](http://virtualservice.networking.istio.io/ratings) created
[virtualservice.networking.istio.io/details](http://virtualservice.networking.istio.io/details) created

</aside>
