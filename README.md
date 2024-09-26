# Cluster API Provider OpenStack
## Establishing the [Kubernetes Cluster for Management Cluster using RKE1](https://rke.docs.rancher.com/installation)
Create 3 VM, Master node, Worker node, and Image Builder in OpenStack. <br/>
### Install Docker in Master & Worker:
```
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update -y && sudo apt install -y docker-ce && sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

### Create RKE Cluster and join Master/worker node to the cluster (Registration Command can be check in RKE Registration)
For Master Node:
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.8.3 --server https://rke-endpoint.localhost --token kd8t7gl2n68z4fpg6h5jh58wrl29qkjcdxp6srzphs6x77gtg96hn5 --ca-checksum 3eeefb6dd6bedba527f907d21d74fadea1851472144829b9377075eab2613a84 --etcd --controlplane
```

For Worker:
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.8.3 --server https://rke-endpoint.localhost --token kd8t7gl2n68z4fpg6h5jh58wrl29qkjcdxp6srzphs6x77gtg96hn5 --ca-checksum 3eeefb6dd6bedba527f907d21d74fadea1851472144829b9377075eab2613a84 --worker
```


If using RKE, download kubeconfig, and save to kubernetes cluster in ~/.kube/config. <br/>
Export kubeconfig with command :
```
export KUBECONFIG=/home/ubuntu/.kube/config
```

### Establishing the Management Control Plane
Install and/or configure a Kubernetes cluster
```
export KUBECONFIG=/home/ubuntu/.kube/config
```
Install clusterctl Binary:
```
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.7.4/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
clusterctl version
```

Initialize the Management cluster [Cluster API](https://main.cluster-api.sigs.k8s.io/user/quick-start#initialize-the-management-cluster)
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure openstack 

<br/>

# Build [Image for OpenStack](https://image-builder.sigs.k8s.io/capi/providers/openstack.html#installing-packages-to-use-qemu-img)

Install and Configure Necessary package on Image Builder Instance
```
sudo apt update && sudo apt upgrade
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst cpu-checker libguestfs-tools libosinfo-bin
sudo usermod -a -G kvm ubuntu
sudo chown root:kvm /dev/kvm
```

Install [Packer](https://developer.hashicorp.com/packer/tutorials/docker-get-started/get-started-install-cli)
```
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install packer
```

Install [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
```
sudo apt-add-repository ppa:ansible/ansible
sudo apt install ansible
```

Building [OpenStack Imange Template](https://image-builder.sigs.k8s.io/capi/providers/openstack.html#building-images)
```
git clone https://github.com/kubernetes-sigs/image-builder.git
cd image-builder/images/capi
apt install make
make deps-qemu
apt-cache madison packer
apt install packer=1.9.5-1
make deps-qemu

make build-qemu-ubuntu-2004 | tee build-qemu-ubuntu-2004.txt
make build-qemu-ubuntu-2204 | tee build-qemu-ubuntu-2204.txt
```

Upload Image to OpenStack Glance

Image locate in /root/image-builder/images/capi/output/BUILD_NAME-kube-KUBERNETES_VERSION
```
source /home/ubuntu/openstack.sh
openstack image create --container-format bare --disk-format qcow2 --file ubuntu-2004-kube-v1.28.9.qcow2 --progress --private "capi-ubuntu-2004-kube-v1.28.9"
openstack image create --container-format bare --disk-format qcow2 --file ubuntu-2204-kube-v1.28.9.qcow2 --progress --private "capi-ubuntu-2204-kube-v1.28.9"
```

## Create Workload Cluster
### To see all required OpenStack environment variables execute:
clusterctl generate cluster --infrastructure openstack --list-variables demo-cluster
```
Required Variables:
  - KUBERNETES_VERSION
  - OPENSTACK_CLOUD
  - OPENSTACK_CLOUD_CACERT_B64
  - OPENSTACK_CLOUD_YAML_B64
  - OPENSTACK_CONTROL_PLANE_MACHINE_FLAVOR
  - OPENSTACK_DNS_NAMESERVERS
  - OPENSTACK_EXTERNAL_NETWORK_ID
  - OPENSTACK_FAILURE_DOMAIN
  - OPENSTACK_IMAGE_NAME
  - OPENSTACK_NODE_MACHINE_FLAVOR
  - OPENSTACK_SSH_KEY_NAME

