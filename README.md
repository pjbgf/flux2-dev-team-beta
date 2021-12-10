# dev-team-beta

This repo serves as an example of how to structure a tenant Git repository when leveraging [Hierarchical Namespaces].

The [Platform Admin] already [onboared the tenant] assigning it the namespace `dev-team-beta-ns` and the service account `system:serviceaccount:dev-team-beta-ns:flux`.

The hnc multi-tenant example configures this repo as a [gitrepository for a tenant here].

### Structure Caveats
Some structural decisions were intentional and are highly recommended:

- All flux objects are deployed at the top-level namespace, as it enables:
    a) Separation of duties: the tenant's top namespace is solely responsible for its flux's configuration.
    b) RBAC isolation from applications: the flux `ServiceAccount` used to apply changes is not propagated to subnamespaces. Therefore removing the risk of it being used by users with `pod create` permissions in subnamespaces.

- Tenants must always split the creation and use of `SubNamespace` objects in different `Kustomization` objects. This example creates `dev-team-beta-subnamespaces` to create all its subnamespaces, and `dev-team-beta-resources` to create everything else. This is required as kustomize is unaware that [Hierarchical Namespaces] will propagate the RBAC permissions to the subnamespaces automatically, therefore attempting to do both operations in a single `Kustomization` will fail during the `dry-run` due to "lack of permissions".


### Inspection of deployed repo

#### Namespaces

Once reconciled, three namespaces should be available:

```shell
$ kubectl get namespace dev-team-beta-ns apps some-other-ns
NAME                STATUS   AGE
dev-team-beta-ns   Active   56m
redis              Active   55m
```

Using the [hnc plugin] they should structured as a tree:
```
$ kubectl hns tree dev-team-beta-ns
dev-team-beta-ns
├── [s] redis
```

#### Permissions

The `ServiceAccount` and `RoleBinding` defined by the Platform Admin is created at the top level:

```shell
$ kubectl get rolebinding,serviceaccount --namespace dev-team-beta-ns
NAME                                                              ROLE                        AGE
rolebinding.rbac.authorization.k8s.io/dev-team-beta-reconciler   ClusterRole/cluster-admin   34m

NAME                     SECRETS   AGE
serviceaccount/default   1         34m
serviceaccount/flux      1         34m
```

Only the `RoleBinding` is propagated downstream:
```shell
$ kubectl get rolebinding,serviceaccount --namespace apps
NAME                                                              ROLE                        AGE
rolebinding.rbac.authorization.k8s.io/dev-team-beta-reconciler   ClusterRole/cluster-admin   34m

NAME                     SECRETS   AGE
serviceaccount/default   1         34m


$ kubectl get rolebinding,serviceaccount --namespace some-other-ns
NAME                                                              ROLE                        AGE
rolebinding.rbac.authorization.k8s.io/dev-team-beta-reconciler   ClusterRole/cluster-admin   34m

NAME                     SECRETS   AGE
serviceaccount/default   1         34m
```

#### Applications

```shell
$ kubectl get pod,svc --namespace redis
NAME                 READY   STATUS    RESTARTS   AGE
pod/redis-master-0   1/1     Running   0          97m

NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/redis-headless   ClusterIP   None           <none>        6379/TCP   97m
service/redis-master     ClusterIP   10.96.49.242   <none>        6379/TCP   97m
```

```shell
$ kubectl get HelmRelease -A
NAMESPACE           NAME      READY   STATUS                             AGE
dev-team-alpha-ns   podinfo   True    Release reconciliation succeeded   65m
```

For more information, refer to the [hnc multi-tenant repo].

[Hierarchical Namespaces]: https://github.com/kubernetes-sigs/hierarchical-namespaces
[Platform Admin]: ../flux2-hnc-multi-tenancy#roles
[hnc plugin]: https://github.com/kubernetes-sigs/hierarchical-namespaces/releases
[onboared the tenant]: ../flux2-hnc-multi-tenancy/tenants/base/dev-team-beta
[gitrepository for a tenant here]: ../flux2-hnc-multi-tenancy/blob/main/tenants/base/dev-team-beta/sync.yaml#L9
[hnc multi-tenant repo]: ../flux2-hnc-multi-tenancy
