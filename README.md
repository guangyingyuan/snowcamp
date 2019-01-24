# Service Mesh Patterns Workshop (snowcamp.io)

Please go through the instructions below to create a Kubernetes cluster with Istio installed. Alternatively, you can use any other cloud providers Kubernetes cluster offering. 

If you're having issues installing Minikube and getting it to work, you should try Docker for Mac/Windows instead.

## Setup Minikube with Istio

These instructions will help you set up and run a Kubernetes cluster using Minikube on your computer. When that's set up, we will download and install Istio on that cluster and at the end of this walkthrough, you will have a working Kubernetes cluster with Istio running on it.

### Installing Prerequisites

In this section we will follow the steps to enable virtualization (if needed) and install all necessary prerequisites for running a Kubernetes cluster on your computer.

#### Enable virtualization
A prerequisite to running Minikube is to ensure that virtualization is enabled in your computer's BIOS. If you are running a Windows OS, you will have to reboot your computer, enter BIOS and enable a feature that's either called `Intel Virtualization Technology (Intel VT)` or `AMD-V`, depending on the brand of the processor. For MacOS, virtualizaiton is already enabled.

#### Install Hypervisor
We will use VirtualBox hypervisor to run Minikube. VirtualBox can be installed on computers with Windows, Linux or MacOS.