Optional Variables:
  - CLUSTER_NAME                 (defaults to demo-cluster)
  - CONTROL_PLANE_MACHINE_COUNT  (defaults to 1)
  - WORKER_MACHINE_COUNT         (defaults to 0)
```

### Prepare openstack clouds.yaml file
Login to horizon, klik Project > API Access > Downlaod OpenStack RF File > OpenStack clouds.yaml file <br/>
```
cat clouds.yaml
clouds:
  capi:
    auth:
      auth_url: https://endpoint/identity/v3
      username: "username"
      password: password
      project_id: projectid
      project_name: "projectname"
      user_domain_name: "Default"
    region_name: "RegionOne"
    interface: "public"
    identity_api_version: 3
```

### Generate requirement variable for openstack:
```
cat ~/openstack-environmentdc4.yaml
export OPENSTACK_DNS_NAMESERVERS="8.8.8.8"
export OPENSTACK_FAILURE_DOMAIN="AZ_Public01_JBBK"
export OPENSTACK_CONTROL_PLANE_MACHINE_FLAVOR="GP.4C8G-intel"
export OPENSTACK_NODE_MACHINE_FLAVOR="GP.4C8G-intel"
export OPENSTACK_IMAGE_NAME="capi-ubuntu-2004-kube-v1.28.9"
export OPENSTACK_SSH_KEY_NAME="rf-macos"
export OPENSTACK_EXTERNAL_NETWORK_ID=f9881508-7de0-488e-a4ff-c437f4c8687f
```

### Generate Variables:
```
source openstack-environmentdc4.yaml
wget https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-openstack/master/templates/env.rc -O /tmp/env.rc
source /tmp/env.rc <path/to/clouds.yaml> <cloud>
source /tmp/env.rc clouds.yaml capi

# Verified generate variable
VARIABLES=$(clusterctl generate cluster --infrastructure openstack --list-variables demo-cluster | grep -A10 KUBERNETES | cut -d- -f2 | xargs)
for VAR in $VARIABLES; do echo "$VAR=${!VAR}"; done
KUBERNETES_VERSION=
OPENSTACK_CLOUD=capi
OPENSTACK_CLOUD_CACERT_B64=Cg==
OPENSTACK_CLOUD_YAML_B64=#REDACTED #BASE64 ENCODED
OPENSTACK_CONTROL_PLANE_MACHINE_FLAVOR=GP.4C8G-intel
OPENSTACK_DNS_NAMESERVERS=8.8.8.8
OPENSTACK_EXTERNAL_NETWORK_ID=f9881508-7de0-488e-a4ff-c437f4c8687f
OPENSTACK_FAILURE_DOMAIN=AZ_Public01_JBBK
OPENSTACK_IMAGE_NAME=capi-ubuntu-2004-kube-v1.28.9
OPENSTACK_NODE_MACHINE_FLAVOR=GP.4C8G-intel
OPENSTACK_SSH_KEY_NAME=rf-macos
```

### Generate the cluster configuration
```
clusterctl generate cluster demo-cluster --kubernetes-version v1.28.9 --control-plane-machine-count=1 --worker-machine-count=3 > demo-cluster.yaml

kubectl apply -f demo-cluster.yaml
secret/demo-cluster-cloud-config unchanged
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/demo-cluster-md-0 configured
cluster.cluster.x-k8s.io/demo-cluster created
machinedeployment.cluster.x-k8s.io/demo-cluster-md-0 created
kubeadmcontrolplane.controlplane.cluster.x-k8s.io/demo-cluster-control-plane created
openstackcluster.infrastructure.cluster.x-k8s.io/demo-cluster created
openstackmachinetemplate.infrastructure.cluster.x-k8s.io/demo-cluster-control-plane created
openstackmachinetemplate.infrastructure.cluster.x-k8s.io/demo-cluster-md-0 created
```

`Verification`

```
kubectl get openstackcluster
NAME           CLUSTER        READY   NETWORK                                BASTION IP   AGE
demo-cluster   demo-cluster   true    c710af8b-6472-431a-90c9-31e5a8f2ddf8                5m47s

