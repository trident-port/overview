# Trident Port v1.2.12: install by operator, offline (Kubernetes and OpenShift)

This path uses your own registry (Harbor, Artifactory or similar). Nothing is pulled from `ghcr.io`. Run every block in order, top to bottom, in the same shell. You can paste every block as it is. The only values you change are in step 1, once.

The bundle carries these images (linux/amd64): `ghcr.io/calzdevops/trident-port:v1.2.12`, `ghcr.io/calzdevops/trident-port-operator:v1.2.12` and `curlimages/curl:8.11.0`.

This guide is written so that a person who has never used Kubernetes can follow it. Where OpenShift differs from plain Kubernetes, the step says "On OpenShift" and gives the exact commands. You need no Kubernetes knowledge, only a terminal on the administration host.

## How to read this guide

- **One terminal.** Open one terminal window on the administration host and keep it open until the end. The values you set in step 1 live only in that window. If you close it, `cd` back into the bundle folder, paste the first line of step 0 (`export VERSION=...`), repeat step 1, and go on from the step you were at.
- **Every step has three parts.** What the step is for, one block of commands, and the line that starts with `Result:`. The `Result:` line is the expected result: the exact output, or a precise description of it, and what to do if you see something else.
- **How to paste.** Copy the whole grey block, paste it into the terminal and press Enter. Do not retype it and do not change it. Lines that start with `#` are comments, the shell ignores them.
- **Do not go on until the result matches.** If it does not, read the line under the block, fix it, and run the same block again. If you cannot fix it, go to "If something goes wrong" at the end.
- **Words.** A project (or namespace) is a folder inside the cluster where things live. A pod is one running program. A secret is a stored password. A StorageClass is the type of disk the cluster hands out. You will not need more than that.
- **`kubectl` and `oc`.** The commands use `kubectl`. On OpenShift the `oc` program does the same job. The checklist below tells you whether `kubectl` is on your host. If it is not, the checklist gives the one line that makes it work.

## Before you begin

- Kubernetes 1.25 or later is running. On OpenShift this means OpenShift 4.18 or later, which is Kubernetes 1.31 underneath.
- Trident 24 or later runs with an online backend, and you have created the destination StorageClass. If Trident is not installed, stop here and install it first, following the Trident installation guide for your cluster. This guide does not install Trident.
- A StorageClass exists for the tracking volume (2 GiB). Its reclaim policy is `Retain`, and it is not a migration source. A Trident NAS class (ReadWriteMany) is recommended.
- You have cluster-admin on the cluster. `install.yaml` creates a CRD, a ClusterRole and a ClusterRoleBinding. On OpenShift, the security context constraints of step 8 also need cluster-admin. If you are not cluster-admin, a cluster-admin runs steps 7 and 8 for you.
- The NFS client is installed on every node: `nfs-utils` on RHEL, SLES and Rocky, `nfs-common` on Ubuntu. A migration needs it for every NFS source and every NFS destination, because the copy pod mounts the volume through the node. Without it the copy pod stays in `ContainerCreating` with `mount failed: exit status 32`. Check a node with `ls /sbin/mount.nfs`. If the file is missing, install the package, for example `sudo dnf install -y nfs-utils` on Rocky or `sudo apt-get install -y nfs-common` on Ubuntu.
- A host reaches your registry and the cluster, and has `kubectl`, `sha256sum` and `curl`. To push the images it needs `skopeo`, `docker` or `crane`. Without them, step 5 shows an alternative that needs no root. On OpenShift the host also needs `helm` 3.8 or later, to render the security context constraints from the chart in the bundle.
- Your registry has a private project. If the registry requires authentication, you also have a user and token that can push to the project and pull from it.

Do step 0 and step 1 first. Then run the checklist that follows step 1. It proves each item above with a command, and it uses the values of step 1.

## 0. Get the two files

**What it is for:** to put the two files of the release on the administration host.

You need `trident-port-offline-$VERSION.tar.gz` (about 350 MB) and `trident-port-offline-$VERSION.tar.gz.sha256`, where `VERSION` is the release of this guide, set in the first line of the block below. The bundle comes from the link and the token your vendor gave you. Put both files in an empty working folder and `cd` into it.

A generic example. Set `BUNDLE_URL` to the link your vendor gave you (the token is typed, not shown). The same link serves the `.sha256` file:

```bash
export VERSION=v1.2.12                         # the release of this guide: do not edit it
read -rsp "Download token: " DL_TOKEN
echo
BUNDLE_URL="https://downloads.example.com/trident-port"   # replace with the link your vendor gave you
for f in "trident-port-offline-$VERSION.tar.gz" "trident-port-offline-$VERSION.tar.gz.sha256"
do
  curl -fL -H "Authorization: Bearer $DL_TOKEN" -o "$f" "$BUNDLE_URL/$f"
done
unset DL_TOKEN
ls -la "trident-port-offline-$VERSION.tar.gz"*
```

If your vendor gave you the files another way, skip the download but still paste the first line (`export VERSION=...`), because every later step reads `VERSION`.

Result: the folder holds two files, and the `.tar.gz` is about 350 MB. If `curl` prints `401` or `403`, the token is wrong or expired: ask your NetApp contact for a new one. If it prints `Could not resolve host`, the host cannot reach the link: ask for the files another way and skip this block.

## 1. Set your values (the only block you edit)

**What it is for:** to write down, once, your registry, your project and your StorageClasses. Every later block reads these values, so nothing else needs editing. `VERSION` is already set, in step 0, and you do not change it.

Change the right-hand side of each line to your own value, then paste the block. A private Artifactory registry, for example, uses `MIRROR_HOST=docker-local.artifactory.spimageworks.com` and the project `trident-port`, so `MIRROR` becomes `docker-local.artifactory.spimageworks.com/trident-port`.

Every later block starts with a line that checks the values it needs. Each block is wrapped in braces, with that check as its first line. If you open a new shell and paste a later block before you paste this one again, the whole block stops with a message such as `MIRROR: run step 1 first: MIRROR is not set`, and no line of the block runs.

```bash
export MIRROR_HOST=harbor.example.com          # your registry host, as the nodes see it
export MIRROR=$MIRROR_HOST/trident-port        # your registry host / your project
export REGISTRY_USER=robot-trident-port        # your registry user (any value if the registry has no login)
export TRACKING_SC=sc-nas-retain               # your StorageClass for the tracking volume
export DEST_SC=sc-nas-economy                  # your Trident StorageClass for the migrated volumes (only check 5 reads it: later steps do not use it)
export TRIDENT_NS=trident                      # namespace where Trident runs
read -rsp "Registry token for $REGISTRY_USER: " REGISTRY_TOKEN
echo
export REGISTRY_TOKEN
```

Result: the block prints only the token prompt. Type or paste the token and press Enter. Nothing appears while you type it, and that is normal. The prompt returns to the shell with no message. If you pasted the wrong token, paste the whole block again.

On OpenShift with the internal registry, do not type the token of your own login at this prompt: it expires in hours. Read the OpenShift section at the end first.

