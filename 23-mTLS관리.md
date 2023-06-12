# K8s mTLS 관리

# mTLS란?

mTLS는 mutual Transport Layer Security의 약자다.

## 필요성

POD 사이에 통신이 오갈때 평문 통신하기 때문에 보안에 취약할 수 있다. 이 문제를 해결하기 위해서 파드에 TLS 암호 인증을 진행하는 proxy sidcar container를 생성한다.

# mTLS 사용

Istio와 같은 서비스메시를 구축하면 파드간 통신에 TLS를 사용할 수 있다.

자세한 내용은 차차 정리해보겠다.