kubectl describe osc demo-cluster
```


kubectl get kubeadmcontrolplane
```
NAME                         CLUSTER        INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE     VERSION
demo-cluster-control-plane   demo-cluster   true                                 1                  1         1             8m14s   v1.28.9
```
kubectl get md
```
NAME                CLUSTER        REPLICAS   READY   UPDATED   UNAVAILABLE   PHASE       AGE     VERSION
demo-cluster-md-0   demo-cluster   3                  3         3             ScalingUp   8m42s   v1.28.9
```

kubectl get ms
```
NAME                      CLUSTER        REPLICAS   READY   AVAILABLE   AGE     VERSION
demo-cluster-md-0-ptd7q   demo-cluster   3                              9m30s   v1.28.9
```
kubectl get ma
```
NAME                               CLUSTER        NODENAME                           PROVIDERID                                          PHASE     AGE     VERSION
demo-cluster-control-plane-hhx8k   demo-cluster   demo-cluster-control-plane-hhx8k   openstack:///4bdb1946-eba7-44ed-a0a5-6a787c666c21   Running   6m56s   v1.28.9
demo-cluster-md-0-ptd7q-5w8fn      demo-cluster   demo-cluster-md-0-ptd7q-5w8fn      openstack:///d616537d-9a8f-4f18-ae79-801a879b6cde   Running   9m51s   v1.28.9
demo-cluster-md-0-ptd7q-dbnhp      demo-cluster   demo-cluster-md-0-ptd7q-dbnhp      openstack:///ced85cf7-ddb5-4aaf-910c-20c384b100cb   Running   9m51s   v1.28.9
demo-cluster-md-0-ptd7q-qrcpn      demo-cluster   demo-cluster-md-0-ptd7q-qrcpn      openstack:///6e8d45c8-c4f6-410c-af6d-bb93e4a3190e   Running   9m51s   v1.28.9
```

Get Kubeconfig:
```
clusterctl get kubeconfig demo-cluster > demo-cluster.kubeconfig
```

Operation
```
kubectl --kubeconfig=demo-cluster.kubeconfig get nodes
NAME                               STATUS     ROLES           AGE     VERSION
demo-cluster-control-plane-hhx8k   NotReady   control-plane   5m15s   v1.28.9
demo-cluster-md-0-ptd7q-5w8fn      NotReady   <none>          2m54s   v1.28.9
demo-cluster-md-0-ptd7q-dbnhp      NotReady   <none>          2m22s   v1.28.9
demo-cluster-md-0-ptd7q-qrcpn      NotReady   <none>          2m19s   v1.28.9
```

### Install [OCCM](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md#config-openstack-cloud-controller-manager) on workload cluster
  
#### Create [Openstack Application Credential](https://docs.openstack.org/keystone/latest/user/application_credentials.html)
```
openstack application credential create occmcapi --unrestricted
```
```
cat clouds.yaml
---
clouds:
  capi:
    auth:
      auth_url: https://endpoint/identity/v3
      username: "username"
      password: password
      project_id: projectid
      project_name: "projectname"
      user_domain_name: "Default"
    region_name: "RegionOne"
    interface: "public"
    identity_api_version: 3

[LoadBalancer]
use-octavia=true
enabled=true
subnet-id=xxxxx-5b25-4cae-bf0a-deb7b1f2554d
floating-network-id=xxxxx-7de0-488e-a4ff-c437f4c8687f
provider=amphora
flavor-id=4d6aa951-bf6d-4985-9890-c31dad14dd1e
---
```

#### Create Secret:
```
kubectl --kubeconfig=demo-cluster.kubeconfig -n kube-system create secret generic cloud-config --from-file=cloud.conf
secret/cloud-config created
```

#### Install OCCM Roles
```
kubectl --kubeconfig=demo-cluster.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-roles.yaml

clusterrole.rbac.authorization.k8s.io/system:cloud-controller-manager created
clusterrole.rbac.authorization.k8s.io/system:cloud-node-controller created
```

#### Install OCCM Roles Bindings
```
kubectl --kubeconfig=demo-cluster.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml

clusterrolebinding.rbac.authorization.k8s.io/system:cloud-node-controller created
clusterrolebinding.rbac.authorization.k8s.io/system:cloud-controller-manager created
```

#### Install OCCM DS
```
kubectl --kubeconfig=demo-cluster.kubeconfig apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml

