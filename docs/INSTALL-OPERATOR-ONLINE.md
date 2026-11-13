# Trident Port v1.2.12: install by operator, online (Kubernetes)

Use this procedure when the cluster reaches `ghcr.io` and you hold a token from NetApp. Run every block in order, top to bottom, in the same shell. You can paste every block as it is. The only values you change are in step 1, and you change them once. If you have no access to `ghcr.io` and use a registry of your own, follow `INSTALL-OPERATOR-OFFLINE.md` instead.

The images, published at the version you set in step 1, are `ghcr.io/calzdevops/trident-port`, `ghcr.io/calzdevops/trident-port-operator` and `curlimages/curl:8.11.0` (the pull-secret check init container, `checkPullSecret.image`).

## Before you begin

- The cluster runs Kubernetes 1.25 or later, and Trident 24 or later with an online backend. You have created the destination StorageClass.
- A StorageClass exists for the tracking volume (2 GiB). It has reclaim policy `Retain` and is not a migration source. Use a Trident NAS class (ReadWriteMany). With a ReadWriteOnce class, every `tp` command is tied to the node of the Trident Port pod.
- You have cluster-admin on the cluster. `install.yaml` creates a CRD, a ClusterRole and a ClusterRoleBinding.
- The NFS client is installed on every node: `nfs-utils` on RHEL, SLES and Rocky, `nfs-common` on Ubuntu. A migration needs it for every NFS source and every NFS destination, because the copy pod mounts the volume through the node. Without it the copy pod stays in `ContainerCreating` with `mount failed: exit status 32`. Check a node with `ls /sbin/mount.nfs`. If the file is missing, install the package, for example `sudo dnf install -y nfs-utils` on Rocky or `sudo apt-get install -y nfs-common` on Ubuntu.
- You have a host that reaches the cluster and `ghcr.io`, with `kubectl`, `tar` and `curl`. On OpenShift the host also needs `helm` 3.8 or later, to render the security context constraints.

## 0. Get the file

Get `trident-port-operator-v1.2.12.tar.gz`. The file comes from the link and the token your vendor gave you. Put it in an empty working folder and `cd` into it.

A generic example. Set `BUNDLE_URL` to the link your vendor gave you (the token is typed, not shown):

```bash
read -rsp "Download token: " DL_TOKEN
echo
BUNDLE_URL="https://downloads.example.com/trident-port"   # replace with the link your vendor gave you
f=trident-port-operator-v1.2.12.tar.gz
curl -fL -H "Authorization: Bearer $DL_TOKEN" -o "$f" "$BUNDLE_URL/$f"
unset DL_TOKEN
ls -la "$f"
```

If your vendor gave you the file another way, skip this block.

Result: the folder lists one `.tar.gz` file of about 43 KB.

## 1. Set your values (the only block you edit)

Change the right-hand side of each line to your own value, then paste the block.

```bash
export VERSION=v1.2.12
export TRACKING_SC=sc-nas-retain               # your StorageClass for the tracking volume
export TRIDENT_NS=trident                      # namespace where Trident runs
export REG_USER=my-ghcr-user                   # your registry user, the one you were given
read -rsp "Registry token for $REG_USER (read:packages): " REG_TOKEN
echo
export REG_TOKEN
```

Result: the shell shows only the token prompt and does not echo the token.

Check the two cluster values:

```bash
kubectl get sc "$TRACKING_SC" -o jsonpath='{.reclaimPolicy}{"\n"}'
kubectl get pods -n "$TRIDENT_NS" -l app=controller.csi.trident.netapp.io
```

Result: the first command prints `Retain`, and the second lists one Trident controller pod in `Running`. If the second command lists nothing, find the namespace with `kubectl get pods -A -l app=controller.csi.trident.netapp.io`, correct `TRIDENT_NS` and paste the block again.

## 2. Extract the package

```bash
tar xzf "trident-port-operator-$VERSION.tar.gz" && cd "trident-port-operator-$VERSION"
ls install.yaml examples
```

Result: `ls` lists `install.yaml`, then `tridentport-minimal.yaml` and `tridentport-private-registry.yaml`.

## 3. Set the Trident namespace (only if it is not `trident`)

```bash
if [ "$TRIDENT_NS" != trident ]
then
  sed -i "s/^  namespace: trident$/  namespace: $TRIDENT_NS/" install.yaml
  sed -i "s/^spec:$/spec:\n  config:\n    TRIDENT_NAMESPACE: $TRIDENT_NS/" examples/tridentport-*.yaml
  grep -c "namespace: $TRIDENT_NS$" install.yaml
fi
```

