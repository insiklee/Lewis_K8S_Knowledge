# K8s NetworkPolicy 관리

[네트워크 정책](https://kubernetes.io/ko/docs/concepts/services-networking/network-policies/)

# 01. NetworkPolicy란?

파드나 네임스페이스의 네트워크 통신을 통제하는 리소스다. 특정 네트워크 대역대가 특정 네임스페이스로 접근할 수 없도록 막거나, 특정 파드가 어떤 네임스페이스로 패킷 전송을 못하게 막을 수 있다.

## 01. Ingress와 Egress

우리가 흔히 생각하는 Ingress 리소스와 Egress 리소스와는 다른 개념이다. 방화벽에서 Inbound 또는 Outbound를 다루는 정책에 가깝다.  특정 파드를 기준으로 파드를 향해 들어오는 네트워크 패킷을 Ingress 정책으로 관리하고, 파드에서 밖으로 나가는 네트워크 패킷을 Egress 정책으로 관리한다.

만약 NetworkPolicy를 통해서 ingress 정책이 존재할 경우, 해당 파드는 ingress에 정의된 접근만 허용된다.  반대로 egress 정책이 존재할 경우엔 egress에 정의된 전송만 허용된다.

# 02. NetworkPolicy 리소스

## 01. Ingress

### 01. Ingress 기본 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
          endPort: 6500
```

위 YAML에 대해 설명을 해보자.

해당 NetworkPolicy는 default 네임스페이스에서 동작한다. 따라서 default 네임스페이스에서 실행된 Pod에 대해서 네트워크 접근통제를 실시한다.

.spec.podSelector는 NetworkPolicy가 실해된 네임스페이스에서 어떤 Pod에 적용할지 선택한다. Label을 통해서 해당되는 Pod를 선택하며, 아무런 Pod가 선택되지 않을 경우 해당 네임스페이스에 존재하는 모든 Pod에 대해서 접근이 통제된다.

policyTypes에 정의된대로 ingress 정책을 설정한다. ingress는 외부에서의 접근을 통제하기 때문에 from 셀렉터를 사용한다. ipBlock은 172,17.0.0/16 대역대의 접근을 허용ㅎ되, 172.17.1.0/24 대역대는 허용하지 않는다. project: myproject label을 가지고 있는 네임스페이스에서만 접근이 가능하며, 그 중에서 role: frontend label을 갖고 있는 Pod만 접근이 가능하다.

위의 조건에 해당되는 Source에 대해서 6379~6500/TCP로 접근을 허용한다. 

### 02. Ingress 모든 접근 차단

```yaml
apiVerion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyType:
  - Ingress
```

아무런 Pod가 선택되지 않은데다가 아무런 ingress 정책이 설정되지 않았다. 이러면 모든 접근이 차단된다.

### 03. Ingress 모든 접근 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-ingress
spec:
  podSelector: {}
  policyType:
  - Ingress
  ingress:
  - {}
```

위 모든 접근 차단과 다른 점은 .spec.ingress 하나가 추가됐단 점이다. 일단 ingress 정책을 선언했는데 딱히 selector를 통해 정의된 내용이 없으니 전부 다 허용하겠다는 뜻이다. 

## 02. Egress

### 01. Egress 기본 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
          endPort: 6000
```

위 YAML에 대해 설명을 해보자.

해당 NetworkPolicy는 default 네임스페이스에서 동작한다. 따라서 default 네임스페이스에서 실행된 Pod에 대해서 네트워크 접근통제를 실시한다.

.spec.podSelector는 NetworkPolicy가 실해된 네임스페이스에서 어떤 Pod에 적용할지 선택한다. Label을 통해서 해당되는 Pod를 선택하며, 아무런 Pod가 선택되지 않을 경우 해당 네임스페이스에 존재하는 모든 Pod에 대해서 접근이 통제된다.

policyTypes에 정의된대로 egress 정책을 설정한다. egress 정책은 외부로 전송할 네트워크를 통제하기 때문에 **to** 셀렉터를 사용한다. ipblock을 통해서 10.0.0.0/24 대역대의 모든 파드로 전송을 허용하며, 연결을 허용할 포트는 5978~6000/TCP다. 

### 02. Egress 모든 전송 차단

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyType:
  - Egress
```

아무런 Pod가 선택되지 않은데다가 아무런 egress 정책이 설정되지 않았다. 이러면 모든 전송이 차단된다.

### 03. Egress 모든 전송 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-egress
spec:
  podSelector: {}
  policyType:
  - Egress
  egress:
  - {}
```

위 모든 전송 차단과 다른 점은 .spec.egress 하나가 추가됐단 점이다. 일단 egress 정책을 선언했는데 딱히 selector를 통해 정의된 내용이 없으니 전부 다 허용하겠다는 뜻이다.
