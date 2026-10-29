# Trident Port v1.2.4: install by operator, offline (Kubernetes)

Your own registry (Harbor or similar); nothing is pulled from `ghcr.io`. Run every block in order, top to bottom, in the same shell. Every block can be pasted as it is: the only values you change are in step 1, once.

Images carried by the bundle: `ghcr.io/calzdevops/trident-port:v1.2.4`, `ghcr.io/calzdevops/trident-port-operator:v1.2.4`, `curlimages/curl:8.11.0` (linux/amd64).

## Requirements

- Kubernetes 1.25 or later; Trident 24 or later with an online backend; destination StorageClass created by you.
- A StorageClass for the tracking volume (2 GiB): reclaim policy `Retain`, not a migration source. A Trident NAS class (ReadWriteMany) is recommended.
- cluster-admin on the cluster (`install.yaml` creates a CRD, a ClusterRole and a ClusterRoleBinding).
- A host that reaches your registry and the cluster, with `kubectl`, `skopeo` (or `docker`), `sha256sum`, `curl`.
- A private project in your registry, and a user and token that can push to it and pull from it.

## 0. Get the two files

`trident-port-offline-v1.2.4.tar.gz` (~350 MB) and `trident-port-offline-v1.2.4.tar.gz.sha256`, from NetApp or from the GitHub Release `v1.2.4`. Put both in an empty working folder and `cd` into it.

With a GitHub token that can read the release (typed, not shown):

```bash
read -rsp "GitHub token: " GH_TOKEN; echo
API=https://api.github.com/repos/CalzDevOps/trident-port
for f in trident-port-offline-v1.2.4.tar.gz trident-port-offline-v1.2.4.tar.gz.sha256; do
  id=$(curl -s -H "Authorization: Bearer $GH_TOKEN" $API/releases/tags/v1.2.4 \
       | python3 -c "import sys,json; print([a['id'] for a in json.load(sys.stdin)['assets'] if a['name']=='$f'][0])")
  curl -sL -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/octet-stream" -o "$f" $API/releases/assets/$id
done
unset GH_TOKEN
ls -la trident-port-offline-v1.2.4.tar.gz*
```

Expected: two files, the `.tar.gz` around 350 MB.

## 1. Your values (the only block you edit)

Change the right-hand side of each line to your own value, then paste the block.

```bash
export VERSION=v1.2.4
export MIRROR_HOST=harbor.example.com          # your registry host
export MIRROR=$MIRROR_HOST/trident-port        # your registry host / your project
export REGISTRY_USER=robot-trident-port        # your registry user
export TRACKING_SC=sc-nas-retain               # your StorageClass for the tracking volume
export TRIDENT_NS=trident                      # namespace where Trident runs
read -rsp "Registry token for $REGISTRY_USER: " REGISTRY_TOKEN; echo; export REGISTRY_TOKEN
```

Expected: only the token prompt.

Check the two cluster values:

```bash
kubectl get sc "$TRACKING_SC" -o jsonpath='{.reclaimPolicy}{"\n"}'
kubectl get pods -n "$TRIDENT_NS" -l app=controller.csi.trident.netapp.io
```

Expected: `Retain`, then one Trident controller pod `Running`. If the second command lists nothing, find the namespace with `kubectl get pods -A -l app=controller.csi.trident.netapp.io`, fix `TRIDENT_NS` and paste the block again.

## 2. Verify and extract

```bash
sha256sum -c "trident-port-offline-$VERSION.tar.gz.sha256"
```

Expected: the bundle name followed by `: OK`.

```bash
tar xzf "trident-port-offline-$VERSION.tar.gz" && cd "trident-port-offline-$VERSION"
sha256sum -c SHA256SUMS
```

Expected: one `OK` line per file (`images/…oci.tar`, `install.yaml`, `IMAGES.txt`, …) and no `FAILED`. Stop if there is one.

```bash
cat IMAGES.txt
```

Expected: two comment lines (source reference, `# platform: linux/amd64`), then three lines `reference sha256:digest images/file.oci.tar` for `trident-port`, `trident-port-operator` (both at the version you set in step 1) and `curlimages/curl:8.11.0`.

## 3. Trident namespace (only if it is not `trident`)

