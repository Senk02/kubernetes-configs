# Renkko Talos Setup

## Building the ISO

The RTL8127 10GbE NIC driver is not yet mainlined, so we build the ISO locally using imager instead of the Image Factory.

Match the imager tag to your target Talos version, and check extension versions at https://github.com/siderolabs/extensions/releases.

```bash
docker run -t --rm -v "${PWD}/_out":/out \
  ghcr.io/siderolabs/imager:<TALOS_VERSION> iso \
  --arch arm64 \
  --system-extension-image siderolabs/iscsi-tools:<VERSION> \
  --system-extension-image siderolabs/util-linux-tools:<VERSION> \
  --system-extension-image ghcr.io/senk02/talos-r8127:11.015.00-v1.12.6
```

The ISO will be in `_out/`. Flash to USB with:

```bash
sudo dd if=_out/talos-arm64.iso of=/dev/sdX bs=4M status=progress
sync
```

## When RTL8127 is mainlined

1. Remove the `--system-extension-image ghcr.io/senk02/talos-r8127:...` flag
2. Switch to using the Image Factory with `schematic.yaml` instead of local imager builds
3. Upgrade existing nodes with: `talosctl upgrade --nodes <IP> --image "factory.talos.dev/installer/<SCHEMATIC_ID>:<TALOS_VERSION>"`
