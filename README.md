# Kubernetes-configs

Two clusters:
- **makko** - x86
- **renkko** - arm64, running on a Minisforum MS-R1 (CIX CP8180 ARM SoC) with a custom RTL8127 10GbE NIC driver extension

## Talos setup

### Image

**makko** can use a standard ISO from the [Image Factory](https://factory.talos.dev).

**renkko** requires a custom ISO built with the Talos imager because it includes an out-of-tree RTL8127 NIC driver extension:
```sh
docker run --rm -t -v "${PWD}/_out":/out \
  ghcr.io/siderolabs/imager:v1.12.6 iso \
  --arch arm64 \
  --system-extension-image ghcr.io/siderolabs/iscsi-tools:v0.2.0 \
  --system-extension-image ghcr.io/siderolabs/util-linux-tools:2.41.1 \
  --system-extension-image ghcr.io/senk02/talos-r8127:11.015.00-v1.12.6
```

Flash to USB with Rufus (GPT, DD mode) or `dd`. See `renkko/talos/README.md` for full details including the raw image workaround for running without a dedicated boot drive.

### Config

Set variables for your target cluster:
```sh
export TALOS_IP="10.0.0.73"
export TALOS_API_PORT="6443"
export CLUSTER_NAME="makko"  # or "renkko"
```

**makko** uses a minimal patch at `makko/patch.yaml`:
```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

**renkko** uses `renkko/talos/patch.yaml` which additionally includes Rook-Ceph kubelet mounts and kernel modules (`rbd`, `ceph`).

Generate and apply config, then bootstrap. Should take about 5 minutes to fully finish before moving on to Cilium:
```sh
talosctl gen config "${CLUSTER_NAME}" "https://${TALOS_IP}:${TALOS_API_PORT}" --config-patch @patch.yaml

talosctl apply-config --insecure -n "${TALOS_IP}" --file controlplane.yaml

talosctl bootstrap --nodes "${TALOS_IP}" --endpoints "${TALOS_IP}" --talosconfig=./talosconfig

talosctl kubeconfig --nodes "${TALOS_IP}" --endpoints "${TALOS_IP}" --talosconfig=./talosconfig
```

## Cilium install

The cluster WILL NOT show as ready for the control plane, so run the following commands when you see the node added and can `kubectl get nodes`.

For **renkko**, apply the `network-critical` PriorityClass before Cilium:
```sh
kubectl apply -f renkko/kube-system/cilium/priority-class.yaml
```

Install the CRDs and add the helm repo:
```sh
kubectl create namespace certificate

helm repo add cilium https://helm.cilium.io/

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_gateways.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_grpcroutes.yaml
```

Installing Cilium is pretty straight forward, but I have not been able to get it to work on another namespace.
```sh
export cilium_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/Senk02/kubernetes-configs/refs/heads/main/${CLUSTER_NAME}/kube-system/cilium/application.yaml" | yq eval-all '. | select(.metadata.name == "cilium-application" and .kind == "Application")' -)
export cilium_name=$(echo "$cilium_applicationyaml" | yq eval '.metadata.name' -)
export cilium_chart=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.chart' -)
export cilium_repo=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.repoURL' -)
export cilium_namespace=$(echo "$cilium_applicationyaml" | yq eval '.spec.destination.namespace' -)
export cilium_version=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export cilium_values=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.helm.valuesObject' -)

echo "$cilium_values" | helm template $cilium_name $cilium_chart --repo $cilium_repo --version $cilium_version --namespace $cilium_namespace --values - | kubectl apply --namespace $cilium_namespace --filename -
```

## ArgoCD setup

This is grabbing all the relevant info from the git repo, and then applying all of it to the cluster.
```sh
kubectl create namespace argocd

export argocd_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/Senk02/kubernetes-configs/refs/heads/main/${CLUSTER_NAME}/argocd/argocd/application.yaml" | yq eval-all '. | select(.metadata.name == "argocd-application" and .kind == "Application")' -)
export argocd_name=$(echo "$argocd_applicationyaml" | yq eval '.metadata.name' -)
export argocd_chart=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.chart' -)
export argocd_repo=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.repoURL' -)
export argocd_namespace=$(echo "$argocd_applicationyaml" | yq eval '.spec.destination.namespace' -)
export argocd_version=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export argocd_values=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.helm.valuesObject' - | yq eval 'del(.configs.cm)' -)
export argocd_config=$(curl -sL "https://raw.githubusercontent.com/Senk02/kubernetes-configs/refs/heads/main/${CLUSTER_NAME}/argocd/argocd/appset.yaml" | yq eval-all '. | select(.kind == "AppProject" or .kind == "ApplicationSet")' -)

echo "$argocd_values" | helm template $argocd_name $argocd_chart --repo $argocd_repo --version $argocd_version --namespace $argocd_namespace --values - | kubectl apply --namespace $argocd_namespace --filename -

echo "$argocd_config" | kubectl apply --filename -
```

## RTL8127 driver extension

The `ghcr.io/senk02/talos-r8127` extension is a stopgap until the RTL8127 driver is mainlined in the kernel. Once mainlined, remove the custom extension from the imager command and switch to using the Image Factory with the schematic in `renkko/talos/schematic.yaml`.