**Registry without authentication.** If your registry allows anonymous push and pull, set `REGISTRY_USER` to any value and type any value at the token prompt, for example `none`. Skip step 4. Run steps 6 and 8 as they are. The `ghcr` secrets are created with these values, and the registry does not check them. Step 8 is required, because the `TridentPort` resource of step 9 names the `ghcr` secret and the product pod does not start without it.

**Registry on plain HTTP.** This applies only to a registry without TLS. A registry with TLS, such as Harbor, needs none of it. containerd on the nodes pulls over plain HTTP only from `127.0.0.1:<port>`, or from a registry that is configured as an `http` mirror on every node. Configure the mirror before step 9:

- On RKE2, add the registry to `/etc/rancher/rke2/registries.yaml`, then restart `rke2-server` or `rke2-agent` on that node.

  ```yaml
  mirrors:
    "registry.example.com:5000":
      endpoint:
        - "http://registry.example.com:5000"
  ```

- On k3s, create `/var/lib/rancher/k3s/agent/etc/containerd/certs.d/registry.example.com:5000/hosts.toml`, then restart `k3s` or `k3s-agent` on that node.

  ```toml
  server = "http://registry.example.com:5000"

  [host."http://registry.example.com:5000"]
    capabilities = ["pull", "resolve"]
  ```

Use your registry address in place of `registry.example.com:5000`. It is the same address as `MIRROR_HOST`.

**`MIRROR_HOST` is the address the nodes use.** The manifests name the images by `MIRROR_HOST`, and the nodes pull them by that name. If the jump host reaches the registry by a different address (for example another port or a name that only the jump host resolves), set `MIRROR_HOST` to the address the nodes see. In step 5, set `PUSH` to the jump host address.

`DEST_SC` is only checked here: check 5 proves that the class exists. No later step of this guide uses it, because you choose the destination class when you plan a migration.

`TRIDENT_NS` and `TRACKING_SC` can name a Trident namespace and a `Retain` class that already exist. The install creates nothing for them.

Check the two cluster values (the checklist after this step repeats and widens them):

```bash
{
: "${TRACKING_SC:?run step 1 first: TRACKING_SC is not set}"
: "${TRIDENT_NS:?run step 1 first: TRIDENT_NS is not set}"
kubectl get sc "$TRACKING_SC" -o jsonpath='{.reclaimPolicy}{"\n"}'
kubectl get pods -n "$TRIDENT_NS" -l app=controller.csi.trident.netapp.io
}
```

Result: the first command prints `Retain`. The second lists one Trident controller pod `Running`. If the second command lists nothing, find the namespace with `kubectl get pods -A -l app=controller.csi.trident.netapp.io`, fix `TRIDENT_NS` and paste the block again.

## Before you start: the checklist

Run these checks now, one by one, in the same shell. Each one is read-only: it looks and changes nothing. Do not start step 2 until every check has its expected result.

### Check 1. Are you logged in as cluster-admin?

**What it is for:** to prove the commands will talk to the right cluster with enough rights.

```bash
kubectl config current-context
kubectl auth can-i '*' '*' --all-namespaces
```

Result: the first command prints the name of the cluster context this host talks to. The second prints `yes`. If the first prints `error: current-context is not set`, or the second cannot connect, log in first: on OpenShift with `oc login` (your OpenShift administrator gives you the command), on plain Kubernetes by pointing `KUBECONFIG` at your kubeconfig file. Then paste the block again. If the second prints `no`, you are not cluster-admin: ask a cluster-admin to log in on this host, or to run steps 7 and 8 for you. On OpenShift, `oc whoami` also prints your user name.

### Check 2. Is the cluster up and are its nodes ready?

**What it is for:** to prove the cluster can run new pods.

```bash
kubectl get nodes
```

Result: one row per node, and the column `STATUS` shows `Ready` on every row. If a node shows `NotReady`, tell your cluster administrator before you go on.

### Check 3. Does the host have the tools?

**What it is for:** to prove the host can run the commands of this guide.

```bash
for tool in kubectl oc helm sha256sum curl skopeo docker crane
do
  command -v "$tool" || echo "MISSING: $tool"
done
```

Result: one path line for each tool that exists. `kubectl`, `sha256sum` and `curl` must have a path. At least one of `skopeo`, `docker` and `crane` must have a path (step 5 explains `crane`). `oc` and `helm` are needed on OpenShift only: a `MISSING: oc` line on plain Kubernetes is fine. A `MISSING:` line for the others of `skopeo`, `docker` and `crane` is fine. If `kubectl` is missing and `oc` exists, run `alias kubectl=oc` in this terminal. The alias works only in this terminal window, so run it again if you open a new one.

### Check 4. Does the host reach your registry?

**What it is for:** to prove the registry address you set in step 1 answers, before you push 350 MB to it.

```bash
{
: "${MIRROR_HOST:?run step 1 first: MIRROR_HOST is not set}"
curl -sS -o /dev/null -w '%{http_code}\n' "https://$MIRROR_HOST/v2/"
}
```

Result: the command prints a number. `200` or `401` is the normal answer of a registry (`401` means it wants a login, which step 4 does). Any other three-digit number still means that the host reached the registry, and step 4 or step 5 is then the real test. `000` or an error line such as `Could not resolve host` means that the host does not reach it: check `MIRROR_HOST` in step 1, then the network or the proxy. For a registry on plain HTTP, replace `https` with `http` in this block, for this check only.

### Check 5. Do the StorageClasses exist?

**What it is for:** to prove that the tracking class and the destination class you set in step 1 exist, and that the tracking class keeps its volumes.

```bash
{
: "${TRACKING_SC:?run step 1 first: TRACKING_SC is not set}"
: "${DEST_SC:?run step 1 first: DEST_SC is not set}"
kubectl get sc "$TRACKING_SC" "$DEST_SC"
kubectl get sc "$TRACKING_SC" -o jsonpath='{.reclaimPolicy}{"\n"}'
}
```

Result: the first command prints two rows, one for each class. The second prints `Retain`. If a row is missing with `NotFound`, list what exists with `kubectl get sc`, correct step 1 and paste step 1 again. If the second prints `Delete`, pick another class for `TRACKING_SC`: with a `Delete` class the migration records go when the volume goes.

### Check 6. Is Trident running?

**What it is for:** to prove that Trident, which creates the destination volumes, is installed and running.

```bash
{
: "${TRIDENT_NS:?run step 1 first: TRIDENT_NS is not set}"
kubectl get pods -n "$TRIDENT_NS"
}
```

Result: a row `trident-controller-...` with every container ready (for example `6/6`) and `STATUS` `Running`, and one `trident-node-linux-...` row per node, also `Running`. If the command prints `No resources found`, Trident is not in `$TRIDENT_NS`: find it with `kubectl get pods -A | grep trident-controller`, correct step 1 and paste step 1 again. If Trident is not installed at all, stop: install Trident first, following the Trident installation guide for OpenShift. This guide does not install it.

## 2. Verify and extract the bundle

**What it is for:** to prove the bundle arrived whole and unchanged, then to unpack it.

