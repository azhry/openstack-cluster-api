## Build Custom Image
### Installing packages qemu-img
```bash
sudo -i
apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst cpu-checker libguestfs-tools libosinfo-bin build-essentials
```

### Add your user to group
```bash
sudo usermod -a -G kvm <yourusername>
sudo chown root:kvm /dev/kvm
```

### Download the image builder and build QCOW2 image
```bash
git clone https://github.com/kubernetes-sigs/image-builder.git
cd image-builder/images/capi/
make build-qemu-ubuntu-2004
```

If you want to build the image with another version of Kubernetes, try changing the version placeholder inside `images/capi/packer/config/kubernetes.json`
```json
{
  "crictl_arch": "amd64",
  "crictl_sha256": "https://github.com/kubernetes-sigs/cri-tools/releases/download/v{{user `crictl_version`}}/crictl-v{{user `crictl_version`}}-linux-{{user `crictl_arch`}}.tar.gz.sha256",
  "crictl_source_type": "pkg",
  "crictl_url": "https://github.com/kubernetes-sigs/cri-tools/releases/download/v{{user `crictl_version`}}/crictl-v{{user `crictl_version`}}-linux-{{user `crictl_arch`}}.tar.gz",
  "crictl_version": "1.23.0",
  "kubeadm_template": "etc/kubeadm.yml",
  "kubernetes_container_registry": "registry.k8s.io",
  "kubernetes_deb_gpg_key": "https://packages.cloud.google.com/apt/doc/apt-key.gpg",
  "kubernetes_deb_repo": "\"https://apt.kubernetes.io/ kubernetes-xenial\"",
  "kubernetes_deb_version": "<KUBERNETES_DEB_VERSION>",
  "kubernetes_http_source": "https://dl.k8s.io/release",
  "kubernetes_load_additional_imgs": "false",
  "kubernetes_rpm_gpg_check": "True",
  "kubernetes_rpm_gpg_key": "\"https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg\"",
  "kubernetes_rpm_repo": "https://packages.cloud.google.com/yum/repos/kubernetes-el7-{{user `kubernetes_rpm_repo_arch`}}",
  "kubernetes_rpm_repo_arch": "x86_64",
  "kubernetes_rpm_version": "<KUBERNETES_RPM_VERSION>",
  "kubernetes_semver": "<KUBERNETES_SEMVER>",
  "kubernetes_series": "<KUBERNETES_SERIES>",
  "kubernetes_source_type": "pkg",
  "systemd_prefix": "/usr/lib/systemd",
  "sysusr_prefix": "/usr",
  "sysusrlocal_prefix": "/usr/local"
}
```

You can see the list of commands to build another image version on `images/capi/Makefile` indicated with prefix `build-`

## Setting Up OpenStack CLI
### Install the OpenStack CLI
Currently the CLI only supported with Python 2.7.x. Install the CLI with `pip`
```bash
pip install python-openstackclient
```

### Setting Up CLI Authentication
Download the OpenStack RC File from your cloud provider and export the variables with source command
```bash
source openstack.rc
```

Test if the authentication is correct by executing
```bash
openstack network list
```

Now, your OpenStack CLI is ready to use!

## Setting Up Environment Variables
Download the `clouds.yaml` from your OpenStack cloud provider and set `clouds.openstack.verify` to `false` and `clouds.openstack.region_name` to the region where your cluster management will be provisioned. 
Then, download the `env.rc` to set up the required variables
```bash
wget https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-openstack/master/templates/env.rc -O /tmp/env.rc
```

Edit `env.rc` file and add the remaining required variables at the bottom (these values below are just for documentation purpose, not real value)
```.env
export OPENSTACK_CONTROL_PLANE_MACHINE_FLAVOR="SS2.2"
export OPENSTACK_NODE_MACHINE_FLAVOR="SM8.4"
export OPENSTACK_EXTERNAL_NETWORK_ID="79241ddc-c51b-4677-a763-f48c60870923"
export OPENSTACK_IMAGE_NAME="ubuntu-2004-kube-v1.24.8"
export OPENSTACK_SSH_KEY_NAME="kube-key"
export OPENSTACK_DNS_NAMESERVERS="8.8.8.8"
export OPENSTACK_FAILURE_DOMAIN="az-01"
```

The description of these configuration variables are as follows

| Variable      | Description | Example Value |
| ----------- | ----------- | ----------- |
| OPENSTACK_CONTROL_PLANE_MACHINE_FLAVOR | Machine flavor name that you can get by executing `openstack flavor list` | `SS2.2` |
| OPENSTACK_NODE_MACHINE_FLAVOR | Machine flavor name that you can get by executing `openstack flavor list` | `SM8.4` |
| OPENSTACK_EXTERNAL_NETWORK_ID | External network ID that you can get by executing `openstack network list` | `79241ddc-c51b-4677-a763-f48c60870923` |
| OPENSTACK_IMAGE_NAME | Image name that you can get by executing `openstack image list` | `Ubuntu 22.04 LTS` |
| OPENSTACK_SSH_KEY_NAME | Keypair name that you can get by executing `openstack keypair list` | `kube-key` |
| OPENSTACK_DNS_NAMESERVERS | Public DNS server | `8.8.8.8` |
| OPENSTACK_FAILURE_DOMAIN | Availability zone name that you can get by executing `openstack availability zone list` | `az-01` |

