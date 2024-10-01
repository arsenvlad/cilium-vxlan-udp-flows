# Azure CNI Powered by Cilium with Overlay

## Create AKS cluster using Azure CNI Powered by Cilium

```bash
az group create --name rg-aksazurecilium1 -l eastus2

az aks create --resource-group rg-aksazurecilium1 --name aksazurecilium1 --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --pod-cidr 192.168.0.0/16 --node-vm-size Standard_D4ds_v5 --node-count 2

az aks get-credentials --resource-group rg-aksazurecilium1 --name aksazurecilium1

kubectl get nodes -o wide
```

```bash
kubectl get ds -A

kubectl get deployments --all-namespaces

kubectl -n kube-system exec -ti ds/cilium -- bash
cilium status | grep Encryption
```

## Exec into nodes

```bash
kubectl get nodes -o wide
kubectl debug node/aks-nodepool1-31879185-vmss000000 -it --image=jrecord/nettools --profile=netadmin
kubectl debug node/aks-nodepool1-31879185-vmss000001 -it --image=jrecord/nettools --profile=netadmin
```

## Exec into containers

```bash
kubectl delete -f nettools.yaml
kubectl apply -f nettools.yaml
kubectl get pods -o wide

kubectl exec -it nettools-deployment-67d7f7df9-gwn7m -- /bin/bash
kubectl exec -it nettools-deployment-67d7f7df9-lljr4 -- /bin/bash

# Simulate many TCP connections, which translate into Cilium VXLAN UDP flows
# Open TCP sockets on the listening pod and keep listening
for i in {30000..30001}; do nc -k -l $i & done
netstat -an | wc

# Connect to the listening pod from the sending pod for 100 iterations, each time opening 1000 TCP sockets and iterating again
for n in {1..100}; do for i in {30000..30001}; do echo "$n-$i" | nc 192.168.1.92 $i & done; done

while true; do
   echo "Start again..."
   ./x.sh
   echo "pkill nc..."
   pkill nc
   echo "Sleep..."
   sleep 10
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
tcpdump -i eth0 -w /tmp/vm0-eth0.pcap -n -nn icmp
tcpdump -i eth0 -w /tmp/vm1-eth0.pcap -n -nn icmp

kubectl cp default/node-debugger-aks-nodepool1-31879185-vmss000000-6rhvd:/tmp/vm0-eth0.pcap /mnt/c/temp/overlay-capture/vm0-eth0.pcap
kubectl cp default/node-debugger-aks-nodepool1-31879185-vmss000001-crq5h:/tmp/vm1-eth0.pcap /mnt/c/temp/overlay-capture/vm1-eth0.pcap
```

Understand the Azure overlay

<https://docs.cilium.io/en/stable/network/concepts/ipam/azure-delegated-ipam/>

```bash
kubectl get nodenetworkconfigs.acn.azure.com -A
kubectl describe nodenetworkconfigs.acn.azure.com -A
```