```bash
{
: "${VERSION:?run step 0 first: VERSION is not set}"
sha256sum -c "trident-port-offline-$VERSION.tar.gz.sha256"
}
```

Result: the command prints the bundle name followed by `: OK`. If it prints `FAILED` or `No such file or directory`, the download is damaged or you are in the wrong folder: go back to step 0 and download again.

```bash
{
: "${VERSION:?run step 0 first: VERSION is not set}"
tar xzf "trident-port-offline-$VERSION.tar.gz" && cd "trident-port-offline-$VERSION"
sha256sum -c SHA256SUMS
}
```

Result: the command prints one `OK` line per file (`images/...oci.tar`, `install.yaml`, `IMAGES.txt` and the rest) and no `FAILED`. Stop if there is one.

```bash
cat IMAGES.txt
```

Result: the file shows two comment lines (source reference and `# platform: linux/amd64`), then three lines of the form `reference sha256:digest images/file.oci.tar`. They name `trident-port` and `trident-port-operator`, both at the version you set in step 1, and `curlimages/curl:8.11.0`.

## 3. Set the Trident namespace (only if it is not `trident`)

**What it is for:** to tell the install in which project Trident runs, when it is not the usual `trident`. If it is `trident`, the block does nothing.

```bash
{
: "${TRIDENT_NS:?run step 1 first: TRIDENT_NS is not set}"
if [ "$TRIDENT_NS" != trident ]
then
  sed -i "s/^  namespace: trident$/  namespace: $TRIDENT_NS/" install.yaml
  sed -i "s/^spec:$/spec:\n  config:\n    TRIDENT_NAMESPACE: $TRIDENT_NS/" examples/tridentport-*.yaml
  grep -c "namespace: $TRIDENT_NS$" install.yaml
fi
}
```

Result: the block prints nothing when Trident runs in `trident`. Otherwise it prints `2`.

The install accepts an existing Trident namespace as it is. It creates and changes nothing in it.

## 4. Log in to your registry

**What it is for:** to let this host push images to your registry.

Run this step only if your registry requires login. With a registry that allows anonymous push and pull, skip it.

```bash
{
: "${MIRROR_HOST:?run step 1 first: MIRROR_HOST is not set}"
: "${REGISTRY_USER:?run step 1 first: REGISTRY_USER is not set}"
: "${REGISTRY_TOKEN:?run step 1 first: REGISTRY_TOKEN is not set}"
echo "$REGISTRY_TOKEN" | skopeo login "$MIRROR_HOST" -u "$REGISTRY_USER" --password-stdin
}
```

Result: `skopeo` prints `Login Succeeded!`. If it prints `unauthorized` or `invalid username/password`, the user or the token of step 1 is wrong: paste step 1 again with the right ones. If it prints `x509: certificate signed by unknown authority`, this host does not trust the registry certificate: ask your registry administrator for the CA file, or see the `TLS_VERIFY=false` note in step 5. With `crane`, the login is in step 5, inside the `crane` block: skip this step. With docker, run `echo "$REGISTRY_TOKEN" | docker login "$MIRROR_HOST" -u "$REGISTRY_USER" --password-stdin` instead.

If the jump host pushes through a different address than `MIRROR_HOST` (`PUSH` in step 5), log in to that address.

## 5. Load the images

**What it is for:** to copy the three images from the bundle into your registry, and to check that each one arrived intact.

`PUSH` is the address this host pushes to. Leave it unset when it is the same as `MIRROR`. Set it before the block when this host reaches the registry at an address that the cluster nodes cannot use, for example `export PUSH=localhost:5000/trident-port` behind a port-forward. `MIRROR` stays the address that the nodes pull from, and it is the one written into the manifests.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
export PUSH=${PUSH:-$MIRROR}
REGISTRY="$MIRROR" PUSH="$PUSH" ./load-images.sh
}
```

Result: the script prints one `== source -> destination` block per image, then `Loaded 3 images:` with `$MIRROR/trident-port`, `$MIRROR/trident-port-operator` (your version) and `$MIRROR/curl:8.11.0`. The script exits with a non-zero code if a pushed digest differs from `IMAGES.txt`. If it stops with `unauthorized`, repeat step 4. If it stops with a certificate error, see the next paragraph.

If the registry uses a certificate that this host does not trust, or is served over plain HTTP, run `TLS_VERIFY=false REGISTRY="$MIRROR" PUSH="$PUSH" ./load-images.sh`.

### If the host has no skopeo, docker or sudo

`load-images.sh` needs `skopeo` or `docker`. Without them, use the static `crane` binary, which runs without root. It comes from https://github.com/google/go-containerregistry/releases (file `go-containerregistry_Linux_x86_64.tar.gz`, version v0.20.2 or later). The bundle does not carry `crane`. On a host with no internet access, copy the `crane` binary onto the host before you start, into the bundle folder or into a folder on your `PATH`. The block below downloads it only when no `crane` is found. A static `skopeo` build also works, with the same `skopeo copy` command that `load-images.sh` runs.

For a registry served over plain HTTP, or with a certificate that this host does not trust, run `export CRANE_FLAGS=--insecure` before the block. Leave it unset otherwise.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
: "${REGISTRY_USER:?run step 1 first: REGISTRY_USER is not set}"
: "${REGISTRY_TOKEN:?run step 1 first: REGISTRY_TOKEN is not set}"
export PATH="$PWD:$PATH"
command -v crane >/dev/null || {
  curl -fL -o gcr.tar.gz https://github.com/google/go-containerregistry/releases/download/v0.20.2/go-containerregistry_Linux_x86_64.tar.gz
  tar xzf gcr.tar.gz crane && chmod +x crane
}
export PUSH=${PUSH:-$MIRROR}
export CRANE_FLAGS=${CRANE_FLAGS:-}
echo "$REGISTRY_TOKEN" | crane auth login "${PUSH%%/*}" -u "$REGISTRY_USER" --password-stdin   # only if the registry requires login
grep -v '^#' IMAGES.txt | while read -r ref digest archive
do
  dest="$PUSH/${ref##*/}"
  rm -rf oci-layout && mkdir oci-layout && tar xf "$archive" -C oci-layout
  crane push $CRANE_FLAGS oci-layout "$dest"
  [ "$(crane digest $CRANE_FLAGS "$dest")" = "$digest" ] && echo "OK $dest" || echo "DIGEST MISMATCH $dest"
done
rm -rf oci-layout
}
```

Result: the loop prints one `OK` line per image, three in total. Each line means that the digest in the registry is the one that `IMAGES.txt` lists. Stop if any line says `DIGEST MISMATCH`.

To copy an image that is already in one registry into another one, for example from the push address to the address the nodes use, run `crane copy --insecure <source> <destination>` (drop `--insecure` with TLS). Then check the digest with `crane digest --insecure <destination>` against `IMAGES.txt`.

## 6. Create the operator namespace and pull secret

**What it is for:** to create the project of the operator and the secret that lets the cluster pull the images from your registry.

The secret name stays `ghcr`. It holds your registry credential.