Then, export the variables using the source command
```bash
source /tmp/env.rc <path/to/clouds.yaml> openstack
```

## Cluster API

### Install clusterctl CLI
```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.2.6/clusterctl-linux-amd64 -o clusterctl
chmod +x ./clusterctl
sudo mv ./clusterctl /usr/local/bin/clusterctl
clusterctl version
```

### Install Management Cluster
Initialize the OpenStack infrastructure using this command
```bash
clusterctl init --infrastructure openstack
```
Then, generate the cluster yaml configuration using this command
```bash
clusterctl generate cluster <CLUSTER_NAME> \
  --flavor external-cloud-provider 
  --kubernetes-version v1.23.10 \   
  --infrastructure openstack \   
  --control-plane-machine-count=1 \   
  --worker-machine-count=3 > <CLUSTER_NAME>.yaml
```
For example, if the `CLUSTER_NAME` is `capi-quickstart`, then it will generate yaml configuration file `capi-quickstart.yaml`. Note that the `kubernetes-version` <b>must be the same as the Kubernetes version on your custom image</b>. Apply the configuration to provision the workload cluster.
```bash
kubectl apply -f <CLUSTER_NAME>.yaml
```

To get the cluster config, use this command.
```bash
clusterctl get kubeconfig <CLUSTER_NAME> > <KUBECONFIG_FILENAME>
```

If you want to delete the infrastructure, use this command.
```bash
clusterctl delete --infrastructure openstack
```

You can see the information about your workload cluster by running 
```bash
clusterctl describe cluster <CLUSTER_NAME>
```

See the workload cluster pods status by running
```bash
kubectl get pods -A --kubeconfig=<KUBECONFIG_FILENAME>
```

You can see that the `coredns-` pods status is not ready, because we need to apply the CNI solution first.

## Deploy CNI Flannel

Before applying CNI Flannel, you must change your cluster network `cidrBlocks` to `10.244.0.0/16` and set `allowAllInClusterTraffic` to `true` on your OpenStackCluster kind

```bash
kubectl --kubeconfig=./${KUBECONFIG_FILENAME} apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Download `create_cloud_conf.sh` from [https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-openstack/main/templates/create_cloud_conf.sh](https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-openstack/main/templates/create_cloud_conf.sh)
```bash
templates/create_cloud_conf.sh <path/to/clouds.yaml> <cloud> > /tmp/cloud.conf
```

```bash
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig create secret -n kube-system generic cloud-config --from-file=/tmp/cloud.conf
```

```bash
kubectl --kubeconfig=./${KUBECONFIG_FILENAME} apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
kubectl --kubeconfig=./${KUBECONFIG_FILENAME} apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
kubectl --kubeconfig=./${KUBECONFIG_FILENAME} apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
```

## Install CSI Cinder driver for Volume Provisioning

Download all the manifests needed for CSI Cinder driver from here [https://github.com/kubernetes/cloud-provider-openstack/tree/master/manifests/cinder-csi-plugin](https://github.com/kubernetes/cloud-provider-openstack/tree/master/manifests/cinder-csi-plugin)

```bash
base64 -w 0 cloud.conf
```

Change the `cloud.conf` value in `csi-secret-cinderplugin.yaml` to your base64 result

```bash
kubectl create -f manifests/cinder-csi-plugin/csi-secret-cinderplugin.yaml --kubeconfig=./${KUBECONFIG_FILENAME}
```

```bash
kubectl create -f manifests/cinder-csi-plugin/ --kubeconfig=./${KUBECONFIG_FILENAME}
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


## Another useful commands

To see current cluster info. 
```bash
kubectl cluster-info --kubeconfig=<KUBECONFIG_FILENAME>
```
The `--kubeconfig` flag is used to choose what kubeconfig file is used for running the kubectl command. You can also set the `KUBECONFIG` variable globally using 
```bash
export KUBECONFIG=<PATH_TO_KUBECONFIG_FILE>
```

To see the pod's logs
```bash
kubectl logs -f <POD_NAME> -n <NAMESPACE>
```

To see the cluster description, events, and status
```bash
kubectl describe cluster <CLUSTER_NAME>
```

To delete the applied configuration
```bash
kubectl delete -f <configuration>.yaml
```

### Author
&copy; Azhary Arliansyah

### Source
- [https://image-builder.sigs.k8s.io/capi/providers/openstack.html](https://image-builder.sigs.k8s.io/capi/providers/openstack.html)
- [https://cluster-api.sigs.k8s.io/user/quick-start.html](https://cluster-api.sigs.k8s.io/user/quick-start.html)
- [https://github.com/flannel-io/flannel/blob/master/README.md](https://github.com/flannel-io/flannel/blob/master/README.md)
- [https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md)