serviceaccount/cloud-controller-manager created
daemonset.apps/openstack-cloud-controller-manager created
```

Deploy CNI
```
kubectl --kubeconfig=demo-cluster.kubeconfig apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
serviceaccount/calico-cni-plugin created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrole.rbac.authorization.k8s.io/calico-cni-plugin created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-cni-plugin created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created
````

### Verification Workload Cluster
```
kubectl --kubeconfig=demo-cluster.kubeconfig get nodes
NAME                               STATUS   ROLES           AGE     VERSION
demo-cluster-control-plane-hhx8k   Ready    control-plane   12m     v1.28.9
demo-cluster-md-0-ptd7q-5w8fn      Ready    <none>          9m45s   v1.28.9
demo-cluster-md-0-ptd7q-dbnhp      Ready    <none>          9m13s   v1.28.9
demo-cluster-md-0-ptd7q-qrcpn      Ready    <none>          9m10s   v1.28.9
```
```
kubectl --kubeconfig=demo-cluster.kubeconfig get pods -o wide -A
NAMESPACE     NAME                                                       READY   STATUS    RESTARTS   AGE     IP              NODE                               NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-7ddc4f45bc-7mt45                   1/1     Running   0          56s     192.168.191.1   demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   calico-node-bc5m9                                          1/1     Running   0          56s     10.6.0.114      demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   calico-node-m5ssn                                          1/1     Running   0          56s     10.6.0.47       demo-cluster-md-0-ptd7q-dbnhp      <none>           <none>
kube-system   calico-node-mpc5q                                          1/1     Running   0          56s     10.6.0.43       demo-cluster-md-0-ptd7q-qrcpn      <none>           <none>
kube-system   calico-node-qr5b5                                          1/1     Running   0          56s     10.6.0.72       demo-cluster-md-0-ptd7q-5w8fn      <none>           <none>
kube-system   coredns-5dd5756b68-2hqls                                   1/1     Running   0          12m     192.168.191.3   demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   coredns-5dd5756b68-b85xd                                   1/1     Running   0          12m     192.168.191.2   demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   etcd-demo-cluster-control-plane-hhx8k                      1/1     Running   0          12m     10.6.0.114      demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   kube-apiserver-demo-cluster-control-plane-hhx8k            1/1     Running   0          12m     10.6.0.114      demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   kube-controller-manager-demo-cluster-control-plane-hhx8k   1/1     Running   0          12m     10.6.0.114      demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   kube-proxy-g426b                                           1/1     Running   0          9m39s   10.6.0.43       demo-cluster-md-0-ptd7q-qrcpn      <none>           <none>
kube-system   kube-proxy-kmfwf                                           1/1     Running   0          10m     10.6.0.72       demo-cluster-md-0-ptd7q-5w8fn      <none>           <none>
kube-system   kube-proxy-kx4sn                                           1/1     Running   0          9m42s   10.6.0.47       demo-cluster-md-0-ptd7q-dbnhp      <none>           <none>
kube-system   kube-proxy-mds4l                                           1/1     Running   0          12m     10.6.0.114      demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   kube-scheduler-demo-cluster-control-plane-hhx8k            1/1     Running   0          12m     10.6.0.114      demo-cluster-control-plane-hhx8k   <none>           <none>
kube-system   openstack-cloud-controller-manager-qlmgf                   1/1     Running   0          30s     10.6.0.114      demo-cluster-control-plane-hhx8k   <none>           <none>
```

```
clusterctl describe cluster demo-cluster
NAME                                                             READY  SEVERITY  REASON  SINCE  MESSAGE                                                               
Cluster/demo-cluster                                             True                     18m                                                                           
├─ClusterInfrastructure - OpenStackCluster/demo-cluster                                                                                                                 
├─ControlPlane - KubeadmControlPlane/demo-cluster-control-plane  True                     18m                                                                           
│ └─Machine/demo-cluster-control-plane-hhx8k                     True                     18m                                                                           
└─Workers                                                                                                                                                               
  └─MachineDeployment/demo-cluster-md-0                          True                     6m34s                                                                         
    └─3 Machines...                                              True                     16m    See demo-cluster-md-0-ptd7q-5w8fn, demo-cluster-md-0-ptd7q-dbnhp, ...
```
