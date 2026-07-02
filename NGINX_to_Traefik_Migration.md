# Ingress NGINX to Traefik Migration Guide

Ingress NGINX is being retired in March 2026. Starting with RKE2 v1.36, the default ingress controller is **Traefik**. This guide covers the Ingress NGINX → Traefik migration steps for existing RKE2 clusters.

## Cluster Note

This cluster has **3 master (control-plane) nodes**. All config changes and `systemctl restart rke2-server` commands below must be applied on **all 3 masters**.

## Prerequisites

- RKE2 version: `v1.32.11+rke2r1`, `v1.33.7+rke2r1`, `v1.34.3+rke2r1` or later (or any release `> v1.35`).
- Traefik's NGINX compatibility layer does not support all annotations. Check [Traefik & NGINX Annotations](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/).
- Use the **Discovery Tool** below to detect unsupported annotations.

---

## Discovery Tool: Annotation Compatibility Scan

Before starting the migration, it's recommended to scan existing Ingresses for annotations that Traefik doesn't support.

Repo: [traefik/ingress-nginx-migration](https://github.com/traefik/ingress-nginx-migration) · [Installation](https://github.com/traefik/ingress-nginx-migration?tab=readme-ov-file#installation) · [Releases](https://github.com/traefik/ingress-nginx-migration/releases)

### 1. Install

**On a machine with internet access (recommended):** the install script always fetches the latest release.

```bash
curl -sSL https://raw.githubusercontent.com/traefik/ingress-nginx-migration/main/scripts/install.sh | bash

ingress-nginx-migration version
```

**Airgap install:** if the node has no internet access, download the correct tarball from [Releases](https://github.com/traefik/ingress-nginx-migration/releases) (check the current version and filename there), transfer it to the airgap node, and install manually:

```bash
# Check the archive contents first (to confirm the binary name)
tar -tzf ingress-nginx-migration_<version>_linux_amd64.tar.gz

# Verify checksum (compare against the checksums file published on the Releases page)
sha256sum ingress-nginx-migration_<version>_linux_amd64.tar.gz

tar -xzf ingress-nginx-migration_<version>_linux_amd64.tar.gz

sudo mv ingress-nginx-migration /usr/local/bin/
sudo chmod +x /usr/local/bin/ingress-nginx-migration

ingress-nginx-migration version
```

> Replace `<version>` with the release you downloaded (e.g. `v1.2.1`). Check the Releases page for the current version.

### 2. Run the scan

```bash
ingress-nginx-migration --kubeconfig ~/.kube/config
```

Or run it as a local web server and generate an HTML report:

```bash
ingress-nginx-migration --kubeconfig ~/.kube/config --addr 127.0.0.1:18080 &
SRV=$!

sleep 3

curl -s http://127.0.0.1:18080/ > report.html

kill $SRV
wait $SRV 2>/dev/null
```

> **Viewing in a browser:** download `report.html` to your local machine and open it in a browser.
>
> Example (replace `<node-ip>`, user, and path with your own):
> ```bash
> scp <user>@<node-ip>:/home/user/report.html ~/Downloads/report.html
> ```
> Then open `~/Downloads/report.html` in your browser (double-click in file explorer, or `open ~/Downloads/report.html` / `xdg-open ~/Downloads/report.html`).

### 3. Parse the report

The generated `report.html` embeds a JSON report (`reportJSON`) that can also be read from the terminal. The commands below require `jq`.

> **No `jq` (airgap):** on a machine with internet access, download the `jq` binary from [releases](https://github.com/jqlang/jq/releases) or the `.deb`/`.rpm` package for your package manager, transfer it to the airgap node, and install it:
> ```bash
> # Binary example (amd64)
> sudo mv jq-linux-amd64 /usr/local/bin/jq
> sudo chmod +x /usr/local/bin/jq
> ```

```bash
# Full JSON report
grep -o 'const reportJSON = {.*};' report.html | sed 's/const reportJSON = //;s/;$//' | jq .
```

```bash
# Summary: total / compatible / unsupported counts
grep -o 'const reportJSON = {.*};' report.html | sed 's/const reportJSON = //;s/;$//' \
  | jq -r '"Total: \(.ingressCount) | Compatible: \(.compatibleIngressCount) | Unsupported: \(.unsupportedIngressCount)"'
```

```bash
# List annotation badges / version gates found in the report
grep -A1 'annotation-badge\|badge-v3' report.html \
  | grep -oE 'nginx\.ingress\.kubernetes\.io/[a-z-]+|v3\.[67]' \
  | paste -d ' ' - -
```

```bash
# List only the unsupported annotations
grep -o 'const reportJSON = {.*};' report.html | sed 's/const reportJSON = //;s/;$//' \
  | jq '.unsupportedIngressAnnotations'
```

Recommended workflow before moving on:

1. Identify the unsupported annotations from the report.
2. Add their Traefik equivalents (native middlewares / IngressRoute options).
3. Remove the unsupported annotations once replaced.
4. Re-run the Discovery Tool.
5. Once the unsupported count reaches an acceptable level, proceed with the migration.
6. Validate application functionality — a zero unsupported-annotation count doesn't guarantee behavior is unchanged.

---

## Backup Recommendations

```bash
# Export all Ingress resources
kubectl get ingress --all-namespaces -o yaml > ingress-backup.yaml

# Export NGINX ConfigMaps
kubectl get configmap --all-namespaces -l app.kubernetes.io/name=ingress-nginx -o yaml > nginx-configmaps.yaml

# Back up the RKE2 / Traefik config files (needed most often for rollback)
sudo cp /etc/rancher/rke2/config.yaml ./config.yaml.bak
sudo cp /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml ./rke2-traefik-config.yaml.bak
```

Inspect only the controller's ConfigMap (useful for mapping values to Traefik equivalents):

```bash
kubectl get configmap --all-namespaces -l app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/component=controller -o yaml
```

Common ConfigMap key → Traefik equivalent:

| NGINX ConfigMap key | Traefik equivalent |
|---|---|
| `proxy-body-size` | `middlewares.buffering` (`maxRequestBodyBytes`) |
| `proxy-read-timeout` | `serversTransport` / `transport.respondingTimeouts.readTimeout` |
| `proxy-send-timeout` | `transport.respondingTimeouts.writeTimeout` |
| `proxy-buffering` | `middlewares.buffering` |
| `use-forwarded-headers` | `entryPoints.<name>.forwardedHeaders` |

> Map these values to their Traefik equivalents using the [Traefik NGINX migration guide](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/) — the table above covers the most common keys, not all of them. These mappings are conceptual equivalents and may require implementation through Traefik static configuration, middlewares, or ServersTransport depending on your deployment; they are not always a 1:1 mapping.

---

## Migration Overview

| Phase | Description |
|-------|-------------|
| **Phase 1** | Enable Traefik alongside Ingress NGINX on temporary ports. |
| **Phase 2** | Duplicate Ingresses and test them via Traefik. |
| **Phase 3** | Once testing is complete, remove Ingress NGINX. |
| **Phase 4** | Clean up duplicated resources. |

---

## Phase 1: Dual Ingress Controller Setup

### 1. Assign `ingressClassName: nginx` to existing Ingresses

```bash
kubectl get ingress --all-namespaces -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name' --no-headers | while read NS NAME; do
    echo "Patching Ingress: $NS/$NAME"
    kubectl patch ingress "$NAME" -n "$NS" --type=merge -p '{"spec": {"ingressClassName": "nginx"}}'
done
```

Verify:

```bash
kubectl get ingress --all-namespaces -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,ICLASS:.spec.ingressClassName'
```

> ⚠️ If any Ingress shows `none` or a different class in the `ICLASS` column, patch it manually before continuing.

### 2. Update RKE2 config

```bash
sudo -i
vi /etc/rancher/rke2/config.yaml
```

```yaml
# /etc/rancher/rke2/config.yaml
ingress-controller:
- ingress-nginx
- traefik
```

**Airgap:** `rke2-images.linux-amd64.tar.zst` doesn't include Traefik. Download `rke2-images-traefik.linux-amd64.tar.zst` separately and place it on the airgap node.

### 3. Configure Traefik ports and compatibility settings

```bash
vi /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml
```

This manifest: moves Traefik to temporary ports (`8000`/`8443`) and enables NGINX annotation compatibility.

```yaml
# rke2-traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
    name: rke2-traefik
    namespace: kube-system
spec:
    valuesContent: |-
        ports:
            web:
                hostPort: 8000
            websecure:
                hostPort: 8443
        providers:
            # See Version Gate below
            kubernetesIngressNGINX:
                enabled: true
                ingressClass: "rke2-ingress-nginx-migration"
                controllerClass: "rke2.cattle.io/ingress-nginx-migration"
```

> **Version Gate:** starting with `v1.36.2+rke2r1` / `v1.35.6+rke2r1` / `v1.34.9+rke2r1` / `v1.33.13+rke2r1` and later, the provider name changed from `kubernetesIngressNginx` to `kubernetesIngressNGINX`.

### 4. Restart RKE2

On all 3 masters:

```bash
sudo systemctl restart rke2-server
```

> For HA safety, restart one control-plane node at a time and verify it becomes `Ready` before continuing to the next.

Verify:

```bash
kubectl get daemonset -n kube-system
kubectl get ingressclass
kubectl get helmchart -n kube-system
kubectl get helmchartconfig -n kube-system
kubectl get helmchartconfig rke2-traefik -o yaml -n kube-system

# Additional checks useful on migration day
kubectl get pods -A | grep traefik
kubectl logs -n kube-system daemonset/rke2-traefik
kubectl get svc -A
```

> `kubectl logs daemonset/...` doesn't work on all Kubernetes versions. If it fails, use a label selector or pod name instead:
> ```bash
> kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
> # or
> kubectl logs -n kube-system <traefik-pod-name>
> ```

### 5. Verify functionality

- Existing Ingress NGINX Ingresses should still be reachable on the standard ports (80/443).
- You can test new Ingresses with the `traefik` class on the temporary ports (8000/8443).
- The Traefik DaemonSet should show `hostPort: 8000` and `hostPort: 8443`.
- A new `rke2-ingress-nginx-migration` IngressClass should exist.
- Check Traefik logs for the Ingress NGINX provider starting:

```
INF Starting provider *ingressnginx.Provider
```

---

## Phase 2: Parallel Migration and Validation

> ⚠️ **Rancher Ingress:** in clusters with Rancher installed, see SUSE Rancher Support for the Rancher Ingress resource.

### 1. Duplicate and reclassify Ingresses

For every Ingress using `ingressClassName: nginx`, create a copy with the class set to `rke2-ingress-nginx-migration`.

```bash
#!/bin/bash
# Duplicates all Ingresses using the 'ingress-nginx' class, renames them,
# sets the class to 'rke2-ingress-nginx-migration', and applies them to the cluster.

echo "Starting automated Ingress duplication and reclassification..."

kubectl get ingress --all-namespaces -o json | \
jq -c '.items[] | select(.spec.ingressClassName == "nginx")' | \
while read -r INGRESS; do

    NS=$(echo "$INGRESS" | jq -r '.metadata.namespace')
    NAME=$(echo "$INGRESS" | jq -r '.metadata.name')
    NEW_NAME="${NAME}-traefik"

    echo "Processing Ingress: $NS/$NAME"

    MODIFIED_INGRESS=$(echo "$INGRESS" | jq \
        'del(.metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"], .status, .metadata.managedFields)' | \
        jq --arg NEW_NAME "$NEW_NAME" '.metadata.name = $NEW_NAME | .spec.ingressClassName = "rke2-ingress-nginx-migration"')

    echo "$MODIFIED_INGRESS" | kubectl apply -f -
    echo "  -> Created duplicate Ingress: $NS/$NEW_NAME"
done

echo "Ingress duplication complete."
```

### 2. Test both controllers

- Ingress NGINX: `http://<Node_IP>` (80/443)
- Traefik: `http://<Node_IP>:8000` (8000/8443)

> Traefik also provides a `ClusterIP` service by default.

Test all services on the Traefik port (8000/8443), especially NGINX-specific annotations. Pay particular attention to:

- Redirects
- Authentication
- Rewrites
- WebSocket
- File uploads
- TLS
- Sticky sessions (if used)

### 3. (Optional) External load balancer

If you use an external LB, add Traefik as a backend (`http://<Node_IP>:8000`). See the [Traefik Migration Guide](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/) for details.

**Health check endpoints differ:**

| Controller | Endpoint |
|---|---|
| Ingress NGINX | `/healthz` |
| Traefik | `/ping` |

Verify the duplicate Ingresses were actually created:

```bash
kubectl get ingress -A | grep traefik
```

---

## Phase 3: Final Switchover and Port Reassignment

> Removing Ingress NGINX can take a while. If downtime matters: remove Ingress NGINX first (keep Traefik on 8000/8443), then switch Traefik to standard ports.

### 1. Remove Ingress NGINX

```yaml
# /etc/rancher/rke2/config.yaml
ingress-controller: 
- traefik
```

> After saving the configuration, restart `rke2-server` on each control-plane node (see step 3 below) — it's easy to change the config and forget to restart.

If splitting this phase, restart here and don't move to the next step until Ingress NGINX is fully removed.

### 2. Move Traefik to standard ports

```yaml
# rke2-traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
    name: rke2-traefik
    namespace: kube-system
spec:
    valuesContent: |-
        providers:
            # See Version Gate below
            kubernetesIngressNGINX:
                enabled: true
                ingressClass: "rke2-ingress-nginx-migration"
                controllerClass: "rke2.cattle.io/ingress-nginx-migration"
```

> **Version Gate:** starting with `v1.36.2+rke2r1` / `v1.35.6+rke2r1` / `v1.34.9+rke2r1` / `v1.33.13+rke2r1` and later, the provider name changed from `kubernetesIngressNginx` to `kubernetesIngressNGINX`.

### 3. Restart RKE2

On all 3 masters:

```bash
sudo systemctl restart rke2-server
```

> For HA safety, restart one control-plane node at a time and verify it becomes `Ready` before continuing to the next.

### 4. Verify on standard ports

- The Ingress NGINX DaemonSet should be gone.
- Services should now be reachable via the duplicated Traefik Ingresses on standard ports (80/443).
- Check `kubectl get ingressclass` — the `nginx` class should be gone, and the expected IngressClasses for your deployment should be present.

---

## Phase 4: Cleanup

### 1. Remove Ingress NGINX objects

Delete the legacy Ingresses bound to `ingressClassName: nginx`:

```bash
kubectl get ingress --all-namespaces \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,ICLASS:.spec.ingressClassName' \
  --no-headers | awk '$3 == "nginx" {print}' | while read NS NAME ICLASS; do
    echo "Deleting legacy Ingress: $NS/$NAME"
    kubectl delete ingress "$NAME" -n "$NS"
done
```

---

## Rollback (if required after Phase 3)

Ingress NGINX keeps running through Phase 1 and Phase 2, so rollback is only relevant once Ingress NGINX has actually been removed in Phase 3.

If rollback is required after removing Ingress NGINX:

1. Restore the original RKE2 configuration:
   ```bash
   sudo cp config.yaml.bak /etc/rancher/rke2/config.yaml
   ```

2. Restore the original Traefik HelmChartConfig:
   ```bash
   sudo cp rke2-traefik-config.yaml.bak \
     /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml
   ```

3. Restart `rke2-server` on each control-plane node (one node at a time):
   ```bash
   sudo systemctl restart rke2-server
   ```

4. If any Ingress resources were modified or deleted, restore them (this is only needed if the originals were changed — during Phase 1/2 the original Ingresses aren't touched):
   ```bash
   kubectl apply -f ingress-backup.yaml
   ```

   > If duplicate `*-traefik` Ingress resources exist, delete them before restoring the original Ingress resources:
   > ```bash
   > kubectl get ingress --all-namespaces | grep -- '-traefik'
   > # delete the matching ones, then apply the backup
   > ```

5. Verify:
   - Ingress NGINX DaemonSet is running.
   - `nginx` IngressClass exists.
   - Applications are reachable through Ingress NGINX.

---

## Additional Notes

A separate "bridge" IngressClass (`rke2-ingress-nginx-migration`) is used because:

1. Both controllers updating the same Ingress's status at the same time could cause race conditions.
2. The `nginx` IngressClass is automatically removed once Ingress NGINX is uninstalled.

> `rke2-ingress-nginx-migration` doesn't have to be used permanently. Once the migration is complete, Ingresses can optionally be switched to the standard `traefik` IngressClass.

## References

- [Traefik & Ingresses with NGINX Annotations](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/)
- [traefik/ingress-nginx-migration](https://github.com/traefik/ingress-nginx-migration)