```bash
{
: "${MIRROR_HOST:?run step 1 first: MIRROR_HOST is not set}"
: "${REGISTRY_USER:?run step 1 first: REGISTRY_USER is not set}"
: "${REGISTRY_TOKEN:?run step 1 first: REGISTRY_TOKEN is not set}"
kubectl create namespace trident-port-system --save-config
kubectl -n trident-port-system create secret docker-registry ghcr \
  --docker-server="$MIRROR_HOST" --docker-username="$REGISTRY_USER" --docker-password="$REGISTRY_TOKEN"
}
```

Result: the commands print `namespace/trident-port-system created` and `secret/ghcr created`. If you see `AlreadyExists`, a previous attempt created them: that is fine for the namespace. For the secret, delete it with `kubectl -n trident-port-system delete secret ghcr` and paste the block again.

With a registry that allows anonymous pull, run both commands with the values from step 1. The registry does not check them.

## 7. Install the operator from your registry

**What it is for:** to install the operator, the program that installs and looks after Trident Port, taking its images from your registry. On OpenShift, read "On OpenShift" below before you paste the first block. This step needs cluster-admin.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
sed -e "s#ghcr.io/calzdevops/#$MIRROR/#" -e "s#image: curlimages/curl#image: $MIRROR/curl#" install.yaml > install.mirror.yaml
grep -E '^\s+image: \S' install.mirror.yaml
}
```

Result: `grep` prints exactly two lines, `image: $MIRROR/curl:8.11.0` and `image: $MIRROR/trident-port-operator:` followed by your version, with your registry in place of `$MIRROR`. No line names `ghcr.io`.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
: "${VERSION:?run step 0 first: VERSION is not set}"
FILE_OK=$(grep -qxE " *image: $MIRROR/trident-port-operator:$VERSION" install.mirror.yaml && echo ok)
: "${FILE_OK:?install.mirror.yaml does not name $MIRROR/trident-port-operator:$VERSION: paste the sed block of this step again}"
kubectl apply -f install.mirror.yaml
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
}
```

Result: `kubectl` prints `created` for the CRD, ClusterRole, ClusterRoleBinding, ServiceAccount and Deployment, `namespace/trident-port created` and `namespace/trident-port-system configured`. On a reinstall the `trident-port` namespace already exists, and the line reads `namespace/trident-port configured`: both are correct, and the objects that did not change print `unchanged`. Then it prints `deployment "trident-port-operator" successfully rolled out`.

If the command does not return, run `kubectl -n trident-port-system get pods`. A pod in `Init:Error` names the missing secret. A pod in `ImagePullBackOff` or `ErrImagePull` cannot pull from your registry: run `kubectl -n trident-port-system describe pod -l app.kubernetes.io/name=trident-port-operator` and read the `Events` at the end for the reason (a wrong secret, a certificate the cluster does not trust, or an address the nodes cannot reach).

### On OpenShift

**What it is for:** to start the operator on OpenShift, whose security rules refuse the plain file. Use this block in place of the first block above, then run the `kubectl apply` block above.

Do not use the commands above. On OpenShift, the operator pod is refused with `unable to validate against any security context constraint` and `.initContainers[0].runAsUser: Invalid value: 100`. The init container `check-pull-secret` pins that user, and `restricted-v2` admits only the range of the namespace. Run this `sed` in place of the first one. It also drops the `runAsUser: 100` line, so OpenShift assigns the user itself:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
sed -e "s#ghcr.io/calzdevops/#$MIRROR/#" -e "s#image: curlimages/curl#image: $MIRROR/curl#" -e '/^ *runAsUser: 100$/d' install.yaml > install.mirror.yaml
grep -E '^\s+image: \S' install.mirror.yaml
grep -c 'runAsUser' install.mirror.yaml
}
```

Result: `grep` prints the same two image lines as above. The last command prints `0`. Then run the two `kubectl apply` and `rollout status` commands above. The operator starts under `restricted-v2`, with no `anyuid` and no extra constraint. The `curl` image runs with any user ID, and it needs no home directory.

If you install through OLM, the bundle already has no fixed user. This step does not apply.

## 8. Create the product pull secret

**What it is for:** to give the project of the product the secret that lets it pull its image from your registry.

`install.yaml` already created the `trident-port` namespace. Create the secret there, before the `TridentPort` in step 9:

```bash
{
: "${MIRROR_HOST:?run step 1 first: MIRROR_HOST is not set}"
: "${REGISTRY_USER:?run step 1 first: REGISTRY_USER is not set}"
: "${REGISTRY_TOKEN:?run step 1 first: REGISTRY_TOKEN is not set}"
kubectl -n trident-port create secret docker-registry ghcr \
  --docker-server="$MIRROR_HOST" --docker-username="$REGISTRY_USER" --docker-password="$REGISTRY_TOKEN"
kubectl -n trident-port get secret ghcr
}
```

Result: `kubectl` prints `secret/ghcr created`, then one row `ghcr` of type `kubernetes.io/dockerconfigjson`. If it prints `namespace "trident-port" not found`, step 7 did not finish: repeat its `kubectl apply` block. Without this secret, the product pod stays in `Init:CrashLoopBackOff` and the log says `secret(s) ghcr not found`. On a reinstall the secret may still exist from the first install, and `create` then prints `Error from server (AlreadyExists): secrets "ghcr" already exists`. Delete it with `kubectl -n trident-port delete secret ghcr` and paste the block again, as in step 6.

With a registry that allows anonymous pull, run this step too, with the values from step 1. The `TridentPort` resource of step 9 names the `ghcr` secret, so the product pod needs it even when the registry does not check it.

If you reinstall and `kubectl get ns trident-port` shows `Terminating`, wait until the namespace is gone and repeat from step 7.

### 8a. On OpenShift: create the two security context constraints that Trident Port needs

**What it is for:** OpenShift refuses pods that run with extra rights unless a security context constraint (SCC) allows them. These two let the copy pods mount the source NFS export and let `tp import` and the Block and NAS conversions run. Skip 8a to 8c on plain Kubernetes.

Who runs it: a cluster-admin, once per cluster, here, before step 9. The operator cannot create SCCs, which is why the product is told `create: false` in step 9. Run it in the bundle folder, in the same shell. It needs `helm` 3.8 or later. It renders the SCCs from the chart inside this bundle, not from the online chart.

```bash
{
: "${VERSION:?run step 0 first: VERSION is not set}"
helm template trident-port "chart/trident-port-${VERSION#v}.tgz" --namespace trident-port \
  --api-versions security.openshift.io/v1 --show-only templates/scc-copy-nfs.yaml | oc apply -f -
helm template trident-port "chart/trident-port-${VERSION#v}.tgz" --namespace trident-port \
  --api-versions security.openshift.io/v1 --show-only templates/scc-nfs-import.yaml | oc apply -f -
}
```

Result: `oc apply` prints `created` for each object: the first command creates two ClusterRoles, one ClusterRoleBinding and the SCC `trident-port-copy-nfs`, and the second creates the SCC `trident-port-nfs-import`. If it prints `Forbidden`, you are not cluster-admin: ask a cluster-admin to paste this block. If it prints `helm: command not found`, install `helm` on this host (see check 3). If it prints `no such file` for the chart, you are not in the bundle folder: `cd` into `trident-port-offline-$VERSION` and paste the block again.

### 8b. On OpenShift: create the Block Mirror constraint (optional)

**What it is for:** Block Mirror is an optional feature. Only if you will use it, this block creates its third constraint. If you will not use it, skip this block: you can run it later, with the same command.

```bash
{
: "${VERSION:?run step 0 first: VERSION is not set}"
helm template trident-port "chart/trident-port-${VERSION#v}.tgz" --namespace trident-port \
  --api-versions security.openshift.io/v1 --show-only templates/scc-blockmirror-node.yaml | oc apply -f -
}
```

Result: `oc apply` prints `created` for two ClusterRoles, one ClusterRoleBinding and the SCC `trident-port-blockmirror-node`. If you skip the block, nothing else in this guide changes.

### 8c. On OpenShift: check that the constraints took

**What it is for:** to prove the objects 8a (and 8b) created exist in the cluster.

```bash
oc get scc | grep trident-port
oc get clusterrole,clusterrolebinding | grep trident-port
```

Result: the first command lists `trident-port-copy-nfs` and `trident-port-nfs-import`, and `trident-port-blockmirror-node` only if you ran 8b. The second lists the ClusterRoles `trident-port-copy-nfs-use` and `trident-port-copy-nfs-bind`, and the ClusterRoleBinding `trident-port-copy-nfs-bind`. With 8b it also lists `trident-port-blockmirror-node-use`, `trident-port-blockmirror-bind` and the ClusterRoleBinding `trident-port-blockmirror-bind`. If the first command prints nothing, 8a did not run: paste it again. `UNINSTALL.md` removes these objects by name.

If you do not apply the first of these (`scc-copy-nfs.yaml`), the copy pods need the `anyuid` constraint in every namespace that you migrate. `tp live-sync start` checks this before it starts. For each namespace where the copy pods are not admitted, it prints `COPY PODS NOT ADMITTED` and the exact `oc create rolebinding ...` command that grants `anyuid` to the `trident-port-copy` ServiceAccount. Run that command as printed, then run the start again. Those namespaces are left out of the start until then.

## 9. Install the product from your registry

**What it is for:** to create the `TridentPort` resource, which tells the operator to install the product, from your registry and with your tracking class. On OpenShift, also paste the OpenShift block below, before the `kubectl apply`.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
: "${TRACKING_SC:?run step 1 first: TRACKING_SC is not set}"
: "${VERSION:?run step 0 first: VERSION is not set}"
sed -e "s/[<]version[>]/$VERSION/" -e "s/[<]tracking-sc[>]/$TRACKING_SC/" \
  -e "s#ghcr.io/calzdevops/#$MIRROR/#" -e "s#image: curlimages/curl#image: $MIRROR/curl#" \
  -e 's/^spec:$/spec:\n  copyImage:\n    pullPolicy: IfNotPresent/' \
  examples/tridentport-minimal.yaml > tridentport.mirror.yaml
grep -E 'repository:|tag:|image: |accessMode|pullPolicy' tridentport.mirror.yaml
}
```

