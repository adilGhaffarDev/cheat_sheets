# Setup kubeadm cluster

## Enable iptables bridged traffic on all the nodes
```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```sh
sudo modprobe overlay
sudo modprobe br_netfilter
```
## Sysctl params required by setup, params persist across reboots  on all nodes
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
Apply sysctl params without reboot  on all nodes
```sh
sudo sysctl --system
```

## Disable swap on all nodes
```sh
sudo swapoff -a
sudo nano /etc/fstab
```
comment this line: "/swap.img	none	swap	sw	0	0"

## Install container runtime on all nodes
1. We are using cri-o.
    ```sh
    cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
    overlay
    br_netfilter
    EOF
    ```
2. Set up required sysctl params, these persist across reboots.
    ```sh
    cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF
    ```
3. Execute the following commands to enable overlayFS & VxLan pod communication.
    ```sh
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```
4. Reload the parameters.
    ```sh
    sudo sysctl --system
    ```
5. Enable cri-o repositories for version 1.23
    ```sh
    OS="xUbuntu_20.04"
    VERSION="1.23"
    ```
    ```sh
    cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
    deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
    EOF
    ```
    ```sh
    cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
    deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
    EOF
    ```
6. Add the gpg keys.
    ```sh
    curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
    ```
    ```sh
    curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
    ```
7. Update and install crio and crio-tools.
    ```sh
    sudo apt-get update
    sudo apt-get install cri-o cri-o-runc cri-tools -y
    ```
8. Reload the systemd configurations and enable cri-o.
    ```sh
    sudo systemctl daemon-reload
    sudo systemctl enable crio --now
    ```

## Install Kubeadm & Kubelet & Kubectl on all Nodes
1. Update the system repositories:
    ```sh
    sudo apt update
    ```
2. Install the apt-transport-https and ca-certificates packages, along with the curl CLI tool.
    ```sh
    sudo apt install -y apt-transport-https ca-certificates curl
    ```
3. Use curl to download Google Cloud GPG key and verify the integrity of downloads.
   ```sh
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
     ```
4. Add the Kubernetes repository to your system.
    ```sh
   echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```
5. Refresh the package list.
    ```sh
   sudo apt update
    ```
6. Install the Kubernetes management tools - the kubelet node agent, the kubeadm command, and the kubectl CLI tool for cluster management.
    ```sh
    sudo apt install -y kubelet kubeadm kubectl
    ```
7. Use the apt-mark hold command to ensure the tools cannot be accidentally reinstalled, upgraded, or removed.
    ```sh
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
8. Add the node IP to KUBELET_EXTRA_ARGS.
    ```sh
    sudo apt-get install -y jq
    ```
    ```sh
    local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
    cat > /etc/default/kubelet << EOF
    KUBELET_EXTRA_ARGS=--node-ip=$local_ip
    EOF
    ```
## Init kubeadm on master node
1. Set the following environment variables. Replace 10.0.0.10 with the IP of your master node. IMPORTANT: make sure pod ip rnage dont overlap with node ips
    ```sh
    IPADDR="10.0.0.10"
    NODENAME=$(hostname -s)
    POD_CIDR="10.10.24.0/16"
    ```
2. Now, initialize the master node control plane configurations using the following kubeadm command.
    ```sh
    sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME
    ```
3. On success, Use the following commands from the output to create the kubeconfig in master so that you can use kubectl to interact with cluster API
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
4. By default, apps won’t get scheduled on the master node. If you want to use the master node for scheduling apps, taint the master node.
    ```sh
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```
## Install Calico on mater
Execute the following command to install the calico network plugin on the cluster.
```sh
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```


## Join worker nodes
Run following command on master to get join command
```sh
kubeadm token create --print-join-command
```
Join command will look like this:
```sh
sudo kubeadm join 192.168.122.37:6443 --token 5vta5m.wrgpzaysgkbskld2 --discovery-token-ca-cert-hash sha256:b98d7b6b94d6d38c8303a77fbbf0416937cd42e7ace5388094400e18866ddc7f
```
Copy it and run it on all workers with sudo

You can set Role of worker nodes by stting labe using the following command on master:
```sh
kubectl label node <worker-hostname>  node-role.kubernetes.io/worker=worker
```