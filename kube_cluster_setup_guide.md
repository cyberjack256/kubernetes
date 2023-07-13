# Kubernetes Cluster Setup Guide: Installing containerd and Bootstrapping the Cluster

**Description:**  This document provides a step-by-step guide for setting up a Kubernetes cluster. It covers the installation of containerd, necessary system configurations, adding Docker's repository, and bootstrapping the cluster. Follow the instructions to install containerd, initialize the master node, join worker nodes, and verify successful cluster formation. Start your Kubernetes journey with confidence using this comprehensive setup guide. 

Requirements: Debian-based distributions, such as Ubuntu. It's important to note that the exact steps and package management commands may vary depending on the Linux distribution you are using. It's always recommended to consult the official documentation or specific installation guides for your chosen distribution to ensure compatibility and obtain accurate instructions.

## Installing containerd

- **Create configuration file for containerd**: This command creates a configuration file that containerd will use.
    ```
    cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
    overlay
    br_netfilter
    EOF
    ```

- **Load modules**: Load the necessary Linux kernel modules for containerd.
    ```
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

- **Set system configurations for Kubernetes networking**: This command sets necessary system configurations to enable Kubernetes to interact with the network.
    ```
    cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF
    ```

- **Apply new settings**: Apply the new system configurations.
    ```
    sudo sysctl --system
    ```

- **Add Docker’s official GPG key**: Import Docker's official GPG key for verifying packages.
    ```
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    ```

- **Set up the repository**: Add Docker's Ubuntu repository to your system's software repositories list.
    ```
    echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

- **Install containerd**: Update your local package index and install containerd.
    ```
    sudo apt-get update && sudo apt-get install -y containerd.io
    ```

- **Create default configuration file for containerd**: Create a directory for containerd configuration files if it doesn't already exist.
    ```
    sudo mkdir -p /etc/containerd
    ```

- **Generate default containerd configuration**: Generate a default containerd configuration file and save it to the directory you just created.
    ```
    sudo containerd config default | sudo tee /etc/containerd/config.toml
    ```

- **Restart containerd**: Restart the containerd service to ensure it is using the new configuration.
    ```
    sudo systemctl restart containerd
    ```

- **Verify that containerd is running**: Check that the containerd service is running.
    ```
    sudo systemctl status containerd
    ```

- **Disable swap**: Kubernetes requires swap to be disabled on Linux, so we do this with the swapoff command.
    ```
    sudo swapoff -a
    ```
    
## Bootstrapping the Cluster
- **Initialize the Cluster**: On the master node, the cluster is initialized with the following command:
    ```
    sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.24.0
    ```
    This step establishes the master node, specifies the pod network range, and designates the Kubernetes version to use.

- **Set Kubectl Access**: Next, access is established for `kubectl`, the command-line tool for Kubernetes:
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
- **Join Worker Nodes to the Cluster**: After initializing the master node, `kubeadm init` outputs a `kubeadm join` command, which includes a unique token and hash. This command is used to join the worker nodes to the cluster.
    ```
    sudo kubeadm join $some_ip:6443 --token $some_token --discovery-token-ca-cert-hash $some_hash
    kubeadm token create --print-join-command
    sudo kubeadm join <IP_ADDRESS> --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
    ```

- **Verify Nodes Have Joined**: Finally, it's confirmed that all nodes have successfully joined the cluster using the `kubectl get nodes` command. The output should list all nodes in a `NotReady` state, which is expected at this point as the network plugin is yet to be set up.
    ```
    kubectl get nodes
    ```
