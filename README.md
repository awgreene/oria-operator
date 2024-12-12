## ⚠️ Deprecation Notice

> This project is **deprecated** and is no longer actively maintained. 

# oria-operator

## Summary

The Oria Operator provides tooling that allows cluster admins and operator authors to control which namespaces an operator reconciles resource events in.

The `oria-operator` will introduce two cluster scoped CRDs, the `ScopeTemplate` and `ScopeInstance`.

### ScopeTemplate CRD

The `ScopeTemplate` CRD is used to define the RBAC needed by an operator. It basically allows one to define:

- A `ClusterRole`

An example of a `ScopeTemplate` CR can be seen below:

```
apiVersion: operators.io.operator-framework/v1
kind: ScopeTemplate
metadata:
  name: scopetemplate-sample
spec:
  clusterRoles:
  - generateName: test
    rules:
    - apiGroups: [""]
      # at the HTTP level, the name of the resource for accessing Secret
      # objects is "secrets"
      resources: ["secrets"]
      verbs: ["get", "watch", "list"]
    subjects:
    - kind: Group
      name: manager  # Name is case sensitive
      apiGroup: rbac.authorization.k8s.io
```

The reconciliation process will verify the below steps:
1. It will check if any `ScopeInstance` CRs reference to `ScopeTemplate` name or not.
2. If it is referencing then the `ClusterRole` defined in the `ScopeTemplate` will be created if it does not exist. The created `ClusterRole` will include an owner reference to the `ScopeTemplate` CR.
3. If no `ScopeInstance` references the `ScopeTemplate`, the `ClusterRole` defined in the `ScopeTemplate` will be deleted if it exists.


### ScopeInstance CRD

The `ScopeInstance` CRD is used to define a list of `namespaces` that the RBAC in a `ScopeTemplate` will be created in. A cluster admin will create the `ScopeInstance` CR and will specify:

- The name of a `ScopeTemplate` which defines the RBAC required by the operator
- A set of namespaces that the operator should be scoped to. An empty set of namespaces is equivalent to specifying all namespaces.

```
apiVersion: operators.io.operator-framework/v1
kind: ScopeInstance
metadata:
  name: scopeinstance-sample
spec:
  scopeTemplateName: scopetemplate-sample
  namespaces:
  - default
```

The reconciliation process will verify the below steps:

1. It will look for `ScopeTemplate` that `ScopeInstance` is referencing. if it is not referencing then throw an error with the appropriate message.
2. If it is referencing and if the `namespaces` array is empty, a single `ClusterRoleBinding` will be created. Otherwise, a `RoleBinding` will be created in each of the `namespaces`. These resources will include an owner reference to the `ScopeInstance` CR.

## Installation
To install the latest release of `oria-operator`, run:
```
kubectl apply -f https://github.com/operator-framework/oria-operator/releases/latest/download/oria-operator.yaml
```

To install a specific release of `oria-operator`, run: 
```
ORIA_VERSION=vX.Y.Z; kubectl apply -f https://github.com/operator-framework/oria-operator/releases/download/$ORIA_VERSION/oria-operator.yaml
```

## Run the Operator Locally

### 1. Run locally outside the cluster 

First, install newly created `ScopeInstance` and `ScopeTemplate` CRs

```
make install
```

It will create CRDs and log the below message

```
customresourcedefinition.apiextensions.k8s.io/scopeinstances.operators.io.operator-framework created
customresourcedefinition.apiextensions.k8s.io/scopetemplates.operators.io.operator-framework created
```

Then, run the `oria-operator` with below command and apply `ScopeInstance` and `ScopeTemplate` CRDs

```
make run
```

Apply `ScopeTemplate` CRD as below. This will create a `ScopeTemplate` with the name `scopetemplate-sample`.

```
$ kubectl apply -f config/samples/operators_v1_scopetemplate.yaml
scopetemplate.operators.io.operator-framework/scopetemplate-sample created
```

Now, create `ScopeInstance` CRD as below. This will create a `ScopeInstance` with the name `scopeinstance-sample` that references the `ScopeTemplate` created in the previous step.

