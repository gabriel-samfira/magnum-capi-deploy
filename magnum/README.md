# Deploying Magnum with CAPI

In this section we'll discuss the needed bits to deploy Magnum with the Vexxhost magnum-cluster-api driver. We will need a Kubernetes management cluster that is separate form OpenStack. Within this cluster we will enable the Cluster API provider for OpenStack.

We will then configure Magnum to use the CAPI provider to deploy clusters.


## The management cluster

It is recommended that the management cluster run on a vanilla kubernetes distribution, or at least one that uses etcd as a key-value store. In our tests, microk8s has issues with kubernetes leases needed for global locking in the CAPI driver. Microk8s uses dqlite instead of etcd which may be the cause of the issue.

In this guide we'll use charmed kubernetes, as that fits best in our existing tooling.


We will need at least 3 VMs added to MAAS, each with the following specs:

* 8 CPUs
* 16 GB of memory
* 100 GB of disk space

We will assume for the purpose of this guide that the VMs will have the following tags in MAAS:

* magnum-k8s

We will also assume that the VMs will be added to the `romania` zone in MAAS.

We will assume that the VIP (Virtual IP) for the management cluster will be `10.10.3.164`. This IP will be configured in keepalived and will ensure access to the management cluster API.

To deploy the management cluster, we will use the following bundle:

```yaml
description: A highly-available, production-grade Kubernetes cluster.
issues: https://bugs.launchpad.net/charmed-kubernetes-bundles
default-base: ubuntu@22.04
source: https://github.com/charmed-kubernetes/bundle
website: https://ubuntu.com/kubernetes/charmed-k8s
name: charmed-kubernetes
variables:
  keepalived-vip: &keepalived-vip 10.10.3.164
machines:
  '0':
    base: ubuntu@22.04
    constraints: tags=magnum-k8s zones=romania
  '1':
    base: ubuntu@22.04
    constraints: tags=magnum-k8s zones=romania
  '2':
    base: ubuntu@22.04
    constraints: tags=magnum-k8s zones=romania
applications:
  keepalived:
    charm: keepalived
    channel: latest/stable
    options:
      virtual_ip: *keepalived-vip
  calico:
    annotations:
    channel: stable
    charm: calico
    options:
      vxlan: Always
      ignore-loose-rpf: true
  containerd:
    annotations:
    channel: stable
    charm: containerd
  easyrsa:
    annotations:
    channel: stable
    charm: easyrsa
    num_units: 1
    to: ["lxd:0"]
  etcd:
    annotations:
    channel: stable
    charm: etcd
    num_units: 3
    to: ["lxd:0", "lxd:1", "lxd:2"]
    options:
      channel: 3.4/stable
  kubeapi-load-balancer:
    annotations:
    channel: stable
    charm: kubeapi-load-balancer
    expose: true
    to: ["lxd:0", "lxd:1", "lxd:2"]
    num_units: 3
    options:
      extra_sans: *keepalived-vip
  kubernetes-control-plane:
    annotations:
      gui-x: '800'
      gui-y: '850'
    channel: stable
    charm: kubernetes-control-plane
    to: ["lxd:0", "lxd:1", "lxd:2"]
    num_units: 3
    options:
      channel: 1.31/stable
      loadbalancer-ips: *keepalived-vip
  kubernetes-worker:
    annotations:
      gui-x: '90'
      gui-y: '850'
    channel: stable
    charm: kubernetes-worker
    expose: true
    to: ["0", "1", "2"]
    num_units: 3
    options:
      channel: 1.31/stable
relations:
- - kubernetes-control-plane:loadbalancer-external
  - kubeapi-load-balancer:lb-consumers
- - kubernetes-control-plane:loadbalancer-internal
  - kubeapi-load-balancer:lb-consumers
- - kubernetes-control-plane:kube-control
  - kubernetes-worker:kube-control
- - kubernetes-control-plane:certificates
  - easyrsa:client
- - etcd:certificates
  - easyrsa:client
- - kubernetes-control-plane:etcd
  - etcd:db
- - kubernetes-worker:certificates
  - easyrsa:client
- - kubeapi-load-balancer:certificates
  - easyrsa:client
- - calico:etcd
  - etcd:db
- - calico:cni
  - kubernetes-control-plane:cni
- - calico:cni
  - kubernetes-worker:cni
- - containerd:containerd
  - kubernetes-worker:container-runtime
- - containerd:containerd
  - kubernetes-control-plane:container-runtime
- - keepalived:juju-info
  - kubeapi-load-balancer:juju-info
```