```bash
if [ "$TRIDENT_NS" != trident ]; then
  sed -i "s/^  namespace: trident$/  namespace: $TRIDENT_NS/" install.yaml
  sed -i "s/^spec:$/spec:\n  config:\n    TRIDENT_NAMESPACE: $TRIDENT_NS/" examples/tridentport-*.yaml
  grep -c "namespace: $TRIDENT_NS$" install.yaml
fi
```

Expected: nothing when Trident runs in `trident`; otherwise `2`.

## 4. Log in to your registry

```bash
echo "$REGISTRY_TOKEN" | skopeo login "$MIRROR_HOST" -u "$REGISTRY_USER" --password-stdin
```

Expected: `Login Succeeded!` (with docker: `echo "$REGISTRY_TOKEN" | docker login "$MIRROR_HOST" -u "$REGISTRY_USER" --password-stdin`).

## 5. Load the images

```bash
REGISTRY="$MIRROR" ./load-images.sh
```

Expected: one `== source -> destination` block per image, then `Loaded 3 images:` with `$MIRROR/trident-port`, `$MIRROR/trident-port-operator` (your version) and `$MIRROR/curl:8.11.0`. The script exits non-zero if a pushed digest differs from `IMAGES.txt`. Registry with a certificate this host does not trust: run `TLS_VERIFY=false REGISTRY="$MIRROR" ./load-images.sh`.

## 6. Operator namespace and pull secret (the secret name stays `ghcr`; it holds your registry credential)

```bash
kubectl create namespace trident-port-system --save-config
kubectl -n trident-port-system create secret docker-registry ghcr \
  --docker-server="$MIRROR_HOST" --docker-username="$REGISTRY_USER" --docker-password="$REGISTRY_TOKEN"
```

Expected: `namespace/trident-port-system created`, `secret/ghcr created`

## 7. Operator, from your registry

```bash
sed -e "s#ghcr.io/calzdevops/#$MIRROR/#" -e "s#image: curlimages/curl#image: $MIRROR/curl#" install.yaml > install.mirror.yaml
grep -E '^\s+image: \S' install.mirror.yaml
```

Expected: exactly two lines, `image: $MIRROR/curl:8.11.0` and `image: $MIRROR/trident-port-operator:` + your version (with your registry in place of `$MIRROR`). No `ghcr.io`.

```bash
kubectl apply -f install.mirror.yaml
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

Expected: `created` for the CRD, ClusterRole, ClusterRoleBinding, ServiceAccount and Deployment, `namespace/trident-port created`, `namespace/trident-port-system configured`, then `deployment "trident-port-operator" successfully rolled out`.

If it does not return: `kubectl -n trident-port-system get pods`. Pod `Init:Error` names the missing secret.

## 8. Product pull secret

`install.yaml` already created the `trident-port` namespace. Create the secret there, before the `TridentPort` in step 9:

```bash
kubectl -n trident-port create secret docker-registry ghcr \
  --docker-server="$MIRROR_HOST" --docker-username="$REGISTRY_USER" --docker-password="$REGISTRY_TOKEN"
kubectl -n trident-port get secret ghcr
```

Expected: `secret/ghcr created`, then one row `ghcr` of type `kubernetes.io/dockerconfigjson`. Without it the product pod stays in `Init:CrashLoopBackOff` (`secret(s) ghcr not found`).

Reinstalling: if `kubectl get ns trident-port` shows `Terminating`, wait until it is gone and repeat from step 7.

## 9. Product, from your registry

```bash
sed -e "s/[<]version[>]/$VERSION/" -e "s/[<]tracking-sc[>]/$TRACKING_SC/" \
  -e "s#ghcr.io/calzdevops/#$MIRROR/#" -e "s#image: curlimages/curl#image: $MIRROR/curl#" \
  -e 's/^spec:$/spec:\n  copyImage:\n    pullPolicy: IfNotPresent/' \
  examples/tridentport-minimal.yaml > tridentport.mirror.yaml
