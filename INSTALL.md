Presentation:

- Prerequisites:
  - Plaform: VM, RPI4/5, PC...
  - Linux OS: raspbian, debian, fedora, RHEL (or variants)... Needs to have native packages for kubernetes (and crio); Already installed (basic installation)
  - A hard-drive big enough for your data, (i recommend anything bigger than 100gb, e.g.: 2Tb, but for testing you can go lower, e.g.: 20gb, maybe even lower)  
  - internet connection
  - a simple DNS server in your network (not really needed but highly recommended)
  - node must have a vaild hostname
  
For this example i choose to use a raspberry pi 5 as the hardware and Raspberry pi OS (debian) as operating system.

Steps:
- Install kubernetes in the node (one master):
  https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
  Steps:
  - First choose a container runtime (e.g.: crio) and add the needed repositories to the system:
    https://github.com/cri-o/packaging/blob/main/README.md#usage
    ~~~
KUBERNETES_VERSION=v1.33
CRIO_VERSION=v1.32
sudo apt-get update
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates curl gpg
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
    ~~~
  - Next install the important packages:
    ~~~
sudo apt-get install -y cri-o kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
    ~~~
  - Enable the services (Note: kubelet will be failing until it gets the config):
    ~~~
sudo systemctl enable --now crio.service
sudo systemctl enable --now kubelet
    ~~~
  - Disable swap:
    ~~~
sudo systemctl disable --now dphys-swapfile
    ~~~
  - Add "br_netfilter" kernel module:
    ~~~
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf 
    ~~~
  - Enable ipv4 forward:
    ~~~
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/98-ipv4_forward.conf
sudo sysctl -p /etc/sysctl.d/98-ipv4_forward.conf
    ~~~
  - Enable some kernel parameters (based on experience):
    ~~~
sudo vi /boot/firmware/cmdline.txt
# add this params: ipv6.disable=0 cgroup_enable=memory cgroup_memory=1 cgroup_enable=cpuset
reboot
    ~~~
  - Initialize your cluster:
    ~~~
sudo kubeadm config images pull
sudo kubeadm init
    ~~~
  - Verify that the systemd is the cgroup driver (modify it other way):
    ~~~
grep cgroupDriver /var/lib/kubelet/config.yaml
    ~~~
  - Copy the config file so you can access and control your "cluster":
    ~~~
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ~~~
  
  Now we can deploy applications in kubernetes, simple test:
  ~~~
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
kubectl get pods -w
kubectl rollout status deployment kubernetes-bootcamp
kubectl exec -ti deployment/kubernetes-bootcamp -- /bin/bash
   ~~~
- In order to not deal with modifying lot of yaml files for the deployments we will use HELM, think of it as a kind of repository with yaml templates for products + Jinja like variable substitution so it is easy to deploy an application with one command already configured for your environment:
  https://helm.sh/docs/intro/install/
  This is client, so install it in the host/s from where you want to control de cluster:



- For our applications we will need storage... but we are lazy so we want the storage to be managed by kubernetes, we want it to be dynamic, isolated (from the rest of containers, what about LVM volumes?)... we are in a single node deployment... looks like a good place to try "topolvm" a simple lvm provisioner for kubernetes