Result: `grep` prints `repository: $MIRROR/trident-port`, `tag:` with the version you set in step 1, `image: $MIRROR/curl:8.11.0` (your registry in place of `$MIRROR`) and `pullPolicy: IfNotPresent`. With `IfNotPresent`, the copy pods use the image that is already on the node. No line names `ghcr.io` or `curlimages`.

For a NAS (ReadWriteMany) tracking class, the recommended case, enable `accessMode: ReadWriteMany`:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
: "${VERSION:?run step 0 first: VERSION is not set}"
FILE_OK=$(grep -qE "^ *repository: $MIRROR/trident-port\$" tridentport.mirror.yaml \
  && grep -qE "^ *tag: $VERSION( |\$)" tridentport.mirror.yaml && echo ok)
: "${FILE_OK:?tridentport.mirror.yaml does not name $MIRROR/trident-port:$VERSION: paste the first block of step 9 again}"
sed -i 's/^\(\s*\)# *accessMode: ReadWriteMany/\1accessMode: ReadWriteMany/' tridentport.mirror.yaml
grep -n 'accessMode' tridentport.mirror.yaml
}
```

Result: `grep` prints a line `accessMode: ReadWriteMany` without `#`. Skip this block for a ReadWriteOnce class.

**On OpenShift**, tell the product that it runs there. Without this, its pods are refused with `unable to validate against any security context constraint` and `runAsUser: 100`. The three `create: false` lines are there because you created the constraints yourself in step 8. Skip this block on plain Kubernetes.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
: "${VERSION:?run step 0 first: VERSION is not set}"
FILE_OK=$(grep -qE "^ *repository: $MIRROR/trident-port\$" tridentport.mirror.yaml \
  && grep -qE "^ *tag: $VERSION( |\$)" tridentport.mirror.yaml && echo ok)
: "${FILE_OK:?tridentport.mirror.yaml does not name $MIRROR/trident-port:$VERSION: paste the first block of step 9 again}"
sed -i 's/^spec:$/spec:\n  openshift:\n    enabled: true\n    copyNfsScc:\n      create: false\n    nfsImportScc:\n      create: false\n    blockMirrorScc:\n      create: false/' tridentport.mirror.yaml
grep -n -A7 'openshift:' tridentport.mirror.yaml
}
```

Result: `grep` prints `openshift:`, then `enabled: true`, then the three names `copyNfsScc:`, `nfsImportScc:` and `blockMirrorScc:`, each followed by `create: false`. If it prints nothing, the file has no `spec:` line on its own: start step 9 again from its first block.


**Argo CD in another cluster (optional).** If an Argo CD hub that runs in another cluster manages this one, give the product its kubeconfig, or the product cannot suspend the Applications and the hub puts a stopped workload back. Create a Secret in the `trident-port` namespace that holds the hub kubeconfig (the Secret holds it under the key `kubeconfig`), and name it in the `TridentPort` under `spec.argocdHub` (add it under `spec:` in `tridentport.mirror.yaml`, which the next block applies). The fields are `existingSecret` (the Secret name, required for this feature), `key` (the key in the Secret, default `kubeconfig`), `context` (a context inside the kubeconfig, optional) and `namespace` (where Argo CD runs, default `argocd`). Without `existingSecret` the product pod has no Argo CD settings, and the guide behaves as it does today. Create the Secret before you apply the resource; the example `examples/tridentport-private-registry.yaml` in the bundle carries the block commented out. The hub account needs only `get`, `list` and `patch` on `applications` and `applicationsets`: see "Argo CD in another cluster" in `MIGRATE-A-CLUSTER.md`.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
kubectl -n trident-port create secret generic argocd-hub --from-file=kubeconfig=/path/to/hub.kubeconfig
}
```

Result: `kubectl` prints `secret/argocd-hub created`. If it prints `AlreadyExists`, run `kubectl -n trident-port delete secret argocd-hub` and paste the block again. Skip this block when Argo CD runs in this cluster.

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
: "${VERSION:?run step 0 first: VERSION is not set}"
FILE_OK=$(grep -qE "^ *repository: $MIRROR/trident-port\$" tridentport.mirror.yaml \
  && grep -qE "^ *tag: $VERSION( |\$)" tridentport.mirror.yaml && echo ok)
