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

Before applying CNI Flannel, you must change your cluster network `cidrBlocks` to `10.244.0.0/16` and set `allowAllInClusterTraffic` to `true` on your OpenStackCluster kind

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
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
```

## Install CSI Cinder driver for Volume Provisioning

Download all the manifests needed for CSI Cinder driver from here [https://github.com/kubernetes/cloud-provider-openstack/tree/master/manifests/cinder-csi-plugin](https://github.com/kubernetes/cloud-provider-openstack/tree/master/manifests/cinder-csi-plugin)

```bash
base64 -w 0 cloud.conf
```

Change the `cloud.conf` value in `csi-secret-cinderplugin.yaml` to your base64 result

```bash
kubectl create -f manifests/cinder-csi-plugin/csi-secret-cinderplugin.yaml
```

```bash
kubectl -f apply manifests/cinder-csi-plugin/
```

An example of manifest for provisioning a block storage and attached it to a pod. If a bad request occured that says 'invalid availability zone', you can change the `parameters.availability` value. If not set, the default availability from instance will be used

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-sc-cinderplugin
provisioner: cinder.csi.openstack.org
parameters:
  availability: nova      # <----- change the value here

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc-cinderplugin
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-sc-cinderplugin

---
apiVersion: v1
kind: Pod
metadata:
  name: test-block
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    volumeDevices:
      - devicePath: /dev/xvda
        name: csi-data-cinderplugin
  volumes:
  - name: csi-data-cinderplugin
    persistentVolumeClaim:
      claimName: csi-pvc-cinderplugin
      readOnly: false
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
