# Command List

```bash
clusterctl generate cluster <CLUSTER_NAME> \
  --flavor external-cloud-provider 
  --kubernetes-version v1.23.10 \   
  --infrastructure openstack \   
  --control-plane-machine-count=1 \   
  --worker-machine-count=3 > <CLUSTER_NAME>.yaml
```
