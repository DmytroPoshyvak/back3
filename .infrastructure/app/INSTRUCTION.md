# Validation instructions

This change gives the ToDo app pod permission to list Secrets in its own namespace:

- `security/rbac.yml` defines the ServiceAccount `secrets-reader`, the Role `role`
  (allows `get` and `list` on `secrets` in the namespace `todoapp`) and the RoleBinding `role`
  that binds the Role to the ServiceAccount;
- the Deployment `todoapp` runs its pods with `serviceAccountName: secrets-reader`.

Run all commands from the repository root.

## 0. Prerequisites

`docker`, `kind` and `kubectl` are installed and Docker is running.

## 1. Create the cluster

```bash
kind create cluster --config cluster.yml
kubectl get nodes
```

Expected: all nodes are `Ready`.

## 2. Deploy

```bash
chmod +x bootstrap.sh
./bootstrap.sh
```

`bootstrap.sh` applies `security/rbac.yml` before the Deployment. If your `bootstrap.sh`
does not contain it, apply it manually (the namespace `todoapp` must already exist):

```bash
kubectl apply -f security/rbac.yml
```

Wait until the app is ready:

```bash
kubectl rollout status deployment/todoapp -n todoapp --timeout=180s
kubectl get pods -n todoapp
```

Expected: the `todoapp` pods are `1/1 Running`.

If the Deployment was applied before the ServiceAccount, its pods appear only after the
ServiceAccount is created (events say `serviceaccount "secrets-reader" not found`). This
resolves itself once `security/rbac.yml` is applied.

## 3. Check the RBAC objects

```bash
kubectl get serviceaccount,role,rolebinding -n todoapp
kubectl describe role role -n todoapp
kubectl describe rolebinding role -n todoapp
```

Expected:

- ServiceAccount `secrets-reader`, Role `role` and RoleBinding `role` exist in `todoapp`;
- the Role has one rule: resource `secrets`, verbs `[get list]`;
- the RoleBinding refers to `Role/role` and has the subject
  `ServiceAccount secrets-reader` in the namespace `todoapp`.

## 4. Check that the Deployment uses the ServiceAccount

```bash
kubectl get deployment todoapp -n todoapp -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
kubectl get pods -n todoapp -l app=todoapp -o custom-columns='POD:.metadata.name,SERVICE_ACCOUNT:.spec.serviceAccountName'
```

Expected: `secrets-reader` for the Deployment and for every pod.

The ServiceAccount token is mounted into the pod:

```bash
POD=$(kubectl get pod -n todoapp -l app=todoapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n todoapp "$POD" -- ls /var/run/secrets/kubernetes.io/serviceaccount
```

Expected: `ca.crt`, `namespace`, `token`.

## 5. Quick permission check (without entering the pod)

```bash
kubectl auth can-i list secrets   -n todoapp --as=system:serviceaccount:todoapp:secrets-reader
kubectl auth can-i list secrets   -n todoapp --as=system:serviceaccount:todoapp:default
kubectl auth can-i list secrets   -n default --as=system:serviceaccount:todoapp:secrets-reader
kubectl auth can-i delete secrets -n todoapp --as=system:serviceaccount:todoapp:secrets-reader
kubectl auth can-i list pods      -n todoapp --as=system:serviceaccount:todoapp:secrets-reader
```

Expected, in this order: `yes`, `no`, `no`, `no`, `no`.
Only `secrets-reader` may list Secrets, and only in the namespace `todoapp`.

## 6. List Secrets with curl from the Deployment pod

```bash
POD=$(kubectl get pod -n todoapp -l app=todoapp -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n todoapp "$POD" -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  curl -sS --cacert $SA/ca.crt \
       -H "Authorization: Bearer $(cat $SA/token)" \
       https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets'
```

Expected: a JSON document with `"kind": "SecretList"` and the Secrets of the namespace
`todoapp` in `items` (for example `app-secret`). The values in `data` are only base64-encoded,
not encrypted.

Shorter output (only the Secret names):

```bash
kubectl exec -n todoapp "$POD" -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  curl -sS --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" \
       https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets' | grep -E '"kind"|"name"'
```

**Take a screenshot of this output and attach it to the PR.**

If `curl` is not installed in the image (`kubectl exec -n todoapp "$POD" -- which curl`
prints nothing), the same request can be made with Python:

```bash
kubectl exec -n todoapp "$POD" -- python -c "import ssl,urllib.request as u; sa='/var/run/secrets/kubernetes.io/serviceaccount'; r=u.Request('https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets', headers={'Authorization':'Bearer '+open(sa+'/token').read()}); print(u.urlopen(r, context=ssl.create_default_context(cafile=sa+'/ca.crt')).read().decode())"
```

## 7. Check that the permissions are limited

```bash
kubectl exec -n todoapp "$POD" -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  for path in namespaces/todoapp/secrets namespaces/default/secrets namespaces/todoapp/pods; do
    printf "%s -> " "$path"
    curl -s -o /dev/null -w "%{http_code}\n" --cacert $SA/ca.crt \
         -H "Authorization: Bearer $(cat $SA/token)" \
         https://kubernetes.default.svc/api/v1/$path
  done'
```

Expected:

```
namespaces/todoapp/secrets -> 200
namespaces/default/secrets -> 403
namespaces/todoapp/pods -> 403
```

`200` means the Role works. `403` means the permission is limited to Secrets in the
namespace `todoapp`: other namespaces and other resources are forbidden. The reason of a
rejection is visible when the request is repeated without `-o /dev/null`:
`secrets is forbidden: User "system:serviceaccount:todoapp:secrets-reader" cannot list resource "secrets" ...`.

## Requirements checklist

| Requirement | Where to verify |
|---|---|
| Cluster created from `cluster.yml` | section 1 |
| `security/rbac.yml` contains ServiceAccount, Role and RoleBinding | section 3 |
| Role allows listing secrets | sections 3, 5, 6 |
| RoleBinding binds the Role to the ServiceAccount | sections 3, 5 |
| Deployment uses the ServiceAccount | section 4 |
| Secrets listed with curl from the pod, screenshot attached | section 6 |

## Cleanup

```bash
kind delete cluster
```