Result: the block prints nothing when Trident runs in `trident`. Otherwise it prints `2`.

## 4. Create the operator namespace and pull secret

The pull secret must be named `ghcr`.

```bash
kubectl create namespace trident-port-system --save-config
kubectl -n trident-port-system create secret docker-registry ghcr \
  --docker-server=ghcr.io --docker-username="$REG_USER" --docker-password="$REG_TOKEN"
```

Result: the commands print `namespace/trident-port-system created` and `secret/ghcr created`. On a reinstall both may exist: `namespace ... AlreadyExists` is fine, and for the secret run `kubectl -n trident-port-system delete secret ghcr` and paste the block again.

## 5. Install the operator

```bash
sed "s#trident-port-operator:v[0-9][^ ]*#trident-port-operator:$VERSION#" install.yaml | kubectl apply -f -
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

Result: `kubectl apply` prints `created` for the CRD, ClusterRole, ClusterRoleBinding, ServiceAccount and Deployment, `namespace/trident-port created` and `namespace/trident-port-system configured`. On a reinstall the `trident-port` namespace already exists and the line reads `namespace/trident-port configured`: both are correct. The rollout then prints `deployment "trident-port-operator" successfully rolled out`.

### On OpenShift

Do not use the commands above. On OpenShift, the operator pod is refused with `unable to validate against any security context constraint` and `.initContainers[0].runAsUser: Invalid value: 100`. The init container `check-pull-secret` pins that user, and `restricted-v2` admits only the range of the namespace. Run this `sed` in place of the first one. It also drops the `runAsUser: 100` line, so OpenShift assigns the user itself:

```bash
sed -e "s#trident-port-operator:v[0-9][^ ]*#trident-port-operator:$VERSION#" -e '/^ *runAsUser: 100$/d' install.yaml | kubectl apply -f -
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

Result: the same as above. The operator starts under `restricted-v2`, with no `anyuid` and no extra constraint. Use this same `sed` when you upgrade the operator.

If you install through OLM, the bundle already has no fixed user. This step does not apply.

If the rollout does not return, run `kubectl -n trident-port-system get pods`. A pod in `Init:Error` names the missing secret.

## 6. Create the product pull secret

The pull secret must be named `ghcr`. `install.yaml` already created the `trident-port` namespace. Create the secret there, before the `TridentPort` in step 7:

```bash
kubectl -n trident-port create secret docker-registry ghcr \
  --docker-server=ghcr.io --docker-username="$REG_USER" --docker-password="$REG_TOKEN"
kubectl -n trident-port get secret ghcr
```

Result: the commands print `secret/ghcr created`, then one row `ghcr` of type `kubernetes.io/dockerconfigjson`. Without this secret, the product pod stays in `Init:CrashLoopBackOff` with `secret(s) ghcr not found`. On a reinstall the secret may still exist, and `create` prints `Error from server (AlreadyExists): secrets "ghcr" already exists`: run `kubectl -n trident-port delete secret ghcr` and paste the block again.

If you reinstall and `kubectl get ns trident-port` shows `Terminating`, wait until the namespace is gone and repeat from step 5.

## 7. Install the product

```bash
sed -e "s/[<]version[>]/$VERSION/" -e "s/[<]tracking-sc[>]/$TRACKING_SC/" \
  examples/tridentport-minimal.yaml > tridentport.yaml
grep -E 'tag:|storageClass:' tridentport.yaml
```

Result: `tag:` shows the version you set in step 1, and `storageClass:` shows your tracking class.

For a NAS (ReadWriteMany) tracking class, the recommended case, enable `accessMode: ReadWriteMany`:

```bash
sed -i 's/^\(\s*\)# *accessMode: ReadWriteMany/\1accessMode: ReadWriteMany/' tridentport.yaml
grep -n 'accessMode' tridentport.yaml
```

Result: the file has a line `accessMode: ReadWriteMany` without `#`. Skip this block for a ReadWriteOnce class.


