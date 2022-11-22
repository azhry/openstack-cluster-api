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

## Deploy CNI Flannel

```bash
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Download `create_cloud_conf.sh` from [https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-openstack/main/templates/create_cloud_conf.sh](https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-openstack/main/templates/create_cloud_conf.sh)
```bash
templates/create_cloud_conf.sh <path/to/clouds.yaml> <cloud> > /tmp/cloud.conf
```

```bash
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig create secret -n kube-system generic cloud-config --from-file=/tmp/cloud.conf
```

```bash
rm /tmp/cloud.conf
```

```bash
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
```

-----------------------------------------------------------

```bash
kubectl cluster-info --kubeconfig=<KUBECONFIG_FILENAME>
```

```bash
kubectl logs -f <POD_NAME> -n <NAMESPACE>
```

```bash
kubectl describe cluster <CLUSTER_NAME>
```