grep -E 'repository:|tag:|image: |accessMode|pullPolicy' tridentport.mirror.yaml
```

Expected: `repository: $MIRROR/trident-port`, `tag:` = the version you set in step 1, `image: $MIRROR/curl:8.11.0` (your registry in place of `$MIRROR`), `pullPolicy: IfNotPresent` (the copy pods use the image already on the node). No `ghcr.io` and no `curlimages`.

NAS (ReadWriteMany) tracking class, the recommended case: enable `accessMode: ReadWriteMany`:

```bash
sed -i 's/^\(\s*\)# *accessMode: ReadWriteMany/\1accessMode: ReadWriteMany/' tridentport.mirror.yaml
grep -n 'accessMode' tridentport.mirror.yaml
```

Expected: a line `accessMode: ReadWriteMany` without `#`. Skip this block for a ReadWriteOnce class.

```bash
kubectl apply -f tridentport.mirror.yaml
kubectl -n trident-port get tridentport trident-port
```

Expected: `tridentport.trident-port.netapp.io/trident-port created`, then column `DEPLOYED` = `True` (repeat the `get` until it is).

```bash
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
```

Expected: `deployment "trident-port" successfully rolled out`. The webhook pod (`trident-port-webhook-…`) may stay in `ContainerCreating` for a few minutes: it starts by itself once the product pod is Ready.

The copy pods run the same product image from your registry. The product copies its own pull secret into each application namespace for the run and removes it afterwards.

## 10. Licence

Cluster ID to send to NetApp:

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

Expected: the web address, `http://` + node IP + port. `NotFound` on the secret means the product pod is not Ready yet: repeat step 9's `rollout status`.

```bash
curl -s -X POST "http://$NODE_IP:$NODE_PORT/api/license" -H "X-Trident-Port-Token: $API_TOKEN" \
  -H 'Content-Type: application/json' -d "{\"token\":\"$(cat licence.txt)\"}"
```

Expected: JSON with `"valid": true`. The same token can be pasted in Settings > License.

## 11. Verify

```bash
kubectl -n trident-port get pods
kubectl -n trident-port exec deploy/trident-port -c trident-port -- curl -s http://localhost:8000/api/health
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version
kubectl -n trident-port get pvc trident-port-tracking
```

Expected:
- `trident-port-…` and `trident-port-webhook-…`, both `1/1 Running`;
- `{"status":"ok", ...}`;
- a line naming the version you set in step 1;
- `STATUS` = `Bound`, `STORAGECLASS` = your tracking class.

Web: open the address from step 10. Settings > Product version shows the version you set in step 1; Settings > License shows tier and expiry. The web asks for the API token on the first request:

```bash
kubectl -n trident-port get secret trident-port-api-token -o jsonpath='{.data.token}' | base64 -d; echo
```

Expected: a token string.

## 12. Check no pod pulls from outside your registry

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | grep -E 'trident-port|curl' | sort -u
kubectl -n trident-port logs deploy/trident-port -c check-pull-secret
```

Expected: only lines that start with your registry (none with `curlimages/curl` or `ghcr.io/`), then `pull secret check skipped: secret 'ghcr' is not a ghcr.io credential (registry: ` + your registry host + `)`.

## 13. Upgrade to a new version

With the operator the `TridentPort` resource decides the version. Do not run `helm upgrade` on this release: the operator puts the old tag back at its next reconcile, and the chart refuses it. Do not upgrade while a cutover is running.

With the new release's bundle: steps 0 to 5 with the new `VERSION` in step 1. Then the operator first (it carries the RBAC the new product needs):

```bash
sed -e "s#ghcr.io/calzdevops/#$MIRROR/#" -e "s#image: curlimages/curl#image: $MIRROR/curl#" install.yaml > install.mirror.yaml
grep -E '^\s+image: \S' install.mirror.yaml
kubectl apply -f install.mirror.yaml
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

Expected: the two `image:` lines name your registry and the new version; `configured` for the ClusterRole and the Deployment; `successfully rolled out`.

Then the product:

```bash
kubectl -n trident-port patch tridentport trident-port --type merge \
  -p "{\"spec\":{\"image\":{\"repository\":\"$MIRROR/trident-port\",\"tag\":\"$VERSION\"}}}"
kubectl -n trident-port get tridentport trident-port
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version
```

Expected: `patched`, `DEPLOYED` = `True` (repeat until it is), `successfully rolled out`, then the new version. The web is unavailable for about a minute while the pod restarts.

## 14. After the last namespace of a cluster

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
