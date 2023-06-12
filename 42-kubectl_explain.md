# kubectl explain

특정 리소스의 yaml이나 json 형식을 확인하고자 할때 사용한다.

<aside>
💡 **# kubectl explain {리소스 종류} --recursive**

</aside>

- --recursive : 리소스 spec의 세부 항목에 대한 설명을 출력하는 옵션이다. 해당 옵션을 주지 않으면 apiversion, kind, metadata, spec, status에 대한 기본적인 설명만 출력된다.

```json
**# kubectl explain networkpolicy --recursive**
KIND:     NetworkPolicy
VERSION:  networking.k8s.io/v1

DESCRIPTION:
     NetworkPolicy describes what network traffic is allowed for a set of Pods

FIELDS:
   apiVersion	<string>
   kind	<string>
   metadata	<Object>
      annotations	<map[string]string>
      creationTimestamp	<string>
      deletionGracePeriodSeconds	<integer>
      deletionTimestamp	<string>
      finalizers	<[]string>
      generateName	<string>
      generation	<integer>
      labels	<map[string]string>
      managedFields	<[]Object>
         apiVersion	<string>
         fieldsType	<string>
         fieldsV1	<map[string]>
         manager	<string>
         operation	<string>
         subresource	<string>
         time	<string>
      name	<string>
      namespace	<string>
      ownerReferences	<[]Object>
         apiVersion	<string>
         blockOwnerDeletion	<boolean>
         controller	<boolean>
         kind	<string>
         name	<string>
         uid	<string>
      resourceVersion	<string>
      selfLink	<string>
      uid	<string>
   spec	<Object>
      egress	<[]Object>
         ports	<[]Object>
            endPort	<integer>
            port	<string>
            protocol	<string>
         to	<[]Object>
            ipBlock	<Object>
               cidr	<string>
               except	<[]string>
            namespaceSelector	<Object>
               matchExpressions	<[]Object>
                  key	<string>
                  operator	<string>
                  values	<[]string>
               matchLabels	<map[string]string>
            podSelector	<Object>
               matchExpressions	<[]Object>
                  key	<string>
                  operator	<string>
                  values	<[]string>
               matchLabels	<map[string]string>
      ingress	<[]Object>
         from	<[]Object>
            ipBlock	<Object>
               cidr	<string>
               except	<[]string>
            namespaceSelector	<Object>
               matchExpressions	<[]Object>
                  key	<string>
                  operator	<string>
                  values	<[]string>
               matchLabels	<map[string]string>
            podSelector	<Object>
               matchExpressions	<[]Object>
                  key	<string>
                  operator	<string>
                  values	<[]string>
               matchLabels	<map[string]string>
         ports	<[]Object>
            endPort	<integer>
            port	<string>
            protocol	<string>
      podSelector	<Object>
         matchExpressions	<[]Object>
            key	<string>
            operator	<string>
            values	<[]string>
         matchLabels	<map[string]string>
      policyTypes	<[]string>
   status	<Object>
      conditions	<[]Object>
         lastTransitionTime	<string>
         message	<string>
         observedGeneration	<integer>
         reason	<string>
         status	<string>
         type	<string>
```