: "${FILE_OK:?tridentport.mirror.yaml does not name $MIRROR/trident-port:$VERSION: paste the first block of step 9 again}"
kubectl apply -f tridentport.mirror.yaml
kubectl -n trident-port get tridentport trident-port
}
```

Result: `kubectl` prints `tridentport.trident-port.netapp.io/trident-port created`, then the column `DEPLOYED` shows `True`. Repeat the `get` until it does. If it stays `False` for more than two minutes, run `kubectl -n trident-port describe tridentport trident-port` and read the `Message`.

```bash
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
```

Result: `kubectl` prints `deployment "trident-port" successfully rolled out`. The webhook pod (`trident-port-webhook-...`) can stay in `ContainerCreating` for up to about 7 minutes. It starts by itself once the product pod is Ready. If the product pod stays `Pending` with `pod has unbound immediate PersistentVolumeClaims` after the tracking claim is `Bound`, delete the pod (`kubectl -n trident-port delete pod <pod>`). The ReplicaSet creates it again and it starts.

The copy pods run the same product image from your registry. The product copies its own pull secret into each application namespace for the run and removes it afterwards.

## 10. Apply the license

**What it is for:** to give the product the license that NetApp issued for this cluster, so it can run migrations.

Get the cluster ID to send to NetApp:

```bash
kubectl get ns kube-system -o jsonpath='{.metadata.uid}{"\n"}'
```

Result: the command prints a UUID. NetApp sends you a token for that ID. Save the token as `license.txt` in this folder. It is one line that starts with `eyJ`.

```bash
NODE_PORT=$(kubectl -n trident-port get svc trident-port -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
API_TOKEN=$(kubectl -n trident-port get secret trident-port-api-token -o jsonpath='{.data.token}' | base64 -d)
echo "http://$NODE_IP:$NODE_PORT"
```

Result: the block prints the web address, `http://` followed by the node IP and port. If the secret returns `NotFound`, the product pod is not Ready yet. Repeat the `rollout status` command of step 9. The web `NodePort` is chosen by Kubernetes when the service is created, so after a reinstall it is usually another number: use the one this block prints, and tell whoever has the old address bookmarked or a firewall rule on it. To keep a fixed port, add `service:` with `nodePort: <port>` under `spec:` in the `TridentPort` of step 9 (the port must be inside the cluster's NodePort range, 30000-32767 by default) and apply it again; the operator changes the service. Not measured on a cluster.

```bash
curl -s -X POST "http://$NODE_IP:$NODE_PORT/api/license" -H "X-Trident-Port-Token: $API_TOKEN" \
  -H 'Content-Type: application/json' -d "{\"token\":\"$(cat license.txt)\"}"
```

Result: the API returns JSON with `"valid": true`. You can also paste the same token in Settings > License. If it says `"valid": false`, the token is for another cluster or has expired: send the cluster ID above to your NetApp contact and ask for a new token.

## 11. Verify the installation

**What it is for:** to prove that the product runs, answers, and has its tracking volume.

```bash
kubectl -n trident-port get pods
kubectl -n trident-port exec deploy/trident-port -c trident-port -- curl -s http://localhost:8000/api/health
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version
kubectl -n trident-port get pvc trident-port-tracking
```

Result:

- The pods `trident-port-...` and `trident-port-webhook-...` are both `1/1 Running`.
- The health check returns `{"status":"ok", ...}`.
- `tp --version` prints a line with the version you set in step 1.
- The tracking claim shows `STATUS` = `Bound` and `STORAGECLASS` = your tracking class.

Open the web address from step 10. Settings > Product version shows the version you set in step 1. Settings > License shows the tier and the expiry date. The web asks for the API token on the first request. To print it:

```bash
kubectl -n trident-port get secret trident-port-api-token -o jsonpath='{.data.token}' | base64 -d
echo
```

Result: the command prints a token string.

## 12. Check that no pod pulls from outside your registry

**What it is for:** to prove that the cluster never needs the internet to run Trident Port.

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | grep -E '/trident-port(-operator)?:|/curl:' | sort -u
kubectl -n trident-port logs deploy/trident-port -c check-pull-secret
```

Result: every line starts with your registry, and no line names `curlimages/curl` or `ghcr.io/`. The log has two lines:

- `pull secret check skipped: secret 'ghcr' is not a ghcr.io credential (registry: <your registry host> )`
- `pull secret(s) 'ghcr' found in namespace 'trident-port' -- continuing`

If a line names `ghcr.io/` or `curlimages/curl`, a `sed` of step 7 or step 9 was skipped: repeat that step.

## Next step: register the ONTAP array

The install is done. Before the first migration, register the ONTAP array as in step 0 of [`MIGRATE-A-CLUSTER.md`](MIGRATE-A-CLUSTER.md#0-before-you-begin) (the "Register the ONTAP array" item).

Without it, the product cannot raise the inode limit of the destination volume, and a volume with many files fails with `[ERROR] data: source holds N files and the destination volume allows M ... the copy would fail with 'No space left on device'`.

The account that Trident Port uses for the array also needs read access to the aggregates (`GET /storage/aggregates`), and to the SVM's aggregate list (`GET /svm/svms`), because the pre-flight reads them to check that the destination has room. An account limited to one SVM with the `vsadmin` role was refused on `/storage/aggregates` with HTTP 403 in a measurement, so give the account a role that can read aggregates. Without it the pre-flight cannot check the free space of the aggregate and the line stays at WARN. This is a permission on the array, not on Kubernetes. The array calls of each mode are in [`ontap-account-per-mode.md`](ontap-account-per-mode.md), in the bundle folder.

## Speed analysis after the install

About two minutes after the product serves, Trident Port measures the speed of your storage once, in the background, so the first estimate already rests on a measurement. It creates small scratch volumes of its own in the namespace of the product, one at a time, labelled `trident-port-speed-analysis`, and deletes them. It never blocks or fails the install. When it cannot measure (no Trident class yet, no permission, no room) it says why in Settings > Speed analysis, and you can run it again there or with `tp speed-analyze`. To turn it off, add this to the `spec` of the `TridentPort` resource and apply it again:

```yaml
spec:
  speedAnalysis:
    onInstall: false
```

## 13. Upgrade to a new version

**What it is for:** to move an installed Trident Port to a newer release, operator first, then the product. Skip this step on a first install.

With the operator, the `TridentPort` resource decides the version. Do not run `helm upgrade` on this release. The operator puts the old tag back at its next reconcile, and the chart refuses it. Do not upgrade while a cutover is running.

With the new release's bundle, run steps 0 to 5 with the new `VERSION` in step 1.

The RBAC that the operator carries includes `patch` on `pods`. The product uses it only to put the label `skipPreStopChecks` on the pods of a RabbitMQ with more than one replica before it stops them, and to remove it. Without it, a cutover of such a RabbitMQ is refused with `refusing to stop RabbitmqCluster ... could not put the label skipPreStopChecks` and nothing is stopped.

The RBAC that the operator carries also grants `patch` and `update` on the custom resources that own an application workload and are not on the list of supported operators, such as `clickhousekeeperinstallations`. The product writes the note it needs to restore the application on the custom resource that owns a workload. Without the verbs, the pre-flight refuses with `the restore note would be refused on clickhousekeeperinstallation/...: cannot patch` and nothing is stopped. It also grants `list` on `csidrivers` and `get`, `list` and `watch` on `limitranges`, read only, which the pre-flight uses and would otherwise report as unknown. Upgrade the operator first, because it carries the RBAC that the new product needs.

Check that the new pods fit before you upgrade. The old pod stops first (both Deployments use `Recreate`: the product mounts one ReadWriteOnce volume, and two operator pods must never reconcile at the same time), so a new pod whose CPU request fits on no node stays `Pending` with `Insufficient cpu` and nothing runs until you fix it. The new pod can take the room the old one frees, so this only happens when its request is larger than the old one or another pod took the room. The default requests are `10m` CPU / `128Mi` for the operator and `50m` CPU / `512Mi` for the product. Look at the requests already booked on each node:

```bash
kubectl describe nodes | grep -E '^Name:|^  cpu '
```

Result: one `cpu` line per node, with the requests and their share of the allocatable CPU. A node at 99% has little room left. If no node can take the request, lower it before you upgrade:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
kubectl -n trident-port-system patch deploy trident-port-operator -p '{"spec":{"template":{"spec":{"containers":[{"name":"manager","resources":{"requests":{"cpu":"10m"}}}]}}}}'
kubectl -n trident-port patch tridentport trident-port --type merge -p '{"spec":{"resources":{"requests":{"cpu":"10m"}}}}'
}
```

Result: the first command prints `deployment.apps/trident-port-operator patched` and the second prints `tridentport.trident-port.netapp.io/trident-port patched`. The first is for the operator pod, the second for the product pod. If a pod is already stuck, `kubectl -n trident-port-system get events --field-selector reason=FailedScheduling` (and the same with `-n trident-port`) shows `Insufficient cpu`; the same commands unblock it. The operator install file sets the operator request again each time you apply it, so lower it again after an operator upgrade if the node is still full.

Upgrade the operator first, because it carries the RBAC that the new product needs:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
sed -e "s#ghcr.io/calzdevops/#$MIRROR/#" -e "s#image: curlimages/curl#image: $MIRROR/curl#" install.yaml > install.mirror.yaml
grep -E '^\s+image: \S' install.mirror.yaml
kubectl apply -f install.mirror.yaml
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
}
```

Result: the two `image:` lines name your registry and the new version. `kubectl apply` prints `deployment.apps/trident-port-operator configured`. The other objects print `unchanged`, unless the new release changes them. Then `rollout status` prints `successfully rolled out`.

On OpenShift, use the `sed` of the OpenShift part of step 7 in place of the first command.

Then upgrade the product:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
: "${VERSION:?run step 0 first: VERSION is not set}"
kubectl -n trident-port patch tridentport trident-port --type merge \
  -p "{\"spec\":{\"image\":{\"repository\":\"$MIRROR/trident-port\",\"tag\":\"$VERSION\"}}}"
echo "Waiting for the operator to give the product Deployment the image $MIRROR/trident-port:$VERSION (up to 300 s)"
SWITCHED=no
for i in $(seq 1 60)
do
  if kubectl -n trident-port get deploy trident-port -o jsonpath='{range .spec.template.spec.containers[*]}{.image}{"\n"}{end}' | grep -qxF "$MIRROR/trident-port:$VERSION"
  then
    SWITCHED=yes
    break
  fi
  sleep 5
done
if [ "$SWITCHED" = yes ]
then
  kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
  kubectl -n trident-port get tridentport trident-port
  GOT=$(kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp --version)
  echo "$GOT"
  if echo "$GOT" | grep -qF "$VERSION"
  then
    echo "UPGRADED: the product runs $VERSION"
  else
    echo "NOT UPGRADED: tp --version does not print $VERSION. Run: kubectl -n trident-port describe tridentport trident-port"
  fi
else
  echo "NOT UPGRADED: after 300 s the Deployment still runs another image. Run: kubectl -n trident-port describe tridentport trident-port"
fi
}
```

