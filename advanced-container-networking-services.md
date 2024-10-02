# Advanced Container Networking Services

<https://learn.microsoft.com/en-us/azure/aks/advanced-container-networking-services-overview>

## Network Observability

Create Azure Monitor resource and Grafana instance

```bash
az resource create --resource-group rg-aksazurecilium1 --name aksazurecilium-monitor --namespace microsoft.monitor --resource-type accounts --properties '{}'

az grafana create --resource-group rg-aksazurecilium1 --name aksazurecilium-grafana
```

Link Azure Monitor and Grafana to AKS cluster

```bash
azureMonitorId=$(az resource show --resource-group rg-aksazurecilium1  --name aksazurecilium-monitor --namespace microsoft.monitor --resource-type accounts --query id --output tsv)
grafanaId=$(az grafana show --resource-group rg-aksazurecilium1 --name aksazurecilium-grafana --query id --output tsv)
echo $azureMonitorId
echo $grafanaId

az aks update --resource-group rg-aksazurecilium1 --name aksazurecilium1 --enable-azure-monitor-metrics --azure-monitor-workspace-resource-id $azureMonitorId --grafana-resource-id $grafanaId
```

Visualize using Grafana

```bash
az aks get-credentials --resource-group rg-aksazurecilium1 --name aksazurecilium1

kubectl get pod -o wide -n kube-system | grep ama-
```

## Advanced Network Observability

<https://learn.microsoft.com/en-us/azure/aks/advanced-network-observability-concepts>

<https://learn.microsoft.com/en-us/azure/aks/network-observability-managed-cli>

```bash
az feature register --namespace "Microsoft.ContainerService" --name "AdvancedNetworkingPreview"
az feature show --namespace "Microsoft.ContainerService" --name "AdvancedNetworkingPreview"
az provider register --namespace Microsoft.ContainerService

az aks update --resource-group rg-aksazurecilium1 --name aksazurecilium1 --enable-advanced-network-observability

kubectl get pods -o wide -n kube-system | grep ama-
```

Install Hubble CLI

```bash
# Set environment variables
export HUBBLE_VERSION=v1.16.1
export HUBBLE_ARCH=amd64

# Install Hubble CLI
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

Visualize Hubble Flows

```bash
kubectl get pods -o wide -n kube-system -l k8s-app=hubble-relay
kubectl port-forward -n kube-system svc/hubble-relay --address 127.0.0.1 4245:443
```

Configure Hubble certificates

```bash
#!/usr/bin/env bash

set -euo pipefail
set -x

# Directory where certificates will be stored
CERT_DIR="$(pwd)/.certs"
mkdir -p "$CERT_DIR"

declare -A CERT_FILES=(
  ["tls.crt"]="tls-client-cert-file"
  ["tls.key"]="tls-client-key-file"
  ["ca.crt"]="tls-ca-cert-files"
)

for FILE in "${!CERT_FILES[@]}"; do
  KEY="${CERT_FILES[$FILE]}"
  JSONPATH="{.data['${FILE//./\\.}']}"

  # Retrieve the secret and decode it
  kubectl get secret hubble-relay-client-certs -n kube-system -o jsonpath="${JSONPATH}" | base64 -d > "$CERT_DIR/$FILE"

  # Set the appropriate hubble CLI config
  hubble config set "$KEY" "$CERT_DIR/$FILE"
done

hubble config set tls true
hubble config set tls-server-name instance.hubble-relay.cilium.io
```

Check if certificates are correctly configured

```bash
kubectl get secrets -n kube-system | grep hubble-
hubble observe --pod hubble-relay-7ff97868ff-bjdrg
```

Deploy hubble-ui to visualize Hubble Flows

```bash
kubectl apply -f hubble-ui.yaml

kubectl -n kube-system port-forward svc/hubble-ui 12000:80
```

<http://localhost:12000>
