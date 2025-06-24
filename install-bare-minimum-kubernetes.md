
- Install kubernetes in the node (one master):
  https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
  Steps:
  - First choose a container runtime (e.g.: crio) and add the needed repositories to the system:
    https://github.com/cri-o/packaging/blob/main/README.md#usage
```
KUBERNETES_VERSION=v1.33
CRIO_VERSION=v1.32
sudo apt-get update
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates curl gpg
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```
  - Next install the important packages:
```
sudo apt-get install -y cri-o kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
  - Enable the services (Note: kubelet will be failing until it gets the config):
```
sudo systemctl enable crio.service
sudo systemctl enable kubelet
```
  - Disable swap (on raspbian):
```
sudo systemctl disable --now dphys-swapfile
```
  - Add "br_netfilter" kernel module:
```
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf 
```
  - Enable ipv4 forward:
```
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/98-ipv4_forward.conf
sudo sysctl -p /etc/sysctl.d/98-ipv4_forward.conf
```
  - Enable some kernel parameters (on raspbian:
```
sudo vi /boot/firmware/cmdline.txt
# add this params: ipv6.disable=0 cgroup_enable=memory cgroup_memory=1 cgroup_enable=cpuset
reboot
```
  - Initialize your cluster:
```
sudo kubeadm config images pull
sudo kubeadm init
```
  - Verify that the systemd is the cgroup driver (modify it other way):
```
grep cgroupDriver /var/lib/kubelet/config.yaml
```
  - Copy the config file so you can access and control your "cluster":
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
  - At this point we should see the node as Ready:
```
$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
test      Ready    control-plane   14m   v1.33.2
```
  - As this is the only node and it has a role of "control-plane" it will not get any workload due to a condition added, as this is a just an educative approach we can just remove the "taint":
```
kubectl taint node test node-role.kubernetes.io/control-plane-
```
  - Now we can deploy applications in kubernetes, simple test:
```
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
kubectl get pods
kubectl rollout status deployment kubernetes-bootcamp
kubectl exec -ti deployment/kubernetes-bootcamp -- /bin/bash
```
  - Once we have finished playing with that recently creted pod we can remove it:
```
kubectl delete deployment kubernetes-bootcamp
```

At this point we have a working kubernetes, it has the bareminimum and there are things (like metrics) that wont work yet because we don't have yet the neccesary "plugins" installed.
We can now start installing our application.