Result: `kubectl` prints `patched`. The block then waits, because the operator needs a few seconds to change the Deployment, and a `rollout status` run at once reads the old Deployment as complete and prints the old version. When the Deployment names the new image the block prints `successfully rolled out`, the `DEPLOYED` column (`True` once the operator has finished, so repeat the `get` until it does) and the `tp --version` line, and it ends with `UPGRADED: the product runs` followed by the new version. If it ends with `NOT UPGRADED`, read the `Message` of the resource, as the line says. The web is unavailable for about a minute while the pod restarts.

## 14. After the last namespace of a cluster

**What it is for:** to make the Trident class the default, so that new claims land on it. Skip this step until the last namespace of the cluster has been migrated.

New claims that name no StorageClass are created on the default class. Make the Trident class the default. Set the two names first (see `kubectl get sc`):

```bash
kubectl get sc
export OLD_DEFAULT_SC=local-path                # the class marked (default) today
export TRIDENT_DEFAULT_SC=sc-nas                # the Trident class that becomes the default
kubectl patch sc "$OLD_DEFAULT_SC" -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch sc "$TRIDENT_DEFAULT_SC" -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc
```

Result: only the Trident class shows `(default)`.

## No registry

Use this section for a cluster that reaches no registry. Skip steps 4 and 5 and the `sed` rewrites of `install.yaml` and `tridentport.mirror.yaml`. The manifests keep the `ghcr.io/calzdevops/` names, and the pods use the image that is already on the node (`IfNotPresent`). Without a `ghcr` secret in `trident-port-system`, the pull-secret check of the operator prints `no pull secret 'ghcr': images must already be on the nodes or pullable without a login` and continues. The product pod still needs a `ghcr` secret in the `trident-port` namespace, because `tridentport-minimal.yaml` names it. Create it as in step 8, with any values.

On k3s and RKE2, import the three images into the containerd of each node. Run this on every node, in the bundle folder:

```bash
for f in images/*.oci.tar
do
  sudo ctr -n k8s.io images import "$f"
done
sudo ctr -n k8s.io images ls -q | grep -E 'trident-port|curl'
```

On k3s, use `sudo k3s ctr` in place of `sudo ctr`. On RKE2, `ctr` is not on the `PATH` and needs the socket of its containerd, so run this block in place of the one above:

```bash
CTR="sudo /var/lib/rancher/rke2/bin/ctr --address /run/k3s/containerd/containerd.sock"
for f in images/*.oci.tar
do
  $CTR -n k8s.io images import "$f"
done
$CTR -n k8s.io images ls -q | grep -E 'trident-port|curl'
```

Result: the list shows the three references from `IMAGES.txt`.

Then apply `install.yaml` (step 7) and `examples/tridentport-minimal.yaml` with the version and class set (step 9), with `spec.copyImage.pullPolicy: IfNotPresent`.

