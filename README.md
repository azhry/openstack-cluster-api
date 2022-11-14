# Command List

```bash
clusterctl init --infrastructure openstack
```

```bash
clusterctl delete --infrastructure openstack
```

```bash
clusterctl generate cluster <CLUSTER_NAME> \
  --flavor external-cloud-provider 
  --kubernetes-version v1.23.10 \   
  --infrastructure openstack \   
  --control-plane-machine-count=1 \   
  --worker-machine-count=3 > <CLUSTER_NAME>.yaml
```

```bash
kubectl apply -f <CLUSTER_NAME>.yaml
```

```bash
clusterctl get kubeconfig <CLUSTER_NAME> > <KUBECONFIG_FILENAME>
```

```bash
kubectl cluster-info --kubeconfig=<KUBECONFIG_FILENAME>
```

```bash
kubectl logs -f <POD_NAME> -n <NAMESPACE>
```

```bash
kubectl describe cluster <CLUSTER_NAME>
```
