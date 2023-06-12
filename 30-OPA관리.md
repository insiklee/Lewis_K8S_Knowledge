# K8s OPA 관리

[Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

[OPA Gatekeeper: Policy and Governance for Kubernetes](https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes/)

[Introduction](https://www.openpolicyagent.org/docs/latest/#running-opa)

[Overview & Architecture](https://www.openpolicyagent.org/docs/latest/kubernetes-introduction/)

# 01. OPA란?

## 01. 개요

OPA는 Open Policy Agent의 줄임말로 쿠버네티스 클러스터 환경의 거버넌스와 컴필라이언스를 강화하기 위한 하나의 방법이다.

## 02. OPA Gatekeeper의 등장 배경

클러스터를 사용하는 조직이 확장하면서 더욱 복잡한 컴필라이언스 구현이 필요해졌다. 다만 과도한 내부 통제는 신속한 개발 환경에 악영향을 줄 수 있다는 점 역시 고려 해야만 했다. 

특히 아래와 리소스에 대한 정책 강화 수요가 늘어나고 있었다.

- 승인된 레포지토리로 부터 받은 컨테이너 이미지만 사용할 것
- 모든 ingress의 호스트네임은 유일할 것
- 모든 파드에 limits가 걸려 있을 것
- 모든 네임스페이스에 접점이 되는 라벨 리스트를 갖게 할것

## 03. Gatekeeper의 특징

Gatekeeper는 아래와 같은 특징을 지니고 있다.

- admission controller webhook을 통해 API 서버로 전달되는 요청에 대해 승인여부를 결정한다.
- config 파일을 통해서 admission control을 커스텀할 수 있음

## 04. GateKeeper 동작 원리

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9e97d610-9359-4d33-b048-c06c58818765/Untitled.png)

# 02. Gatekeeper 설치

## 01. 설치

아래 명령어를 사용한다.

<aside>
💡 # **kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml**

</aside>

```json
**# kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml**
namespace/gatekeeper-system created
resourcequota/gatekeeper-critical-pods created
customresourcedefinition.apiextensions.k8s.io/assign.mutations.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/assignmetadata.mutations.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/configs.config.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constraintpodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constrainttemplatepodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constrainttemplates.templates.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/expansiontemplate.expansion.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/modifyset.mutations.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/mutatorpodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/providers.externaldata.gatekeeper.sh created
serviceaccount/gatekeeper-admin created
role.rbac.authorization.k8s.io/gatekeeper-manager-role created
clusterrole.rbac.authorization.k8s.io/gatekeeper-manager-role created
rolebinding.rbac.authorization.k8s.io/gatekeeper-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/gatekeeper-manager-rolebinding created
secret/gatekeeper-webhook-server-cert created
service/gatekeeper-webhook-service created
deployment.apps/gatekeeper-audit created
deployment.apps/gatekeeper-controller-manager created
poddisruptionbudget.policy/gatekeeper-controller-manager created
mutatingwebhookconfiguration.admissionregistration.k8s.io/gatekeeper-mutating-webhook-configuration created
validatingwebhookconfiguration.admissionregistration.k8s.io/gatekeeper-validating-webhook-configuration created
```

## 02. 설치 확인

### 01. crd 생성 확인

```json
**# kubectl get crd | grep gatekeeper**
assign.mutations.gatekeeper.sh                        2022-10-25T05:57:03Z
assignmetadata.mutations.gatekeeper.sh                2022-10-25T05:57:03Z
configs.config.gatekeeper.sh                          2022-10-25T05:57:03Z
constraintpodstatuses.status.gatekeeper.sh            2022-10-25T05:57:03Z
constrainttemplatepodstatuses.status.gatekeeper.sh    2022-10-25T05:57:03Z
constrainttemplates.templates.gatekeeper.sh           2022-10-25T05:57:03Z
expansiontemplate.expansion.gatekeeper.sh             2022-10-25T05:57:03Z
modifyset.mutations.gatekeeper.sh                     2022-10-25T05:57:03Z
mutatorpodstatuses.status.gatekeeper.sh               2022-10-25T05:57:03Z
providers.externaldata.gatekeeper.sh                  2022-10-25T05:57:04Z
```

### 02. 네임스페이스, 서비스, 파드 생성 확인

```json
**# kubectl -n gatekeeper-system get po,svc,deploy**
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/gatekeeper-audit-56ddcd8749-nspsg                1/1     Running   0          17m
pod/gatekeeper-controller-manager-64fd6c8cfd-6nnqj   1/1     Running   0          17m
pod/gatekeeper-controller-manager-64fd6c8cfd-6w47j   1/1     Running   0          17m
pod/gatekeeper-controller-manager-64fd6c8cfd-zl9tp   1/1     Running   0          17m

NAME                                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/gatekeeper-webhook-service   ClusterIP   10.103.89.98   <none>        443/TCP   17m

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gatekeeper-audit                1/1     1            1           17m
deployment.apps/gatekeeper-controller-manager   3/3     3            3           17m
```

# 03. Gatekeeper 활용

## 01. constraint Template

[](https://open-policy-agent.github.io/gatekeeper/website/docs/howto)

### 01. 기본 양식

제약사항을 정의하기 전에 생성해야하는 템플릿이다. 제약사항의 개요(schema)나 제약사항을 강화하는 Rego를 정의한다.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
```

### 01. .spec.crd

어떤 제약사항을 정의할 것이냐에 따라서 필요로 하는 정보가 달라지는 경우가 많다.

만약 디플로이에 특정 라벨이 추가되어야 하는 경우, 생성되는 디플로이의 라벨 정보와 관련된 내용을 수집해야 할 것이다.

또는 파드에서 특정 포트를 오픈할 수 없도록 막기 위해서는 파드 컨테이너의 포트 정보를 수집해야 할 것이다.

워낙 다양한 제약 방식이 있고 그에 따라 요구하는 정보 역시 천차만별이기 때문에 템플릿에서는 CRD를 직접 정의해서 필요로 하는 정보들의 스키마를 정의한다.

```yaml
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
```

위의 예시로 든 crd 정의 방식은 우리가 평소에 CRD를 생성하는 방식과 동일하다. openAPIV3shcema를 통해 우리가 리소스에 포함될 파라미터들을 정의한다. propertes에는 container port가 들어올 수 있고, deployment의 label이 들어올 수 있다.

위의 예시는 타겟이 되는 object에 대해서 label 정보를 수집하겠다는 스키마 정의내용이다.

### 02. .spec.targets.rego

.spec.crd에 정의된 내용대로 constraint를 생성했다면, 이제 OPA의 policy중 하나가 적용된 것이다. 그런데 이 constraint에 정의된 내용과 위반되는 API 요청이 발생한다면, 그때부터 templateConstraint에서 정의한 rego가 일을 시작한다.

r**ego는 rego 언어를 사용하며**, crd에서 정의한 특정 kind의 constraint에 대해서 무엇이 위반사항인지를 정의하고, 어떤 메시지를 전달할 것인지를 정의한다. 이 부분을 정의하고자 한다면 go 언어는 필수로 익혀야 할 것이다.

```yaml
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
```

위의 예시를 보자. package는 crd에서 정의한 리소스의 kind 이름을 적는다. 

그 후 violation 항목에서 **go 언어 형식**으로 constraint 위반 기준을 정의한다.

이 때 msg 내용을 정의해서 사용자가 constraint를 위반했을 때 출력할 에러 메시지를 정의할 수 있다.

## 02. constraint

contraint는 constraintTemplate에서 생성한 crd로 생성한다.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-hr
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["hr"]
```

crd에서 kind 명을 k8sRequiredLabels로 지정했기 때문에 해당 constraint의 kind도 동일하게 설정한다.

## 03. Constraint 수행

[01. constraint Template](https://www.notion.so/01-constraint-Template-ab095c5907a74f69b3cc168609641580?pvs=21) 와 [02. constraint](https://www.notion.so/02-constraint-3a4241a22b7f4c778a59ec76b11a85ab?pvs=21) 에서 생성한 리소스를 등록한다.

```json
**# kubectl apply -f template.yaml** 
constrainttemplate.templates.gatekeeper.sh/k8srequiredlabels created

**# kubectl get constrainttemplate**
NAME                AGE
**k8srequiredlabels**   7s

**# kubectl apply -f crd.yaml** 
k8srequiredlabels.constraints.gatekeeper.sh/ns-must-have-hr created

**# kubectl get constraint**
NAME              ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
**ns-must-have-hr**
```

이후 위 제약사항에 따라 네임스페이스를 생성해본다.

제약사항에 따르면 라벨에 hr이 포함되지 않으면 네임스페이스 생성이 불가능하게 되어 있다.

우선 아무런 라벨 없이 네임스페이스를 실행해본다.

```json
**# kubectl create ns test-opa**
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: **[ns-must-have-hr] you must provide labels: {"hr"}**
```

라벨에 hr이 없어서 생성할 수 없다는 메시지가 발생한다.

constraint의 status.violations에서 모든 위반사항에 대한 정보가 나온다.

물론 label에 hr을 포함하면 정상적으로 생성된다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-opa
  **labels:
    hr: "1"**
spec: {}
status: {}
```

```json
**# kubectl apply -f ns.yaml** 
namespace/test-opa created
```

## 04. Auditing

OPA는 API Server에서 저장해놓은 모든 리소스의 정보들을 복제한 뒤, 자신의 constraint에 위반되는 내용이 있는지 스스로 감사한다. 그 감사의 결과는 constraint를 정의한 리소스의 status 항목에서 확인할 수 있다.

```json
**# kubectl get constraint ns-must-have-hr -o jsonpath='{.status.violations}' | jq**
[
  {
    "enforcementAction": "deny",
    "group": "",
    "kind": "Namespace",
    "message": "you must provide labels: {\"hr\"}",
    "name": "gitlab-ns",
    "version": "v1"
  },
  {
    "enforcementAction": "deny",
    "group": "",
    "kind": "Namespace",
    "message": "you must provide labels: {\"hr\"}",
    "name": "kube-node-lease",
    "version": "v1"
  },
  {
    "enforcementAction": "deny",
    "group": "",
    "kind": "Namespace",
    "message": "you must provide labels: {\"hr\"}",
    "name": "ingress-nginx",
    "version": "v1"
  },
(하략)
```

# 04. rego 문법

[Policy Language](https://www.openpolicyagent.org/docs/latest/policy-language/)

시간 되면 위 링크에 들어가서 익숙해져 보자.