Save the above contents in a file called `magnum-k8s.yaml` and deploy the bundle using the following set of commands:

```bash
juju add-model magnum-k8s
juju deploy ./magnum-k8s.yaml
```

Wait for the model to settle. This may take a long time.

## Installing dependencies

We will need:

* kubectl
* clusterctl
* jq
* yq

### Install packages

```bash
sudo apt-get install -y curl jq yq
```

### Get the `kubectl` and `clusterctl` binaries

The `kubectl` binary we will also bundle as a resource for the `package-customization` charm (more on that later), so let's download it to a folder called `magnum-resources`:

```bash
mkdir $HOME/magnum-resources
curl -o $HOME/magnum-resources/kubectl \
    -L https://dl.k8s.io/release/v1.31.1/bin/linux/amd64/kubectl
chmod +x $HOME/magnum-resources/kubectl
```

We also need to copy the `kubectl` binary somewhere in our `$PATH`:

```bash
cp $HOME/magnum-resources/kubectl /usr/local/bin
```

Download the `clustercrl` binary:

```bash
CAPI_VERSION=${CAPI_VERSION:-v1.9.3}
sudo curl -Lo /usr/local/bin/clusterctl \
    https://github.com/kubernetes-sigs/cluster-api/releases/download/${CAPI_VERSION}/clusterctl-linux-amd64
sudo chmod +x /usr/local/bin/clusterctl
```

## Getting the kube config file

After the model has settled and our management cluster is up and running, we need to fetch the kubeconfig file. We will use this file to initialize the cluster API provider for OpenStack and we will also need to send this file to the Magnum deployment.

```bash
CFG=$(juju run -q -m magnum-k8s kubernetes-control-plane/leader get-kubeconfig)
echo $CFG | yq -r '.kubeconfig' > $HOME/magnum-resources/kubeconfig
```

## Deploying the CAPI provider for OpenStack

Before we begin, make sure that the config that was saved to `$HOME/magnum-resources/kubeconfig` has the VIP set as the server address. If not, make sure you replace whatever IP is there with the VIP.

After that, we can initialize the CAPI provider for OpenStack:

```bash
export KUBECONFIG=$HOME/magnum-resources/kubeconfig
export EXP_CLUSTER_RESOURCE_SET=true
export EXP_KUBEADM_BOOTSTRAP_FORMAT_IGNITION=true
export CLUSTER_TOPOLOGY=true

# Create the magnum-system namespace
kubectl create namespace magnum-system

# Deploy cluster api
clusterctl init \
    --core cluster-api \
    --bootstrap kubeadm \
    --control-plane kubeadm \
    --infrastructure openstack
```

Verify that the CAPI provider is up and running:

```bash
kubectl get deploy -A
```

Ensure that the autoscaler pods can be scheduled to the machines in the management cluster:

```bash
MACHINES=$(juju machines -m magnum-k8s --format=json | jq -r '.machines.[].hostname')
for i in $MACHINES;do
    kubectl label node $i openstack-control-plane=enabled
done
```

Pnce you start creating clusters with autoscaling enabled, you should start seeing autoscaling pods spin up in the `magnum-system` namespace:

```bash
kubectl get pods -n magnum-system
```

## Deploying magnum

The upstream magnum charm does not currently support the CAPI driver we're about to install. Installing and using the driver requires more frequent updates than what is currently available in the ubuntu repositories. 

This means that we need to use a PPA to install the needed package and maintain that long term. Efforts will be made to eventually integrate the driver in the upstream charm, however this might take a long time and may not be alligned with what Canonical want to do long term.

As such, we will use a fork of the charm for the time being:

```bash
git clone https://github.com/gabriel-samfira/charm-magnum
cd charm-magnum
git checkout add-options-and-change-templates
```

Install `charmcraft`:

```bash
sudo snap install charmcraft --classic
```

Build the charm:

```bash
charmcraft pack
```

At the end of the build process, you should have a new file in the current folder called `magnum_amd64.charm`. This file can then be deployed to the OpenStack model of choice.


Before we do so, make sure you entirely remove any existing magnum deployment, especially if the deployment is made on any other version of ubuntu tham 24.04.

```bash
juju status magnum
juju remove-application magnum --force
```

Create a magnum configuration file. Make sure you change the `vip`, `region` and `trustee-admin` options to match your environment:

```yaml
magnum:
    worker-multiplier: 0.1
    openstack-origin: distro
    vip: 10.10.3.167 10.10.15.167 10.10.11.167
    region: Romania
    trustee-admin: magnum_romania_admin
    cluster-user-trust: true
    use-internal-endpoints: true
```

Then we can deploy the charm:

```bash
juju deploy ./magnum_amd64.charm \
    --base ubuntu@24.04/stable \
    --to lxd:0 \
    --config ./magnum-config.yaml magnum \
    --bind romania-mgmt public=romania-api-public admin=romania-mgmt internal=romania-api-internal shared-db=romania-mgmt \
    magnum
```

Scale that up to 3 units:

```bash
juju add-unit magnum --to lxd:1
juju add-unit magnum --to lxd:2
```

Deploy mysql router:

```bash
juju deploy mysql-router \
    --base ubuntu@24.04/stable \
    --channel latest/edge \
    --bind romania-mgmt db-router=romania-mgmt \
    magnum-mysql-router
```

Deploy hacluster:

```bash
juju deploy hacluster \
    --base ubuntu@24.04/stable \
    --channel 2.4/edge \
    --config cluster_count=3 \
    magnum-hacluster
```

Deploy the package customization charm:

```bash
juju deploy --base ubuntu@24.04/stable package-customization
juju config package-customization \
    packages="python3-magnum-cluster-api,magnum-api,magnum-common,magnum-conductor,python3-magnum" \
    ppa="ppa:gabriel-samfira/magnum-cluster-api"
```

Add relations:

```bash
juju relate magnum:shared-db mysql-innodb-cluster:shared-db
juju relate magnum-mysql-router:shared-db mysql-innodb-cluster:db-router
juju relate magnum-hacluster:ha magnum:ha
juju relate magnum keystone
juju relate magnum rabbitmq-server
juju relate magnum-mysql-router:db-router mysql-innodb-cluster:db-router
juju relate magnum vault
juju relate magnum-package-customization magnum
```

## Create the resource file

In the `magnum-resources` folder, we should already have the kubeconfig and kubectl we downloaded earlier. We also need to download `helm`.

```bash
cd $HOME/magnum-resources
wget https://get.helm.sh/helm-v3.17.0-linux-amd64.tar.gz
tar -zxvf helm-v3.17.0-linux-amd64.tar.gz
mv linux-amd64/helm .
rm -rf linux-amd64
```

We should now have 3 files:

```bash
~$ ls -l
total 123472
-rwxr-xr-x 1 ubuntu ubuntu 57176216 Dec 16 18:40 helm
-rw-rw-r-- 1 ubuntu ubuntu     1951 Jan 11 23:41 kubeconfig
-rw-rw-r-- 1 ubuntu ubuntu 69243032 Jan  7 01:07 kubectl
```

We need to create a metadata.yaml file in the same folder with the following contents:

```yaml
files:
- source: kubeconfig
  target: /var/lib/magnum/.kube/config
  permissions: 0o600
  owner: magnum
  group: magnum
- source: helm
  target: /usr/bin/helm
  permissions: 0o755
  owner: root
  group: root
- source: kubectl
  target: /usr/bin/kubectl
  permissions: 0o755
  owner: root
  group: root
```

Now we zip the file:

```bash
zip -r ../magnum-resources.zip .
```

And attach the resource to the charm:

```bash
juju attach-resource magnum-package-customization drop-in-files=$HOME/magnum-resources.zip
```