**Argo CD in another cluster (optional).** If an Argo CD hub that runs in another cluster manages this one, give the product its kubeconfig, or the product cannot suspend the Applications and the hub puts a stopped workload back. Create a Secret in the `trident-port` namespace that holds the hub kubeconfig (the Secret holds it under the key `kubeconfig`), and name it in the `TridentPort` under `spec.argocdHub` (add it under `spec:` in `tridentport.yaml`, which the next block applies). The fields are `existingSecret` (the Secret name, required for this feature), `key` (the key in the Secret, default `kubeconfig`), `context` (a context inside the kubeconfig, optional) and `namespace` (where Argo CD runs, default `argocd`). Without `existingSecret` the product pod has no Argo CD settings, and the guide behaves as it does today. Create the Secret before you apply the resource; the example `examples/tridentport-private-registry.yaml` carries the block commented out. The hub account needs only `get`, `list` and `patch` on `applications` and `applicationsets`: see "Argo CD in another cluster" in `MIGRATE-A-CLUSTER.md`.

```bash
kubectl -n trident-port create secret generic argocd-hub --from-file=kubeconfig=/path/to/hub.kubeconfig
```

Result: `kubectl` prints `secret/argocd-hub created`. If it prints `AlreadyExists`, run `kubectl -n trident-port delete secret argocd-hub` and paste the block again. Skip this block when Argo CD runs in this cluster.

```bash
kubectl apply -f tridentport.yaml
kubectl -n trident-port get tridentport trident-port
```

Result: `kubectl apply` prints `tridentport.trident-port.netapp.io/trident-port created`, and the `DEPLOYED` column shows `True`. Repeat the `get` command until it does.

```bash
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
```

Result: the command prints `deployment "trident-port" successfully rolled out`. The webhook pod (`trident-port-webhook-…`) can stay in `ContainerCreating` for up to about 7 minutes. It starts by itself once the product pod is Ready.

## 8. Apply the license

Get the cluster ID to send to NetApp:

```bash
kubectl get ns kube-system -o jsonpath='{.metadata.uid}{"\n"}'
```

Result: the command prints a UUID. NetApp sends you a token for that ID. Save it as `license.txt` in this folder. The file holds one line that starts with `eyJ`.

```bash
NODE_PORT=$(kubectl -n trident-port get svc trident-port -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
API_TOKEN=$(kubectl -n trident-port get secret trident-port-api-token -o jsonpath='{.data.token}' | base64 -d)
echo "http://$NODE_IP:$NODE_PORT"
```