On OpenShift, you cannot load the nodes one by one, so a registry is needed. You can use the internal registry with `managementState: Managed` and storage:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge \
  -p '{"spec":{"managementState":"Managed","storage":{"emptyDir":{}}}}'
```

Or you can use an external registry: follow steps 4 to 9 with `MIRROR_HOST` set to it.

With `emptyDir` storage, the internal registry loses every image when its pod restarts. The pods cannot pull their images the next time they start. To keep the images, give the registry a volume before step 5 and load the images again:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"storage":{"pvc":{"claim":""}}}}'
```

## OpenShift

### Internal registry

If `MIRROR_HOST` is the internal registry of OpenShift, two things go before the steps that use it.

The project in `MIRROR` has to exist before the image push of step 5. Create it after step 4:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
oc new-project "${MIRROR#*/}"
}
```

Result: `oc` prints `Now using project`. If the project exists, it prints `already exists`, and you can go on.

**Do not build the pull secret from your own login.** The token of your `oc` session (the one `oc whoami` is logged in with, prefix `sha256~`) lives a few hours. A pull secret made from it works on the day and fails later: the pod that starts after it expired stays in `Init:ImagePullBackOff` with `unable to retrieve auth token: invalid username/password`. The operator Deployment replaces its pod by stopping the old one first, so no operator runs until the secret is fixed. Pods that already run keep going on the cached image, which hides the problem until a node restarts. Use the token of a dedicated ServiceAccount in the images project for `REGISTRY_USER` and `REGISTRY_TOKEN` in steps 6 and 8. Create it once. Set `REGISTRY_USER=tp-registry` in step 1 and type the token at its prompt:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
oc -n "${MIRROR#*/}" create serviceaccount tp-registry
oc -n "${MIRROR#*/}" policy add-role-to-user registry-viewer -z tp-registry
oc -n "${MIRROR#*/}" create token tp-registry --duration=720h
date -d '+720 hours' +%F
}
```

Result: the third command prints the token, and the last one prints the date on which it stops working. **Write that date down: renew before it.** The cluster may cap the duration to a lower value; then use the cap. The push of step 5 still uses your own login.

To renew, run the `create token` command again and replace both secrets (the namespaces are `trident-port-system` and `trident-port`):

```bash
{
: "${MIRROR_HOST:?run step 1 first: MIRROR_HOST is not set}"
: "${REGISTRY_TOKEN:?run step 1 first: REGISTRY_TOKEN is not set}"
read -rsp "New registry token: " REGISTRY_TOKEN; echo
for NS in trident-port-system trident-port
do
  kubectl -n "$NS" create secret docker-registry ghcr \
    --docker-server="$MIRROR_HOST" --docker-username=tp-registry --docker-password="$REGISTRY_TOKEN" \
    --dry-run=client -o yaml | kubectl -n "$NS" apply -f -
done
unset REGISTRY_TOKEN
}
```

Trident Port also warns, in the pre-flight, when its own pull secret holds a session token or a token that expires in less than 14 days. The warning does not stop a migration.

### Check the pull secret before the upgrade

Run this after steps 6 and 8, and again before every upgrade. A throwaway pod pulls the curl image with the `ghcr` secret of the operator namespace. If the secret is expired, the pod stays in `ImagePullBackOff` and the command ends with the error:

```bash
{
: "${MIRROR:?run step 1 first: MIRROR is not set}"
kubectl -n trident-port-system run pull-check --image="$MIRROR/curl:8.11.0" --restart=Never \
  --overrides='{"spec":{"imagePullSecrets":[{"name":"ghcr"}]}}' --command -- true
kubectl -n trident-port-system wait --for=jsonpath='{.status.phase}'=Succeeded pod/pull-check --timeout=90s \
  || kubectl -n trident-port-system describe pod pull-check | grep -iE 'pull|auth'
kubectl -n trident-port-system delete pod pull-check --ignore-not-found
}
```

Result: `pod/pull-check condition met`. Use the curl tag listed in `IMAGES.txt` if it is not `8.11.0`. On `invalid username/password`, renew the token as above. This checks the operator namespace only: repeat with `-n trident-port` for the product namespace.

### Operator pod

The operator needs the `sed` of the OpenShift part of step 7.

### Product pod

On OpenShift, the product `ReplicaSet` is refused with `unable to validate against any security context constraint` and `runAsUser: 100` unless the chart knows that it runs there. Put this block in the `TridentPort` before the `kubectl apply` of step 9:

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

Step 9 adds this block for you with one command, and steps 8a to 8c create the constraints. They are in the steps because the order matters: constraints first, then the `TridentPort`.

## If something goes wrong

**What it is for:** to collect what the support team needs, in one folder, without changing anything on the cluster. The commands below copy no secret. Do not add your registry token or your license token to what you send.

On OpenShift, `oc` takes the same arguments as `kubectl` in the block below. Stop at the step that failed. Do not delete and reinstall to see if it goes away: the pods and events of the failure are the evidence.

```bash
mkdir -p tp-support
kubectl get events -n trident-port --sort-by=.lastTimestamp > tp-support/events-trident-port.txt 2>&1
kubectl get events -n trident-port-system --sort-by=.lastTimestamp > tp-support/events-trident-port-system.txt 2>&1
kubectl get pods -n trident-port -o wide > tp-support/pods-trident-port.txt 2>&1
kubectl get pods -n trident-port-system -o wide > tp-support/pods-trident-port-system.txt 2>&1
kubectl describe pods -n trident-port > tp-support/describe-trident-port.txt 2>&1
kubectl describe pods -n trident-port-system > tp-support/describe-trident-port-system.txt 2>&1
kubectl get tridentport -n trident-port -o yaml > tp-support/tridentport.txt 2>&1
kubectl logs -n trident-port-system deploy/trident-port-operator --all-containers > tp-support/operator-log.txt 2>&1
ls -la tp-support
```

Result: `ls` lists eight files. A file can hold an error line such as `No resources found` or `NotFound`, and that is still useful: it tells the team what did not exist. Send the whole `tp-support` folder.

If the product pod is running (step 11 shows it `1/1 Running`), also collect the product's own support bundle:

```bash
kubectl -n trident-port exec deploy/trident-port -c trident-port -- tp support-bundle --out /tmp/trident-port-support.tar.gz
POD=$(kubectl -n trident-port get pods -o name | grep 'pod/trident-port-' | grep -v webhook | head -1)
kubectl -n trident-port cp -c trident-port "${POD#pod/}":/tmp/trident-port-support.tar.gz ./trident-port-support.tar.gz
ls -la trident-port-support.tar.gz
```

Result: the last command lists `trident-port-support.tar.gz` with a size above zero. `kubectl cp` needs the pod name, which the `POD=` line finds. It does not accept `deploy/trident-port`. If the first command fails because the pod is not running, skip this block and send the `tp-support` folder alone.

Send `tp-support` and, when you have it, `trident-port-support.tar.gz` to your NetApp contact. Say in the message which step you were at and paste the `Result:` line you expected and the output you saw.
