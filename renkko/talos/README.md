# Renkko Talos Setup

## Hardware

Minisforum MS-R1 (CIX CP8180 ARM SoC, 32GB LPDDR5, dual RTL8127 10GbE). The RTL8127 NIC driver is not yet mainlined, so we use a custom Talos system extension.

## Building the ISO

```bash
docker run --rm -t -v "${PWD}/_out":/out \
  ghcr.io/siderolabs/imager:v1.12.6 iso \
  --arch arm64 \
  --system-extension-image ghcr.io/siderolabs/iscsi-tools:v0.2.0 \
  --system-extension-image ghcr.io/siderolabs/util-linux-tools:2.41.1 \
  --system-extension-image ghcr.io/senk02/talos-r8127:11.015.00-v1.12.6
```

Flash to USB with Rufus (GPT, DD mode) or dd:

```bash
sudo dd if=_out/metal-arm64.iso of=/dev/sdX bs=4M status=progress && sync
```

Boot from USB, Talos installs to the boot drive when config is applied.

## Running from USB only (no install disk)

If you don't have a boot drive yet, build a raw disk image instead. This requires Docker Desktop on Windows (PowerShell) or a real Linux host. Does NOT work in WSL2.

```bash
docker run --rm -t --privileged -v /dev:/dev -v "${PWD}/_out":/out \
  ghcr.io/siderolabs/imager:v1.12.6 metal \
  --arch arm64 \
  --system-extension-image ghcr.io/siderolabs/iscsi-tools:v0.2.0 \
  --system-extension-image ghcr.io/siderolabs/util-linux-tools:2.41.1 \
  --system-extension-image ghcr.io/senk02/talos-r8127:11.015.00-v1.12.6
```

```bash
zstd -d _out/metal-arm64.raw.zst
```

Flash with Rufus (DD mode, GPT) or dd on Linux.

## Bootstrapping

```bash
export TALOS_IP="<IP from console>"
export CLUSTER_NAME="renkko"

talosctl gen config "${CLUSTER_NAME}" "https://${TALOS_IP}:6443" \
  --config-patch @renkko/talos/patch.yaml
talosctl apply-config --insecure -n "${TALOS_IP}" --file controlplane.yaml
talosctl bootstrap --nodes ${TALOS_IP} --endpoints ${TALOS_IP} --talosconfig=./talosconfig
talosctl kubeconfig --nodes ${TALOS_IP} --endpoints ${TALOS_IP} --talosconfig=./talosconfig
```

Then install the PriorityClass, Cilium, and ArgoCD following the main README.

## When RTL8127 is mainlined

1. Remove the `--system-extension-image ghcr.io/senk02/talos-r8127:...` flag from imager commands
2. Switch to using the Image Factory with `schematic.yaml` instead of local imager builds
3. Upgrade existing nodes with: `talosctl upgrade --nodes <IP> --image "factory.talos.dev/installer/<SCHEMATIC_ID>:<TALOS_VERSION>"`
