# Bring Your Own CNI Cilium VXLAN UDP flows

## Create AKS cluster with BYO-CNI

```bash
az group create --name rg-akscilium1 -l eastus2

az aks create --resource-group rg-akscilium1 --name akscilium1 --network-plugin none --node-vm-size Standard_D4ds_v5 --node-count 2

az aks get-credentials --resource-group rg-akscilium1 --name akscilium1

kubectl get nodes -o wide
```

## Get Cilium client

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium version
```

## Install Cilium using Helm

```bash
helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium --version 1.16.0 --namespace kube-system --set operator.replicas=1 --set azure.resourceGroup=rg-akscilium4 --set ipam.operator.clusterPoolIPv4PodCIDRList=192.168.0.0/16

cilium status --wait
```

```bash
# kubectl rollout restart daemonset/cilium -n kube-system

kubectl get ds -A

kubectl get deployments --all-namespaces

kubectl -n kube-system exec -ti ds/cilium -- bash
cilium status | grep Encryption
```

## Exec into nodes

```bash
kubectl get nodes -o wide
kubectl debug node/aks-nodepool1-19704097-vmss000000 -it --image=jrecord/nettools --profile=netadmin
kubectl debug node/aks-nodepool1-19704097-vmss000001 -it --image=jrecord/nettools --profile=netadmin
```

## Exec into containers

```bash
kubectl delete -f nettools.yaml
kubectl apply -f nettools.yaml
kubectl get pods -o wide

kubectl exec -it nettools-deployment-67d7f7df9-7hm6h -- /bin/bash
kubectl exec -it nettools-deployment-67d7f7df9-285l7 -- /bin/bash

# Simulate many TCP connections, which translate into Cilium VXLAN UDP flows
# Open TCP sockets on the listening pod and keep listening
for i in {30000..30000}; do nc -k -l $i & done
for i in {40000..45000}; do nc -k -l $i & done
for i in {50000..55000}; do nc -k -l $i & done
for i in {60000..64000}; do nc -k -l $i & done
netstat -an | wc

# Connect to the listening pod from the sending pod for 100 iterations, each time opening 1000 TCP sockets and iterating again
for n in {1..100}; do for i in {30000..30000}; do echo "$n-$i" | nc 192.168.0.121 $i & done; done
for n in {1..100}; do for i in {40000..45000}; do echo "$n-$i" | nc 192.168.0.100 $i & done; done &
for n in {1..100}; do for i in {50000..55000}; do echo "$n-$i" | nc 192.168.0.100 $i & done; done &
for n in {1..100}; do for i in {60000..64000}; do echo "$n-$i" | nc 192.168.0.100 $i & done; done &

# For long running test, can copy one of the commands into x.sh and execute it in a loop
while true; do
   echo "Start again..."
   ./x.sh
   echo "pkill nc..."
   pkill nc
   echo "Sleep..."
   sleep 2
done

iperf3 -s
iperf3 -c 4.227.121.12 -t 10 -P2 -b 500

kubectl delete -f netperf.yaml
kubectl apply -f netperf.yaml
kubectl get pods -o wide

kubectl exec -it netperf-deployment-59479fb7ff-fjbnc -- /bin/bash
kubectl exec -it netperf-deployment-59479fb7ff-sxbjs -- /bin/bash

netserver
netperf -H 10.0.1.110 -l 10 -t TCP_STREAM
netperf -H 10.0.1.110 -l 10 -t UDP_STREAM -- -1 1
```

Copy capture files

```bash
tcpdump -i eth0 -w /tmp/vm0-eth0.pcap -W 120 -G 60 -n -nn udp

kubectl cp default/node-debugger-aks-nodepool1-19704097-vmss000000-rk5h4:/tmp/vm0-eth0.pcap /mnt/c/temp/vxlan-capture/vm0-eth0.pcap
```
