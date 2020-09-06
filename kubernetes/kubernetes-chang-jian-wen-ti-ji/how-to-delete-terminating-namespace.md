---
description: 如何删除 terminating 的 namespace
---

# how to delete terminating namespace?

```text
$ kubectl get namespace naftis
NAME     STATUS        AGE
naftis   Terminating   82m

$ kubectl get namespace naftis  -o json > naftis.yaml

# remove the value “kubernetes” from spec.finalizers[]
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{},\"name\":\"naftis\"}}\n"
        },
        ...
    },
    "spec": {
        "finalizers": [
            "kubernetes"
        ]
    },
    "status": {
        "conditions": [
          ...
        ],
        "phase": "Terminating"
    }
}

# as

{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{},\"name\":\"naftis\"}}\n"
        },
        ...
    },
    "spec": {
        "finalizers": [
        ]
    },
    "status": {
        "conditions": [
          ...
        ],
        "phase": "Terminating"
    }
}


# replace the namespace file, kubectl replace —raw "/api/v1/namespace/<your_namespace>/finalize” -f ./xx.yaml
kubectl replace --raw "/api/v1/namespaces/naftis/finalize" -f ./naftis.yaml

```