1. Download VirtualBox for your platform: 
    - [VirtualBox for Windows](https://download.virtualbox.org/virtualbox/5.2.22/VirtualBox-5.2.22-126460-Win.exe)
    - [VirtualBox for OS X](https://download.virtualbox.org/virtualbox/5.2.22/VirtualBox-5.2.22-126460-OSX.dmg)
    - [VirtualBox for Linux Distributions](https://www.virtualbox.org/wiki/Linux_Downloads)
1. Run the install package and follow the instructions to install it.


#### Install Minikube

**MacOS**

Use the following command that downloads the binary, makes is executable and puts it in your path by copying it to `/usr/local/bin`. Finally, it deletes the originally downloaded file:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.30.0/minikube-darwin-amd64 && chmod +x minikube && sudo cp minikube /usr/local/bin/ && rm minikube
```

Alternatively, if you use Brew, run:
```bash
brew cask install minikube
```

**Windows**

Download the executable from [here](https://github.com/kubernetes/minikube/releases/download/v0.30.0/minikube-windows-amd64), rename it to `minikube.exe` and add it to your path. Alternatively, you can try using the [installer](https://github.com/kubernetes/minikube/releases/download/v0.30.0/minikube-installer.exe) (experimental)

**Linux**

Use the following command that downloads the binary, makes is executable and puts it in your path by copying it to `/usr/local/bin`. Finally, it deletes the originally downloaded file:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.30.0/minikube-linux-amd64 && chmod +x minikube && sudo cp minikube /usr/local/bin/ && rm minikube
```

#### Install Kubernetes CLI

Kubernetes CLI (`kubectl`) is a tool we will use to control the Kubernetes cluster. Version of the latest CLI at the time of writing this was `v1.12.0`. Kubernetes CLI uses a config file (located in `./kube/config`) that defines the different contexts and Kubernetes clusters. Once we install Minikube in the next section, you will use the `kubectl` to switch to the `minikube` cluster and basically tell the CLI that's the cluster we want to run our commands against. 

**MacOS**

- Using curl (download the binary, make it executable and copy to path): 
    ```
    curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl
    ```

- Using Homebrew:
    ```
    brew install kubernetes-cli
    ```

**Windows**

Download the executable from [here](https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/windows/amd64/kubectl.exe) and copy it to your path.

**Linux**

Download the binary, make it executable and move it to the path:
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl
```

##### Verify the installation
To check the installation, simply run `kubectl version` and you should get see a response similar to this:

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"fc32d2f3698e36b93322a3465f63a14e9f0eaead", GitTreeState:"clean", BuildDate:"2018-03-26T16:55:54Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"darwin/amd64"}
```

>Note: there might be an error in the response as well, but that's because we don't have a running cluster to talk to yet.

### Starting Minikube

With all prerequisites installed, it's time to start Minikube. Open the terminal/console window and type in the following command to start Minikube:

```bash
minikube start --memory=8192 --cpus=4 --vm-driver="virtualbox"
```
>Note: the above command might take some time as it will download the Minikube ISO and install/start everything needed to run Kubernetes.

The output from the start command should look something like this:
```bash
$ minikube start --memory=8192 --cpus=4
Starting local Kubernetes v1.12.4 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```

When the above command finishes, you can run `minikube status` to ensure the cluster is up and running. You should get an output similar to this one: 

```bash
$ minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.101
```

Next, we will use `kubectl` to switch to the `minikube` context, so we can talk to the cluster: 
```
kubectl config use-context minikube
```
>Note: alternatively, you can use a tool called [`kubectx`](https://github.com/ahmetb/kubectx) to quickly switch between contexts.

Finally, run `minikube dashboard` to open the Kubernetes dashboard and confirm Kubernetes cluster is up and running (hint: click on the `Nodes` link on the left to see the information about the cluster).

A couple of useful Minikube commands:

- `minikube start` - starts the Kubernetes cluster
- `minikube delete` - deletes the Kubernetes cluster, everything running in your cluster will be lost. Run this command if you want to start from a clean, empty cluster
- `minikube stop` - stops the Kubernetes cluster
- `minikube dashboard` - opens the Kubernetes dashboard in the browser
- `minikube tunnel` - creates a tunnel, so you can access Services with LoadBalancer type by their actual IPs


### Download and install Istio

Now that we have the Kubernetes cluster set up, we can download and install Istio. 

**MacOS and Linux**

Download and install the latest Istio release: 
```
curl -L https://git.io/getLatestIstio | sh -
```
>Warning: never blindly pipe any scripts to the shell. Always check the contents of the script before doing this.

The above script will download and extract the contents to the `istio-1.0.5` folder. This folder contains the installation files for Kubernetes, samples and the `istioctl` binary.

Let's start by creating a symbolic link for `istioctl` (alternatively, you can copy the binary to `/usr/local/bin`). Open the `/istio-1.0.5/bin` folder in your terminal/console and run:

```bash
ln -s $PWD/istioctl /usr/local/bin/istioctl
```

**Windows**

Download the executable from [here](https://github.com/istio/istio/releases/download/1.1.0-snapshot.3/istio-1.1.0-snapshot.3-win.zip) and extract the contents. Make sure the path to `istioctl` is in your `PATH` variable, so you can run it from anywhere.

##### Verify the installation 
Run `istioctl version` and check that you get an output similar to this:
```bash
Version: 1.0.5
GitRevision: a44d4c8bcb427db16ca4a439adfbd8d9361b8ed3
User: root@0ead81bba27d
Hub: docker.io/istio
GolangVersion: go1.10.4
BuildStatus: Clean
```

#### Install Istio on the cluster

1. Open the `istio-1.0.5` folder in your terminal/console.
1. Install Istio custom resource definitions and wait for about a minute or so for the CRDs to get applied.
    ```bash
    kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
    ```
1. Install Istio with mutual TLS authentication:
    ```bash
    kubectl apply -f install/kubernetes/istio-demo-auth.yaml
    ```
1. Wait for all pods to start (i.e. status changes to `Running` or `Completed`). Note that this can take a while as all images need to be downloaded and containers started. You can run the command below to check the status of pods:
    ```bash
    kubectl get pods -n istio-system 
    ```

Once you see output similar to the one below, you have successfully installed Istio on a Minikube Kubernetes cluster.

```bash
$ kubectl get pods -n istio-system
NAME                                      READY     STATUS      RESTARTS   AGE
grafana-9cfc9d4c9-85fhm                   1/1       Running     0          20m
istio-citadel-6d7f9c545b-j5g57            1/1       Running     0          20m
istio-cleanup-secrets-dj2vp               0/1       Completed   0          20m
istio-egressgateway-75dbb8f95d-vpbtx      1/1       Running     0          20m
istio-galley-6d74549bb9-2qdbh             1/1       Running     0          20m
istio-grafana-post-install-lqxdh          0/1       Completed   0          20m
istio-ingressgateway-6bd4957bc-942th      1/1       Running     0          20m
istio-pilot-7f8c49bbd8-55nd8              2/2       Running     0          20m
istio-policy-6c65d8cff4-jtvzw             2/2       Running     0          20m
istio-security-post-install-7w8cn         0/1       Completed   0          20m
istio-sidecar-injector-74855c54b9-tp2xv   1/1       Running     0          20m
istio-telemetry-65cdd46d6c-7jvpr          2/2       Running     0          20m
istio-tracing-ff94688bb-df7jq             1/1       Running     0          20m
prometheus-f556886b8-t4tvk                1/1       Running     0          20m
servicegraph-778f94d6f8-45kcf             1/1       Running     0          20m
```
