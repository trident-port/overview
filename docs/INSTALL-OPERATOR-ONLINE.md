# Trident Port v1.2.3: install by operator, online (Kubernetes)

The cluster reaches `ghcr.io` and you hold a token of ours. Run every block in order, top to bottom, in the same shell. Every block can be pasted as it is: the only values you change are in step 1, once. No registry of your own: `INSTALL-OPERATOR-OFFLINE.md`.

Images (published at the version you set in step 1): `ghcr.io/calzdevops/trident-port`, `ghcr.io/calzdevops/trident-port-operator`, and `curlimages/curl:8.11.0` (the pull-secret check init container, `checkPullSecret.image`).

## Requirements

- Kubernetes 1.25 or later; Trident 24 or later with an online backend; destination StorageClass created by you.
- A StorageClass for the tracking volume (2 GiB): reclaim policy `Retain`, not a migration source. Use a Trident NAS class (ReadWriteMany): with a ReadWriteOnce class every `tp` command is tied to the node of the Trident Port pod.
- cluster-admin on the cluster (`install.yaml` creates a CRD, a ClusterRole and a ClusterRoleBinding).
- A host that reaches the cluster and `ghcr.io`, with `kubectl`, `tar`, `curl`.

## 0. Get the file

`trident-port-operator-v1.2.3.tar.gz`, from NetApp or from the GitHub Release `v1.2.3`. Put it in an empty working folder and `cd` into it.

With a GitHub token that can read the release (typed, not shown):

```bash
read -rsp "GitHub token: " GH_TOKEN; echo
API=https://api.github.com/repos/CalzDevOps/trident-port
f=trident-port-operator-v1.2.3.tar.gz
id=$(curl -s -H "Authorization: Bearer $GH_TOKEN" $API/releases/tags/v1.2.3 \
     | python3 -c "import sys,json; print([a['id'] for a in json.load(sys.stdin)['assets'] if a['name']=='$f'][0])")
curl -sL -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/octet-stream" -o "$f" $API/releases/assets/$id
unset GH_TOKEN
ls -la "$f"
```

Expected: one `.tar.gz` file.

## 1. Your values (the only block you edit)

Change the right-hand side of each line to your own value, then paste the block.

```bash
export VERSION=v1.2.3
export TRACKING_SC=sc-nas-retain               # your StorageClass for the tracking volume
export TRIDENT_NS=trident                      # namespace where Trident runs
export REG_USER=my-ghcr-user                   # your registry user, the one you were given
read -rsp "Registry token for $REG_USER (read:packages): " REG_TOKEN; echo; export REG_TOKEN
```

Expected: only the token prompt; nothing is echoed.

Check the two cluster values:

```bash
kubectl get sc "$TRACKING_SC" -o jsonpath='{.reclaimPolicy}{"\n"}'
kubectl get pods -n "$TRIDENT_NS" -l app=controller.csi.trident.netapp.io
```

Expected: `Retain`, then one Trident controller pod `Running`. If the second command lists nothing, find the namespace with `kubectl get pods -A -l app=controller.csi.trident.netapp.io`, fix `TRIDENT_NS` and paste the block again.

## 2. Extract

```bash
tar xzf "trident-port-operator-$VERSION.tar.gz" && cd "trident-port-operator-$VERSION"
ls install.yaml examples
```

Expected: `install.yaml`, then `tridentport-minimal.yaml` and `tridentport-private-registry.yaml`.

## 3. Trident namespace (only if it is not `trident`)

```bash
if [ "$TRIDENT_NS" != trident ]; then
  sed -i "s/^  namespace: trident$/  namespace: $TRIDENT_NS/" install.yaml
  sed -i "s/^spec:$/spec:\n  config:\n    TRIDENT_NAMESPACE: $TRIDENT_NS/" examples/tridentport-*.yaml
  grep -c "namespace: $TRIDENT_NS$" install.yaml
fi
```

Expected: nothing when Trident runs in `trident`; otherwise `2`.

## 4. Operator namespace and pull secret (the name must be `ghcr`)

```bash
kubectl create namespace trident-port-system --save-config
kubectl -n trident-port-system create secret docker-registry ghcr \
  --docker-server=ghcr.io --docker-username="$REG_USER" --docker-password="$REG_TOKEN"
```

Expected: `namespace/trident-port-system created`, `secret/ghcr created`

## 5. Operator

```bash
sed "s#trident-port-operator:v[0-9][^ ]*#trident-port-operator:$VERSION#" install.yaml | kubectl apply -f -
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

Expected: `created` for the CRD, ClusterRole, ClusterRoleBinding, ServiceAccount and Deployment, `namespace/trident-port created`, `namespace/trident-port-system configured`, then `deployment "trident-port-operator" successfully rolled out`.

If it does not return: `kubectl -n trident-port-system get pods`. Pod `Init:Error` names the missing secret.

## 6. Product pull secret (the name must be `ghcr`)

`install.yaml` already created the `trident-port` namespace. Create the secret there, before the `TridentPort` in step 7:

```bash
kubectl -n trident-port create secret docker-registry ghcr \
  --docker-server=ghcr.io --docker-username="$REG_USER" --docker-password="$REG_TOKEN"
