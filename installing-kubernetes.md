# Installing Kubernetes

Following the instructions for using
[kubeadm](https://kubernetes.io/docs/getting-started-guides/kubeadm/) to setup
a Kubernetes cluster.

## Configuring the Master

### Installing Kubernetes on the Master

```sh
gcloud compute ssh controller0
sudo bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y docker.io
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
kubeadm init
```

After `kubeadm` completes, it will print out a `kubeadm` command line. Keep a
copy of that. You'll need to run that on your worker. It has the shared secret,
and the host IP of the master, that will allow a worker node to join the
cluster.

It will look something like this:

```sh
kubeadm join --token=ced5ef.c5e083173f9eb4a9 10.121.0.2
```

## Checking the Cluster Status

At this point your cluster should be responding to `kubectl` commands, though
it will be a cluster of only one node, the master.

```sh
kubectl get nodes
```

```
NAME          STATUS         AGE
controller0   Ready,master   14m
```

### Installing a Network Overlay

```sh
kubectl apply -f https://git.io/weave-kube
```

Confirm this worked by looking to see that your dns pod is running:

```sh
kubectl get pods --all-namespaces | grep "kube-dns.*Running"
```

```sh
kube-system   kube-dns-2924299975-r40tz             4/4       Running    0          10m
```

## Installing Kubernetes on a Worker Node

```sh
gcloud compute ssh worker0
sudo bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
# Install docker if you don't have it already.
apt-get install -y docker.io
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

## Check the Node Joined

Back on the master, you should be able to see your new worker node in the
cluster:

```sh
kubectl get nodes
```

```sh
NAME          STATUS         AGE
controller0   Ready,master   8m
worker0       Ready          11s
```

## Test Your Cluster

Check that you can run a pod.

```sh
kubectl run nginx --image=nginx:latest
```

Wait for a minute or two while it initializes, and you should see:

```sh
kubectl get pods
```

```sh
NAME                     READY     STATUS    RESTARTS   AGE
nginx-1984600839-rjcr6   1/1       Running   0          17s
```

