# Kubernetes course

This repository contains excercises with Kubernetes.

<hr>

## Setup

1. Create or use one user for SSH communication on all servers with same password, and premissions in `visudo` to use commands wihtout password.
2. Install docker
3. Install kubernetes modules (kubeadm, kubelet, & kubectl) for corresponding machines (+ open port, add repository key, add repository to package manager sources, install kubernetes tools) [how-to](https://phoenixnap.com/kb/install-kubernetes-on-ubuntu)
4. Add docker and kubernetes tools to be held back to automatic update/remove/installation
5. Add user to the docker group and change premissions for ~/.kube to current user

[More info](https://www.densify.com/kubernetes-tools/kubeadm/)

<hr>

### Cluster creation

1. Setup cluster on master node with:

    ```bash
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 
    ```

    NOTE: when prompted for containered.sock error try:

    ```bash
    sudo rm /etc/containerd/config.toml
    sudo systemctl restart containerd
    sudo kubeadm init
    ```

2. Set kubectl access (commands from step 1 output):

    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

3. The kubeadm init command should output a kubeadm join command containing a token and hash. Copy that command and run it with sudo on both worker nodes. It should look something like this:

    ```bash
    sudo kubeadm join $some_ip:6443 --token $some_token --discovery-token-ca-cert-hash $some_hash
    ```

    NOTE1: If error about containerd is thrown these steps might help:

    ```bash
    sudo rm /etc/containerd/config.toml
    sudo systemctl restart containerd
    ```

    NOTE2: User can print the full 'kubeadm join' flag needed to join the cluster with the following command:

    ```bash
    kubeadm token create --print-join-command
    ```

4. Run the kubeadm join command on your Kube Node 1 and Kube Node 2 servers:

    ```bash
    sudo kubeadm join <IP_ADDRESS> --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
    ```

5. Check from master if nodes got added succesfully:

    ```bash
    kubectl get nodes
    ```

    Output should be similar to this:

    ```bash
    NAME                           STATUS     ROLES           AGE    VERSION
    f8bbdd78c31c.mylabserver.com   NotReady   control-plane   112s   v1.24.0
    f8bbdd78c32c.mylabserver.com   NotReady   <none>          55s    v1.24.0
    f8bbdd78c33c.mylabserver.com   NotReady   <none>          39s    v1.24.0
    ```

    NOTE: The nodes are expected to have a STATUS of NotReady at this point

    FINAL NOTE: If `kubectl version` or `kubectl get nodes` returns server connection error this might help:

    ```bash
    sudo -i
    swapoff -a
    exit
    strace -eopenat kubectl version
    ```

    [More info](https://discuss.kubernetes.io/t/the-connection-to-the-server-host-6443-was-refused-did-you-specify-the-right-host-or-port/552/4)
    [Kubernetes swap](https://serverfault.com/questions/881517/why-disable-swap-on-kubernetes)
    [kubeadm swap errors](https://stackoverflow.com/questions/47094861/error-while-executing-and-initializing-kubeadm)

<hr>

### Network setup

For this setup the Flannel solution is used.

1. To enable networking on nodes:

    ```bash
    echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf

    sudo sysctl -p
    ```

2. Install Flanner on Master node:

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    ```

<hr>

## Containers vs pods

* Pods are the smallest building blocks in Kubernetes module. Usully pod contains one container (but can use more).

* Nodes (machines) contain pods.

* Cluster contain nodes.

To create pods in Kubernetes use YAML to declarate pod specyfication:

```bash
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
  image: nginx
EOF
```

To view more information about pod:

```bash
kubectl describe pods $pod_name -n $namespace
```

NOTE: Every Kubernetes object is in some namespace, eg. kubernetes backend pods: `kubectl get pods -n kube-system` (namespace kube-system)

<hr>

## Clusters and Nodes

As in ansible Nodes divide into two groups: Controllers and Workers. Each type of nodes migh be deployed multiple times.

* View nodes: `kubectl get nodes`
* Get more information about node: `kubectl describe node $node_name`

<hr>

## Networking in kubernetes

Kubernetes cluster creates virtual network witch contains all nodes. It's separete from phisical network.

So pods don't know they are ran on separated machines (nodes)

<hr>

## Kubernetes architecture

By using `kubectl get pods -n kube-system` all kubernetes system pods are shown, witch helps understand kubernetes architecture.

eg.

* etcd - provides distributed and sync. data storage for cluster
* kube-apiserver - serves Kubernetes API, used for primary interface for cluseter
* kube-controller-manager - bundles several components into one package, for managing cluster
* kube-scheduler - schedules pods to run on individual hosts

This combined provides control plane. kubeadm runs this components as pods wihtin cluster itself.

In addition, each node has: kubelet  (service for running pods) and kube-proxy

<hr>

## Deploying applications with Kuberneter deployments

After managing single pods there is a deployment management.

Deployment is a automation for pods, it allows to specify desired state for set of pods. The cluster then will maintain the desired state.

Other functionality:

* Scaling - scaling up/down replicas, and deployment will meet that number

* Rolling updates - gradual updates for new versions of image (app). It will gradually replace existing containers with new version

* Self-Healing - deployment is watching over existing pods and desired state. After pod is down it will spin up new one to replace it.

Simple Deployment:

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-products
  labels:
    app: store-products
spec:
  replicas: 4
  selector:
    matchLabels:
      app: store-products
  template:
    metadata:
      labels:
        app: store-products
    spec:
      containers:
      - name: store-products
        image: linuxacademycontent/store-products:1.0.0
        ports:
        - containerPort: 80
EOF
```

NOTE: to check if service is running try: `kubectl get svc store-products`

Create service (combine pods as one service) for pods:

```bash
cat << EOF | kubectl apply -f -
kind: Service
apiVersion: v1
metadata:
  name: store-products
spec:
  selector:
    app: store-products
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF
```

NOTE: Services are another kind of load balancing in kubernetes network model. They combine pods into single "service"

Use busybox pod to check if service is running correctly:

```shell
kubectl exec busybox -- curl -s store-products
```

[Busybox creation](https://rjshree.medium.com/how-to-get-busybox-image-running-in-kubernetes-cluster-ac3e10fddcf7)

<hr>

## Microservices - where Kubernetes shine

Deployment of microservice application:

0. Create cluster and networking

1. clone repository with k8s YAML descriptors ([example repository](https://github.com/linuxacademy/robot-shop))

2. Create namespace (`kubectl create namespace test-namespace`)

3. Install app in cluster (`kubectl -n test-namespace create -f ~/gh-repo/K8s/descriptors/`)

4. Get list of app pods and let them finish starting up (`kubectl get pods -n robot-shop -w`, namespace needs to be specified!)

5. App should run (in case of robot-shop, on port: )

NOTES: Descriptors can be edited, eg. `kubectl edit deployment mongodb -n robot-shop` or vim into descriptor file.

Replicas can be checked via `kubectl get deployment mongodb -n robot-shop`

<hr>

## Helm

Helm is a Kubernetes package manager. Operates on charts with are blueprints for Deploment/service and other descriptors. They are like packages for kubernetes.

It supports templating via GO Templating. (Templating like jinja2 but start with `.Values.other_subcategory`, where Values is values.yaml file).

Descriptors can be changes to helm charts. Chart needs to has its own directory, with Chart.yaml, templates/ and values.yaml (minimal setup). Also one can craete chart sketch by running `helm create chart $chartName`.

Also charts can contain dependencies (subcharts) stored in template folder.

### Chart releases

Release is a running chart. But also a normal release number.

Charts also have releases. One can change it manually inside Chart.yaml file.

Or update running chart: `helm upgrade $chartName -set alpineimage=1.1.6`

<hr>

### Convert Kubernetes manifest file into Helm Chart

After minimal chart is created, move manifest file into `chartName/templates/`.

Here one can add templating functionality to manifest, using GO templating.

Now after Chart.yaml, values.yaml and template/serviceName.yaml are configured properly, command `helm install $chartName --dry-run` shlould output the same service manifest as Kubernetes one, but with dynamic values.

<hr>

### Helm Debugging and other commands

* Check Helm status (current state of releases): `helm status`
* See setted values: `helm show values $chartName`
* Dry run chart: `helm install $chartName --dry-run`
* Install chart (package): `helm install $chartName`
* Get chart from repo: `helm fetch $chartName` - it downloads the tarball of chart
* Add Helm repository: `helm repo add $name $url`

Also charts can be rollbacked to previous versions.

<hr>

### Pre- and post- actions with hooks

Like in git, hooks can run after some action.

They are defined as normal template with special annotations, that define it as hook

Hooks are part of chart. The parent chart cannot overide subchart's hooks.

Hooks have weights - lower means first in execution. If this is not important, set it to 0.

If hooks fails whole release fails.

! Objects created by hooks are not a part of (chart) release. Administrator must keep in mind cleaning or tearing them down. When release is removed, hooks can presist.

They can:

* Load data

* Manipulate data

* Stage data

#### Hook chart example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: posthooktainer
  annotations:
    "helm.sh/hook": "post-install"
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  containers:
  - name: hooktainer
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo post-install hook Pod is running && sleep 10']
  restartPolicy: Never
  terminationGracePeriodSeconds: 0
```

[More on hooks](https://helm.sh/docs/topics/charts_hooks/)

<hr>

### Testing charts

In helm charts can be tested by special hook, with creates container with running command to assert condition.

This can be used to check state, configuration, validation.

Test hooks are stored in `chartName/templates/tests/test-something.yaml`.

#### To run tests use: `helm test releaseName`

[More on tests](https://helm.sh/docs/topics/chart_tests/)

<hr>

### Creating and using libraries

Library is a shared template, used to avoid code repetition.

To use library:

1. In Chart.yaml specify dependency on library:

    ```yaml
    ...
    decription: ...
    dependencies:
    - name: common
    version: "^0.0.5"
    repository: "https://charts.helm.sh/incubator/"
    ```

2. In chart directory (`chartName/`) run:

    `helm dependency update` - this will grab dependency, at specified version. In case of version change, it deleted old one and replaces it with newer.

3. Extract library tarball:

    `tar -xzf common-0.0.5.tar`

4. In the `chartName/common/templates` one can see, the templates from library.

#### Library use example

Every chart manifest contains: apiVersion,  kind, and metadata. Let's use library for it.

```yaml
{{- template "common.deployment" (list . "demo.deployment") -}}
{{- define "demodeployment" -}}
{{- end}}
spec:
  ...
```

In this case 'common' is a library.

[More on libraries](https://helm.sh/docs/topics/library_charts/)

<hr>

### Validating and packing charts

Charts from repositories are signed. To check if downloaded chart is correct run `helm verify $chartName`.

To package charts:

1. Create gpg key.

2. Package chart with key: `helm package --sign --key keyName --keyring /key/path`

3. Now when chart will be changed without signing again, the `helm verify` would return error.

<hr>

### Helm adminsitration

#### Role based access control

Helm uses kubectl context for user resources. This might limit some users from performing helm operations.

So in short - helm uses user's context to execute kubectl.

#### Trublueshooting helm

Well Helm contains of 3 major components:

1. Helm binary

    Issues might presents as: binary premissions, versions, access. Also interaction with kubectl and its configuration shloud be taken at consideration.

2. Chart configuration

    This might present as: syntax errors, checking output with --dry-run.

3. Kubernetes

    Questions to check includes: is cluster configured correctly? Are ones user is able to perform commands via kubectl?

As said in previous section, role based access might be a issue at some point.

To fix helm binary it is the easiest to reinstall it.

### Helm Plugins

HELM_PLUGINS env is accounted for the plugins install location.
This directory contains all installed plugins.

To find HELM_PLUGINS call `helm env`.

#### Installing plugins

via: `helm plugin install url` - this can point to any url or version control repo.

#### Managin plugins

`helm plugins list`
`helm plugins uninstall pluginName`

### Helm storage backend

Helm has 3 types of sotrage backends:

1. ConfigMap

    Stores config information in configmaps.

2. Secret

    In Helm v3 this is a default storage backend. It is located in Helm namespace.

    `helm list`
    `kubectl get secret -l "owner=helm" --all-namespaces`
    `kubectl get secret secret.name.v1 -o yaml`

3. SQL

    As for day 5.06.24 it supports only PostgreSQL. Reqiures configured connection string and deployed SQL instance