```
$ kubectl apply -f config/samples/operators_v1_scopeinstance.yaml
scopeinstance.operators.io.operator-framework/scopeinstance-sample created
```

Once `scopeinstance-sample` is created, it will trigger the reconciliation process of `ScopeTemplate` and `ScopeInstance` controllers.

`ScopeInstance` reconciliation process will create `(Cluster)RoleBinding` as defined in CR.

```
$ kubectl get clusterroles
NAME   CREATED AT
test   2022-09-20T18:39:32Z
```

`ScopeInstance` reconciliation process will create `(Cluster)RoleBinding`s as defined in CRD.

```
$ kubectl get rolebindings --all-namespaces
NAMESPACE   NAME         ROLE               AGE
default     test-x8hdc   ClusterRole/test   33m
```

Now, let's update the `ScopeInstance` with a new namespace. Create the `test` namespace with:

```
$ kubectl create namespace test
namespace/test created
```

Update the `ScopeInstance` to look similar to:

```
apiVersion: operators.io.operator-framework/v1
kind: ScopeInstance
metadata:
  name: scopeinstance-sample
spec:
  scopeTemplateName: scopetemplate-sample
  namespaces:
  - default
  - test
```

```
$ kubectl apply -f config/samples/operators_v1_scopeinstance.yaml
scopeinstance.operators.io.operator-framework/scopeinstance-sample configured
```

Now, verify that there is a `RoleBinding` created in both namespaces:

```
$ kubectl get rolebindings --all-namespaces
NAMESPACE   NAME         ROLE               AGE
default     test-x8hdc   ClusterRole/test   37m
test        test-64hk7   ClusterRole/test   80s
```

Now, update the `ScopeInstance` and remove the `default` namespace from it:

```
apiVersion: operators.io.operator-framework/v1
kind: ScopeInstance
metadata:
  name: scopeinstance-sample
spec:
  scopeTemplateName: scopetemplate-sample
  namespaces:
  - test
```

```
$ kubectl apply -f config/samples/operators_v1_scopeinstance.yaml
scopeinstance.operators.io.operator-framework/scopeinstance-sample configured
```

Verify that the `RoleBinding` in the `default` namespace is removed but the `RoleBinding` in the `test` namespace still exists:

```
NAMESPACE   NAME         ROLE               AGE
test        test-64hk7   ClusterRole/test   2m45s
```

In the end, remove all namespaces from the `ScopeInstance` and verify that it creates a `ClusterRoleBinding` and removes any associated `RoleBinding`s:

```
apiVersion: operators.io.operator-framework/v1
kind: ScopeInstance
metadata:
  name: scopeinstance-sample
spec:
  scopeTemplateName: scopetemplate-sample
```

```
$ kubectl apply -f config/samples/operators_v1_scopeinstance.yaml
scopeinstance.operators.io.operator-framework/scopeinstance-sample configured
```

Verify `RoleBinding`s are removed:

```
$ kubectl get rolebindings --all-namespaces 
No resources found
```

Verify `ClusterRoleBinding` is created:

```
$ kubectl get clusterrolebindings
NAME         ROLE               AGE
test-mskl2   ClusterRole/test   50s
```

## How to contribute

For contributing guidelines, see the [CONTRIBUTING.md][contributing-file] file.

## License

Oria Operator is under Apache 2.0 license. See the [LICENSE][license_file] file for details.

[controller-runtime]: https://github.com/kubernetes-sigs/controller-runtime
[license_file]:./LICENSE
[of-home]: https://github.com/operator-framework
[of-blog]: https://www.openshift.com/blog/introducing-the-operator-framework
[operator-link]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
[sdk-docs]: https://sdk.operatorframework.io
[operator-framework-community]: https://github.com/operator-framework/community
[operator-framework-communication]: https://github.com/operator-framework/community#get-involved
[operator-framework-meetings]: https://github.com/operator-framework/community#meetings
[contributing-file]:./CONTRIBUTING.md