Result: the command prints the web address, `http://` followed by the node IP and port. If the secret lookup returns `NotFound`, the product pod is not Ready yet. Repeat the `rollout status` command of step 7. The web `NodePort` is chosen by Kubernetes when the service is created, so after a reinstall it is usually another number: use the one this block prints, and tell whoever has the old address bookmarked or a firewall rule on it. To keep a fixed port, add `service:` with `nodePort: <port>` under `spec:` in the `TridentPort` of step 7 (the port must be inside the cluster's NodePort range, 30000-32767 by default) and apply it again; the operator changes the service. Not measured on a cluster.

```bash
curl -s -X POST "http://$NODE_IP:$NODE_PORT/api/license" -H "X-Trident-Port-Token: $API_TOKEN" \
  -H 'Content-Type: application/json' -d "{\"token\":\"$(cat license.txt)\"}"
```

Result: the API returns JSON with `"valid": true`. You can also paste the same token in Settings > License.

## 9. Verify the installation

```bash
kubectl -n trident-port get pods
kubectl -n trident-port exec deploy/trident-port -c trident-port -- curl -s http://localhost:8000/api/health
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version
kubectl -n trident-port get pvc trident-port-tracking
```

Result:
- The pods `trident-port-…` and `trident-port-webhook-…` are both `1/1 Running`.
- The health endpoint returns `{"status":"ok", ...}`. Without the API token, the endpoint returns only `status` and `platform`.
- `tp --version` prints a line with the version you set in step 1.
- The tracking claim shows `STATUS` = `Bound` and `STORAGECLASS` = your tracking class.

Open the web address from step 8. Settings > Product version shows the version you set in step 1. Settings > License shows the tier and the expiry date. The web asks for the API token on the first request. Print it with this command:

```bash
kubectl -n trident-port get secret trident-port-api-token -o jsonpath='{.data.token}' | base64 -d
echo
```

Result: the command prints a token string.

## Next step: register the ONTAP array

The install is done. Before the first migration, register the ONTAP array as in step 0 of [`MIGRATE-A-CLUSTER.md`](MIGRATE-A-CLUSTER.md#0-before-you-begin) (the "Register the ONTAP array" item).

Without it, the product cannot raise the inode limit of the destination volume, and a volume with many files fails with `[ERROR] data: source holds N files and the destination volume allows M ... the copy would fail with 'No space left on device'`.

The account that Trident Port uses for the array also needs read access to the aggregates (`GET /storage/aggregates`), and to the SVM's aggregate list (`GET /svm/svms`), because the pre-flight reads them to check that the destination has room. An account limited to one SVM with the `vsadmin` role was refused on `/storage/aggregates` with HTTP 403 in a measurement, so give the account a role that can read aggregates. Without it the pre-flight cannot check the free space of the aggregate and the line stays at WARN. This is a permission on the array, not on Kubernetes. The array calls of each mode are in `ontap-account-per-mode.md`.

## Speed analysis after the install

About two minutes after the product serves, Trident Port measures the speed of your storage once, in the background, so the first estimate already rests on a measurement. It creates small scratch volumes of its own in the namespace of the product, one at a time, labelled `trident-port-speed-analysis`, and deletes them. It never blocks or fails the install. When it cannot measure (no Trident class yet, no permission, no room) it says why in Settings > Speed analysis, and you can run it again there or with `tp speed-analyze`. To turn it off, add this to the `spec` of the `TridentPort` resource and apply it again:

```yaml
spec:
  speedAnalysis:
    onInstall: false
```

## 10. Upgrade to a new version

With the operator, the `TridentPort` resource decides the version. Do not run `helm upgrade` on this release. It ends with exit code 0, and the operator puts the old tag back at its next reconcile. The chart refuses it and prints the commands below. Do not upgrade while a cutover is running.

Download the operator package of the new release, extract it and `cd` into it (steps 0 and 2 with the new version). Then set the new version in step 1 and paste that block again.

Since the release after v1.2.12 the RBAC that the operator carries also includes `patch` on `pods`. The product uses it only to put the label `skipPreStopChecks` on the pods of a RabbitMQ with more than one replica before it stops them, and to remove it. Without it, a cutover of such a RabbitMQ is refused with `refusing to stop RabbitmqCluster ... could not put the label skipPreStopChecks` and nothing is stopped.

The RBAC that the operator carries also grants `patch` and `update` on the custom resources that own an application workload and are not on the list of supported operators, such as `clickhousekeeperinstallations`. The product writes the note it needs to restore the application on the custom resource that owns a workload. Without the verbs, the pre-flight refuses with `the restore note would be refused on clickhousekeeperinstallation/...: cannot patch` and nothing is stopped. It also grants `list` on `csidrivers` and `get`, `list` and `watch` on `limitranges`, read only, which the pre-flight uses and would otherwise report as unknown. Upgrade the operator first, because it carries the RBAC that the new product needs.

Check that the new pods fit before you upgrade. The old pod stops first (both Deployments use `Recreate`: the product mounts one ReadWriteOnce volume, and two operator pods must never reconcile at the same time), so a new pod whose CPU request fits on no node stays `Pending` with `Insufficient cpu` and nothing runs until you fix it. The new pod can take the room the old one frees, so this only happens when its request is larger than the old one or another pod took the room. The default requests are `10m` CPU / `128Mi` for the operator and `50m` CPU / `512Mi` for the product. Look at the requests already booked on each node:

```bash
kubectl describe nodes | grep -E '^Name:|^  cpu '
```

Result: one `cpu` line per node, with the requests and their share of the allocatable CPU. A node at 99% has little room left. If no node can take the request, lower it before you upgrade:

```bash
kubectl -n trident-port-system patch deploy trident-port-operator -p '{"spec":{"template":{"spec":{"containers":[{"name":"manager","resources":{"requests":{"cpu":"10m"}}}]}}}}'
kubectl -n trident-port patch tridentport trident-port --type merge -p '{"spec":{"resources":{"requests":{"cpu":"10m"}}}}'
```

Result: the first command prints `deployment.apps/trident-port-operator patched` and the second prints `tridentport.trident-port.netapp.io/trident-port patched`. The first is for the operator pod, the second for the product pod. If a pod is already stuck, `kubectl -n trident-port-system get events --field-selector reason=FailedScheduling` (and the same with `-n trident-port`) shows `Insufficient cpu`; the same commands unblock it. The operator install file sets the operator request again each time you apply it, so lower it again after an operator upgrade if the node is still full.

Upgrade the operator first, because it carries the RBAC that the new product needs. Without it, the product upgrade is refused with `ClusterRole/trident-port-operator does not match this chart's RBAC revision. Nothing has been changed by this run.`

```bash
sed "s#trident-port-operator:v[0-9][^ ]*#trident-port-operator:$VERSION#" install.yaml | kubectl apply -f -
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

Result: `kubectl apply` prints `deployment.apps/trident-port-operator configured`. The other objects print `unchanged`, unless the new release changes them. The rollout prints `deployment "trident-port-operator" successfully rolled out`.

On OpenShift, use the `sed` of the OpenShift part of step 5 in place of the first command.

Then upgrade the product image. The `TridentPort` pins `spec.image.tag`, so the image does not change until the tag does.

```bash
kubectl -n trident-port patch tridentport trident-port --type merge -p "{\"spec\":{\"image\":{\"tag\":\"$VERSION\"}}}"
kubectl -n trident-port get tridentport trident-port
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version
```

Result: the patch prints `patched`, and the `DEPLOYED` column shows `True` (repeat the `get` command until it does). The rollout prints `successfully rolled out`, and `tp --version` prints the new version. The web is unavailable for about a minute while the pod restarts.

## 11. After the last namespace of a cluster

New PVCs that name no StorageClass are created on the default class. Make the Trident class the default. First set the two class names (see `kubectl get sc`):

```bash
kubectl get sc
export OLD_DEFAULT_SC=local-path                # the class marked (default) today
export TRIDENT_DEFAULT_SC=sc-nas                # the Trident class that becomes the default
kubectl patch sc "$OLD_DEFAULT_SC" -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch sc "$TRIDENT_DEFAULT_SC" -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc
```

Result: `kubectl get sc` shows `(default)` next to the Trident class only.

## OpenShift

On OpenShift, the product `ReplicaSet` is refused with `unable to validate against any security context constraint` and `runAsUser: 100` unless the chart knows that it runs there. Put this block in the `TridentPort` before the `kubectl apply` of step 7:

```yaml
spec:
  openshift:
    enabled: true
    copyNfsScc:
      create: false
    nfsImportScc:
      create: false
    blockMirrorScc:
      create: false
```

Merge the `openshift:` block into the `spec:` of your file and keep the rest. Result: the product pod starts under the `restricted-v2` constraint.

The three `create: false` lines are needed because the operator cannot create security context constraints. The cluster admin applies them by hand, once, from the chart of the same version:

```bash
helm template trident-port oci://ghcr.io/calzdevops/charts/trident-port --version "${VERSION#v}" --namespace trident-port \
  --api-versions security.openshift.io/v1 --show-only templates/scc-copy-nfs.yaml | oc apply -f -
helm template trident-port oci://ghcr.io/calzdevops/charts/trident-port --version "${VERSION#v}" --namespace trident-port \
  --api-versions security.openshift.io/v1 --show-only templates/scc-nfs-import.yaml | oc apply -f -
helm template trident-port oci://ghcr.io/calzdevops/charts/trident-port --version "${VERSION#v}" --namespace trident-port \
  --api-versions security.openshift.io/v1 --show-only templates/scc-blockmirror-node.yaml | oc apply -f -
oc get scc | grep trident-port
```

Result: `oc apply` prints `created` for each object. `oc get scc` lists `trident-port-copy-nfs`, `trident-port-nfs-import` and `trident-port-blockmirror-node`. The commands also create four ClusterRoles, `trident-port-copy-nfs-use`, `trident-port-copy-nfs-bind`, `trident-port-blockmirror-node-use` and `trident-port-blockmirror-bind`. They create two ClusterRoleBindings, `trident-port-copy-nfs-bind` and `trident-port-blockmirror-bind`. Check them with `oc get clusterrole,clusterrolebinding | grep trident-port`. The first constraint lets the Live Sync copy pods mount the source export. The second is for `tp import` and the Block and NAS conversions. The third is needed only for Block Mirror, and you can leave it out without Block Mirror. On a cluster without access to `ghcr.io`, render the same templates from the chart `.tgz` of the offline bundle, as in `INSTALL-OPERATOR-OFFLINE.md`. `UNINSTALL.md` removes these objects by name.

If you do not apply `scc-copy-nfs.yaml`, the copy pods need the `anyuid` constraint in every namespace that you migrate. `tp live-sync start` checks this before it starts. For each namespace where the copy pods are not admitted, it prints `COPY PODS NOT ADMITTED` and the exact `oc create rolebinding ...` command that grants `anyuid` to the `trident-port-copy` ServiceAccount. Run that command as printed, then run the start again. Those namespaces are left out of the start until then.
