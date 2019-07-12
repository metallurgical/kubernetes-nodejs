## Introduction
Deploy simple NodeJS apps with kubernetes deployment. At this point, this repo used `LoadBalancer` type `ingress` to expose HTTP(s) from outside cluster to communicate with the services inside a cluster.

## Requirements
- Master Node - VPS Ubuntu 18.04 LTS with minimum 2 cores CPU and 2GB RAM
- Worker/Slave Node - VPS Ubuntu 18.04 LTS with minimum 2 cores CPU and 2GB RAM(able to skip validation if using 1GB RAM)
- Docker Image - `metallurgical/nodeexample:v0.0.1`(My own sample image which is publicly accessible on docker hub)
- Git - To clone this repository inside master node

## Installation
Installation should be done on both `master` and `worker` node

### Disable swap
Kubernetes required `swap` memory set to disable for both master and worker node 
```
sudo swapoff -a
```

### Install docker
Update server library and so forth
```
sudo apt-get update
```

Install docker
```
sudo apt install docker.io
```

After done installation, verify docker installation
```
docker --version
```

Enable docker services
```
sudo systemctl enable docker
```

Start docker services
```
sudo systemctl start docker
```

Change default driver for docker from `cgroup` to `systemd`
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

```
mkdir -p /etc/systemd/system/docker.service.d
```

```
# Restart docker.
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Set Hostname
Installations should be done on both `master` and `worker` node. For ease of identification, we might need to change the nodes's hostname.

On master node
```
sudo hostnamectl set-hostname master-node
```

On worker node
```
sudo hostnamectl set-hostname slave-node
```

## Install kubernetes
Installations should be done on both `master` and `worker` node.

Add Kubernetes signing key
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```

Add the Xenial Kubernetes repository
```
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

Install `kubeadm`(for interaction with api server)
```
sudo apt install kubeadm
```

After installation, check the version number of Kubeadm and also verify the installation through the following command
```
kubeadm version
```

### Initialize kubernetes cluster
This step only for master node, while the worker nodes only need to join the connection

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# If using 1GB RAM and 1 code CPU, you can skip validation
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU
```


After done **(copy join command to use later inside worker node)**, to start using cluster, run following command as `normal` user
```
mkdir -p $HOME/.kube
```

Copy the configuration
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

Change ownership
```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join network between worker nodes with master node

This step only for worker node

Paste `kubeadm join` command that showing up when setting up master cluster
```
kubeadm join <master-ip-address>:6443 --token <token> --discovery-token-ca-cert-hash <cert-hash>
```

Verify successful join command if the output showing

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster
```

### Verify successful join command from Master
This step only for master node

```
kubeadm get nodes
```

The output must showing status in `notready` state

### Install POD network using flannel as a medium
This step only for master node
This will install `kube-apiserver`, `kube-controller-manager`, `kube-flannel`, `kube-proxy` and `kube-scheduler`
```
sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

After done, verify the installation to see the status

```
kubectl get pods --all-namespaces
```

Ensure all the status is running, it may take sometimes to make it running, so please keep issuing the command.


### Deploy NodeJs App
This step only for master node

If everything is ready, up and running, its time to deploy our application. Head over to master CLI, navigate to `/var/www/html`. Clone this repository and run 

```
# If services still not created and running
kubectl create -f deployments.yaml --validate=false

# If services already running and created
kubectl apply -f deployments.yaml --validate=false
```

To see the created pods, run
```
kubectl get pods
```

You should able to see 3 pods were created. To open in browser, browse using following `http://<master-node-ip>:30001` and should see `hello world`

### Scale up application instance
Currently the deployment services only create 3 replicas, to scaling up issue this command

```
kubectl scale --replicas=7 deployment/myappdeployment
```

Kubernetes will spin up 4 more pods based on our configurations.