kubectl -n trident-port get secret ghcr
```

Expected: `secret/ghcr created`, then one row `ghcr` of type `kubernetes.io/dockerconfigjson`. Without it the product pod stays in `Init:CrashLoopBackOff` (`secret(s) ghcr not found`).

Reinstalling: if `kubectl get ns trident-port` shows `Terminating`, wait until it is gone and repeat from step 5.

## 7. Product

```bash
sed -e "s/[<]version[>]/$VERSION/" -e "s/[<]tracking-sc[>]/$TRACKING_SC/" \
  examples/tridentport-minimal.yaml > tridentport.yaml
grep -E 'tag:|storageClass:' tridentport.yaml
```

Expected: `tag:` = the version you set in step 1 and `storageClass:` = your tracking class.

NAS (ReadWriteMany) tracking class, the recommended case: enable `accessMode: ReadWriteMany`:

```bash
sed -i 's/^\(\s*\)# *accessMode: ReadWriteMany/\1accessMode: ReadWriteMany/' tridentport.yaml
grep -n 'accessMode' tridentport.yaml
```

Expected: a line `accessMode: ReadWriteMany` without `#`. Skip this block for a ReadWriteOnce class.

```bash
kubectl apply -f tridentport.yaml
kubectl -n trident-port get tridentport trident-port
```

Expected: `tridentport.trident-port.netapp.io/trident-port created`, then column `DEPLOYED` = `True` (repeat the `get` until it is).

```bash
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
```

Expected: `deployment "trident-port" successfully rolled out`. The webhook pod (`trident-port-webhook-…`) may stay in `ContainerCreating` for a few minutes: it starts by itself once the product pod is Ready.

## 8. Licence

Cluster ID to send to us:

```bash
kubectl get ns kube-system -o jsonpath='{.metadata.uid}{"\n"}'
```

Expected: a UUID. You receive a token: save it as `licence.txt` (this exact name, British spelling) in this folder (one line, starts `eyJ`).

```bash
NODE_PORT=$(kubectl -n trident-port get svc trident-port -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
API_TOKEN=$(kubectl -n trident-port get secret trident-port-api-token -o jsonpath='{.data.token}' | base64 -d)
echo "http://$NODE_IP:$NODE_PORT"
```

Expected: the web address, `http://` + node IP + port. `NotFound` on the secret means the product pod is not Ready yet: repeat step 7's `rollout status`.

```bash
curl -s -X POST "http://$NODE_IP:$NODE_PORT/api/license" -H "X-Trident-Port-Token: $API_TOKEN" \
  -H 'Content-Type: application/json' -d "{\"token\":\"$(cat licence.txt)\"}"
```

Expected: JSON with `"valid": true`. The same token can be pasted in Settings > License.

## 9. Verify

```bash
kubectl -n trident-port get pods
kubectl -n trident-port exec deploy/trident-port -c trident-port -- curl -s http://localhost:8000/api/health
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version
kubectl -n trident-port get pvc trident-port-tracking
```

Expected:
- `trident-port-…` and `trident-port-webhook-…`, both `1/1 Running`;
- `{"status":"ok", ...}` (without the API token the endpoint answers only `status` and `platform`);
- a line naming the version you set in step 1;
- `STATUS` = `Bound`, `STORAGECLASS` = your tracking class.

Web: open the address from step 8. Settings > Product version shows the version you set in step 1; Settings > License shows tier and expiry. The web asks for the API token on the first request:

```bash
kubectl -n trident-port get secret trident-port-api-token -o jsonpath='{.data.token}' | base64 -d; echo
```

Expected: a token string.

## 10. Upgrade to a new version

With the operator the `TridentPort` resource decides the version. Do not run `helm upgrade` on this release: it ends with rc 0, and the operator puts the old tag back at its next reconcile. The chart refuses it and prints the commands below. Do not upgrade while a cutover is running.

Download the new release's operator package, extract it and `cd` into it (step 0 and step 2 with the new version). Then set the new version in step 1 and paste that block again.

Operator first: it carries the RBAC the new product needs. Without it, the product upgrade is refused with `ClusterRole/trident-port-operator does not match this chart's RBAC revision. Nothing has been changed by this run.`

```bash
sed "s#trident-port-operator:v[0-9][^ ]*#trident-port-operator:$VERSION#" install.yaml | kubectl apply -f -
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

Expected: `configured` for the ClusterRole and the Deployment, then `deployment "trident-port-operator" successfully rolled out`.

The product image: the `TridentPort` pins `spec.image.tag`, so the image does not change until the tag does.

```bash
kubectl -n trident-port patch tridentport trident-port --type merge -p "{\"spec\":{\"image\":{\"tag\":\"$VERSION\"}}}"
kubectl -n trident-port get tridentport trident-port
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version
```

Expected: `patched`, `DEPLOYED` = `True` (repeat until it is), `successfully rolled out`, then the new version. The web is unavailable for about a minute while the pod restarts.

## 11. After the last namespace of a cluster

New PVCs that name no StorageClass are created on the default one. Make the Trident class the default. Set the two names first (see `kubectl get sc`):

```bash
kubectl get sc
export OLD_DEFAULT_SC=local-path                # the class marked (default) today
export TRIDENT_DEFAULT_SC=sc-nas                # the Trident class that becomes the default
kubectl patch sc "$OLD_DEFAULT_SC" -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch sc "$TRIDENT_DEFAULT_SC" -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc
```

Expected: `(default)` next to the Trident class only.
